# Day 22: Configuring Secure SSH Access to an EC2 Instance

The Nautilus DevOps team needs to set up a new EC2 instance that can be accessed securely from their landing host (`aws-client`). The instance should be of type `t2.micro` and named `devops-ec2`. A new SSH key with name `id_rsa` should be created on the `aws-client` host under the/root/.ssh/ folder, if it doesn't already exist. This key should then be added to the root user's authorised keys on the EC2 instance, allowing passwordless SSH access from the `aws-client` host.

## Specific Requirements:

1. The Nautilus DevOps team needs to set up a new EC2 instance that can be accessed securely from their landing host (`aws-client`).
2. The instance should be of type `t2.micro` and named `devops-ec2`.
3. A new SSH key with name `id_rsa` should be created on the `aws-client` host under the/root/.ssh/ folder, if it doesn't already exist.
4. This key should then be added to the root user's authorised keys on the EC2 instance, allowing passwordless SSH access from the `aws-client` host.

## Solution

The EC2 key pair registered with AWS is used to access the default Amazon Linux user, `ec2-user`. The same public key is then installed in `/root/.ssh/authorized_keys`. Amazon Linux may include a forced command on the original key entry that rejects direct `root` sessions, so the workflow replaces entries for this key and sets `PermitRootLogin prohibit-password`. This allows key-based root access while keeping password authentication disabled.

### 📦 Variables

```bash
AWS_REGION="us-east-1"
INSTANCE_NAME="devops-ec2"
INSTANCE_TYPE="t2.micro"
KEY_NAME="id_rsa"
KEY_DIR="/root/.ssh"
```

### 🔑 Step 1: Prepare the local SSH key

```bash
PRIVATE_KEY="$KEY_DIR/$KEY_NAME"
PUBLIC_KEY="$PRIVATE_KEY.pub"

install -d -m 700 "$KEY_DIR"

ls "$PRIVATE_KEY" "$PUBLIC_KEY"
```

Neither file existed yet on this lab's `aws-client` host, so both had to be generated:

```bash
ssh-keygen -t rsa -b 2048 -f "$PRIVATE_KEY" -N ""
ssh-keygen -y -f "$PRIVATE_KEY" > "$PUBLIC_KEY"

chmod 600 "$PRIVATE_KEY"
chmod 644 "$PUBLIC_KEY"
ssh-keygen -lf "$PUBLIC_KEY"
```

The key was created on `aws-client` with this fingerprint:

```text
2048 SHA256:9SLaz/ZSayyQxLte0TXUTaqjn7C1ZgPblKjxSfq7RTw root@aws-client (RSA)
```

> **Why:** `install -d` creates the `.ssh` directory when needed, and `-m 700` limits access to its owner. `ls` checks whether the key files already exist, since the task only requires creating them "if it doesn't already exist" — on this lab run they were absent, so both were generated. `ssh-keygen -t rsa` creates an RSA key pair, `-b 2048` selects the key size, `-f` sets the output path, and `-N ""` creates the key without a passphrase for the lab's passwordless SSH requirement. `ssh-keygen -y` derives a public key from an existing private key. `chmod 600` protects the private key, while `chmod 644` makes the public key readable. The `-l` and `-f` options of `ssh-keygen` print the public key fingerprint.

### 📥 Step 2: Register the public key with EC2

```bash
aws ec2 describe-key-pairs \
  --region "$AWS_REGION" \
  --key-names "$KEY_NAME"
```

No key pair named `id_rsa` existed yet in this account, so the public key had to be imported:

```bash
aws ec2 import-key-pair \
  --region "$AWS_REGION" \
  --key-name "$KEY_NAME" \
  --public-key-material "fileb://$PUBLIC_KEY"
```

The public key was imported as the EC2 key pair `id_rsa`:

```json
{
    "KeyFingerprint": "d9:26:ba:08:71:f3:db:8b:e6:b8:d8:76:34:ac:bc:2b",
    "KeyName": "id_rsa",
    "KeyPairId": "key-0b46f1758580acb36"
}
```

> **Why:** `describe-key-pairs` checks whether an EC2 key pair with the requested name already exists, preventing a duplicate import; `--key-names` selects the key pair to look up. `import-key-pair` registers an existing public key with EC2; `--key-name` gives it the AWS name, and `--public-key-material fileb://...` tells the AWS CLI to read the public key bytes from a local file. AWS stores only the public key, so the private key remains on `aws-client`.

### 🌐 Step 3: Discover the default network and restrict SSH ingress

