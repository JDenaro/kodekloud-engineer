# Task 09: Create EBS Volume Using Terraform

The Nautilus DevOps team is strategizing the migration of a portion of their infrastructure to the AWS cloud. Recognizing the scale of this undertaking, they have opted to approach the migration in incremental steps rather than as a single massive transition. To achieve this, they have segmented large tasks into smaller, more manageable units. This granular approach enables the team to execute the migration in gradual phases, ensuring smoother implementation and minimizing disruption to ongoing operations. By breaking down large tasks into smaller, more manageable units, the Nautilus DevOps team can systematically progress through each stage, allowing for better control, risk mitigation, and optimization of resources throughout the migration process.

For this task, create an AWS EBS volume using Terraform with the following requirements:

Name of the volume should be datacenter-volume.

Volume type must be gp3.

Volume size must be 2 GiB.

Ensure the volume is created in us-east-1.

The Terraform working directory is /home/bob/terraform. Create the main.tf file (do not create a different .tf file) to accomplish this task.

Note: Right-click under the EXPLORER section in VS Code and select Open in Integrated Terminal to launch the terminal.

## Task Requirements

1. Name of the volume should be `datacenter-volume`.
2. Volume type must be `gp3`.
3. Volume size must be `2 GiB`.
4. Ensure the volume is created in `us-east-1`.
5. The Terraform working directory is `/home/bob/terraform`. Create the `main.tf` file (do not create a different `.tf` file) to accomplish this task.
6. Note: Right-click under the **EXPLORER** section in VS Code and select Open in Integrated Terminal to launch the terminal.

## Solution

The lab starts with a fresh Terraform workspace, so `main.tf` must be created manually. The existing `provider.tf` remains unchanged and supplies the AWS provider configuration for `us-east-1`. EBS volumes are created in an Availability Zone rather than directly in a region; `us-east-1a` is an Availability Zone within `us-east-1`.

### 📄 Step 1: Create main.tf in the fresh Terraform workspace

In the VS Code Explorer, open `/home/bob/terraform`. Create a new file named `main.tf` and add:

```hcl
resource "aws_ebs_volume" "datacenter_volume" {
  availability_zone = "us-east-1a"
  size              = 2
  type              = "gp3"

  tags = {
    Name = "datacenter-volume"
  }
}
```

> **Why:** `aws_ebs_volume` creates an Amazon Elastic Block Store (EBS) volume. `availability_zone` places it in `us-east-1a`, which ensures it belongs to the `us-east-1` region. `type = "gp3"` selects the requested general-purpose SSD volume type. The `size` argument is an integer measured in GiB, so `size = 2` creates a 2 GiB volume. The `Name` tag gives the volume the requested name. No provider block is added because `provider.tf` already exists.

### 📏 GiB versus GB

For EBS volumes, the `size` parameter is expressed in **GiB** (gibibytes), not decimal GB (gigabytes):

- **1 GiB** = 1,073,741,824 bytes (`2^30` bytes).
- **1 GB** = 1,000,000,000 bytes (decimal units).

Therefore, `size = 2` means 2 GiB, or 2,147,483,648 bytes. Terraform expects a numeric value, so units such as `"2GiB"` must not be added to the argument.

### ⚙️ Step 2: Initialize Terraform

From the integrated terminal in `/home/bob/terraform`, run:

```bash
terraform init
```

Successful output includes:

```text
Terraform has been successfully initialized!
```

> **Why:** `terraform init` prepares the fresh working directory, installs the AWS provider plugin, and creates or updates the dependency lock file.

### 🚀 Step 3: Apply the configuration

Run:

```bash
terraform apply --auto-approve
```

The lab returned:

```text
Plan: 1 to add, 0 to change, 0 to destroy.
aws_ebs_volume.datacenter_volume: Creating...
aws_ebs_volume.datacenter_volume: Still creating... [10s elapsed]
aws_ebs_volume.datacenter_volume: Creation complete after 11s [id=vol-8a2781c653774f538]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

> **Why:** `terraform apply` creates the EBS volume described by `main.tf`. `--auto-approve` accepts the plan without an interactive prompt, which is suitable for this controlled lab. The completion output returns the volume ID assigned by AWS.

### ✅ Step 4: Verify

The successful apply confirmed:

```text
Volume ID: vol-8a2781c653774f538
Name tag: datacenter-volume
Volume type: gp3
Size: 2 GiB
Availability Zone: us-east-1a
Region: us-east-1
Resources: 1 added, 0 changed, 0 destroyed
```

The requested 2 GiB `gp3` EBS volume was created successfully in the `us-east-1` region.

> **Why:** The apply output confirms that Terraform created exactly one volume. The configuration and successful resource creation match the required name, volume type, size, and region.

## Best Practices

- **Document storage units explicitly.** EBS `size` values are always expressed in GiB; use descriptive variable names such as `volume_size_gib` when a configuration is shared with others.
- **Choose the Availability Zone deliberately.** An EBS volume can be attached only to instances in the same Availability Zone.
- **Use tags consistently.** A descriptive `Name` tag makes storage resources easier to identify and manage.
- **Prefer gp3 for general-purpose SSD storage.** It provides independently configurable performance and is the requested volume type for this lab.
- **Treat each lab as independent.** Create a fresh `main.tf` in every new Terraform workspace instead of assuming files or state from a previous lab remain.

### 📚 Official Documentation

- [AWS EBS volume creation](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-creating-volume.html)
- [AWS `CreateVolume` API](https://docs.aws.amazon.com/AWSEC2/latest/APIReference/API_CreateVolume.html)
- [AWS provider `aws_ebs_volume` resource](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/ebs_volume)
- [Terraform `init` command](https://developer.hashicorp.com/terraform/cli/commands/init)
- [Terraform `apply` command](https://developer.hashicorp.com/terraform/cli/commands/apply)
