# Day 26: Configuring an EC2 Instance as a Web Server with Nginx

The Nautilus DevOps Team is working on setting up a new web server for a critical application. The team lead has requested you to create an EC2 instance that will serve as a web server using Nginx. This instance will be part of the initial infrastructure setup for the Nautilus project. Ensuring that the server is correctly configured and accessible from the internet is crucial for the upcoming deployment phase.

As a member of the Nautilus DevOps Team, your task is to create an EC2 instance with the following specifications:

**Instance Name:** The EC2 instance must be named `nautilus-ec2`.

**AMI:** Use any available Ubuntu AMI to create this instance.

**User Data Script:** Configure the instance to run a user data script during its launch. This script should:

Install the Nginx package.
Start the Nginx service.
**Security Group:** Ensure that the instance allows HTTP traffic on port 80 from the internet.

## Specific Requirements:

1. **Instance Name:** The EC2 instance must be named `nautilus-ec2`.
2. **AMI:** Use any available Ubuntu AMI to create this instance.
3. **User Data Script:** Configure the instance to run a user data script during its launch. This script should:
4. Install the Nginx package.
5. Start the Nginx service.
6. **Security Group:** Ensure that the instance allows HTTP traffic on port 80 from the internet.

## Solution

The instance is bootstrapped with EC2 user data. Cloud-init runs the shell script during the first boot, installs Nginx, and starts the service. The instance receives a public IPv4 address, while its security group allows inbound TCP port `80` from the internet so the web server can be reached externally.

### 📦 Variables

```bash
AWS_REGION="us-east-1"
INSTANCE_NAME="nautilus-ec2"
HTTP_PORT=80
```

### 🔎 Step 1: Discover the default VPC, subnet, and security group

Find the default VPC:

```bash
aws ec2 describe-vpcs \
  --filters "Name=isDefault,Values=true" \
  --query "Vpcs[0].VpcId" \
  --output text
```

The lab returned:

```text
vpc-0ba7d44cef45203ae
```

Find a default subnet in that VPC:

```bash
aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=vpc-0ba7d44cef45203ae" "Name=default-for-az,Values=true" \
  --query "Subnets[0].SubnetId" \
  --output text
```

The selected subnet was:

```text
subnet-08e7019f60369b1c0
```

Find the default security group in the VPC:

```bash
aws ec2 describe-security-groups \
  --filters "Name=vpc-id,Values=vpc-0ba7d44cef45203ae" "Name=group-name,Values=default" \
  --query "SecurityGroups[0].GroupId" \
  --output text
```

The security group was:

```text
sg-0e1420612c2b631c9
```

> **Why:** `describe-vpcs`, `describe-subnets`, and `describe-security-groups` retrieve the existing network resources. `--filters` limits each lookup: `isDefault` selects the default VPC, `vpc-id` keeps the subnet and security-group searches inside that VPC, `default-for-az` selects a default subnet, and `group-name=default` selects the default security group. `--query` extracts only the ID needed by later commands, and `--output text` prints a clean value instead of JSON.

### 🖼️ Step 2: Find an available Ubuntu AMI

```bash
aws ec2 describe-images \
  --owners 099720109477 \
  --filters \
    "Name=name,Values=ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*" \
    "Name=state,Values=available" \
    "Name=architecture,Values=x86_64" \
    "Name=root-device-type,Values=ebs" \
    "Name=virtualization-type,Values=hvm" \
  --query "sort_by(Images, &CreationDate)[-1].ImageId" \
  --output text
```

The newest matching Ubuntu AMI was:

```text
ami-0d001f8052688dc45
```

> **Why:** `describe-images` searches available AMIs. `--owners 099720109477` limits the search to Canonical's Ubuntu images. The filters require an available `x86_64` Ubuntu 22.04 image with an EBS root device and HVM virtualization. `sort_by(Images, &CreationDate)[-1]` selects the newest matching image, while `--query` and `--output text` return only its AMI ID.

### 🔍 Step 3: Check whether the web-server instance already exists

```bash
aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=nautilus-ec2" "Name=instance-state-name,Values=pending,running,stopping,stopped" \
  --query "Reservations[0].Instances[0].InstanceId" \
  --output text
```

