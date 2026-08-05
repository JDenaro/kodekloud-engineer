# Day 24: Setting Up an Application Load Balancer for an EC2 Instance

The Nautilus DevOps team is currently working on setting up a simple application on the AWS cloud. They aim to establish an Application Load Balancer (ALB) in front of an EC2 instance where an Nginx server is currently running. While the Nginx server currently serves a sample page, the team plans to deploy the actual application later.

Set up an Application Load Balancer named `datacenter-alb`.
Create a target group named `datacenter-tg`.
Create a security group named `datacenter-sg` to open port 80 for the public.
Attach this security group to the ALB.
The ALB should route traffic on port 80 to port 80 of the `datacenter-ec2` instance.
Make appropriate changes in the default security group attached to the EC2 instance if necessary.

## Specific Requirements:

1. Set up an Application Load Balancer named `datacenter-alb`.
2. Create a target group named `datacenter-tg`.
3. Create a security group named `datacenter-sg` to open port 80 for the public.
4. Attach this security group to the ALB.
5. The ALB should route traffic on port 80 to port 80 of the `datacenter-ec2` instance.
6. Make appropriate changes in the default security group attached to the EC2 instance if necessary.

## Solution

The ALB is the public entry point, while the EC2 instance remains reachable through a security-group reference from the ALB security group. The target group connects the listener to the instance, and its health check confirms that the Nginx service is responding on port 80.

### 📦 Variables

```bash
AWS_REGION="us-east-1"
ALB_NAME="datacenter-alb"
TARGET_GROUP_NAME="datacenter-tg"
ALB_SECURITY_GROUP_NAME="datacenter-sg"
INSTANCE_NAME="datacenter-ec2"
HTTP_PORT=80
```

### 🔎 Step 1: Discover the EC2 instance and its VPC resources

```bash
INSTANCE_ID=$(aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=$INSTANCE_NAME" "Name=instance-state-name,Values=pending,running,stopping,stopped" \
  --query "Reservations[0].Instances[0].InstanceId" \
  --output text)

VPC_ID=$(aws ec2 describe-instances \
  --instance-ids "$INSTANCE_ID" \
  --query "Reservations[0].Instances[0].VpcId" \
  --output text)

INSTANCE_SUBNET_ID=$(aws ec2 describe-instances \
  --instance-ids "$INSTANCE_ID" \
  --query "Reservations[0].Instances[0].SubnetId" \
  --output text)

EC2_DEFAULT_SG_ID=$(aws ec2 describe-instances \
  --instance-ids "$INSTANCE_ID" \
  --query "Reservations[0].Instances[0].SecurityGroups[?GroupName=='default'] | [0].GroupId" \
  --output text)

ALB_SUBNET_1=$(aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=$VPC_ID" "Name=state,Values=available" \
  --query "sort_by(Subnets[?DefaultForAz==\`true\` && MapPublicIpOnLaunch==\`true\`], &AvailabilityZone)[0].SubnetId" \
  --output text)

ALB_SUBNET_2=$(aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=$VPC_ID" "Name=state,Values=available" \
  --query "sort_by(Subnets[?DefaultForAz==\`true\` && MapPublicIpOnLaunch==\`true\`], &AvailabilityZone)[1].SubnetId" \
  --output text)

echo "INSTANCE_ID=$INSTANCE_ID"
echo "VPC_ID=$VPC_ID"
echo "INSTANCE_SUBNET_ID=$INSTANCE_SUBNET_ID"
echo "EC2_DEFAULT_SG_ID=$EC2_DEFAULT_SG_ID"
echo "ALB_SUBNET_1=$ALB_SUBNET_1"
echo "ALB_SUBNET_2=$ALB_SUBNET_2"
```

The existing instance and the network resources selected for the ALB were:

```text
INSTANCE_ID=i-02f7132cdda8c798f
VPC_ID=vpc-00ad8ea0a60f79e10
INSTANCE_SUBNET_ID=subnet-008438d93ace5b00f
EC2_DEFAULT_SG_ID=sg-0066bb4945479ea66
ALB_SUBNET_1=subnet-03c2acba54d5096e8
ALB_SUBNET_2=subnet-03697b9f84720e3f5
```

