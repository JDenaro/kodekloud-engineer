# Day 31: Configuring a Private RDS Instance for Application Development

The Nautilus Development Team is working on a new application feature that requires a reliable and scalable database solution. To facilitate development and testing, they need a new private RDS instance. This instance will be used to store critical application data and must be provisioned using the AWS free tier to minimize costs during the initial development phase. The team has chosen MySQL as the database engine due to its compatibility with their existing systems. The DevOps team has been tasked with setting up this RDS instance, ensuring that it is correctly configured and available for use by the development team.

As a member of the Nautilus DevOps Team, your task is to perform the following:

Provision a Private RDS Instance: Create a new private RDS instance named `xfusion-rds` using the Full configuration database creation method, and select the Free tier template. Further, it must be a `db.t3.micro` type instance.
Engine Configuration: Use the MySQL engine with version 8.4.x.
Enable Storage Autoscaling: Enable storage autoscaling and set the threshold value to 50GB. Keep the rest of the configurations as default.
Instance Availability: Ensure the instance is in the available state before submitting this task.

## Specific Requirements:

1. Provision a Private RDS Instance: Create a new private RDS instance named `xfusion-rds` using the Full configuration database creation method, and select the Free tier template. Further, it must be a `db.t3.micro` type instance.
2. Engine Configuration: Use the MySQL engine with version 8.4.x.
3. Enable Storage Autoscaling: Enable storage autoscaling and set the threshold value to 50GB. Keep the rest of the configurations as default.
4. Instance Availability: Ensure the instance is in the available state before submitting this task.

## Solution

**Amazon RDS** (Relational Database Service) is AWS's managed database offering: AWS runs the database engine, the underlying server, patching, and backups for you. Two details drive this challenge. First, "private" means the instance must **not be publicly accessible** — reachable only from inside the VPC, never from the internet. Second, every RDS instance needs a **DB subnet group** — a named collection of subnets spanning at least **two Availability Zones**, from which RDS picks where to place the instance; you cannot create an instance without one. The "Free tier template" maps to a `db.t3.micro` class, single-AZ, with `20 GiB` of storage, and **storage autoscaling** (which lets RDS grow the disk automatically) is enabled by setting a maximum threshold — here `50 GB`.

### 📦 Variables

```bash
AWS_REGION="us-east-1"
DB_IDENTIFIER="xfusion-rds"
DB_CLASS="db.t3.micro"
DB_ENGINE="mysql"
DB_SUBNET_GROUP="xfusion-rds-subnet-group"
DB_SG_NAME="xfusion-rds-sg"
MASTER_USER="admin"
MASTER_PASS="Xfusion12345!"
ALLOCATED_STORAGE="20"
MAX_ALLOCATED_STORAGE="50"
```

> **Why:** These are the values the challenge fixes (the instance name `xfusion-rds`, the `db.t3.micro` class, the MySQL engine, the `20 GiB` free-tier storage, and the `50 GB` autoscaling threshold) plus a few we choose: the DB subnet group name, the security group name, and the master credentials. `AWS_REGION` is pinned to `us-east-1` because the KodeKloud lab always runs there. The password just needs to satisfy RDS's minimum (8+ characters).

### 🔎 Step 1: Discover the default VPC and its subnets

```bash
VPC_ID=$(aws ec2 describe-vpcs --region "$AWS_REGION" \
  --filters "Name=is-default,Values=true" \
  --query "Vpcs[0].VpcId" --output text)

VPC_CIDR=$(aws ec2 describe-vpcs --region "$AWS_REGION" --vpc-ids "$VPC_ID" \
  --query "Vpcs[0].CidrBlock" --output text)

aws ec2 describe-subnets --region "$AWS_REGION" \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --query "Subnets[].SubnetId" --output text
```

Real values from the lab run:

```
VPC_ID=vpc-06812c24f13a4a614  VPC_CIDR=172.31.0.0/16
Subnets: subnet-03557171db830114f subnet-0ac81caa93c933cf7 subnet-03d28e6eb38790000 subnet-083267d171af52096 subnet-043e1d9cd2e9ab668 subnet-06ab61f535ce2c3c9
```

> **Why:** RDS needs a DB subnet group covering at least two Availability Zones, so we start from the account's **default VPC** (found with the `is-default` filter) and list its subnets. The default VPC ships with one subnet per AZ, so these six subnets already span multiple AZs — exactly what the subnet group needs. `describe-vpcs`/`describe-subnets` are read-only lookups; `--query` with `--output text` extracts the IDs. We also capture the VPC's **CIDR** (its IP range) so the security group can allow database traffic only from inside the VPC.

