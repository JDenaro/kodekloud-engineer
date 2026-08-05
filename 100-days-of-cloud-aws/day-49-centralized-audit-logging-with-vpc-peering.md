# Day 49: Centralized Audit Logging with VPC Peering

The Nautilus DevOps team needs to build a secure and scalable log aggregation setup within their AWS environment. The goal is to gather log files from an internal EC2 instance running in a private VPC, transfer them securely to another EC2 instance in a public VPC, and then push those logs to a secure S3 bucket.

## Specific Requirements:

1. A VPC named `datacenter-priv-vpc` already exists with a private subnet named `datacenter-priv-subnet`, a route table named `datacenter-priv-rt`, and an EC2 instance named `datacenter-priv-ec2` (using ubuntu image). This instance uses the SSH key pair `datacenter-key.pem` already available on the AWS client host at `/root/.ssh/`.
2. Your task is to:
   - Create a new VPC named `datacenter-pub-vpc`.
   - Create a subnet named `datacenter-pub-subnet` and a route table named `datacenter-pub-rt` under this public VPC.
   - Attach an internet gateway to `datacenter-pub-vpc` and configure the public route table to enable internet access.
   - Launch an EC2 instance named `datacenter-pub-ec2` into the public subnet using the same key pair as the private instance.
   - Create an IAM role named `datacenter-s3-role` with PutObject permission to an S3 bucket and attach it to the public EC2 instance.
   - Create a new private S3 bucket named `datacenter-s3-logs-2707`.
   - Configure a VPC Peering named `datacenter-vpc-peering` between the private and public VPCs.
   - Modify both `datacenter-priv-rt` and `datacenter-pub-rt` to route each other's CIDR blocks through the peering connection.
   - On the private instance, configure a cron job to push the `/var/log/boots.log` file to the public instance (using scp or rsync).
   - On the public instance, configure a cron job to push that same file to the created S3 bucket.
   - The uploaded file must be stored in the S3 bucket under the path `datacenter-priv-vpc/boot/boots.log`.

## Solution

This challenge chains three ideas: **VPC peering** (a private link between two VPCs so their instances talk over private IPs), an **IAM instance role** (so the public instance uploads to S3 without static keys), and two **cron jobs** that move the log file hop by hop (private → public → S3).

The key gotcha discovered while solving it: **the AWS client host cannot reach the private instance directly**. The private instance lives in a private subnet with no public IP, and the client host has no route into the private VPC's CIDR — so every command against the private instance must be tunneled *through* the public instance using SSH `ProxyCommand` (a jump host). That tunnel only works once the peering connection **and** both cross-routes exist, so it doubles as an end-to-end connectivity test.

### 📦 Variables

Values that come directly from the challenge description. Discovered IDs (private VPC ID, CIDR, route table, instance ID/IP, AZ, AMI) are captured in Step 1.

```bash
AWS_REGION="us-east-1"

PRIV_VPC_NAME="datacenter-priv-vpc"
PRIV_RT_NAME="datacenter-priv-rt"
PRIV_EC2_NAME="datacenter-priv-ec2"

PUB_VPC_NAME="datacenter-pub-vpc"
PUB_SUBNET_NAME="datacenter-pub-subnet"
PUB_RT_NAME="datacenter-pub-rt"
PUB_IGW_NAME="datacenter-pub-igw"
PUB_SG_NAME="datacenter-pub-sg"
PUB_EC2_NAME="datacenter-pub-ec2"

PEERING_NAME="datacenter-vpc-peering"
ROLE_NAME="datacenter-s3-role"
POLICY_NAME="datacenter-s3-put-policy"
BUCKET="datacenter-s3-logs-2707"

KEY_PATH="/root/.ssh/datacenter-key.pem"
KEY_NAME="datacenter-key"

PUB_CIDR="10.20.0.0/16"
PUB_SUBNET_CIDR="10.20.1.0/24"

S3_KEY_PATH="datacenter-priv-vpc/boot/boots.log"
```

### 🔎 Step 1: Discover the existing private VPC

We must never create the private side — it already exists. First we look up its identifiers, because the peering and the cross-routes need the private VPC's ID and CIDR, and we want to place the public instance in the same Availability Zone and use the same Ubuntu AMI/instance type as the private one.

