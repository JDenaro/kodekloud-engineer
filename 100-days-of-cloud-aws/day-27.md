# Day 27: Configuring a Public VPC with an EC2 Instance for Internet Access

The Nautilus DevOps Team has received a request from the Networking Team to set up a new public VPC to support a set of public-facing services. This VPC will host various resources that need to be accessible over the internet. As part of this setup, you need to ensure the VPC has public subnets with automatic IP assignment for resources. Additionally, a new EC2 instance will be launched within this VPC to host public applications that require SSH access. This setup will enable the Networking Team to deploy and manage public-facing applications.

Create a public VPC named `devops-pub-vpc`, and a subnet named `devops-pub-subnet` under the same, make sure public IP is being auto assigned to resources under this subnet. Further, create an EC2 instance named `devops-pub-ec2` under this VPC with instance type `t2.micro`. Make sure SSH port 22 is open for this instance and accessible over the internet.

## Specific Requirements:

1. Create a public VPC named `devops-pub-vpc`.
2. Create a subnet named `devops-pub-subnet` under the same, make sure public IP is being auto assigned to resources under this subnet.
3. Further, create an EC2 instance named `devops-pub-ec2` under this VPC with instance type `t2.micro`.
4. Make sure SSH port `22` is open for this instance and accessible over the internet.

## Solution

A subnet becomes public only when the complete network path exists: the subnet automatically assigns public IPv4 addresses, its route table sends `0.0.0.0/0` to an Internet Gateway, and the instance security group allows the required inbound port. The instance was launched only after that public network foundation was configured.

### 📦 Variables

```bash
AWS_REGION="us-east-1"
VPC_NAME="devops-pub-vpc"
VPC_CIDR="10.0.0.0/16"
SUBNET_NAME="devops-pub-subnet"
SUBNET_CIDR="10.0.1.0/24"
INSTANCE_NAME="devops-pub-ec2"
INSTANCE_TYPE="t2.micro"
SSH_PORT=22
```

### 🔎 Step 1: Check for and create the public VPC

First, check whether a VPC named `devops-pub-vpc` already exists:

```bash
aws ec2 describe-vpcs \
  --filters "Name=tag:Name,Values=devops-pub-vpc" \
  --query "Vpcs[0].VpcId" \
  --output text
```

No existing VPC was found, so a new VPC was created with CIDR block `10.0.0.0/16`:

```bash
aws ec2 create-vpc \
  --cidr-block 10.0.0.0/16 \
  --query "Vpc.VpcId" \
  --output text
```

The VPC ID was:

```text
vpc-0abeef3bb9ab297e8
```

Apply the required name tag:

```bash
aws ec2 create-tags \
  --resources vpc-0abeef3bb9ab297e8 \
  --tags "Key=Name,Value=devops-pub-vpc"
```

> **Why:** `describe-vpcs` looks up existing VPCs, and the `tag:Name` filter searches for the requested name. `create-vpc` creates the isolated virtual network; `--cidr-block` defines its private IPv4 address range. `--query` extracts the new VPC ID and `--output text` prints it plainly. `create-tags` adds the name after creation; `--resources` identifies the VPC and `--tags` assigns its `Name` value.

### 🧩 Step 2: Create the subnet and enable automatic public IP assignment

Check whether the requested subnet already exists in the new VPC:

```bash
aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=vpc-0abeef3bb9ab297e8" "Name=tag:Name,Values=devops-pub-subnet" \
  --query "Subnets[0].SubnetId" \
  --output text
```

No existing subnet was found, so the subnet was created with CIDR block `10.0.1.0/24`:

```bash
aws ec2 create-subnet \
  --vpc-id vpc-0abeef3bb9ab297e8 \
  --cidr-block 10.0.1.0/24 \
  --query "Subnet.SubnetId" \
  --output text
```

The subnet ID was:

```text
subnet-09737061cea935996
```

Tag the subnet:

```bash
aws ec2 create-tags \
  --resources subnet-09737061cea935996 \
  --tags "Key=Name,Value=devops-pub-subnet"
```

Enable automatic public IPv4 assignment for resources launched in the subnet:

```bash
aws ec2 modify-subnet-attribute \
  --subnet-id subnet-09737061cea935996 \
  --map-public-ip-on-launch
```