> **Why:** `describe-instances` retrieves the existing EC2 instance instead of creating another one. The `tag:Name` filter finds `datacenter-ec2`, while `instance-state-name` limits the result to states that can still serve traffic. `--instance-ids` narrows a second lookup to the discovered instance. `describe-subnets` lists subnets in the instance's VPC; the `vpc-id` and `state` filters limit the result, and the JMESPath expression selects default public subnets in different Availability Zones. `--query` extracts only the fields needed later, and `--output text` or `--output table` makes the values easy to read and reuse.

### 🛡️ Step 2: Reuse or configure the ALB security group

```bash
ALB_SG_ID=$(aws ec2 describe-security-groups \
  --filters "Name=vpc-id,Values=$VPC_ID" "Name=group-name,Values=$ALB_SECURITY_GROUP_NAME" \
  --query "SecurityGroups[0].GroupId" \
  --output text)
```

The security group already existed in the correct VPC, so the script reused it:

```text
Reusing ALB security group: sg-0d1a68c040d0dd249
```

The public HTTP rule was then added to `datacenter-sg`:

```bash
aws ec2 authorize-security-group-ingress \
  --group-id "$ALB_SG_ID" \
  --protocol tcp \
  --port "$HTTP_PORT" \
  --cidr 0.0.0.0/0
```

The command returned:

```text
{
    "Return": true,
    "SecurityGroupRules": [
        {
            "SecurityGroupRuleId": "sgr-0e9d49d781e22161f",
            "GroupId": "sg-0d1a68c040d0dd249",
            "GroupOwnerId": "884354549143",
            "IsEgress": false,
            "IpProtocol": "tcp",
            "FromPort": 80,
            "ToPort": 80,
            "CidrIpv4": "0.0.0.0/0",
            "SecurityGroupRuleArn": "arn:aws:ec2:us-east-1:884354549143:security-group-rule/sgr-0e9d49d781e22161f"
        }
    ]
}
Opened public TCP port 80 on datacenter-sg.
```

The default security group attached to the instance was also updated to allow port 80 only from the ALB security group:

```bash
aws ec2 authorize-security-group-ingress \
  --group-id "$EC2_DEFAULT_SG_ID" \
  --protocol tcp \
  --port "$HTTP_PORT" \
  --source-group "$ALB_SG_ID"
```

The rule was created successfully:

```text
{
    "Return": true,
    "SecurityGroupRules": [
        {
            "SecurityGroupRuleId": "sgr-0053b2da8f2c6e4a1",
            "GroupId": "sg-0066bb4945479ea66",
            "GroupOwnerId": "884354549143",
            "IsEgress": false,
            "IpProtocol": "tcp",
            "FromPort": 80,
            "ToPort": 80,
            "ReferencedGroupInfo": {
                "GroupId": "sg-0d1a68c040d0dd249",
                "UserId": "884354549143"
            },
            "SecurityGroupRuleArn": "arn:aws:ec2:us-east-1:884354549143:security-group-rule/sgr-0053b2da8f2c6e4a1"
        }
    ]
}
Allowed HTTP traffic from datacenter-sg to datacenter-ec2.
```

> **Why:** `describe-security-groups` locates the named security group in the instance's VPC. `authorize-security-group-ingress` adds an inbound rule. `--group-id` identifies the security group being changed, `--protocol tcp` selects TCP, `--port 80` limits the rule to HTTP, and `--cidr 0.0.0.0/0` permits public IPv4 clients to reach the ALB. For the EC2 default security group, `--source-group` uses a security-group reference instead of a public CIDR; this permits traffic from resources associated with `datacenter-sg` without exposing the instance directly to the internet.

### 🎯 Step 3: Create the target group and register the instance

```bash
aws elbv2 describe-target-groups \
  --names "$TARGET_GROUP_NAME" \
  --query "TargetGroups[0].TargetGroupArn" \
  --output text
```

No existing target group named `datacenter-tg` was found, so it was created in the instance's VPC:

```bash
TARGET_GROUP_ARN=$(aws elbv2 create-target-group \
  --name "$TARGET_GROUP_NAME" \
  --protocol HTTP \
  --port "$HTTP_PORT" \
  --vpc-id "$VPC_ID" \
  --target-type instance \
  --health-check-protocol HTTP \
  --health-check-port traffic-port \
  --health-check-path / \
  --query "TargetGroups[0].TargetGroupArn" \
  --output text)

aws elbv2 add-tags \
  --resource-arns "$TARGET_GROUP_ARN" \
  --tags "Key=Name,Value=$TARGET_GROUP_NAME"

aws elbv2 register-targets \
  --target-group-arn "$TARGET_GROUP_ARN" \
  --targets "Id=$INSTANCE_ID,Port=$HTTP_PORT"
```

