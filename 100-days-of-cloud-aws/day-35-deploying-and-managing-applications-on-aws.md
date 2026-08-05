# Day 35: Deploying and Managing Applications on AWS

The Nautilus DevOps team needs a new private RDS instance for their application. They need to set up a MySQL database and ensure that their existing EC2 instance can connect to it. This will help in managing their database needs efficiently and securely.

1) Task Details:

Create a private RDS instance named `datacenter-rds` using a sandbox template.
The engine type must be MySQL v8.4.5, and it must be a `db.t3.micro` type instance.
The master username must be `datacenter_admin` with an appropriate password.
The RDS storage type must be gp2, and the storage size must be 5GiB.
Create a database named `datacenter_db`.
Keep the rest of the configurations as default. Ensure the instance is in available state.
Adjust the security groups so that the `datacenter-ec2` instance can connect to the RDS on port 3306 and also open port 80 for the instance.
2) An EC2 instance named `datacenter-ec2` exists. Connect to this instance from the AWS console. Create an SSH key (`/root/.ssh/id_rsa`) on the aws-client host if it doesn't already exist. Add the public key to the authorized keys of the root user on the EC2 instance for password-less SSH access.

3) There is a file named `index.php` under the `/root` directory on the aws-client host. Copy this file to the `datacenter-ec2` instance under the `/var/www/html/` directory. Make the appropriate changes in the file to connect to the RDS.

4) You should see a Connected successfully message in the browser once you access the instance using the public IP.

## Specific Requirements:

1. Create a private RDS instance named `datacenter-rds` (sandbox template), MySQL v8.4.5, `db.t3.micro`, master username `datacenter_admin` with a password, storage type gp2 and size 5GiB, an initial database `datacenter_db`, in the available state.
2. Adjust security groups so `datacenter-ec2` can reach the RDS on port 3306, and open port 80 on the instance.
3. Create an SSH key `/root/.ssh/id_rsa` on the aws-client host (if absent) and add its public key to the root user's `authorized_keys` on `datacenter-ec2` for password-less SSH.
4. Copy `/root/index.php` to `datacenter-ec2` under `/var/www/html/` and edit it to connect to the RDS.
5. Accessing the instance's public IP in a browser must show `Connected successfully`.

## Solution

This challenge wires a web app to a private database end to end: a **private RDS MySQL** instance, **security groups** that let only the app reach it, a **web stack** (Apache + PHP) on the existing EC2 instance, and a PHP page that connects to the database. A private RDS instance has no public endpoint — it's reachable only from inside the VPC, which is why the EC2 instance (same VPC) is the client.

Two things this lab's environment decided for us. First, RDS requires a **DB subnet group** spanning at least two Availability Zones, so we build one from the default VPC's subnets. Second — the important gotcha — the `datacenter-ec2` instance turned out to be **Ubuntu 22.04**, not Amazon Linux: its default login user is `ubuntu` (not `ec2-user`), its web stack is `apache2` + `php` from **APT** (not `httpd`/`yum`), and it has no key pair or SSM agent we can use. So we bootstrap access with **EC2 Instance Connect** (which the challenge calls "connect from the console") as the `ubuntu` user, then install our own key for the root user.

### 📦 Variables

```bash
AWS_REGION="us-east-1"
EC2_NAME="datacenter-ec2"
DB_IDENTIFIER="datacenter-rds"
DB_SUBNET_GROUP="datacenter-rds-subg"
RDS_SG_NAME="datacenter-rds-sg"
ENGINE_VERSION="8.4.5"
DB_CLASS="db.t3.micro"
MASTER_USER="datacenter_admin"
MASTER_PASS="Datacenter1234!"
DB_NAME="datacenter_db"
KEY_PATH="/root/.ssh/id_rsa"
INDEX_SRC="/root/index.php"
```

