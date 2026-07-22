# Day 23: Data Migration Between S3 Buckets Using AWS CLI

As part of a data migration project, the team lead has tasked the team with migrating data from an existing S3 bucket to a new S3 bucket. The existing bucket contains a substantial amount of data that must be accurately transferred to the new bucket. The team is responsible for creating the new S3 bucket and ensuring that all data from the existing bucket is copied or synced to the new bucket completely and accurately. It is imperative to perform thorough verification steps to confirm that all data has been successfully transferred to the new bucket without any loss or corruption.

As a member of the Nautilus DevOps Team, your task is to perform the following:

**Create a New Private S3 Bucket:** Name the bucket `xfusion-sync-13122`.

**Data Migration:** Migrate the entire data from the existing `xfusion-s3-28684` bucket to the new `xfusion-sync-13122` bucket.

**Ensure Data Consistency:** Ensure that both buckets have the same data.

**Use AWS CLI:** Use the AWS CLI to perform the creation and data migration tasks.

## Specific Requirements:

1. **Create a New Private S3 Bucket:** Name the bucket `xfusion-sync-13122`.
2. **Data Migration:** Migrate the entire data from the existing `xfusion-s3-28684` bucket to the new `xfusion-sync-13122` bucket.
3. **Ensure Data Consistency:** Ensure that both buckets have the same data.
4. **Use AWS CLI:** Use the AWS CLI to perform the creation and data migration tasks.

## Solution

The migration uses `aws s3 sync` with `--delete`: it copies missing or outdated objects and removes destination objects that are not present in the source. The result is checked using object counts, total bytes, a second dry-run synchronization, and the destination bucket's public access configuration.

### 📦 Variables

```bash
AWS_REGION="us-east-1"
SOURCE_BUCKET="xfusion-s3-28684"
DESTINATION_BUCKET="xfusion-sync-13122"
```

### 🔎 Step 1: Verify the source and destination bucket state

```bash
aws s3api head-bucket \
  --bucket "$SOURCE_BUCKET" \
  --output json

aws s3api head-bucket \
  --bucket "$DESTINATION_BUCKET" \
  --output json
```

The source bucket was accessible, while the destination bucket did not exist yet. The source bucket is in `us-east-1`:

```text
{
    "BucketArn": "arn:aws:s3:::xfusion-s3-28684",
    "BucketRegion": "us-east-1",
    "AccessPointAlias": false
}
```

After the destination was created, both buckets were confirmed as accessible:

```text
{
    "BucketArn": "arn:aws:s3:::xfusion-s3-28684",
    "BucketRegion": "us-east-1",
    "AccessPointAlias": false
}
{
    "BucketArn": "arn:aws:s3:::xfusion-sync-13122",
    "BucketRegion": "us-east-1",
    "AccessPointAlias": false
}
```

> **Why:** `head-bucket` checks whether a bucket exists and whether the caller can access it. `--bucket` identifies the bucket being checked, and `--output json` displays the response fields in JSON. Checking the existing bucket first prevents treating the source as a resource that needs to be created. Checking the target before creation makes the workflow safe to rerun after a partial migration.

### 🪣 Step 2: Create and secure the private destination bucket

```bash
aws s3api create-bucket \
  --bucket "$DESTINATION_BUCKET" \
  --region "$AWS_REGION"

aws s3api put-public-access-block \
  --bucket "$DESTINATION_BUCKET" \
  --public-access-block-configuration BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true
```

The bucket was created successfully:

```text
{
    "Location": "/xfusion-sync-13122",
    "BucketArn": "arn:aws:s3:::xfusion-sync-13122"
}
Created destination bucket: xfusion-sync-13122
Public access block enabled for xfusion-sync-13122.
```

> **Why:** `create-bucket` creates the new S3 bucket. `--bucket` supplies its globally unique name, and `--region` places it in `us-east-1`; that Region does not require a separate `LocationConstraint` in the request. `put-public-access-block` configures the bucket-level public access protections. The four settings block public ACLs, ignore public ACLs, block public bucket policies, and restrict public bucket policies, ensuring the new bucket remains private.

### 🔄 Step 3: Synchronize the source bucket

```bash
aws s3 sync \
  "s3://$SOURCE_BUCKET" \
  "s3://$DESTINATION_BUCKET" \
  --delete \
  --only-show-errors
```

The migration completed successfully:

```text
Synchronized data from xfusion-s3-28684 to xfusion-sync-13122.
```

> **Why:** `s3 sync` compares the source and destination locations and copies objects that are missing or outdated. The two `s3://` paths identify the source and destination buckets. `--delete` removes objects from the destination when they are absent from the source, which is necessary for both buckets to contain the same data rather than merely ensuring that the destination contains a subset. `--only-show-errors` suppresses normal transfer details while preserving errors.

### 📊 Step 4: Compare object counts and total sizes