```bash
PRIV_VPC_ID=$(aws ec2 describe-vpcs --region "$AWS_REGION" \
  --filters "Name=tag:Name,Values=$PRIV_VPC_NAME" \
  --query "Vpcs[0].VpcId" --output text)

PRIV_CIDR=$(aws ec2 describe-vpcs --region "$AWS_REGION" \
  --vpc-ids "$PRIV_VPC_ID" --query "Vpcs[0].CidrBlock" --output text)

PRIV_RT_ID=$(aws ec2 describe-route-tables --region "$AWS_REGION" \
  --filters "Name=tag:Name,Values=$PRIV_RT_NAME" \
  --query "RouteTables[0].RouteTableId" --output text)

PRIV_EC2_ID=$(aws ec2 describe-instances --region "$AWS_REGION" \
  --filters "Name=tag:Name,Values=$PRIV_EC2_NAME" "Name=instance-state-name,Values=running" \
  --query "Reservations[0].Instances[0].InstanceId" --output text)

PRIV_EC2_IP=$(aws ec2 describe-instances --region "$AWS_REGION" --instance-ids "$PRIV_EC2_ID" \
  --query "Reservations[0].Instances[0].PrivateIpAddress" --output text)

AZ=$(aws ec2 describe-instances --region "$AWS_REGION" --instance-ids "$PRIV_EC2_ID" \
  --query "Reservations[0].Instances[0].Placement.AvailabilityZone" --output text)

AMI=$(aws ec2 describe-instances --region "$AWS_REGION" --instance-ids "$PRIV_EC2_ID" \
  --query "Reservations[0].Instances[0].ImageId" --output text)

ITYPE=$(aws ec2 describe-instances --region "$AWS_REGION" --instance-ids "$PRIV_EC2_ID" \
  --query "Reservations[0].Instances[0].InstanceType" --output text)
```

Real values from the lab run:

```
PRIV_VPC_ID=vpc-006d576490d5c95ee  CIDR=10.10.0.0/16  RT=rtb-049ab26eb67b3071a
PRIV_EC2_ID=i-0b4b897ecf7c27cf2    IP=10.10.1.21      AZ=us-east-1a
AMI=ami-0fb0b230890ccd1e6          TYPE=t3.micro
```

> **Why:** `describe-vpcs` and `describe-instances` are read-only lookups. A *VPC* (Virtual Private Cloud) is an isolated virtual network; its *CIDR block* is its private IP range. The `--filters "Name=tag:Name,..."` argument finds a resource by its *Name* tag (a label, not the resource ID). `--query` uses JMESPath to pull a single field, and `--output text` returns it bare so we can store it in a shell variable. We filter instances by `instance-state-name=running` so a leftover terminated instance never matches. Capturing the *Availability Zone* (an isolated datacenter within the region), *AMI* (Amazon Machine Image — the OS template), and *instance type* (the hardware size) lets us make the public instance a faithful twin of the private one.

### 🌐 Step 2: Create the public VPC

```bash
PUB_VPC_ID=$(aws ec2 create-vpc --region "$AWS_REGION" --cidr-block "$PUB_CIDR" --query "Vpc.VpcId" --output text)

aws ec2 create-tags --region "$AWS_REGION" --resources "$PUB_VPC_ID" --tags Key=Name,Value="$PUB_VPC_NAME"
```

> **Why:** `create-vpc` provisions the new isolated network; `--cidr-block 10.20.0.0/16` sets its IP range. This range must **not overlap** with the private VPC's `10.10.0.0/16` — overlapping CIDRs make VPC peering impossible because routing between them would be ambiguous, and AWS rejects the peering. We add the *Name* tag in a separate `create-tags` call (rather than inline) to keep the creation command simple; `--resources` lists the IDs to tag and `--tags` supplies the key/value.

### 🧩 Step 3: Create the public subnet

```bash
PUB_SUBNET_ID=$(aws ec2 create-subnet --region "$AWS_REGION" --vpc-id "$PUB_VPC_ID" \
  --cidr-block "$PUB_SUBNET_CIDR" --availability-zone "$AZ" \
  --query "Subnet.SubnetId" --output text)

aws ec2 create-tags --region "$AWS_REGION" --resources "$PUB_SUBNET_ID" --tags Key=Name,Value="$PUB_SUBNET_NAME"

aws ec2 modify-subnet-attribute --region "$AWS_REGION" --subnet-id "$PUB_SUBNET_ID" --map-public-ip-on-launch
```

