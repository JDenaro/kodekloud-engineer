# Day 07: Change EC2 Instance Type

During the migration process, the Nautilus DevOps team created several EC2 instances in different regions. They are currently in the process of identifying the correct resources and utilization and are making continuous changes to ensure optimal resource utilization. Recently, they discovered that one of the EC2 instances was underutilized, prompting them to decide to change the instance type. Please make sure the Status check is completed (if its still in Initializing state) before making any changes to the instance.

## Specific Requirements:

1. Change the instance type from `t2.micro` to `t2.nano` for `devops-ec2` instance.
2. Make sure the ec2 instance `devops-ec2` is in running state after the change.

## Solution

The instance type is the **virtual hardware** (vCPU/RAM) backing an instance, and it can only be changed while the instance is **stopped** — the hypervisor has to move the instance to a host that offers the new type. So this is a stop → modify → start cycle. The task also explicitly asks to wait until the **status checks** pass (leave the `Initializing` state) before touching it, so we gate on that first.

### 📦 Variables

```bash
AWS_REGION="us-east-1"
INSTANCE_NAME="devops-ec2"
NEW_TYPE="t2.nano"
```

### 🔎 Step 1: Find the instance

```bash
INSTANCE_ID=$(aws ec2 describe-instances --region "$AWS_REGION" \
  --filters "Name=tag:Name,Values=$INSTANCE_NAME" "Name=instance-state-name,Values=running,pending" \
  --query "Reservations[0].Instances[0].InstanceId" --output text)
```

Real values from the lab run: `INSTANCE_ID=i-0aae0d81d46f9acb3`, current type `t2.micro`.

> **Why:** we locate `devops-ec2` by its *Name* tag. We include both `running` and `pending` states so a freshly booted instance still matches while it initializes.

### ⏳ Step 2: Wait for status checks to complete

```bash
aws ec2 wait instance-status-ok --region "$AWS_REGION" --instance-ids "$INSTANCE_ID"
```

> **Why:** the task requires waiting out the `Initializing` status. `wait instance-status-ok` blocks until **both** EC2 status checks (system and instance) report `ok` — i.e. the instance has fully booted and the host is healthy. Changing the type before this could interrupt initialization.

### ⏹️ Step 3: Stop the instance

```bash
aws ec2 stop-instances --region "$AWS_REGION" --instance-ids "$INSTANCE_ID"

aws ec2 wait instance-stopped --region "$AWS_REGION" --instance-ids "$INSTANCE_ID"
```

> **Why:** the instance type cannot be changed on a running instance, so `stop-instances` powers it down gracefully. `wait instance-stopped` blocks until it reaches the `stopped` state, which is required before the modify call will succeed.

### 🔧 Step 4: Change the instance type

```bash
aws ec2 modify-instance-attribute --region "$AWS_REGION" --instance-id "$INSTANCE_ID" \
  --instance-type "{\"Value\":\"t2.nano\"}"
```

> **Why:** `modify-instance-attribute` edits a single attribute of a stopped instance. The `--instance-type` attribute takes a small JSON object `{"Value":"t2.nano"}` (the attribute-value shape the API expects). This is the actual change from `t2.micro` to `t2.nano`.

### ▶️ Step 5: Start the instance again

```bash
aws ec2 start-instances --region "$AWS_REGION" --instance-ids "$INSTANCE_ID"

aws ec2 wait instance-running --region "$AWS_REGION" --instance-ids "$INSTANCE_ID"
```

> **Why:** `start-instances` boots the instance back up on the new hardware type, and `wait instance-running` blocks until it returns to `running` — satisfying the requirement that `devops-ec2` end up running.

### ✅ Step 6: Verify

```bash
aws ec2 describe-instances --region "$AWS_REGION" --instance-ids "$INSTANCE_ID" \
  --query "Reservations[0].Instances[0].{Id:InstanceId,Type:InstanceType,State:State.Name,Name:Tags[?Key=='Name']|[0].Value}" \
  --output table
```

Success — type is now `t2.nano` and the instance is `running`:

```
+----------------------+-------------+----------+-----------+
|          Id          |    Name     |  State   |   Type    |
+----------------------+-------------+----------+-----------+
|  i-0aae0d81d46f9acb3 |  devops-ec2 |  running |  t2.nano  |
+----------------------+-------------+----------+-----------+
```

> **Why:** `describe-instances` confirms both requirements in one read: `InstanceType` is `t2.nano` and `State` is `running`.

## Best Practices

- **Type changes require a stop.** Plan for the brief downtime; there is no in-place resize for the instance type.
- **Wait for status checks before and running after.** Gate the change on `instance-status-ok`, and confirm `instance-running` afterward so you don't leave it stopped.
- **The public IP may change.** Stopping/starting a non-Elastic-IP instance assigns a new public IPv4 on start; update anything that referenced the old address.
- **Keep the same family when possible.** Staying within `t2.*` keeps the same virtualization/AMI compatibility, avoiding surprises from ENA/driver differences across families.

### 📚 Official Documentation

- [Change the instance type](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-resize.html)
- [Status checks for your instances](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/monitoring-system-instance-status-check.html)
- [modify-instance-attribute — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/ec2/modify-instance-attribute.html)
