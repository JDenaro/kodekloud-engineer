# Day 44: Implementing Auto Scaling for High Availability in AWS

The DevOps team is tasked with setting up a highly available web application using AWS. To achieve this, they plan to use an Auto Scaling Group (ASG) to ensure that the required number of EC2 instances are always running, and an Application Load Balancer (ALB) to distribute traffic across these instances. The goal of this task is to set up an ASG that automatically scales EC2 instances based on CPU utilization, and an ALB that directs incoming traffic to the instances. The EC2 instances should have Nginx installed and running to serve web traffic.

## Specific Requirements:

1. Create an EC2 launch template named `devops-launch-template` that specifies the configuration for the EC2 instances, including the Amazon Linux 2 AMI, `t2.micro` instance type, and a security group that allows HTTP traffic on port 80.
2. Add a User Data script to the launch template to install Nginx on the EC2 instances when they are launched. The script should install Nginx, start the Nginx service, and enable it to start on boot.
3. Create an Auto Scaling Group named `devops-asg` that uses the launch template and ensures a minimum of 1 instance, desired capacity is 1 instance and a maximum of 2 instances are running based on CPU utilization. Set the target CPU utilization to 50%.
4. Create a target group named `devops-tg`, an Application Load Balancer named `devops-alb` and configure it to listen on port 80. Ensure the ALB is associated with the Auto Scaling Group and distributes traffic across the instances.
5. Configure health checks on the ALB to ensure it routes traffic only to healthy instances.
6. Verify that the ALB's DNS name is accessible and that it displays the default Nginx page served by the EC2 instances.

## Solution

This challenge wires together several services into one self-healing, self-scaling web tier. The mental model:

- A **launch template** is a blueprint (AMI, instance type, security group, startup script) that says *how* to build each EC2 instance.
- An **Auto Scaling Group (ASG)** keeps a target number of instances alive using that blueprint, replacing failed ones and adding/removing instances based on load.
- A **target group** is the pool of instances the load balancer can send traffic to, and it's also what runs the **health checks**.
- An **Application Load Balancer (ALB)** is the single public entry point; its **listener** accepts traffic on port 80 and forwards it to the target group.

The pieces must be created in dependency order: security group and AMI first, then the launch template, the ALB + target group, and finally the ASG that ties the template to the target group and adds the scaling rule.

### 📦 Variables

Define the values that come directly from the challenge (the region and the resource names). Everything else — the VPC, its subnets, and the AMI — is discovered in Step 1, because those IDs differ on every lab run and must never be hardcoded.

```bash
AWS_REGION="us-east-1"
LT_NAME="devops-launch-template"
ASG_NAME="devops-asg"
TG_NAME="devops-tg"
ALB_NAME="devops-alb"
SG_NAME="devops-sg"
```

> **Why:** Keeping the fixed, challenge-given inputs in one place makes the later commands read cleanly and prevents typos. `AWS_REGION` is pinned to `us-east-1` because that is where the KodeKloud lab runs; the four names come straight from the task requirements.

### 🔎 Step 1: Discover the VPC, Subnets, and AMI

Before creating anything we need to know *where* to build. The lab already ships with a default network, so instead of inventing IDs we ask AWS for the ones that exist and capture them into shell variables.

```bash
VPC_ID=$(aws ec2 describe-vpcs --region "$AWS_REGION" --filters "Name=is-default,Values=true" \
  --query "Vpcs[0].VpcId" --output text)

echo "$VPC_ID"
```

```
vpc-0a1b2c3d4e5f67890
```

```bash
SUBNET_IDS=$(aws ec2 describe-subnets --region "$AWS_REGION" --filters "Name=vpc-id,Values=$VPC_ID" \
  --query "Subnets[*].SubnetId" --output text | tr '\t' ',')

echo "$SUBNET_IDS"
```

```
subnet-01aa,subnet-02bb,subnet-03cc,subnet-04dd,subnet-05ee,subnet-06ff
```

```bash
AMI_ID=$(aws ssm get-parameters --region "$AWS_REGION" \
  --names /aws/service/ami-amazon-linux-latest/amzn2-ami-hvm-x86_64-gp2 \
  --query "Parameters[0].Value" --output text)

echo "$AMI_ID"
```

