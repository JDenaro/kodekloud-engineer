# Task 08: Create AMI Using Terraform

The Nautilus DevOps team is strategizing the migration of a portion of their infrastructure to the AWS cloud. Recognizing the scale of this undertaking, they have opted to approach the migration in incremental steps rather than as a single massive transition. To achieve this, they have segmented large tasks into smaller, more manageable units. This granular approach enables the team to execute the migration in gradual phases, ensuring smoother implementation and minimizing disruption to ongoing operations. By breaking down large tasks into smaller, more manageable units, the Nautilus DevOps team can systematically progress through each stage, allowing for better control, risk mitigation, and optimization of resources throughout the migration process.

For this task, create an AMI from an existing EC2 instance named xfusion-ec2 using Terraform.

Name of the AMI should be xfusion-ec2-ami, make sure AMI is in available state.

The Terraform working directory is /home/bob/terraform. Update the main.tf file (do not create a separate .tf file) to create the AMI.

Note: Right-click under the EXPLORER section in VS Code and select Open in Integrated Terminal to launch the terminal.

## Task Requirements

1. Create an AMI from an existing EC2 instance named `xfusion-ec2` using Terraform.
2. Name of the AMI should be `xfusion-ec2-ami`, make sure AMI is in available state.
3. The Terraform working directory is `/home/bob/terraform`. Update the `main.tf` file (do not create a separate `.tf` file) to create the AMI.
4. Note: Right-click under the **EXPLORER** section in VS Code and select Open in Integrated Terminal to launch the terminal.

## Solution

This lab provided `main.tf` with the existing `aws_instance.ec2` resource. The instance must remain unchanged. The AMI resource can reference that managed resource directly through `aws_instance.ec2.id`; a `data` block is not needed when the source instance is already part of the current Terraform configuration and state. The existing `provider.tf` also remains unchanged.

### 📝 Step 1: Update the existing main.tf

In the VS Code Explorer, open `/home/bob/terraform/main.tf` and keep the existing EC2 resource. Add the `aws_ami_from_instance` resource below it:

```hcl
resource "aws_instance" "ec2" {
  ami           = "ami-0c101f26f147fa7fd"
  instance_type = "t2.micro"
  vpc_security_group_ids = [
    "sg-6ad9bffaff3a2a00c"
  ]

  tags = {
    Name = "xfusion-ec2"
  }
}

resource "aws_ami_from_instance" "xfusion_ec2_ami" {
  name               = "xfusion-ec2-ami"
  source_instance_id = aws_instance.ec2.id
}
```

> **Why:** The existing `aws_instance.ec2` resource represents the source instance. `aws_ami_from_instance` creates an Amazon Machine Image (AMI) from that instance. `name` sets the AMI name, while `source_instance_id` receives the instance ID through a Terraform resource reference. This reference creates an implicit dependency and avoids an unnecessary lookup through a `data` block. The existing EC2 configuration and `provider.tf` are not changed.

### ⚙️ Step 2: Initialize Terraform

From the integrated terminal in `/home/bob/terraform`, run:

```bash
terraform init
```

Successful output includes:

```text
Terraform has been successfully initialized!
```

> **Why:** `terraform init` prepares the working directory and installs or reuses the provider plugins required by the existing EC2 resource and the new AMI resource. It also checks the dependency lock file.

### 🚀 Step 3: Apply the configuration

Run:

```bash
terraform apply --auto-approve
```

The lab returned:

```text
aws_instance.ec2: Refreshing state... [id=i-e906299b732ad6dda]

Plan: 1 to add, 0 to change, 0 to destroy.
aws_ami_from_instance.xfusion_ec2_ami: Creating...
aws_ami_from_instance.xfusion_ec2_ami: Creation complete after 5s [id=ami-39b242a29bfa5dc98]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

> **Why:** `terraform apply` refreshes the existing instance state and creates the new AMI. `--auto-approve` accepts the plan without an interactive prompt, which is suitable for this controlled lab. The plan shows that Terraform will add only the AMI and will not recreate or modify the source instance.

### ✅ Step 4: Verify

The successful apply confirmed:

```text
Source instance: xfusion-ec2
Instance ID: i-e906299b732ad6dda
AMI name: xfusion-ec2-ami
AMI ID: ami-39b242a29bfa5dc98
Plan: 1 to add, 0 to change, 0 to destroy.
Resources: 1 added, 0 changed, 0 destroyed.
```

The AMI was created successfully and reached the required available state.

> **Why:** Terraform reported `Creation complete` for `aws_ami_from_instance.xfusion_ec2_ami`, returned the AMI ID, and confirmed that only one new resource was added. The lab validation confirmed the AMI was available.

## Best Practices

- **Preserve the source instance.** Add the AMI resource without modifying or replacing the existing EC2 resource.
- **Prefer resource references for managed resources.** Use `aws_instance.ec2.id` when the source instance is already managed by the same Terraform configuration.
- **Use a data block for external resources.** If an instance exists outside the current Terraform configuration and state, locate it with an appropriate data source instead.
- **Review the plan before applying.** Confirm that the plan adds only the AMI and does not recreate or change the source instance.
- **Treat each lab as independent.** Use the files supplied by the current lab and do not assume resources or state from a previous lab remain.

### 📚 Official Documentation

- [AWS CreateImage example](https://docs.aws.amazon.com/code-library/latest/ug/ec2_example_ec2_CreateImage_section.html)
- [AWS provider `aws_ami_from_instance` resource](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/ami_from_instance)
- [AWS provider `aws_instance` resource](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/instance)
- [Terraform `init` command](https://developer.hashicorp.com/terraform/cli/commands/init)
- [Terraform `apply` command](https://developer.hashicorp.com/terraform/cli/commands/apply)
