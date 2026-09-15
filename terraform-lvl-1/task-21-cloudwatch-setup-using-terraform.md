# Task 21: CloudWatch Setup Using Terraform

The Nautilus DevOps team needs to set up CloudWatch logging for their application. They need to create a CloudWatch log group and log stream with the following specifications:

1) The log group name should be devops-log-group.

2) The log stream name should be devops-log-stream.

Use Terraform to create the CloudWatch log group and log stream. The Terraform working directory is /home/bob/terraform. Create the main.tf file (do not create a different .tf file) to accomplish this task.

Note: Right-click under the EXPLORER section in VS Code and select Open in Integrated Terminal to launch the terminal.

## Task Requirements

1. The log group name should be `devops-log-group`.
2. The log stream name should be `devops-log-stream`.
3. Use Terraform to create the CloudWatch log group and log stream.
4. The Terraform working directory is `/home/bob/terraform`. Create the `main.tf` file (do not create a different `.tf` file) to accomplish this task.
5. Right-click under the **EXPLORER** section in VS Code and select Open in Integrated Terminal to launch the terminal.

## Solution

This Terraform lab starts with a fresh workspace, so `main.tf` must be created manually if it is absent. The configuration creates one CloudWatch log group and one log stream inside that group. The existing `provider.tf` supplies the AWS provider configuration and should remain unchanged.

A **log group** is the main CloudWatch Logs container for related logs. A **log stream** is an ordered sequence of log events from one source inside a log group. This task creates the containers, but it does not send log events to the stream.

### 📦 Variables

```bash
AWS_REGION="us-east-1"
LOG_GROUP_NAME="devops-log-group"
LOG_STREAM_NAME="devops-log-stream"
```

> **Why:** These values record the AWS region and exact resource names required by the challenge. They are reference values for the guide; the Terraform configuration uses the literal names directly. The existing `provider.tf` supplies the AWS provider configuration.

### 📝 Step 1: Create `main.tf`

In the VS Code Explorer, open `/home/bob/terraform` and create `main.tf` with this content:

```hcl
resource "aws_cloudwatch_log_group" "devops_log_group" {
  name = "devops-log-group"
}

resource "aws_cloudwatch_log_stream" "devops_log_stream" {
  name           = "devops-log-stream"
  log_group_name = aws_cloudwatch_log_group.devops_log_group.name
}
```

> **Why:** `aws_cloudwatch_log_group` creates the main CloudWatch Logs container. `aws_cloudwatch_log_stream` creates a stream inside that container. The reference to `aws_cloudwatch_log_group.devops_log_group.name` connects the stream to the group and makes Terraform create the group first. No retention period, tags, or log events are required by this challenge.

### 📂 Step 2: Open the Terraform working directory

From the integrated terminal, run:

```bash
cd /home/bob/terraform
```

> **Why:** `cd` changes the current terminal directory. Terraform commands must be run from the directory containing `main.tf` and the existing provider configuration.

### ⚙️ Step 3: Initialize Terraform

```bash
terraform init
```

The lab returned:

```text
Terraform has been successfully initialized!
```

> **Why:** `terraform init` prepares the working directory and installs or reuses the AWS provider required by the CloudWatch resources.

### 🚀 Step 4: Create the log group and log stream

```bash
terraform apply --auto-approve
```

The lab returned:

```text
Plan: 2 to add, 0 to change, 0 to destroy.
aws_cloudwatch_log_group.devops_log_group: Creating...
aws_cloudwatch_log_group.devops_log_group: Creation complete after 1s [id=devops-log-group]
aws_cloudwatch_log_stream.devops_log_stream: Creating...
aws_cloudwatch_log_stream.devops_log_stream: Creation complete after 0s [id=devops-log-stream]

Apply complete! Resources: 2 added, 0 changed, 0 destroyed.
```

> **Why:** `terraform apply` compares the desired configuration with Terraform state and creates both CloudWatch resources. `--auto-approve` accepts the generated execution plan without requesting an additional confirmation.

### ✅ Step 5: Verify the log group

```bash
aws logs describe-log-groups \
  --log-group-name-prefix "devops-log-group" \
  --region "us-east-1" \
  --query 'logGroups[?logGroupName==`devops-log-group`].logGroupName' \
  --output text
```

The lab returned:

```text
devops-log-group
```

> **Why:** `aws logs describe-log-groups` lists CloudWatch log groups. `--log-group-name-prefix` narrows the search, `--region` targets `us-east-1`, `--query` selects the exact group name, and `--output text` prints only the matching value.

### 🔎 Step 6: Verify the log stream

```bash
aws logs describe-log-streams \
  --log-group-name "devops-log-group" \
  --log-stream-name-prefix "devops-log-stream" \
  --region "us-east-1" \
  --query 'logStreams[?logStreamName==`devops-log-stream`].logStreamName' \
  --output text
```

The expected result is:

```text
devops-log-stream
```

> **Why:** `aws logs describe-log-streams` lists streams within a log group. `--log-group-name` identifies the parent group, `--log-stream-name-prefix` narrows the stream search, `--region` selects the AWS region, `--query` selects the exact stream name, and `--output text` prints the result directly.

### 🧪 Step 7: Confirm Terraform has no pending changes

```bash
terraform plan
```

The expected result is:

```text
No changes. Your infrastructure matches the configuration.
```

> **Why:** `terraform plan` compares the configuration, Terraform state, and remote resources without modifying them. `No changes` confirms that the log group and log stream match `main.tf`.

## Best Practices

- **Keep the hierarchy explicit.** Create the log stream with a reference to the log group so Terraform understands their dependency.
- **Choose retention deliberately.** This challenge does not specify a retention period, so the default is retained; production systems should define retention according to operational and compliance needs.
- **Separate containers from log ingestion.** Creating a log group and stream does not send application events; an agent, application, or API must publish those events separately.
- **Verify both resources.** Confirm the group and the stream independently because a stream is always associated with a parent log group.
- **Treat each Terraform lab as independent.** Create `main.tf` when it is absent and do not assume state, provider locks, or resources from another lab remain available.

### 📚 Official Documentation

- [Terraform AWS provider `aws_cloudwatch_log_group` resource](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/cloudwatch_log_group)
- [Terraform AWS provider `aws_cloudwatch_log_stream` resource](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/cloudwatch_log_stream)
- [AWS CloudWatch Logs concepts](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/WhatIsCloudWatchLogs.html)
- [AWS CLI `describe-log-groups` command](https://docs.aws.amazon.com/cli/latest/reference/logs/describe-log-groups.html)
- [AWS CLI `describe-log-streams` command](https://docs.aws.amazon.com/cli/latest/reference/logs/describe-log-streams.html)
- [Terraform `init` command](https://developer.hashicorp.com/terraform/cli/commands/init)
- [Terraform `apply` command](https://developer.hashicorp.com/terraform/cli/commands/apply)
- [Terraform `plan` command](https://developer.hashicorp.com/terraform/cli/commands/plan)
