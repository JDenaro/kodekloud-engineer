# Task 22: CloudFormation Template Deployment Using Terraform

The Nautilus DevOps team is working on automating infrastructure deployment using AWS CloudFormation. As part of this effort, they need to create a CloudFormation stack that provisions an S3 bucket with versioning enabled.

Create a CloudFormation stack named xfusion-stack using Terraform. This stack should contain an S3 bucket named xfusion-bucket-26670 as a resource, and the bucket must have versioning enabled. The Terraform working directory is /home/bob/terraform. Create the main.tf file (do not create a different .tf file) to accomplish this task.

Note: Right-click under the EXPLORER section in VS Code and select Open in Integrated Terminal to launch the terminal.

## Task Requirements

1. Create a CloudFormation stack named `xfusion-stack` using Terraform.
2. The stack must contain an S3 bucket named `xfusion-bucket-26670`.
3. The S3 bucket must have versioning enabled.
4. The Terraform working directory is `/home/bob/terraform`. Create the `main.tf` file (do not create a different `.tf` file) to accomplish this task.
5. Right-click under the **EXPLORER** section in VS Code and select Open in Integrated Terminal to launch the terminal.

## Solution

This Terraform lab starts with a fresh workspace, so `main.tf` must be created manually if it is absent. Terraform creates the CloudFormation stack, while the embedded CloudFormation template defines the S3 bucket and its versioning configuration. The existing `provider.tf` supplies the AWS provider configuration and should remain unchanged.

### 📦 Variables

```bash
AWS_REGION="us-east-1"
STACK_NAME="xfusion-stack"
BUCKET_NAME="xfusion-bucket-26670"
```

> **Why:** These values record the AWS region and exact stack and bucket names required by the challenge. They are reference values for the guide; the Terraform resource and embedded template use the literal values directly. The existing `provider.tf` supplies the AWS provider configuration.

### 📝 Step 1: Create `main.tf`

In the VS Code Explorer, open `/home/bob/terraform` and create `main.tf` with this content:

```hcl
resource "aws_cloudformation_stack" "xfusion_stack" {
  name = "xfusion-stack"

  template_body = <<-YAML
    AWSTemplateFormatVersion: '2010-09-09'
    Resources:
      XfusionBucket:
        Type: AWS::S3::Bucket
        Properties:
          BucketName: xfusion-bucket-26670
          VersioningConfiguration:
            Status: Enabled
  YAML
}
```

> **Why:** `aws_cloudformation_stack` creates and manages a CloudFormation stack through Terraform. `name` sets the stack name. `template_body` embeds the CloudFormation template directly in `main.tf`; the `AWS::S3::Bucket` resource creates the bucket, and `VersioningConfiguration` with `Status: Enabled` turns on S3 object versioning. No separate CloudFormation template file is required.

### 📂 Step 2: Open the Terraform working directory

From the integrated terminal, run:

```bash
cd /home/bob/terraform
```

> **Why:** `cd` changes the current terminal directory. Terraform commands must be run from the directory containing `main.tf` and the existing provider configuration.

### ⚙️ Step 3: Initialize Terraform

```bash
terraform init
```

The lab returned:

```text
Terraform has been successfully initialized!
```

> **Why:** `terraform init` prepares the working directory and installs or reuses the AWS provider required by the CloudFormation stack resource.

### 🚀 Step 4: Create the CloudFormation stack

```bash
terraform apply --auto-approve
```

The lab returned:

```text
Plan: 1 to add, 0 to change, 0 to destroy.
aws_cloudformation_stack.xfusion_stack: Creating...
aws_cloudformation_stack.xfusion_stack: Still creating... [10s elapsed]
aws_cloudformation_stack.xfusion_stack: Creation complete after 10s [id=arn:aws:cloudformation:us-east-1:000000000000:stack/xfusion-stack/a9f3b52c-bb3e-40fd-a7df-f845de4ace8f]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

> **Why:** `terraform apply` compares the desired configuration with Terraform state and asks AWS to create the CloudFormation stack. `--auto-approve` accepts the generated execution plan without requesting an additional confirmation. The `Still creating` message is expected while CloudFormation provisions the bucket.

### ✅ Step 5: Verify the CloudFormation stack

```bash
aws cloudformation describe-stacks \
  --stack-name "xfusion-stack" \
  --region "us-east-1" \
  --query 'Stacks[0].[StackName,StackStatus]' \
  --output table
```

The lab returned:

```text
---------------------
|  DescribeStacks   |
+-------------------+
|  xfusion-stack    |
|  CREATE_COMPLETE  |
+-------------------+
```

> **Why:** `aws cloudformation describe-stacks` retrieves information about the stack. `--stack-name` selects `xfusion-stack`, `--region` targets `us-east-1`, `--query` selects the stack name and status, and `--output table` formats the result. `CREATE_COMPLETE` confirms that CloudFormation finished creating the stack successfully.

### 🔐 Step 6: Verify S3 versioning

```bash
aws s3api get-bucket-versioning \
  --bucket "xfusion-bucket-26670" \
  --region "us-east-1" \
  --query 'Status' \
  --output text
```

The expected result is:

```text
Enabled
```

> **Why:** `aws s3api get-bucket-versioning` retrieves the versioning configuration of the bucket. `--bucket` selects the bucket, `--region` targets `us-east-1`, `--query` returns only the versioning status, and `--output text` prints the status directly.

### 🧪 Step 7: Confirm Terraform has no pending changes

```bash
terraform plan
```

The expected result is:

```text
No changes. Your infrastructure matches the configuration.
```

> **Why:** `terraform plan` compares the configuration, Terraform state, and remote stack without changing them. `No changes` confirms that the CloudFormation stack remains synchronized with `main.tf`.

## Best Practices

- **Keep stack ownership clear.** Manage the CloudFormation stack through Terraform and avoid changing the same stack manually outside Terraform.
- **Keep the template self-contained.** Embedding the CloudFormation template in `main.tf` satisfies the requirement to create only one Terraform file.
- **Verify the stack and its resource.** `CREATE_COMPLETE` confirms stack creation, while the S3 versioning check confirms the nested bucket configuration.
- **Use versioning deliberately.** S3 versioning helps preserve previous object versions and recover from accidental overwrites or deletions.
- **Treat each Terraform lab as independent.** Create `main.tf` when it is absent and do not assume state, provider locks, or resources from another lab remain available.

### 📚 Official Documentation

- [Terraform AWS provider `aws_cloudformation_stack` resource](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/cloudformation_stack)
- [AWS CloudFormation `AWS::S3::Bucket` resource](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-s3-bucket.html)
- [CloudFormation S3 versioning configuration](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-s3-bucket-versioningconfiguration.html)
- [AWS CLI `describe-stacks` command](https://docs.aws.amazon.com/cli/latest/reference/cloudformation/describe-stacks.html)
- [AWS CLI `get-bucket-versioning` command](https://docs.aws.amazon.com/cli/latest/reference/s3api/get-bucket-versioning.html)
- [Terraform `init` command](https://developer.hashicorp.com/terraform/cli/commands/init)
- [Terraform `apply` command](https://developer.hashicorp.com/terraform/cli/commands/apply)
- [Terraform `plan` command](https://developer.hashicorp.com/terraform/cli/commands/plan)