> **Why:** a *subnet* is a slice of the VPC's CIDR where instances actually launch. `--cidr-block 10.20.1.0/24` is a block inside the VPC range, and `--availability-zone` pins it to the same AZ as the private instance. `modify-subnet-attribute --map-public-ip-on-launch` makes every instance launched here automatically receive a public IP — required so the public instance can reach the S3 endpoint over the internet (via the gateway we add next) and so we can SSH into it.

### 🚪 Step 4: Create and attach the Internet Gateway

```bash
IGW_ID=$(aws ec2 create-internet-gateway --region "$AWS_REGION" --query "InternetGateway.InternetGatewayId" --output text)

aws ec2 create-tags --region "$AWS_REGION" --resources "$IGW_ID" --tags Key=Name,Value="$PUB_IGW_NAME"

aws ec2 attach-internet-gateway --region "$AWS_REGION" --internet-gateway-id "$IGW_ID" --vpc-id "$PUB_VPC_ID"
```

> **Why:** an *Internet Gateway* (IGW) is the component that allows traffic between a VPC and the public internet. `create-internet-gateway` creates it detached; `attach-internet-gateway` binds it to the public VPC. Without this attachment, any `0.0.0.0/0` route would have no working target.

### 🧭 Step 5: Public route table, internet route, and association

```bash
PUB_RT_ID=$(aws ec2 create-route-table --region "$AWS_REGION" --vpc-id "$PUB_VPC_ID" --query "RouteTable.RouteTableId" --output text)

aws ec2 create-tags --region "$AWS_REGION" --resources "$PUB_RT_ID" --tags Key=Name,Value="$PUB_RT_NAME"

aws ec2 create-route --region "$AWS_REGION" --route-table-id "$PUB_RT_ID" \
  --destination-cidr-block 0.0.0.0/0 --gateway-id "$IGW_ID"

aws ec2 associate-route-table --region "$AWS_REGION" --route-table-id "$PUB_RT_ID" --subnet-id "$PUB_SUBNET_ID"
```

> **Why:** a *route table* holds rules that decide where network traffic goes. `create-route` with destination `0.0.0.0/0` (all internet) pointing to the IGW is exactly what makes the subnet "public". `associate-route-table` links these rules to `datacenter-pub-subnet`; without the association the subnet would fall back to the VPC's main route table, which has no internet route.

### 🪣 Step 6: Create the private S3 bucket

```bash
aws s3api create-bucket --bucket "$BUCKET" --region "$AWS_REGION"

aws s3api put-public-access-block --bucket "$BUCKET" \
  --public-access-block-configuration BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true
```

> **Why:** `create-bucket` creates the S3 bucket; in `us-east-1` you do **not** pass `LocationConstraint` because it is S3's default region. `put-public-access-block` with all four flags set to `true` guarantees the bucket stays **private** — it blocks public ACLs and public bucket policies, satisfying the "secure/private S3 bucket" requirement.

### 🔐 Step 7: IAM role, PutObject policy, and instance profile

```bash
cat > /tmp/ec2-trust.json <<'EOF'
{ "Version": "2012-10-17",
  "Statement": [ { "Effect": "Allow", "Principal": { "Service": "ec2.amazonaws.com" }, "Action": "sts:AssumeRole" } ] }
EOF

aws iam create-role --role-name "$ROLE_NAME" --assume-role-policy-document file:///tmp/ec2-trust.json

cat > /tmp/s3-put.json <<EOF
{ "Version": "2012-10-17",
  "Statement": [ { "Effect": "Allow", "Action": "s3:PutObject", "Resource": "arn:aws:s3:::$BUCKET/*" } ] }
EOF

POLICY_ARN=$(aws iam create-policy --policy-name "$POLICY_NAME" \
  --policy-document file:///tmp/s3-put.json --query "Policy.Arn" --output text)

aws iam attach-role-policy --role-name "$ROLE_NAME" --policy-arn "$POLICY_ARN"

aws iam create-instance-profile --instance-profile-name "$ROLE_NAME"

aws iam add-role-to-instance-profile --instance-profile-name "$ROLE_NAME" --role-name "$ROLE_NAME"

sleep 15
```

> **Why:** an *IAM role* is a set of permissions an entity can assume without long-lived credentials. The `--assume-role-policy-document` (the *trust policy*) declares **who** may assume it: the EC2 service, via `sts:AssumeRole`. We then grant **least privilege** — a customer-managed policy allowing only `s3:PutObject` on `arn:aws:s3:::datacenter-s3-logs-2707/*` (objects inside the bucket) — created with `create-policy` and bound with `attach-role-policy`. We use a *managed* policy rather than an inline one because the KodeKloud sandbox user is denied `iam:PutRolePolicy`. EC2 cannot attach a role directly; it uses an *instance profile* wrapper, so `create-instance-profile` + `add-role-to-instance-profile` puts the role inside a profile of the same name. The `sleep 15` lets the new profile propagate through IAM's eventually-consistent backend before we reference it at launch.

