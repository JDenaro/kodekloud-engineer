# Task 25: Change Instance Type Using Terraform

The Nautilus DevOps team found that the EC2 instance named `devops-ec2` was underutilized. This task changes its type from `t2.micro` to `t2.nano` and checks that it is running afterward.

## Task Requirements

1. Wait for the instance status check to finish initializing before changing its type.
2. Change the `devops-ec2` instance type from `t2.micro` to `t2.nano` in `/home/bob/terraform/main.tf`.
3. Confirm the instance is in the `running` state after the change.

## Solution

The lab starts with an existing `aws_instance` resource in `main.tf`. Update only its `instance_type` value, leaving its AMI, subnet, security group, and tags intact. Since this lab has no AWS console access, use AWS CLI to inspect the instance.

### 🔎 Step 1: Check the instance before editing

Use AWS CLI to find the instance by its `Name` tag and inspect its ID, state, and current type:

```bash
aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=devops-ec2" \
  --query "Reservations[].Instances[].{ID:InstanceId,State:State.Name,Type:InstanceType}" \
  --output table
```

Use the returned instance ID to check the system and instance status checks:

```bash
aws ec2 describe-instance-status \
  --instance-ids <INSTANCE_ID> \
  --query "InstanceStatuses[].{State:InstanceState.Name,System:SystemStatus.Status,Instance:InstanceStatus.Status}" \
  --output table
```

If either check is `initializing`, wait and repeat the status command before making the change.

> **Why:** The first command locates the instance by its lab-provided tag and returns its ID for the status query. `describe-instance-status` reports the separate system and instance checks; waiting for initialization to finish avoids changing the instance before its checks are ready.

The first status query used during the lab attempted to read `StatusChecks` from `describe-instances`; AWS CLI returned `null` for that field and `join()` failed. The separate `describe-instance-status` command provides the status checks. The successful `describe-instances` query showed the instance in `running` state.

### 📝 Step 2: Update the Terraform configuration

In VS Code, open the lab's `/home/bob/terraform/main.tf`. Keep the existing configuration and change only the instance type in the `aws_instance "ec2"` block:

```hcl
resource "aws_instance" "ec2" {
  ami           = "ami-0c101f26f147fa7fd"
  instance_type = "t2.nano"
  subnet_id     = ""
  vpc_security_group_ids = [
    "sg-22b63230dcbf7a91e"
  ]

  tags = {
    Name = "devops-ec2"
  }
}
```

> **Why:** The existing resource is already associated with the target instance. Changing only `instance_type` lets Terraform update that instance in place while preserving its other configured attributes.

### 🚀 Step 3: Plan and apply the change

From `/home/bob/terraform`, review the proposed change:

```bash
terraform plan
```

If the plan shows an in-place update from `t2.micro` to `t2.nano`, apply it:

```bash
terraform apply
```

> **Why:** The plan previews Terraform's action before it modifies the instance. The observed plan showed `update in-place`, `t2.micro -> t2.nano`, and `0 to add, 1 to change, 0 to destroy`. Apply completed with `0 added, 1 changed, 0 destroyed` after 20 seconds.

### ✅ Step 4: Verify the instance state and type

Run the instance query again:

```bash
aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=devops-ec2" \
  --query "Reservations[].Instances[].{ID:InstanceId,State:State.Name,Type:InstanceType}" \
  --output table
```

> **Why:** This confirms the target instance is still `running` and now uses `t2.nano`, as required.

Observed output showed instance `i-1642437abef699def` with state `running` and type `t2.nano`. The user confirmed that KodeKloud marked the lab successful.

## Best Practices

- Inspect an instance's status checks before changing its type, and wait while checks are initializing.
- Review Terraform's plan to confirm the intended in-place update before applying it.
- Use `describe-instance-status` for EC2 status checks; `describe-instances` reports instance state and configuration but does not include a `StatusChecks` array.

### 📚 Official Documentation

- [Terraform AWS instance resource](https://registry.terraform.io/providers/hashicorp/aws/5.91.0/docs/resources/instance)
- [AWS CLI `describe-instances`](https://docs.aws.amazon.com/cli/latest/reference/ec2/describe-instances.html)
- [AWS CLI `describe-instance-status`](https://docs.aws.amazon.com/cli/latest/reference/ec2/describe-instance-status.html)
- [Change the instance type for an EC2 instance](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/change-instance-type-of-ebs-backed-instance.html)
