# Day 30: Enable Internet Access for Private EC2 using NAT Instance

The Nautilus DevOps team is tasked with enabling internet access for an EC2 instance running in a private subnet. This instance should be able to upload a test file to a public S3 bucket once it can access the internet. To minimize costs, the team has decided to use a NAT Instance instead of a NAT Gateway.

The following components already exist in the environment:
1) A VPC named `devops-priv-vpc` and a private subnet named `devops-priv-subnet` have been created.
2) An EC2 instance named `devops-priv-ec2` is already running in the private subnet.
3) The EC2 instance is configured with a cron job that uploads a test file to the S3 bucket `devops-nat-28358` every minute. Upload will only succeed once internet access is established.

Your task is to:

Create a new public subnet named `devops-pub-subnet` in the existing VPC.
Launch a NAT Instance in the public subnet using an Amazon Linux 2023 AMI and name it `devops-nat-instance`. Configure this instance to act as a NAT instance. Make sure to use a custom security group for this instance.
After the configuration, verify that the test file `devops-test.txt` appears in the S3 bucket `devops-nat-28358`. This indicates successful internet access from the private EC2 instance via the NAT Instance.

Note: iptables is not installed by default on Amazon Linux 2023. You will need to install and enable it before configuring NAT setup.

## Specific Requirements:

1. Create a new public subnet named `devops-pub-subnet` in the existing VPC.
2. Launch a NAT Instance in the public subnet using an Amazon Linux 2023 AMI and name it `devops-nat-instance`.
3. Configure this instance to act as a NAT instance.
4. Make sure to use a custom security group for this instance.
5. After the configuration, verify that the test file `devops-test.txt` appears in the S3 bucket `devops-nat-28358`.

## Solution

A **NAT (Network Address Translation) instance** is a regular EC2 instance that you configure by hand to forward traffic from private instances out to the internet — the do-it-yourself, cheaper cousin of the managed **NAT Gateway**. The traffic path we build is: private instance → private route table → NAT instance (in a public subnet) → Internet Gateway → internet. Three things make a NAT instance actually forward packets, and missing any one of them causes silent failure:

