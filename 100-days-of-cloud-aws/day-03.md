# Day 03: Create Subnet

The Nautilus DevOps team is strategizing the migration of a portion of their infrastructure to the AWS cloud. Recognizing the scale of this undertaking, they have opted to approach the migration in incremental steps rather than as a single massive transition.

## Specific Requirements:

For this task, create one subnet named `xfusion-subnet` under default VPC.

## Solution

A *subnet* is a range of IP addresses carved out of a VPC's CIDR block, tied to a single Availability Zone, where resources like EC2 instances get their private IPs. The only real gotcha here is picking a CIDR that **doesn't overlap** an existing subnet: the default VPC (`172.31.0.0/16`) already ships with one `/20` subnet per Availability Zone, so a new subnet must land in a free block. The script below discovers the used blocks and picks a free `/24` automatically.

### 📦 Variables

```bash
AWS_REGION="us-east-1"
SUBNET_NAME="xfusion-subnet"
```

### 🔎 Step 1: Find the default VPC and its CIDR

```bash
VPC_ID=$(aws ec2 describe-vpcs --region "$AWS_REGION" \
  --filters "Name=isDefault,Values=true" \
  --query "Vpcs[0].VpcId" --output text)

VPC_CIDR=$(aws ec2 describe-vpcs --region "$AWS_REGION" --vpc-ids "$VPC_ID" \
  --query "Vpcs[0].CidrBlock" --output text)

BASE="${VPC_CIDR%.*.*/*}"
```

Real values from the lab run: `VPC_ID=vpc-04f1cc61afc648068`, `CIDR=172.31.0.0/16`, `BASE=172.31`.

> **Why:** the subnet must live "under default VPC", so we locate it with the `isDefault=true` filter instead of hardcoding an ID. We also read its `CidrBlock` because a subnet's range must be **within** the VPC's range. The `BASE` shell expansion strips the last two octets and prefix (`172.31.0.0/16` → `172.31`) so we can build candidate subnet CIDRs on top of the VPC's network prefix.

### 🧮 Step 2: Discover used blocks and pick a free /24

```bash
aws ec2 describe-subnets --region "$AWS_REGION" \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --query "Subnets[].CidrBlock" --output text
```

The existing subnets came back as `172.31.0.0/20 172.31.16.0/20 172.31.32.0/20 172.31.48.0/20 172.31.64.0/20 172.31.80.0/20` — one `/20` per Availability Zone, aligned to multiples of 16 in the third octet (`0, 16, 32, 48, 64, 80`). Picking any third octet outside that occupied range (for example, starting from the top of the VPC's space) guarantees no overlap, so we set:

```bash
SUBNET_CIDR="${BASE}.240.0/24"
```

> **Why:** `describe-subnets` filtered by `vpc-id` lists the CIDRs already in use. Because the default subnets are `/20` blocks aligned to multiples of 16 in the third octet, any `/24` carved out of an untouched block (like `240`, far above the highest used block `80`) is guaranteed not to overlap. This avoids the `InvalidSubnet.Conflict` error you'd hit by blindly reusing an occupied range.

### 🧩 Step 3: Create the subnet and tag it

```bash
SUBNET_ID=$(aws ec2 create-subnet --region "$AWS_REGION" \
  --vpc-id "$VPC_ID" \
  --cidr-block "$SUBNET_CIDR" \
  --query "Subnet.SubnetId" --output text)

aws ec2 create-tags --region "$AWS_REGION" \
  --resources "$SUBNET_ID" --tags Key=Name,Value="$SUBNET_NAME"
```

Real value from the lab run: `SUBNET_ID=subnet-0c87dd8b037f2f603`.

> **Why:** `create-subnet` carves the chosen `--cidr-block` out of the VPC. We don't pass `--availability-zone`, so AWS assigns one automatically (it landed in `us-east-1e`). The *Name* tag is applied in a separate `create-tags` call to keep the creation command simple, giving the subnet its required name `xfusion-subnet`.

### ✅ Step 4: Verify

```bash
aws ec2 describe-subnets --region "$AWS_REGION" --subnet-ids "$SUBNET_ID" \
  --query "Subnets[0].{Id:SubnetId,Cidr:CidrBlock,Vpc:VpcId,AZ:AvailabilityZone,Name:Tags[?Key=='Name']|[0].Value}" \
  --output table
```

Success — the subnet exists under the default VPC with the correct name:

```
+------------+------------------+----------------------------+-----------------+-------------------------+
|     AZ     |      Cidr        |            Id              |      Name       |           Vpc           |
+------------+------------------+----------------------------+-----------------+-------------------------+
|  us-east-1e|  172.31.240.0/24 |  subnet-0c87dd8b037f2f603  |  xfusion-subnet |  vpc-04f1cc61afc648068  |
+------------+------------------+----------------------------+-----------------+-------------------------+
```

> **Why:** `describe-subnets` reads the subnet back so we confirm its CIDR, parent VPC, AZ, and the `Name` tag (pulled from the tag list with a JMESPath filter) before declaring the task complete.

## Best Practices

- **Never assume a CIDR is free.** The default VPC comes pre-populated with one subnet per AZ; always list existing CIDRs and pick a non-overlapping range to avoid `InvalidSubnet.Conflict`.
- **A subnet belongs to exactly one AZ.** For high availability you create one subnet per AZ; a single subnet (as here) lives in just one.
- **Right-size the CIDR.** A `/24` gives ~251 usable IPs (AWS reserves 5 per subnet). Size it to expected workload rather than always copying the `/20` defaults.
- **Tag on creation.** Naming the subnet immediately keeps the console and automation readable.

### 📚 Official Documentation

- [Subnets for your VPC](https://docs.aws.amazon.com/vpc/latest/userguide/configure-subnets.html)
- [create-subnet — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/ec2/create-subnet.html)
- [IP addressing for your VPCs and subnets](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-ip-addressing.html)
