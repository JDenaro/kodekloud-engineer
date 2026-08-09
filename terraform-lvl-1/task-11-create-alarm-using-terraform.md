# Task 11: Create Alarm Using Terraform

The Nautilus DevOps team is setting up monitoring in their AWS account. As part of this, they need to create a CloudWatch alarm.

Using Terraform, perform the following:

**Task Details:**
Create a **CloudWatch alarm** named datacenter-alarm.
The alarm should monitor **CPU utilization** of an EC2 instance.
Trigger the alarm when **CPU utilization exceeds 80%**.
Set the **evaluation period** to **5 minutes**.
Use a **single evaluation period**.
Ensure that the entire configuration is implemented using Terraform. The Terraform working directory is /home/bob/terraform. Create the main.tf file (do not create a different .tf file) to accomplish this task.

Note: Right-click under the **EXPLORER** section in VS Code and select Open in Integrated Terminal to launch the terminal.

## Task Requirements

1. Create a **CloudWatch alarm** named `datacenter-alarm`.
2. The alarm should monitor **CPU utilization** of an EC2 instance.
3. Trigger the alarm when **CPU utilization exceeds 80%**.
4. Set the **evaluation period** to **5 minutes**.
5. Use a **single evaluation period**.
6. Ensure that the entire configuration is implemented using Terraform.
7. The Terraform working directory is `/home/bob/terraform`. Create the `main.tf` file (do not create a different `.tf` file) to accomplish this task.
8. Note: Right-click under the **EXPLORER** section in VS Code and select Open in Integrated Terminal to launch the terminal.

## Solution

This Terraform lab started without a `main.tf`, so it had to be created manually from the VS Code Explorer. Before writing the alarm, the AWS CLI lookup confirmed that no EC2 instances existed in `us-east-1`. Therefore, the alarm was created without an `InstanceId` dimension; creating an instance just to supply that dimension would have added a resource not requested by the challenge. The alarm is valid, but because no matching EC2 CPU metric is being published, its state remains `INSUFFICIENT_DATA` in this lab.

The existing `provider.tf` was left unchanged.

### 🔎 Step 1: Check for an existing EC2 instance

From the integrated terminal, run:

```bash
aws ec2 describe-instances \
  --region us-east-1 \
  --filters "Name=instance-state-name,Values=running" \
  --query 'Reservations[].Instances[].[InstanceId,Tags[?Key==`Name`].Value|[0]]' \
  --output table
```

No rows were returned. A second lookup without the running-state filter also returned no instances, so there was no existing instance to identify with an `InstanceId` dimension.

> **Why:** `aws ec2 describe-instances` lists EC2 instances. `--region us-east-1` limits the lookup to the lab region. `--filters` requests only running instances, while `--query` selects the instance ID and optional `Name` tag from the response. `--output table` makes the result easier to read. This pre-check prevents us from inventing or creating an EC2 resource that the task does not request.

### 📝 Step 2: Create `main.tf`

In the VS Code Explorer, create `/home/bob/terraform/main.tf` and add:

```hcl
resource "aws_cloudwatch_metric_alarm" "datacenter_alarm" {
  alarm_name          = "datacenter-alarm"
  alarm_description   = "Monitors EC2 CPU utilization"
  metric_name         = "CPUUtilization"
  namespace           = "AWS/EC2"
  statistic           = "Average"
  period              = 300
  evaluation_periods  = 1
  threshold           = 80
  comparison_operator = "GreaterThanThreshold"
}
```

> **Why:** `aws_cloudwatch_metric_alarm` creates a CloudWatch metric alarm. `alarm_name` gives it the required name, and `alarm_description` documents its purpose. `metric_name` selects the EC2 `CPUUtilization` metric, while `namespace` selects the `AWS/EC2` metric namespace. `statistic = "Average"` evaluates the average CPU value. `period = 300` sets each metric period to 300 seconds, which is five minutes. `evaluation_periods = 1` requires one period, satisfying the single-evaluation requirement. `threshold = 80` sets the limit, and `comparison_operator = "GreaterThanThreshold"` triggers the alarm when the average CPU utilization is greater than 80 percent. No `dimensions` block is included because the lab has no EC2 instance; in a real per-instance alarm, `InstanceId` would identify the monitored instance.

