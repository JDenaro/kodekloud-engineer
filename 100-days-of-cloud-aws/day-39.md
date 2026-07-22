# Day 39: Hosting a Static Website on AWS S3

The Nautilus DevOps team has been tasked with creating an internal information portal for public access. As part of this project, they need to host a static website on AWS using an S3 bucket. The S3 bucket must be configured for public access to allow external users to access the static website directly via the S3 website URL.

## Task Requirements

1. Create an S3 bucket named `nautilus-web-3074624944`.
2. Configure the S3 bucket for static website hosting with `index.html` as the index document.
3. Allow public access to the bucket so that the website is publicly accessible.
4. Upload the `index.html` file from the `/root/` directory of the AWS client host to the S3 bucket.
5. Verify that the website is accessible directly through the S3 website URL.

## Solution

### 📦 Variables

Define all task variables in a single place before running any step.

```bash
AWS_REGION="us-east-1"
BUCKET_NAME="nautilus-web-3074624944"
INDEX_DOC="index.html"
SOURCE_FILE="/root/index.html"
WEBSITE_URL="http://${BUCKET_NAME}.s3-website-${AWS_REGION}.amazonaws.com"
```

### 🪣 Step 1: Create the S3 Bucket

```bash
aws s3api create-bucket --bucket $BUCKET_NAME --region $AWS_REGION
```

```json
{
    "Location": "/nautilus-web-3074624944"
}
```

> **Why:** An **S3 bucket** is a container for files (objects) in AWS's object-storage service, and every bucket lives in a specific region. `aws s3api create-bucket` is the command that creates it: `--bucket` gives the bucket its name (which must be **globally unique** across all AWS accounts — that's why the challenge uses a long number), and `--region` picks the AWS region where the bucket will physically live. One quirk to know: `us-east-1` is S3's original default region, and it *rejects* the `--create-bucket-configuration LocationConstraint=<region>` option that other regions require — so in this region we create the bucket without it.

### 🔓 Step 2: Disable Block Public Access

By default, S3 blocks all public access. It must be disabled before a public bucket policy can be applied.

```bash
aws s3api put-public-access-block \
    --bucket $BUCKET_NAME \
    --public-access-block-configuration \
        "BlockPublicAcls=false,IgnorePublicAcls=false,BlockPublicPolicy=false,RestrictPublicBuckets=false"
```

> **Why:** **Block Public Access (BPA)** is a safety feature that stops a bucket from becoming public *by accident*, and it's turned on by default on every new bucket. Since this challenge deliberately wants a public website, we must turn it off first — otherwise the public policy in the next step would be ignored. `put-public-access-block` sets these guardrails, and `--public-access-block-configuration` takes the four switches that make up BPA: `BlockPublicAcls` and `IgnorePublicAcls` control public **ACLs** (a legacy per-object permission system), while `BlockPublicPolicy` and `RestrictPublicBuckets` control public **bucket policies**. Setting all four to `false` allows the bucket to be made public. (In production you normally want all four `true` — see Best Practices.)

### 📄 Step 3: Apply the Public-Read Bucket Policy

We use a **bucket policy** (not ACLs, which are legacy) to allow read access to any user.

```bash
aws s3api put-bucket-policy \
    --bucket $BUCKET_NAME \
    --policy "{
      \"Version\": \"2012-10-17\",
      \"Statement\": [
        {
          \"Sid\": \"PublicReadGetObject\",
          \"Effect\": \"Allow\",
          \"Principal\": \"*\",
          \"Action\": \"s3:GetObject\",
          \"Resource\": \"arn:aws:s3:::${BUCKET_NAME}/*\"
        }
      ]
    }"
```

