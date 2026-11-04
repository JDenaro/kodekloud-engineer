# Task 23: OpenSearch Setup Using Terraform

The Nautilus DevOps team needs an Amazon OpenSearch Service domain for application logs. This challenge creates a domain named `devops-es` in Terraform.

## Task Requirements

1. Create an OpenSearch domain named `devops-es`.
2. Use Terraform in `/home/bob/terraform` and define the resource in `main.tf`.
3. Wait for the domain to finish creating, then make sure `terraform plan` reports no changes before submitting.

## Solution

The lab workspace did not contain `main.tf`, so create it with the `aws_opensearch_domain` resource. The provider configuration is supplied by the lab workspace.

### 📝 Step 1: Create `main.tf`

In VS Code, right-click under **EXPLORER** and choose **Open in Integrated Terminal**. In the terminal, go to the Terraform working directory:

```bash
cd /home/bob/terraform
```

Create `main.tf` with this configuration:

```hcl
resource "aws_opensearch_domain" "devops_es" {
  domain_name = "devops-es"
}
```

> **Why:** `aws_opensearch_domain` is the Terraform AWS provider resource for an Amazon OpenSearch Service domain. Its `domain_name` is set to the exact name required by the challenge.

### 🚀 Step 2: Initialize and review the configuration

```bash
terraform init
```

```bash
terraform plan
```

> **Why:** Initialization installs the AWS provider selected by the lab. Planning checks the configuration and previews the domain creation before Terraform makes changes.

The observed initialization succeeded with `hashicorp/aws` version `5.91.0`. The plan showed `aws_opensearch_domain.devops_es` with `domain_name = "devops-es"` and reported `1 to add, 0 to change, 0 to destroy`.

### ⏳ Step 3: Create the domain

Apply the reviewed plan:

```bash
terraform apply
```

Enter `yes` when Terraform prompts for confirmation, then let the operation continue until it completes.

> **Why:** OpenSearch provisions managed service infrastructure, so creation takes longer than many basic resources. Terraform continues checking AWS until the domain is ready; keep the command running and wait for its completion before submitting the lab.

In the observed run, Terraform displayed `Still creating...` at ten-second intervals and completed after **10 minutes**. Apply reported `Resources: 1 added, 0 changed, 0 destroyed`.

### ✅ Step 4: Verify and finish

After apply completes, run the required final plan:

```bash
terraform plan
```

> **Why:** A fresh plan compares the created domain with the configuration and should report `No changes. Your infrastructure matches the configuration.` This confirms Terraform sees the requested configuration as applied.

The final plan output was not shared. The user confirmed that KodeKloud marked the lab successful.

## Best Practices

- Allow time for managed services to become ready. AWS says new OpenSearch domains commonly take 15–30 minutes to initialize, and timing can vary; this lab's observed apply completed in 10 minutes.
- Wait for Terraform apply to finish before running the final plan or submitting the lab.

### 📚 Official Documentation

- [Terraform AWS OpenSearch domain resource](https://registry.terraform.io/providers/hashicorp/aws/5.91.0/docs/resources/opensearch_domain)
- [Create an Amazon OpenSearch Service domain](https://docs.aws.amazon.com/opensearch-service/latest/developerguide/gsgcreate-domain.html)
