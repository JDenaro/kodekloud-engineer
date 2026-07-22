# Day 37: Managing EC2 Access with S3 Role-based Permissions

The Nautilus DevOps team needs to set up an application on an EC2 instance to interact with an S3 bucket for storing and retrieving data. To achieve this, the team must create a private S3 bucket, set appropriate IAM policies and roles, and test the application functionality.

Task:
1) EC2 Instance Setup:

An instance named `xfusion-ec2` already exists.
The instance requires access to an S3 bucket.
2) Setup SSH Keys:

Create new SSH key pair (`id_rsa` and `id_rsa.pub`) on the aws-client host and add the public key to the root user's authorized keys on the EC2 instance.
3) Create a Private S3 Bucket:

Name the bucket `xfusion-s3-573434484730`.
Ensure the bucket is private.
4) Create an IAM Policy and Role:

Create an IAM policy allowing `s3:PutObject`, `s3:ListBucket` and `s3:GetObject` access to `xfusion-s3-573434484730`.
Create an IAM role named `xfusion-role`.
Attach the policy to the IAM role.
Attach this role to the `xfusion-ec2` instance.
5) Test the Access:

SSH into the EC2 instance and try to upload a file to `xfusion-s3-573434484730` bucket using following command:
`aws s3 cp <your-file> s3://xfusion-s3-573434484730/`

Now run following command to list the upload file:
`aws s3 ls s3://xfusion-s3-573434484730/`

## Specific Requirements:

1. An instance named `xfusion-ec2` already exists and requires access to an S3 bucket.
2. Create a new SSH key pair (`id_rsa`/`id_rsa.pub`) on the aws-client host and add the public key to the root user's authorized keys on the EC2 instance.
3. Create a private S3 bucket named `xfusion-s3-573434484730`.
4. Create an IAM policy allowing `s3:PutObject`, `s3:ListBucket`, and `s3:GetObject` on `xfusion-s3-573434484730`; create an IAM role `xfusion-role`; attach the policy to the role; attach the role to `xfusion-ec2`.
5. SSH into the instance and confirm you can `aws s3 cp` a file to the bucket and `aws s3 ls` it.

## Solution

The goal is to let an application on `xfusion-ec2` read and write an S3 bucket **without storing any AWS access keys on the instance**. The mechanism is an **IAM role attached to the instance** (via an *instance profile*): the EC2 metadata service hands the instance short-lived, auto-rotating credentials, and the AWS CLI on the box picks them up automatically. So there are two independent trust paths — an **SSH key** (so *we* can log into the instance) and the **IAM role** (so the *instance* can call S3).

Two environment notes carry over from earlier days: the instance is **Ubuntu 22.04** (login user `ubuntu`, and the AWS CLI is already installed), and the KodeKloud sandbox denies `iam:PutRolePolicy`, so we grant S3 access with a **customer-managed policy** (`create-policy` + `attach-role-policy`) rather than an inline one.

### 📦 Variables

```bash
AWS_REGION="us-east-1"
EC2_NAME="xfusion-ec2"
BUCKET="xfusion-s3-573434484730"
POLICY_NAME="xfusion-s3-policy"
ROLE_NAME="xfusion-role"
KEY_PATH="/root/.ssh/id_rsa"
```

> **Why:** The bucket name, role name, and EC2 name come straight from the challenge; `xfusion-s3-policy` is the name we choose for the customer-managed policy. `KEY_PATH` is the SSH key the challenge dictates. `AWS_REGION` is pinned to `us-east-1`. IDs (instance, ARNs) are discovered in the steps.

### 🔎 Step 1: Discover the EC2 instance and its login user

```bash
EC2_ID=$(aws ec2 describe-instances --region "$AWS_REGION" \
  --filters "Name=tag:Name,Values=$EC2_NAME" "Name=instance-state-name,Values=running" \
  --query "Reservations[0].Instances[0].InstanceId" --output text)

EC2_AZ=$(aws ec2 describe-instances --region "$AWS_REGION" --instance-ids "$EC2_ID" \
  --query "Reservations[0].Instances[0].Placement.AvailabilityZone" --output text)

EC2_PUB_IP=$(aws ec2 describe-instances --region "$AWS_REGION" --instance-ids "$EC2_ID" \
  --query "Reservations[0].Instances[0].PublicIpAddress" --output text)

AMI_ID=$(aws ec2 describe-instances --region "$AWS_REGION" --instance-ids "$EC2_ID" \
  --query "Reservations[0].Instances[0].ImageId" --output text)

aws ec2 describe-images --region "$AWS_REGION" --image-ids "$AMI_ID" \
  --query "Images[0].Name" --output text
```

Real values from the lab run:

```
EC2_ID=i-0be77c4416d067ad5  AZ=us-east-1a  PUB_IP=13.223.94.106
AMI=ami-0a0e5d9c7acc336f1  (ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-...)
```

