# Day 32: Snapshot and Restoration of an RDS Instance

The Nautilus Development Team is preparing for a major update to their database infrastructure. To ensure a smooth transition and to safeguard data, the team has requested the DevOps team to take a snapshot of the current RDS instance and restore it to a new instance. This process is crucial for testing and validation purposes before the update is rolled out to the production environment. The snapshot will serve as a backup, and the new instance will be used to verify that the backup process works correctly and that the application can function seamlessly with the restored data.

As a member of the Nautilus DevOps Team, your task is to perform the following:

Take a Snapshot: Take a snapshot of the `xfusion-rds` RDS instance and name it `xfusion-snapshot` (please wait `xfusion-rds` instance to be in available state).

Restore the Snapshot: Restore the snapshot to a new RDS instance named `xfusion-snapshot-restore`.

Configure the New RDS Instance: Ensure that the new RDS instance has a class of `db.t3.micro`.

Verify the New RDS Instance: The new RDS instance must be in the Available state upon completion of the restoration process.

## Specific Requirements:

1. Take a Snapshot: Take a snapshot of the `xfusion-rds` RDS instance and name it `xfusion-snapshot` (please wait `xfusion-rds` instance to be in available state).
2. Restore the Snapshot: Restore the snapshot to a new RDS instance named `xfusion-snapshot-restore`.
3. Configure the New RDS Instance: Ensure that the new RDS instance has a class of `db.t3.micro`.
4. Verify the New RDS Instance: The new RDS instance must be in the Available state upon completion of the restoration process.

## Solution

An RDS **snapshot** is a point-in-time backup of an entire DB instance — its data, engine, and configuration — stored by AWS until you delete it. Two ordering constraints matter here: you can only snapshot an instance that is **available** (not still `creating` or `modifying`), and you can only **restore** from a snapshot once the snapshot itself has finished and reports `available`. Restoring always creates a **brand-new instance** (you never restore "in place"); the new instance inherits the snapshot's engine and data, but you're free to change instance-level settings such as the class — which is exactly what the task asks (`db.t3.micro`).

### 📦 Variables

```bash
AWS_REGION="us-east-1"
SOURCE_DB="xfusion-rds"
SNAPSHOT_ID="xfusion-snapshot"
RESTORED_DB="xfusion-snapshot-restore"
DB_CLASS="db.t3.micro"
```

> **Why:** These come straight from the challenge: the existing source instance (`xfusion-rds`), the snapshot name (`xfusion-snapshot`), the new instance name (`xfusion-snapshot-restore`), and the required class (`db.t3.micro`). `AWS_REGION` is pinned to `us-east-1` because the KodeKloud lab always runs there.

### ⏳ Step 1: Wait for the source instance to be available

```bash
aws rds wait db-instance-available --region "$AWS_REGION" \
  --db-instance-identifier "$SOURCE_DB"
```

> **Why:** The challenge explicitly says to wait for `xfusion-rds` to be in the `available` state before snapshotting — RDS refuses to create a snapshot of an instance that is still initializing or modifying. `aws rds wait db-instance-available` blocks and polls until the instance reports `available`, so the snapshot command below can't run too early. If the instance was already available, this returns immediately.

### 🔎 Step 2: Capture the source instance's subnet group

```bash
DB_SUBNET_GROUP=$(aws rds describe-db-instances --region "$AWS_REGION" \
  --db-instance-identifier "$SOURCE_DB" \
  --query "DBInstances[0].DBSubnetGroup.DBSubnetGroupName" --output text)
```

Real value from the lab run: `DB_SUBNET_GROUP=xfusion-rds-subg`.

> **Why:** When we restore, we want the new instance to land in the same VPC network as the original rather than in some default network. `describe-db-instances` reads the source instance's configuration, and the `--query` pulls its **DB subnet group** name (`xfusion-rds-subg`) — the named set of subnets that determines which VPC and Availability Zones the database uses. We pass this to the restore command in Step 5 so the restored instance is placed consistently.

### 📸 Step 3: Take a snapshot of the source instance

```bash
aws rds create-db-snapshot --region "$AWS_REGION" \
  --db-snapshot-identifier "$SNAPSHOT_ID" \
  --db-instance-identifier "$SOURCE_DB"
```

The snapshot was accepted and entered the `creating` state:

```
-------------------------------------------------
|               CreateDBSnapshot                |
+------------------+---------------+------------+
|     Snapshot     |    Source     |  Status    |
+------------------+---------------+------------+
|  xfusion-snapshot|  xfusion-rds  |  creating  |
+------------------+---------------+------------+
```

> **Why:** `create-db-snapshot` triggers the backup. `--db-snapshot-identifier` gives the snapshot the required name `xfusion-snapshot`, and `--db-instance-identifier` says which instance to back up. This is a **manual snapshot**, meaning it persists until you explicitly delete it (unlike automated backups that expire on a retention schedule). The call returns immediately with status `creating` while AWS copies the data in the background.

### ⏳ Step 4: Wait for the snapshot to finish

```bash
aws rds wait db-snapshot-available --region "$AWS_REGION" \
  --db-snapshot-identifier "$SNAPSHOT_ID"
```

> **Why:** A snapshot can't be restored until it's fully written and reports `available`. `aws rds wait db-snapshot-available` blocks until the snapshot reaches that state. For a small free-tier database this completes in a couple of minutes; larger databases take longer, and the waiter simply polls until it's done.

