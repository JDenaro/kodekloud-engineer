# Day 45: Configure NAT Gateway for Internet Access in a Private VPC

The Nautilus DevOps team is tasked with enabling internet access for an EC2 instance running in a private subnet. This instance should be able to upload a test file to a public S3 bucket once it can access the internet. To achieve this, the team must set up a NAT Gateway in a public subnet within the same VPC.

1. A VPC named `devops-priv-vpc` and a private subnet `devops-priv-subnet` have already been created.
2. An EC2 instance named `devops-priv-ec2` is already running in the private subnet.
3. The EC2 instance is configured with a cron job that uploads a test file to a bucket `devops-nat-142474778` once internet is accessible.

## Specific Requirements:

- Create a public subnet named `devops-pub-subnet` in the same VPC.
- Create an Internet Gateway and attach it to the VPC.
- Create a route table `devops-pub-rt` and associate it with the public subnet.
- Allocate an Elastic IP and create a NAT Gateway named `devops-natgw`.
- Update the private route table to route `0.0.0.0/0` traffic via the NAT Gateway.
- Once complete, verify that the EC2 instance can reach the internet by confirming the presence of the test file in the S3 bucket `devops-nat-142474778`. After completing all the configuration, please wait a few minutes for the test file to appear in the bucket, as it may take 2–3 minutes.

## Solution

An instance in a **private subnet** has no public IP, so it can't talk to the internet directly. The fix is a **NAT Gateway** (Network Address Translation): it lives in a **public subnet**, holds a public Elastic IP, and forwards outbound traffic from private instances to the internet while blocking unsolicited inbound connections. The traffic path we build is: private instance → private route table → NAT Gateway (public subnet) → Internet Gateway → internet. Both a public and a private route table are needed — the public one points the internet at the Internet Gateway, the private one points the internet at the NAT Gateway.

> **Gotcha:** the private subnet must be **explicitly associated with a dedicated private route table** (`devops-priv-rt`). Relying on the VPC's implicit **main** route table fails validation with `Private subnet is not associated with the private route table`. Always create a private route table and associate it with the private subnet.

### 📦 Variables

Define the values that come straight from the challenge description before running any step. Resource IDs that must be discovered (the VPC ID, private subnet ID, and Availability Zone) are captured into variables later, in Step 1, once their lookup commands return.

```bash
AWS_REGION="us-east-1"
PUB_SUBNET_CIDR="10.1.2.0/24"
BUCKET="devops-nat-142474778"
```

> **Why:** `AWS_REGION` is hardcoded to `us-east-1` because that is where the KodeKloud lab always runs. `BUCKET` is the target bucket named in the challenge. The VPC's CIDR is `10.1.0.0/16` and the existing private subnet is `10.1.1.0/24`, so we pick `10.1.2.0/24` — the next `/24` block immediately after it — for the public subnet, keeping the addressing tidy and non-overlapping. The VPC ID, private subnet ID, and its Availability Zone are not known from the description and change per lab run, so we look them up and assign them in Step 1 rather than hardcoding them here.

### 🔍 Step 1: Look up the existing VPC and private subnet

```bash
VPC_ID=$(aws ec2 describe-vpcs \
  --filters "Name=tag:Name,Values=devops-priv-vpc" \
  --query "Vpcs[0].VpcId" \
  --output text \
  --region "$AWS_REGION")
echo "VPC: $VPC_ID"
```

```bash
PRIV_SUBNET_ID=$(aws ec2 describe-subnets \
  --filters "Name=tag:Name,Values=devops-priv-subnet" \
  --query "Subnets[0].SubnetId" \
  --output text \
  --region "$AWS_REGION")
echo "Private subnet: $PRIV_SUBNET_ID"
```

```bash
AZ=$(aws ec2 describe-subnets \
  --subnet-ids "$PRIV_SUBNET_ID" \
  --query "Subnets[0].AvailabilityZone" \
  --output text \
  --region "$AWS_REGION")
echo "AZ: $AZ"
```

Output:

```
VPC: vpc-03c99806d016ca67a
Private subnet: subnet-0c2b322a16bb4ced4
AZ: us-east-1a
```