No existing usable instance named `nautilus-ec2` was found, so the instance was created.

> **Why:** `describe-instances` checks for an existing EC2 instance before creation. The `tag:Name` filter searches for the required logical name, and `instance-state-name` excludes terminated instances. This lookup prevents duplicate resources when a challenge is rerun.

### 🧰 Step 4: Prepare the Nginx user data script

```bash
cat > nginx-user-data.sh <<'EOF'
#!/bin/bash
apt-get update
apt-get install -y nginx
systemctl enable nginx
systemctl start nginx
EOF
```

> **Why:** `cat` writes the launch script to `nginx-user-data.sh`. The `#!/bin/bash` line tells cloud-init which interpreter should run the script. `apt-get update` refreshes the Ubuntu package index, `apt-get install -y nginx` installs Nginx without waiting for interactive confirmation, `systemctl enable nginx` configures the service to start on future boots, and `systemctl start nginx` starts it immediately. User data scripts run as root during the instance's first boot, so `sudo` is not needed.

### 🚀 Step 5: Launch the EC2 instance with user data

```bash
aws ec2 run-instances \
  --image-id ami-0d001f8052688dc45 \
  --instance-type t2.micro \
  --subnet-id subnet-08e7019f60369b1c0 \
  --security-group-ids sg-0e1420612c2b631c9 \
  --associate-public-ip-address \
  --user-data file://nginx-user-data.sh \
  --count 1 \
  --query "Instances[0].InstanceId" \
  --output text
```

The instance was created with ID:

```text
i-09d38e23abdb4857b
```

> **Why:** `run-instances` launches the EC2 server. `--image-id` selects the Ubuntu AMI, `--instance-type t2.micro` chooses a suitable lab instance size, `--subnet-id` places it in the selected subnet, and `--security-group-ids` attaches the discovered security group. `--associate-public-ip-address` requests a public IPv4 address. `--user-data file://nginx-user-data.sh` passes the local script to cloud-init at launch, and `--count 1` requests exactly one instance. `--query` extracts the new instance ID and `--output text` prints it directly.

### 🏷️ Step 6: Apply the instance name tag

```bash
aws ec2 create-tags \
  --resources i-09d38e23abdb4857b \
  --tags "Key=Name,Value=nautilus-ec2"
```

> **Why:** `create-tags` adds metadata after the instance is created. `--resources` identifies the EC2 instance, and `--tags` sets the `Name` key to `nautilus-ec2`. Keeping tagging separate makes the creation and naming operations easy to understand and repeat.

### ⏳ Step 7: Wait for the instance to enter the running state

```bash
aws ec2 wait instance-running \
  --instance-ids i-09d38e23abdb4857b
```

The waiter completed successfully:

```text
Instance is running: i-09d38e23abdb4857b
```

> **Why:** `wait instance-running` polls the instance state until EC2 reports `running`. `--instance-ids` identifies the instance being checked. User data continues during the boot process, so the web endpoint may need a few additional seconds before Nginx is ready.

### 🔓 Step 8: Allow public HTTP traffic

Check whether the security group already has the required rule:

```bash
aws ec2 describe-security-groups \
  --group-ids sg-0e1420612c2b631c9 \
  --query "length(SecurityGroups[0].IpPermissions[?IpProtocol=='tcp' && FromPort==\`80\` && ToPort==\`80\`].IpRanges[?CidrIp=='0.0.0.0/0'])" \
  --output text
```

No matching public HTTP rule existed, so it was added:

```bash
aws ec2 authorize-security-group-ingress \
  --group-id sg-0e1420612c2b631c9 \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0
```

AWS created this security-group rule:

```json
{
    "Return": true,
    "SecurityGroupRules": [
        {
            "SecurityGroupRuleId": "sgr-043148e0cd455cf6c",
            "GroupId": "sg-0e1420612c2b631c9",
            "GroupOwnerId": "081866666028",
            "IsEgress": false,
            "IpProtocol": "tcp",
            "FromPort": 80,
            "ToPort": 80,
            "CidrIpv4": "0.0.0.0/0",
            "SecurityGroupRuleArn": "arn:aws:ec2:us-east-1:081866666028:security-group-rule/sgr-043148e0cd455cf6c"
        }
    ]
}
```