> **Why:** These collect the challenge-fixed values (the RDS name, engine version `8.4.5`, `db.t3.micro`, master user `datacenter_admin`, database `datacenter_db`, the EC2 name, the SSH key path, and the `index.php` source) plus a couple we choose: the DB subnet group and RDS security-group names. `AWS_REGION` is pinned to `us-east-1`. The password `Datacenter1234!` satisfies RDS's rules (8+ characters; it avoids `/`, `@`, `"`, and spaces, which RDS forbids in a master password). Resource **IDs** are discovered in the steps.

### 🔎 Step 1: Discover the EC2 instance and its network

```bash
EC2_ID=$(aws ec2 describe-instances --region "$AWS_REGION" \
  --filters "Name=tag:Name,Values=$EC2_NAME" "Name=instance-state-name,Values=running" \
  --query "Reservations[0].Instances[0].InstanceId" --output text)

EC2_VPC_ID=$(aws ec2 describe-instances --region "$AWS_REGION" --instance-ids "$EC2_ID" \
  --query "Reservations[0].Instances[0].VpcId" --output text)

EC2_AZ=$(aws ec2 describe-instances --region "$AWS_REGION" --instance-ids "$EC2_ID" \
  --query "Reservations[0].Instances[0].Placement.AvailabilityZone" --output text)

EC2_PUB_IP=$(aws ec2 describe-instances --region "$AWS_REGION" --instance-ids "$EC2_ID" \
  --query "Reservations[0].Instances[0].PublicIpAddress" --output text)

EC2_SG_ID=$(aws ec2 describe-instances --region "$AWS_REGION" --instance-ids "$EC2_ID" \
  --query "Reservations[0].Instances[0].SecurityGroups[0].GroupId" --output text)
```

Real values from the lab run:

```
EC2_ID=i-03129c94c13326d32  VPC=vpc-0f1b651349b612f5a  AZ=us-east-1a
PUB_IP=54.197.12.217  EC2_SG=sg-0e151f8dc2decc833  (AMI: Ubuntu 22.04)
```

> **Why:** We look up the existing instance by its *Name* tag with `describe-instances` (never recreate it). From it we capture the **VPC** (to place the RDS and its subnet group in the same network), the **Availability Zone** (required by EC2 Instance Connect), the **public IP** (to SSH and to browse to), and the instance's **security group** (which we both open port 80 on and reference from the RDS security group). Checking the AMI revealed Ubuntu 22.04 — which dictates the `ubuntu` login user and APT-based web stack used later.

### 🔓 Step 2: Open ports 80 and 22 on the EC2 security group

```bash
aws ec2 authorize-security-group-ingress --region "$AWS_REGION" \
  --group-id "$EC2_SG_ID" --protocol tcp --port 80 --cidr 0.0.0.0/0

aws ec2 authorize-security-group-ingress --region "$AWS_REGION" \
  --group-id "$EC2_SG_ID" --protocol tcp --port 22 --cidr 0.0.0.0/0
```

> **Why:** A **security group** is the instance's virtual firewall. `authorize-security-group-ingress` adds inbound rules: port `80` so the web page is reachable from a browser (the challenge's "open port 80"), and port `22` so we can SSH in to configure it. `--protocol tcp` and `--port` pick the port; `--cidr 0.0.0.0/0` allows it from any source. Port 22 was already open in this run, so that rule was a no-op — leaving it in is harmless.

### 🛡️ Step 3: Create an RDS security group that only the EC2 instance can reach

```bash
RDS_SG_ID=$(aws ec2 create-security-group --region "$AWS_REGION" \
  --group-name "$RDS_SG_NAME" \
  --description "RDS access for $EC2_NAME" --vpc-id "$EC2_VPC_ID" \
  --query "GroupId" --output text)

aws ec2 create-tags --region "$AWS_REGION" \
  --resources "$RDS_SG_ID" --tags Key=Name,Value="$RDS_SG_NAME"

aws ec2 authorize-security-group-ingress --region "$AWS_REGION" \
  --group-id "$RDS_SG_ID" \
  --ip-permissions "IpProtocol=tcp,FromPort=3306,ToPort=3306,UserIdGroupPairs=[{GroupId=$EC2_SG_ID}]"
```