```bash
VPC_ID=$(aws ec2 describe-vpcs \
  --region "$AWS_REGION" \
  --filters "Name=isDefault,Values=true" \
  --query "Vpcs[0].VpcId" \
  --output text)

SUBNET_ID=$(aws ec2 describe-subnets \
  --region "$AWS_REGION" \
  --filters "Name=vpc-id,Values=$VPC_ID" "Name=default-for-az,Values=true" \
  --query "Subnets[0].SubnetId" \
  --output text)

CLIENT_IP=$(curl -fsS https://checkip.amazonaws.com)
SSH_CIDR="$CLIENT_IP/32"
SECURITY_GROUP_NAME="devops-ec2-ssh"
SECURITY_GROUP_DESCRIPTION="SSH access for devops-ec2"

aws ec2 describe-security-groups \
  --region "$AWS_REGION" \
  --filters "Name=vpc-id,Values=$VPC_ID" "Name=group-name,Values=$SECURITY_GROUP_NAME" \
  --query "SecurityGroups[0].GroupId" \
  --output text
```

No security group named `devops-ec2-ssh` existed yet in the default VPC, so it had to be created and its ingress rule authorized:

```bash
SECURITY_GROUP_ID=$(aws ec2 create-security-group \
  --region "$AWS_REGION" \
  --group-name "$SECURITY_GROUP_NAME" \
  --description "$SECURITY_GROUP_DESCRIPTION" \
  --vpc-id "$VPC_ID" \
  --query "GroupId" \
  --output text)

aws ec2 create-tags \
  --region "$AWS_REGION" \
  --resources "$SECURITY_GROUP_ID" \
  --tags "Key=Name,Value=$SECURITY_GROUP_NAME"

aws ec2 authorize-security-group-ingress \
  --region "$AWS_REGION" \
  --group-id "$SECURITY_GROUP_ID" \
  --protocol tcp \
  --port 22 \
  --cidr "$SSH_CIDR"
```

The lab discovered these network resources and restricted SSH to the landing host's public IP:

```text
VPC_ID=vpc-02a217764d079b403
SUBNET_ID=subnet-08eec68799134f840
SECURITY_GROUP_ID=sg-0f83741d0f2197149
SSH_CIDR=65.108.255.62/32
```

The ingress rule was created successfully:

```json
{
    "Return": true,
    "SecurityGroupRules": [
        {
            "SecurityGroupRuleId": "sgr-0b0273d9764424479",
            "GroupId": "sg-0f83741d0f2197149",
            "GroupOwnerId": "164291094717",
            "IsEgress": false,
            "IpProtocol": "tcp",
            "FromPort": 22,
            "ToPort": 22,
            "CidrIpv4": "65.108.255.62/32",
            "SecurityGroupRuleArn": "arn:aws:ec2:us-east-1:164291094717:security-group-rule/sgr-0b0273d9764424479"
        }
    ]
}
```

> **Why:** `describe-vpcs` finds the default VPC, while `describe-subnets` selects a default subnet in that VPC. The `isDefault`, `vpc-id`, and `default-for-az` filters narrow each lookup, and `--query` with `--output text` extracts identifiers. `curl -fsS` obtains the public IPv4 address of `aws-client`; adding `/32` creates a CIDR range containing only that address. `describe-security-groups` checks for an existing group before creation. `create-security-group` creates a VPC security group using `--group-name`, `--description`, and `--vpc-id`. `create-tags` applies the Name tag separately. `authorize-security-group-ingress` adds an inbound rule; `--group-id` selects the group, `--protocol tcp` selects TCP, `--port 22` selects SSH, and `--cidr` restricts the source. A `/32` rule is substantially narrower than allowing SSH from `0.0.0.0/0`.

### 🐧 Step 4: Find the Amazon Linux AMI and launch the instance

```bash
AMI_ID=$(aws ssm get-parameter \
  --region "$AWS_REGION" \
  --name "/aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64" \
  --query "Parameter.Value" \
  --output text)

aws ec2 describe-instances \
  --region "$AWS_REGION" \
  --filters "Name=tag:Name,Values=$INSTANCE_NAME" "Name=instance-state-name,Values=pending,running,stopping,stopped" \
  --query "Reservations[0].Instances[0].InstanceId" \
  --output text
```

No instance tagged `devops-ec2` existed yet, so it had to be launched and tagged:

```bash
INSTANCE_ID=$(aws ec2 run-instances \
  --region "$AWS_REGION" \
  --image-id "$AMI_ID" \
  --instance-type "$INSTANCE_TYPE" \
  --key-name "$KEY_NAME" \
  --subnet-id "$SUBNET_ID" \
  --security-group-ids "$SECURITY_GROUP_ID" \
  --associate-public-ip-address \
  --count 1 \
  --query "Instances[0].InstanceId" \
  --output text)

aws ec2 create-tags \
  --region "$AWS_REGION" \
  --resources "$INSTANCE_ID" \
  --tags "Key=Name,Value=$INSTANCE_NAME"
```

The selected AMI and instance were:

```text
AMI_ID=ami-0f303bae6b670e0ed
INSTANCE_ID=i-099fe1065c5b19d86
```

> **Why:** `get-parameter` reads a public Systems Manager Parameter Store value containing the current Amazon Linux 2023 AMI ID for the Region. `--name` selects that parameter, `--query` extracts `Parameter.Value`, and `--output text` makes it usable in the launch command. `describe-instances` checks for an existing instance with the required Name tag before creating another one. `run-instances` launches the instance; `--image-id` selects the AMI, `--instance-type` selects `t2.micro`, `--key-name` associates the EC2 key pair, `--subnet-id` and `--security-group-ids` place and protect the instance, `--associate-public-ip-address` provides direct reachability from `aws-client`, and `--count 1` requests one instance. The final `create-tags` call assigns the required Name tag.

### ⏳ Step 5: Wait for the instance and obtain its public IP

```bash
aws ec2 describe-instances \
  --region "$AWS_REGION" \
  --instance-ids "$INSTANCE_ID" \
  --query "Reservations[0].Instances[0].State.Name" \
  --output text
```

The instance had just been launched, so its state was already `pending`/`running`, not `stopped` — no `start-instances` call was needed. Had the state come back `stopped`, the next command would be `aws ec2 start-instances --region "$AWS_REGION" --instance-ids "$INSTANCE_ID"` before waiting.

```bash
aws ec2 wait instance-running \
  --region "$AWS_REGION" \
  --instance-ids "$INSTANCE_ID"

aws ec2 wait instance-status-ok \
  --region "$AWS_REGION" \
  --instance-ids "$INSTANCE_ID"

PUBLIC_IP=$(aws ec2 describe-instances \
  --region "$AWS_REGION" \
  --instance-ids "$INSTANCE_ID" \
  --query "Reservations[0].Instances[0].PublicIpAddress" \
  --output text)

echo "PUBLIC_IP=$PUBLIC_IP"
```

The instance's public IP was:

```text
PUBLIC_IP=52.207.231.63
```

> **Why:** `describe-instances` reads the current state and public address. `--instance-ids` scopes the response to the known instance. If the instance is stopped, `start-instances` starts it. `wait instance-running` pauses until EC2 reports the `running` state, while `wait instance-status-ok` waits for the instance status checks to pass. These waiters avoid attempting SSH while the operating system or network interface is still initializing.

### 🔐 Step 6: Replace the restricted root key entry

```bash
timeout 30 scp -i "$PRIVATE_KEY" \
  -o StrictHostKeyChecking=no \
  -o UserKnownHostsFile=/dev/null \
  -o ConnectTimeout=10 \
  -o BatchMode=yes \
  "$PUBLIC_KEY" "ec2-user@$PUBLIC_IP:/tmp/day22-id_rsa.pub"

timeout 30 ssh -i "$PRIVATE_KEY" \
  -o StrictHostKeyChecking=no \
  -o UserKnownHostsFile=/dev/null \
  -o ConnectTimeout=10 \
  -o BatchMode=yes \
  -o RequestTTY=no \
  "ec2-user@$PUBLIC_IP" \
  "sudo -n install -d -m 700 /root/.ssh && sudo -n touch /root/.ssh/authorized_keys && sudo -n chmod 600 /root/.ssh/authorized_keys && sudo -n awk 'NR==FNR {key_type=\$1; key_data=\$2; next} {if (\$1 != key_type || \$2 != key_data) print}' /tmp/day22-id_rsa.pub /root/.ssh/authorized_keys | sudo -n tee /tmp/day22-authorized_keys >/dev/null && sudo -n cat /tmp/day22-id_rsa.pub | sudo -n tee -a /tmp/day22-authorized_keys >/dev/null && sudo -n install -o root -g root -m 600 /tmp/day22-authorized_keys /root/.ssh/authorized_keys && sudo -n rm -f /tmp/day22-id_rsa.pub /tmp/day22-authorized_keys"
```

> **Why:** `scp` copies the public key to a temporary path using the private key specified by `-i`. `StrictHostKeyChecking=no` and `UserKnownHostsFile=/dev/null` keep the disposable lab from prompting about its temporary host key, while `ConnectTimeout=10` limits connection setup time and `BatchMode=yes` prevents an interactive password prompt. The first SSH connection uses the Amazon Linux default user, `ec2-user`. `sudo -n` runs each privileged command without prompting for a password. The `awk` command removes every existing entry with the same key type and key data, including entries with a forced `command=` option, and then the clean public key is appended. `install -o root -g root -m 600` restores the required owner and permissions on `authorized_keys`.

### 🛡️ Step 7: Allow key-only root SSH access