> **Why:** `describe-vpcs` and `describe-subnets` are read-only queries that list existing resources. The `--filters "Name=tag:Name,Values=..."` option narrows results to the resource whose `Name` tag matches, which is how we find lab resources by their friendly name. `--query` uses JMESPath to pull out a single field and `--output text` returns it bare, so we can capture it directly into a shell variable (`VPC_ID`, `PRIV_SUBNET_ID`, `AZ`) for use in every later step. We read the private subnet's `AvailabilityZone` so we can place the public subnet in the same AZ as the instance that will use the NAT Gateway, avoiding cross-AZ traffic. These IDs differ on every lab run, which is why they're discovered here rather than hardcoded in the Variables section.

### 🌐 Step 2: Create the public subnet

```bash
PUB_SUBNET_ID=$(aws ec2 create-subnet \
  --vpc-id "$VPC_ID" \
  --cidr-block "$PUB_SUBNET_CIDR" \
  --availability-zone "$AZ" \
  --query "Subnet.SubnetId" \
  --output text \
  --region "$AWS_REGION")
echo "Public subnet: $PUB_SUBNET_ID"
```

```bash
aws ec2 create-tags \
  --resources "$PUB_SUBNET_ID" \
  --tags Key=Name,Value=devops-pub-subnet \
  --region "$AWS_REGION"
```

Output:

```
Public subnet: subnet-083b12f78b3803f3a
```

> **Why:** A **subnet** is a range of IP addresses carved out of the VPC's CIDR, tied to a single Availability Zone. `create-subnet` creates it: `--vpc-id` places it inside our VPC, `--cidr-block` gives it the `10.1.2.0/24` range, and `--availability-zone` pins it to `us-east-1a`. `--query "Subnet.SubnetId" --output text` returns just the new subnet's ID so we can capture it in a shell variable. The subnet is created untagged, so we run `create-tags` separately: `--resources` is the ID to tag and `--tags Key=Name,Value=devops-pub-subnet` sets the `Name` tag, which is what the console displays and what the challenge requires. Splitting creation and tagging into two small commands keeps each step easy to follow.

### 🚪 Step 3: Create and attach the Internet Gateway

First we must find whether our VPC already has an Internet Gateway attached — the lab may have pre-provisioned one, and there's no point creating a duplicate.

```bash
aws ec2 describe-internet-gateways \
  --filters "Name=attachment.vpc-id,Values=$VPC_ID" \
  --query "InternetGateways[].{IgwId:InternetGatewayId,State:Attachments[0].State}" \
  --output table \
  --region "$AWS_REGION"
```

No Internet Gateway is attached to this VPC, and there is no available Internet Gateway to attach, so we must create one.

```bash
IGW_ID=$(aws ec2 create-internet-gateway \
  --query "InternetGateway.InternetGatewayId" \
  --output text \
  --region "$AWS_REGION")
echo "IGW: $IGW_ID"
```

```bash
aws ec2 create-tags \
  --resources "$IGW_ID" \
  --tags Key=Name,Value=devops-igw \
  --region "$AWS_REGION"
```

```bash
aws ec2 attach-internet-gateway \
  --internet-gateway-id "$IGW_ID" \
  --vpc-id "$VPC_ID" \
  --region "$AWS_REGION"
```

> **Why:** An **Internet Gateway (IGW)** is the VPC component that allows communication between the VPC and the public internet; a VPC can have at most one attached. We check first with `describe-internet-gateways`, filtering on `attachment.vpc-id` to see whether one is already bound to our VPC. Since none exists, `create-internet-gateway` creates a new, unattached IGW, `create-tags` names it `devops-igw`, and `attach-internet-gateway` binds it to the VPC via `--internet-gateway-id` and `--vpc-id`. Until an IGW is attached, nothing in the VPC can reach the internet.

### 🗺️ Step 4: Create the public route table and route to the IGW

```bash
PUB_RT_ID=$(aws ec2 create-route-table \
  --vpc-id "$VPC_ID" \
  --query "RouteTable.RouteTableId" \
  --output text \
  --region "$AWS_REGION")
echo "Public RT: $PUB_RT_ID"
```

```bash
aws ec2 create-tags \
  --resources "$PUB_RT_ID" \
  --tags Key=Name,Value=devops-pub-rt \
  --region "$AWS_REGION"
```

