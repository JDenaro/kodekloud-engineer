# Day 21: Setting Up an EC2 Instance with an Elastic IP for Application Hosting

The Nautilus DevOps Team has received a new request from the Development Team to set up a new EC2 instance. This instance will be used to host a new application that requires a stable IP address. To ensure that the instance has a consistent public IP, an Elastic IP address needs to be associated with it. The instance will be named `datacenter-ec2`, and the Elastic IP will be named `datacenter-eip`. This setup will help the Development Team to have a reliable and consistent access point for their application.

Create an EC2 instance named `datacenter-ec2` using any linux AMI like `ubuntu`, the Instance type must be `t2.micro` and associate an Elastic IP address with this instance, name it as `datacenter-eip`.

## Specific Requirements:

1. Create an EC2 instance named `datacenter-ec2` using any linux AMI like `ubuntu`, the Instance type must be `t2.micro` and associate an Elastic IP address with this instance, name it as `datacenter-eip`.

## Solution

An automatically assigned public IP can change when an instance is stopped and started. An Elastic IP is a static public IPv4 address allocated to the AWS account, so it can be associated with the application instance and later remapped if needed. The workflow discovers the required network resources and AMI, checks whether the named resources already exist, then creates and associates the resources required by this lab run.

### 📦 Variables

```bash
AWS_REGION="us-east-1"
INSTANCE_NAME="datacenter-ec2"
EIP_NAME="datacenter-eip"
INSTANCE_TYPE="t2.micro"
```

The AMI ID and network resource IDs are discovered during the workflow because they are supplied by the lab and can differ between runs.

### 🔎 Step 1: Discover the default VPC resources

```bash
VPC_ID=$(aws ec2 describe-vpcs \
  --region "$AWS_REGION" \
  --filters "Name=isDefault,Values=true" \
  --query "Vpcs[0].VpcId" \
  --output text)

SUBNET_ID=$(aws ec2 describe-subnets \
  --region "$AWS_REGION" \
  --filters "Name=vpc-id,Values=$VPC_ID" "Name=default-for-az,Values=true" \
  --query "Subnets[0].SubnetId" \
  --output text)

SECURITY_GROUP_ID=$(aws ec2 describe-security-groups \
  --region "$AWS_REGION" \
  --filters "Name=vpc-id,Values=$VPC_ID" "Name=group-name,Values=default" \
  --query "SecurityGroups[0].GroupId" \
  --output text)

echo "VPC_ID=$VPC_ID"
echo "SUBNET_ID=$SUBNET_ID"
echo "SECURITY_GROUP_ID=$SECURITY_GROUP_ID"
```

The lab provided the following network resources:

```text
VPC_ID=vpc-0e8e01a46fd380cf1
SUBNET_ID=subnet-088a3cf33ab0a7c4b
SECURITY_GROUP_ID=sg-0854abe8232f81092
```

> **Why:** `describe-vpcs` lists VPCs, and the `isDefault` filter selects the default VPC for the lab Region. `describe-subnets` lists subnets; `vpc-id` limits the result to that VPC and `default-for-az` selects a subnet configured as the default for an Availability Zone. `describe-security-groups` lists security groups; its `vpc-id` and `group-name` filters select the default security group in the same VPC. In each command, `--query` extracts only the needed identifier from the response and `--output text` returns a shell-friendly value that can be stored in a variable.

### 🐧 Step 2: Find a compatible Linux AMI

```bash
AMI_ID=$(aws ec2 describe-images \
  --region "$AWS_REGION" \
  --owners 099720109477 \
  --filters "Name=name,Values=ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*" "Name=state,Values=available" "Name=architecture,Values=x86_64" "Name=root-device-type,Values=ebs" "Name=virtualization-type,Values=hvm" \
  --query "sort_by(Images, &CreationDate)[-1].ImageId" \
  --output text)

echo "AMI_ID=$AMI_ID"
```

The selected AMI was:

```text
AMI_ID=ami-0d001f8052688dc45
```

> **Why:** `describe-images` searches AMIs (Amazon Machine Images) available in the current Region. `--owners` restricts the search to the account that publishes the Ubuntu images. The `name`, `state`, `architecture`, `root-device-type`, and `virtualization-type` filters ensure that the result is an available x86_64 Ubuntu image using an EBS root device and HVM virtualization. The JMESPath `sort_by` expression orders the matching images by `CreationDate`, and `[-1]` selects the newest image. `--query` extracts its `ImageId`, while `--output text` makes it easy to assign to `AMI_ID`.

