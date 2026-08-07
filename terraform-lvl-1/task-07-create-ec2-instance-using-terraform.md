# Task 07: Create EC2 Instance Using Terraform

The Nautilus DevOps team is strategizing the migration of a portion of their infrastructure to the AWS cloud. Recognizing the scale of this undertaking, they have opted to approach the migration in incremental steps rather than as a single, massive transition. To achieve this, they have segmented large tasks into smaller, more manageable units.

For this task, create an EC2 instance using Terraform with the following requirements:

The EC2 instance must use the value xfusion-ec2 as its Name tag, which defines the instance name in AWS.

Use the Amazon Linux ami-0c101f26f147fa7fd to launch this instance.

The Instance type must be t2.micro.

Create a new RSA key named xfusion-kp.

Attach the default (available by default) security group.

The Terraform working directory is /home/bob/terraform. Create the main.tf file (do not create a different .tf file) to provision the instance.

Note: Right-click under the EXPLORER section in VS Code and select Open in Integrated Terminal to launch the terminal.

## Task Requirements

1. The EC2 instance must use the value `xfusion-ec2` as its Name tag, which defines the instance name in AWS.
2. Use the Amazon Linux `ami-0c101f26f147fa7fd` to launch this instance.
3. The Instance type must be `t2.micro`.
4. Create a new RSA key named `xfusion-kp`.
5. Attach the default (available by default) security group.
6. The Terraform working directory is `/home/bob/terraform`. Create the `main.tf` file (do not create a different `.tf` file) to provision the instance.
7. Note: Right-click under the **EXPLORER** section in VS Code and select Open in Integrated Terminal to launch the terminal.

## Solution

The lab starts with a fresh Terraform workspace, so `main.tf` must be created manually for this task. The solution uses three resources: `tls_private_key` generates the RSA key material, `aws_key_pair` registers the public key in AWS, and `aws_instance` launches the EC2 instance. The existing `provider.tf` remains unchanged and supplies the AWS provider configuration for `us-east-1`.

### 📄 Step 1: Create main.tf in the fresh Terraform workspace

In the VS Code Explorer, open `/home/bob/terraform`. Create a new file named `main.tf` and add:

```hcl
resource "tls_private_key" "xfusion_kp" {
  algorithm = "RSA"
}

resource "aws_key_pair" "xfusion_kp" {
  key_name   = "xfusion-kp"
  public_key = tls_private_key.xfusion_kp.public_key_openssh
}

resource "aws_instance" "xfusion_ec2" {
  ami           = "ami-0c101f26f147fa7fd"
  instance_type = "t2.micro"
  key_name      = aws_key_pair.xfusion_kp.key_name

  security_groups = ["default"]

  tags = {
    Name = "xfusion-ec2"
  }
}
```

> **Why:** `tls_private_key` generates the RSA key material. `algorithm = "RSA"` selects the required key algorithm. `aws_key_pair` registers the generated public key in AWS; `key_name` sets the AWS key pair name and `public_key` supplies the generated OpenSSH public key. `aws_instance` launches the EC2 instance. `ami` selects the required Amazon Linux image, `instance_type` selects the requested hardware size, and `key_name` associates the new key pair. `security_groups = ["default"]` attaches the default security group by name. The `Name` tag gives the instance the requested AWS name. No provider block is added because `provider.tf` already exists.

### ⚙️ Step 2: Initialize Terraform

From the integrated terminal in `/home/bob/terraform`, run:

```bash
terraform init
```

Successful output includes:

```text
Terraform has been successfully initialized!
```

The lab installed the AWS provider version `5.91.0` and the TLS provider version `4.3.0`.

> **Why:** `terraform init` prepares the fresh working directory and installs the provider plugins required by the AWS instance, AWS key pair, and TLS private-key resources. It also creates or updates the dependency lock file.

### 🚀 Step 3: Apply the configuration

Run:

```bash
terraform apply --auto-approve
```

The lab returned:

```text
Plan: 3 to add, 0 to change, 0 to destroy.
tls_private_key.xfusion_kp: Creating...
tls_private_key.xfusion_kp: Creation complete after 0s [id=a0adceb746aa4c19cba9516d179fd8fd7c8cd93b]
aws_key_pair.xfusion_kp: Creating...
aws_key_pair.xfusion_kp: Creation complete after 1s [id=xfusion-kp]
aws_instance.xfusion_ec2: Creating...
aws_instance.xfusion_ec2: Still creating... [10s elapsed]
aws_instance.xfusion_ec2: Creation complete after 10s [id=i-e4b4eb2c847cbabe5]

Apply complete! Resources: 3 added, 0 changed, 0 destroyed.
```

> **Why:** `terraform apply` creates the resources described by `main.tf`. `--auto-approve` accepts the plan without an interactive prompt, which is suitable for this controlled lab. Terraform follows the resource references: it generates the RSA key first, registers the public key, and then launches the EC2 instance with that key pair.

### ✅ Step 4: Verify

The successful apply confirmed:

```text
Resource: aws_instance.xfusion_ec2
Instance ID: i-e4b4eb2c847cbabe5
AMI: ami-0c101f26f147fa7fd
Instance type: t2.micro
Key pair: xfusion-kp
Security group: default
Name tag: xfusion-ec2
Resources: 3 added, 0 changed, 0 destroyed
```

The EC2 instance was created successfully with the requested AMI, instance type, RSA key pair, default security group, and Name tag.

> **Why:** The Terraform plan and apply output confirm that the instance, AWS key pair, and RSA private-key resource were all created. The reported instance attributes match every requirement from the challenge.

## Best Practices

- **Treat each lab as independent.** Create a fresh `main.tf` in every new Terraform workspace instead of assuming files or state from a previous lab remain.
- **Use resource references for dependencies.** Referencing the generated public key and key pair name ensures Terraform creates resources in the correct order.
- **Use a purpose-built security group in real environments.** The default group satisfies this lab, but production workloads should use a least-privilege group with only the required ingress and egress rules.
- **Protect Terraform state.** The `tls_private_key` resource stores private key material in Terraform state; treat the state as sensitive and never commit it to source control.
- **Review the plan before applying in real environments.** The lab used `--auto-approve`, but production changes should normally be reviewed before approval.

### 📚 Official Documentation

- [AWS RunInstances example](https://docs.aws.amazon.com/code-library/latest/ug/ec2_example_ec2_RunInstances_section.html)
- [AWS provider `aws_instance` resource](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/instance)
- [AWS provider `aws_key_pair` resource](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/key_pair)
- [TLS provider `tls_private_key` resource](https://registry.terraform.io/providers/hashicorp/tls/latest/docs/resources/private_key)
- [Terraform `init` command](https://developer.hashicorp.com/terraform/cli/commands/init)
- [Terraform `apply` command](https://developer.hashicorp.com/terraform/cli/commands/apply)