> **Why:** The `describe-security-groups` query checks the security group's inbound permissions before changing them. `authorize-security-group-ingress` adds an inbound rule. `--group-id` identifies the group, `--protocol tcp` selects TCP, `--port 80` limits traffic to HTTP, and `--cidr 0.0.0.0/0` allows clients from any IPv4 address on the internet. The rule is ingress-only, so it controls traffic entering the instance.

### ✅ Step 9: Verify

Verify the EC2 instance and its public address:

```bash
aws ec2 describe-instances \
  --instance-ids i-09d38e23abdb4857b \
  --query "Reservations[0].Instances[0].{ImageId:ImageId,InstanceId:InstanceId,InstanceType:InstanceType,Name:Tags[?Key=='Name']|[0].Value,PublicIp:PublicIpAddress,State:State.Name}" \
  --output table
```

```text
-------------------------------------------
|            DescribeInstances            |
+---------------+-------------------------+
|  ImageId      |  ami-0d001f8052688dc45  |
|  InstanceId   |  i-09d38e23abdb4857b    |
|  InstanceType |  t2.micro               |
|  Name         |  nautilus-ec2           |
|  PublicIp     |  34.229.119.120         |
|  State        |  running                |
+---------------+-------------------------+
```

Confirm the security group identity:

```bash
aws ec2 describe-security-groups \
  --group-ids sg-0e1420612c2b631c9 \
  --query "SecurityGroups[0].{GroupId:GroupId,GroupName:GroupName}" \
  --output table
```

```text
---------------------------------------
|       DescribeSecurityGroups        |
+-----------------------+-------------+
|        GroupId        |  GroupName  |
+-----------------------+-------------+
|  sg-0e1420612c2b631c9 |  default    |
+-----------------------+-------------+
```

The authorization response in Step 8 confirms that this group allows TCP port `80` from `0.0.0.0/0`. Finally, test the public Nginx endpoint:

```bash
curl -fsS --connect-timeout 5 --max-time 10 http://34.229.119.120 | sed -n '1,8p'
```

```text
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
```

> **Why:** `describe-instances` confirms the AMI, instance ID, type, `Name` tag, public IP, and `running` state. `describe-security-groups` confirms the selected group identity, while the authorization response proves the required ingress rule was created. `curl` makes an HTTP request to the public IP; `-f` treats HTTP errors as failures, `-s` suppresses progress output, `-S` keeps error messages visible, `--connect-timeout 5` limits connection setup to five seconds, and `--max-time 10` limits the complete request. `sed -n '1,8p'` prints only the first eight response lines, which is enough to confirm the default Nginx page.

The instance is `running`, publicly reachable at `34.229.119.120`, and returned the default Nginx page successfully.

## Best Practices

- **Use user data for first-boot configuration.** Installing and starting Nginx during launch makes the instance ready without a separate manual configuration session.
- **Use a public IP only when required.** This challenge requires internet access, so the instance received a public IPv4 address and an HTTP ingress rule.
- **Check before creating.** The instance name was looked up before launch to avoid duplicates if the procedure is rerun.
- **Keep tags separate from creation.** Applying the `Name` tag with `create-tags` makes resource identification explicit and easy to verify.
- **Limit the security-group rule to the required port.** Only TCP port `80` is opened; other inbound ports remain closed by default.
- **Allow time for user data.** The instance can be `running` before cloud-init finishes installing Nginx, so the HTTP check should tolerate a short bootstrap delay.

### 📚 Official Documentation

- [Run commands when you launch an EC2 instance with user data input](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/user-data.html)
- [describe-vpcs — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/ec2/describe-vpcs.html)
- [describe-subnets — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/ec2/describe-subnets.html)
- [describe-security-groups — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/ec2/describe-security-groups.html)
- [describe-images — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/ec2/describe-images.html)
- [describe-instances — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/ec2/describe-instances.html)
- [run-instances — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/ec2/run-instances.html)
- [create-tags — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/ec2/create-tags.html)
- [instance-running waiter — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/ec2/wait/instance-running.html)
- [authorize-security-group-ingress — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/ec2/authorize-security-group-ingress.html)
