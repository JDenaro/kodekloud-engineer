# Day 40: Troubleshooting Internet Accessibility for an EC2-Hosted Application

The Nautilus Development Team deployed a web application on an EC2 instance inside a public VPC named `xfusion-vpc`. The app runs on Nginx and should be reachable from the internet on port 80. The security group `xfusion-sg` already allows port 80 and the instance settings look correct, yet the app is still unreachable. The suspected cause is the VPC configuration.

## Task Requirements

1. **Verify VPC configuration:** Ensure `xfusion-vpc` is properly configured to allow internet access.
2. **Ensure accessibility:** Make sure the EC2 instance `xfusion-ec2` running Nginx is reachable from the internet on port 80.

## Solution

For an EC2 instance in a public subnet to be reachable from the internet, **all** of the following must be true:

- An **Internet Gateway (IGW)** is attached to the VPC.
- The subnet's **route table** has a default route (`0.0.0.0/0`) pointing to that IGW.
- The instance has a **public IP** address.
- The **security group** allows inbound port 80.
- The **network ACL** allows inbound/outbound traffic.

We diagnose by **following the inbound packet path** — `Internet → IGW → Route Table → Public IP → Security Group → NACL` — so we catch whatever breaks the path earliest. Start by resolving the resource IDs, since a single `describe-instances` call feeds every step that follows.

### 📦 Variables

Define the task's fixed values — the names given in the challenge — in one place before running any step. Resource **IDs** are not here: they differ on every lab run, so we discover them with lookup commands in Step 1.

```bash
AWS_REGION="us-east-1"
VPC_NAME="xfusion-vpc"
INSTANCE_NAME="xfusion-ec2"
SG_NAME="xfusion-sg"
```

> **Why:** These are the three resource names the challenge hands us (`xfusion-vpc`, `xfusion-ec2`, `xfusion-sg`) plus the lab region. Keeping them in named variables means the commands below read clearly and there's a single place to change a value. `AWS_REGION` is hardcoded to `us-east-1` because that is where the KodeKloud lab always runs.

### 🔎 Step 1: Resolve the Resource IDs

Before diagnosing anything, we translate those names into the IDs the rest of the commands need — the VPC, the instance, the subnet it sits in, and the public IP it currently has (if any).

```bash
VPC_ID=$(aws ec2 describe-vpcs --region "$AWS_REGION" \
  --filters "Name=tag:Name,Values=$VPC_NAME" \
  --query "Vpcs[0].VpcId" --output text)

INSTANCE_ID=$(aws ec2 describe-instances --region "$AWS_REGION" \
  --filters "Name=tag:Name,Values=$INSTANCE_NAME" "Name=instance-state-name,Values=running" \
  --query "Reservations[0].Instances[0].InstanceId" --output text)

SUBNET_ID=$(aws ec2 describe-instances --region "$AWS_REGION" \
  --instance-ids "$INSTANCE_ID" \
  --query "Reservations[0].Instances[0].SubnetId" --output text)

PUBLIC_IP=$(aws ec2 describe-instances --region "$AWS_REGION" --instance-ids "$INSTANCE_ID" \
  --query "Reservations[0].Instances[0].PublicIpAddress" --output text)
```

Running these prints nothing on their own; inspect what they captured:

```bash
echo "VPC:      $VPC_ID"
echo "Instance: $INSTANCE_ID"
echo "Subnet:   $SUBNET_ID"
echo "Public IP: $PUBLIC_IP"
```

```
VPC:      vpc-0a1b2c3d4e5f67890
Instance: i-0abc123def4567890
Subnet:   subnet-0f9e8d7c6b5a43210
Public IP: 54.210.15.88
```

