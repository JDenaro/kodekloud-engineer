# Task 12: Create Public S3 Bucket Using Terraform

As part of the data migration process, the Nautilus DevOps team is actively creating several S3 buckets on AWS. They plan to utilize both private and public S3 buckets to store the relevant data. Given the ongoing migration of other infrastructure to AWS, it is logical to consolidate data storage within the AWS environment as well.

**Create a** **public** **S3 bucket** named nautilus-s3-21692 using Terraform.

**Ensure the bucket is accessible publicly once created** by setting the proper ACL.

The Terraform working directory is /home/bob/terraform. Create the main.tf file (do not create a different .tf file) to accomplish this task.

**Notes:**

Create the resources only in the us-east-1 region.
Right-click under the EXPLORER section in VS Code and select Open in Integrated Terminal.
The name of the S3 bucket should be based on nautilus-s3-21692.
You can use the ACL settings to ensure the bucket is publicly accessible.

## Task Requirements

1. **Create a** **public** **S3 bucket** named `nautilus-s3-21692` using Terraform.
2. **Ensure the bucket is accessible publicly once created** by setting the proper ACL.
3. The Terraform working directory is `/home/bob/terraform`. Create the `main.tf` file (do not create a different `.tf` file) to accomplish this task.
4. Create the resources only in the `us-east-1` region.
5. Right-click under the EXPLORER section in VS Code and select Open in Integrated Terminal.
6. The name of the S3 bucket should be based on `nautilus-s3-21692`.
7. You can use the ACL settings to ensure the bucket is publicly accessible.

## Solution

This Terraform lab started with a fresh workspace and no `main.tf`, so the file was created manually from the VS Code Explorer. The bucket, its ownership controls, its bucket-level public access settings, and its public ACL were defined together in `main.tf`. The existing `provider.tf` was left unchanged and supplies the `us-east-1` region.

The ownership controls are important because new S3 buckets commonly use `BucketOwnerEnforced`, which disables ACLs. `BucketOwnerPreferred` enables ACLs so that the requested `public-read` ACL can be applied. The bucket-level public access block is also disabled because a public ACL would otherwise be rejected by those settings.

### 📦 Variables

```bash
AWS_REGION="us-east-1"
BUCKET_NAME="nautilus-s3-21692"
```

> **Why:** `AWS_REGION` records the only allowed AWS region for this task, and `BUCKET_NAME` records the exact globally unique S3 bucket name supplied by the challenge. These values are documented as context; the lab's existing `provider.tf` supplies the region to Terraform.

### 📝 Step 1: Create `main.tf`

In the VS Code Explorer, create `/home/bob/terraform/main.tf` with the following content:

```hcl
resource "aws_s3_bucket" "nautilus_s3_21692" {
  bucket = "nautilus-s3-21692"
}

resource "aws_s3_bucket_ownership_controls" "nautilus_s3_21692" {
  bucket = aws_s3_bucket.nautilus_s3_21692.id

  rule {
    object_ownership = "BucketOwnerPreferred"
  }
}

resource "aws_s3_bucket_public_access_block" "nautilus_s3_21692" {
  bucket = aws_s3_bucket.nautilus_s3_21692.id

  block_public_acls       = false
  block_public_policy     = false
  ignore_public_acls      = false
  restrict_public_buckets = false
}

resource "aws_s3_bucket_acl" "nautilus_s3_21692" {
  depends_on = [
    aws_s3_bucket_ownership_controls.nautilus_s3_21692,
    aws_s3_bucket_public_access_block.nautilus_s3_21692
  ]

  bucket = aws_s3_bucket.nautilus_s3_21692.id
  acl    = "public-read"
}
```

> **Why:** `aws_s3_bucket` creates the S3 bucket with the exact requested name. `aws_s3_bucket_ownership_controls` sets `BucketOwnerPreferred`, which keeps ACLs enabled for the bucket. `aws_s3_bucket_public_access_block` sets all four bucket-level blocking options to `false`, allowing a public ACL to be applied. `aws_s3_bucket_acl` applies the canned `public-read` ACL. Its `bucket` argument references the bucket created by Terraform, and `depends_on` ensures that the ownership and public-access settings are ready before Terraform applies the ACL. All four resources are in `main.tf`, while `provider.tf` remains unchanged.

