# Day 29: Establishing Secure Communication Between Public and Private VPCs via VPC Peering

The Nautilus DevOps team has been tasked with demonstrating the use of VPC Peering to enable communication between two VPCs. One VPC will be a private VPC that contains a private EC2 instance, while the other will be the default public VPC containing a publicly accessible EC2 instance.

## Specific Requirements:

1) There is already an existing EC2 instance in the public vpc/subnet: Name `devops-public-ec2`.
2) There is already an existing Private VPC: Name `devops-private-vpc`, CIDR `10.1.0.0/16`.
3) There is already an existing Subnet in `devops-private-vpc`: Name `devops-private-subnet`, CIDR `10.1.1.0/24`.
4) There is already an existing EC2 instance in the private subnet: Name `devops-private-ec2`.
5) Create a Peering Connection between the Default VPC and the Private VPC named `devops-vpc-peering`.
6) Configure Route Tables to enable communication between the two VPCs, ensuring the private EC2 instance is accessible from the public EC2 instance.
7) Test the Connection: add `/root/.ssh/id_rsa.pub` to the public instance's `ec2-user` `authorized_keys`, allow ICMP into the private instance from the public/default VPC CIDR, SSH into the public instance and ping the private instance.

## Solution

**VPC peering** creates a private, direct network link between two VPCs so their instances can talk over private IPs — no internet gateway, VPN, or NAT involved. Peering by itself is inert: after the connection is `active` you must still (a) add a **route in each VPC** pointing the *other* VPC's CIDR at the peering connection, and (b) open the **security groups** for the traffic you want. The two VPCs here have non-overlapping CIDRs (`172.31.0.0/16` and `10.1.0.0/16`), which is mandatory — overlapping ranges make peering routing ambiguous and AWS rejects it.

One practical gotcha: the public instance was launched with a key pair we don't hold, so to add our `id_rsa.pub` to its `authorized_keys` we bootstrap access with **EC2 Instance Connect** (which pushes a temporary key), then append our key for durable SSH from the client host.

### 📦 Variables

```bash
AWS_REGION="us-east-1"
PUBLIC_INSTANCE_NAME="devops-public-ec2"
PRIVATE_VPC_NAME="devops-private-vpc"
PRIVATE_SUBNET_NAME="devops-private-subnet"
PRIVATE_INSTANCE_NAME="devops-private-ec2"
PEERING_NAME="devops-vpc-peering"
KEY_PATH="/root/.ssh/id_rsa"
PUB_KEY_PATH="/root/.ssh/id_rsa.pub"
```

> **Why:** These are the challenge-given names of the existing instances, VPC, subnet, and the peering connection to create, plus the SSH key paths on the AWS client host. `AWS_REGION` is pinned to `us-east-1` because the KodeKloud lab always runs there. Resource **IDs** and **CIDRs** are discovered in the steps below since they differ per lab run.

### 🔑 Step 1: Confirm the SSH key and discover the public instance

```bash
ls -l "$KEY_PATH" "$PUB_KEY_PATH"

PUB_INSTANCE_ID=$(aws ec2 describe-instances --region "$AWS_REGION" \
  --filters "Name=tag:Name,Values=$PUBLIC_INSTANCE_NAME" "Name=instance-state-name,Values=running" \
  --query "Reservations[0].Instances[0].InstanceId" --output text)

PUB_VPC_ID=$(aws ec2 describe-instances --region "$AWS_REGION" --instance-ids "$PUB_INSTANCE_ID" \
  --query "Reservations[0].Instances[0].VpcId" --output text)

PUB_SUBNET_ID=$(aws ec2 describe-instances --region "$AWS_REGION" --instance-ids "$PUB_INSTANCE_ID" \
  --query "Reservations[0].Instances[0].SubnetId" --output text)

PUB_IP=$(aws ec2 describe-instances --region "$AWS_REGION" --instance-ids "$PUB_INSTANCE_ID" \
  --query "Reservations[0].Instances[0].PublicIpAddress" --output text)

PUB_AZ=$(aws ec2 describe-instances --region "$AWS_REGION" --instance-ids "$PUB_INSTANCE_ID" \
  --query "Reservations[0].Instances[0].Placement.AvailabilityZone" --output text)

PUB_SG_ID=$(aws ec2 describe-instances --region "$AWS_REGION" --instance-ids "$PUB_INSTANCE_ID" \
  --query "Reservations[0].Instances[0].SecurityGroups[0].GroupId" --output text)

PUB_VPC_CIDR=$(aws ec2 describe-vpcs --region "$AWS_REGION" --vpc-ids "$PUB_VPC_ID" \
  --query "Vpcs[0].CidrBlock" --output text)
```