> **Why:** A **bucket policy** is a JSON document, attached to the bucket, that spells out who can do what with it — it's how we actually grant the public read access. `put-bucket-policy` attaches it, and `--policy` carries the JSON. Reading the policy's parts: `Version` is the policy language date (always `2012-10-17`); `Statement` is the list of rules; `Sid` is just a label for the rule; `Effect: Allow` grants (rather than denies) the access; `Principal: "*"` means *anyone* (the wildcard `*` = every user on the internet), which is what makes the site public; `Action: "s3:GetObject"` allows only *reading* objects — not listing or writing; and `Resource: "arn:aws:s3:::<bucket>/*"` scopes the rule to every object inside this bucket (the `arn:aws:s3:::` prefix is the bucket's globally unique **ARN**, or Amazon Resource Name).

### 🌐 Step 4: Configure Static Website Hosting

We enable the S3 website endpoint using `$INDEX_DOC` as the index document.

```bash
aws s3api put-bucket-website \
    --bucket $BUCKET_NAME \
    --website-configuration "{
        \"IndexDocument\": {
            \"Suffix\": \"${INDEX_DOC}\"
        }
    }"
```

> **Why:** A plain S3 bucket just stores files; to make it serve a **website** you must switch on static website hosting, which gives the bucket a special **website endpoint** URL (like `http://<bucket>.s3-website-<region>.amazonaws.com`) that returns web pages instead of listing files. `put-bucket-website` turns this on, and `--website-configuration` describes how it behaves. The `IndexDocument`/`Suffix` setting names the **index document** — the default page S3 returns when someone visits the site root or a subfolder with no specific file named (here `index.html`). That's why the same file also has to be uploaded in the next step.

### 📤 Step 5: Upload the index.html File

```bash
aws s3 cp $SOURCE_FILE s3://$BUCKET_NAME/$INDEX_DOC \
    --content-type "text/html"
```

```
upload: ./index.html to s3://nautilus-web-3074624944/index.html
```

> **Why:** `aws s3 cp` copies a file just like the Unix `cp` command, but between your machine and S3. The destination `s3://$BUCKET_NAME/$INDEX_DOC` uses the `s3://` URI scheme to point at a location *inside* a bucket — everything after the bucket name is the object's **key** (its path/name in S3). About `--content-type`: every file served over the web carries a **MIME type** — a label like `text/html` that tells the browser what the file is and how to handle it. If you don't set it, S3 may guess wrong and the browser could *download* your page as a file instead of *displaying* it. Setting `--content-type "text/html"` makes sure `index.html` renders correctly in the browser.

### ✅ Step 6: Verify

```bash
echo "Website URL: $WEBSITE_URL"
curl -o /dev/null -s -w "HTTP Status: %{http_code}\n" "$WEBSITE_URL"
```

```
Website URL: http://nautilus-web-3074624944.s3-website-us-east-1.amazonaws.com
HTTP Status: 200
```

> **Why:** `$WEBSITE_URL` is the S3 **website endpoint** we defined in Variables — the public HTTP address of the site, built from the bucket name and region. `curl` is a command-line tool that makes an HTTP request, just like a browser would, so we can test the site without opening one. Its flags: `-o /dev/null` throws away the page body (we don't need to see the HTML), `-s` runs silently (no progress bar), and `-w "HTTP Status: %{http_code}\n"` prints just the HTTP **status code** the server returned. A code of `200` means "OK" — the page was served successfully.

If you get `HTTP Status: 200`, the site is correctly published and accessible.

### 🔍 Validation Commands (Optional)

To confirm the configuration was applied correctly:

```bash
aws s3api get-bucket-website --bucket $BUCKET_NAME
aws s3api get-bucket-policy --bucket $BUCKET_NAME
aws s3api get-public-access-block --bucket $BUCKET_NAME
aws s3 ls s3://$BUCKET_NAME
```

> **Why:** These are read-only `get-*` commands that echo back the configuration we applied, so you can confirm each piece took effect: `get-bucket-website` shows the website hosting settings from Step 4, `get-bucket-policy` shows the public-read policy from Step 3, and `get-public-access-block` shows the four Block Public Access switches from Step 2 (all should read `false`). `aws s3 ls s3://$BUCKET_NAME` lists the objects in the bucket — like `ls` for a folder — confirming that `index.html` was actually uploaded.

## Best Practices

This challenge requires direct public access via the S3 website endpoint, so we disable Block Public Access and grant public `s3:GetObject`. Two AWS best practices are worth noting:

- **Least privilege in the bucket policy.** Grant **only `s3:GetObject`** — never `s3:ListBucket` or `s3:PutObject`. This prevents anonymous users from listing every object or uploading their own content. The policy in Step 3 already follows this.
- **Prefer CloudFront for production.** AWS explicitly recommends *against* direct public buckets. Instead, keep all four Block Public Access settings enabled and serve the site through **Amazon CloudFront with Origin Access Control (OAC)** (or **AWS Amplify Hosting**). The raw S3 website endpoint is **HTTP-only**; CloudFront adds HTTPS and keeps the bucket private.

### 📚 Official Documentation

- [Access control in Amazon S3](https://docs.aws.amazon.com/AmazonS3/latest/userguide/access-management.html)
- [Setting permissions for website access](https://docs.aws.amazon.com/AmazonS3/latest/userguide/WebsiteAccessPermissionsReqd.html)
- [Hosting a static website using Amazon S3](https://docs.aws.amazon.com/AmazonS3/latest/userguide/WebsiteHosting.html)
- [Blocking public access to your Amazon S3 storage](https://docs.aws.amazon.com/AmazonS3/latest/userguide/access-control-block-public-access.html)
- [Restricting access to an Amazon S3 origin (CloudFront OAC)](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/private-content-restricting-access-to-s3.html)
- [Amazon S3 security best practices](https://docs.aws.amazon.com/AmazonS3/latest/userguide/security-best-practices.html)