The script confirmed:

```text
Enabled automatic public IPv4 assignment for subnet-09737061cea935996.
```

> **Why:** `describe-subnets` checks for an existing subnet in the target VPC. `create-subnet` creates the subnet; `--vpc-id` places it in the requested VPC and `--cidr-block` assigns its address range. `modify-subnet-attribute` changes subnet behavior, and `--map-public-ip-on-launch` makes EC2 automatically assign public IPv4 addresses to new resources launched there. This setting alone does not provide internet routing; the Internet Gateway and route table are configured next.

### 🌐 Step 3: Create and attach the Internet Gateway

Check whether the VPC already has an attached Internet Gateway:

```bash
aws ec2 describe-internet-gateways \
  --filters "Name=attachment.vpc-id,Values=vpc-0abeef3bb9ab297e8" \
  --query "InternetGateways[0].InternetGatewayId" \
  --output text
```

No attached gateway was found, so one was created:

```bash
aws ec2 create-internet-gateway \
  --query "InternetGateway.InternetGatewayId" \
  --output text
```

The Internet Gateway ID was:

```text
igw-0a89f096a86f3e88a
```

Tag the gateway and attach it to the VPC:

```bash
aws ec2 create-tags \
  --resources igw-0a89f096a86f3e88a \
  --tags "Key=Name,Value=devops-pub-vpc-igw"

aws ec2 attach-internet-gateway \
  --internet-gateway-id igw-0a89f096a86f3e88a \
  --vpc-id vpc-0abeef3bb9ab297e8
```

The gateway was attached successfully:

```text
Attached igw-0a89f096a86f3e88a to vpc-0abeef3bb9ab297e8.
```

> **Why:** `describe-internet-gateways` checks for a gateway already attached to the VPC. `create-internet-gateway` creates the VPC component that connects to the public internet. `attach-internet-gateway` connects it to the VPC; `--internet-gateway-id` identifies the gateway and `--vpc-id` identifies the network receiving the attachment. The separate `create-tags` call names the gateway for later discovery.

### 🛣️ Step 4: Create the public route table and default route

Check whether the subnet already has a route table association:

```bash
aws ec2 describe-route-tables \
  --filters "Name=association.subnet-id,Values=subnet-09737061cea935996" \
  --query "RouteTables[0].RouteTableId" \
  --output text
```

No association existed, so a route table was created for the VPC:

```bash
aws ec2 create-route-table \
  --vpc-id vpc-0abeef3bb9ab297e8 \
  --query "RouteTable.RouteTableId" \
  --output text
```

The route table ID was:

```text
rtb-08f3e192b9479dfe5
```

Tag it and associate it with the public subnet:

```bash
aws ec2 create-tags \
  --resources rtb-08f3e192b9479dfe5 \
  --tags "Key=Name,Value=devops-pub-rt"

aws ec2 associate-route-table \
  --route-table-id rtb-08f3e192b9479dfe5 \
  --subnet-id subnet-09737061cea935996 \
  --query "AssociationId" \
  --output text
```

The association ID was:

```text
rtbassoc-038afce6ddf021591
```

Create the public default route:

```bash
aws ec2 create-route \
  --route-table-id rtb-08f3e192b9479dfe5 \
  --destination-cidr-block 0.0.0.0/0 \
  --gateway-id igw-0a89f096a86f3e88a
```

AWS confirmed the route creation:

```json
{
    "Return": true
}
```

> **Why:** `describe-route-tables` checks whether the subnet already uses a route table. `create-route-table` creates a routing table inside the VPC, and `associate-route-table` connects it to the subnet; `--route-table-id` and `--subnet-id` identify both sides of that association. `create-route` adds a route, `--destination-cidr-block 0.0.0.0/0` matches all IPv4 destinations outside the VPC, and `--gateway-id` sends that traffic to the Internet Gateway. Without this route, a public IP would not provide internet connectivity.

### 🔐 Step 5: Create the security group and allow SSH

Check whether the support security group already exists in the VPC:

```bash
aws ec2 describe-security-groups \
  --filters "Name=vpc-id,Values=vpc-0abeef3bb9ab297e8" "Name=group-name,Values=devops-pub-sg" \
  --query "SecurityGroups[0].GroupId" \
  --output text
```

