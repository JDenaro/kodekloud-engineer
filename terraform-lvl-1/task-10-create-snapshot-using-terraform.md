# Task 10: Create Snapshot Using Terraform

The Nautilus DevOps team has some volumes in different regions in their AWS account. They are going to setup some automated backups so that all important data can be backed up on regular basis. For now they shared some requirements to take a snapshot of one of the volumes they have.

Create a snapshot of an existing volume named datacenter-vol in us-east-1 region using terraform.

1) The name of the snapshot must be datacenter-vol-ss.

2) The description must be Datacenter Snapshot.

3) Make sure the snapshot status is completed before submitting the task.

The Terraform working directory is /home/bob/terraform. Update the main.tf file (do not create a separate .tf file) to accomplish this task.

Note: Right-click under the EXPLORER section in VS Code and select Open in Integrated Terminal to launch the terminal.

## Task Requirements

1. Create a snapshot of an existing volume named `datacenter-vol` in `us-east-1` region using terraform.
2. The name of the snapshot must be `datacenter-vol-ss`.
3. The description must be `Datacenter Snapshot`.
4. Make sure the snapshot status is `completed` before submitting the task.
5. The Terraform working directory is `/home/bob/terraform`. Update the `main.tf` file (do not create a separate `.tf` file) to accomplish this task.
6. Note: Right-click under the **EXPLORER** section in VS Code and select Open in Integrated Terminal to launch the terminal.

## Solution

This lab provided `main.tf` with the existing volume resource `aws_ebs_volume.k8s_volume`. The volume must remain unchanged. Because the source volume is already managed by the current Terraform configuration, the snapshot can reference `aws_ebs_volume.k8s_volume.id` directly; a `data` block is not required. The existing `provider.tf` also remains unchanged.

### 📝 Step 1: Update the existing main.tf

In the VS Code Explorer, open `/home/bob/terraform/main.tf` and keep the existing volume resource. Add the snapshot resource below it:

```hcl
resource "aws_ebs_volume" "k8s_volume" {
  availability_zone = "us-east-1a"
  size              = 5
  type              = "gp2"

  tags = {
    Name = "datacenter-vol"
  }
}

resource "aws_ebs_snapshot" "datacenter_vol_ss" {
  volume_id   = aws_ebs_volume.k8s_volume.id
  description = "Datacenter Snapshot"

  tags = {
    Name = "datacenter-vol-ss"
  }
}
```

> **Why:** `aws_ebs_volume.k8s_volume` is the existing source volume. `aws_ebs_snapshot` creates a point-in-time backup of an EBS volume. `volume_id` receives the source volume ID through a Terraform resource reference, creating an implicit dependency. `description` sets the requested snapshot description, and the `Name` tag supplies the requested snapshot name. The existing volume and `provider.tf` are not changed.

### ⚙️ Step 2: Initialize Terraform

From the integrated terminal in `/home/bob/terraform`, run:

```bash
terraform init
```

Successful output includes:

```text
Terraform has been successfully initialized!
```

> **Why:** `terraform init` prepares the working directory and installs or reuses the AWS provider required by the existing EBS volume and new snapshot resources. It also checks the dependency lock file.

### 🚀 Step 3: Apply the configuration

Run:

```bash
terraform apply --auto-approve
```

The lab returned:

```text
aws_ebs_volume.k8s_volume: Refreshing state... [id=vol-4246afecdab45cfb6]

Plan: 1 to add, 0 to change, 0 to destroy.
aws_ebs_snapshot.datacenter_vol_ss: Creating...
aws_ebs_snapshot.datacenter_vol_ss: Creation complete after 0s [id=snap-eadb023186691fe09]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

> **Why:** `terraform apply` refreshes the existing volume and creates only the new snapshot. `--auto-approve` accepts the plan without an interactive prompt, which is suitable for this controlled lab. The plan confirms that the source volume will not be recreated or modified.

### ✅ Step 4: Verify the snapshot state

Run:

```bash
aws ec2 describe-snapshots \
  --region us-east-1 \
  --filters "Name=tag:Name,Values=datacenter-vol-ss" \
  --query 'Snapshots[0].State' \
  --output text
```

The lab returned:

```text
completed
```

> **Why:** `aws ec2 describe-snapshots` queries the EC2 snapshots in AWS. `--region us-east-1` limits the lookup to the required region. `--filters` selects the snapshot by its `Name` tag. `--query 'Snapshots[0].State'` extracts only the snapshot state, and `--output text` prints the value without JSON formatting. The result `completed` satisfies the final requirement.

## Best Practices

- **Preserve the source volume.** Add the snapshot resource without modifying or replacing the existing EBS volume.
- **Use resource references for managed resources.** Referencing `aws_ebs_volume.k8s_volume.id` creates the dependency directly and avoids an unnecessary data lookup.
- **Verify asynchronous operations explicitly.** A snapshot may initially be pending, so confirm that its state is `completed` before considering the task finished.
- **Use descriptive tags and descriptions.** The snapshot tag and description make backups easier to identify and audit.
- **Treat each lab as independent.** Use the files supplied by the current lab and do not assume resources or state from a previous lab remain.

### 📚 Official Documentation

- [Create Amazon EBS snapshots](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-creating-snapshot.html)
- [AWS provider `aws_ebs_snapshot` resource](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/ebs_snapshot)
- [AWS provider `aws_ebs_volume` resource](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/ebs_volume)
- [AWS CLI `describe-snapshots` command](https://docs.aws.amazon.com/cli/latest/reference/ec2/describe-snapshots.html)
- [Terraform `init` command](https://developer.hashicorp.com/terraform/cli/commands/init)
- [Terraform `apply` command](https://developer.hashicorp.com/terraform/cli/commands/apply)