### 🛡️ Step 8: Security group for the public instance

```bash
PUB_SG_ID=$(aws ec2 create-security-group --region "$AWS_REGION" --group-name "$PUB_SG_NAME" \
  --description "SSH access for $PUB_EC2_NAME" --vpc-id "$PUB_VPC_ID" \
  --query "GroupId" --output text)

aws ec2 authorize-security-group-ingress --region "$AWS_REGION" --group-id "$PUB_SG_ID" \
  --protocol tcp --port 22 --cidr 0.0.0.0/0
```

> **Why:** a *security group* is a virtual firewall at the instance level. `create-security-group` creates it in the public VPC. `authorize-security-group-ingress` opens inbound TCP port 22 (SSH) so we can manage the instance from the client host **and** so the private instance's `scp` (arriving over the peering with a `10.10.x.x` source, covered by `0.0.0.0/0`) is allowed.

### 🖥️ Step 9: Launch the public EC2 instance

```bash
PUB_EC2_ID=$(aws ec2 run-instances --region "$AWS_REGION" --image-id "$AMI" --instance-type "$ITYPE" \
  --key-name "$KEY_NAME" --subnet-id "$PUB_SUBNET_ID" --security-group-ids "$PUB_SG_ID" \
  --iam-instance-profile Name="$ROLE_NAME" \
  --query "Instances[0].InstanceId" --output text)

aws ec2 create-tags --region "$AWS_REGION" --resources "$PUB_EC2_ID" --tags Key=Name,Value="$PUB_EC2_NAME"

aws ec2 wait --region "$AWS_REGION" instance-running --instance-ids "$PUB_EC2_ID"

PUB_PRIV_IP=$(aws ec2 describe-instances --region "$AWS_REGION" --instance-ids "$PUB_EC2_ID" \
  --query "Reservations[0].Instances[0].PrivateIpAddress" --output text)

PUB_PUB_IP=$(aws ec2 describe-instances --region "$AWS_REGION" --instance-ids "$PUB_EC2_ID" \
  --query "Reservations[0].Instances[0].PublicIpAddress" --output text)
```

Real values from the lab run: `PUB_EC2_ID=i-02008801839c09326  PRIV_IP=10.20.1.139  PUB_IP=44.222.76.249`.

> **Why:** `run-instances` launches the VM, reusing the same AMI, instance type and `--key-name datacenter-key` as the private one. `--subnet-id` places it in the public subnet (so it gets a public IP), `--security-group-ids` applies the firewall, and `--iam-instance-profile Name=...` attaches the role so the instance can upload to S3 with automatic temporary credentials. The `sleep 15` at the end of Step 7 was enough for the instance profile to finish propagating, so this call succeeded on the first try — a freshly created profile can otherwise briefly be rejected as invalid until IAM catches up, in which case the same `run-instances` command just needs to be re-run a few seconds later. `wait instance-running` blocks until the instance reaches the *running* state, guaranteeing the public/private IPs are assigned before we read them.

### 🔗 Step 10: VPC peering and cross-routes

```bash
PEERING_ID=$(aws ec2 create-vpc-peering-connection --region "$AWS_REGION" --vpc-id "$PUB_VPC_ID" \
  --peer-vpc-id "$PRIV_VPC_ID" --query "VpcPeeringConnection.VpcPeeringConnectionId" --output text)

sleep 3

aws ec2 accept-vpc-peering-connection --region "$AWS_REGION" --vpc-peering-connection-id "$PEERING_ID"

aws ec2 create-tags --region "$AWS_REGION" --resources "$PEERING_ID" --tags Key=Name,Value="$PEERING_NAME"

aws ec2 create-route --region "$AWS_REGION" --route-table-id "$PUB_RT_ID"  \
  --destination-cidr-block "$PRIV_CIDR" --vpc-peering-connection-id "$PEERING_ID"

aws ec2 create-route --region "$AWS_REGION" --route-table-id "$PRIV_RT_ID" \
  --destination-cidr-block "$PUB_CIDR"  --vpc-peering-connection-id "$PEERING_ID"
```

