# Task 13: Create Private S3 Bucket Using Terraform

As part of the data migration process, the Nautilus DevOps team is actively creating several S3 buckets on AWS using Terraform. They plan to utilize both private and public S3 buckets to store the relevant data. Given the ongoing migration of other infrastructure to AWS, it is logical to consolidate data storage within the AWS environment as well.

Create an S3 bucket using Terraform with the following details:

1) The name of the S3 bucket must be datacenter-s3-18998.

2) The S3 bucket must block all public access, making it a private bucket.

The Terraform working directory is /home/bob/terraform. Create the main.tf file (do not create a different .tf file) to accomplish this task.

**Notes:**

Use Terraform to provision the S3 bucket.
Right-click under the EXPLORER section in VS Code and select Open in Integrated Terminal.
Ensure the resources are created in the us-east-1 region.
The bucket must have block public access enabled to restrict any public access.

## Task Requirements

1. The name of the S3 bucket must be `datacenter-s3-18998`.
2. The S3 bucket must block all public access, making it a private bucket.
3. The Terraform working directory is `/home/bob/terraform`. Create the `main.tf` file (do not create a different `.tf` file) to accomplish this task.
4. Use Terraform to provision the S3 bucket.
5. Right-click under the EXPLORER section in VS Code and select Open in Integrated Terminal.
6. Ensure the resources are created in the `us-east-1` region.
7. The bucket must have block public access enabled to restrict any public access.

## Solution

This Terraform lab started with a fresh workspace, so `main.tf` had to be created manually from the VS Code Explorer. The configuration creates the requested S3 bucket and a bucket-level public access block. The existing `provider.tf` was left unchanged and supplies the `us-east-1` region.

Because this bucket must remain private, no public ACL, bucket policy, or ownership-control override is added. The public access block enables all four protections required to prevent public access.

### 📦 Variables

```bash
AWS_REGION="us-east-1"
BUCKET_NAME="datacenter-s3-18998"
```

> **Why:** `AWS_REGION` records the required AWS region, and `BUCKET_NAME` records the exact bucket name from the challenge. These values provide context for the configuration; the lab's existing `provider.tf` supplies the region to Terraform.

### 📝 Step 1: Create `main.tf`

In the VS Code Explorer, create `/home/bob/terraform/main.tf` with the following content:

```hcl
resource "aws_s3_bucket" "datacenter_s3_18998" {
  bucket = "datacenter-s3-18998"
}

resource "aws_s3_bucket_public_access_block" "datacenter_s3_18998" {
  bucket = aws_s3_bucket.datacenter_s3_18998.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
```

> **Why:** `aws_s3_bucket` creates the bucket, and `bucket` assigns the exact requested name. `aws_s3_bucket_public_access_block` manages the bucket's four public-access protections. `bucket` references the bucket resource so Terraform creates the block after the bucket exists. `block_public_acls = true` rejects public ACLs, `block_public_policy = true` rejects public bucket policies, `ignore_public_acls = true` prevents public ACLs from granting access, and `restrict_public_buckets = true` restricts public bucket policies. Together, these settings keep the bucket private.

### ⚙️ Step 2: Initialize Terraform

From the integrated terminal in `/home/bob/terraform`, run:

```bash
terraform init
```

Successful output includes:

```text
Terraform has been successfully initialized!
```

> **Why:** `terraform init` prepares the fresh working directory and installs or reuses the AWS provider needed by the S3 resources. It must complete successfully before Terraform can create the bucket.

### 🚀 Step 3: Create the private bucket

Run:

```bash
terraform apply --auto-approve
```

The lab returned:

```text
Plan: 2 to add, 0 to change, 0 to destroy.
aws_s3_bucket.datacenter_s3_18998: Creation complete after 0s [id=datacenter-s3-18998]
aws_s3_bucket_public_access_block.datacenter_s3_18998: Creation complete after 0s [id=datacenter-s3-18998]

Apply complete! Resources: 2 added, 0 changed, 0 destroyed.
```

> **Why:** `terraform apply` compares the configuration with Terraform state and creates the two declared resources. `--auto-approve` accepts the plan without an additional confirmation prompt. The result confirms that both the bucket and its public access block were created, with no unrelated changes.

### ✅ Step 4: Verify public access is blocked

Run:

```bash
aws s3api get-public-access-block \
  --bucket datacenter-s3-18998 \
  --region us-east-1 \
  --query 'PublicAccessBlockConfiguration' \
  --output json
```

The expected result is a configuration in which all four values are `true`:

```json
{
    "BlockPublicAcls": true,
    "IgnorePublicAcls": true,
    "BlockPublicPolicy": true,
    "RestrictPublicBuckets": true
}
```

> **Why:** `aws s3api get-public-access-block` retrieves the bucket's public access block configuration. `--bucket` selects the target bucket, `--region` limits the request to `us-east-1`, `--query` returns only the relevant configuration object, and `--output json` presents the settings in a readable format. All four `true` values confirm that public ACLs and public policies are blocked or ignored.

## Best Practices

- **Enable all four protections.** Configure every public access block setting so that both ACL-based and policy-based public access are covered.
- **Keep private buckets free of public ACLs.** Do not add `public-read`, `public-read-write`, or a public bucket policy to a bucket intended for private data.
- **Use a separate resource for public access controls.** `aws_s3_bucket_public_access_block` makes the security posture explicit and easy to audit.
- **Limit the change to the target bucket.** Configure the block at bucket level rather than changing account-wide S3 settings for a single lab resource.
- **Treat each Terraform lab as independent.** Create `main.tf` when it is absent and do not assume resources, state, provider locks, or files from another lab remain available.

### 📚 Official Documentation

- [Terraform AWS provider `aws_s3_bucket` resource](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/s3_bucket)
- [Terraform AWS provider `aws_s3_bucket_public_access_block` resource](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/s3_bucket_public_access_block)
- [Amazon S3 Block Public Access](https://docs.aws.amazon.com/AmazonS3/latest/userguide/access-control-block-public-access.html)
- [AWS CLI `get-public-access-block` command](https://docs.aws.amazon.com/cli/latest/reference/s3api/get-public-access-block.html)
- [Terraform `init` command](https://developer.hashicorp.com/terraform/cli/commands/init)
- [Terraform `apply` command](https://developer.hashicorp.com/terraform/cli/commands/apply)
