# Day 50: Expanding EC2 Instance Storage for Development Needs

The Nautilus DevOps Team has recently been informed by the Development Team that their EC2 instance is running out of storage space. This instance, crucial for development activities, is named `datacenter-ec2` and currently has an attached volume of 8 GiB. To accommodate the increasing data requirements, the storage needs to be expanded to 12 GiB. This change should ensure that the expanded space is immediately available for use within the instance without disrupting ongoing activities.

## Specific Requirements:

1. **Identify Volume:** Find the volume attached to the `datacenter-ec2` instance.
2. **Expand Volume:** Increase the volume size from 8 GiB to 12 GiB.
3. **Reflect Changes:** Ensure the root (`/`) partition within the instance reflects the expanded size from 8 GiB to 12 GiB.
4. **SSH Access:** Use the key pair located at `/root/datacenter-keypair.pem` on the `aws-client` host to SSH into the EC2 instance.

## Solution

Growing EBS storage that an instance actually *sees* is a **three-layer** operation, and forgetting any layer leaves the extra space invisible:

1. **Block device (EBS):** `modify-volume` grows the underlying disk — online, no reboot (this is the *Elastic Volumes* feature).
2. **Partition table:** the partition (`xvda1`) must be stretched to cover the new disk space with `growpart`.
3. **Filesystem:** the filesystem inside the partition must be grown to fill the partition — `xfs_growfs` for XFS (this instance) or `resize2fs` for ext4.

All three can be done live, so ongoing development activity is never interrupted.

### 📦 Variables

```bash
AWS_REGION="us-east-1"
INSTANCE_NAME="datacenter-ec2"
NEW_SIZE_GIB="12"
KEY_PATH="/root/datacenter-keypair.pem"
SSH_USER="ec2-user"
```

### 🔎 Step 1: Identify the instance and its root volume

```bash
aws ec2 describe-instances \
  --region "$AWS_REGION" \
  --filters "Name=tag:Name,Values=$INSTANCE_NAME" "Name=instance-state-name,Values=running" \
  --query "Reservations[0].Instances[0].{Id:InstanceId,PublicIp:PublicIpAddress,AZ:Placement.AvailabilityZone,Vol:BlockDeviceMappings[0].Ebs.VolumeId,Dev:RootDeviceName}" \
  --output table
```

Real values from the lab run:

```
AZ        us-east-1a
Dev       /dev/xvda
Id        i-0b97527ecfcdef790
PublicIp  18.206.251.206
Vol       vol-01364adcdb0266365
```

> **Why:** `describe-instances` is a read-only lookup; the `--filters "Name=tag:Name,..."` finds the instance by its *Name* tag and `instance-state-name=running` skips any terminated leftovers. A *volume* (EBS — Elastic Block Store) is the persistent network disk EC2 mounts as a block device. We pull the root `VolumeId` from `BlockDeviceMappings[].Ebs` because `modify-volume` acts on the volume, not the instance; the `PublicIpAddress` for SSH; and `RootDeviceName` (`/dev/xvda`) to know which disk to grow inside the OS.

### 📈 Step 2: Expand the volume to 12 GiB

```bash
aws ec2 modify-volume \
  --region "$AWS_REGION" \
  --volume-id vol-01364adcdb0266365 \
  --size 12
```

> **Why:** `modify-volume --size 12` tells EBS to grow the volume from 8 to 12 GiB **without detaching it** — the *Elastic Volumes* feature allows resizing a live, in-use volume. Volumes can only grow, never shrink. This enlarges the disk at the block level, but the OS does **not** see the extra space yet; the partition and filesystem still need growing. The response shows `ModificationState: modifying` with `TargetSize: 12`.

### ⏳ Step 3: Confirm the modification progressed

```bash
aws ec2 describe-volumes-modifications \
  --region "$AWS_REGION" \
  --volume-id vol-01364adcdb0266365 \
  --query "VolumesModifications[0].{State:ModificationState,Progress:Progress,Size:TargetSize}" \
  --output table
```

Expected once ready to proceed:

```
Progress  Size  State
0         12    optimizing
```

> **Why:** `describe-volumes-modifications` reports the resize progress, which moves through `modifying` → `optimizing` → `completed`. The new size is usable for growing the partition **as soon as the state reaches `optimizing`** — `completed` only means the background performance optimization finished, so there's no need to wait for it.

### 🔎 Step 4: Inspect the disk layout over SSH

```bash
ssh -i "$KEY_PATH" -o StrictHostKeyChecking=no "$SSH_USER"@18.206.251.206 \
  "lsblk && echo '---' && df -hT / && echo '---' && cat /etc/os-release | head -1"
```