The target group and registration succeeded:

```text
Created target group: arn:aws:elasticloadbalancing:us-east-1:884354549143:targetgroup/datacenter-tg/09308b3a2bb01d47
Registered datacenter-ec2 on target port 80.
```

> **Why:** `describe-target-groups` checks whether the named target group already exists. `create-target-group` creates the ALB's backend pool. `--name` assigns its name, `--protocol HTTP` and `--port 80` define the backend connection, `--vpc-id` places it in the instance's VPC, and `--target-type instance` means targets are registered by EC2 instance ID. The health-check parameters make the ALB request HTTP `/` on the target's traffic port. `add-tags` applies the `Name` tag using `--resource-arns` and `--tags`. `register-targets` adds the EC2 instance to the group; `--target-group-arn` identifies the group and `--targets` supplies the instance ID and backend port.

### ⚖️ Step 4: Create the Application Load Balancer

```bash
aws elbv2 describe-load-balancers \
  --names "$ALB_NAME" \
  --query "LoadBalancers[0].LoadBalancerArn" \
  --output text
```

No existing ALB named `datacenter-alb` was found, so it was created across two subnets and associated with `datacenter-sg`:

```bash
ALB_ARN=$(aws elbv2 create-load-balancer \
  --name "$ALB_NAME" \
  --subnets "$ALB_SUBNET_1" "$ALB_SUBNET_2" \
  --security-groups "$ALB_SG_ID" \
  --scheme internet-facing \
  --type application \
  --ip-address-type ipv4 \
  --query "LoadBalancers[0].LoadBalancerArn" \
  --output text)

aws elbv2 add-tags \
  --resource-arns "$ALB_ARN" \
  --tags "Key=Name,Value=$ALB_NAME"

aws elbv2 set-security-groups \
  --load-balancer-arn "$ALB_ARN" \
  --security-groups "$ALB_SG_ID"

aws elbv2 wait load-balancer-available \
  --load-balancer-arns "$ALB_ARN"
```

The ALB was created and became available:

```text
Created ALB: arn:aws:elasticloadbalancing:us-east-1:884354549143:loadbalancer/app/datacenter-alb/1602e9d99e333062
{
    "SecurityGroupIds": [
        "sg-0d1a68c040d0dd249"
    ]
}
ALB is available with datacenter-sg attached.
```

> **Why:** `describe-load-balancers` checks for an existing ALB by name. `create-load-balancer` creates the public Application Load Balancer. `--name` sets its name, `--subnets` places nodes in two Availability Zones, `--security-groups` attaches the ALB security group, `--scheme internet-facing` gives it a public-facing scheme, `--type application` selects an ALB, and `--ip-address-type ipv4` uses IPv4 addresses. `set-security-groups` makes the required security-group attachment explicit. `wait load-balancer-available` pauses until AWS reports that the ALB is ready; `--load-balancer-arns` identifies the ALB being checked.

### 🔗 Step 5: Create the HTTP listener and forward traffic

```bash
aws elbv2 describe-listeners \
  --load-balancer-arn "$ALB_ARN" \
  --query "Listeners[?Protocol=='HTTP' && Port==\`80\`] | [0].ListenerArn" \
  --output text

LISTENER_ARN=$(aws elbv2 create-listener \
  --load-balancer-arn "$ALB_ARN" \
  --protocol HTTP \
  --port "$HTTP_PORT" \
  --default-actions "Type=forward,TargetGroupArn=$TARGET_GROUP_ARN" \
  --query "Listeners[0].ListenerArn" \
  --output text)
```

No HTTP listener existed on the new ALB, so the listener was created on port 80:

```text
Created HTTP listener: arn:aws:elasticloadbalancing:us-east-1:884354549143:listener/app/datacenter-alb/1602e9d99e333062/d1aae58614fd03ac
```

> **Why:** `describe-listeners` checks the ALB's existing listeners. `create-listener` defines how the ALB accepts client traffic. `--load-balancer-arn` identifies the ALB, `--protocol HTTP` and `--port 80` accept public HTTP traffic, and `--default-actions` forwards requests to `datacenter-tg`. The target group ARN inside the action is the link between the listener and the EC2 backend.