```bash
aws ec2 create-route \
  --route-table-id "$PUB_RT_ID" \
  --destination-cidr-block 0.0.0.0/0 \
  --gateway-id "$IGW_ID" \
  --region "$AWS_REGION"
```

```bash
aws ec2 associate-route-table \
  --route-table-id "$PUB_RT_ID" \
  --subnet-id "$PUB_SUBNET_ID" \
  --region "$AWS_REGION"
```

Output:

```
Public RT: rtb-02d9ac9c19c1ad618

{
    "Return": true
}

{
    "AssociationId": "rtbassoc-0437c8eb328a72292",
    "AssociationState": {
        "State": "associated"
    }
}
```

> **Why:** A **route table** holds rules ("routes") that decide where network traffic is sent based on its destination. `create-route-table` makes an empty one in the VPC, and `create-tags` names it `devops-pub-rt`. `create-route` adds a rule: `--destination-cidr-block 0.0.0.0/0` means "all IPv4 traffic" (any address not matched by a more specific route), and `--gateway-id "$IGW_ID"` sends that traffic to the Internet Gateway — this is what makes the subnet "public". `associate-route-table` binds the table to the public subnet with `--subnet-id`, so instances launched there use these routes. `Return: true` and `State: associated` confirm the route and association succeeded.

### 📍 Step 5: Allocate an Elastic IP and create the NAT Gateway

```bash
EIP_ALLOC_ID=$(aws ec2 allocate-address \
  --domain vpc \
  --query "AllocationId" \
  --output text \
  --region "$AWS_REGION")
echo "EIP Allocation: $EIP_ALLOC_ID"
```

```bash
aws ec2 create-tags \
  --resources "$EIP_ALLOC_ID" \
  --tags Key=Name,Value=devops-eip \
  --region "$AWS_REGION"
```

```bash
NATGW_ID=$(aws ec2 create-nat-gateway \
  --subnet-id "$PUB_SUBNET_ID" \
  --allocation-id "$EIP_ALLOC_ID" \
  --query "NatGateway.NatGatewayId" \
  --output text \
  --region "$AWS_REGION")
echo "NAT GW: $NATGW_ID"
```

```bash
aws ec2 create-tags \
  --resources "$NATGW_ID" \
  --tags Key=Name,Value=devops-natgw \
  --region "$AWS_REGION"
```

Output:

```
EIP Allocation: eipalloc-0144a849f5ffa2b7a
NAT GW: nat-0ba6872c8b04af1ac
```

> **Why:** An **Elastic IP (EIP)** is a static public IPv4 address you allocate to your account. `allocate-address` reserves one; `--domain vpc` marks it for use inside a VPC and returns an `AllocationId` (the handle used to reference the EIP). A **public NAT Gateway must have an Elastic IP** — that address becomes the source IP the internet sees for all outbound traffic from the private subnet. `create-nat-gateway` builds the gateway: `--subnet-id "$PUB_SUBNET_ID"` places it in the public subnet (so its own traffic can reach the IGW), and `--allocation-id "$EIP_ALLOC_ID"` attaches the Elastic IP. We name both resources with `create-tags` (`devops-eip` and `devops-natgw`).

### ⏳ Step 6: Wait for the NAT Gateway to become available

```bash
aws ec2 wait nat-gateway-available \
  --nat-gateway-ids "$NATGW_ID" \
  --region "$AWS_REGION"
echo "NAT Gateway is available"
```

> **Why:** A NAT Gateway starts in the `pending` state and takes a minute or two to provision before it can pass traffic. `wait nat-gateway-available` is a built-in poller that blocks until the gateway reaches the `available` state, so the next step (pointing routes at it) doesn't run against a gateway that isn't ready yet.

### 🔁 Step 7: Create the private route table and route through the NAT Gateway

The private subnet is on the VPC's implicit **main** route table, which the lab does not accept. We create a dedicated private route table, add the NAT route, and associate it explicitly with the private subnet.

```bash
PRIV_RT_ID=$(aws ec2 create-route-table \
  --vpc-id "$VPC_ID" \
  --query "RouteTable.RouteTableId" \
  --output text \
  --region "$AWS_REGION")
echo "Private RT: $PRIV_RT_ID"
```