### 🔄 Step 5: Restore the snapshot to a new instance

```bash
aws rds restore-db-instance-from-db-snapshot --region "$AWS_REGION" \
  --db-instance-identifier "$RESTORED_DB" \
  --db-snapshot-identifier "$SNAPSHOT_ID" \
  --db-instance-class "$DB_CLASS" \
  --db-subnet-group-name "$DB_SUBNET_GROUP" \
  --no-publicly-accessible \
  --no-multi-az
```

The restore was accepted and the new instance entered the `creating` state:

```
---------------------------------------------------------
|            RestoreDBInstanceFromDBSnapshot            |
+--------------+----------------------------+-----------+
|     Class    |            Id              |  Status   |
+--------------+----------------------------+-----------+
|  db.t3.micro |  xfusion-snapshot-restore  |  creating |
+--------------+----------------------------+-----------+
```

> **Why:** `restore-db-instance-from-db-snapshot` creates a **new** instance whose data is the snapshot's contents. `--db-instance-identifier` names it `xfusion-snapshot-restore`; `--db-snapshot-identifier` is the source backup. `--db-instance-class db.t3.micro` sets the required class — a restore is the moment you can change instance-level settings, since they aren't carried over from the snapshot. `--db-subnet-group-name` places it in the same network as the source, `--no-publicly-accessible` keeps it private, and `--no-multi-az` keeps it single-AZ (matching the free-tier source). The engine and version come from the snapshot automatically.

### ⏳ Step 6: Wait for the restored instance to become available

```bash
aws rds wait db-instance-available --region "$AWS_REGION" \
  --db-instance-identifier "$RESTORED_DB"
```

> **Why:** Restoring provisions a whole new instance — new storage, host, and engine startup — which takes several minutes. `aws rds wait db-instance-available` blocks until it reports `available`, satisfying the requirement that the new instance be in the Available state before the task is submitted. A multi-minute wait here is normal, not a failure.

### ✅ Step 7: Verify

```bash
aws rds describe-db-snapshots --region "$AWS_REGION" \
  --db-snapshot-identifier "$SNAPSHOT_ID" \
  --query "DBSnapshots[0].{Snapshot:DBSnapshotIdentifier,Source:DBInstanceIdentifier,Status:Status,Engine:Engine,Version:EngineVersion}" \
  --output table

aws rds describe-db-instances --region "$AWS_REGION" \
  --db-instance-identifier "$RESTORED_DB" \
  --query "DBInstances[0].{Id:DBInstanceIdentifier,Status:DBInstanceStatus,Class:DBInstanceClass,Engine:Engine,Version:EngineVersion,Public:PubliclyAccessible,Endpoint:Endpoint.Address}" \
  --output table
```

The snapshot is `available` and the restored instance is `available` with class `db.t3.micro`:

```
-----------------------------------------------------------------------
|                         DescribeDBSnapshots                         |
+--------+--------------------+--------------+------------+-----------+
| Engine |     Snapshot       |   Source     |  Status    |  Version  |
+--------+--------------------+--------------+------------+-----------+
|  mysql |  xfusion-snapshot  |  xfusion-rds |  available |  8.4.5    |
+--------+--------------------+--------------+------------+-----------+
-----------------------------------------------------------------------------------
|                               DescribeDBInstances                               |
+----------+----------------------------------------------------------------------+
|  Class   |  db.t3.micro                                                         |
|  Endpoint|  xfusion-snapshot-restore.ctwktvmb7a6i.us-east-1.rds.amazonaws.com   |
|  Engine  |  mysql                                                               |
|  Id      |  xfusion-snapshot-restore                                            |
|  Public  |  False                                                               |
|  Status  |  available                                                           |
|  Version |  8.4.5                                                               |
+----------+----------------------------------------------------------------------+
```

> **Why:** `describe-db-snapshots` confirms the backup completed (`Status: available`) and captured the MySQL `8.4.5` engine from the source. `describe-db-instances` confirms the restored instance is `available`, has the required `db.t3.micro` class, and carried over the engine and data — the `--query` object projections surface exactly those fields. Seeing both `available` means the snapshot-and-restore workflow succeeded end to end.

## Best Practices

- **Snapshot only an available instance.** RDS rejects snapshots of an instance that is still `creating` or `modifying`; wait for `available` first (as the challenge notes).
- **Restore is a new instance, never in place.** A restore always spins up a fresh instance from the snapshot's data, so plan for a new endpoint and update connection strings accordingly.
- **Tune settings at restore time.** Instance class, subnet group, public accessibility, and Multi-AZ aren't inherited from the snapshot — set them explicitly on the restore command, as we did for `db.t3.micro`.
- **Manual snapshots persist until deleted.** They don't expire like automated backups, so remember to clean them up to avoid ongoing storage charges once they're no longer needed.
- **Validate restores regularly.** Restoring to a test instance (exactly this workflow) is how you prove a backup is actually recoverable before you need it in an emergency.

### 📚 Official Documentation

- [Creating a DB snapshot for a Single-AZ DB instance](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_CreateSnapshot.html)
- [Restoring from a DB snapshot](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_RestoreFromSnapshot.html)
- [create-db-snapshot — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/rds/create-db-snapshot.html)
- [restore-db-instance-from-db-snapshot — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/rds/restore-db-instance-from-db-snapshot.html)