> **Why:** We look up the existing instance by its *Name* tag (never recreate it). We capture its **Availability Zone** (required by EC2 Instance Connect), its **public IP** (to SSH in), and its **AMI name** — which reveals Ubuntu 22.04, so the login user is `ubuntu`. `describe-instances`/`describe-images` are read-only lookups.

### 🔑 Step 2: Create the SSH key and add it to the instance's root user

```bash
ssh-keygen -t rsa -b 2048 -f "$KEY_PATH" -N ""
```

The key pair didn't exist, so it was generated. Because the instance has no key pair we own, we push our public key with **EC2 Instance Connect** for a one-time login as `ubuntu`, then install it for `root`:

```bash
aws ec2-instance-connect send-ssh-public-key --region "$AWS_REGION" \
  --instance-id "$EC2_ID" --availability-zone "$EC2_AZ" \
  --instance-os-user ubuntu --ssh-public-key "file://$KEY_PATH.pub"
```

```bash
PUBKEY=$(cat "$KEY_PATH.pub")

ssh -i "$KEY_PATH" -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null \
  ubuntu@"$EC2_PUB_IP" "sudo bash -s" <<EOF
install -d -m 700 /root/.ssh
echo "$PUBKEY" > /root/.ssh/authorized_keys
chmod 600 /root/.ssh/authorized_keys
echo "PermitRootLogin prohibit-password" > /etc/ssh/sshd_config.d/00-permitroot.conf
systemctl reload ssh
EOF
```

> **Why:** `ssh-keygen -t rsa -b 2048 -f ... -N ""` creates a passphrase-less RSA key pair at `/root/.ssh/id_rsa`. `ec2-instance-connect send-ssh-public-key` injects our public key into the instance's metadata for 60 seconds so we can SSH in as `ubuntu` without owning its original key (`--instance-os-user ubuntu` is essential on this Ubuntu AMI). In that session we write our public key into `/root/.ssh/authorized_keys` (overwriting Ubuntu's default banner entry) and add a `PermitRootLogin prohibit-password` drop-in so `sshd` accepts key-based root logins; `systemctl reload ssh` applies it (the service is `ssh` on Ubuntu). Now password-less `root` SSH works — satisfying task 2. The AWS CLI was already installed on this instance, so no install was needed.

### 🪣 Step 3: Create the private S3 bucket

```bash
aws s3api create-bucket --bucket "$BUCKET"

aws s3api put-public-access-block --bucket "$BUCKET" \
  --public-access-block-configuration BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true
```

