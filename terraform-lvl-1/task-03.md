# Task 03: Create VPC Using Terraform

The Nautilus DevOps team is strategizing the migration of a portion of their infrastructure to the AWS cloud. Recognizing the scale of this undertaking, they have opted to approach the migration in incremental steps rather than as a single massive transition. To achieve this, they have segmented large tasks into smaller, more manageable units. This granular approach enables the team to execute the migration in gradual phases, ensuring smoother implementation and minimizing disruption to ongoing operations. By breaking down the migration into smaller tasks, the Nautilus DevOps team can systematically progress through each stage, allowing for better control, risk mitigation, and optimization of resources throughout the migration process.

Create a VPC named `xfusion-vpc` in region `us-east-1` with any IPv4 CIDR block through terraform.

The Terraform working directory is `/home/bob/terraform`. Create the `main.tf` file (do not create a different `.tf` file) to accomplish this task.

Note: Right-click under the EXPLORER section in VS Code and select Open in Integrated Terminal to launch the terminal.

## Task Requirements

1. Create a VPC named `xfusion-vpc`.
2. Use the `us-east-1` region.
3. Assign any valid IPv4 CIDR block.
4. Store the Terraform configuration in `/home/bob/terraform/main.tf`.

## Solution

The lab already provided `provider.tf`, including the AWS provider and the `us-east-1` region. Therefore, `main.tf` contains only the VPC resource.

The VPC uses the private IPv4 CIDR block `10.0.0.0/16`. The AWS VPC resource does not have a dedicated `name` argument, so the required VPC name is assigned through the `Name` tag.

### 📄 Step 1: Create main.tf in the Terraform working directory

In the VS Code Explorer, open `/home/bob/terraform`, right-click the directory, select **New File**, and name the file `main.tf`.

Add the following configuration:

```hcl
resource "aws_vpc" "xfusion_vpc" {
  cidr_block = "10.0.0.0/16"

  tags = {
    Name = "xfusion-vpc"
  }
}
```

> **Why:** `aws_vpc` is the Terraform resource used to create an AWS VPC. `cidr_block` assigns the VPC's IPv4 address range; `10.0.0.0/16` is a valid private IPv4 block and satisfies the requirement to use any valid IPv4 CIDR. The `tags` map assigns the AWS `Name` tag, which is the visible resource name `xfusion-vpc`. The resource is created in the region configured by `provider.tf`, which is `us-east-1`.

### ⚙️ Step 2: Initialize Terraform

Open the integrated terminal from the `/home/bob/terraform` directory and run:

```bash
terraform init
```

Expected result:

```text
Terraform has been successfully initialized!
```

> **Why:** `terraform init` initializes the working directory and ensures that the AWS provider required by the existing `provider.tf` and new `main.tf` configuration is available.

### 🧭 Step 3: Review the execution plan

```bash
terraform plan
```

The plan should show one VPC for creation:

```text
aws_vpc.xfusion_vpc
```

> **Why:** `terraform plan` previews the infrastructure changes without applying them. Reviewing the plan confirms that Terraform will create one VPC with the requested CIDR block and `Name` tag.

### 🚀 Step 4: Apply the configuration

```bash
terraform apply
```

When Terraform asks for confirmation, enter:

```text
yes
```

Expected result:

```text
Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

> **Why:** `terraform apply` creates the VPC described in `main.tf`. The VPC is created in the `us-east-1` region selected by `provider.tf`, with the `10.0.0.0/16` IPv4 CIDR block and `xfusion-vpc` name tag.

### ✅ Step 5: Confirm the Terraform result

The successful apply should report one created resource:

```text
Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

The expected final state is:

- VPC name: `xfusion-vpc`
- Region: `us-east-1`
- IPv4 CIDR block: `10.0.0.0/16`

> **Why:** The apply summary confirms that Terraform created the VPC. The configured resource attributes satisfy the requested name, region, and valid IPv4 CIDR requirements.

## Best Practices

- **Use a deliberate CIDR block.** `10.0.0.0/16` provides a private address range with room for future subnets in this lab.
- **Name resources with tags.** The AWS VPC resource uses the `Name` tag to provide the human-readable name shown in the AWS console.
- **Keep the provider configuration centralized.** Leave `provider.tf` unchanged and define the VPC resource in the required `main.tf` file.
- **Review before applying.** Run `terraform plan` to confirm the CIDR block, tag, and resource count before creating the VPC.
- **Plan network ranges before production use.** In a real environment, choose a CIDR block that does not overlap with connected networks such as corporate, VPN, or peered VPC ranges.

### 📚 Official Documentation

- [aws_vpc resource](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/vpc)
- [terraform init command](https://developer.hashicorp.com/terraform/cli/commands/init)
- [terraform plan command](https://developer.hashicorp.com/terraform/cli/commands/plan)
- [terraform apply command](https://developer.hashicorp.com/terraform/cli/commands/apply)
