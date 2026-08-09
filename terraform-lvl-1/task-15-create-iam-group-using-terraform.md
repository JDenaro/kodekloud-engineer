# Task 15: Create IAM Group Using Terraform

The john DevOps team has been creating a couple of services on AWS cloud. They have been breaking down the migration into smaller tasks, allowing for better control, risk mitigation, and optimization of resources throughout the migration process. Recently they came up with requirements mentioned below.

Create an IAM group named iamgroup_john using terraform.

The Terraform working directory is /home/bob/terraform. Create the main.tf file (do not create a different .tf file) to accomplish this task.

Note: Right-click under the EXPLORER section in VS Code and select Open in Integrated Terminal to launch the terminal.

## Task Requirements

1. Create an IAM group named `iamgroup_john` using terraform.
2. The Terraform working directory is `/home/bob/terraform`. Create the `main.tf` file (do not create a different `.tf` file) to accomplish this task.
3. Note: Right-click under the **EXPLORER** section in VS Code and select Open in Integrated Terminal to launch the terminal.

## Solution

This Terraform lab started with a fresh workspace, so `main.tf` had to be created manually from the VS Code Explorer. The only resource required was an `aws_iam_group` resource. No IAM users, policies, roles, or group memberships were required by the challenge. The existing `provider.tf` was left unchanged.

### 📦 Variables

```bash
AWS_REGION="us-east-1"
IAM_GROUP_NAME="iamgroup_john"
```

> **Why:** `AWS_REGION` records the lab's AWS region, although IAM is a global service, and `IAM_GROUP_NAME` records the exact group name required by the challenge. The existing `provider.tf` supplies the provider configuration.

### 📝 Step 1: Create `main.tf`

In the VS Code Explorer, create `/home/bob/terraform/main.tf` with the following content:

```hcl
resource "aws_iam_group" "iamgroup_john" {
  name = "iamgroup_john"
}
```

> **Why:** `aws_iam_group` is the Terraform resource that creates and manages an AWS IAM group. `iamgroup_john` is the Terraform logical resource name, while `name = "iamgroup_john"` sets the actual group name in AWS. No additional IAM resources are needed because the task asks only for the group.

### ⚙️ Step 2: Initialize Terraform

From the integrated terminal in `/home/bob/terraform`, run:

```bash
terraform init
```

Successful output includes:

```text
Terraform has been successfully initialized!
```

> **Why:** `terraform init` prepares the fresh working directory and installs or reuses the AWS provider required by the `aws_iam_group` resource.

### 🚀 Step 3: Create the IAM group

Run:

```bash
terraform apply --auto-approve
```

The lab returned:

```text
Plan: 1 to add, 0 to change, 0 to destroy.
aws_iam_group.iamgroup_john: Creating...
aws_iam_group.iamgroup_john: Creation complete after 1s [id=iamgroup_john]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

> **Why:** `terraform apply` compares the configuration with Terraform state and creates the IAM group. `--auto-approve` accepts the generated plan without an additional confirmation prompt. The plan confirms that exactly one group was added and no unrelated resources were changed.

### ✅ Step 4: Verify the IAM group

Run:

```bash
aws iam get-group \
  --group-name iamgroup_john \
  --query 'Group.GroupName' \
  --output text
```

The lab returned:

```text
iamgroup_john
```

> **Why:** `aws iam get-group` retrieves information about the specified IAM group. `--group-name` selects the group to inspect, `--query` extracts only the returned `GroupName` field, and `--output text` prints the value without JSON formatting. The matching name confirms that Terraform created the requested group.

## Best Practices

- **Create only the requested IAM resource.** Do not add users, policies, roles, or memberships unless the requirements explicitly call for them.
- **Manage membership deliberately.** If users must be added later, manage group membership consistently through Terraform or through the AWS console, rather than mixing both methods.
- **Use group-based permissions.** Attach shared permissions to a group when multiple users need the same access, instead of duplicating policies per user.
- **Apply least privilege.** Add only the policies required by the group's future workload.
- **Treat each Terraform lab as independent.** Create `main.tf` when it is absent and do not assume resources, state, provider locks, or files from another lab remain available.

### 📚 Official Documentation

- [Terraform AWS provider `aws_iam_group` resource](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/iam_group)
- [AWS IAM `GetGroup` API](https://docs.aws.amazon.com/IAM/latest/APIReference/API_GetGroup.html)
- [AWS CLI `get-group` command](https://docs.aws.amazon.com/cli/latest/reference/iam/get-group.html)
- [Terraform `init` command](https://developer.hashicorp.com/terraform/cli/commands/init)
- [Terraform `apply` command](https://developer.hashicorp.com/terraform/cli/commands/apply)
