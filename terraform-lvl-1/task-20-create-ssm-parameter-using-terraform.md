# Task 20: Create SSM Parameter Using Terraform

The Nautilus DevOps team needs to create an SSM parameter in AWS with the following requirements:

1) The name of the parameter should be devops-ssm-parameter.

2) Set the parameter type to String.

3) Set the parameter value to devops-value.

4) The parameter should be created in the us-east-1 region.

5) Ensure the parameter is successfully created using terraform and can be retrieved when the task is completed.

The Terraform working directory is /home/bob/terraform. Create the main.tf file (do not create a different .tf file) to accomplish this task.

Note: Right-click under the EXPLORER section in VS Code and select Open in Integrated Terminal to launch the terminal.

## Task Requirements

1. The name of the parameter should be `devops-ssm-parameter`.
2. Set the parameter type to `String`.
3. Set the parameter value to `devops-value`.
4. The parameter should be created in the `us-east-1` region.
5. Ensure the parameter is successfully created using Terraform and can be retrieved when the task is completed.
6. The Terraform working directory is `/home/bob/terraform`. Create the `main.tf` file (do not create a different `.tf` file) to accomplish this task.
7. Right-click under the **EXPLORER** section in VS Code and select Open in Integrated Terminal to launch the terminal.

## Solution

This Terraform lab starts with a fresh workspace, so `main.tf` must be created manually if it is absent. The only resource required is an `aws_ssm_parameter` resource. The existing `provider.tf` supplies the AWS provider configuration and should remain unchanged.

### 📦 Variables

```bash
AWS_REGION="us-east-1"
PARAMETER_NAME="devops-ssm-parameter"
PARAMETER_TYPE="String"
PARAMETER_VALUE="devops-value"
```

> **Why:** These values record the exact region, name, type, and value required by the challenge. They are reference values for the guide; the Terraform resource below uses the literal values directly. The existing `provider.tf` supplies the AWS provider configuration.

### 📝 Step 1: Create `main.tf`

In the VS Code Explorer, open `/home/bob/terraform` and create `main.tf` with this content:

```hcl
resource "aws_ssm_parameter" "devops_ssm_parameter" {
  name  = "devops-ssm-parameter"
  type  = "String"
  value = "devops-value"
}
```

> **Why:** `aws_ssm_parameter` creates and manages a parameter in AWS Systems Manager Parameter Store. `name` sets the required parameter name, `type = "String"` stores the value as plain text, and `value` sets the required content. No `SecureString` encryption or additional parameter settings are required by this challenge.

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

> **Why:** `terraform init` prepares the working directory and installs or reuses the AWS provider required by the SSM parameter resource.

### 🚀 Step 4: Create the SSM parameter

```bash
terraform apply --auto-approve
```

The lab returned:

```text
Plan: 1 to add, 0 to change, 0 to destroy.
aws_ssm_parameter.devops_ssm_parameter: Creating...
aws_ssm_parameter.devops_ssm_parameter: Creation complete after 1s [id=devops-ssm-parameter]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

Terraform displayed the parameter value as `(sensitive value)` in its plan output, but the parameter type was still `String` as required.

> **Why:** `terraform apply` compares the desired configuration with Terraform state and creates the SSM parameter. `--auto-approve` accepts the generated execution plan without requesting an additional confirmation.

### ✅ Step 5: Retrieve and verify the parameter

```bash
aws ssm get-parameter \
  --name "devops-ssm-parameter" \
  --region "us-east-1" \
  --query 'Parameter.[Name,Type,Value]' \
  --output table
```

The lab returned:

```text
--------------------------
|      GetParameter      |
+------------------------+
|  devops-ssm-parameter  |
|  String                |
|  devops-value          |
+------------------------+
```

> **Why:** `aws ssm get-parameter` retrieves one Parameter Store parameter. `--name` selects the parameter, `--region` targets `us-east-1`, `--query` selects its name, type, and value, and `--output table` formats the result for easy verification. The returned values confirm that the parameter was created with the required configuration and can be retrieved successfully.

## Best Practices

- **Use the exact parameter name.** Parameter names are part of the resource identity, so `devops-ssm-parameter` must match the challenge exactly.
- **Choose the requested parameter type.** Use `String` for a non-encrypted text value; use `SecureString` only when the challenge requires encryption for sensitive data.
- **Keep provider configuration separate.** Leave the pre-existing `provider.tf` unchanged and create only the requested resource in `main.tf`.
- **Verify retrieval directly.** Use `aws ssm get-parameter` after applying so the parameter's name, type, and value are confirmed in AWS.
- **Treat each Terraform lab as independent.** Create `main.tf` when it is absent and do not assume state, provider locks, or resources from another lab remain available.

### 📚 Official Documentation

- [Terraform AWS provider `aws_ssm_parameter` resource](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/ssm_parameter)
- [AWS Systems Manager Parameter Store](https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-parameter-store.html)
- [AWS CLI `get-parameter` command](https://docs.aws.amazon.com/cli/latest/reference/ssm/get-parameter.html)
- [Terraform `init` command](https://developer.hashicorp.com/terraform/cli/commands/init)
- [Terraform `apply` command](https://developer.hashicorp.com/terraform/cli/commands/apply)