```bash
aws ec2 create-tags \
  --resources "$PRIV_RT_ID" \
  --tags Key=Name,Value=devops-priv-rt \
  --region "$AWS_REGION"
```

```bash
aws ec2 create-route \
  --route-table-id "$PRIV_RT_ID" \
  --destination-cidr-block 0.0.0.0/0 \
  --nat-gateway-id "$NATGW_ID" \
  --region "$AWS_REGION"
```

```bash
aws ec2 associate-route-table \
  --route-table-id "$PRIV_RT_ID" \
  --subnet-id "$PRIV_SUBNET_ID" \
  --region "$AWS_REGION"
```

Output:

```
Private RT: rtb-0c4f1420566c31261

{
    "Return": true
}

{
    "AssociationId": "rtbassoc-0a1b2c3d4e5f60718",
    "AssociationState": {
        "State": "associated"
    }
}
```

> **Why:** A subnet with no explicit route table falls back to the VPC's **main** route table, but the challenge validator requires the private subnet to be associated with its own route table — otherwise it fails with `Private subnet is not associated with the private route table`. So we `create-route-table` for a fresh one, name it `devops-priv-rt` with `create-tags`, and add the key route with `create-route`: `--destination-cidr-block 0.0.0.0/0` (all internet-bound traffic) with `--nat-gateway-id "$NATGW_ID"` forwards that traffic to the NAT Gateway. Finally `associate-route-table` binds this table to the private subnet via `--subnet-id`, replacing the implicit main-table association. This explicit association plus the NAT route is what gives the private instance outbound internet access. `Return: true` and `State: associated` confirm success.

### ✅ Step 8: Verify

Wait 2–3 minutes for the instance's cron job to detect internet access and upload the test file, then list the bucket:

```bash
aws s3 ls s3://$BUCKET/ --region "$AWS_REGION"
```

Output:

```
2026-07-17 00:19:21          0 devops-test.txt
```

The presence of `devops-test.txt` confirms the private instance reached S3 over the internet through the NAT Gateway — the challenge is complete.

> **Why:** `aws s3 ls s3://<bucket>/` lists the objects currently in the bucket. Because the private instance's cron job only uploads once it detects internet access, the listing may be empty on the first try — if so, wait a moment and re-run the same command, since the cron runs on an interval. Once `devops-test.txt` appears, the outbound path (private instance → NAT Gateway → Internet Gateway → S3) is proven end to end.

## Best Practices

- **Put the NAT Gateway in the same AZ as the instances it serves.** Traffic that crosses Availability Zones incurs data-transfer charges; keeping the NAT Gateway in `us-east-1a` alongside the private subnet avoids that cost and reduces latency.
- **Check for existing resources before creating them.** A VPC can only have one Internet Gateway attached; querying first (as in Step 3) prevents duplicate-resource errors and wasted resources when the lab pre-provisions infrastructure.
- **Keep public and private route tables separate.** The public route table points `0.0.0.0/0` at the Internet Gateway; the private one points it at the NAT Gateway. Mixing them would either expose private instances or break the NAT path.
- **Associate the private subnet with a dedicated route table, not the main table.** Relying on the VPC's implicit main route table leaves the subnet unassociated from any named table and fails validation. Create `devops-priv-rt` and explicitly associate it.
- **Use `aws ec2 wait` instead of guessing timings.** Waiters poll AWS for the real resource state, so dependent steps only run once the NAT Gateway is truly `available`.
- **For production, deploy one NAT Gateway per AZ.** A single NAT Gateway is highly available only within its own AZ; multi-AZ workloads should run a NAT Gateway in each zone for resilience.

### 📚 Official Documentation

- [NAT gateways — Amazon VPC User Guide](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-nat-gateway.html)
- [Example routing options (route to a NAT device)](https://docs.aws.amazon.com/vpc/latest/userguide/route-table-options.html)
- [create-nat-gateway — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/ec2/create-nat-gateway.html)
- [Connect to the internet using an internet gateway](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Internet_Gateway.html)
- [Elastic IP addresses](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/elastic-ip-addresses-eip.html)
