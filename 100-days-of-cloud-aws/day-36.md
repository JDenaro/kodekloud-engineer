# Day 36: Load Balancing EC2 Instances with Application Load Balancer

The Nautilus Development Team needs to set up a new EC2 instance and configure it to run a web server. This EC2 instance should be part of an Application Load Balancer (ALB) setup to ensure high availability and better traffic management. The task involves creating an EC2 instance, setting up an ALB, configuring a target group, and ensuring the web server is accessible via the ALB DNS.

Create a security group: Create a security group named `nautilus-sg` to open port 80 for the default security group (which will be attached to the ALB). Attach `nautilus-sg` security group to the EC2 instance.

Create an EC2 instance: Create an EC2 instance named `nautilus-ec2`. Use any available Ubuntu AMI to create this instance. Configure the instance to run a user data script during its launch.

This script should:

Install the Nginx package.
Start the Nginx service.
Set up an Application Load Balancer: Set up an Application Load Balancer named `nautilus-alb`. Attach default security group to the same.

Create a target group: Create a target group named `nautilus-tg`.

Route traffic: The ALB should route traffic on port 80 to port 80 of the `nautilus-ec2` instance.

Security group adjustments: Make appropriate changes in the default security group attached to the ALB if necessary. Eventually, the Nginx server running under `nautilus-ec2` instance must be accessible using the ALB DNS.

## Specific Requirements:

1. Create a security group named `nautilus-sg` that opens port 80 for the default security group (which is attached to the ALB), and attach `nautilus-sg` to the EC2 instance.
2. Create an EC2 instance named `nautilus-ec2` using any available Ubuntu AMI, with a user data script that installs and starts Nginx.
3. Set up an Application Load Balancer named `nautilus-alb` with the default security group attached.
4. Create a target group named `nautilus-tg`.
5. Route ALB traffic on port 80 to port 80 of the `nautilus-ec2` instance.
6. Adjust the default (ALB) security group as needed so the Nginx server on `nautilus-ec2` is reachable via the ALB DNS.

## Solution

An **Application Load Balancer (ALB)** is a single, stable entry point that distributes incoming HTTP traffic across one or more backend instances — even one instance benefits, since the ALB gives a fixed DNS name and health checking. The pieces connect like this: the **ALB** (in the default security group) receives traffic on port 80; its **listener** forwards to a **target group**; the target group holds the **EC2 instance** and health-checks it; and the instance's own security group (`nautilus-sg`) allows the ALB in.

The security-group design is the crux of this challenge and is done with **source-group references**, not IP ranges: the default SG (on the ALB) allows port 80 from the internet, and `nautilus-sg` (on the instance) allows port 80 **only from the default SG**. That way public traffic can only reach the instance *through* the ALB. An ALB also requires subnets in at least **two Availability Zones**, so we hand it all of the default VPC's subnets.

### 📦 Variables

```bash
AWS_REGION="us-east-1"
SG_NAME="nautilus-sg"
EC2_NAME="nautilus-ec2"
ALB_NAME="nautilus-alb"
TG_NAME="nautilus-tg"
INSTANCE_TYPE="t2.micro"
```

> **Why:** These are the challenge-given names plus the instance size we choose (`t2.micro`, a small free-tier-eligible type). `AWS_REGION` is pinned to `us-east-1`. All IDs (VPC, subnets, security groups, AMI, instance, ARNs) are discovered in the steps because they differ per lab run.

### 🔎 Step 1: Discover the default VPC, subnets, and default security group

```bash
VPC_ID=$(aws ec2 describe-vpcs --region "$AWS_REGION" \
  --filters "Name=is-default,Values=true" \
  --query "Vpcs[0].VpcId" --output text)

DEFAULT_SG_ID=$(aws ec2 describe-security-groups --region "$AWS_REGION" \
  --filters "Name=vpc-id,Values=$VPC_ID" "Name=group-name,Values=default" \
  --query "SecurityGroups[0].GroupId" --output text)

aws ec2 describe-subnets --region "$AWS_REGION" \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --query "Subnets[].SubnetId" --output text
```

