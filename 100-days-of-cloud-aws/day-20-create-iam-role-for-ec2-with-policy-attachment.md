# Day 20: Create IAM Role for EC2 with Policy Attachment

When establishing infrastructure on the AWS cloud, Identity and Access Management (IAM) is among the first and most critical services to configure. IAM facilitates the creation and management of user accounts, groups, roles, policies, and other access controls. The Nautilus DevOps team is currently in the process of configuring these resources and has outlined the following requirements:

Create an IAM role as below:

1. IAM role name must be `iamrole_mark`.

2. Entity type must be AWS Service and use case must be `EC2`.

3. Attach a policy named `iampolicy_mark`.

## Specific Requirements:

1. IAM role name must be `iamrole_mark`.
2. Entity type must be AWS Service and use case must be `EC2`.
3. Attach a policy named `iampolicy_mark`.

## Solution

An IAM role has two separate permission concepts: its trust policy defines who can assume the role, while attached permissions policies define what the role can do. For this challenge, the trust policy names the EC2 service as the trusted entity, and the existing customer-managed policy `iampolicy_mark` is attached separately.

### 📦 Variables

```bash
AWS_REGION="us-east-1"
ROLE_NAME="iamrole_mark"
POLICY_NAME="iampolicy_mark"
```

### 🔎 Step 1: Check whether the IAM role exists

```bash
aws iam get-role \
  --role-name "$ROLE_NAME" \
  --query "Role.Arn" \
  --output text
```

The lab did not contain the role, so the script continued with creation:

```
Role does not exist and will be created.
```

> **Why:** `get-role` retrieves information about a named IAM role, including its ARN and trust policy. The required `--role-name` parameter identifies the role by its friendly name. The `--query` parameter extracts only the role ARN from the response, and `--output text` prints that value without JSON formatting. Checking first prevents the workflow from attempting to create a duplicate role.

### 🛡️ Step 2: Create the IAM role for EC2

```bash
aws iam create-role \
  --role-name "$ROLE_NAME" \
  --assume-role-policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"ec2.amazonaws.com"},"Action":"sts:AssumeRole"}]}' \
  --query "Role.Arn" \
  --output text
```

The command returned:

```
arn:aws:iam::449247691734:role/iamrole_mark
```

> **Why:** `create-role` creates the IAM role. `--role-name` assigns the required name, while `--assume-role-policy-document` supplies the trust policy that controls who may assume the role. The `Principal` identifies the EC2 service as the trusted AWS service, and `sts:AssumeRole` is the action EC2 uses to obtain temporary role credentials. The `--query` and `--output text` options return the new role's ARN, which is its unique AWS identifier.

### 🔎 Step 3: Find the policy ARN

```bash
POLICY_ARN=$(aws iam list-policies \
  --scope Local \
  --query "Policies[?PolicyName=='$POLICY_NAME'] | [0].Arn" \
  --output text)
```

The lab returned:

```
POLICY_ARN=arn:aws:iam::449247691734:policy/iampolicy_mark
```

> **Why:** `list-policies` lists managed policies available to the account. `--scope Local` limits the results to customer-managed policies, which is where the lab policy is located. The `--query` expression selects the policy whose `PolicyName` matches `iampolicy_mark` and extracts its ARN. The ARN is stored in `POLICY_ARN` because IAM attachment commands identify policies by ARN rather than by friendly name. `--output text` keeps the captured value in a simple shell-friendly format.

### 🔗 Step 4: Attach the policy to the role

```bash
aws iam attach-role-policy \
  --role-name "$ROLE_NAME" \
  --policy-arn "$POLICY_ARN"
```

The script confirmed the attachment:

```
Attached iampolicy_mark to iamrole_mark.
```

> **Why:** `attach-role-policy` attaches an existing managed policy to an IAM role, making that policy part of the role's permissions. `--role-name` identifies the target role, and `--policy-arn` identifies the exact managed policy to attach. AWS returns no response body when the attachment succeeds, so the script printed a confirmation message and then verified the result in the final step.

### ✅ Step 5: Verify

```bash
aws iam get-role \
  --role-name "$ROLE_NAME" \
  --query "{RoleName:Role.RoleName,RoleArn:Role.Arn,TrustedService:Role.AssumeRolePolicyDocument.Statement[0].Principal.Service}" \
  --output table

aws iam list-attached-role-policies \
  --role-name "$ROLE_NAME" \
  --query "AttachedPolicies[?PolicyName=='$POLICY_NAME'].{PolicyArn:PolicyArn,PolicyName:PolicyName}" \
  --output table
```

The verification returned:

```
-------------------------------------------------------------------
|                             GetRole                             |
+-----------------+-----------------------------------------------+
|  RoleArn        |  arn:aws:iam::449247691734:role/iamrole_mark  |
|  RoleName       |  iamrole_mark                                 |
|  TrustedService |  ec2.amazonaws.com                            |
+-----------------+-----------------------------------------------+
-------------------------------------------------------------------
|                      ListAttachedRolePolicies                       |
+--------------------------------------------------+------------------+
|                     PolicyArn                    |   PolicyName     |
+--------------------------------------------------+------------------+
|  arn:aws:iam::449247691734:policy/iampolicy_mark |  iampolicy_mark  |
+--------------------------------------------------+------------------+
```

> **Why:** `get-role` reads the role and confirms that its trust relationship names `ec2.amazonaws.com`. `list-attached-role-policies` lists the managed policies directly attached to the role. The `--query` expressions select only the role identity, trusted service, policy name, and policy ARN, while `--output table` formats the evidence for easy inspection. The role and policy satisfy all three challenge requirements.

## Best Practices

- **Separate trust from permissions.** The trust policy controls who can assume a role; the attached permissions policy controls what that role can do.
- **Use least privilege.** Review `iampolicy_mark` and grant only the actions and resources the EC2 workload actually needs.
- **Attach policies by ARN.** Friendly policy names are not sufficient to uniquely identify a policy across AWS-managed and customer-managed scopes.
- **Use an instance profile when attaching the role to EC2.** EC2 receives an IAM role through an instance profile; creating the instance profile was outside this challenge's requirements.
- **Check before creating.** Looking up `iamrole_mark` first avoids duplicate-resource errors and makes the workflow safe to rerun.

### 📚 Official Documentation

- [IAM roles](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles.html)
- [get-role — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/iam/get-role.html)
- [create-role — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/iam/create-role.html)
- [list-policies — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/iam/list-policies.html)
- [attach-role-policy — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/iam/attach-role-policy.html)
- [list-attached-role-policies — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/iam/list-attached-role-policies.html)
