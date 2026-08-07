# Task 04: Create VPC with CIDR Using Terraform

The Nautilus DevOps team is strategically planning the migration of a portion of their infrastructure to the AWS cloud. Acknowledging the magnitude of the endeavor, they have chosen to tackle the migration incrementally rather than as a single, massive transition. Their approach involves creating Virtual Private Clouds (VPCs) as the initial step, as they will be provisioning various services under different VPCs.

Create a VPC named xfusion-vpc in us-east-1 region with 192.168.0.0/24 IPv4 CIDR using terraform.

The Terraform working directory is /home/bob/terraform. Create the main.tf file (do not create a different .tf file) to accomplish this task.

Note: Right-click under the EXPLORER section in VS Code and select Open in Integrated Terminal to launch the terminal.

## Task Requirements

1. Create a VPC named `xfusion-vpc` in the `us-east-1` region with the `192.168.0.0/24` IPv4 CIDR block using Terraform.
2. Store the Terraform configuration in `/home/bob/terraform/main.tf`.
3. Keep the existing `provider.tf` file unchanged and do not create another Terraform configuration file.

## Solution

The lab already provided `provider.tf`, including the AWS provider and the `us-east-1` region. The required resource was therefore defined only in `main.tf`. The VPC received the requested CIDR block and the human-readable name through the AWS `Name` tag.

### 📄 Step 1: Create main.tf in the Terraform working directory

In the VS Code Explorer, open `/home/bob/terraform`, right-click the directory, select **New File**, and name the file `main.tf`.

Add the following configuration:

```hcl
resource "aws_vpc" "xfusion_vpc" {
  cidr_block = "192.168.0.0/24"

  tags = {
    Name = "xfusion-vpc"
  }
}
```

> **Why:** `aws_vpc` is the Terraform resource used to create an AWS VPC. `cidr_block` assigns the VPC's IPv4 address range. The `tags` map assigns the AWS `Name` tag, which provides the visible resource name `xfusion-vpc`. The region comes from the existing `provider.tf`, so no provider block is needed in `main.tf`.

### ⚙️ Step 2: Initialize Terraform

From the integrated terminal opened in `/home/bob/terraform`, run:

```bash
terraform init
```

Successful output:

```text
Terraform has been successfully initialized!
```

> **Why:** `terraform init` initializes the working directory and installs the provider plugins required by the Terraform configuration. It also creates or updates the dependency lock file so future initializations use the selected provider version.

### 🚀 Step 3: Apply the Terraform configuration

The lab applied the configuration without an interactive confirmation prompt:

```bash
terraform apply --auto-approve
```

Relevant successful output:

```text
Plan: 1 to add, 0 to change, 0 to destroy.
aws_vpc.xfusion_vpc: Creating...
aws_vpc.xfusion_vpc: Creation complete after 1s [id=vpc-54edd9cbff067db69]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

> **Why:** `terraform apply` creates or updates the infrastructure described by the configuration. `--auto-approve` skips the confirmation prompt, which is convenient in this temporary lab because the requested change is known and intentional. The plan confirms that Terraform will create exactly one resource, and the apply result shows that the VPC was created successfully.

### ✅ Step 4: Verify

The successful apply confirmed the final state:

```text
Resource: aws_vpc.xfusion_vpc
VPC ID: vpc-54edd9cbff067db69
Name tag: xfusion-vpc
IPv4 CIDR: 192.168.0.0/24
Region: us-east-1
Resources: 1 added, 0 changed, 0 destroyed
```

The VPC was created with the requested name, CIDR block, and region.

> **Why:** The apply output confirms that Terraform created the `aws_vpc.xfusion_vpc` resource and reports the resulting VPC ID. The resource attributes match every requirement from the challenge.

## Best Practices

- **Keep provider configuration centralized.** Leave `provider.tf` unchanged and define the requested resource in `main.tf`.
- **Use a descriptive AWS Name tag.** The `Name` tag makes the VPC easy to identify in the AWS console and CLI.
- **Review the plan before applying in real environments.** The lab used `--auto-approve`, but production changes should normally be reviewed before approval.
- **Plan CIDR ranges carefully.** Ensure the VPC range does not overlap with connected networks such as VPNs, peered VPCs, or corporate networks.
- **Treat the Terraform state as important.** Terraform uses its state to track the VPC it created and determine future changes.

### 📚 Official Documentation

- [AWS provider aws_vpc resource](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/vpc)
- [Terraform init command](https://developer.hashicorp.com/terraform/cli/commands/init)
- [Terraform apply command](https://developer.hashicorp.com/terraform/cli/commands/apply)
