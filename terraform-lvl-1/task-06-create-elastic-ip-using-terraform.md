# Task 06: Create Elastic IP Using Terraform

The Nautilus DevOps team is strategizing the migration of a portion of their infrastructure to the AWS cloud. Recognizing the scale of this undertaking, they have opted to approach the migration in incremental steps rather than as a single massive transition. To achieve this, they have segmented large tasks into smaller, more manageable units. This granular approach enables the team to execute the migration in gradual phases, ensuring smoother implementation and minimizing disruption to ongoing operations. By breaking down large tasks into smaller, more manageable units, the Nautilus DevOps team can systematically progress through each stage, allowing for better control, risk mitigation, and optimization of resources throughout the migration process.

For this task, allocate an Elastic IP address named datacenter-eip using Terraform.

The Terraform working directory is /home/bob/terraform. Create the main.tf file (do not create a different .tf file) to accomplish this task.

Note: Right-click under the EXPLORER section in VS Code and select Open in Integrated Terminal to launch the terminal.

## Task Requirements

1. For this task, allocate an Elastic IP address named `datacenter-eip` using Terraform.
2. The Terraform working directory is `/home/bob/terraform`. Create the `main.tf` file (do not create a different `.tf` file) to accomplish this task.
3. Note: Right-click under the **EXPLORER** section in VS Code and select Open in Integrated Terminal to launch the terminal.

## Solution

The lab starts with a fresh Terraform workspace, so `main.tf` must be created manually for this task. The existing `provider.tf` remains unchanged and supplies the AWS provider configuration. The Elastic IP is allocated for the VPC domain and receives the requested name through the AWS `Name` tag; it is not associated with an instance because the challenge only asks for allocation.

### 📄 Step 1: Create main.tf in the fresh Terraform workspace

In the VS Code Explorer, open `/home/bob/terraform`. Create a new file named `main.tf` and add:

```hcl
resource "aws_eip" "datacenter_eip" {
  domain = "vpc"

  tags = {
    Name = "datacenter-eip"
  }
}
```

> **Why:** `aws_eip` is the Terraform resource used to allocate an AWS Elastic IP address. `domain = "vpc"` requests an address from the VPC address pool, which is the current EC2 networking model. The `Name` tag supplies the requested visible name. No association is configured because the challenge does not provide an instance or network interface to attach it to. Since `provider.tf` is already present, no provider block is added to `main.tf`.

### ⚙️ Step 2: Initialize Terraform

From the integrated terminal in `/home/bob/terraform`, run:

```bash
terraform init
```

Successful output includes:

```text
Terraform has been successfully initialized!
```

> **Why:** `terraform init` prepares the fresh working directory, installs the required AWS provider plugin, and creates or updates the dependency lock file.

### 🚀 Step 3: Apply the configuration

Run:

```bash
terraform apply --auto-approve
```

The lab returned:

```text
Plan: 1 to add, 0 to change, 0 to destroy.
aws_eip.datacenter_eip: Creating...
aws_eip.datacenter_eip: Creation complete after 1s [id=eipalloc-14a03d7cdd1b67c34]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

> **Why:** `terraform apply` creates the Elastic IP described by `main.tf`. `--auto-approve` accepts the plan without an interactive prompt, which is appropriate for this controlled lab. The allocation ID in the completion message identifies the newly allocated address.

### ✅ Step 4: Verify

The successful apply confirmed:

```text
Resource: aws_eip.datacenter_eip
Allocation ID: eipalloc-14a03d7cdd1b67c34
Domain: vpc
Name tag: datacenter-eip
Resources: 1 added, 0 changed, 0 destroyed
```

The Elastic IP was allocated successfully in the provider's `us-east-1` region with the requested VPC domain and name tag.

> **Why:** The apply output confirms that Terraform created exactly one `aws_eip.datacenter_eip` resource and that AWS returned its allocation ID. The configuration's VPC domain and `Name` tag satisfy the task requirements.

## Best Practices

- **Treat each lab as independent.** Create a fresh `main.tf` in every new Terraform workspace instead of assuming files or state from a previous lab remain.
- **Use the VPC domain.** Allocate Elastic IPs with `domain = "vpc"` for resources running in a VPC.
- **Release unused Elastic IPs.** AWS may charge for public IPv4 addresses that are allocated but not actively used; release addresses that are no longer needed.
- **Tag allocated addresses.** A descriptive `Name` tag makes the address easier to identify and manage.
- **Review the plan before applying in real environments.** The lab used `--auto-approve`, but production changes should normally be reviewed before approval.

### 📚 Official Documentation

- [AWS Elastic IP allocation example](https://docs.aws.amazon.com/code-library/latest/ug/ec2_example_ec2_AllocateAddress_section.html)
- [AWS provider `aws_eip` resource](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/eip)
- [Terraform `init` command](https://developer.hashicorp.com/terraform/cli/commands/init)
- [Terraform `apply` command](https://developer.hashicorp.com/terraform/cli/commands/apply)