Real value from the lab run: `RDS_SG_ID=sg-04da52ef3c02b6a4e`.

> **Why:** Rather than opening MySQL to the world, we create a dedicated security group for the RDS instance and allow port `3306` **only from the EC2 instance's security group**. The `--ip-permissions` form with `UserIdGroupPairs=[{GroupId=...}]` is a **source-security-group** rule: it permits any instance in `sg-0e151f8dc2decc833` (our EC2) to connect, without hardcoding IPs — the cleanest way to satisfy "the `datacenter-ec2` instance can connect to the RDS on port 3306." `3306` is MySQL's port.

### 🧩 Step 4: Create the DB subnet group

```bash
aws rds create-db-subnet-group --region "$AWS_REGION" \
  --db-subnet-group-name "$DB_SUBNET_GROUP" \
  --db-subnet-group-description "Subnet group for $DB_IDENTIFIER" \
  --subnet-ids subnet-0971814634dc6edc7 subnet-013661f5a40f307b6 subnet-0126e4ea0a50b0993 subnet-0d69bd75445c918b5 subnet-08d23694400ed1ede subnet-02245273e72c0a7ed
```

> **Why:** RDS needs a **DB subnet group** — a named set of subnets across at least two Availability Zones — before it can place an instance in a VPC. `create-db-subnet-group` builds one from all six of the default VPC's subnets (discovered with `aws ec2 describe-subnets --filters "Name=vpc-id,Values=$EC2_VPC_ID"`), which guarantees multi-AZ coverage. `--subnet-ids` lists the members.

### 🗄️ Step 5: Create the private RDS instance

```bash
aws rds create-db-instance --region "$AWS_REGION" \
  --db-instance-identifier "$DB_IDENTIFIER" \
  --db-instance-class "$DB_CLASS" \
  --engine mysql --engine-version "$ENGINE_VERSION" \
  --master-username "$MASTER_USER" --master-user-password "$MASTER_PASS" \
  --db-name "$DB_NAME" \
  --storage-type gp2 --allocated-storage 5 \
  --db-subnet-group-name "$DB_SUBNET_GROUP" \
  --vpc-security-group-ids "$RDS_SG_ID" \
  --no-publicly-accessible --no-multi-az --backup-retention-period 0
```

The instance entered the `creating` state (Engine `8.4.5`, Class `db.t3.micro`).

> **Why:** `create-db-instance` provisions the database with exactly the challenge's spec: `--engine mysql --engine-version 8.4.5`, `--db-instance-class db.t3.micro`, `--storage-type gp2` with `--allocated-storage 5` (5 GiB), master login via `--master-username`/`--master-user-password`, and `--db-name datacenter_db` which creates the initial database. `--db-subnet-group-name` and `--vpc-security-group-ids` place it in our VPC behind the RDS security group. `--no-publicly-accessible` makes it **private** (no public endpoint). `--no-multi-az` and `--backup-retention-period 0` keep it to the minimal "sandbox" footprint the challenge describes. It returns immediately as `creating`; we wait for it in Step 8.

### 🔑 Step 6: Ensure the SSH key exists on aws-client

```bash
ssh-keygen -t rsa -b 2048 -f "$KEY_PATH" -N ""
```

Real values from the lab run: the key `/root/.ssh/id_rsa` (and `.pub`) did not exist, so it was generated.

> **Why:** The challenge wants password-less SSH from the aws-client host using `/root/.ssh/id_rsa`. `ssh-keygen -t rsa` creates an RSA key pair, `-b 2048` sets the size, `-f` the output path, and `-N ""` gives it no passphrase (required for unattended, password-less use). The key didn't exist yet on this run, so it was created; if it already existed we'd reuse it as-is.

### 🐧 Step 7: Configure root SSH and the Apache/PHP stack on the Ubuntu instance

Because the instance has no key pair we own, we use **EC2 Instance Connect** (the CLI equivalent of "connect from the console") to push our public key for a one-time login as `ubuntu`, then install it for `root` and set up the web stack:

```bash
aws ec2-instance-connect send-ssh-public-key --region "$AWS_REGION" \
  --instance-id "$EC2_ID" --availability-zone "$EC2_AZ" \
  --instance-os-user ubuntu --ssh-public-key "file://$KEY_PATH.pub"
```

```bash
PUBKEY=$(cat "$KEY_PATH.pub")

ssh -i "$KEY_PATH" -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null \
  ubuntu@"$EC2_PUB_IP" "sudo bash -s" <<EOF
cloud-init status --wait
install -d -m 700 /root/.ssh
echo "$PUBKEY" > /root/.ssh/authorized_keys
chmod 600 /root/.ssh/authorized_keys
echo "PermitRootLogin prohibit-password" > /etc/ssh/sshd_config.d/00-permitroot.conf
systemctl reload ssh
DEBIAN_FRONTEND=noninteractive apt-get update -qq
DEBIAN_FRONTEND=noninteractive apt-get install -y apache2 php libapache2-mod-php php-mysql
systemctl enable --now apache2
EOF
```

> **Why:** `ec2-instance-connect send-ssh-public-key` injects our public key into the instance's metadata for a 60-second window; `--instance-os-user ubuntu` is critical — on this Ubuntu AMI the default user is `ubuntu`, not `ec2-user` (using the wrong user gives `Permission denied (publickey)`). We then SSH in and run the setup as root via `sudo bash -s`. `cloud-init status --wait` blocks until first-boot setup finishes and releases the APT/dpkg lock, so the install doesn't fail with "Could not get lock." We write our public key into `/root/.ssh/authorized_keys` (overwriting Ubuntu's default forced-command banner entry) so root SSH works cleanly, and add a `PermitRootLogin prohibit-password` drop-in so `sshd` accepts key-based root logins while still refusing passwords; `systemctl reload ssh` applies it (the service is `ssh` on Ubuntu, not `sshd`). Finally APT installs `apache2`, `php`, `libapache2-mod-php` (runs PHP inside Apache), and `php-mysql` (the MySQL/`mysqli` driver the page uses), and `systemctl enable --now apache2` starts the web server.

### ⏳ Step 8: Wait for the RDS instance and get its endpoint

```bash
aws rds wait db-instance-available --region "$AWS_REGION" \
  --db-instance-identifier "$DB_IDENTIFIER"

RDS_ENDPOINT=$(aws rds describe-db-instances --region "$AWS_REGION" \
  --db-instance-identifier "$DB_IDENTIFIER" \
  --query "DBInstances[0].Endpoint.Address" --output text)
```

Real value from the lab run: `RDS_ENDPOINT=datacenter-rds.cbuowuqg4cfx.us-east-1.rds.amazonaws.com`.

> **Why:** Creating an RDS instance takes several minutes. `wait db-instance-available` blocks until it reports `available` (and until the **endpoint** — the DNS hostname the app connects to — is assigned). We then read that endpoint with `describe-db-instances` because the PHP file needs it as its database host. A multi-minute wait here is expected, not a failure.

### 🌐 Step 9: Deploy index.php and inject the RDS connection details

The provided `/root/index.php` connects to MySQL using four placeholder variables:

```php
$dbname = '<dbname>';
$dbuser = '<dbuser>';
$dbpass = '<dbpass>';
$dbhost = '<dbhost>';
```

We copy it to the web root as root, then replace the placeholders with the real values:

```bash
scp -i "$KEY_PATH" -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null \
  "$INDEX_SRC" root@"$EC2_PUB_IP":/var/www/html/index.php

ssh -i "$KEY_PATH" -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null \
  root@"$EC2_PUB_IP" "bash -s" <<EOF
sed -i -e 's|<dbname>|datacenter_db|g' -e 's|<dbuser>|datacenter_admin|g' -e 's|<dbpass>|Datacenter1234!|g' -e 's|<dbhost>|datacenter-rds.cbuowuqg4cfx.us-east-1.rds.amazonaws.com|g' /var/www/html/index.php
rm -f /var/www/html/index.html
systemctl restart apache2
EOF
```

> **Why:** `scp` copies the file over SSH; because we set up root access in Step 7, we can write straight into `/var/www/html/` (owned by root). `sed -i` edits the file in place, substituting each `<placeholder>` with the real database name, user, password, and the RDS **endpoint** as the host — this is the "make the appropriate changes to connect to the RDS" step. We `rm -f /var/www/html/index.html` because Apache serves the default `index.html` **before** `index.php`, so leaving it would hide our page. `systemctl restart apache2` reloads everything. The page uses `mysqli_connect($dbhost, $dbuser, $dbpass)` then `mysqli_select_db($link, $dbname)` and prints `Connected successfully` on success.

### ✅ Step 10: Verify

```bash
curl -s "http://$EC2_PUB_IP/index.php"

aws rds describe-db-instances --region "$AWS_REGION" \
  --db-instance-identifier "$DB_IDENTIFIER" \
  --query "DBInstances[0].{Id:DBInstanceIdentifier,Status:DBInstanceStatus,Version:EngineVersion,Class:DBInstanceClass,Storage:AllocatedStorage,StorageType:StorageType,Public:PubliclyAccessible,Endpoint:Endpoint.Address}" \
  --output table
```

The page returns the success message, and the RDS instance matches every requirement:

```
Connected successfully<br />

----------------------------------------------------------------------------
|                            DescribeDBInstances                           |
+-------------+------------------------------------------------------------+
|  Class      |  db.t3.micro                                               |
|  Endpoint   |  datacenter-rds.cbuowuqg4cfx.us-east-1.rds.amazonaws.com   |
|  Id         |  datacenter-rds                                            |
|  Public     |  False                                                     |
|  Status     |  available                                                 |
|  Storage    |  5                                                         |
|  StorageType|  gp2                                                       |
|  Version    |  8.4.5                                                     |
+-------------+------------------------------------------------------------+
```

> **Why:** `curl -s http://<public-ip>/index.php` fetches the page the way a browser would; `Connected successfully` proves the whole chain works — Apache serves the PHP, PHP's `mysqli` driver reaches the RDS endpoint over port 3306 (allowed by the RDS security group from the EC2's group), and the credentials and `datacenter_db` database are correct. `describe-db-instances` confirms the instance is `available` and matches the spec: MySQL `8.4.5`, `db.t3.micro`, `gp2` storage of `5` GiB, and `Public: False` (private).