Real values from the lab run:

```
VPC_ID=vpc-03ec36de9f8af3f4a  DEFAULT_SG_ID=sg-0488fc1593e3fa9d9
Subnets: subnet-08fae4b5628bddfb3 subnet-05107a24489aad220 subnet-0a20e74c3aa566dcc subnet-09bedfd8d2dfd9555 subnet-03946fb4f84103fe5 subnet-01487ca0a0f3f307e
```

> **Why:** Everything lives in the account's **default VPC**, found with the `is-default` filter. We need its **default security group** (the challenge attaches it to the ALB) and its **subnets** — an ALB must span at least two Availability Zones, and the default VPC provides one subnet per AZ, so listing them all satisfies that. `describe-*` are read-only lookups; `--query`/`--output text` extract the IDs.

### 🔒 Step 2: Create nautilus-sg allowing port 80 from the default (ALB) security group

```bash
SG_ID=$(aws ec2 create-security-group --region "$AWS_REGION" \
  --group-name "$SG_NAME" \
  --description "Web SG for $EC2_NAME (allows 80 from ALB)" \
  --vpc-id "$VPC_ID" \
  --query "GroupId" --output text)

aws ec2 create-tags --region "$AWS_REGION" \
  --resources "$SG_ID" --tags Key=Name,Value="$SG_NAME"

aws ec2 authorize-security-group-ingress --region "$AWS_REGION" \
  --group-id "$SG_ID" \
  --ip-permissions "IpProtocol=tcp,FromPort=80,ToPort=80,UserIdGroupPairs=[{GroupId=$DEFAULT_SG_ID}]"
```

Real value from the lab run: `SG_ID=sg-05bcac2430d9bbb5f`.

> **Why:** `nautilus-sg` is the instance's firewall. Instead of opening port 80 to the world here, the `--ip-permissions` rule with `UserIdGroupPairs=[{GroupId=$DEFAULT_SG_ID}]` is a **source-security-group** rule: it permits port 80 only from anything in the default security group — i.e., only from the ALB. This enforces that visitors reach the instance strictly through the load balancer. We tag it and will attach it to the instance at launch.

### 🌐 Step 3: Open port 80 on the default security group from the internet

```bash
aws ec2 authorize-security-group-ingress --region "$AWS_REGION" \
  --group-id "$DEFAULT_SG_ID" --protocol tcp --port 80 --cidr 0.0.0.0/0
```

> **Why:** The default security group is attached to the ALB, so it must accept public HTTP. This rule allows inbound port `80` from `0.0.0.0/0` (any address), which is the "adjust the default security group" the challenge asks for — without it, the ALB DNS would be unreachable from a browser.

### 🐧 Step 4: Find the latest Ubuntu 22.04 AMI

```bash
aws ec2 describe-images --region "$AWS_REGION" --owners 099720109477 \
  --filters "Name=name,Values=ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*" "Name=state,Values=available" \
  --query "sort_by(Images,&CreationDate)[-1].ImageId" --output text
```

Real value from the lab run: `AMI_ID=ami-0d001f8052688dc45`.