No group named `devops-pub-sg` existed, so it was created:

```bash
aws ec2 create-security-group \
  --group-name devops-pub-sg \
  --description "SSH access for public DevOps applications" \
  --vpc-id vpc-0abeef3bb9ab297e8 \
  --query "GroupId" \
  --output text
```

The security group ID was:

```text
sg-06b52555a801d87ba
```

Tag the security group:

```bash
aws ec2 create-tags \
  --resources sg-06b52555a801d87ba \
  --tags "Key=Name,Value=devops-pub-sg"
```

Check for an existing public SSH rule:

```bash
aws ec2 describe-security-groups \
  --group-ids sg-06b52555a801d87ba \
  --query "length(SecurityGroups[0].IpPermissions[?IpProtocol=='tcp' && FromPort==\`22\` && ToPort==\`22\`].IpRanges[?CidrIp=='0.0.0.0/0'])" \
  --output text
```

No matching rule existed, so SSH was opened:

```bash
aws ec2 authorize-security-group-ingress \
  --group-id sg-06b52555a801d87ba \
  --protocol tcp \
  --port 22 \
  --cidr 0.0.0.0/0
```

AWS created this rule:

```json
{
    "Return": true,
    "SecurityGroupRules": [
        {
            "SecurityGroupRuleId": "sgr-0a1db490cdca6f8cf",
            "GroupId": "sg-06b52555a801d87ba",
            "GroupOwnerId": "265164650170",
            "IsEgress": false,
            "IpProtocol": "tcp",
            "FromPort": 22,
            "ToPort": 22,
            "CidrIpv4": "0.0.0.0/0",
            "SecurityGroupRuleArn": "arn:aws:ec2:us-east-1:265164650170:security-group-rule/sgr-0a1db490cdca6f8cf"
        }
    ]
}
```

> **Why:** `describe-security-groups` checks the VPC-scoped security group and its inbound permissions. `create-security-group` creates a group for the VPC; `--group-name`, `--description`, and `--vpc-id` define its identity and location. `authorize-security-group-ingress` adds an inbound rule. `--group-id` selects the group, `--protocol tcp` selects TCP, `--port 22` limits access to SSH, and `--cidr 0.0.0.0/0` allows SSH from any IPv4 address on the internet. The separate tag makes the support resource discoverable.

### 🖼️ Step 6: Find an Ubuntu AMI

```bash
aws ec2 describe-images \
  --owners 099720109477 \
  --filters \
    "Name=name,Values=ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*" \
    "Name=state,Values=available" \
    "Name=architecture,Values=x86_64" \
    "Name=root-device-type,Values=ebs" \
    "Name=virtualization-type,Values=hvm" \
  --query "sort_by(Images, &CreationDate)[-1].ImageId" \
  --output text
```

The selected Ubuntu AMI was:

```text
ami-0d001f8052688dc45
```

> **Why:** `describe-images` searches the AMI catalog. `--owners 099720109477` limits the results to Canonical's Ubuntu images. The filters select an available `x86_64` Ubuntu image with an EBS root device and HVM virtualization. Sorting by `CreationDate` chooses the newest matching image.

### 🚀 Step 7: Create and tag the EC2 instance

Check whether an instance with the required name already exists in this VPC:

```bash
aws ec2 describe-instances \
  --filters "Name=vpc-id,Values=vpc-0abeef3bb9ab297e8" "Name=tag:Name,Values=devops-pub-ec2" "Name=instance-state-name,Values=pending,running,stopping,stopped" \
  --query "Reservations[0].Instances[0].InstanceId" \
  --output text
```

No existing instance was found, so one was launched:

```bash
aws ec2 run-instances \
  --image-id ami-0d001f8052688dc45 \
  --instance-type t2.micro \
  --subnet-id subnet-09737061cea935996 \
  --security-group-ids sg-06b52555a801d87ba \
  --associate-public-ip-address \
  --count 1 \
  --query "Instances[0].InstanceId" \
  --output text
```

The instance ID was:

```text
i-0d09bf4299c0ea5e7
```

Apply its required name tag:

```bash
aws ec2 create-tags \
  --resources i-0d09bf4299c0ea5e7 \
  --tags "Key=Name,Value=devops-pub-ec2"
```

Wait for the instance to run:

```bash
aws ec2 wait instance-running \
  --instance-ids i-0d09bf4299c0ea5e7
```

> **Why:** `describe-instances` prevents duplicate instances by checking the VPC, `Name` tag, and non-terminated states. `run-instances` launches the requested server. `--image-id` selects Ubuntu, `--instance-type t2.micro` satisfies the challenge, `--subnet-id` places the instance in the public subnet, and `--security-group-ids` attaches the SSH group. `--associate-public-ip-address` ensures this instance receives a public IPv4 address in addition to the subnet's automatic assignment setting. `--count 1` requests one instance. `create-tags` names it after creation, and `wait instance-running` polls until EC2 reports the instance as `running`; `--instance-ids` identifies the instance being checked.

### ✅ Step 8: Verify

The VPC verification showed the correct CIDR, name, and available state:

```bash
aws ec2 describe-vpcs \
  --vpc-ids vpc-0abeef3bb9ab297e8 \
  --query "Vpcs[0].{VpcId:VpcId,CidrBlock:CidrBlock,IsDefault:IsDefault,Name:Tags[?Key=='Name']|[0].Value,State:State}" \
  --output table
```

```text
----------------------------------------
|             DescribeVpcs             |
+------------+-------------------------+
|  CidrBlock |  10.0.0.0/16            |
|  IsDefault |  False                  |
|  Name      |  devops-pub-vpc         |
|  State     |  available              |
|  VpcId     |  vpc-0abeef3bb9ab297e8  |
+------------+-------------------------+
```

The subnet verification confirmed automatic public IP assignment:

```bash
aws ec2 describe-subnets \
  --subnet-ids subnet-09737061cea935996 \
  --query "Subnets[0].{SubnetId:SubnetId,VpcId:VpcId,CidrBlock:CidrBlock,MapPublicIpOnLaunch:MapPublicIpOnLaunch,Name:Tags[?Key=='Name']|[0].Value,State:State}" \
  --output table
```

```text
-----------------------------------------------------
|                  DescribeSubnets                  |
+----------------------+----------------------------+
|  CidrBlock           |  10.0.1.0/24               |
|  MapPublicIpOnLaunch |  True                      |
|  Name                |  devops-pub-subnet         |
|  State               |  available                 |
|  SubnetId            |  subnet-09737061cea935996  |
|  VpcId               |  vpc-0abeef3bb9ab297e8     |
+----------------------+----------------------------+
```

The Internet Gateway was attached to the VPC:

```bash
aws ec2 describe-internet-gateways \
  --internet-gateway-ids igw-0a89f096a86f3e88a \
  --query "InternetGateways[0].{InternetGatewayId:InternetGatewayId,AttachmentState:Attachments[?VpcId=='vpc-0abeef3bb9ab297e8']|[0].State,VpcId:Attachments[?VpcId=='vpc-0abeef3bb9ab297e8']|[0].VpcId}" \
  --output table
```

```text
-----------------------------------------------------------------------
|                      DescribeInternetGateways                       |
+-----------------+-------------------------+-------------------------+
| AttachmentState |    InternetGatewayId    |          VpcId          |
+-----------------+-------------------------+-------------------------+
|  available      |  igw-0a89f096a86f3e88a  |  vpc-0abeef3bb9ab297e8  |
+-----------------+-------------------------+-------------------------+
```

The route table was associated with the subnet and sent the default route through the Internet Gateway:

```bash
aws ec2 describe-route-tables \
  --route-table-ids rtb-08f3e192b9479dfe5 \
  --query "RouteTables[0].{RouteTableId:RouteTableId,VpcId:VpcId,SubnetAssociation:Associations[?SubnetId=='subnet-09737061cea935996']|[0].AssociationState,DefaultRoute:Routes[?DestinationCidrBlock=='0.0.0.0/0']|[0].GatewayId}" \
  --output table
```

```text
-----------------------------------------------------------------------------
|                            DescribeRouteTables                            |
+-----------------------+-------------------------+-------------------------+
|     DefaultRoute      |      RouteTableId       |          VpcId          |
+-----------------------+-------------------------+-------------------------+
|  igw-0a89f096a86f3e88a |  rtb-08f3e192b9479dfe5  |  vpc-0abeef3bb9ab297e8  |
+-----------------------+-------------------------+-------------------------+
||                            SubnetAssociation                            ||
|+---------------------------+---------------------------------------------+|
||  State                    |  associated                                 ||
|+---------------------------+---------------------------------------------+|
```