> **Why:** `create-vpc-peering-connection` establishes a private network link between two VPCs — `--vpc-id` is the requester (public) and `--peer-vpc-id` the accepter (private). Because both VPCs are in the same account and region, `accept-vpc-peering-connection` approves it immediately, moving it to *active*. Peering by itself routes nothing, so we add symmetric routes: the public route table learns the private CIDR (`10.10.0.0/16`) and the private route table learns the public CIDR (`10.20.0.0/16`), both with `--vpc-peering-connection-id` as the next hop. Without **both** routes, traffic would flow one way but never return.

### 📡 Step 11: Prepare the SSH jump host and install AWS CLI

The client host cannot reach the private instance directly, so every private-instance command is tunneled through the public instance. We first wait for SSH on the public instance, then install the AWS CLI there (the Ubuntu AMI ships without it).

```bash
SSH_OPTS="-o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -o ConnectTimeout=10"

ssh -i "$KEY_PATH" $SSH_OPTS ubuntu@"$PUB_PUB_IP" "true"
```

A freshly launched instance can take a little while to finish booting `sshd`; a connection refused/timed out here just means re-running the same `ssh` command a few seconds later until it succeeds silently.

```bash
PROXY="ssh -i $KEY_PATH $SSH_OPTS -W %h:%p ubuntu@$PUB_PUB_IP"

ssh -i "$KEY_PATH" $SSH_OPTS ubuntu@"$PUB_PUB_IP" '
  sudo cloud-init status --wait
  sudo DEBIAN_FRONTEND=noninteractive apt-get update -qq
  sudo DEBIAN_FRONTEND=noninteractive apt-get install -y awscli
  aws --version
  aws sts get-caller-identity
'
```

Expected output confirms the CLI is installed and the instance is using the role:

```
aws-cli/1.18.69 Python/3.8.10 Linux/5.15.0-1084-aws botocore/1.16.19
"Arn": "arn:aws:sts::431380056739:assumed-role/datacenter-s3-role/i-02008801839c09326"
```

> **Why:** `StrictHostKeyChecking=no` and `UserKnownHostsFile=/dev/null` stop SSH from prompting about unknown host keys (which would hang a script), and `ConnectTimeout=10` bounds each attempt. The `PROXY` string is an SSH `ProxyCommand`: `-W %h:%p` tells the public instance to forward the connection to the final host/port, turning it into a *jump host* so we can reach the private instance's `10.10.x.x` address over the peering. We install `awscli` via `apt-get` (with `DEBIAN_FRONTEND=noninteractive` to avoid prompts) because the public instance's cron needs `aws s3 cp`. **Crucial gotcha:** a freshly booted instance runs `cloud-init` and `unattended-upgrades`, which hold the `dpkg`/`apt` lock — installing too early fails with "Could not get lock". `cloud-init status --wait` blocks until boot-time setup finishes and releases the lock before `apt-get` runs, which is what let this single pass through `apt-get update`/`install` succeed without contention. `sts get-caller-identity` returning an `assumed-role/datacenter-s3-role` ARN proves the instance profile is working and no static keys are needed.

### ⏱️ Step 12: Cron on the public instance (upload to S3)

```bash
ssh -i "$KEY_PATH" $SSH_OPTS ubuntu@"$PUB_PUB_IP" \
  "(crontab -l 2>/dev/null; echo '* * * * * /usr/bin/aws s3 cp /home/ubuntu/boots.log s3://$BUCKET/$S3_KEY_PATH --region $AWS_REGION') | crontab -"
```

> **Why:** the cron schedule `* * * * *` runs every minute. `aws s3 cp` copies the `boots.log` that the private instance delivers into `/home/ubuntu/` up to the bucket, using the exact required object path `datacenter-priv-vpc/boot/boots.log`. We use the absolute path `/usr/bin/aws` because cron runs with a minimal `PATH`. No credentials are configured — the command uses the instance profile automatically. The `(crontab -l …; echo …) | crontab -` idiom appends a line without clobbering any existing crontab.

### ⏱️ Step 13: Deliver the key and set the private cron (scp to public)

The private instance needs the private key to authenticate to the public instance, so we copy it through the jump host, verify the hop works, then install its cron.