### 🧩 Step 2: Create the DB subnet group

```bash
aws rds create-db-subnet-group --region "$AWS_REGION" \
  --db-subnet-group-name "$DB_SUBNET_GROUP" \
  --db-subnet-group-description "Subnet group for $DB_IDENTIFIER" \
  --subnet-ids subnet-03557171db830114f subnet-0ac81caa93c933cf7 subnet-03d28e6eb38790000 subnet-083267d171af52096 subnet-043e1d9cd2e9ab668 subnet-06ab61f535ce2c3c9
```

> **Why:** A **DB subnet group** tells RDS which subnets (and therefore which Availability Zones) it may launch the database into — it's a mandatory prerequisite for any RDS instance in a VPC. `create-db-subnet-group` builds it: `--db-subnet-group-name` names it, `--db-subnet-group-description` is a required human-readable note, and `--subnet-ids` lists the subnets to include. Passing all of the default VPC's subnets guarantees coverage of at least two AZs, which RDS requires even for a single-AZ instance (so it can fail over or move the instance if needed).

### 🛡️ Step 3: Create a security group for the RDS instance

```bash
DB_SG_ID=$(aws ec2 create-security-group --region "$AWS_REGION" \
  --group-name "$DB_SG_NAME" \
  --description "Security group for $DB_IDENTIFIER" \
  --vpc-id "$VPC_ID" \
  --query "GroupId" --output text)

aws ec2 create-tags --region "$AWS_REGION" \
  --resources "$DB_SG_ID" --tags Key=Name,Value="$DB_SG_NAME"

aws ec2 authorize-security-group-ingress --region "$AWS_REGION" \
  --group-id "$DB_SG_ID" --protocol tcp --port 3306 --cidr "$VPC_CIDR"
```

Real value from the lab run: `DB_SG_ID=sg-023e22500cad82e3f`.

> **Why:** A **security group** is the virtual firewall attached to the RDS instance. `create-security-group` makes it inside our VPC. The `authorize-security-group-ingress` rule allows inbound TCP on port `3306` (the MySQL port) but **only from `172.31.0.0/16`**, the VPC's own CIDR — so the database is reachable from resources inside the VPC and never from the public internet, reinforcing the "private" requirement. `create-tags` gives the group a Name for easy identification.

### 🔢 Step 4: Resolve the latest MySQL 8.4.x engine version

```bash
aws rds describe-db-engine-versions --region "$AWS_REGION" \
  --engine "$DB_ENGINE" \
  --query "DBEngineVersions[?starts_with(EngineVersion,'8.4')].EngineVersion" \
  --output text
```

The newest matching version returned was `8.4.10`, so we use it:

```bash
ENGINE_VERSION="8.4.10"
```

> **Why:** The challenge asks for MySQL `8.4.x` without pinning an exact patch level, so instead of guessing we ask RDS which `8.4` versions it actually offers. `describe-db-engine-versions` lists the engine versions available for a given `--engine`; the JMESPath filter `[?starts_with(EngineVersion,'8.4')]` keeps only the `8.4` line, and we pick the newest (`8.4.10`). Using a version RDS truly supports avoids an `InvalidParameterValue` error at creation time.

### 🗄️ Step 5: Create the private RDS instance

```bash
aws rds create-db-instance --region "$AWS_REGION" \
  --db-instance-identifier "$DB_IDENTIFIER" \
  --db-instance-class "$DB_CLASS" \
  --engine "$DB_ENGINE" \
  --engine-version "$ENGINE_VERSION" \
  --master-username "$MASTER_USER" \
  --master-user-password "$MASTER_PASS" \
  --allocated-storage "$ALLOCATED_STORAGE" \
  --max-allocated-storage "$MAX_ALLOCATED_STORAGE" \
  --db-subnet-group-name "$DB_SUBNET_GROUP" \
  --vpc-security-group-ids "$DB_SG_ID" \
  --no-publicly-accessible \
  --no-multi-az \
  --backup-retention-period 0
```

The instance was accepted and entered the `creating` state:

```
------------------------------------------------------
|                  CreateDBInstance                  |
+--------------+---------+---------------+-----------+
|     Class    | Engine  |      Id       |  Status   |
+--------------+---------+---------------+-----------+
|  db.t3.micro |  8.4.10 |  xfusion-rds  |  creating |
+--------------+---------+---------------+-----------+
```

