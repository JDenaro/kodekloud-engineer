# Day 25: Setting Up an EC2 Instance and CloudWatch Alarm

The Nautilus DevOps team has been tasked with setting up an EC2 instance for their application. To ensure the application performs optimally, they also need to create a CloudWatch alarm to monitor the instance's CPU utilization. The alarm should trigger if the CPU utilization exceeds 90% for one consecutive 5-minute period. To send notifications, use the SNS topic named `devops-sns-topic` which is already created.

**Launch EC2 Instance:** Create an EC2 instance named `devops-ec2` using any appropriate Ubuntu AMI.

**Create CloudWatch Alarm:** Create a CloudWatch alarm named `devops-alarm` with the following specifications:

Statistic: Average
Metric: CPU Utilization
Threshold: >= 90% for 1 consecutive 5-minute period.
Alarm Actions: Send a notification to `devops-sns-topic`.

## Specific Requirements:

1. **Launch EC2 Instance:** Create an EC2 instance named `devops-ec2` using any appropriate Ubuntu AMI.
2. **Create CloudWatch Alarm:** Create a CloudWatch alarm named `devops-alarm` with the following specifications:
3. Statistic: Average
4. Metric: CPU Utilization
5. Threshold: >= 90% for 1 consecutive 5-minute period.
6. Alarm Actions: Send a notification to `devops-sns-topic`.

## Solution

The instance publishes the `CPUUtilization` metric to the `AWS/EC2` namespace. The CloudWatch alarm watches that metric for the specific instance and sends its state-change notification to the existing SNS topic. A new alarm can initially report `INSUFFICIENT_DATA` while CloudWatch waits for the first complete metric period; the configuration can still be verified immediately.

### 📦 Variables

```bash
AWS_REGION="us-east-1"
INSTANCE_NAME="devops-ec2"
ALARM_NAME="devops-alarm"
SNS_TOPIC_NAME="devops-sns-topic"
METRIC_NAMESPACE="AWS/EC2"
METRIC_NAME="CPUUtilization"
STATISTIC="Average"
PERIOD=300
THRESHOLD=90
EVALUATION_PERIODS=1
DATAPOINTS_TO_ALARM=1
COMPARISON_OPERATOR="GreaterThanOrEqualToThreshold"
```

### 🔎 Step 1: Discover the default VPC and subnet

First, identify the default VPC that will host the instance:

```bash
aws ec2 describe-vpcs \
  --filters "Name=isDefault,Values=true" \
  --query "Vpcs[0].VpcId" \
  --output text
```

The lab returned:

```text
vpc-049af305978b44eb7
```

Now find a default subnet in that VPC:

```bash
aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=vpc-049af305978b44eb7" "Name=default-for-az,Values=true" \
  --query "Subnets[0].SubnetId" \
  --output text
```

The selected subnet was:

```text
subnet-01474bdbdad3ab80e
```

> **Why:** `aws ec2 describe-vpcs` reads the VPC metadata. The `--filters` parameter limits the result to the VPC whose `isDefault` property is `true`. `--query` uses JMESPath to extract only the first VPC ID, and `--output text` prints the ID without the surrounding JSON. `describe-subnets` lists subnets, while the `vpc-id` filter keeps the lookup inside the selected VPC and `default-for-az` selects a subnet that AWS marks as the default for an Availability Zone.

### 🔐 Step 2: Discover the default security group

```bash
aws ec2 describe-security-groups \
  --filters "Name=vpc-id,Values=vpc-049af305978b44eb7" "Name=group-name,Values=default" \
  --query "SecurityGroups[0].GroupId" \
  --output text
```

The instance used the default security group:

```text
sg-0f72d1c8b0871bd29
```

> **Why:** `describe-security-groups` retrieves security-group metadata. The `vpc-id` filter selects groups in the instance VPC, and `group-name=default` selects the VPC's default security group. The resulting group ID is passed to the instance launch command through `--security-group-ids`.

### 🖼️ Step 3: Find an appropriate Ubuntu AMI

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

The newest available compatible Ubuntu AMI was:

```text
ami-0d001f8052688dc45
```

> **Why:** `describe-images` searches the AMI catalog. `--owners 099720109477` limits the search to Canonical's public Ubuntu images. The `--filters` parameter selects an Ubuntu 22.04 image that is available, uses the `x86_64` architecture, has an EBS root device, and uses HVM virtualization. `sort_by(Images, &CreationDate)[-1]` selects the newest matching image, while `--query` extracts its ID for the launch request.

### 🔍 Step 4: Check whether the EC2 instance already exists

```bash
aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=devops-ec2" "Name=instance-state-name,Values=pending,running,stopping,stopped" \
  --query "Reservations[0].Instances[0].InstanceId" \
  --output text
```

No usable instance named `devops-ec2` existed, so the instance had to be created.