### 🩺 Step 6: Wait for the Nginx target to become healthy

```bash
aws elbv2 wait target-in-service \
  --target-group-arn "$TARGET_GROUP_ARN" \
  --targets "Id=$INSTANCE_ID,Port=$HTTP_PORT"
```

The health check passed:

```text
EC2 target is healthy behind the ALB.
```

> **Why:** `wait target-in-service` repeatedly checks the target group's health until the specified target is in service. `--target-group-arn` identifies the backend group, and `--targets` identifies the instance and port being checked. A healthy result confirms that the ALB can reach the Nginx service through the security-group path on port 80.

### ✅ Step 7: Verify

```bash
ALB_DNS_NAME=$(aws elbv2 describe-load-balancers \
  --load-balancer-arns "$ALB_ARN" \
  --query "LoadBalancers[0].DNSName" \
  --output text)

aws elbv2 describe-load-balancers \
  --load-balancer-arns "$ALB_ARN" \
  --query "LoadBalancers[0].{Name:LoadBalancerName,DNSName:DNSName,Scheme:Scheme,State:State.Code,Type:Type,SecurityGroups:SecurityGroups}" \
  --output table

aws elbv2 describe-target-groups \
  --target-group-arns "$TARGET_GROUP_ARN" \
  --query "TargetGroups[0].{Name:TargetGroupName,Protocol:Protocol,Port:Port,TargetType:TargetType,VpcId:VpcId}" \
  --output table

aws elbv2 describe-listeners \
  --load-balancer-arn "$ALB_ARN" \
  --query "Listeners[?Port==\`80\`].{Port:Port,Protocol:Protocol,TargetGroupArn:DefaultActions[0].TargetGroupArn}" \
  --output table

aws elbv2 describe-target-health \
  --target-group-arn "$TARGET_GROUP_ARN" \
  --targets "Id=$INSTANCE_ID,Port=$HTTP_PORT" \
  --query "TargetHealthDescriptions[].{TargetId:Target.Id,TargetPort:Target.Port,Health:TargetHealth.State,Reason:TargetHealth.Reason}" \
  --output table

aws ec2 describe-security-groups \
  --group-ids "$ALB_SG_ID" "$EC2_DEFAULT_SG_ID" \
  --query "SecurityGroups[].{GroupId:GroupId,GroupName:GroupName,VpcId:VpcId,InboundRules:IpPermissions}" \
  --output json
```

The final verification confirmed the complete ALB path:

```text
ALB_DNS_NAME=datacenter-alb-157885296.us-east-1.elb.amazonaws.com
---------------------------------------------------------------------
|                       DescribeLoadBalancers                       |
+---------+---------------------------------------------------------+
|  DNSName|  datacenter-alb-157885296.us-east-1.elb.amazonaws.com   |
|  Name   |  datacenter-alb                                         |
|  Scheme |  internet-facing                                        |
|  State  |  active                                                 |
|  Type   |  application                                            |
+---------+---------------------------------------------------------+
||                         SecurityGroups                          ||
|+-----------------------------------------------------------------+|
||  sg-0d1a68c040d0dd249                                           ||
|+-----------------------------------------------------------------+|
------------------------------------------------------------------------------
|                            DescribeTargetGroups                            |
+---------------+-------+-----------+-------------+--------------------------+
|     Name      | Port  | Protocol  | TargetType  |          VpcId           |
+---------------+-------+-----------+-------------+--------------------------+
|  datacenter-tg|  80   |  HTTP     |  instance   |  vpc-00ad8ea0a60f79e10   |
+---------------+-------+-----------+-------------+--------------------------+
----------------------------------------------------------------------------------------------------------------------
|                                                  DescribeListeners                                                 |
+----------------+---------------------------------------------------------------------------------------------------+
|  Port          |  80                                                                                               |
|  Protocol      |  HTTP                                                                                             |
|  TargetGroupArn|  arn:aws:elasticloadbalancing:us-east-1:884354549143:targetgroup/datacenter-tg/09308b3a2bb01d47   |
+----------------+---------------------------------------------------------------------------------------------------+
------------------------------------------------------------
|                   DescribeTargetHealth                   |
+---------+---------+-----------------------+--------------+
| Health  | Reason  |       TargetId        | TargetPort   |
+---------+---------+-----------------------+--------------+
|  healthy|  None   |  i-02f7132cdda8c798f  |  80          |
+---------+---------+-----------------------+--------------+
```

