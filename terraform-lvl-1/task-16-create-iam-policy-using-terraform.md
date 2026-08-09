# Task 16: Create IAM Policy Using Terraform

When establishing infrastructure on the AWS cloud, Identity and Access Management (IAM) is among the first and most critical services to configure. IAM facilitates the creation and management of user accounts, groups, roles, policies, and other access controls. The Nautilus DevOps team is currently in the process of configuring these resources and has outlined the following requirements.

Create an IAM policy named iampolicy_anita in us-east-1 region using Terraform. It must allow read-only access to the EC2 console, i.e., this policy must allow users to view all instances, AMIs, and snapshots in the Amazon EC2 console.

The Terraform working directory is /home/bob/terraform. Create the main.tf file (do not create a different .tf file) to accomplish this task.

Note: Right-click under the EXPLORER section in VS Code and select Open in Integrated Terminal to launch the terminal.

## Task Requirements

1. Create an IAM policy named `iampolicy_anita` in `us-east-1` region using Terraform.
2. It must allow read-only access to the EC2 console, i.e., this policy must allow users to view all instances, AMIs, and snapshots in the Amazon EC2 console.
3. The Terraform working directory is `/home/bob/terraform`. Create the `main.tf` file (do not create a different `.tf` file) to accomplish this task.
4. Note: Right-click under the **EXPLORER** section in VS Code and select Open in Integrated Terminal to launch the terminal.

## Solution

This Terraform lab started with a fresh workspace, so `main.tf` had to be created manually from the VS Code Explorer. The only resource required was an `aws_iam_policy` resource. The policy grants the three EC2 read-only actions required by the challenge: viewing instances, AMIs, and snapshots. It was not attached to a user, group, or role because the task only asks for the policy itself. The existing `provider.tf` was left unchanged.

IAM is a global AWS service, but the lab's Terraform provider is configured for `us-east-1` as required by the challenge.

### 📦 Variables

```bash
AWS_REGION="us-east-1"
IAM_POLICY_NAME="iampolicy_anita"
```

> **Why:** `AWS_REGION` records the required lab region, and `IAM_POLICY_NAME` records the exact policy name supplied by the challenge. The existing `provider.tf` supplies the provider configuration.

### 📝 Step 1: Create `main.tf`

In the VS Code Explorer, create `/home/bob/terraform/main.tf` with the following content:

```hcl
resource "aws_iam_policy" "iampolicy_anita" {
  name = "iampolicy_anita"

  policy = jsonencode({
    Version = "2012-10-17"

    Statement = [
      {
        Effect = "Allow"

        Action = [
          "ec2:DescribeInstances",
          "ec2:DescribeImages",
          "ec2:DescribeSnapshots"
        ]

        Resource = "*"
      }
    ]
  })
}
```

> **Why:** `aws_iam_policy` creates a customer-managed IAM policy, and `name` sets its AWS name. `policy` receives the policy document as JSON. `jsonencode` converts the Terraform object into valid JSON while avoiding manual JSON formatting errors. `Version = "2012-10-17"` identifies the IAM policy language version. `Effect = "Allow"` grants the listed permissions. `ec2:DescribeInstances` permits viewing EC2 instances, `ec2:DescribeImages` permits viewing AMIs, and `ec2:DescribeSnapshots` permits viewing snapshots. `Resource = "*"` applies these read-only actions to all matching EC2 resources because these describe operations are account-wide read operations. No attachment resource is added because the challenge requests only the policy.

### ⚙️ Step 2: Initialize Terraform

From the integrated terminal in `/home/bob/terraform`, run:

```bash
terraform init
```

Successful output includes:

```text
Terraform has been successfully initialized!
```

> **Why:** `terraform init` prepares the fresh working directory and installs or reuses the AWS provider required by the `aws_iam_policy` resource.

### 🚀 Step 3: Create the IAM policy

Run:

```bash
terraform apply --auto-approve
```

The lab returned:

```text
Plan: 1 to add, 0 to change, 0 to destroy.
aws_iam_policy.iampolicy_anita: Creating...
aws_iam_policy.iampolicy_anita: Creation complete after 0s [id=arn:aws:iam::000000000000:policy/iampolicy_anita]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

The Terraform plan also displayed the three requested actions:

```text
ec2:DescribeInstances
ec2:DescribeImages
ec2:DescribeSnapshots
```

> **Why:** `terraform apply` compares the configuration with Terraform state and creates the IAM policy. `--auto-approve` accepts the generated plan without an additional confirmation prompt. The result confirms that exactly one policy was created and no unrelated resources were changed.

### ✅ Step 4: Verify the IAM policy

Run:

```bash
aws iam list-policies \
  --scope Local \
  --region us-east-1 \
  --query "Policies[?PolicyName=='iampolicy_anita'].[PolicyName,Arn]" \
  --output table
```

The lab returned:

```text
-------------------------------------------------------------------------
|                             ListPolicies                              |
+------------------+----------------------------------------------------+
|  iampolicy_anita  |  arn:aws:iam::000000000000:policy/iampolicy_anita  |
+------------------+----------------------------------------------------+
```

> **Why:** `aws iam list-policies` lists managed IAM policies. `--scope Local` limits the result to customer-managed policies, `--region us-east-1` uses the lab's configured region, `--query` selects the policy name and ARN, and `--output table` formats the result for readability. The returned policy name and ARN confirm that the requested policy exists.

## Best Practices

- **Grant only the required read permissions.** Include the three `Describe` actions needed by the challenge and avoid write or mutation actions.
- **Use `jsonencode` for policy documents.** It keeps the policy readable in Terraform and prevents JSON quoting and formatting mistakes.
- **Use `Resource = "*"` only when required.** EC2 describe actions are read-only account-wide operations and require a wildcard resource in this policy.
- **Separate creation from attachment.** Create the policy independently and attach it to a user, group, or role only when a later requirement explicitly calls for it.
- **Treat each Terraform lab as independent.** Create `main.tf` when it is absent and do not assume resources, state, provider locks, or files from another lab remain available.

### 📚 Official Documentation

- [Terraform AWS provider `aws_iam_policy` resource](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/iam_policy)
- [AWS IAM policy elements](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_elements.html)
- [Amazon EC2 API actions](https://docs.aws.amazon.com/service-authorization/latest/reference/list_amazonec2.html)
- [AWS CLI `list-policies` command](https://docs.aws.amazon.com/cli/latest/reference/iam/list-policies.html)
- [Terraform `init` command](https://developer.hashicorp.com/terraform/cli/commands/init)
- [Terraform `apply` command](https://developer.hashicorp.com/terraform/cli/commands/apply)