```bash
timeout 30 ssh -i "$PRIVATE_KEY" \
  -o StrictHostKeyChecking=no \
  -o UserKnownHostsFile=/dev/null \
  -o ConnectTimeout=10 \
  -o BatchMode=yes \
  -o RequestTTY=no \
  "ec2-user@$PUBLIC_IP" \
  "sudo -n sed -i -E '/^[[:space:]]*#?[[:space:]]*PermitRootLogin[[:space:]]+/d' /etc/ssh/sshd_config && echo 'PermitRootLogin prohibit-password' | sudo -n tee -a /etc/ssh/sshd_config >/dev/null && sudo -n sshd -t && sudo -n systemctl reload sshd"
```

> **Why:** Amazon Linux can place a forced command on the original key entry that prints a message telling the operator to use `ec2-user` instead of `root`. The command updates `PermitRootLogin` to `prohibit-password`, which permits root authentication with an authorized SSH key while continuing to reject password authentication. `sed -i -E '/.../d'` deletes any existing `PermitRootLogin` line (commented or not) so the file never ends up with duplicate directives, then `tee -a` appends the single desired line — an idempotent replace that works the same whether or not the directive was already present. `sshd -t` validates the SSH daemon configuration before the service is reloaded, and `systemctl reload` applies the change without unnecessarily restarting the instance.

### ✅ Step 8: Verify

```bash
timeout 30 ssh -i "$PRIVATE_KEY" \
  -o StrictHostKeyChecking=no \
  -o UserKnownHostsFile=/dev/null \
  -o ConnectTimeout=10 \
  -o BatchMode=yes \
  -o RequestTTY=no \
  "root@$PUBLIC_IP" \
  "id -u && whoami && test -r /root/.ssh/authorized_keys"

aws ec2 describe-instances \
  --region "$AWS_REGION" \
  --instance-ids "$INSTANCE_ID" \
  --query "Reservations[0].Instances[0].{Name:Tags[?Key=='Name']|[0].Value,InstanceId:InstanceId,KeyName:KeyName,State:State.Name,Type:InstanceType,PublicIp:PublicIpAddress}" \
  --output table
```

The EC2 verification returned:

```text
---------------------------------------
|          DescribeInstances          |
+-------------+-----------------------+
|  InstanceId |  i-099fe1065c5b19d86  |
|  KeyName    |  id_rsa               |
|  Name       |  devops-ec2           |
|  PublicIp   |  52.207.231.63        |
|  State      |  running              |
|  Type       |  t2.micro             |
+-------------+-----------------------+
```

The successful lab run also confirmed passwordless SSH access as `root` and the presence of the key in `/root/.ssh/authorized_keys`.

> **Why:** The first SSH command proves that the private key authenticates to the root account, `id -u` confirms the numeric root identity, `whoami` confirms the account name, and `test -r` confirms that root's authorized-key file is readable. `describe-instances` provides AWS-side evidence: the `--query` expression selects the Name tag, instance ID, key pair name, state, instance type, and public IP, and `--output table` formats those fields for comparison with the requirements.

## Best Practices

- **Restrict SSH to the landing host.** Allowing TCP port `22` only from `65.108.255.62/32` limits the attack surface to the lab's actual client address.
- **Protect the private key.** Keep `/root/.ssh/id_rsa` at mode `600` and never copy it to the EC2 instance.
- **Use key-only root access.** `PermitRootLogin prohibit-password` satisfies the lab's root-key requirement without enabling root password authentication.
- **Remove forced-command entries carefully.** Amazon Linux may use an authorized-key option to steer connections to `ec2-user`; replacing only the matching public-key entry preserves unrelated keys.
- **Validate before reloading SSH.** Running `sshd -t` before `systemctl reload` prevents applying a malformed SSH configuration.
- **Avoid broad ingress rules.** Do not replace the `/32` source with `0.0.0.0/0` unless the access requirement and compensating controls explicitly justify it.
- **Reuse named resources.** The key pair, security group, and instance are looked up before creation so the workflow does not create duplicates when rerun.

### 📚 Official Documentation

- [Calling AMI public parameters in Parameter Store](https://docs.aws.amazon.com/systems-manager/latest/userguide/parameter-store-public-parameters-ami.html)
- [import-key-pair — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/ec2/import-key-pair.html)
- [describe-vpcs — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/ec2/describe-vpcs.html)
- [describe-subnets — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/ec2/describe-subnets.html)
- [describe-security-groups — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/userguide/cli_ec2_code_examples.html)
- [create-security-group — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/ec2/create-security-group.html)
- [create-tags — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/ec2/create-tags.html)
- [authorize-security-group-ingress — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/ec2/authorize-security-group-ingress.html)
- [run-instances — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/ec2/run-instances.html)
- [wait instance-running — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/ec2/wait/instance-running.html)
- [describe-instances — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/ec2/describe-instances.html)
- [Connect to your Linux instance using SSH](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/connect-to-linux-instance.html)