### ⚙️ Step 3: Initialize Terraform

From `/home/bob/terraform`, run:

```bash
terraform init
```

Successful output includes:

```text
Terraform has been successfully initialized!
```

> **Why:** `terraform init` prepares the working directory and installs or reuses the provider required by the configuration. It also prepares Terraform to use the existing provider configuration and dependency lock file when available.

### 🚀 Step 4: Create the alarm

Run:

```bash
terraform apply --auto-approve
```

The lab returned:

```text
Plan: 1 to add, 0 to change, 0 to destroy.
aws_cloudwatch_metric_alarm.datacenter_alarm: Creating...
aws_cloudwatch_metric_alarm.datacenter_alarm: Creation complete after 0s [id=datacenter-alarm]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

> **Why:** `terraform apply` compares the configuration with Terraform state and creates the required alarm. `--auto-approve` accepts the generated plan without an additional confirmation prompt. The plan shows that exactly one alarm was added and no unrelated resources were changed.

### ✅ Step 5: Verify

Run:

```bash
aws cloudwatch describe-alarms \
  --region us-east-1 \
  --alarm-names datacenter-alarm \
  --query 'MetricAlarms[0].[AlarmName,MetricName,Namespace,Period,EvaluationPeriods,Threshold,StateValue]' \
  --output table
```

The lab returned:

```text
-----------------------
|   DescribeAlarms    |
+---------------------+
|  datacenter-alarm   |
|  CPUUtilization     |
|  AWS/EC2            |
|  300                |
|  1                  |
|  80.0               |
|  INSUFFICIENT_DATA  |
+---------------------+
```

The alarm exists with the requested metric, namespace, five-minute period, single evaluation period, and threshold of 80. `INSUFFICIENT_DATA` is expected here because the lab contained no EC2 instance publishing CPU metrics; it does not mean that Terraform failed to create the alarm.

> **Why:** `aws cloudwatch describe-alarms` retrieves CloudWatch alarm details. `--region us-east-1` selects the lab region, and `--alarm-names` limits the response to `datacenter-alarm`. `--query` extracts only the fields needed to validate the requirements, while `--output table` formats those values for quick inspection. CloudWatch initially uses `INSUFFICIENT_DATA` when it has not received enough matching metric data.

## Best Practices

- **Check dependencies before creating resources.** Confirm whether the EC2 instance required by a metric already exists before adding resources outside the task.
- **Use seconds for CloudWatch periods.** A value of `300` represents five minutes, while `evaluation_periods = 1` means that one five-minute data point is evaluated.
- **Use dimensions for per-instance monitoring.** In a production alarm, add the target EC2 `InstanceId` as a dimension so the alarm monitors one specific instance instead of an undifferentiated metric.
- **Distinguish creation from alarm state.** A successfully created alarm can remain `INSUFFICIENT_DATA` when no matching metric data is available.
- **Treat each Terraform lab as independent.** Create `main.tf` when it is absent and do not assume files, state, providers, or resources from an earlier lab remain available.

### 📚 Official Documentation

- [Terraform AWS provider `aws_cloudwatch_metric_alarm` resource](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/cloudwatch_metric_alarm)
- [AWS CloudWatch `PutMetricAlarm` API](https://docs.aws.amazon.com/AmazonCloudWatch/latest/APIReference/API_PutMetricAlarm.html)
- [AWS CLI `describe-alarms` command](https://docs.aws.amazon.com/cli/latest/reference/cloudwatch/describe-alarms.html)
- [Terraform `init` command](https://developer.hashicorp.com/terraform/cli/commands/init)
- [Terraform `apply` command](https://developer.hashicorp.com/terraform/cli/commands/apply)
