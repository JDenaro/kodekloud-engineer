# Day 17: Create IAM Group

The Nautilus DevOps team has been creating a couple of services on AWS cloud. They have been breaking down the migration into smaller tasks, allowing for better control, risk mitigation, and optimization of resources throughout the migration process. Recently they came up with requirements mentioned below.

## Specific Requirements:

Create an IAM group named `iamgroup_james`.

## Solution

An *IAM group* is a named collection of IAM users that lets you attach policies once and have them apply to every member, instead of managing permissions per user. This task creates the empty group itself — no users or policies attached yet — as the foundation for that shared-permissions pattern.

### 📦 Variables

```bash
AWS_REGION="us-east-1"
GROUP_NAME="iamgroup_james"
```

### 👥 Step 1: Create the IAM group

```bash
aws iam create-group --group-name "$GROUP_NAME"
```

Real response from the lab run:

```json
{
    "Group": {
        "Path": "/",
        "GroupName": "iamgroup_james",
        "GroupId": "AGPAY524VEDZ7MP4AU7FA",
        "Arn": "arn:aws:iam::613837775091:group/iamgroup_james",
        "CreateDate": "2026-07-20T22:53:24Z"
    }
}
```

> **Why:** `create-group` registers a new IAM group named `iamgroup_james`. Like IAM users, groups are **global** (not region-scoped). The `Arn` returned is the identifier used later to attach policies (`attach-group-policy`) or add users to the group (`add-user-to-group`); `GroupId` is IAM's internal stable identifier, separate from the mutable name.

### ✅ Step 2: Verify

```bash
aws iam get-group --group-name "$GROUP_NAME" \
  --query "Group.{Name:GroupName,Arn:Arn,Created:CreateDate}" \
  --output table
```

Success — the group exists with the required name:

```
+---------+---------------------------------------------------+
|  Arn    |  arn:aws:iam::613837775091:group/iamgroup_james   |
|  Created|  2026-07-20T22:53:24Z                             |
|  Name   |  iamgroup_james                                   |
+---------+---------------------------------------------------+
```

> **Why:** `get-group` reads the group back by name (and would also list its members, empty here) confirming it was created.

## Best Practices

- **Groups can't be nested.** A group cannot contain another group — only users. Design flat permission tiers instead of hierarchies.
- **A group has no permissions until you attach a policy.** Creating the group is step one; use `attach-group-policy` to grant it access, then `add-user-to-group` to add members like [`iamuser_ravi`](day-16-create-iam-user.md).
- **Prefer groups over per-user policies.** Attaching policies to a group (rather than each user individually) keeps permissions consistent and easy to audit as team membership changes.
- **Name groups by role/function, not by a single person.** `iamgroup_james` works for this lab, but in practice a group usually represents a role (e.g. `developers`, `billing-admins`) shared by many users.

### 📚 Official Documentation

- [IAM user groups](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_groups.html)
- [create-group — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/iam/create-group.html)
- [Adding and removing users in an IAM user group](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_groups_manage_add-remove-users.html)
