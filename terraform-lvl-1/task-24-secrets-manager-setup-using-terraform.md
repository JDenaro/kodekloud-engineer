# Task 24: Secrets Manager Setup Using Terraform

The Nautilus DevOps team needs to store an application username and password in AWS Secrets Manager. This challenge creates a secret named `devops-secret` and adds a JSON key-value version.

## Task Requirements

1. Use Terraform in `/home/bob/terraform` and define the resources in `main.tf`.
2. Create a Secrets Manager secret named `devops-secret`.
3. Store the required `username` and `password` fields in its secret value.

## Solution

The configuration uses `aws_secretsmanager_secret` for the secret metadata and `aws_secretsmanager_secret_version` for its JSON value. The lab workspace supplies the AWS provider configuration. Secret values can be stored in Terraform state, so this guide intentionally does not reproduce the lab password.

### 🧭 Step 1: Open the working directory

In VS Code, right-click under **EXPLORER** and select **Open in Integrated Terminal**, then go to the Terraform directory:

```bash
cd /home/bob/terraform
```

> **Why:** Terraform loads configuration from the working directory. Using the lab's specified path ensures the resources are managed in the correct workspace.

### 📝 Step 2: Create `main.tf`

Create `main.tf` with the following resources. Replace the password placeholder with the exact value provided in the active lab; do not share the completed file or state if it contains the credential.

```hcl
resource "aws_secretsmanager_secret" "devops_secret" {
  name = "devops-secret"
}

resource "aws_secretsmanager_secret_version" "devops_secret" {
  secret_id = aws_secretsmanager_secret.devops_secret.id

  secret_string = jsonencode({
    username = "admin"
    password = "<lab-provided password>"
  })
}
```

> **Why:** The first resource creates the secret container. The version resource stores its contents as JSON, and the `secret_id` reference links the version to the secret and makes Terraform create them in the right order. `jsonencode` safely serializes the username and password fields into JSON.

### 🚀 Step 3: Initialize, plan, and apply

```bash
terraform init
```

```bash
terraform plan
```

Review the proposed resources, then apply the configuration and confirm when prompted:

```bash
terraform apply
```

> **Why:** Initialization installs the configured AWS provider. Planning previews the secret and version resources, while apply creates them in Secrets Manager.

The observed initialization succeeded with AWS provider version `5.91.0`. The plan marked `secret_string` as `(sensitive value)` and showed `2 to add, 0 to change, 0 to destroy`. Apply completed with `Resources: 2 added, 0 changed, 0 destroyed`.

### ✅ Step 4: Verify and finish

Run a fresh plan after apply:

```bash
terraform plan
```

> **Why:** A new plan checks that Terraform sees the requested secret and version as applied. Avoid sharing its output if it reveals secret material.

The final plan output was not shared. The user confirmed that KodeKloud marked the lab successful.

## Best Practices

- Keep credentials out of source control and redact them from terminal output shared with others.
- Terraform state may contain secret values; restrict access to state and use a protected remote backend for shared environments.

### 📚 Official Documentation

- [Terraform AWS Secrets Manager secret resource](https://registry.terraform.io/providers/hashicorp/aws/5.91.0/docs/resources/secretsmanager_secret)
- [Terraform AWS Secrets Manager secret version resource](https://registry.terraform.io/providers/hashicorp/aws/5.91.0/docs/resources/secretsmanager_secret_version)
- [AWS Secrets Manager concepts](https://docs.aws.amazon.com/secretsmanager/latest/userguide/intro.html)