```bash
scp -i "$KEY_PATH" $SSH_OPTS -o ProxyCommand="$PROXY" \
  "$KEY_PATH" ubuntu@"$PRIV_EC2_IP":/home/ubuntu/datacenter-key.pem

ssh -i "$KEY_PATH" $SSH_OPTS -o ProxyCommand="$PROXY" ubuntu@"$PRIV_EC2_IP" \
  "chmod 600 /home/ubuntu/datacenter-key.pem"

ssh -i "$KEY_PATH" $SSH_OPTS -o ProxyCommand="$PROXY" ubuntu@"$PRIV_EC2_IP" \
  "scp -i /home/ubuntu/datacenter-key.pem -o StrictHostKeyChecking=no /var/log/boots.log ubuntu@$PUB_PRIV_IP:/home/ubuntu/boots.log && echo SCP_OK"

ssh -i "$KEY_PATH" $SSH_OPTS -o ProxyCommand="$PROXY" ubuntu@"$PRIV_EC2_IP" \
  "(crontab -l 2>/dev/null; echo '* * * * * /usr/bin/scp -i /home/ubuntu/datacenter-key.pem -o StrictHostKeyChecking=no /var/log/boots.log ubuntu@$PUB_PRIV_IP:/home/ubuntu/boots.log') | crontab -"
```

> **Why:** `scp` (secure copy over SSH) with `-o ProxyCommand` places the key on the private instance via the jump host. `chmod 600` makes the key readable only by its owner — SSH refuses to use a key that others can read. We test the private→public hop manually first (`SCP_OK`) to avoid debugging blind inside cron; it targets the public instance's **private** IP (`10.20.1.139`) so traffic flows over the peering. Finally the private cron runs `scp` every minute, completing the chain **private → public → S3**.

### ✅ Step 14: Verify

```bash
ssh -i "$KEY_PATH" $SSH_OPTS ubuntu@"$PUB_PUB_IP" \
  "/usr/bin/aws s3 cp /home/ubuntu/boots.log s3://$BUCKET/$S3_KEY_PATH --region $AWS_REGION"

aws s3 ls "s3://$BUCKET/$S3_KEY_PATH"
```

Success looks like the object listed at the required path:

```
upload: ./boots.log to s3://datacenter-s3-logs-2707/datacenter-priv-vpc/boot/boots.log
2026-07-20 00:07:03         27 boots.log
```

The file `boots.log` now exists in the bucket under `datacenter-priv-vpc/boot/boots.log`, proving the full pipeline works end to end.

> **Why:** we trigger one upload immediately (instead of waiting up to a minute for cron) by running the same `aws s3 cp` command the public instance's cron uses, so verification is instant. `aws s3 ls s3://<bucket>/<key>` then lists the object at the exact required path — if it prints a size and timestamp, the object landed where the challenge demands (`datacenter-priv-vpc/boot/boots.log`). This confirms the whole chain (private → public → S3) end to end.

## Best Practices

- **Non-overlapping CIDRs are mandatory for peering.** We chose `10.20.0.0/16` for the public VPC precisely because it does not overlap the private `10.10.0.0/16`; overlapping ranges make peering impossible.
- **Peering needs routes on both sides.** The connection only carries traffic once each route table has a route to the other VPC's CIDR via the peering — a one-sided route yields silent, one-way failures.
- **Use an instance role, never static keys.** Attaching `datacenter-s3-role` lets the public instance upload to S3 with automatic, rotating credentials; no access keys ever touch the disk.
- **Least-privilege IAM.** The policy grants only `s3:PutObject` on the one bucket's objects, not blanket S3 access.
- **Keep the bucket private.** A full public-access block ensures logs are never exposed, matching the "secure S3 bucket" requirement.
- **Reach private instances through a jump host.** With no public IP and no route from the client host, tunneling via the public instance over the peering is the correct, NAT-free way to administer the private instance.
- **Protect the SSH private key.** `chmod 600` on the copied key is required or SSH will reject it.

### 📚 Official Documentation

- [Create a VPC peering connection](https://docs.aws.amazon.com/vpc/latest/peering/create-vpc-peering-connection.html)
- [Update your route tables for a VPC peering connection](https://docs.aws.amazon.com/vpc/latest/peering/vpc-peering-routing.html)
- [Using an IAM role to grant permissions to applications running on Amazon EC2 instances](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_use_switch-role-ec2.html)
- [Blocking public access to your Amazon S3 storage](https://docs.aws.amazon.com/AmazonS3/latest/userguide/access-control-block-public-access.html)
- [cp — AWS CLI S3 Command Reference](https://docs.aws.amazon.com/cli/latest/reference/s3/cp.html)
