# Task 02: Create Security Group Using Terraform

The Nautilus DevOps team is strategizing the migration of a portion of their infrastructure to the AWS cloud. Recognizing the scale of this undertaking, they have opted to approach the migration in incremental steps rather than as a single massive transition. To achieve this, they have segmented large tasks into smaller, more manageable units. This granular approach enables the team to execute the migration in gradual phases, ensuring smoother implementation and minimizing disruption to ongoing operations. By breaking down the migration into smaller tasks, the Nautilus DevOps team can systematically progress through each stage, allowing for better control, risk mitigation, and optimization of resources throughout the process.

Use **Terraform** to create a security group under the default VPC with the following requirements:

1) The name of the security group must be `xfusion-sg`.

2) The description must be Security group for Nautilus App Servers.

3) Add an **inbound rule** of type HTTP, with a port range of 80, and source CIDR range 0.0.0.0/0.

4) Add another **inbound rule** of type SSH, with a port range of 22, and source CIDR range 0.0.0.0/0.

Ensure that the security group is created in the **us-east-1** region using Terraform. The Terraform working directory is `/home/bob/terraform`. Create the `main.tf` file (do not create a different `.tf` file) to accomplish this task.

Note: Right-click under the EXPLORER section in VS Code and select Open in Integrated Terminal to launch the terminal.

## Task Requirements

1. Create a security group named `xfusion-sg`.
2. Set the description to `Security group for Nautilus App Servers`.
3. Create the security group in the default VPC.
4. Add an inbound HTTP rule for TCP port `80` from `0.0.0.0/0`.
5. Add an inbound SSH rule for TCP port `22` from `0.0.0.0/0`.
6. Use the `us-east-1` region.
7. Store the Terraform configuration in `/home/bob/terraform/main.tf`.

## Solution

The lab already provided `provider.tf`, including the AWS provider and the `us-east-1` region. Therefore, `main.tf` contains only the data source and security group resource needed for this task.

The data source finds the existing default VPC. The security group is then created in that VPC with two inline inbound rules: HTTP on port `80` and SSH on port `22`.

### 📄 Step 1: Create main.tf in the Terraform working directory

In the VS Code Explorer, open `/home/bob/terraform`, right-click the directory, select **New File**, and name the file `main.tf`.

Add the following configuration:

```hcl
data "aws_vpc" "default" {
  default = true
}

resource "aws_security_group" "xfusion_sg" {
  name        = "xfusion-sg"
  description = "Security group for Nautilus App Servers"
  vpc_id      = data.aws_vpc.default.id

  ingress {
    description = "Allow HTTP"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    description = "Allow SSH"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

> **Why:** The `aws_vpc` data source reads information about an existing VPC instead of creating a new one. `default = true` selects the default VPC in the provider's configured region. The `aws_security_group` resource creates the security group. `name` assigns `xfusion-sg`, `description` records its purpose, and `vpc_id` places it in the discovered default VPC.

Each `ingress` block defines an incoming traffic rule. `description` explains the rule, `from_port` and `to_port` define the allowed port range, `protocol = "tcp"` selects TCP traffic, and `cidr_blocks = ["0.0.0.0/0"]` allows the specified port from every IPv4 address. The first block allows HTTP on port `80`; the second allows SSH on port `22`.

### ⚙️ Step 2: Initialize Terraform

Open the integrated terminal from the `/home/bob/terraform` directory and run:

```bash
terraform init
```

Expected result:

```text
Terraform has been successfully initialized!
```

> **Why:** `terraform init` initializes the working directory and ensures that the provider plugins required by the existing `provider.tf` and new `main.tf` configuration are available.

### 🧭 Step 3: Review the execution plan

```bash
terraform plan
```

The plan should show one security group for creation and a lookup of the default VPC:

```text
data.aws_vpc.default
aws_security_group.xfusion_sg
```

> **Why:** `terraform plan` previews the changes without applying them. The data source reads the VPC information, while `aws_security_group.xfusion_sg` represents the new security group and its two inbound rules.

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

> **Why:** `terraform apply` creates the security group in the default VPC and registers both inbound rules in AWS. The data source is only a lookup, so Terraform counts the security group as the single created resource.

### ✅ Step 5: Confirm the Terraform result

The successful apply should report one created resource:

```text
Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

The expected final configuration is:

- Security group name: `xfusion-sg`
- Description: `Security group for Nautilus App Servers`
- VPC: default VPC in `us-east-1`
- Inbound HTTP: TCP `80` from `0.0.0.0/0`
- Inbound SSH: TCP `22` from `0.0.0.0/0`

> **Why:** The apply summary confirms that Terraform created the security group. The configuration values correspond to the requested name, description, region, default VPC, and inbound rules.

## Best Practices

- **Reuse the existing default VPC.** The `aws_vpc` data source avoids creating an unnecessary VPC and ensures the security group is attached to the pre-existing default VPC.
- **Describe inbound rules clearly.** Rule descriptions make the purpose of HTTP and SSH access visible in the AWS console.
- **Review the plan before applying.** `terraform plan` makes it possible to catch incorrect ports, protocols, or VPC references before changing AWS.
- **Restrict CIDR ranges in production.** `0.0.0.0/0` is required by this training task but exposes the ports to every IPv4 address; real SSH access should normally use a trusted network or VPN range.
- **Avoid mixing rule management styles.** If standalone security-group rule resources are introduced later, do not manage the same rules simultaneously with inline `ingress` blocks.
- **Keep provider configuration separate when provided.** Leaving `provider.tf` unchanged keeps the AWS region and provider setup centralized while `main.tf` contains the task resources.

### 📚 Official Documentation

- [aws_vpc data source](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/data-sources/vpc)
- [aws_security_group resource](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/security_group)
- [terraform init command](https://developer.hashicorp.com/terraform/cli/commands/init)
- [terraform plan command](https://developer.hashicorp.com/terraform/cli/commands/plan)
- [terraform apply command](https://developer.hashicorp.com/terraform/cli/commands/apply)