### 🔎 Step 3: Check whether the EC2 instance exists

```bash
INSTANCE_ID=$(aws ec2 describe-instances \
  --region "$AWS_REGION" \
  --filters "Name=tag:Name,Values=$INSTANCE_NAME" "Name=instance-state-name,Values=pending,running,stopping,stopped" \
  --query "Reservations[0].Instances[0].InstanceId" \
  --output text)
```

The lookup returned no existing instance with the `datacenter-ec2` Name tag, so the instance was created in the next step:

```text
INSTANCE_ID=None
```

> **Why:** `describe-instances` lists EC2 instances and the `tag:Name` and state filters narrow the result to a usable instance with the required name. `--query` extracts the instance ID and `--output text` makes it easy to inspect or store. Checking first prevents an accidental duplicate when a suitable instance is already present.

### 🚀 Step 4: Create and tag the EC2 instance

```bash
INSTANCE_ID=$(aws ec2 run-instances \
  --region "$AWS_REGION" \
  --image-id "$AMI_ID" \
  --instance-type "$INSTANCE_TYPE" \
  --subnet-id "$SUBNET_ID" \
  --security-group-ids "$SECURITY_GROUP_ID" \
  --count 1 \
  --query "Instances[0].InstanceId" \
  --output text)

aws ec2 create-tags \
  --region "$AWS_REGION" \
  --resources "$INSTANCE_ID" \
  --tags "Key=Name,Value=$INSTANCE_NAME"

aws ec2 wait instance-running \
  --region "$AWS_REGION" \
  --instance-ids "$INSTANCE_ID"
```

The command returned:

```text
Created instance: i-0c15b88bb239d480c
```

> **Why:** `run-instances` launches one EC2 instance. `--image-id` selects the AMI, `--instance-type` sets the required `t2.micro` size, `--subnet-id` places the instance in the discovered subnet, `--security-group-ids` applies the discovered security group, and `--count 1` requests exactly one instance. The `--query` and `--output text` options capture its ID. `create-tags` applies the `Name` tag after creation; `--resources` identifies the instance and `--tags` supplies the key-value tag. `wait instance-running` polls the instance state, and `--instance-ids` tells the waiter which instance must reach `running` before the next step continues.

### 🔎 Step 5: Check whether the Elastic IP exists

```bash
ALLOCATION_ID=$(aws ec2 describe-addresses \
  --region "$AWS_REGION" \
  --filters "Name=tag:Name,Values=$EIP_NAME" \
  --query "Addresses[0].AllocationId" \
  --output text)
```

The lab had no existing Elastic IP named `datacenter-eip`, so a new address was allocated in the next step:

```text
ALLOCATION_ID=None
```

> **Why:** `describe-addresses` lists Elastic IP allocations, and the `Name` tag filter checks whether the requested address already exists. `--query` extracts its allocation ID when present, while `--output text` returns a shell-friendly value. This lookup lets us decide whether allocation is necessary before creating another public IPv4 address.

### 🌐 Step 6: Allocate, tag, and associate the Elastic IP

```bash
ALLOCATION_ID=$(aws ec2 allocate-address \
  --region "$AWS_REGION" \
  --domain vpc \
  --query "AllocationId" \
  --output text)

aws ec2 create-tags \
  --region "$AWS_REGION" \
  --resources "$ALLOCATION_ID" \
  --tags "Key=Name,Value=$EIP_NAME"

PUBLIC_IP=$(aws ec2 describe-addresses \
  --region "$AWS_REGION" \
  --allocation-ids "$ALLOCATION_ID" \
  --query "Addresses[0].PublicIp" \
  --output text)

ASSOCIATION_ID=$(aws ec2 associate-address \
  --region "$AWS_REGION" \
  --instance-id "$INSTANCE_ID" \
  --allocation-id "$ALLOCATION_ID" \
  --query "AssociationId" \
  --output text)

echo "Allocated Elastic IP: $ALLOCATION_ID"
echo "Associated $PUBLIC_IP with $INSTANCE_ID: $ASSOCIATION_ID"
```

The lab had no existing Elastic IP named `datacenter-eip`, so the script allocated a new one and associated it:

```text
Allocated Elastic IP: eipalloc-06fce9e8c27c9b319
Associated 107.23.82.205 with i-0c15b88bb239d480c: eipassoc-020b82f4390b41e14
```

> **Why:** `allocate-address` reserves a new public IPv4 address in the account; `--domain vpc` requests an address for a VPC. The returned `AllocationId` identifies the reservation, which is tagged with `create-tags` using `--resources` and `--tags`. A second `describe-addresses` call uses `--allocation-ids` to retrieve the actual `PublicIp`. Finally, `associate-address` binds that allocation to the instance: `--instance-id` identifies the target instance, `--allocation-id` identifies the Elastic IP reservation, and the query extracts the returned `AssociationId`.

### ✅ Step 7: Verify

```bash
aws ec2 describe-instances \
  --region "$AWS_REGION" \
  --instance-ids "$INSTANCE_ID" \
  --query "Reservations[0].Instances[0].{Name:Tags[?Key=='Name']|[0].Value,InstanceId:InstanceId,State:State.Name,InstanceType:InstanceType,PublicIp:PublicIpAddress}" \
  --output table

aws ec2 describe-addresses \
  --region "$AWS_REGION" \
  --allocation-ids "$ALLOCATION_ID" \
  --query "Addresses[].{Name:Tags[?Key=='Name']|[0].Value,PublicIp:PublicIp,AllocationId:AllocationId,AssociationId:AssociationId,InstanceId:InstanceId}" \
  --output table
```

The verification confirmed the requested instance type, name, running state, public IP, Elastic IP name, allocation, association, and instance relationship:

```text
-----------------------------------------
|           DescribeInstances           |
+---------------+-----------------------+
|  InstanceId   |  i-0c15b88bb239d480c  |
|  InstanceType |  t2.micro             |
|  Name         |  datacenter-ec2       |
|  PublicIp     |  107.23.82.205        |
|  State        |  running              |
+---------------+-----------------------+
-------------------------------------------------
|               DescribeAddresses               |
+----------------+------------------------------+
|  AllocationId  |  eipalloc-06fce9e8c27c9b319  |
|  AssociationId |  eipassoc-020b82f4390b41e14  |
|  InstanceId   |  i-0c15b88bb239d480c         |
|  Name         |  datacenter-eip              |
|  PublicIp     |  107.23.82.205               |
+----------------+------------------------------+
```

> **Why:** `describe-instances` reads the created instance, and `--instance-ids` limits the response to the known resource. Its `--query` selects the Name tag, instance ID, state, type, and public IP. `describe-addresses` reads the Elastic IP reservation identified by `--allocation-ids`; its query selects the tag, public IP, allocation ID, association ID, and attached instance ID. The `--output table` format makes the final evidence easy to compare with the challenge requirements.

## Best Practices

- **Use an Elastic IP for a deliberate stability requirement.** It provides a static public IPv4 address that can be reassociated with another resource during recovery.
- **Check before creating.** Looking up the instance and Elastic IP by their Name tags makes the create-or-reuse decision explicit before the resource operation.
- **Select an AMI for the current Region.** AMI IDs are Region-specific, so discover a compatible Linux image instead of copying an ID from another Region.
- **Wait for the target state.** Confirm the instance is running before associating dependent networking resources or using it for application hosting.
- **Tag resources consistently.** The `Name` tags make the instance and Elastic IP easy to identify in later operations and audits.
- **Release unused Elastic IPs.** An address that is no longer required should be disassociated and released to avoid retaining unnecessary public IPv4 resources.

### 📚 Official Documentation

- [Elastic IP addresses for Amazon EC2](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/elastic-ip-addresses-eip.html)
- [describe-vpcs — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/ec2/describe-vpcs.html)
- [describe-subnets — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/ec2/describe-subnets.html)
- [describe-security-groups — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/ec2/describe-security-groups.html)
- [describe-images — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/ec2/describe-images.html)
- [run-instances — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/ec2/run-instances.html)
- [create-tags — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/ec2/create-tags.html)
- [wait instance-running — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/ec2/wait/instance-running.html)
- [allocate-address — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/ec2/allocate-address.html)
- [associate-address — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/ec2/associate-address.html)
- [describe-addresses — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/ec2/describe-addresses.html)
- [describe-instances — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/ec2/describe-instances.html)