Real values from the lab run:

```
PUB_INSTANCE_ID=i-0a24054d10731abff  PUB_VPC_ID=vpc-097989f859715d3f2  PUB_VPC_CIDR=172.31.0.0/16
PUB_SUBNET_ID (in AZ us-east-1b)  PUB_IP=3.83.50.111  PUB_SG_ID=sg-0a0fed7095b1b1828
```

> **Why:** We look up the existing public instance by its *Name* tag rather than creating anything — `describe-instances` with `--filters "Name=tag:Name,..."` and `instance-state-name=running` finds it. From it we read the fields the later steps need: its **VPC** and **subnet** (to know which route table to edit), its **public IP** (to SSH from the client host), its **Availability Zone** (required by EC2 Instance Connect), and its **security group** (to open SSH). `describe-vpcs` gives the public VPC's **CIDR** (`172.31.0.0/16`) — the source range we'll allow ICMP from and the destination the private VPC must route back to. `ls -l` just confirms the SSH key pair the challenge references is present on the client host.

### 🔎 Step 2: Discover the private VPC, subnet, and private instance

```bash
PRIV_VPC_ID=$(aws ec2 describe-vpcs --region "$AWS_REGION" \
  --filters "Name=tag:Name,Values=$PRIVATE_VPC_NAME" \
  --query "Vpcs[0].VpcId" --output text)

PRIV_VPC_CIDR=$(aws ec2 describe-vpcs --region "$AWS_REGION" --vpc-ids "$PRIV_VPC_ID" \
  --query "Vpcs[0].CidrBlock" --output text)

PRIV_SUBNET_ID=$(aws ec2 describe-subnets --region "$AWS_REGION" \
  --filters "Name=vpc-id,Values=$PRIV_VPC_ID" "Name=tag:Name,Values=$PRIVATE_SUBNET_NAME" \
  --query "Subnets[0].SubnetId" --output text)

PRIV_INSTANCE_ID=$(aws ec2 describe-instances --region "$AWS_REGION" \
  --filters "Name=tag:Name,Values=$PRIVATE_INSTANCE_NAME" "Name=instance-state-name,Values=running" \
  --query "Reservations[0].Instances[0].InstanceId" --output text)

PRIV_IP=$(aws ec2 describe-instances --region "$AWS_REGION" --instance-ids "$PRIV_INSTANCE_ID" \
  --query "Reservations[0].Instances[0].PrivateIpAddress" --output text)

PRIV_SG_ID=$(aws ec2 describe-instances --region "$AWS_REGION" --instance-ids "$PRIV_INSTANCE_ID" \
  --query "Reservations[0].Instances[0].SecurityGroups[0].GroupId" --output text)
```

Real values from the lab run:

```
PRIV_VPC_ID=vpc-07e8289277fbfa2ff  PRIV_VPC_CIDR=10.1.0.0/16
PRIV_SUBNET_ID=subnet-0ecd008f17190f409  PRIV_INSTANCE_ID=i-0288244a08f2a4545
PRIV_IP=10.1.1.59  PRIV_SG_ID=sg-09e2079743c15c718
```

> **Why:** Same discovery pattern for the private side, found by Name tag. We capture the private VPC's **CIDR** (`10.1.0.0/16`) for the peering route, the private instance's **private IP** (`10.1.1.59`) — the address we'll ping across the peering — and its **security group** so we can allow ICMP into it. `describe-subnets` locates the private subnet whose route table we'll edit.

### 🔗 Step 3: Create and accept the VPC peering connection

```bash
PEERING_ID=$(aws ec2 create-vpc-peering-connection --region "$AWS_REGION" \
  --vpc-id "$PUB_VPC_ID" --peer-vpc-id "$PRIV_VPC_ID" \
  --query "VpcPeeringConnection.VpcPeeringConnectionId" --output text)

aws ec2 create-tags --region "$AWS_REGION" \
  --resources "$PEERING_ID" --tags Key=Name,Value="$PEERING_NAME"

aws ec2 accept-vpc-peering-connection --region "$AWS_REGION" \
  --vpc-peering-connection-id "$PEERING_ID"
```