> **Why:** `describe-instances` retrieves EC2 instance details. The `tag:Name` filter searches for the challenge's logical name, and `instance-state-name` avoids treating terminated instances as reusable resources. Checking first prevents accidentally creating a duplicate instance.

### 🚀 Step 5: Launch and tag the EC2 instance

```bash
aws ec2 run-instances \
  --image-id ami-0d001f8052688dc45 \
  --instance-type t2.micro \
  --subnet-id subnet-01474bdbdad3ab80e \
  --security-group-ids sg-0f72d1c8b0871bd29 \
  --count 1 \
  --query "Instances[0].InstanceId" \
  --output text
```

AWS created the instance with ID:

```text
i-01b5b46f994d882e2
```

Apply the required `Name` tag in a separate command:

```bash
aws ec2 create-tags \
  --resources i-01b5b46f994d882e2 \
  --tags "Key=Name,Value=devops-ec2"
```

> **Why:** `run-instances` launches one or more EC2 instances. `--image-id` selects the Ubuntu AMI, `--instance-type t2.micro` chooses the lab-sized instance type, `--subnet-id` places the instance in the discovered subnet, and `--security-group-ids` attaches the discovered security group. `--count 1` requests exactly one instance. `--query` and `--output text` return only the new instance ID. `create-tags` adds metadata after creation; `--resources` identifies the instance and `--tags` assigns the `Name` key and `devops-ec2` value.

### ⏳ Step 6: Wait until the instance is running

```bash
aws ec2 wait instance-running \
  --instance-ids i-01b5b46f994d882e2
```

The waiter completed successfully, and the final verification reported the instance as `running`.

> **Why:** `wait instance-running` polls the EC2 state until the specified instance reaches `running`. `--instance-ids` identifies the instance to check. Waiting avoids configuring the alarm against an instance launch request that has not finished yet.

### 📣 Step 7: Find the existing SNS topic

```bash
aws sns list-topics \
  --query "Topics[?contains(TopicArn, ':devops-sns-topic')].TopicArn | [0]" \
  --output text
```

The existing topic was found with this ARN:

```text
arn:aws:sns:us-east-1:171784365236:devops-sns-topic
```

> **Why:** `list-topics` lists the SNS topics available in the account and region. The `--query` expression filters the returned `TopicArn` values to the topic whose name is `devops-sns-topic`; CloudWatch alarm actions require the topic ARN, not only the friendly topic name. `--output text` makes the ARN easy to pass to the alarm command.

### 🔔 Step 8: Check whether the CloudWatch alarm already exists

```bash
aws cloudwatch describe-alarms \
  --alarm-names devops-alarm \
  --query "MetricAlarms[0].AlarmArn" \
  --output text
```

The alarm did not exist, so it was created in the next step.

> **Why:** `describe-alarms` retrieves CloudWatch alarm definitions. `--alarm-names` narrows the lookup to `devops-alarm`, and `--query` extracts its ARN when present. This pre-check makes the create-or-reuse decision explicit; the same lookup can also detect an existing alarm that needs to be updated.

### ⚙️ Step 9: Create the CPU utilization alarm

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name devops-alarm \
  --alarm-description "Alarm when devops-ec2 CPU utilization is at least 90 percent for one consecutive 5-minute period." \
  --metric-name CPUUtilization \
  --namespace AWS/EC2 \
  --statistic Average \
  --period 300 \
  --threshold 90 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --dimensions "Name=InstanceId,Value=i-01b5b46f994d882e2" \
  --evaluation-periods 1 \
  --datapoints-to-alarm 1 \
  --alarm-actions arn:aws:sns:us-east-1:171784365236:devops-sns-topic \
  --unit Percent
```

The command completed successfully:

```text
Configured alarm: devops-alarm
```

> **Why:** `put-metric-alarm` creates or updates a metric alarm. `--alarm-name` sets the alarm name, and `--alarm-description` documents its purpose. `--metric-name CPUUtilization` selects the EC2 CPU metric in the `--namespace AWS/EC2` namespace. `--statistic Average` evaluates the average value during each period. `--period 300` makes each period five minutes, while `--threshold 90` sets the numeric limit. `--comparison-operator GreaterThanOrEqualToThreshold` means the alarm condition is met at 90% or higher. `--dimensions` scopes the metric to instance `i-01b5b46f994d882e2` instead of all EC2 instances. `--evaluation-periods 1` requires one period to be evaluated, and `--datapoints-to-alarm 1` requires that one datapoint to meet the condition. `--alarm-actions` supplies the SNS topic ARN that receives the notification. `--unit Percent` states that the metric is measured as a percentage.

### ✅ Step 10: Verify

Verify the EC2 instance:

```bash
aws ec2 describe-instances \
  --instance-ids i-01b5b46f994d882e2 \
  --query "Reservations[0].Instances[0].{ImageId:ImageId,InstanceId:InstanceId,InstanceType:InstanceType,Name:Tags[?Key=='Name']|[0].Value,State:State.Name}" \
  --output table
