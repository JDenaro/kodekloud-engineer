# Task 18: Create Kinesis Stream Using Terraform

The Nautilus DevOps team needs to create an AWS Kinesis data stream for real-time data processing. This stream will be used to ingest and process large volumes of streaming data, which will then be consumed by various applications for analytics and real-time decision-making.

The stream should be named devops-stream.

Use Terraform to create this Kinesis stream.

The Terraform working directory is /home/bob/terraform. Create the main.tf file (do not create a different .tf file) to accomplish this task.

Note:

Right-click under the EXPLORER section in VS Code and select Open in Integrated Terminal to launch the terminal.
Before submitting the task, ensure that terraform plan returns No changes. Your infrastructure matches the configuration.

## Task Requirements

1. The stream should be named `devops-stream`.
2. Use Terraform to create this Kinesis stream.
3. The Terraform working directory is `/home/bob/terraform`. Create the `main.tf` file (do not create a different `.tf` file) to accomplish this task.
4. Right-click under the **EXPLORER** section in VS Code and select Open in Integrated Terminal to launch the terminal.
5. Before submitting the task, ensure that `terraform plan` returns `No changes. Your infrastructure matches the configuration.`

## Solution

This Terraform lab started with a fresh workspace, so `main.tf` had to be created manually from the VS Code Explorer. The only resource required was an `aws_kinesis_stream` resource. Because the challenge did not specify a capacity, the configuration uses one shard, the minimum capacity for a provisioned stream. The existing `provider.tf` was left unchanged.

### 📦 Variables

```bash
AWS_REGION="us-east-1"
STREAM_NAME="devops-stream"
```

> **Why:** `AWS_REGION` records the lab region, and `STREAM_NAME` records the exact stream name supplied by the challenge. The existing `provider.tf` supplies the AWS provider configuration. The shard count is a minimal configuration choice rather than a value supplied by the challenge.

### 📝 Step 1: Create `main.tf`

In the VS Code Explorer, create `/home/bob/terraform/main.tf` with the following content:

```hcl
resource "aws_kinesis_stream" "devops_stream" {
  name        = "devops-stream"
  shard_count = 1
}
```

> **Why:** `aws_kinesis_stream` creates and manages an Amazon Kinesis data stream. `name` assigns the required stream name. `shard_count = 1` provisions one shard, which supplies the minimum explicit capacity for a provisioned stream. No consumers, tags, retention overrides, or encryption settings are required by the challenge.

### ⚙️ Step 2: Initialize Terraform

From the integrated terminal in `/home/bob/terraform`, run:

```bash
terraform init
```

Successful output includes:

```text
Terraform has been successfully initialized!
```

> **Why:** `terraform init` prepares the fresh working directory and installs or reuses the AWS provider required by the Kinesis stream resource.

### 🚀 Step 3: Create the Kinesis stream

Run:

```bash
terraform apply --auto-approve
```

The lab returned:

```text
Plan: 1 to add, 0 to change, 0 to destroy.
aws_kinesis_stream.devops_stream: Creating...
aws_kinesis_stream.devops_stream: Still creating... [10s elapsed]
aws_kinesis_stream.devops_stream: Still creating... [20s elapsed]
aws_kinesis_stream.devops_stream: Creation complete after 21s [id=arn:aws:kinesis:us-east-1:000000000000:stream/devops-stream]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

> **Why:** `terraform apply` compares the configuration with Terraform state and creates the Kinesis stream. `--auto-approve` accepts the generated plan without an additional confirmation prompt. Stream creation can take several seconds while AWS provisions the stream, so the `Still creating` messages are expected.

### ✅ Step 4: Verify there are no pending changes

After `terraform apply` completes, run:

```bash
terraform plan
```

The required final result is:

```text
No changes. Your infrastructure matches the configuration.
```

This confirms that the `devops-stream` resource exists and that the Terraform state matches `main.tf`.

> **Why:** `terraform plan` compares the desired configuration with the current Terraform state and the remote infrastructure without changing anything. `No changes` confirms that the stream is fully synchronized and that no additional create, update, or delete operation is pending.

## Best Practices

- **Keep the stream configuration minimal.** Define only the name and the required capacity when the challenge does not request consumers, tags, encryption, or retention changes.
- **Choose capacity deliberately.** One shard is the minimal explicit configuration for a provisioned stream; production workloads should size shards according to throughput requirements.
- **Wait for asynchronous creation.** Kinesis stream provisioning can take several seconds, so wait for Terraform to report creation completion before running the final plan.
- **Always verify drift before submission.** Run `terraform plan` after applying and confirm that it reports no changes.
- **Treat each Terraform lab as independent.** Create `main.tf` when it is absent and do not assume resources, state, provider locks, or files from another lab remain available.

### 📚 Official Documentation

- [Terraform AWS provider `aws_kinesis_stream` resource](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/kinesis_stream)
- [Amazon Kinesis Data Streams](https://docs.aws.amazon.com/streams/latest/dev/introduction.html)
- [Terraform `init` command](https://developer.hashicorp.com/terraform/cli/commands/init)
- [Terraform `apply` command](https://developer.hashicorp.com/terraform/cli/commands/apply)
- [Terraform `plan` command](https://developer.hashicorp.com/terraform/cli/commands/plan)
