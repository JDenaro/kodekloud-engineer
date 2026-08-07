# Task 05: Create VPC with IPv6 Using Terraform

The Nautilus DevOps team is strategically planning the migration of a portion of their infrastructure to the AWS cloud. Acknowledging the magnitude of the endeavor, they have chosen to tackle the migration incrementally rather than as a single, massive transition. Their approach involves creating Virtual Private Clouds (VPCs) as the initial step, as they will be provisioning various services under different VPCs.

For this task, create a VPC named devops-vpc in the us-east-1 region with the Amazon-provided IPv6 CIDR block using terraform.

The Terraform working directory is /home/bob/terraform. Create the main.tf file (do not create a different .tf file) to accomplish this task.

Note: Right-click under the EXPLORER section in VS Code and select Open in Integrated Terminal to launch the terminal.

## Task Requirements

1. For this task, create a VPC named `devops-vpc` in the `us-east-1` region with the Amazon-provided IPv6 CIDR block using terraform.
2. The Terraform working directory is `/home/bob/terraform`. Create the `main.tf` file (do not create a different `.tf` file) to accomplish this task.
3. Note: Right-click under the **EXPLORER** section in VS Code and select Open in Integrated Terminal to launch the terminal.

## Solution

An AWS VPC must have an IPv4 CIDR block. The Amazon-provided IPv6 range is an additional address range, so the Terraform resource includes a private IPv4 CIDR block together with `assign_generated_ipv6_cidr_block = true`. The lab starts with a fresh Terraform workspace, so `main.tf` must be created manually for this task. The existing `provider.tf` remains unchanged and supplies the AWS provider configuration for `us-east-1`.

### 📄 Step 1: Create main.tf in the fresh Terraform workspace

In the VS Code Explorer, open `/home/bob/terraform`. Create a new file named `main.tf` and add:

```hcl
resource "aws_vpc" "devops_vpc" {
  cidr_block                       = "10.0.0.0/16"
  assign_generated_ipv6_cidr_block = true

  tags = {
    Name = "devops-vpc"
  }
}
```

> **Why:** `aws_vpc` is the Terraform resource for an AWS VPC. `cidr_block` provides the required IPv4 address range; `10.0.0.0/16` is the private range selected for this lab. `assign_generated_ipv6_cidr_block = true` asks AWS to associate an Amazon-provided IPv6 CIDR block. The `Name` tag gives the VPC the requested visible name. Because `provider.tf` is already present in the lab, no provider block is added to `main.tf`.

### ⚙️ Step 2: Initialize Terraform

From the integrated terminal in `/home/bob/terraform`, run:

```bash
terraform init
```

Successful output includes:

```text
Terraform has been successfully initialized!
```

> **Why:** `terraform init` prepares a Terraform working directory, downloads the required provider plugins, and records the selected provider versions in the dependency lock file. Run it in every fresh lab workspace before creating resources.

### 🚀 Step 3: Apply the configuration

Run:

```bash
terraform apply --auto-approve
```

The lab returned:

```text
Plan: 1 to add, 0 to change, 0 to destroy.
aws_vpc.devops_vpc: Creating...
aws_vpc.devops_vpc: Still creating... [10s elapsed]
aws_vpc.devops_vpc: Creation complete after 10s [id=vpc-ae7408257b60a94ee]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

> **Why:** `terraform apply` creates the infrastructure described by the configuration. `--auto-approve` accepts the plan without an interactive confirmation, which is suitable for this controlled lab. The plan shows that only one VPC will be created, and the completion message provides the resulting VPC ID.

### ✅ Step 4: Verify

The successful apply confirmed:

```text
Resource: aws_vpc.devops_vpc
VPC ID: vpc-ae7408257b60a94ee
Name tag: devops-vpc
IPv4 CIDR: 10.0.0.0/16
Amazon-provided IPv6 CIDR: requested with assign_generated_ipv6_cidr_block = true
Resources: 1 added, 0 changed, 0 destroyed
```

The VPC was created in the provider's `us-east-1` region with the requested name and an Amazon-provided IPv6 CIDR block.

> **Why:** The apply result confirms that Terraform created the `aws_vpc.devops_vpc` resource successfully. The configuration includes both the required IPv4 range and the flag that requests AWS to assign the VPC's IPv6 range.

## Best Practices

- **Treat each lab as independent.** Create a fresh `main.tf` in every new Terraform workspace instead of assuming files or state from a previous lab remain.
- **Keep provider configuration centralized.** Leave the existing `provider.tf` unchanged and place the task resource in `main.tf`.
- **Always define an IPv4 CIDR for a VPC.** An Amazon-provided IPv6 CIDR block supplements the VPC; it does not replace the required IPv4 range.
- **Use a descriptive Name tag.** Tags make resources easier to identify in the AWS console and command-line tools.
- **Review the plan before applying in real environments.** The lab used `--auto-approve`, but production changes should normally be reviewed before approval.

### 📚 Official Documentation

- [AWS CreateVpc API](https://docs.aws.amazon.com/AWSEC2/latest/APIReference/API_CreateVpc.html)
- [AWS provider `aws_vpc` resource](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/vpc)
- [Terraform `init` command](https://developer.hashicorp.com/terraform/cli/commands/init)
- [Terraform `apply` command](https://developer.hashicorp.com/terraform/cli/commands/apply)