The security group and instance were verified as follows:

```bash
aws ec2 describe-security-groups \
  --group-ids sg-06b52555a801d87ba \
  --query "SecurityGroups[0].{GroupId:GroupId,GroupName:GroupName}" \
  --output table

aws ec2 describe-instances \
  --instance-ids i-0d09bf4299c0ea5e7 \
  --query "Reservations[0].Instances[0].{ImageId:ImageId,InstanceId:InstanceId,InstanceType:InstanceType,Name:Tags[?Key=='Name']|[0].Value,PrivateIp:PrivateIpAddress,PublicIp:PublicIpAddress,SubnetId:SubnetId,VpcId:VpcId,State:State.Name}" \
  --output table
```

```text
-------------------------------------------
|         DescribeSecurityGroups          |
+-----------------------+-----------------+
|        GroupId        |    GroupName    |
+-----------------------+-----------------+
|  sg-06b52555a801d87ba |  devops-pub-sg  |
+-----------------------+-----------------+

----------------------------------------------
|              DescribeInstances             |
+---------------+----------------------------+
|  ImageId      |  ami-0d001f8052688dc45     |
|  InstanceId   |  i-0d09bf4299c0ea5e7       |
|  InstanceType |  t2.micro                  |
|  Name         |  devops-pub-ec2            |
|  PrivateIp    |  10.0.1.179                |
|  PublicIp     |  52.0.180.232              |
|  State        |  running                   |
|  SubnetId     |  subnet-09737061cea935996  |
|  VpcId        |  vpc-0abeef3bb9ab297e8     |
+---------------+----------------------------+
```

The authorization response in Step 5 confirms that `sg-06b52555a801d87ba` allows TCP port `22` from `0.0.0.0/0`. The public VPC configuration completed successfully.

> **Why:** `describe-vpcs`, `describe-subnets`, `describe-internet-gateways`, `describe-route-tables`, `describe-security-groups`, and `describe-instances` provide the final evidence for each requirement. Their resource-ID parameters target the exact objects created, `--query` selects the properties that matter, and `--output table` makes the relationships easy to inspect: the VPC contains the subnet, the subnet assigns public IPs, the route table uses the Internet Gateway, the security group contains the SSH rule, and the instance is `running` with both private and public IP addresses.

## Best Practices

- **Build the public path end to end.** A public IP requires subnet auto-assignment, an Internet Gateway, a default route, and an ingress rule; configuring only one of these is insufficient.
- **Use dedicated network resources.** A route table and security group dedicated to this public subnet make the intended exposure easier to audit and change.
- **Check before creating.** Every named resource is looked up first so rerunning the procedure can reuse existing infrastructure.
- **Keep tags separate.** Applying `Name` tags with `create-tags` makes each VPC resource easy to discover without hiding the naming operation inside a dense create command.
- **Expose only the required port.** The security group opens TCP port `22` for the challenge; application ports should be added only when a service actually needs them.
- **Restrict SSH in real environments.** `0.0.0.0/0` is required by this lab, but production SSH should be limited to a trusted CIDR, VPN, bastion, or Systems Manager access.

### 📚 Official Documentation

- [create-vpc — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/ec2/create-vpc.html)
- [describe-vpcs — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/ec2/describe-vpcs.html)
- [create-subnet — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/ec2/create-subnet.html)
- [modify-subnet-attribute — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/ec2/modify-subnet-attribute.html)
- [create-internet-gateway — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/ec2/create-internet-gateway.html)
- [attach-internet-gateway — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/ec2/attach-internet-gateway.html)
- [create-route-table — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/ec2/create-route-table.html)
- [associate-route-table — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/ec2/associate-route-table.html)
- [create-route — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/ec2/create-route.html)
- [create-security-group — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/ec2/create-security-group.html)
- [authorize-security-group-ingress — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/ec2/authorize-security-group-ingress.html)
- [run-instances — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/ec2/run-instances.html)
- [create-tags — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/ec2/create-tags.html)
- [instance-running waiter — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/ec2/wait/instance-running.html)
