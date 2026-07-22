# Day 19: Attach IAM Policy to IAM User

The Nautilus DevOps team has been creating a couple of services on AWS cloud. They have been breaking down the migration into smaller tasks, allowing for better control, risk mitigation, and optimization of resources throughout the migration process. Recently they came up with requirements mentioned below.

## Specific Requirements:

An IAM user named `iamuser_john` and a policy named `iampolicy_john` already exist. Attach the IAM policy `iampolicy_john` to the IAM user `iamuser_john`.

## Solution

Creating a policy and creating a user are independent actions — neither grants anything by itself. *Attaching* a policy is the step that actually wires permissions to an identity. Both resources already exist here, so the task is purely to look up the policy's ARN and bind it directly to the user (as opposed to attaching it to a group the user belongs to).

### 📦 Variables

```bash
AWS_REGION="us-east-1"
USER_NAME="iamuser_john"
POLICY_NAME="iampolicy_john"
```

### 🔎 Step 1: Find the policy ARN

```bash
POLICY_ARN=$(aws iam list-policies --scope Local \
  --query "Policies[?PolicyName=='$POLICY_NAME'].Arn | [0]" --output text)
```

Real value from the lab run: `POLICY_ARN=arn:aws:iam::042988257746:policy/iampolicy_john`.

> **Why:** `list-policies` returns every policy visible to the account; `--scope Local` restricts the list to **customer-managed** policies (as opposed to `AWS`-managed ones), which is what a policy named `iampolicy_john` would be. The `--query` expression filters that list down to the one entry whose `PolicyName` matches and extracts its `Arn` — `attach-user-policy` requires the ARN, not the name.

### 🔗 Step 2: Attach the policy to the user

```bash
aws iam attach-user-policy \
  --user-name "$USER_NAME" \
  --policy-arn "$POLICY_ARN"
```

> **Why:** `attach-user-policy` binds a managed policy directly to an IAM user, granting that user every permission the policy defines. `--user-name` identifies the target user and `--policy-arn` identifies the policy by its unique ARN. The command produces no output on success.

### ✅ Step 3: Verify

```bash
aws iam list-attached-user-policies --user-name "$USER_NAME" \
  --query "AttachedPolicies[].{Name:PolicyName,Arn:PolicyArn}" \
  --output table
```

Success — the policy shows up in the user's attached list:

```
+--------------------------------------------------+------------------+
|                        Arn                       |      Name        |
+--------------------------------------------------+------------------+
|  arn:aws:iam::042988257746:policy/iampolicy_john |  iampolicy_john  |
+--------------------------------------------------+------------------+
```

> **Why:** `list-attached-user-policies` lists every managed policy directly attached to a user. Seeing `iampolicy_john` here confirms the attachment succeeded.

## Best Practices

- **Prefer attaching policies to groups over individual users.** Direct user attachments (as required here) work fine for one-off cases, but as the user base grows, managing permissions through group membership (like [`iamgroup_james`](day-17.md)) is easier to audit and keep consistent.
- **A user can have both direct and group-inherited policies.** Effective permissions are the union of both — worth checking `list-attached-user-policies` *and* group memberships when auditing what a user can actually do.
- **Detach, don't delete, when revoking access.** Use `detach-user-policy` to remove access without destroying the reusable policy itself.
- **Use the ARN, never the name, for attach/detach calls.** Policy names aren't unique across scopes (`Local` vs `AWS` managed), so the ARN is the only unambiguous identifier.

### 📚 Official Documentation

- [Adding and removing IAM identity permissions](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_manage-attach-detach.html)
- [attach-user-policy — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/iam/attach-user-policy.html)
- [list-attached-user-policies — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/iam/list-attached-user-policies.html)