```

```text
-------------------------------------------
|            DescribeInstances            |
+---------------+-------------------------+
|  ImageId      |  ami-0d001f8052688dc45  |
|  InstanceId   |  i-01b5b46f994d882e2    |
|  InstanceType |  t2.micro               |
|  Name         |  devops-ec2             |
|  State        |  running                |
+---------------+-------------------------+
```

Verify the SNS topic:

```bash
aws sns list-topics \
  --query "Topics[?TopicArn=='arn:aws:sns:us-east-1:171784365236:devops-sns-topic'].{TopicArn:TopicArn}" \
  --output table
```

```text
---------------------------------------------------------
|                      ListTopics                       |
+-------------------------------------------------------+
|                       TopicArn                        |
+-------------------------------------------------------+
|  arn:aws:sns:us-east-1:171784365236:devops-sns-topic  |
+-------------------------------------------------------+
```

Finally, verify the alarm configuration:

```bash
aws cloudwatch describe-alarms \
  --alarm-names devops-alarm \
  --query "MetricAlarms[0].{AlarmName:AlarmName,Comparison:ComparisonOperator,DatapointsToAlarm:DatapointsToAlarm,EvaluationPeriods:EvaluationPeriods,InstanceId:Dimensions[?Name=='InstanceId']|[0].Value,Metric:MetricName,Namespace:Namespace,Period:Period,State:StateValue,Statistic:Statistic,Threshold:Threshold,AlarmActions:AlarmActions}" \
  --output table
```

```text
-----------------------------------------------------------
|                     DescribeAlarms                      |
+---------------------+-----------------------------------+
|  AlarmName          |  devops-alarm                     |
|  Comparison         |  GreaterThanOrEqualToThreshold    |
|  DatapointsToAlarm  |  1                                |
|  EvaluationPeriods  |  1                                |
|  InstanceId         |  i-01b5b46f994d882e2              |
|  Metric             |  CPUUtilization                   |
|  Namespace          |  AWS/EC2                          |
|  Period             |  300                              |
|  State              |  INSUFFICIENT_DATA                |
|  Statistic          |  Average                          |
|  Threshold          |  90.0                             |
+---------------------+-----------------------------------+
||                     AlarmActions                      ||
|+-------------------------------------------------------+|
||  arn:aws:sns:us-east-1:171784365236:devops-sns-topic  ||
|+-------------------------------------------------------+|
```

The instance is `running`, the alarm monitors the correct instance's average `CPUUtilization`, and the alarm action points to `devops-sns-topic`. The initial `INSUFFICIENT_DATA` state is expected until CloudWatch receives enough metric data for the first evaluation period; it does not invalidate the alarm configuration.

> **Why:** `describe-instances` confirms the image, instance ID, type, `Name` tag, and runtime state. Its `--instance-ids` parameter targets one known instance, while the `--query` projection selects only the fields needed for verification. The SNS `list-topics` query confirms that the exact topic ARN exists. The final `describe-alarms` query confirms the metric, namespace, statistic, period, threshold, comparison, evaluation settings, instance dimension, and SNS action. `--output table` presents the result in a readable verification table.

## Best Practices

- **Check before creating.** Look up named resources first so rerunning the procedure does not create duplicate instances or alarms.
- **Use a metric dimension.** The `InstanceId` dimension ensures the alarm monitors `devops-ec2`, not aggregate CPU utilization from unrelated instances.
- **Match the period to the requirement.** A `300`-second period with one evaluation period represents one consecutive five-minute measurement window.
- **Use the SNS ARN for actions.** CloudWatch alarm actions reference the topic ARN discovered from SNS, which avoids ambiguity when topic names are reused across regions or accounts.
- **Interpret the initial state correctly.** A new alarm can remain `INSUFFICIENT_DATA` until CloudWatch receives the metric data required for its first evaluation.

### 📚 Official Documentation

- [Amazon EC2 examples using the AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/cli_ec2_code_examples.html)
- [describe-vpcs — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/ec2/describe-vpcs.html)
- [describe-subnets — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/ec2/describe-subnets.html)
- [describe-security-groups — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/ec2/describe-security-groups.html)
- [describe-images — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/ec2/describe-images.html)
- [run-instances — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/ec2/run-instances.html)
- [create-tags — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/ec2/create-tags.html)
- [instance-running waiter — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/ec2/wait/instance-running.html)
- [Amazon SNS examples using the AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/cli_sns_code_examples.html)
- [put-metric-alarm — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/cloudwatch/put-metric-alarm.html)
- [describe-alarms — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/cloudwatch/describe-alarms.html)
- [Create a CPU usage alarm](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/US_AlarmAtThresholdEC2.html)