> **Why:** The challenge says "any available Ubuntu AMI," so we look one up rather than hardcode an ID that ages out. `describe-images --owners 099720109477` restricts the search to **Canonical** (Ubuntu's official publisher); the `name` filter matches Ubuntu 22.04 (jammy) x86_64 server images; `sort_by(Images,&CreationDate)[-1]` picks the newest. This is the standard way to always launch a current Ubuntu.

### 🚀 Step 5: Launch the EC2 instance with Nginx user data

```bash
USER_DATA=$(base64 -w0 <<'EOF'
#!/bin/bash
apt-get update -y
apt-get install -y nginx
systemctl enable --now nginx
EOF
)

EC2_ID=$(aws ec2 run-instances --region "$AWS_REGION" \
  --image-id ami-0d001f8052688dc45 --instance-type "$INSTANCE_TYPE" \
  --subnet-id subnet-08fae4b5628bddfb3 --security-group-ids "$SG_ID" \
  --associate-public-ip-address --user-data "$USER_DATA" --count 1 \
  --query "Instances[0].InstanceId" --output text)

aws ec2 create-tags --region "$AWS_REGION" \
  --resources "$EC2_ID" --tags Key=Name,Value="$EC2_NAME"

aws ec2 wait instance-running --region "$AWS_REGION" --instance-ids "$EC2_ID"
```

Real value from the lab run: `EC2_ID=i-03710d3f3ee17b060`.

> **Why:** **User data** is a script AWS runs on the instance's first boot; we base64-encode it (`base64 -w0` keeps it on one line, which `run-instances` expects). On Ubuntu the script uses **APT**: `apt-get install -y nginx` installs the web server and `systemctl enable --now nginx` starts it immediately and on future boots. `run-instances` launches into one of the default subnets with `nautilus-sg` attached (`--security-group-ids`) and a public IP (`--associate-public-ip-address`) so it can reach the internet to download the Nginx package. `wait instance-running` blocks until it's running; the user data finishes shortly after, which is why the target may briefly report unhealthy before Nginx is up.

### 🎯 Step 6: Create the target group and register the instance

```bash
TG_ARN=$(aws elbv2 create-target-group --region "$AWS_REGION" \
  --name "$TG_NAME" \
  --protocol HTTP --port 80 --vpc-id "$VPC_ID" --target-type instance \
  --health-check-protocol HTTP --health-check-path "/" \
  --query "TargetGroups[0].TargetGroupArn" --output text)

aws elbv2 register-targets --region "$AWS_REGION" \
  --target-group-arn "$TG_ARN" \
  --targets "Id=$EC2_ID,Port=80"
```

Real value from the lab run: `TG_ARN=arn:aws:elasticloadbalancing:us-east-1:536088174569:targetgroup/nautilus-tg/732b32feb31bfa9e`.

> **Why:** A **target group** is the pool of backends the ALB forwards to, and it owns the **health check**. `create-target-group` makes it (`elbv2` is the CLI for modern Elastic Load Balancing v2): `--protocol HTTP --port 80` says it receives HTTP on port 80, `--vpc-id` ties it to our VPC, and `--target-type instance` means targets are EC2 instances. `--health-check-protocol HTTP --health-check-path "/"` tells the ALB to poll `http://<instance>/` and treat a success response as healthy. `register-targets` adds our instance with `Id=$EC2_ID,Port=80` — routing traffic to port 80 on the instance, as required.

### ⚖️ Step 7: Create the ALB and its listener

```bash
ALB_ARN=$(aws elbv2 create-load-balancer --region "$AWS_REGION" \
  --name "$ALB_NAME" --type application --scheme internet-facing \
  --subnets subnet-08fae4b5628bddfb3 subnet-05107a24489aad220 subnet-0a20e74c3aa566dcc subnet-09bedfd8d2dfd9555 subnet-03946fb4f84103fe5 subnet-01487ca0a0f3f307e \
  --security-groups "$DEFAULT_SG_ID" \
  --query "LoadBalancers[0].LoadBalancerArn" --output text)

aws elbv2 wait load-balancer-available --region "$AWS_REGION" --load-balancer-arns "$ALB_ARN"

aws elbv2 create-listener --region "$AWS_REGION" \
  --load-balancer-arn "$ALB_ARN" \
  --protocol HTTP --port 80 \
  --default-actions "Type=forward,TargetGroupArn=$TG_ARN"

ALB_DNS=$(aws elbv2 describe-load-balancers --region "$AWS_REGION" \
  --load-balancer-arns "$ALB_ARN" \
  --query "LoadBalancers[0].DNSName" --output text)
```

Real values from the lab run:

```
ALB_ARN=arn:aws:elasticloadbalancing:us-east-1:536088174569:loadbalancer/app/nautilus-alb/5550d9f7309cd6a3
ALB_DNS=nautilus-alb-1421023262.us-east-1.elb.amazonaws.com
```

> **Why:** `create-load-balancer --type application` creates the ALB; `--scheme internet-facing` gives it a public DNS name; `--subnets` places it across all the AZs' subnets (an ALB needs at least two); `--security-groups` attaches the default SG as the challenge requires. `wait load-balancer-available` blocks until it finishes provisioning. A load balancer does nothing without a **listener**, so `create-listener` adds one on `--protocol HTTP --port 80` whose `--default-actions Type=forward,TargetGroupArn=...` forwards every request to `nautilus-tg`. Finally `describe-load-balancers` reads the **DNS name** — the public address users hit.

### ⏳ Step 8: Wait for the target to pass health checks

```bash
aws elbv2 wait target-in-service --region "$AWS_REGION" --target-group-arn "$TG_ARN"
```

> **Why:** The ALB only routes to **healthy** targets. `wait target-in-service` blocks until at least one target passes its health check — meaning Nginx has finished installing (via user data) and is answering on `/`. Until then the ALB has nothing to route to and would return `503`.

### ✅ Step 9: Verify

```bash
aws elbv2 describe-target-health --region "$AWS_REGION" \
  --target-group-arn "$TG_ARN" \
  --query "TargetHealthDescriptions[].{Target:Target.Id,Port:Target.Port,State:TargetHealth.State}" \
  --output table

curl -s -o /dev/null -w "HTTP %{http_code}\n" "http://$ALB_DNS/"

curl -s "http://$ALB_DNS/" | head -5
```

The target is `healthy` and the ALB serves the default Nginx page with a `200`:

```
--------------------------------------------
|           DescribeTargetHealth           |
+------+-----------+-----------------------+
| Port |   State   |        Target         |
+------+-----------+-----------------------+
|  80  |  healthy  |  i-03710d3f3ee17b060  |
+------+-----------+-----------------------+

HTTP 200
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
```

> **Why:** `describe-target-health` confirms the instance registered in `nautilus-tg` is `healthy` — proof the ALB → instance path and the health check both work. `curl` against the **ALB DNS** returns `HTTP 200` and the "Welcome to nginx!" page, confirming end to end: the default SG lets the request into the ALB, the listener forwards to the target group, `nautilus-sg` admits the ALB, and Nginx serves the page. Right after a target first turns healthy there can be a brief window where a request still returns `502` while Nginx settles — if that happens, simply re-run the same `curl` a few seconds later.

## Best Practices

- **Chain security groups by reference.** The ALB SG allows the internet; the instance SG allows only the ALB SG. This keeps instances unreachable except through the load balancer, without brittle IP allow-lists.
- **Front instances with an ALB, not a public instance IP.** The ALB DNS is stable across instance replacements and adds health checks and (optionally) TLS termination.
- **Let user data bootstrap the app.** Installing Nginx via user data makes the instance self-configuring at launch and reproducible when scaled or replaced.
- **Span multiple Availability Zones.** Give the ALB subnets in at least two AZs so it survives the loss of one; add more instances across AZs for real high availability.
- **Look up AMIs dynamically.** Query Canonical's images for the newest Ubuntu instead of pinning an AMI ID that will be deprecated.

### 📚 Official Documentation

- [What is an Application Load Balancer?](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/introduction.html)
- [Target groups for your Application Load Balancers](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-target-groups.html)
- [Run commands on your Linux instance at launch (user data)](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/user-data.html)
- [create-load-balancer — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/elbv2/create-load-balancer.html)
- [create-target-group — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/elbv2/create-target-group.html)
