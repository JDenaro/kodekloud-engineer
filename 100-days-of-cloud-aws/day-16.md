# Day 16: Create IAM User

When establishing infrastructure on the AWS cloud, Identity and Access Management (IAM) is among the first and most critical services to configure. IAM facilitates the creation and management of user accounts, groups, roles, policies, and other access controls. The Nautilus DevOps team is currently in the process of configuring these resources and has outlined the following requirements:

## Specific Requirements:

For this task, create an IAM user named `iamuser_ravi`.

## Solution

An *IAM user* is an identity representing a person or application that can authenticate to AWS and, once granted permissions via policies, act on resources. Creating a bare user is the first step in the identity lifecycle — this task creates the identity itself, with no permissions, console access, or credentials attached yet.

### 📦 Variables

```bash
AWS_REGION="us-east-1"
USER_NAME="iamuser_ravi"
```

### 👤 Step 1: Create the IAM user

```bash
aws iam create-user --user-name "$USER_NAME"
```

Real response from the lab run:

```json
{
    "User": {
        "Path": "/",
        "UserName": "iamuser_ravi",
        "UserId": "AIDA4QV37SXZ7YS2TJKZ4",
        "Arn": "arn:aws:iam::860461438451:user/iamuser_ravi",
        "CreateDate": "2026-07-20T22:48:14Z"
    }
}
```

> **Why:** `create-user` registers a new IAM identity named `iamuser_ravi`. IAM is a **global** service (not tied to a region), so no `--region` is needed for this call. The response's `Arn` is the unique identifier used to attach policies, add to groups, or reference the user in resource policies; `UserId` is IAM's internal stable ID for the user, distinct from the name (which can be changed later).

### ✅ Step 2: Verify

```bash
aws iam get-user --user-name "$USER_NAME" \
  --query "User.{Name:UserName,Arn:Arn,Created:CreateDate}" \
  --output table
```

Success — the user exists with the required name:

```
+---------+------------------------------------------------+
|  Arn    |  arn:aws:iam::860461438451:user/iamuser_ravi   |
|  Created|  2026-07-20T22:48:14Z                          |
|  Name   |  iamuser_ravi                                  |
+---------+------------------------------------------------+
```

> **Why:** `get-user` reads the user back by name, confirming it was created and giving its ARN and creation timestamp.

## Best Practices

- **Prefer IAM roles over long-lived users for workloads.** Users with static access keys are best reserved for humans or systems that truly need persistent credentials; EC2/Lambda workloads should assume roles instead (see [Day 08](day-08.md) for another EC2 instance-configuration example).
- **A bare user has no permissions.** `create-user` alone grants nothing — attach managed policies or add the user to a group before it can do anything.
- **Enforce MFA and strong password policies for console users.** Not part of this task, but essential before granting console login.
- **Use a consistent naming convention.** `iamuser_<name>` here makes users easy to distinguish from roles/groups when auditing IAM.

### 📚 Official Documentation

- [IAM users](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users.html)
- [create-user — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/iam/create-user.html)
- [Security best practices in IAM](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html)