> **Why:** `create-db-instance` provisions the database. `--db-instance-identifier` sets the required name; `--db-instance-class db.t3.micro` picks the free-tier-eligible size; `--engine mysql` and `--engine-version 8.4.10` select the engine; `--master-username`/`--master-user-password` create the initial admin login. `--allocated-storage 20` is the free-tier starting disk size, and `--max-allocated-storage 50` **enables storage autoscaling** with a `50 GB` ceiling — the key requirement (RDS will grow the disk on its own up to that limit). `--db-subnet-group-name` and `--vpc-security-group-ids` place the instance in our subnet group behind our firewall. `--no-publicly-accessible` is what makes it **private** — no public IP, no internet reachability. `--no-multi-az` keeps it single-AZ (free tier), and `--backup-retention-period 0` disables automated backups to stay within the minimal free-tier setup. The call returns immediately with status `creating` while provisioning continues in the background.

### ⏳ Step 6: Wait for the instance to become available

```bash
aws rds wait db-instance-available --region "$AWS_REGION" \
  --db-instance-identifier "$DB_IDENTIFIER"
```

> **Why:** Creating an RDS instance takes several minutes while AWS provisions storage, launches the underlying host, and initializes MySQL. `wait db-instance-available` blocks and polls until the instance reports the `available` state, so we don't check for success too early. It's normal for this command to run for 5–10 minutes; that's not a failure.

### ✅ Step 7: Verify

```bash
aws rds describe-db-instances --region "$AWS_REGION" \
  --db-instance-identifier "$DB_IDENTIFIER" \
  --query "DBInstances[0].{Id:DBInstanceIdentifier,Status:DBInstanceStatus,Engine:Engine,Version:EngineVersion,Class:DBInstanceClass,Public:PubliclyAccessible,Allocated:AllocatedStorage,MaxAllocated:MaxAllocatedStorage,MultiAZ:MultiAZ,Endpoint:Endpoint.Address}" \
  --output table
```

Everything matches the requirements — `available`, MySQL `8.4.10`, `db.t3.micro`, private (`Public=False`), and storage autoscaling capped at `50`:

```
--------------------------------------------------------------------------
|                           DescribeDBInstances                          |
+--------------+---------------------------------------------------------+
|  Allocated   |  20                                                     |
|  Class       |  db.t3.micro                                            |
|  Endpoint    |  xfusion-rds.clgqykmu0a41.us-east-1.rds.amazonaws.com   |
|  Engine      |  mysql                                                  |
|  Id          |  xfusion-rds                                            |
|  MaxAllocated|  50                                                     |
|  MultiAZ     |  False                                                  |
|  Public      |  False                                                  |
|  Status      |  available                                              |
|  Version     |  8.4.10                                                 |
+--------------+---------------------------------------------------------+
```

> **Why:** `describe-db-instances` reads the instance's final configuration. The `--query` object projection surfaces exactly the fields the challenge cares about: `Status: available` (the instance is ready), `Version: 8.4.10` (MySQL 8.4.x), `Class: db.t3.micro`, `Public: False` (it's private), and `MaxAllocated: 50` (storage autoscaling threshold). Seeing all of them confirms the task is complete.

## Best Practices

- **Keep production databases private.** `--no-publicly-accessible` ensures the database has no public endpoint; reach it only from inside the VPC (or via a bastion/VPN), never straight from the internet.
- **Scope the security group tightly.** Allow port `3306` only from the VPC CIDR (or the specific application security group), not `0.0.0.0/0`.
- **Enable storage autoscaling.** Setting `--max-allocated-storage` lets RDS grow the disk automatically before it fills up, avoiding an outage — without over-provisioning storage you may never use.
- **Store credentials in a secret, not on the command line.** For real workloads, use AWS Secrets Manager (RDS can manage the master password for you) instead of passing `--master-user-password` in plaintext.
- **Right-size and use Multi-AZ for production.** `db.t3.micro` single-AZ is fine for free-tier development; production databases should use Multi-AZ for automatic failover and a class matched to the workload.

### 📚 Official Documentation

- [Creating an Amazon RDS DB instance](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_CreateDBInstance.html)
- [Working with a DB instance in a VPC](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_VPC.WorkingWithRDSInstanceinaVPC.html)
- [Managing capacity automatically with Amazon RDS storage autoscaling](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_PIOPS.StorageTypes.html#USER_PIOPS.Autoscaling)
- [create-db-instance — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/rds/create-db-instance.html)
- [create-db-subnet-group — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/rds/create-db-subnet-group.html)