Real value from the lab run: `PEERING_ID=pcx-059a3bbb1925f3389`.

> **Why:** `create-vpc-peering-connection` requests the link: `--vpc-id` is the requester (the default/public VPC) and `--peer-vpc-id` the accepter (the private VPC). A new connection starts in `pending-acceptance`, so `accept-vpc-peering-connection` approves it — because both VPCs are in the same account and region, acceptance is immediate and the connection moves to `active`. `create-tags` gives it the required name `devops-vpc-peering`. At this point a private network path exists between the VPCs, but no traffic flows until the route tables know about it (next step).

### 🧭 Step 4: Find the route tables for both subnets

```bash
PUB_RT_ID=$(aws ec2 describe-route-tables --region "$AWS_REGION" \
  --filters "Name=association.subnet-id,Values=$PUB_SUBNET_ID" \
  --query "RouteTables[0].RouteTableId" --output text)

PRIV_RT_ID=$(aws ec2 describe-route-tables --region "$AWS_REGION" \
  --filters "Name=association.subnet-id,Values=$PRIV_SUBNET_ID" \
  --query "RouteTables[0].RouteTableId" --output text)
```

Real values from the lab run: `PUB_RT_ID=rtb-03deb174251983bdd`, `PRIV_RT_ID=rtb-0c493b022e5b9879f`.

> **Why:** Each subnet uses a route table that decides where its traffic goes. `describe-route-tables` filtered by `association.subnet-id` returns the table explicitly associated with each subnet. Both subnets here had their own explicitly associated table, so no fallback to the VPC's main table was needed. We edit these two tables in the next step so each side knows how to reach the other VPC.

### ↔️ Step 5: Add cross-VPC routes through the peering connection

```bash
aws ec2 create-route --region "$AWS_REGION" \
  --route-table-id "$PUB_RT_ID" \
  --destination-cidr-block "$PRIV_VPC_CIDR" \
  --vpc-peering-connection-id "$PEERING_ID"

aws ec2 create-route --region "$AWS_REGION" \
  --route-table-id "$PRIV_RT_ID" \
  --destination-cidr-block "$PUB_VPC_CIDR" \
  --vpc-peering-connection-id "$PEERING_ID"
```

> **Why:** Routing must be configured on **both** sides — a one-sided route gives silent, one-way failure. The first `create-route` tells the public subnet's table "to reach `10.1.0.0/16` (the private VPC), send traffic to the peering connection" (`--vpc-peering-connection-id` as the target). The second does the mirror image on the private side for `172.31.0.0/16`. `--destination-cidr-block` is the range being matched. With both routes in place, packets can travel in each direction over the peering link.

### 🛡️ Step 6: Allow ICMP into the private instance from the public VPC

```bash
aws ec2 authorize-security-group-ingress --region "$AWS_REGION" \
  --group-id "$PRIV_SG_ID" \
  --protocol icmp --port -1 --cidr "$PUB_VPC_CIDR"
```

> **Why:** Routing gets packets to the private instance, but its **security group** (a stateful firewall) still has to accept them. `ping` uses the **ICMP** protocol, so we add an inbound rule allowing ICMP from the public VPC's CIDR (`172.31.0.0/16`). `--protocol icmp` with `--port -1` means "all ICMP types" (ICMP has types, not ports, so `-1` covers them). Without this rule the ping would be routed to the instance and then silently dropped by the firewall.

### 🔑 Step 7: Open SSH on the public instance and install our public key

```bash
aws ec2 authorize-security-group-ingress --region "$AWS_REGION" \
  --group-id "$PUB_SG_ID" \
  --protocol tcp --port 22 --cidr 0.0.0.0/0
```

The public instance was launched with a key we don't hold, so we use **EC2 Instance Connect** to push our public key for a one-time login, then append it to `authorized_keys` so future SSH with `/root/.ssh/id_rsa` works directly:

```bash
aws ec2-instance-connect send-ssh-public-key --region "$AWS_REGION" \
  --instance-id "$PUB_INSTANCE_ID" --availability-zone "$PUB_AZ" \
  --instance-os-user ec2-user --ssh-public-key "file://$PUB_KEY_PATH"

ssh -i "$KEY_PATH" \
  -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -o ConnectTimeout=10 \
  ec2-user@"$PUB_IP" \
  "cat >> ~/.ssh/authorized_keys < /dev/stdin" < "$PUB_KEY_PATH"
```

> **Why:** First `authorize-security-group-ingress` opens TCP port `22` on the public instance's security group so the client host can reach it over SSH. **EC2 Instance Connect** (`send-ssh-public-key`) pushes our public key into the instance's metadata for a 60-second window — enough to establish one SSH session using the matching `id_rsa` even though our key isn't yet in `authorized_keys`. `--availability-zone` and `--instance-os-user` tell it where and as whom to inject the key. Inside that session we append `id_rsa.pub` to `ec2-user`'s `authorized_keys`, making the access **persistent** so subsequent connections (like the verification ping) work with the key alone. The SSH options disable host-key prompts, which would otherwise hang an automated connection to a brand-new host.

### ✅ Step 8: Verify

```bash
ssh -i "$KEY_PATH" \
  -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -o ConnectTimeout=10 \
  ec2-user@"$PUB_IP" "ping -c 4 -W 2 $PRIV_IP"
```

The ping succeeds with no packet loss — the public instance reaches the private instance across the peering:

```
PING 10.1.1.59 (10.1.1.59) 56(84) bytes of data.
64 bytes from 10.1.1.59: icmp_seq=1 ttl=127 time=2.72 ms
64 bytes from 10.1.1.59: icmp_seq=2 ttl=127 time=0.597 ms
64 bytes from 10.1.1.59: icmp_seq=3 ttl=127 time=0.896 ms
64 bytes from 10.1.1.59: icmp_seq=4 ttl=127 time=0.755 ms

--- 10.1.1.59 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3018ms
```

The peering connection is `active` and both route tables carry the cross-VPC route:

```
|     DescribeVpcPeeringConnections    |
|  Accepter  |  vpc-07e8289277fbfa2ff  |
|  Id        |  pcx-059a3bbb1925f3389  |
|  Requester |  vpc-097989f859715d3f2  |
|  Status    |  active                 |

public RT rtb-03deb174251983bdd:  10.1.0.0/16 -> pcx-059a3bbb1925f3389
private RT rtb-0c493b022e5b9879f: 172.31.0.0/16 -> pcx-059a3bbb1925f3389
```

> **Why:** We SSH into the public instance and run `ping` from *there* (not from the client host), because only the public instance sits in a VPC with a peering route to `10.1.0.0/16`. `ping -c 4` sends four ICMP echo requests and `-W 2` waits up to two seconds for each reply. Four replies and `0% packet loss` prove the full path works: both route tables forward across the peering, and the private instance's security group now permits ICMP. `describe-vpc-peering-connections` confirms the connection is `active`, closing out the challenge.

## Best Practices

- **Non-overlapping CIDRs are mandatory.** Peering requires distinct address ranges (`172.31.0.0/16` vs `10.1.0.0/16`); overlapping VPCs cannot be peered because routing would be ambiguous.
- **Add routes on both sides.** Traffic only flows once each VPC's route table has a route to the other's CIDR via the peering connection — a single-sided route yields silent one-way failure.
- **Open security groups for the exact traffic.** Peering plus routes still isn't enough; the destination's security group must allow the protocol (here ICMP) from the source CIDR.
- **Scope ICMP to the peer CIDR, not the world.** Allowing ICMP only from `172.31.0.0/16` keeps the private instance reachable from the peer VPC without exposing it broadly.
- **Prefer EC2 Instance Connect over baking in keys.** When you don't hold an instance's key, Instance Connect grants short-lived access to bootstrap your own key, avoiding long-lived shared credentials.

### 📚 Official Documentation

- [Create a VPC peering connection](https://docs.aws.amazon.com/vpc/latest/peering/create-vpc-peering-connection.html)
- [Update your route tables for a VPC peering connection](https://docs.aws.amazon.com/vpc/latest/peering/vpc-peering-routing.html)
- [Connect using EC2 Instance Connect](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/connect-using-eic.html)
- [Control traffic to your AWS resources using security groups](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-security-groups.html)