### ⚙️ Step 2: Initialize and apply Terraform

From the integrated terminal in `/home/bob/terraform`, run:

```bash
terraform init
terraform apply --auto-approve
```

The lab returned the following relevant output:

```text
Plan: 4 to add, 0 to change, 0 to destroy.
aws_s3_bucket.nautilus_s3_21692: Creation complete after 0s [id=nautilus-s3-21692]
aws_s3_bucket_ownership_controls.nautilus_s3_21692: Creation complete after 1s [id=nautilus-s3-21692]
aws_s3_bucket_public_access_block.nautilus_s3_21692: Creation complete after 1s [id=nautilus-s3-21692]
aws_s3_bucket_acl.nautilus_s3_21692: Creation complete after 0s [id=nautilus-s3-21692,public-read]

Apply complete! Resources: 4 added, 0 changed, 0 destroyed.
```

> **Why:** `terraform init` prepares the fresh working directory and installs or reuses the AWS provider. `terraform apply` compares the configuration with Terraform state and creates the four declared resources. `--auto-approve` accepts the generated plan without an additional confirmation prompt. The result confirms that the bucket and its public-access configuration were created successfully.

### ✅ Step 3: Verify the bucket ACL

Run:

```bash
aws s3api get-bucket-acl \
  --bucket nautilus-s3-21692 \
  --region us-east-1 \
  --query 'Grants[?Grantee.URI==`http://acs.amazonaws.com/groups/global/AllUsers`].Permission' \
  --output text
```

The expected result is:

```text
READ
```

This confirms that the bucket ACL grants public read permission to the S3 `AllUsers` group. The successful Terraform output already confirmed that the applied ACL was `public-read`.

> **Why:** `aws s3api get-bucket-acl` retrieves the bucket's access control list. `--bucket` selects the bucket, `--region` selects `us-east-1`, `--query` extracts the permission granted to the public `AllUsers` group, and `--output text` prints only the permission value. `READ` confirms the public-read ACL required by the challenge.

## Best Practices

- **Use separate S3 resources for separate concerns.** Keep bucket creation, ownership controls, public-access settings, and ACL management explicit so their dependency order is clear.
- **Enable ACLs deliberately.** `BucketOwnerPreferred` is used only because this challenge specifically requires an ACL; modern S3 designs should generally prefer IAM or bucket policies when public access is not required.
- **Limit public access to the requested bucket.** The configuration changes public-access settings only for `nautilus-s3-21692`, not for the entire AWS account.
- **Treat public buckets as exceptional.** Public read access can expose data, so apply it only when the application requirement explicitly calls for it.
- **Treat each Terraform lab as independent.** Create `main.tf` when it is absent and do not assume provider state, lock files, resources, or configurations from another lab remain available.

### 📚 Official Documentation

- [Terraform AWS provider `aws_s3_bucket` resource](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/s3_bucket)
- [Terraform AWS provider `aws_s3_bucket_ownership_controls` resource](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/s3_bucket_ownership_controls)
- [Terraform AWS provider `aws_s3_bucket_public_access_block` resource](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/s3_bucket_public_access_block)
- [Terraform AWS provider `aws_s3_bucket_acl` resource](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/s3_bucket_acl)
- [Amazon S3 Block Public Access](https://docs.aws.amazon.com/AmazonS3/latest/userguide/access-control-block-public-access.html)
- [Amazon S3 Object Ownership](https://docs.aws.amazon.com/AmazonS3/latest/userguide/about-object-ownership.html)
- [AWS CLI `get-bucket-acl` command](https://docs.aws.amazon.com/cli/latest/reference/s3api/get-bucket-acl.html)
- [Terraform `init` command](https://developer.hashicorp.com/terraform/cli/commands/init)
- [Terraform `apply` command](https://developer.hashicorp.com/terraform/cli/commands/apply)
