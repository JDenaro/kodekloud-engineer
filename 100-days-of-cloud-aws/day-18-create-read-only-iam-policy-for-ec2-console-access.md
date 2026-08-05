# Day 18: Create Read-Only IAM Policy for EC2 Console Access

When establishing infrastructure on the AWS cloud, Identity and Access Management (IAM) is among the first and most critical services to configure. IAM facilitates the creation and management of user accounts, groups, roles, policies, and other access controls. The Nautilus DevOps team is currently in the process of configuring these resources and has outlined the following requirements.

## Specific Requirements:

Create an IAM policy named `iampolicy_anita` in `us-east-1` region, it must allow read-only access to the EC2 console, i.e this policy must allow users to view all instances, AMIs, and snapshots in the Amazon EC2 console.

## Solution

An IAM *policy* is a JSON document that defines a set of permissions; it grants no access on its own until attached to a user, group, or role. This task creates a **customer-managed policy** scoped to exactly the three view actions the EC2 console needs to render its Instances, AMIs, and Snapshots pages — narrower than the AWS-managed `AmazonEC2ReadOnlyAccess`, which also covers many other EC2 sub-resources (security groups, volumes, etc.) not requested here.

### 📦 Variables

```bash
AWS_REGION="us-east-1"
POLICY_NAME="iampolicy_anita"
```

### 📄 Step 1: Write the policy document

```bash
cat > /tmp/iampolicy_anita.json <<'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:DescribeInstances",
        "ec2:DescribeImages",
        "ec2:DescribeSnapshots"
      ],
      "Resource": "*"
    }
  ]
}
EOF
```

> **Why:** the policy grants exactly the three read-only actions the task lists: `ec2:DescribeInstances` (view instances), `ec2:DescribeImages` (view AMIs), and `ec2:DescribeSnapshots` (view snapshots). `Effect: Allow` grants rather than denies, and `Resource: "*"` is required for these actions since EC2 `Describe*` calls don't support resource-level restriction — they always operate account/region-wide.

### 🔧 Step 2: Create the IAM policy

```bash
POLICY_ARN=$(aws iam create-policy \
  --policy-name "$POLICY_NAME" \
  --policy-document file:///tmp/iampolicy_anita.json \
  --query "Policy.Arn" --output text)
```

Real value from the lab run: `POLICY_ARN=arn:aws:iam::767853987301:policy/iampolicy_anita`.

> **Why:** `create-policy` registers the JSON document above as a reusable, customer-managed policy named `iampolicy_anita`. `--policy-document file://...` reads the policy from the local file rather than inlining JSON on the command line, keeping the command readable. The returned `Arn` is what you'd later pass to `attach-user-policy` or `attach-group-policy` to actually grant this access to someone.

### ✅ Step 3: Verify

```bash
aws iam get-policy --policy-arn "$POLICY_ARN" \
  --query "Policy.{Name:PolicyName,Arn:Arn,Created:CreateDate}" \
  --output table
```

Success — the policy exists with the required name:

```
+---------+-----------------------------------------------------+
|  Arn    |  arn:aws:iam::767853987301:policy/iampolicy_anita   |
|  Created|  2026-07-20T22:59:48Z                               |
|  Name   |  iampolicy_anita                                    |
```

> **Why:** `get-policy` reads the policy's metadata back by ARN, confirming it was created with the required name. (It doesn't return the JSON document itself — that requires a separate `get-policy-version` call.)

## Best Practices

- **A policy grants nothing until attached.** Creating `iampolicy_anita` is only step one; use `attach-user-policy` or `attach-group-policy` (e.g. onto [`iamgroup_james`](day-17-create-iam-group.md)) to actually apply it.
- **Scope `Describe*` narrowly when possible.** Here we granted only the three actions requested instead of the broader AWS-managed read-only policy, following least privilege.
- **`Resource: "*"` is correct, not a red flag, for these actions.** EC2 `Describe*` API calls are inherently account-wide and don't support ARN-based resource restriction, unlike `ec2:TerminateInstances` which can be scoped to specific instance ARNs.
- **Version your policy documents.** IAM keeps up to 5 policy versions; use `create-policy-version` for future edits rather than deleting and recreating the policy.

### 📚 Official Documentation

- [Overview of IAM policies](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies.html)
- [Actions, resources, and condition keys for Amazon EC2](https://docs.aws.amazon.com/service-authorization/latest/reference/list_amazonec2.html)
- [create-policy — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/iam/create-policy.html)