> **Why:** These IDs must be *discovered* per lab because AWS assigns them randomly each run, so they live here in Step 1 rather than in the Variables block. The `describe-*` commands (`describe-vpcs`, `describe-instances`) are read-only queries that return details about existing resources. `--filters` narrows the results by an attribute — here `tag:Name` (a resource's *Name* tag) to find our VPC/instance by name, and `instance-state-name` to pick only a *running* instance; `--instance-ids` instead looks up one specific instance directly. `--query` then extracts just the field we care about using **JMESPath**, the AWS CLI's built-in filtering language, and `--output text` prints that raw value without JSON formatting so it can be stored cleanly in a shell variable. A single `describe-instances` call feeds the subnet and public-IP lookups that every later step depends on.

### 🔎 Step 2: Diagnose — Check for an Internet Gateway

```bash
aws ec2 describe-internet-gateways --region "$AWS_REGION" \
  --filters "Name=attachment.vpc-id,Values=$VPC_ID" \
  --query "InternetGateways[0].InternetGatewayId" --output text
```

```
None
```

> **Why:** A VPC (Virtual Private Cloud) is your own private, isolated network inside AWS. By default nothing in it can reach the internet. An **Internet Gateway (IGW)** is the door that connects a VPC to the internet — without one attached, traffic has no way in or out, no matter how everything else is set up. So this is the very first thing to check. Here the command returns `None`, meaning no IGW is attached to `xfusion-vpc` — that is our root cause. `describe-internet-gateways` is a read-only query, and `--filters "Name=attachment.vpc-id,Values=$VPC_ID"` restricts it to gateways currently attached to our VPC.

### ♻️ Step 3: Reuse a Detached IGW (or Create One)

Before creating a new gateway, check whether an unattached IGW already exists in the account and reuse it — this avoids leaving orphaned gateways behind.

```bash
aws ec2 describe-internet-gateways --region "$AWS_REGION" \
  --query "InternetGateways[?length(Attachments)==\`0\`].InternetGatewayId | [0]" \
  --output text
```

```
None
```

No detached Internet Gateway existed in the account either, so a new one has to be created and tagged before attaching it:

```bash
IGW_ID=$(aws ec2 create-internet-gateway --region "$AWS_REGION" \
  --query "InternetGateway.InternetGatewayId" --output text)

aws ec2 create-tags --region "$AWS_REGION" \
  --resources "$IGW_ID" --tags Key=Name,Value=xfusion-igw

aws ec2 attach-internet-gateway --region "$AWS_REGION" \
  --internet-gateway-id "$IGW_ID" --vpc-id "$VPC_ID"
```

```
igw-0c1d2e3f4a5b60718
```

> **Why:** An Internet Gateway must be **attached** to a VPC to work — an IGW that exists but isn't attached does nothing. First we run `describe-internet-gateways` (a read-only query) filtered by `--query` to find an existing IGW with no attachments; on this lab run none existed, so `create-internet-gateway` makes a new one, and the following `create-tags` call assigns it a Name tag — kept as a separate command rather than an inline `--tag-specifications` so each step stays simple. Reusing a detached IGW instead of always creating one teaches a good cloud habit: reuse existing resources instead of piling up orphaned ones that clutter your account (and can cost money). Finally, `attach-internet-gateway` plugs the gateway into our VPC — `--internet-gateway-id` says which IGW and `--vpc-id` which VPC.

### 🧭 Step 4: Verify the Route Table Has a Default Route

Find the route table associated with the instance's subnet and confirm it sends `0.0.0.0/0` to the IGW.

```bash
aws ec2 describe-route-tables --region "$AWS_REGION" \
  --filters "Name=association.subnet-id,Values=$SUBNET_ID" \
  --query "RouteTables[0].RouteTableId" --output text
```

No route table is explicitly associated with the subnet, so the subnet falls back to the VPC's default **main** route table — we look that up instead:

```bash
RTB_ID=$(aws ec2 describe-route-tables --region "$AWS_REGION" \
  --filters "Name=vpc-id,Values=$VPC_ID" "Name=association.main,Values=true" \
  --query "RouteTables[0].RouteTableId" --output text)

aws ec2 describe-route-tables --region "$AWS_REGION" --route-table-ids "$RTB_ID" \
  --query "RouteTables[0].Routes" --output table
```

```
-------------------------------------------------------------------
|                        DescribeRouteTables                       |
+-------------------------+-------------+-------------------------+
|  DestinationCidrBlock   |  GatewayId  |         State           |
+-------------------------+-------------+-------------------------+
|  172.31.0.0/16          |  local      |  active                 |
|  0.0.0.0/0              |  igw-0c1... |  active                 |
+-------------------------+-------------+-------------------------+
```

> **Why:** Having an IGW isn't enough — the subnet also needs to be *told* to send internet-bound traffic to it. That job belongs to the **route table**, a set of rules that decides where network traffic goes. The rule `0.0.0.0/0 → igw-...` means "send traffic destined for anywhere on the internet to the Internet Gateway" (`0.0.0.0/0` is shorthand for "all IP addresses"). A subnet that has this route is called a **public subnet**. We find the right route table with `describe-route-tables`: `--filters "Name=association.subnet-id,Values=$SUBNET_ID"` returns the table explicitly associated with our subnet, and the fallback filter `"Name=association.main,Values=true"` returns the VPC's default *main* table (the one a subnet uses when it isn't explicitly linked to any other). A subtle gotcha: the route can already exist but sit **inactive** while its IGW is detached — attaching the IGW in Step 3 brings it back to life. If the route is missing entirely, add it with `aws ec2 create-route`, which inserts a single new route into a table: `--route-table-id` says which table to edit, `--destination-cidr-block 0.0.0.0/0` is the traffic being matched (all internet addresses), and `--gateway-id` is the target the matched traffic is sent to (our IGW):

> ```bash
> aws ec2 create-route --route-table-id "$RTB_ID" \
>   --destination-cidr-block 0.0.0.0/0 --gateway-id "$IGW_ID"
> ```

### 📡 Step 5: Confirm the Instance Has a Public IP

```bash
echo "Public IP: $PUBLIC_IP"
```

```
Public IP: 54.210.15.88
```

> **Why:** For someone on the internet to reach your instance, the instance needs a **public IP address** — a globally reachable address, as opposed to the private IP it uses inside the VPC. Even with a working IGW and route, a server with no public IP simply has no internet-facing address to connect to. If this prints `None`, allocate an **Elastic IP** (a static public IP that belongs to your account) and attach it to the instance. In the snippet below, `allocate-address --domain vpc` reserves a new Elastic IP for use inside a VPC and returns an `AllocationId` (its handle), and `associate-address` binds that address to the instance — `--instance-id` says which instance and `--allocation-id` which Elastic IP:

> ```bash
> ALLOC_ID=$(aws ec2 allocate-address --region "$AWS_REGION" --domain vpc --query "AllocationId" --output text)
> aws ec2 associate-address --region "$AWS_REGION" --instance-id "$INSTANCE_ID" --allocation-id "$ALLOC_ID"
> PUBLIC_IP=$(aws ec2 describe-instances --region "$AWS_REGION" --instance-ids "$INSTANCE_ID" \
>   --query "Reservations[0].Instances[0].PublicIpAddress" --output text)
> ```

### 🔓 Step 6: Confirm Security Group and Network ACL Allow Port 80

```bash
SG_ID=$(aws ec2 describe-security-groups --region "$AWS_REGION" \
  --filters "Name=group-name,Values=$SG_NAME" "Name=vpc-id,Values=$VPC_ID" \
  --query "SecurityGroups[0].GroupId" --output text)

aws ec2 describe-security-group-rules --region "$AWS_REGION" \
  --filters "Name=group-id,Values=$SG_ID" \
  --query "SecurityGroupRules[?FromPort==\`80\` && !IsEgress]" --output table
```

```
------------------------------------------------------------------------------
|                          DescribeSecurityGroupRules                        |
+---------------+-----------+----------+----------+------------------------+
| CidrIpv4      | FromPort  | ToPort   | IpProtocol|  SecurityGroupRuleId   |
+---------------+-----------+----------+----------+------------------------+
| 0.0.0.0/0     | 80        | 80       | tcp       | sgr-0a1b2c3d4e5f60718 |
+---------------+-----------+----------+----------+------------------------+
```

An inbound rule for port 80 already exists, so no `authorize-security-group-ingress` call is needed here. Had the list come back empty, the fix would be:

```bash
aws ec2 authorize-security-group-ingress --region "$AWS_REGION" \
  --group-id "$SG_ID" --protocol tcp --port 80 --cidr 0.0.0.0/0
```

```bash
aws ec2 describe-network-acls --region "$AWS_REGION" \
  --filters "Name=association.subnet-id,Values=$SUBNET_ID" \
  --query "NetworkAcls[0].Entries" --output table
```

```
------------------------------------------------------
|                 DescribeNetworkAcls                 |
+-------------+----------+---------+--------+----------+
| CidrBlock   | Egress   | Protocol| Rule#  | RuleAction|
+-------------+----------+---------+--------+----------+
| 0.0.0.0/0   | False    | -1      | 100    | allow    |
| 0.0.0.0/0   | True     | -1      | 100    | allow    |
+-------------+----------+---------+--------+----------+
```

> **Why:** These are the two firewalls AWS puts in front of your instance. First `describe-security-groups` (a read-only query) resolves the security group's ID from its name — `--filters "Name=group-name,Values=$SG_NAME"` matches by group name and `"Name=vpc-id,Values=$VPC_ID"` scopes the search to our VPC (group names are unique only within a VPC). A **security group** acts at the instance level and is *stateful* (if you allow traffic in, the reply is automatically allowed out) — it must permit inbound port 80, the port web servers use for HTTP. `describe-security-group-rules` lists the group's individual rules so we can check whether one already covers inbound TCP port 80 before adding a duplicate; `authorize-security-group-ingress` is the command that would add that inbound rule if missing — `--group-id` picks the security group, `--protocol tcp` and `--port 80` define the traffic type, and `--cidr 0.0.0.0/0` sets the allowed sources (here, anywhere on the internet). A **network ACL (NACL)** is a second firewall at the subnet level and is *stateless*. Its default allows everything, but a custom NACL could silently block port 80 — so `describe-network-acls` lists its rules to rule that out. Both layers already allow port 80, so this layer is fine.

### ✅ Step 7: Verify

```bash
curl -o /dev/null -s -w "HTTP Status: %{http_code}\n" "http://$PUBLIC_IP"
```

```
HTTP Status: 200
```

> **Why:** `curl` is a command-line tool that makes an HTTP request — here we use it to check whether the web server actually answers from outside. `-o /dev/null` throws away the page body (we don't care about the HTML, only whether it responds), `-s` runs silently (no progress bar), and `-w "...%{http_code}..."` prints the returned HTTP status code. An `HTTP Status: 200` confirms the Nginx application is now reachable from the internet; a hang or `000` means traffic still isn't getting through.

## Best Practices

- **Trace the full path in order.** Public reachability requires IGW → default route → public IP → security group → NACL. Diagnose along the packet path so you catch the earliest break instead of guessing.
- **Reuse before creating.** Check for detached gateways/EIPs before allocating new ones to avoid orphaned, billable resources.
- **Least privilege on ingress.** Open only the ports you need (port 80 here), not `0-65535`. For SSH/admin, restrict the source CIDR to known IPs instead of `0.0.0.0/0`.
- **Use security groups as the primary control.** They are stateful and easier to reason about; keep network ACLs at their permissive default unless you need subnet-wide deny rules.
- **Prefer a load balancer for production.** Front the instance with an ALB so the public entry point is stable and TLS terminates at the edge, keeping instances in private subnets.

### 📚 Official Documentation

- [Enable internet access for a VPC using an internet gateway](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Internet_Gateway.html)
- [Configure route tables](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Route_Tables.html)
- [Control traffic to your AWS resources using security groups](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-security-groups.html)
- [Control subnet traffic with network access control lists](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-network-acls.html)
- [Elastic IP addresses](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/elastic-ip-addresses-eip.html)