> **Why:** `s3api create-bucket` creates the bucket; in `us-east-1` no `LocationConstraint` is needed (it's the default region). `put-public-access-block` turns on all four **Block Public Access** switches, guaranteeing the bucket is **private** regardless of any ACL or policy — the "ensure the bucket is private" requirement. (New buckets already block public access by default; setting it explicitly makes the guarantee unambiguous.)

### 📜 Step 4: Create the IAM policy

```bash
cat > policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    { "Effect": "Allow", "Action": ["s3:PutObject","s3:GetObject"], "Resource": "arn:aws:s3:::$BUCKET/*" },
    { "Effect": "Allow", "Action": ["s3:ListBucket"], "Resource": "arn:aws:s3:::$BUCKET" }
  ]
}
EOF

aws iam create-policy --policy-name "$POLICY_NAME" \
  --policy-document file://policy.json \
  --query "Policy.Arn" --output text
```

Real value from the lab run: `arn:aws:iam::573434484730:policy/xfusion-s3-policy`.

> **Why:** The policy grants exactly the three actions the challenge lists, split by **resource level**: `s3:PutObject`/`s3:GetObject` act on **objects**, so their resource is `arn:aws:s3:::<bucket>/*` (everything *inside* the bucket); `s3:ListBucket` acts on the **bucket itself**, so its resource is `arn:aws:s3:::<bucket>` (no `/*`). Getting these ARNs right is the usual S3 IAM gotcha. `create-policy` registers it as a **customer-managed policy** and returns its ARN, which we attach to the role next.

### 🎭 Step 5: Create the role, attach the policy, and attach the role to the instance

```bash
cat > trust.json <<'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    { "Effect": "Allow", "Principal": { "Service": "ec2.amazonaws.com" }, "Action": "sts:AssumeRole" }
  ]
}
EOF

aws iam create-role --role-name "$ROLE_NAME" \
  --assume-role-policy-document file://trust.json \
  --query "Role.Arn" --output text

aws iam attach-role-policy --role-name "$ROLE_NAME" --policy-arn "$POLICY_ARN"

aws iam create-instance-profile --instance-profile-name "$ROLE_NAME"
aws iam add-role-to-instance-profile --instance-profile-name "$ROLE_NAME" --role-name "$ROLE_NAME"

aws ec2 associate-iam-instance-profile --region "$AWS_REGION" \
  --instance-id "$EC2_ID" --iam-instance-profile "Name=$ROLE_NAME"
```

Real values from the lab run: role `arn:aws:iam::573434484730:role/xfusion-role`; association `iip-assoc-0926578e256ac1546`.

> **Why:** `create-role` makes `xfusion-role` with a **trust policy** allowing the **EC2 service** (`ec2.amazonaws.com`) to `sts:AssumeRole` — that's what lets an instance take on the role. `attach-role-policy` links our S3 policy to it. EC2 can't attach a role directly; it uses an **instance profile** wrapper, so `create-instance-profile` + `add-role-to-instance-profile` put the role inside a profile of the same name. Finally `associate-iam-instance-profile` attaches that profile to the running instance (`describe-iam-instance-profile-associations` first confirmed the instance had no profile yet, so we associate rather than replace). Within moments the instance's metadata starts serving temporary credentials for the role.

### 🧪 Step 6: Test S3 access from the instance

```bash
ssh -i "$KEY_PATH" -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null \
  root@"$EC2_PUB_IP" "bash -s" <<EOF
export AWS_DEFAULT_REGION=$AWS_REGION
echo 'hello from xfusion-ec2' > /tmp/xfusion-test.txt
aws s3 cp /tmp/xfusion-test.txt s3://$BUCKET/
aws s3 ls s3://$BUCKET/
EOF
```

The upload and listing succeed using the role's credentials:

```
upload: ../tmp/xfusion-test.txt to s3://xfusion-s3-573434484730/xfusion-test.txt
2026-07-22 02:54:43         23 xfusion-test.txt
```

> **Why:** We SSH into the instance as `root` (using the key installed in Step 2) and run the exact commands the challenge specifies. Crucially, no credentials are configured on the box — the AWS CLI transparently fetches the `xfusion-role` credentials from the instance metadata service, and the S3 policy lets it `PutObject` (the `cp` upload) and `ListBucket` (the `ls`). If the very first attempt fails with `AccessDenied`, it's because the profile association hadn't propagated yet — re-running the same command a few seconds later succeeds.

### ✅ Step 7: Verify

```bash
aws s3 ls "s3://$BUCKET/"

aws s3api get-public-access-block --bucket "$BUCKET" \
  --query "PublicAccessBlockConfiguration" --output table

aws ec2 describe-instances --region "$AWS_REGION" --instance-ids "$EC2_ID" \
  --query "Reservations[0].Instances[0].IamInstanceProfile.Arn" --output text
```

The uploaded object is present, the bucket blocks all public access, and the role is attached to the instance:

```
2026-07-22 02:54:43         23 xfusion-test.txt

-----------------------------------
|      GetPublicAccessBlock       |
+-------------------------+-------+
|  BlockPublicAcls        |  True |
|  BlockPublicPolicy      |  True |
|  IgnorePublicAcls       |  True |
|  RestrictPublicBuckets  |  True |
+-------------------------+-------+

arn:aws:iam::573434484730:instance-profile/xfusion-role
```

> **Why:** `aws s3 ls` from the aws-client side confirms the object the instance uploaded really landed in the bucket. `get-public-access-block` shows all four switches `True`, proving the bucket is private. `describe-instances` returns the instance's `IamInstanceProfile.Arn`, confirming `xfusion-role` is attached — the instance now has S3 access purely through its role.

## Best Practices

- **Use instance roles, never static keys.** Attaching `xfusion-role` gives the instance auto-rotating temporary credentials; no access keys ever touch the disk or risk being leaked.
- **Least-privilege the policy.** Grant only the actions needed (`PutObject`, `GetObject`, `ListBucket`) on the single bucket, with object vs. bucket ARNs split correctly — not blanket `s3:*` on `*`.
- **Keep the bucket private.** Full Block Public Access ensures the data is never exposed even if an ACL or policy is misconfigured later.
- **Prefer managed policies in restricted environments.** The sandbox blocks inline policies (`iam:PutRolePolicy`); a customer-managed policy attaches cleanly and is reusable across roles.
- **Bootstrap access with EC2 Instance Connect.** When you don't hold an instance's key, Instance Connect grants short-lived access to install your own key rather than sharing long-lived credentials.

### 📚 Official Documentation

- [IAM roles for Amazon EC2](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/iam-roles-for-amazon-ec2.html)
- [Using an IAM role to grant permissions to applications running on Amazon EC2 instances](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_use_switch-role-ec2.html)
- [Blocking public access to your Amazon S3 storage](https://docs.aws.amazon.com/AmazonS3/latest/userguide/access-control-block-public-access.html)
- [associate-iam-instance-profile — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/ec2/associate-iam-instance-profile.html)
- [create-policy — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/iam/create-policy.html)