- The **source/destination check must be disabled** — by default EC2 drops any packet whose source/destination isn't the instance itself, which is exactly what forwarding requires.
- The OS must **forward IP packets** (`net.ipv4.ip_forward=1`) and **masquerade** them (an `iptables` NAT rule that rewrites the private source IP to the NAT instance's own address).
- The private subnet's **route table** must send `0.0.0.0/0` to the NAT instance.

One Amazon Linux 2023 gotcha called out by the challenge: `iptables` isn't preinstalled, so the instance's **user data** (a script AWS runs on first boot) installs it before configuring NAT. Because user data only runs on a *freshly launched* instance, we bake the whole NAT setup into it at launch time.

### 📦 Variables

```bash
AWS_REGION="us-east-1"
VPC_NAME="devops-priv-vpc"
PRIVATE_SUBNET_NAME="devops-priv-subnet"
PRIVATE_INSTANCE_NAME="devops-priv-ec2"
PUBLIC_SUBNET_NAME="devops-pub-subnet"
NAT_INSTANCE_NAME="devops-nat-instance"
NAT_SG_NAME="devops-nat-sg"
S3_BUCKET="devops-nat-28358"
TEST_FILE="devops-test.txt"
```

> **Why:** We keep the challenge-given names (VPC, private subnet, private instance, the public subnet and NAT instance we must create, the bucket, and the test file) in one place so the commands below read cleanly. `devops-nat-sg` is the name we choose for the *custom* security group the task asks for. `AWS_REGION` is pinned to `us-east-1` because the KodeKloud lab always runs there. Resource **IDs** (VPC ID, subnet IDs, AMI, etc.) are discovered in the steps below because they differ on every lab run.

### 🔎 Step 1: Discover the VPC, private subnet, private instance, and bucket

```bash
VPC_ID=$(aws ec2 describe-vpcs --region "$AWS_REGION" \
  --filters "Name=tag:Name,Values=$VPC_NAME" \
  --query "Vpcs[0].VpcId" --output text)

VPC_CIDR=$(aws ec2 describe-vpcs --region "$AWS_REGION" --vpc-ids "$VPC_ID" \
  --query "Vpcs[0].CidrBlock" --output text)

PRIVATE_SUBNET_ID=$(aws ec2 describe-subnets --region "$AWS_REGION" \
  --filters "Name=vpc-id,Values=$VPC_ID" "Name=tag:Name,Values=$PRIVATE_SUBNET_NAME" \
  --query "Subnets[0].SubnetId" --output text)

PRIVATE_SUBNET_CIDR=$(aws ec2 describe-subnets --region "$AWS_REGION" --subnet-ids "$PRIVATE_SUBNET_ID" \
  --query "Subnets[0].CidrBlock" --output text)

PRIVATE_AZ=$(aws ec2 describe-subnets --region "$AWS_REGION" --subnet-ids "$PRIVATE_SUBNET_ID" \
  --query "Subnets[0].AvailabilityZone" --output text)

PRIVATE_INSTANCE_ID=$(aws ec2 describe-instances --region "$AWS_REGION" \
  --filters "Name=tag:Name,Values=$PRIVATE_INSTANCE_NAME" "Name=instance-state-name,Values=running" \
  --query "Reservations[0].Instances[0].InstanceId" --output text)

aws s3api head-bucket --region "$AWS_REGION" --bucket "$S3_BUCKET"
```

Real values from the lab run:

```
VPC_ID=vpc-0bddffd95980a2ec9  VPC_CIDR=10.1.0.0/16
PRIVATE_SUBNET_ID=subnet-03d38b4f4a45ef65b  PRIVATE_SUBNET_CIDR=10.1.1.0/24  PRIVATE_AZ=us-east-1a
PRIVATE_INSTANCE_ID=i-0a504feefb440c708
```

> **Why:** We must never recreate the resources the challenge says already exist, so we look them up by their *Name* tag first. `describe-vpcs` and `describe-subnets` are read-only queries; `--filters "Name=tag:Name,Values=..."` matches a resource by its Name tag, and `--vpc-ids`/`--subnet-ids` look one up directly by ID. `--query` uses **JMESPath** (the CLI's built-in filter language) to pull one field, and `--output text` returns it bare so it stores cleanly in a shell variable. We capture the **VPC CIDR** (the VPC's IP range) to choose a non-overlapping public-subnet range, the private subnet's **CIDR** (needed for the NAT masquerade rule and the security group), and its **Availability Zone** (an isolated data center location) so the public subnet lands in the same AZ. `describe-instances` finds the running private instance; filtering on `instance-state-name=running` skips any terminated leftovers. `aws s3api head-bucket` simply confirms the target bucket exists and is reachable before we go further.

### 🧩 Step 2: Create the public subnet

The VPC is `10.1.0.0/16` and its only existing subnet is the private `10.1.1.0/24`, so `10.1.0.0/24` is free and non-overlapping — that's what we give the public subnet.

```bash
PUBLIC_SUBNET_CIDR="10.1.0.0/24"

PUBLIC_SUBNET_ID=$(aws ec2 create-subnet --region "$AWS_REGION" \
  --vpc-id "$VPC_ID" \
  --cidr-block "$PUBLIC_SUBNET_CIDR" \
  --availability-zone "$PRIVATE_AZ" \
  --query "Subnet.SubnetId" --output text)

aws ec2 create-tags --region "$AWS_REGION" \
  --resources "$PUBLIC_SUBNET_ID" --tags Key=Name,Value="$PUBLIC_SUBNET_NAME"

aws ec2 modify-subnet-attribute --region "$AWS_REGION" \
  --subnet-id "$PUBLIC_SUBNET_ID" --map-public-ip-on-launch
```

Real value from the lab run: `PUBLIC_SUBNET_ID=subnet-0c8a7eaccb801337e`.

> **Why:** A **subnet** is a slice of the VPC's IP range tied to one Availability Zone. `create-subnet` carves out our new range with `--cidr-block`; `--vpc-id` places it in the existing VPC and `--availability-zone` puts it in the same AZ as the private subnet. We add the Name tag in a separate `create-tags` call (rather than inline `--tag-specifications`) to keep each command simple. `modify-subnet-attribute --map-public-ip-on-launch` makes the subnet automatically give a **public IP** to instances launched into it — the NAT instance needs one so the Internet Gateway can reach it, which is what turns this into a true *public* subnet.

### 🚪 Step 3: Attach an Internet Gateway to the VPC

First we check whether the VPC already has an Internet Gateway attached.

```bash
aws ec2 describe-internet-gateways --region "$AWS_REGION" \
  --filters "Name=attachment.vpc-id,Values=$VPC_ID" \
  --query "InternetGateways[0].InternetGatewayId" --output text
```

This returned `None` — a private VPC ships without one — so we create an Internet Gateway and attach it:

```bash
IGW_ID=$(aws ec2 create-internet-gateway --region "$AWS_REGION" \
  --query "InternetGateway.InternetGatewayId" --output text)

aws ec2 create-tags --region "$AWS_REGION" \
  --resources "$IGW_ID" --tags Key=Name,Value=devops-nat-igw

aws ec2 attach-internet-gateway --region "$AWS_REGION" \
  --internet-gateway-id "$IGW_ID" --vpc-id "$VPC_ID"
```

Real value from the lab run: `IGW_ID=igw-06cc494c6e632c4b7`.

> **Why:** An **Internet Gateway (IGW)** is the doorway between a VPC and the internet — the NAT instance can only reach the internet if one is attached to the VPC. `describe-internet-gateways` filtered by `attachment.vpc-id` asks "is a gateway already attached to this VPC?"; it came back `None`, so `create-internet-gateway` makes a new one and `attach-internet-gateway` plugs it into the VPC (`--internet-gateway-id` says which gateway, `--vpc-id` which VPC). The IGW alone doesn't route anything yet — the route tables in the next steps decide what actually uses it.

### 🧭 Step 4: Create a public route table and route it to the Internet Gateway

```bash
PUBLIC_RT_ID=$(aws ec2 create-route-table --region "$AWS_REGION" \
  --vpc-id "$VPC_ID" \
  --query "RouteTable.RouteTableId" --output text)

aws ec2 create-tags --region "$AWS_REGION" \
  --resources "$PUBLIC_RT_ID" --tags Key=Name,Value=devops-nat-public-rt

aws ec2 create-route --region "$AWS_REGION" \
  --route-table-id "$PUBLIC_RT_ID" \
  --destination-cidr-block 0.0.0.0/0 --gateway-id "$IGW_ID"

aws ec2 associate-route-table --region "$AWS_REGION" \
  --route-table-id "$PUBLIC_RT_ID" --subnet-id "$PUBLIC_SUBNET_ID"
```

Real value from the lab run: `PUBLIC_RT_ID=rtb-0e4de2c12ec4cbf8c`.

> **Why:** A **route table** is a set of rules deciding where network traffic goes. `create-route-table` makes a fresh one for our public subnet so we don't disturb the VPC's main table. `create-route` adds the rule that makes the subnet "public": `--destination-cidr-block 0.0.0.0/0` means "any internet address" and `--gateway-id` sends that traffic to the Internet Gateway. `associate-route-table` binds this table to the public subnet (`--subnet-id`), so the NAT instance launched there uses these rules. Without this association the subnet would fall back to the main table and have no path to the IGW.

### 🛡️ Step 5: Create a custom security group for the NAT instance

```bash
NAT_SG_ID=$(aws ec2 create-security-group --region "$AWS_REGION" \
  --group-name "$NAT_SG_NAME" \
  --description "NAT instance for private subnet internet access" \
  --vpc-id "$VPC_ID" \
  --query "GroupId" --output text)

aws ec2 create-tags --region "$AWS_REGION" \
  --resources "$NAT_SG_ID" --tags Key=Name,Value="$NAT_SG_NAME"

aws ec2 authorize-security-group-ingress --region "$AWS_REGION" \
  --group-id "$NAT_SG_ID" --protocol -1 --cidr "$PRIVATE_SUBNET_CIDR"

aws ec2 authorize-security-group-ingress --region "$AWS_REGION" \
  --group-id "$NAT_SG_ID" --protocol tcp --port 22 --cidr 0.0.0.0/0
```

Real value from the lab run: `NAT_SG_ID=sg-08126a5ff0ff256b3`.

> **Why:** A **security group** is a virtual firewall attached to an instance. The challenge requires a *custom* one for the NAT instance, so `create-security-group` makes it (`--group-name`, `--description`, and `--vpc-id` are all required). The first `authorize-security-group-ingress` rule is the important one for NAT: `--protocol -1` means **all protocols**, and `--cidr "$PRIVATE_SUBNET_CIDR"` (`10.1.1.0/24`) lets the private instance's traffic **into** the NAT instance so it can be forwarded. The second rule opens TCP port `22` (SSH) so the instance can be reached for administration. A security group is **stateful**, so replies to allowed inbound traffic and all outbound traffic (allowed by the default egress rule) flow back automatically — no extra egress rule is needed for the masqueraded internet traffic.

### 🐧 Step 6: Launch the NAT instance with NAT configuration in user data

We first resolve the latest Amazon Linux 2023 AMI ID from a public SSM parameter, then write the user-data script that turns a plain instance into a NAT instance, and launch it.

```bash
AMI_ID=$(aws ssm get-parameter --region "$AWS_REGION" \
  --name /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64 \
  --query "Parameter.Value" --output text)
```

```bash
cat > nat-userdata.sh <<'EOF'
#!/bin/bash
dnf install -y iptables-services
systemctl enable --now iptables
echo "net.ipv4.ip_forward = 1" > /etc/sysctl.d/99-devops-nat.conf
sysctl --system
PUBLIC_INTERFACE=$(ip route show default | awk 'NR==1 {print $5}')
iptables -t nat -A POSTROUTING -s 10.1.1.0/24 -o "$PUBLIC_INTERFACE" -j MASQUERADE
iptables -A FORWARD -s 10.1.1.0/24 -o "$PUBLIC_INTERFACE" -j ACCEPT
iptables -A FORWARD -d 10.1.1.0/24 -i "$PUBLIC_INTERFACE" -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
service iptables save
EOF
```

```bash
NAT_INSTANCE_ID=$(aws ec2 run-instances --region "$AWS_REGION" \
  --image-id "$AMI_ID" \
  --instance-type t2.micro \
  --subnet-id "$PUBLIC_SUBNET_ID" \
  --security-group-ids "$NAT_SG_ID" \
  --associate-public-ip-address \
  --user-data file://nat-userdata.sh \
  --count 1 \
  --query "Instances[0].InstanceId" --output text)

aws ec2 create-tags --region "$AWS_REGION" \
  --resources "$NAT_INSTANCE_ID" --tags Key=Name,Value="$NAT_INSTANCE_NAME"
```

Real values from the lab run: `AMI_ID=ami-0f303bae6b670e0ed`, `NAT_INSTANCE_ID=i-0fce6cbbd4d412b55`.

> **Why:** `aws ssm get-parameter` reads a **public Systems Manager parameter** that AWS keeps pointing at the newest Amazon Linux 2023 AMI, so we never hardcode an AMI ID that ages out. **User data** is a script AWS runs automatically, as root, on the instance's *first* boot — the natural place to configure NAT so it's ready the moment the instance is up. Line by line, the script: installs `iptables-services` with `dnf` (the challenge reminds us `iptables` isn't preinstalled on AL2023) and enables the service so the rules persist across reboots; writes `net.ipv4.ip_forward = 1` and applies it with `sysctl --system`, which turns on **IP forwarding** (the kernel will now pass packets between interfaces — off by default); finds the instance's primary network interface (`enX0` on AL2023) from the default route; and adds the key rule `iptables -t nat -A POSTROUTING ... -j MASQUERADE`, which rewrites the source address of packets coming from the private subnet (`10.1.1.0/24`) to the NAT instance's own IP so replies can find their way back — the essence of NAT. The two `FORWARD` rules explicitly allow the private subnet's outbound packets and the established return traffic. `service iptables save` writes the live rules to `/etc/sysconfig/iptables` so the enabled service restores them on reboot. Then `run-instances` launches it: `--image-id` (the AL2023 AMI), `--instance-type t2.micro` (a small, free-tier size), `--subnet-id` (the public subnet), `--security-group-ids` (our custom firewall), `--associate-public-ip-address` (so the IGW can reach it), `--user-data file://...` (feeds the script from disk), and `--count 1` (one instance). A final `create-tags` gives it the required Name.

### 🚦 Step 7: Disable the source/destination check

```bash
aws ec2 modify-instance-attribute --region "$AWS_REGION" \
  --instance-id "$NAT_INSTANCE_ID" --no-source-dest-check

aws ec2 wait instance-running --region "$AWS_REGION" --instance-ids "$NAT_INSTANCE_ID"

aws ec2 wait instance-status-ok --region "$AWS_REGION" --instance-ids "$NAT_INSTANCE_ID"
```

Real value from the lab run: `NAT_PUBLIC_IP=32.197.213.219`.

> **Why:** By default every EC2 instance performs a **source/destination check** — it silently drops any packet whose source or destination address isn't the instance itself. A NAT instance's entire job is to relay traffic *on behalf of other instances*, so this check must be turned off or nothing forwards. `modify-instance-attribute --no-source-dest-check` disables it. `wait instance-running` blocks until the instance reaches the `running` state, and `wait instance-status-ok` blocks until its status checks pass — together they ensure the instance has finished booting (and its user data has run) before we rely on it.

### 🧭 Step 8: Route the private subnet's internet traffic through the NAT instance

```bash
PRIVATE_RT_ID=$(aws ec2 describe-route-tables --region "$AWS_REGION" \
  --filters "Name=association.subnet-id,Values=$PRIVATE_SUBNET_ID" \
  --query "RouteTables[0].RouteTableId" --output text)

aws ec2 create-route --region "$AWS_REGION" \
  --route-table-id "$PRIVATE_RT_ID" \
  --destination-cidr-block 0.0.0.0/0 --instance-id "$NAT_INSTANCE_ID"
```

Real value from the lab run: `PRIVATE_RT_ID=rtb-01fca16b321efecfe`.

> **Why:** This is the step that actually sends the private instance's internet traffic to the NAT. `describe-route-tables` filtered by `association.subnet-id` finds the route table bound to the private subnet. `create-route` then adds a default route (`--destination-cidr-block 0.0.0.0/0`) whose target is the NAT instance — `--instance-id` tells the route to resolve to that instance's primary network interface. From now on, any packet the private instance sends toward the internet is handed to the NAT instance, which masquerades it and forwards it through the Internet Gateway.

### ✅ Step 9: Verify

The private instance's cron job uploads `devops-test.txt` every minute, and it only succeeds once the NAT path works — so the file appearing in the bucket is end-to-end proof. Give the cron a minute, then list the bucket.

```bash
aws ec2 describe-instances --region "$AWS_REGION" --instance-ids "$NAT_INSTANCE_ID" \
  --query "Reservations[0].Instances[0].{Name:Tags[?Key=='Name']|[0].Value,Id:InstanceId,State:State.Name,PubIP:PublicIpAddress,SrcDestCheck:SourceDestCheck,Subnet:SubnetId}" \
  --output table

aws ec2 describe-route-tables --region "$AWS_REGION" --route-table-ids "$PRIVATE_RT_ID" \
  --query "RouteTables[0].Routes[].{Destination:DestinationCidrBlock,Gateway:GatewayId,Instance:InstanceId,State:State}" \
  --output table

aws s3 ls "s3://$S3_BUCKET/"
```

The NAT instance shows `SrcDestCheck: False`, the private route table sends `0.0.0.0/0` to the NAT instance, and the test file is in the bucket:

```
----------------------------------------------
|              DescribeInstances             |
+---------------+----------------------------+
|  Id           |  i-0fce6cbbd4d412b55       |
|  Name         |  devops-nat-instance   |
|  PubIP        |  32.197.213.219            |
|  SrcDestCheck |  False                     |
|  State        |  running                   |
|  Subnet       |  subnet-0c8a7eaccb801337e  |
+---------------+----------------------------+
-------------------------------------------------------------
|                    DescribeRouteTables                    |
+--------------+----------+-----------------------+---------+
|  Destination | Gateway  |       Instance        |  State  |
+--------------+----------+-----------------------+---------+
|  10.1.0.0/16 |  local   |  None                 |  active |
|  0.0.0.0/0   |  None    |  i-0fce6cbbd4d412b55  |  active |
+--------------+----------+-----------------------+---------+

2026-07-21 18:56:34         17 devops-test.txt
```

> **Why:** `describe-instances` with a `--query` object projection prints just the fields that prove the NAT instance is correct — most importantly `SourceDestCheck: False`. `describe-route-tables` confirms the private subnet's default route targets the NAT instance. `aws s3 ls` lists the bucket's contents; the presence of `devops-test.txt` (uploaded by the private instance's cron) confirms the private instance reached S3 over the internet **through the NAT instance** — the whole objective. If the listing is empty at first, wait a moment and re-run it, since the cron runs once a minute.

## Best Practices

- **Disable the source/destination check.** A NAT instance forwards traffic that isn't addressed to itself, so leaving the default check enabled silently drops every forwarded packet — the single most common NAT-instance mistake.
- **Put NAT configuration in user data.** Baking `iptables`/`ip_forward` setup into the launch-time user-data script makes the instance a working NAT the moment it boots, and re-creating it reproduces the setup exactly. Remember user data runs only on first boot.
- **Persist the firewall rules.** `service iptables save` after enabling `iptables-services` writes the rules to disk so they survive a reboot; skipping it means NAT breaks the next time the instance restarts.
- **Scope the security group to the private subnet.** Allowing inbound traffic only from the private subnet CIDR (not `0.0.0.0/0`) keeps the NAT instance from relaying traffic for anyone else.
- **Prefer a NAT Gateway for production.** A NAT instance is cheaper and fine for labs, but a managed NAT Gateway is highly available, scales automatically, and needs no patching — choose it when reliability matters more than cost.

### 📚 Official Documentation

- [Connect to the internet using a NAT instance](https://docs.aws.amazon.com/vpc/latest/userguide/work-with-nat-instances.html)
- [Compare NAT gateways and NAT instances](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-nat-comparison.html)
- [Configure route tables](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Route_Tables.html)
- [Run commands on your Linux instance at launch (user data)](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/user-data.html)
- [Find a Systems Manager public parameter for the latest AMI](https://docs.aws.amazon.com/systems-manager/latest/userguide/parameter-store-public-parameters-ami.html)
- [modify-instance-attribute — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/ec2/modify-instance-attribute.html)
