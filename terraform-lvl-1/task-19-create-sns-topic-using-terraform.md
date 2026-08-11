# Task 19: Create SNS Topic Using Terraform

The Nautilus DevOps team needs to set up an SNS topic for sending notifications. They need to create an SNS topic with the following specifications:

1) The topic name should be devops-notifications.

Use Terraform to create this SNS topic. The Terraform working directory is /home/bob/terraform. Create the main.tf file (do not create a different .tf file) to accomplish this task.

Note: Right-click under the EXPLORER section in VS Code and select Open in Integrated Terminal to launch the terminal.

## Task Requirements

1. The topic name should be `devops-notifications`.
2. Use Terraform to create this SNS topic.
3. The Terraform working directory is `/home/bob/terraform`. Create the `main.tf` file (do not create a different `.tf` file) to accomplish this task.
4. Right-click under the **EXPLORER** section in VS Code and select Open in Integrated Terminal to launch the terminal.

## Solution

This Terraform lab starts with a fresh working directory, so `main.tf` must be created manually if it is absent. The only resource required is an `aws_sns_topic` resource. The existing `provider.tf` supplies the AWS provider configuration and must remain unchanged.

### 📦 Variables

```bash
AWS_REGION="us-east-1"
TOPIC_NAME="devops-notifications"
```

> **Why:** `AWS_REGION` records the AWS region used by the lab, while `TOPIC_NAME` records the exact SNS topic name required by the challenge. The existing `provider.tf` supplies the provider configuration; these variables are reference values for the guide and are not required by the Terraform resource itself.

### 📝 Step 1: Create `main.tf`

In the VS Code Explorer, open `/home/bob/terraform` and create `main.tf` with this content:

```hcl
resource "aws_sns_topic" "devops_notifications" {
  name = "devops-notifications"
}
```

> **Why:** `aws_sns_topic` creates and manages an Amazon Simple Notification Service topic. The Terraform resource label `devops_notifications` is an internal name used by Terraform, while the `name` argument sets the actual AWS topic name to `devops-notifications`. No subscription, delivery endpoint, or topic policy is required by this challenge.

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

> **Why:** `terraform init` prepares the working directory and installs or reuses the AWS provider required to create the SNS topic.

### 🚀 Step 4: Create the SNS topic

```bash
terraform apply --auto-approve
```

The lab returned:

```text
Plan: 1 to add, 0 to change, 0 to destroy.
aws_sns_topic.devops_notifications: Creating...
aws_sns_topic.devops_notifications: Creation complete after 1s [id=arn:aws:sns:us-east-1:000000000000:devops-notifications]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

> **Why:** `terraform apply` compares the desired configuration with Terraform state and creates the SNS topic. `--auto-approve` accepts the generated execution plan without asking for an additional confirmation.

### ✅ Step 5: Verify the Terraform configuration

```bash
terraform plan
```

The expected result is:

```text
No changes. Your infrastructure matches the configuration.
```

This confirms that the `devops-notifications` topic exists and that the deployed infrastructure matches `main.tf`.

> **Why:** `terraform plan` compares the configuration, Terraform state, and remote infrastructure without changing anything. `No changes` confirms that no additional create, update, or delete operation is pending.

## Best Practices

- **Use the exact required name.** The Terraform resource label may use underscores, but the AWS topic name must remain `devops-notifications`.
- **Keep provider configuration separate.** Leave the pre-existing `provider.tf` unchanged and create only the requested resource in `main.tf`.
- **Keep the configuration minimal.** Do not add subscriptions, policies, tags, or delivery settings that the challenge does not request.
- **Verify before submitting.** Run `terraform plan` after applying and confirm that it reports no changes.
- **Treat each Terraform lab as independent.** Create `main.tf` when it is absent and do not assume state, provider locks, or resources from another lab remain available.

### 📚 Official Documentation

- [Terraform AWS provider `aws_sns_topic` resource](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/sns_topic)
- [Amazon Simple Notification Service documentation](https://docs.aws.amazon.com/sns/latest/dg/welcome.html)
- [Terraform `init` command](https://developer.hashicorp.com/terraform/cli/commands/init)
- [Terraform `apply` command](https://developer.hashicorp.com/terraform/cli/commands/apply)
- [Terraform `plan` command](https://developer.hashicorp.com/terraform/cli/commands/plan)