## Best Practices

- **Reference the client's security group, not a CIDR.** Allowing `3306` from the EC2's security group (`UserIdGroupPairs`) is tighter and survives IP changes better than opening a CIDR range.
- **Keep the database private.** `--no-publicly-accessible` ensures the RDS instance has no internet endpoint; only in-VPC resources like the app server can reach it.
- **Check the AMI before assuming the login user.** Ubuntu uses `ubuntu` + APT + `apache2`; Amazon Linux uses `ec2-user` + yum/dnf + `httpd`. Guessing wrong wastes an SSH attempt and installs the wrong packages.
- **Wait out cloud-init before APT.** On a fresh boot, `cloud-init status --wait` avoids the "Could not get lock" dpkg error.
- **Remove the default index.html.** Apache prefers `index.html` over `index.php`; delete it so your application page is actually served.
- **Don't hardcode DB credentials in source for production.** This lab edits them into `index.php`; real apps should read them from AWS Secrets Manager or environment variables.

### 📚 Official Documentation

- [Creating an Amazon RDS DB instance](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_CreateDBInstance.html)
- [Controlling access with security groups (Amazon RDS)](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Overview.RDSSecurityGroups.html)
- [Connect using EC2 Instance Connect](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/connect-using-eic.html)
- [create-db-instance — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/rds/create-db-instance.html)
- [authorize-security-group-ingress — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/ec2/authorize-security-group-ingress.html)