The security-group verification also confirmed that `datacenter-sg` accepts public TCP port 80 and the EC2 default security group accepts port 80 only from `sg-0d1a68c040d0dd249`:

```text
[
    {
        "GroupId": "sg-0066bb4945479ea66",
        "GroupName": "default",
        "VpcId": "vpc-00ad8ea0a60f79e10",
        "InboundRules": [
            {
                "IpProtocol": "tcp",
                "FromPort": 80,
                "ToPort": 80,
                "UserIdGroupPairs": [
                    {
                        "UserId": "884354549143",
                        "GroupId": "sg-0d1a68c040d0dd249"
                    }
                ],
                "IpRanges": [],
                "Ipv6Ranges": [],
                "PrefixListIds": []
            },
            {
                "IpProtocol": "-1",
                "UserIdGroupPairs": [
                    {
                        "UserId": "884354549143",
                        "GroupId": "sg-0066bb4945479ea66"
                    }
                ],
                "IpRanges": [],
                "Ipv6Ranges": [],
                "PrefixListIds": []
            }
        ]
    },
    {
        "GroupId": "sg-0d1a68c040d0dd249",
        "GroupName": "datacenter-sg",
        "VpcId": "vpc-00ad8ea0a60f79e10",
        "InboundRules": [
            {
                "IpProtocol": "tcp",
                "FromPort": 80,
                "ToPort": 80,
                "IpRanges": [
                    {
                        "CidrIp": "0.0.0.0/0"
                    }
                ],
                "Ipv6Ranges": [],
                "PrefixListIds": []
            }
        ]
    }
]
ALB setup completed successfully.
```

> **Why:** `describe-load-balancers` confirms the ALB DNS name, type, scheme, state, and attached security group. `describe-target-groups` confirms that `datacenter-tg` uses HTTP on port 80 in the expected VPC. `describe-listeners` confirms that the port-80 HTTP listener forwards to the target group. `describe-target-health` confirms the instance is `healthy`. The final `describe-security-groups` call displays both inbound paths: public traffic stops at the ALB, while the EC2 instance accepts HTTP only from the ALB security group.

## Best Practices

- **Expose only the load balancer.** Public HTTP access belongs on `datacenter-sg`; the EC2 security group should trust the ALB security group instead of `0.0.0.0/0`.
- **Use multiple Availability Zones.** Deploying the ALB across two subnets improves resilience if one Availability Zone becomes unavailable.
- **Use target health checks.** The HTTP health check verifies that the Nginx service is actually responding before the ALB sends it traffic.
- **Keep listener and target ports explicit.** The listener accepts client traffic on port 80 and the target group forwards it to port 80 on `datacenter-ec2`.
- **Tag load-balancing resources.** Name tags make the ALB and target group easier to identify during operations and cleanup.
- **Prefer HTTPS for production.** This lab requires HTTP on port 80, but a real public application should normally use an HTTPS listener with a certificate and redirect HTTP to HTTPS.

### 📚 Official Documentation

- [describe-instances — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/ec2/describe-instances.html)
- [describe-subnets — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/ec2/describe-subnets.html)
- [describe-security-groups — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/ec2/describe-security-groups.html)
- [create-security-group — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/ec2/create-security-group.html)
- [authorize-security-group-ingress — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/ec2/authorize-security-group-ingress.html)
- [create-tags — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/ec2/create-tags.html)
- [create-target-group — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/elbv2/create-target-group.html)
- [register-targets — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/elbv2/register-targets.html)
- [describe-target-health — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/elbv2/describe-target-health.html)
- [describe-target-groups — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/elbv2/describe-target-groups.html)
- [add-tags — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/elbv2/add-tags.html)
- [create-load-balancer — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/elbv2/create-load-balancer.html)
- [describe-load-balancers — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/elbv2/describe-load-balancers.html)
- [set-security-groups — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/elbv2/set-security-groups.html)
- [wait load-balancer-available — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/elbv2/wait/load-balancer-available.html)
- [create-listener — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/elbv2/create-listener.html)
- [describe-listeners — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/elbv2/describe-listeners.html)
- [wait target-in-service — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/elbv2/wait/target-in-service.html)
