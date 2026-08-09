# Task 17: Create DynamoDB Table Using Terraform

The Nautilus DevOps team needs to set up a DynamoDB table for storing user data. They need to create a DynamoDB table with the following specifications:

1) The table name should be nautilus-users.

2) The primary key should be nautilus_id (String).

3) The table should use PAY_PER_REQUEST billing mode.

Use Terraform to create this DynamoDB table. The Terraform working directory is /home/bob/terraform. Create the main.tf file (do not create a different .tf file) to create the DynamoDB table.

Note: Right-click under the EXPLORER section in VS Code and select Open in Integrated Terminal to launch the terminal.

## Task Requirements

1. The table name should be `nautilus-users`.
2. The primary key should be `nautilus_id` (String).
3. The table should use `PAY_PER_REQUEST` billing mode.
4. Use Terraform to create this DynamoDB table. The Terraform working directory is `/home/bob/terraform`. Create the `main.tf` file (do not create a different `.tf` file) to create the DynamoDB table.
5. Note: Right-click under the **EXPLORER** section in VS Code and select Open in Integrated Terminal to launch the terminal.

## Solution

This Terraform lab started with a fresh workspace, so `main.tf` had to be created manually from the VS Code Explorer. The only resource required was an `aws_dynamodb_table` resource. The table uses `nautilus_id` as its partition key, defines that key as a String, and uses on-demand billing. The existing `provider.tf` was left unchanged.

### 📦 Variables

```bash
AWS_REGION="us-east-1"
TABLE_NAME="nautilus-users"
PARTITION_KEY="nautilus_id"
```

> **Why:** `AWS_REGION` records the lab region, `TABLE_NAME` records the exact DynamoDB table name, and `PARTITION_KEY` records the required primary-key attribute. The existing `provider.tf` supplies the AWS provider configuration.

### 📝 Step 1: Create `main.tf`

In the VS Code Explorer, create `/home/bob/terraform/main.tf` with the following content:

```hcl
resource "aws_dynamodb_table" "nautilus_users" {
  name         = "nautilus-users"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "nautilus_id"

  attribute {
    name = "nautilus_id"
    type = "S"
  }
}
```

> **Why:** `aws_dynamodb_table` creates and manages a DynamoDB table. `name` sets the table name. `billing_mode = "PAY_PER_REQUEST"` selects on-demand capacity, so no `read_capacity` or `write_capacity` values are configured. `hash_key` identifies the partition key. The `attribute` block defines that key, and `type = "S"` means String. The attribute name must match the `hash_key` value.

### ⚙️ Step 2: Initialize Terraform

From the integrated terminal in `/home/bob/terraform`, run:

```bash
terraform init
```

Successful output includes:

```text
Terraform has been successfully initialized!
```

> **Why:** `terraform init` prepares the fresh working directory and installs or reuses the AWS provider required by the DynamoDB table resource.

### 🚀 Step 3: Create the DynamoDB table

Run:

```bash
terraform apply --auto-approve
```

The lab returned:

```text
Plan: 1 to add, 0 to change, 0 to destroy.
aws_dynamodb_table.nautilus_users: Creating...
aws_dynamodb_table.nautilus_users: Creation complete after 2s [id=nautilus-users]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

The Terraform plan showed the requested values:

```text
name         = "nautilus-users"
hash_key     = "nautilus_id"
billing_mode = "PAY_PER_REQUEST"
attribute:
  name = "nautilus_id"
  type = "S"
```

> **Why:** `terraform apply` compares the configuration with Terraform state and creates the table. `--auto-approve` accepts the generated plan without an additional confirmation prompt. The plan and completion output confirm that exactly one table was created with the requested key and billing configuration.

### ✅ Step 4: Verify the DynamoDB table

Run:

```bash
aws dynamodb describe-table \
  --table-name nautilus-users \
  --region us-east-1 \
  --query 'Table.[TableName,KeySchema[0].AttributeName,AttributeDefinitions[0].AttributeType,BillingModeSummary.BillingMode,TableStatus]' \
  --output table
```

The expected values are:

```text
nautilus-users
nautilus_id
S
PAY_PER_REQUEST
ACTIVE
```

The lab's first query used `Table.BillingMode`, which is not the field returned by `DescribeTable`, so that column displayed `None`. The correct response path is `Table.BillingModeSummary.BillingMode`; the Terraform plan already confirmed that the table was configured with `PAY_PER_REQUEST`.

> **Why:** `aws dynamodb describe-table` retrieves the table's metadata. `--table-name` selects the table, `--region` selects `us-east-1`, `--query` extracts the table name, key attribute, key type, billing mode, and status, and `--output table` formats the result for readability. `BillingModeSummary.BillingMode` is the correct field for checking whether the table uses `PAY_PER_REQUEST`.

## Best Practices

- **Use on-demand billing when requested.** `PAY_PER_REQUEST` avoids manually setting read and write capacity units and adapts to variable traffic.
- **Define only key attributes.** Declare `nautilus_id` because it is used as the partition key; do not add unrelated attributes to the table definition.
- **Keep the key definitions consistent.** The `hash_key` value and the `attribute.name` value must match exactly, including capitalization.
- **Use the correct API response path.** DynamoDB exposes the billing mode under `BillingModeSummary.BillingMode`, not directly under `Table.BillingMode`.
- **Treat each Terraform lab as independent.** Create `main.tf` when it is absent and do not assume resources, state, provider locks, or files from another lab remain available.

### 📚 Official Documentation

- [Terraform AWS provider `aws_dynamodb_table` resource](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/dynamodb_table)
- [AWS CLI `describe-table` command](https://docs.aws.amazon.com/cli/latest/reference/dynamodb/describe-table.html)
- [DynamoDB read/write capacity modes](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/HowItWorks.ReadWriteCapacityMode.html)
- [Terraform `init` command](https://developer.hashicorp.com/terraform/cli/commands/init)
- [Terraform `apply` command](https://developer.hashicorp.com/terraform/cli/commands/apply)
