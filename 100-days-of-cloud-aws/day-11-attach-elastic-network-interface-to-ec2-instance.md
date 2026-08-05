# Day 11: Attach Elastic Network Interface to EC2 Instance

The Nautilus DevOps team has been creating a couple of services on AWS cloud. They have been breaking down the migration into smaller tasks, allowing for better control, risk mitigation, and optimization of resources throughout the migration process. Recently they came up with requirements mentioned below.

## Specific Requirements:

An instance named `xfusion-ec2` and an elastic network interface named `xfusion-eni` already exists in `us-east-1` region.

1. Attach the `xfusion-eni` network interface to the `xfusion-ec2` instance.
2. Make sure status is attached before submitting the task.
3. Please make sure instance initialisation has been completed before submitting this task.

## Solution

An *Elastic Network Interface* (ENI) is a virtual network card that can be detached from one instance and attached to another, independent of the instance lifecycle — useful for things like failover or dedicated management interfaces. Attaching a **secondary** ENI (the instance already has its primary one) requires a free *device index*, so before attaching we inspect the instance's existing interfaces to pick the next unused index. The task also explicitly requires waiting for instance initialization and for the attachment to reach `attached` before finishing.

### 📦 Variables

```bash
AWS_REGION="us-east-1"
INSTANCE_NAME="xfusion-ec2"
ENI_NAME="xfusion-eni"
```

### 🔎 Step 1: Find the instance

```bash
INSTANCE_ID=$(aws ec2 describe-instances --region "$AWS_REGION" \
  --filters "Name=tag:Name,Values=$INSTANCE_NAME" \
  --query "Reservations[0].Instances[0].InstanceId" --output text)
```

Real value from the lab run: `INSTANCE_ID=i-00517450dc6409441`.

> **Why:** we locate `xfusion-ec2` by its *Name* tag rather than hardcoding an instance ID.

### ⏳ Step 2: Wait for instance initialization

```bash
aws ec2 wait instance-status-ok --region "$AWS_REGION" --instance-ids "$INSTANCE_ID"
```

> **Why:** the task requires confirming instance initialization is complete before attaching the ENI. `wait instance-status-ok` blocks until **both** EC2 status checks (system and instance reachability) report `ok`, meaning the instance has fully booted.

### 🔎 Step 3: Find the ENI

```bash
ENI_ID=$(aws ec2 describe-network-interfaces --region "$AWS_REGION" \
  --filters "Name=tag:Name,Values=$ENI_NAME" \
  --query "NetworkInterfaces[0].NetworkInterfaceId" --output text)
```

Real value from the lab run: `ENI_ID=eni-087377c51428a417d`.

> **Why:** `describe-network-interfaces` filtered by the *Name* tag locates `xfusion-eni` and gives us the `NetworkInterfaceId` required for attaching.

### 🧮 Step 4: Determine a free device index

```bash
aws ec2 describe-instances --region "$AWS_REGION" --instance-ids "$INSTANCE_ID" \
  --query "Reservations[0].Instances[0].NetworkInterfaces[].Attachment.DeviceIndex" --output text
```

The only device index in use was `0` (the instance's primary ENI, created at launch), so the next free index is `1`:

```bash
DEVICE_INDEX=1
```

> **Why:** every network interface on an instance occupies a unique *device index* (`0` is always the primary ENI created at launch). Attaching a second ENI at an already-used index fails, so we check the current `Attachment.DeviceIndex` values and pick one past the highest in use.

### 🔗 Step 5: Attach the ENI

```bash
aws ec2 attach-network-interface --region "$AWS_REGION" \
  --network-interface-id "$ENI_ID" \
  --instance-id "$INSTANCE_ID" \
  --device-index "$DEVICE_INDEX"
```

> **Why:** `attach-network-interface` connects the ENI to the instance at the given `--device-index`. The call returns immediately with an `AttachmentId`, but the attachment itself transitions asynchronously through `attaching` before reaching `attached`.

### ⏳ Step 6: Wait for the attachment to reach `attached`

```bash
aws ec2 describe-network-interfaces --region "$AWS_REGION" --network-interface-ids "$ENI_ID" \
  --query "NetworkInterfaces[0].Attachment.Status" --output text
```

The attachment transitions from `attaching` to `attached` within seconds, so a single check already reported `attached`. If it had still shown `attaching`, the same `describe-network-interfaces` command would simply be re-run a few seconds later until it reports `attached`.

> **Why:** the task explicitly requires confirming `attached` status before finishing. `describe-network-interfaces` with the `Attachment.Status` query reports the current attachment state.

### ✅ Step 7: Verify

```bash
aws ec2 describe-network-interfaces --region "$AWS_REGION" --network-interface-ids "$ENI_ID" \
  --query "NetworkInterfaces[0].{Id:NetworkInterfaceId,Status:Status,Instance:Attachment.InstanceId,AttachStatus:Attachment.Status,DeviceIndex:Attachment.DeviceIndex}" \
  --output table
```

Success — the ENI is attached to the instance:

```
+---------------+-------------------------+
|  AttachStatus |  attached               |
|  DeviceIndex  |  1                      |
|  Id           |  eni-087377c51428a417d  |
|  Instance     |  i-00517450dc6409441    |
|  Status       |  in-use                 |
+---------------+-------------------------+
```

> **Why:** `describe-network-interfaces` reads the ENI back. `AttachStatus: attached` and `Status: in-use` (the ENI's own top-level status, which flips to `in-use` once bound to any instance) together confirm the attachment succeeded.

## Best Practices

- **Never assume device index 0 is free.** The primary ENI already occupies index 0; always inspect existing attachments before picking one for a new interface.
- **Poll for terminal state, don't assume immediacy.** `attach-network-interface` returns before the attachment finishes; always confirm `attached` before considering the task done, as required here.
- **Wait out instance initialization first.** Attaching network resources to a still-booting instance can interact poorly with the guest OS's network configuration at boot.
- **A secondary ENI needs OS-level configuration to be usable.** AWS attaches it at the hypervisor level; the guest OS may still need `dhclient`/netplan configuration to actually use the new interface — out of scope for this task but relevant in production.

### 📚 Official Documentation

- [Elastic network interfaces](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-eni.html)
- [attach-network-interface — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/ec2/attach-network-interface.html)
- [Status checks for your instances](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/monitoring-system-instance-status-check.html)