```
ami-0abcdef1234567890
```

> **Why:** These read-only lookups gather the IDs the rest of the guide needs, one at a time so each result is easy to inspect. `aws ec2 describe-vpcs` returns details about your **VPCs** (Virtual Private Clouds — isolated private networks in AWS); `--filters` narrows results by an attribute, here `is-default` to find the ready-made default VPC. `aws ec2 describe-subnets` lists the **subnets** (address ranges, one per Availability Zone) inside that VPC, filtered with `vpc-id`. In both, `--query` extracts just the field we want using **JMESPath** (the CLI's built-in filter language) and `--output text` prints raw values so they store cleanly in a shell variable. `tr '\t' ','` converts the tab-separated subnet list into the comma-separated form later commands expect. For the AMI we avoid hardcoding an ID that changes over time: `aws ssm get-parameters` reads an AWS-published **Systems Manager public parameter** that always points at the newest **Amazon Linux 2 AMI** (an AMI, Amazon Machine Image, is the OS template an instance boots from), and `--names` selects which parameter to read.

### 🔒 Step 2: Create the Security Group (Allow HTTP Port 80)

```bash
SG_ID=$(aws ec2 create-security-group --region "$AWS_REGION" \
  --group-name $SG_NAME \
  --description "Allow HTTP traffic on port 80" \
  --vpc-id $VPC_ID \
  --query "GroupId" --output text)

echo "$SG_ID"
```

```
sg-0f1e2d3c4b5a69870
```

```bash
aws ec2 authorize-security-group-ingress --region "$AWS_REGION" \
  --group-id $SG_ID \
  --protocol tcp --port 80 --cidr 0.0.0.0/0
```

> **Why:** A **security group** is a virtual firewall attached to an instance that controls which traffic is allowed in and out. `aws ec2 create-security-group` creates one: `--group-name` names it, `--description` is a required human-readable note, and `--vpc-id` places it in our VPC; we capture the returned `GroupId`. By default a new security group allows *no* inbound traffic, so `aws ec2 authorize-security-group-ingress` adds an inbound rule: `--protocol tcp` and `--port 80` open the HTTP port, and `--cidr 0.0.0.0/0` allows it from any source IP (the whole internet) — required so the ALB and public visitors can reach the web server.

### 📝 Step 3: Create the Launch Template with Nginx User Data

```bash
cat <<'EOF' > user-data.sh
#!/bin/bash
yum update -y
amazon-linux-extras install -y nginx1
systemctl start nginx
systemctl enable nginx
EOF

USER_DATA=$(base64 -w0 user-data.sh)

aws ec2 create-launch-template --region "$AWS_REGION" \
  --launch-template-name $LT_NAME \
  --launch-template-data "{
    \"ImageId\": \"$AMI_ID\",
    \"InstanceType\": \"t2.micro\",
    \"SecurityGroupIds\": [\"$SG_ID\"],
    \"UserData\": \"$USER_DATA\"
  }"

rm user-data.sh
```

> **Why:** A **launch template** captures everything needed to launch an instance so the ASG can stamp out identical copies. `aws ec2 create-launch-template` creates it: `--launch-template-name` names it, and `--launch-template-data` is the JSON blueprint — `ImageId` (the Amazon Linux 2 AMI discovered in Step 1), `InstanceType` (`t2.micro`, a small, free-tier-eligible size), `SecurityGroupIds` (the firewall from Step 1), and `UserData`. **User Data** is a script AWS runs automatically the first time an instance boots; here it updates packages, installs Nginx via `amazon-linux-extras`, `systemctl start`s the service now, and `systemctl enable`s it so it also starts on future reboots. AWS requires User Data to be **base64-encoded**, which is what `base64 -w0` does (`-w0` keeps it on a single line with no wrapping). The `cat <<'EOF'` here-document writes the script to a file first; the quotes around `'EOF'` stop the shell from expanding `$` inside the script.

### 🎯 Step 4: Create the Target Group with Health Checks

```bash
TG_ARN=$(aws elbv2 create-target-group --region "$AWS_REGION" \
  --name $TG_NAME \
  --protocol HTTP --port 80 \
  --vpc-id $VPC_ID \
  --target-type instance \
  --health-check-protocol HTTP \
  --health-check-path "/" \
  --query "TargetGroups[0].TargetGroupArn" --output text)

echo "$TG_ARN"
```

```
arn:aws:elasticloadbalancing:us-east-1:123456789012:targetgroup/devops-tg/abc123def4567890
```

> **Why:** A **target group** is the pool of destinations the load balancer forwards requests to, and it owns the **health check** logic. `aws elbv2 create-target-group` creates it (`elbv2` is the CLI for the modern Elastic Load Balancing v2, which includes ALBs): `--name` names it; `--protocol HTTP` and `--port 80` say the group receives HTTP traffic on port 80; `--vpc-id` ties it to our VPC; `--target-type instance` means targets are registered as EC2 instances. The health-check flags define how the ALB decides an instance is healthy: `--health-check-protocol HTTP` and `--health-check-path "/"` make the ALB periodically request `http://<instance>/` and expect a success response — instances that fail stop receiving traffic. We capture the group's **ARN** (Amazon Resource Name, its unique ID) for later steps.

### ⚖️ Step 5: Create the ALB and Its Listener

```bash
ALB_ARN=$(aws elbv2 create-load-balancer --region "$AWS_REGION" \
  --name $ALB_NAME \
  --type application \
  --subnets $(echo $SUBNET_IDS | tr ',' ' ') \
  --security-groups $SG_ID \
  --query "LoadBalancers[0].LoadBalancerArn" --output text)

echo "$ALB_ARN"
```

```
arn:aws:elasticloadbalancing:us-east-1:123456789012:loadbalancer/app/devops-alb/9876543210fedcba
```

```bash
aws elbv2 wait load-balancer-available --region "$AWS_REGION" --load-balancer-arns $ALB_ARN

aws elbv2 create-listener --region "$AWS_REGION" \
  --load-balancer-arn $ALB_ARN \
  --protocol HTTP --port 80 \
  --default-actions Type=forward,TargetGroupArn=$TG_ARN
```

> **Why:** The **Application Load Balancer** is the single public front door that spreads traffic across instances. `aws elbv2 create-load-balancer` creates it: `--type application` selects an ALB (Layer 7, HTTP-aware); `--subnets` places it across multiple Availability Zones for high availability (an ALB needs at least two, so we expand our subnet list, converting commas back to spaces); `--security-groups` attaches the firewall. `aws elbv2 wait load-balancer-available` blocks until the ALB finishes provisioning, polling the load balancer named by `--load-balancer-arns`. A load balancer does nothing until it has a **listener** — `aws elbv2 create-listener` adds one: `--load-balancer-arn` says which ALB to attach it to, `--protocol HTTP --port 80` tells it to accept traffic on port 80, and `--default-actions Type=forward,TargetGroupArn=...` forwards every request to the target group from Step 4.

### 🔄 Step 6: Create the Auto Scaling Group

```bash
aws autoscaling create-auto-scaling-group --region "$AWS_REGION" \
  --auto-scaling-group-name $ASG_NAME \
  --launch-template "LaunchTemplateName=$LT_NAME,Version=\$Latest" \
  --min-size 1 --max-size 2 --desired-capacity 1 \
  --target-group-arns $TG_ARN \
  --health-check-type ELB \
  --health-check-grace-period 300 \
  --vpc-zone-identifier "$SUBNET_IDS"
```

> **Why:** The **Auto Scaling Group** keeps the right number of instances running and self-heals. `aws autoscaling create-auto-scaling-group` creates it: `--auto-scaling-group-name` names the group; `--launch-template LaunchTemplateName=...,Version=$Latest` tells it to build instances from our template's newest version; `--min-size 1`, `--max-size 2`, and `--desired-capacity 1` set the floor, ceiling, and current target instance counts as the challenge requires. `--target-group-arns` registers every instance the ASG launches into the target group automatically, wiring the ASG to the ALB. `--health-check-type ELB` makes the ASG trust the load balancer's health checks (not just the EC2 status check), so an instance failing its HTTP check gets replaced; `--health-check-grace-period 300` gives a new instance 300 seconds to boot and start Nginx before health checks count against it — a short grace period is a common cause of a `502 Bad Gateway`, because the ALB routes to an instance whose Nginx isn't up yet. `--vpc-zone-identifier` lists the subnets (across AZs) where instances may launch.

### 📈 Step 7: Attach the Target-Tracking Scaling Policy (CPU 50%)

```bash
aws autoscaling put-scaling-policy --region "$AWS_REGION" \
  --auto-scaling-group-name $ASG_NAME \
  --policy-name cpu50-target-tracking \
  --policy-type TargetTrackingScaling \
  --target-tracking-configuration '{
    "PredefinedMetricSpecification": {
      "PredefinedMetricType": "ASGAverageCPUUtilization"
    },
    "TargetValue": 50.0
  }'
```

> **Why:** This is the rule that makes the group "scale based on CPU utilization." `aws autoscaling put-scaling-policy` attaches a scaling policy: `--auto-scaling-group-name` points the policy at the group from the previous step and `--policy-name` gives the policy a label; `--policy-type TargetTrackingScaling` is the simplest kind — you name a metric and a target value, and AWS adds or removes instances to keep the metric near that value, like a thermostat. `--target-tracking-configuration` holds that setup: the predefined metric `ASGAverageCPUUtilization` is the average CPU across the group's instances, and `TargetValue: 50.0` tells the ASG to keep it around 50% — if average CPU climbs above 50%, it launches another instance (up to `max-size` 2); if it drops well below, it removes one (down to `min-size` 1).

### ✅ Step 8: Verify

```bash
aws elbv2 wait target-in-service --region "$AWS_REGION" --target-group-arn $TG_ARN

ALB_DNS=$(aws elbv2 describe-load-balancers --region "$AWS_REGION" --names $ALB_NAME \
  --query "LoadBalancers[0].DNSName" --output text)

echo "ALB URL: http://$ALB_DNS"
curl -s -o /dev/null -w "HTTP %{http_code}\n" "http://$ALB_DNS"
```

```
HTTP 200
```

> **Why:** `aws elbv2 wait target-in-service` blocks until at least one instance passes the target group's health checks — until then the ALB has nothing healthy to route to. `aws elbv2 describe-load-balancers` then reads the ALB's public **DNS name** (`--names` selects it by name, `DNSName` is the address AWS assigns), which is the URL users hit. `curl` requests that URL: `-s` runs silently, `-o /dev/null` discards the page body, and `-w "HTTP %{http_code}"` prints only the HTTP status. A `200` confirms the ALB reached a healthy instance and Nginx served its default page. Right after an instance first passes its health check there can be a brief window where a request still returns `502 Bad Gateway` while Nginx finishes coming up — if that happens, simply re-run the same `curl` command a few seconds later; the wait already guarantees the instance is healthy, so this settles quickly.

## Best Practices

- **Use launch templates, not launch configurations.** Launch templates are the current standard; AWS recommends against launch configurations, which lack full Auto Scaling and EC2 functionality.
- **Let the ASG use ELB health checks (`--health-check-type ELB`).** This replaces instances that are running but not serving traffic, not just ones that failed the basic EC2 status check.
- **Spread instances and the ALB across multiple Availability Zones.** High availability comes from surviving the loss of an entire AZ; both the ASG (`--vpc-zone-identifier`) and the ALB (`--subnets`) should span at least two.
- **Prefer target-tracking scaling for simple metrics.** It's easier and safer than manual step-scaling for a single goal like "keep CPU near 50%," since AWS manages the alarms and adjustments for you.
- **Set a realistic health-check grace period.** Too short and healthy new instances get killed before their startup script (Nginx install) finishes; too long delays replacing genuinely broken instances.

### 📚 Official Documentation

- [create-launch-template — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/ec2/create-launch-template.html)
- [create-auto-scaling-group — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/autoscaling/create-auto-scaling-group.html)
- [create-target-group — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/elbv2/create-target-group.html)
- [Target tracking scaling policies for Amazon EC2 Auto Scaling](https://docs.aws.amazon.com/autoscaling/ec2/userguide/as-scaling-target-tracking.html)
- [Elastic Load Balancing and Amazon EC2 Auto Scaling](https://docs.aws.amazon.com/autoscaling/ec2/userguide/autoscaling-load-balancer.html)
