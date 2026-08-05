# Day 42: Building and Managing NoSQL Databases with AWS DynamoDB

The Nautilus DevOps team is developing a simple 'To-Do' application using DynamoDB to store and manage tasks efficiently. The team needs to create a DynamoDB table to hold tasks, each identified by a unique task ID. Each task will have a description and a status, which indicates the progress of the task (e.g., 'completed' or 'in-progress').

## Specific Requirements:

1. Create a DynamoDB table named `datacenter-tasks` with a primary key called `taskId` (string).
2. Insert the following tasks into the table:
   - Task 1: `taskId`: '1', description: 'Learn DynamoDB', status: 'completed'
   - Task 2: `taskId`: '2', description: 'Build To-Do App', status: 'in-progress'
3. Verify that Task 1 has a status of 'completed' and Task 2 has a status of 'in-progress'.
4. Ensure the DynamoDB table is created successfully and that both tasks are inserted correctly with the appropriate statuses.

## Solution

DynamoDB is a NoSQL database, so unlike a SQL table you don't predefine every column — only the primary key attribute(s) are declared up front. Every other attribute (like `description` or `status`) is just included on each item when you write it. Items also aren't inserted as plain JSON; each value must be wrapped with a type marker (`S` for string, `N` for number, `B` for binary), which is what makes the CLI commands below look more verbose than a typical SQL `INSERT`.

### 📦 Variables

Define all task variables in a single place before running any step.

```bash
AWS_REGION="us-east-1"
TABLE_NAME="datacenter-tasks"
PARTITION_KEY="taskId"
```

### 🔧 Step 1: Create the DynamoDB Table

```bash
aws dynamodb create-table \
  --table-name $TABLE_NAME \
  --attribute-definitions AttributeName=$PARTITION_KEY,AttributeType=S \
  --key-schema AttributeName=$PARTITION_KEY,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST \
  --region $AWS_REGION
```

> **Why:** In DynamoDB, the **primary key** is the only attribute you must define when creating a table, and it's what uniquely identifies each item (row). `--table-name` sets the unique name of the table within this account and region. Here the key is a **partition key** (`KeyType=HASH`) named `taskId` of type `S` (string) — DynamoDB uses this value to decide which internal partition stores the item, so it must be unique. `--attribute-definitions` declares the key attribute's data type (`AttributeName` is its name, `AttributeType=S` marks it as a String), while `--key-schema` says which attribute plays which role in the key. `--billing-mode PAY_PER_REQUEST` (on-demand) means you pay only for the reads/writes you actually make, instead of pre-provisioning fixed capacity — ideal for a small, unpredictable workload like this To-Do app. Finally, `--region` tells the CLI which AWS Region to create the resource in; every command in this guide passes it so everything lands in the same place.

### ⏳ Step 2: Wait for the Table to Become Active

```bash
aws dynamodb wait table-exists --table-name $TABLE_NAME --region $AWS_REGION
```

> **Why:** Creating a table is not instantaneous — DynamoDB needs a moment to provision it behind the scenes. This command blocks until the table's status is `ACTIVE`, so you don't try to insert data into a table that isn't ready yet.

### ✍️ Step 3: Insert Task 1

```bash
aws dynamodb put-item \
  --table-name $TABLE_NAME \
  --item '{
    "taskId": {"S": "1"},
    "description": {"S": "Learn DynamoDB"},
    "status": {"S": "completed"}
  }' \
  --region $AWS_REGION
```

> **Why:** `put-item` writes a single item (the NoSQL equivalent of a row) to the table. Note the `{"S": "..."}` wrapper around every value — DynamoDB requires you to state each attribute's type explicitly (`S` for string here), since it doesn't infer types the way a SQL engine does. `taskId` matches the partition key we defined in Step 1; `description` and `status` are extra attributes that didn't need to be declared anywhere in advance — that's the flexible ("schemaless") part of NoSQL.

### ✍️ Step 4: Insert Task 2

```bash
aws dynamodb put-item \
  --table-name $TABLE_NAME \
  --item '{
    "taskId": {"S": "2"},
    "description": {"S": "Build To-Do App"},
    "status": {"S": "in-progress"}
  }' \
  --region $AWS_REGION
```

> **Why:** Same operation as Step 3, for the second task. Each item in a DynamoDB table can have a different set of attributes — nothing forces Task 2 to look identical to Task 1 beyond sharing the `taskId` key — but in this app both happen to use the same shape.

### ✅ Step 5: Verify

```bash
aws dynamodb get-item \
  --table-name $TABLE_NAME \
  --key '{"taskId": {"S": "1"}}' \
  --region $AWS_REGION \
  --query "Item.status.S" --output text

aws dynamodb get-item \
  --table-name $TABLE_NAME \
  --key '{"taskId": {"S": "2"}}' \
  --region $AWS_REGION \
  --query "Item.status.S" --output text
```

> **Why:** `get-item` fetches a single item by its primary key, which you supply with `--key` (same typed `{"S": ...}` format used when writing). `--query "Item.status.S"` is a **JMESPath** expression that filters the response on the client side so only the `status` field's string value is returned instead of the full JSON — here it drills into the returned `Item`, then its `status` attribute, then the `S` (string) value. `--output text` prints that value as plain text (rather than JSON), which makes it easy to read and to use in scripts. The first command should print `completed` and the second `in-progress`, confirming both tasks were inserted with the correct status.

You can also see every item at once with a full table scan:

```bash
aws dynamodb scan --table-name $TABLE_NAME --region $AWS_REGION --output table
```

> **Why:** `scan` reads **every** item in the table and returns them all, unlike `get-item` which fetches one item by key. `--output table` formats the result as an ASCII table that's easy to eyeball in the terminal (the same `--output` flag from the previous step, just using its `table` format instead of `text`). Scans are fine for a quick check on a tiny table like this, but read the whole table, so they get slow and costly as data grows.

## Best Practices

- **Prefer on-demand billing for unpredictable or low-traffic workloads.** `PAY_PER_REQUEST` avoids the operational overhead of estimating capacity units; switch to `PROVISIONED` capacity only once traffic is stable and predictable enough to make reserved capacity cheaper.
- **Choose a partition key that distributes access evenly.** A high-cardinality key like `taskId` avoids "hot partitions," where too many requests land on the same physical partition and get throttled.
- **Use `get-item` for known keys, `query` for a range of items, and reserve `scan` for small tables or ad-hoc checks.** A `scan` reads the entire table and gets expensive as data grows.
- **Wait for resource readiness with the `wait` subcommands.** `aws dynamodb wait table-exists` (and its counterparts in other services) avoids race conditions in scripts instead of guessing with a fixed `sleep`.

### 📚 Official Documentation

- [Amazon DynamoDB FAQs](https://aws.amazon.com/dynamodb/faqs/)
- [create-table — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/dynamodb/create-table.html)
- [put-item — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/dynamodb/put-item.html)
- [get-item — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/dynamodb/get-item.html)