Result — the disk is already 12G but the partition and filesystem still show 8G:

```
NAME      MAJ:MIN RM SIZE RO TYPE MOUNTPOINTS
xvda      202:0    0  12G  0 disk
├─xvda1   202:1    0   8G  0 part /
├─xvda127 259:0    0   1M  0 part
└─xvda128 259:1    0  10M  0 part /boot/efi
---
Filesystem     Type  Size  Used Avail Use% Mounted on
/dev/xvda1     xfs   8.0G  1.6G  6.5G  19% /
---
NAME="Amazon Linux"
```

> **Why:** we inspect before changing anything. `lsblk` lists disks and partitions — here `xvda` is now 12G but `xvda1` is still 8G, leaving 4G unallocated. `df -hT /` shows the size **and type** of the root filesystem; the type (`xfs` here, not `ext4`) decides which grow tool to use (`xfs_growfs` vs `resize2fs`). The default login user is AMI-specific: `ec2-user` on Amazon Linux (the `ubuntu` user fails here), which is why we connect as `ec2-user`.

### 🧩 Step 5: Grow the partition

```bash
ssh -i "$KEY_PATH" -o StrictHostKeyChecking=no "$SSH_USER"@18.206.251.206 \
  "sudo growpart /dev/xvda 1 && lsblk /dev/xvda"
```

Result — `xvda1` now spans the full 12G:

```
CHANGED: partition=1 start=24576 old: size=16752607 end=16777183 new: size=25141215 end=25165791
NAME      MAJ:MIN RM SIZE RO TYPE MOUNTPOINTS
xvda      202:0    0  12G  0 disk
├─xvda1   202:1    0  12G  0 part /
```

> **Why:** the disk (`xvda`) is 12G but partition `xvda1` was left at 8G. `growpart /dev/xvda 1` extends **partition 1** to consume the free space on the disk. Note the space between the device (`/dev/xvda`) and the partition number (`1`) — `growpart` takes them as two separate arguments. The filesystem inside the partition still needs to be told to use the new room.

### 🧬 Step 6: Grow the XFS filesystem

```bash
ssh -i "$KEY_PATH" -o StrictHostKeyChecking=no "$SSH_USER"@18.206.251.206 \
  "sudo xfs_growfs /"
```

```
data blocks changed from 2094075 to 3142651
```

> **Why:** `xfs_growfs` extends an **XFS** filesystem to fill its partition. Unlike ext4 (`resize2fs`), XFS can only grow and **must be mounted** while doing so — that's why the argument is the mount point `/`, not the device. The `data blocks changed` line confirms the filesystem now spans the enlarged partition.

### ✅ Step 7: Verify

```bash
ssh -i "$KEY_PATH" -o StrictHostKeyChecking=no "$SSH_USER"@18.206.251.206 \
  "df -hT /"
```

Success — the root filesystem is now 12G, live and without a reboot:

```
Filesystem     Type  Size  Used Avail Use% Mounted on
/dev/xvda1     xfs    12G  1.6G   11G  13% /
```

> **Why:** `df -hT /` reports the size, type, and usage of the filesystem mounted at `/`. Seeing `12G` confirms the root partition reflects the expanded size end to end — EBS volume, partition, and filesystem — meeting the requirement that the space be immediately available without disrupting the instance.

## Best Practices

- **Resize all three layers.** EBS `modify-volume` alone is invisible to the OS; you must also `growpart` the partition and grow the filesystem. Skipping a layer is the most common reason "the disk is still full."
- **`optimizing` is good enough.** Don't wait for `completed` — the extra capacity is usable once the modification reaches `optimizing`.
- **Match the tool to the filesystem.** `xfs_growfs` for XFS, `resize2fs` for ext4. Check with `df -hT` first.
- **Grow online, no reboot.** Elastic Volumes + `growpart` + `xfs_growfs` all work on a live root volume, so development activity is never interrupted.
- **Use the correct default user.** Amazon Linux uses `ec2-user`; guessing `ubuntu` just wastes an SSH attempt.

### 📚 Official Documentation

- [Request modifications to your EBS volumes](https://docs.aws.amazon.com/ebs/latest/userguide/requesting-ebs-volume-modifications.html)
- [Extend the file system after resizing an EBS volume](https://docs.aws.amazon.com/ebs/latest/userguide/recognize-expanded-volume-linux.html)
- [Monitor the progress of EBS volume modifications](https://docs.aws.amazon.com/ebs/latest/userguide/monitoring-volume-modifications.html)