```bash
SOURCE_SUMMARY_FILE=$(mktemp)
DESTINATION_SUMMARY_FILE=$(mktemp)
trap 'rm -f "$SOURCE_SUMMARY_FILE" "$DESTINATION_SUMMARY_FILE"' EXIT

aws s3 ls \
  "s3://$SOURCE_BUCKET" \
  --recursive \
  --summarize > "$SOURCE_SUMMARY_FILE"

aws s3 ls \
  "s3://$DESTINATION_BUCKET" \
  --recursive \
  --summarize > "$DESTINATION_SUMMARY_FILE"

SOURCE_OBJECTS=$(awk -F': ' '/Total Objects:/ {print $2}' "$SOURCE_SUMMARY_FILE")
SOURCE_BYTES=$(awk -F': ' '/Total Size:/ {print $2}' "$SOURCE_SUMMARY_FILE")
DESTINATION_OBJECTS=$(awk -F': ' '/Total Objects:/ {print $2}' "$DESTINATION_SUMMARY_FILE")
DESTINATION_BYTES=$(awk -F': ' '/Total Size:/ {print $2}' "$DESTINATION_SUMMARY_FILE")

echo "Source objects: $SOURCE_OBJECTS"
echo "Source total bytes: $SOURCE_BYTES"
echo "Destination objects: $DESTINATION_OBJECTS"
echo "Destination total bytes: $DESTINATION_BYTES"
```

The two buckets matched:

```text
Source objects: 3945
Source total bytes: 85570923
Destination objects: 3945
Destination total bytes: 85570923
Object counts and total sizes match.
```

> **Why:** `s3 ls` lists objects at an S3 path. `--recursive` includes every object below the bucket path, and `--summarize` adds the total object count and total size. The output is redirected to temporary files so a bucket with thousands of objects does not flood the terminal. Comparing both summaries checks that the migration transferred the same number of objects and the same total amount of data.

### ✅ Step 5: Verify

```bash
aws s3 sync \
  "s3://$SOURCE_BUCKET" \
  "s3://$DESTINATION_BUCKET" \
  --delete \
  --dryrun

aws s3api get-public-access-block \
  --bucket "$DESTINATION_BUCKET" \
  --query "PublicAccessBlockConfiguration" \
  --output table

echo "Source bucket: $SOURCE_BUCKET"
echo "Destination bucket: $DESTINATION_BUCKET"
echo "Source objects: $SOURCE_OBJECTS"
echo "Destination objects: $DESTINATION_OBJECTS"
echo "Source total bytes: $SOURCE_BYTES"
echo "Destination total bytes: $DESTINATION_BYTES"
```

The dry-run produced no synchronization actions, and the destination bucket had all public access protections enabled:

```text
No synchronization changes remain.
-----------------------------------
|      GetPublicAccessBlock       |
+-------------------------+-------+
|  BlockPublicAcls        |  True |
|  BlockPublicPolicy      |  True |
|  IgnorePublicAcls       |  True |
|  RestrictPublicBuckets  |  True |
+-------------------------+-------+
Source bucket: xfusion-s3-28684
Destination bucket: xfusion-sync-13122
Source objects: 3945
Destination objects: 3945
Source total bytes: 85570923
Destination total bytes: 85570923
Migration verification completed successfully.
```

> **Why:** Running `s3 sync` again with `--dryrun` calculates what would change without modifying either bucket. An empty result confirms that the source and destination are synchronized according to the CLI's comparison rules. `get-public-access-block` reads the destination's public access configuration; `--query` selects only that configuration and `--output table` formats the four Boolean protections for review. Matching counts, matching total bytes, no dry-run actions, and all four protections set to `True` confirm the challenge requirements.

## Best Practices

- **Verify before creating.** Check the source and destination bucket state first so a rerun does not overwrite or recreate resources unexpectedly.
- **Use `--delete` deliberately.** It removes destination-only objects, which is appropriate when the goal is identical bucket contents but should not be used casually against a bucket containing unrelated data.
- **Validate more than one metric.** Object counts and total bytes provide a fast completeness check, while a dry-run synchronization detects remaining missing, changed, or extra objects.
- **Keep migration targets private.** Enable all four S3 Block Public Access settings unless the application explicitly requires public access.
- **Plan for large transfers.** `s3 sync` is suitable for this lab, but production migrations may also require transfer monitoring, retry planning, versioning decisions, and checksum or inventory-based validation.

### 📚 Official Documentation

- [head-bucket — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/s3api/head-bucket.html)
- [create-bucket — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/s3api/create-bucket.html)
- [put-public-access-block — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/s3api/put-public-access-block.html)
- [Using high-level S3 commands in the AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/cli-services-s3-commands.html)
- [sync — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/s3/sync.html)
- [ls — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/s3/ls.html)
- [get-public-access-block — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/s3api/get-public-access-block.html)
