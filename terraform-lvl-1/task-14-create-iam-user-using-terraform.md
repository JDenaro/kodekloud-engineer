# Task 14: Create IAM User Using Terraform

When establishing infrastructure on the AWS cloud, Identity and Access Management (IAM) is among the first and most critical services to configure. IAM facilitates the creation and management of user accounts, groups, roles, policies, and other access controls. The Nautilus DevOps team is currently in the process of configuring these resources and has outlined the following requirements:

For this task, create an IAM user named iamuser_kareem using terraform. The Terraform working directory is /home/bob/terraform. Create the main.tf file (do not create a different .tf file) to accomplish this task.

Note: Right-click under the EXPLORER section in VS Code and select Open in Integrated Terminal to launch the terminal.

## Task Requirements

1. For this task, create an IAM user named `iamuser_kareem` using terraform.
2. The Terraform working directory is `/home/bob/terraform`. Create the `main.tf` file (do not create a different `.tf` file) to accomplish this task.
3. Note: Right-click under the **EXPLORER** section in VS Code and select Open in Integrated Terminal to launch the terminal.

## Solution

This Terraform lab started with a fresh workspace, so `main.tf` had to be created manually from the VS Code Explorer. The only resource required was an `aws_iam_user` resource. No IAM group, policy, role, password, or access key was required by the challenge. The existing `provider.tf` was left unchanged.

### 📦 Variables

```bash
AWS_REGION="us-east-1"
IAM_USER_NAME="iamuser_kareem"
```

> **Why:** `AWS_REGION` records the lab's AWS region, although IAM is a global service, and `IAM_USER_NAME` records the exact user name required by the challenge. The existing `provider.tf` supplies the provider configuration.

### 📝 Step 1: Create `main.tf`

In the VS Code Explorer, create `/home/bob/terraform/main.tf` with the following content:

```hcl
resource "aws_iam_user" "iamuser_kareem" {
  name = "iamuser_kareem"
}
```

> **Why:** `aws_iam_user` is the Terraform resource that creates and manages an AWS IAM user. `iamuser_kareem` is the Terraform logical resource name, while `name = "iamuser_kareem"` sets the actual IAM user name in AWS. No additional IAM resources are needed because the task only asks for the user itself.

### ⚙️ Step 2: Initialize Terraform

From the integrated terminal in `/home/bob/terraform`, run:

```bash
terraform init
```

Successful output includes:

```text
Terraform has been successfully initialized!
```

> **Why:** `terraform init` prepares the fresh working directory and installs or reuses the AWS provider required by the `aws_iam_user` resource.

### 🚀 Step 3: Create the IAM user

Run:

```bash
terraform apply --auto-approve
```

The lab returned:

```text
Plan: 1 to add, 0 to change, 0 to destroy.
aws_iam_user.iamuser_kareem: Creating...
aws_iam_user.iamuser_kareem: Creation complete after 1s [id=iamuser_kareem]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

> **Why:** `terraform apply` compares the configuration with Terraform state and creates the IAM user. `--auto-approve` accepts the generated plan without an additional confirmation prompt. The plan confirms that exactly one IAM user was added and no unrelated resources were changed.

### ✅ Step 4: Verify the IAM user

Run:

```bash
aws iam get-user \
  --user-name iamuser_kareem \
  --query 'User.UserName' \
  --output text
```

The lab returned:

```text
iamuser_kareem
```

> **Why:** `aws iam get-user` retrieves information about an IAM user. `--user-name` selects the user to inspect, `--query` extracts only the returned `UserName` field, and `--output text` prints the value without JSON formatting. The matching name confirms that Terraform created the requested user.

## Best Practices

- **Create only the requested IAM resource.** Do not add policies, groups, roles, passwords, or access keys unless the requirements explicitly call for them.
- **Use Terraform resource references consistently.** The Terraform logical name can differ from the AWS name, but keeping both identical makes the configuration easier to understand.
- **Apply least privilege separately.** If permissions are required later, attach only the policies the user needs in a separate, explicit change.
- **Avoid long-lived access keys by default.** This task does not require programmatic credentials, so no access key was created.
- **Treat each Terraform lab as independent.** Create `main.tf` when it is absent and do not assume resources, state, provider locks, or files from another lab remain available.

### 📚 Official Documentation

- [Terraform AWS provider `aws_iam_user` resource](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/iam_user)
- [AWS IAM `GetUser` API](https://docs.aws.amazon.com/IAM/latest/APIReference/API_GetUser.html)
- [AWS CLI `get-user` command](https://docs.aws.amazon.com/cli/latest/reference/iam/get-user.html)
- [Terraform `init` command](https://developer.hashicorp.com/terraform/cli/commands/init)
- [Terraform `apply` command](https://developer.hashicorp.com/terraform/cli/commands/apply)
