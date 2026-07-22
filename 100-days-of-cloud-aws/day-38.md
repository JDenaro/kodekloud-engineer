# Day 38: Deploying Containerized Applications with Amazon ECS

The Nautilus DevOps team needs to deploy a containerized application on AWS. The image is stored in a private **Amazon Elastic Container Registry (ECR)** repository and runs on **Amazon Elastic Container Service (ECS)** using the **Fargate** launch type, publicly reachable over HTTP (port 80).

## Task Requirements

1. Create an IAM execution role (`ecsTaskExecutionRole`) that ECS tasks can assume.
2. Create a private ECR repository and push the application Docker image to it.
3. Create an ECS cluster and register a Fargate task definition.
4. Open port 80 (HTTP) on the security group so the container is reachable.
5. Deploy an ECS service that keeps the task running.
6. Retrieve the public IP and verify the application is accessible.

## Solution

### 📦 Variables

Define all task variables in a single place before running any step. The challenge doesn't prescribe specific names for the repository, cluster, task definition, container, or service, so we pick clear `nautilus-*` names here; adjust them to match whatever names your task assigns.

```bash
AWS_REGION="us-east-1"
AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
ECR_REPO_NAME="nautilus-app"
CLUSTER_NAME="nautilus-cluster"
TASK_DEF_NAME="nautilus-task"
CONTAINER_NAME="nautilus-container"
SERVICE_NAME="nautilus-service"
DOCKERFILE_DIR="/root/nautilus-app"
ECR_URL="${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
IMAGE_URI="${ECR_URL}/${ECR_REPO_NAME}:latest"
EXEC_ROLE_ARN="arn:aws:iam::${AWS_ACCOUNT_ID}:role/ecsTaskExecutionRole"
```

> **Why:** We collect every value in one place so the later commands stay short and consistent. `aws sts get-caller-identity` asks AWS "who am I?" and returns details about the identity making the call — we use it here to grab your 12-digit **AWS account ID**, which is part of the ECR image address. Two flags on that command appear again and again throughout this guide: `--query` filters the JSON response using a JMESPath expression (here `Account` picks just the account number), and `--output text` strips the JSON formatting so the result is a plain string you can store in a shell variable. Keep these two in mind — every `describe-*`/`get-*` command below uses the same `--query`/`--output` pattern to extract one value.

### 🛠️ Step 1: Configure the IAM Execution Role

Fargate needs a role named `ecsTaskExecutionRole` to pull images from ECR and send logs to CloudWatch. The trust policy must allow the ECS tasks service to assume the role.

```bash
cat <<EOF > trust-policy.json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ecs-tasks.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

aws iam get-role --role-name ecsTaskExecutionRole
```

No role named `ecsTaskExecutionRole` existed yet in this account, so it has to be created:

```bash
aws iam create-role --role-name ecsTaskExecutionRole --assume-role-policy-document file://trust-policy.json

aws iam attach-role-policy \
    --role-name ecsTaskExecutionRole \
    --policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy

rm trust-policy.json
```

> **Why:** In AWS, a service can't act on your behalf unless you give it permission through an **IAM role** — a set of permissions a service can temporarily "assume." Here, ECS needs to do things *before* your app runs: download (pull) the container image and write logs. The **trust policy** (the JSON we write to `trust-policy.json`) is what says "the ECS tasks service is allowed to assume this role." `aws iam get-role` checks whether the role already exists, so we don't try to create a duplicate.
>
> Walking through the commands: `aws iam create-role` makes the new role — `--role-name` gives it its name and `--assume-role-policy-document` attaches the trust policy, where `file://` tells the CLI to read the JSON from a local file rather than passing it inline. Finally, `aws iam attach-role-policy` links a permissions policy to the role; `--policy-arn` points to `AmazonECSTaskExecutionRolePolicy`, a ready-made policy maintained by AWS that grants exactly those startup permissions — nothing more. Using the managed policy instead of writing your own follows the principle of **least privilege**: give only the access that's needed.

### 🐳 Step 2: Create the ECR Repository and Push the Image

Create the private repository, authenticate Docker with AWS, then build, tag, and push the image.

```bash
aws ecr create-repository --repository-name $ECR_REPO_NAME --region $AWS_REGION

aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $ECR_URL

cd $DOCKERFILE_DIR
docker build -t $ECR_REPO_NAME:latest .
docker tag $ECR_REPO_NAME:latest $IMAGE_URI
docker push $IMAGE_URI
```

> **Why:** **Amazon ECR** is AWS's private Docker image registry — a secure place to store container images that ECS can pull from. `aws ecr create-repository` makes an empty private repository; `--repository-name` names it and `--region` places it in your region. Because the repository is private, Docker must prove who it is before pushing: `aws ecr get-login-password` retrieves a temporary authentication token, and piping it into `docker login --username AWS --password-stdin <registry-url>` logs Docker in — `--password-stdin` reads the token from the pipe instead of typing it on the command line (safer, since it won't show up in your shell history). The remaining Docker commands are the standard build-and-publish cycle: `docker build -t` builds the image from the `Dockerfile` and tags it with a local name, `docker tag` re-tags that image with the full ECR address (`IMAGE_URI`) so Docker knows where it belongs, and `docker push` uploads it to the repository.

### 🏗️ Step 3: Create the ECS Cluster and Task Definition

The cluster is the logical environment; the task definition is the blueprint telling Fargate which resources and container to run.

```bash
aws ecs create-cluster --region "$AWS_REGION" --cluster-name $CLUSTER_NAME

cat <<EOF > task-definition.json
{
  "family": "${TASK_DEF_NAME}",
  "networkMode": "awsvpc",
  "containerDefinitions": [
    {
      "name": "${CONTAINER_NAME}",
      "image": "${IMAGE_URI}",
      "essential": true,
      "portMappings": [
        {
          "containerPort": 80,
          "hostPort": 80,
          "protocol": "tcp"
        }
      ]
    }
  ],
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512",
  "executionRoleArn": "${EXEC_ROLE_ARN}"
}
EOF

aws ecs register-task-definition --region "$AWS_REGION" --cli-input-json file://task-definition.json
rm task-definition.json
```

> **Why:** An **ECS cluster** is just a logical grouping that your tasks and services run inside; `aws ecs create-cluster` creates it and `--cluster-name` names it. A **task definition** is like a recipe that tells ECS how to run your container: which image to use, how much CPU/memory, which ports to open, and which IAM role to use. We write that recipe as JSON and register it with `aws ecs register-task-definition`, where `--cli-input-json file://task-definition.json` feeds the whole configuration in from the file we just created (`file://` reads it from disk). Inside the JSON, **Fargate** is requested via `requiresCompatibilities: ["FARGATE"]` — Fargate is the "serverless" way to run containers, where you don't manage any EC2 servers yourself, AWS runs them for you. Fargate requires the `awsvpc` network mode, which gives each task its own network interface (an **ENI**, like a virtual network card) and its own private IP inside the VPC — so the container behaves like its own little machine on the network. `cpu`/`memory` size the task, `portMappings` opens container port 80, and `executionRoleArn` links back to the role from Step 1 so the task is allowed to pull the image and write logs.

### 🌐 Step 4: Configure Networking and Security (Port 80)

To make the container reachable over the web, open the HTTP port (80) on the default VPC's security group.

```bash
VPC_ID=$(aws ec2 describe-vpcs --region "$AWS_REGION" --filters "Name=is-default,Values=true" --query "Vpcs[0].VpcId" --output text)

SG_ID=$(aws ec2 describe-security-groups --region "$AWS_REGION" --filters "Name=vpc-id,Values=$VPC_ID" "Name=group-name,Values=default" --query "SecurityGroups[0].GroupId" --output text)

aws ec2 authorize-security-group-ingress --region "$AWS_REGION" \
    --group-id "$SG_ID" \
    --protocol tcp \
    --port 80 \
    --cidr 0.0.0.0/0
```

> **Why:** A **security group** is a virtual firewall that controls which traffic can reach your container. First we find the right one: `aws ec2 describe-vpcs` locates the account's **default VPC** (your private network in AWS) using `--filters "Name=is-default,Values=true"` — the lab account always has exactly one flagged default; then `aws ec2 describe-security-groups` finds that VPC's `default` security group by filtering on `vpc-id` and `group-name`. Once we have its ID, `aws ec2 authorize-security-group-ingress` adds an **inbound rule** that lets traffic in: `--group-id` picks the security group, `--protocol tcp` and `--port 80` allow HTTP web traffic, and `--cidr 0.0.0.0/0` means "from any IP address on the internet." Without this rule the security group would block the incoming connection by default.

### 🚀 Step 5: Deploy the ECS Service

Create the ECS service that keeps the task running in the cluster using the detected network.

```bash
SUBNET_IDS=$(aws ec2 describe-subnets --region "$AWS_REGION" --filters "Name=vpc-id,Values=$VPC_ID" --query "Subnets[*].SubnetId" --output text | sed 's/	/,/g')

aws ecs create-service --region "$AWS_REGION" \
    --cluster $CLUSTER_NAME \
    --service-name $SERVICE_NAME \
    --task-definition $TASK_DEF_NAME \
    --desired-count 1 \
    --launch-type FARGATE \
    --network-configuration "awsvpcConfiguration={subnets=[$SUBNET_IDS],securityGroups=[$SG_ID],assignPublicIp=ENABLED}"
```

> **Why:** An **ECS service** keeps a desired number of task copies running and restarts them if they stop — it's what turns a one-off task into a managed, always-on deployment. First `aws ec2 describe-subnets` lists the subnets in our VPC (a **subnet** is a slice of the VPC's network in one Availability Zone); `--output text` returns them tab-separated, so `sed 's/\t/,/g'` rewrites the tabs as commas to build the list the next command needs. Then `aws ecs create-service` launches the service: `--cluster` and `--service-name` say where it runs and what it's called, `--task-definition` points to the recipe from Step 3, `--desired-count 1` asks for one running copy, and `--launch-type FARGATE` runs it serverless. `--network-configuration` places the task on the network — inside `awsvpcConfiguration`, `subnets` and `securityGroups` attach the subnets and firewall we found in Step 4, and `assignPublicIp=ENABLED` gives the task a public IP so it's reachable from the internet (required for Fargate tasks in public subnets that need inbound access).

### ✅ Step 6: Verify

The service runs a long-lived web server. Wait for the service to stabilize, find the public IP that Fargate assigned to the task's network interface, then confirm the application answers on port 80.

```bash
aws ecs wait services-stable --region "$AWS_REGION" --cluster $CLUSTER_NAME --services $SERVICE_NAME
```

> **Why:** `aws ecs wait services-stable` **blocks** until the service reaches a steady state — the desired number of tasks are running and any deployment has settled — so the CLI polls ECS for you instead of you refreshing the console. Running the check now avoids reading a task that is still starting up and has no IP yet.

```bash
TASK_ARN=$(aws ecs list-tasks --region "$AWS_REGION" --cluster $CLUSTER_NAME --service-name $SERVICE_NAME --query "taskArns[0]" --output text)
echo $TASK_ARN
```

```
arn:aws:ecs:us-east-1:123456789012:task/nautilus-cluster/8f0e2a1b4c5d4e6f9a0b1c2d3e4f5a6b
```

> **Why:** `aws ecs list-tasks` returns the ARNs (Amazon Resource Names — unique identifiers) of the tasks in the cluster; `--service-name` narrows the list to just the tasks our service launched, and `--query "taskArns[0]"` picks the first one. We store it in `TASK_ARN` because the next command needs it to look up the task's networking details.

```bash
ENI_ID=$(aws ecs describe-tasks --region "$AWS_REGION" --cluster $CLUSTER_NAME --tasks "$TASK_ARN" \
  --query "tasks[0].attachments[0].details[?name=='networkInterfaceId'].value | [0]" \
  --output text)
echo $ENI_ID
```

```
eni-0a1b2c3d4e5f6a7b8
```

> **Why:** `aws ecs describe-tasks` returns the full details of a task, including its network attachments. Because each `awsvpc` task gets its own **elastic network interface (ENI)** — a virtual network card — we dig into `attachments[0].details` and use the JMESPath filter `[?name=='networkInterfaceId']` to select the entry whose name is `networkInterfaceId`, then `.value | [0]` to pull out that single string. That ENI ID is the bridge to the public IP, which lives on the network-interface side, not on the task record itself.

```bash
PUBLIC_IP=$(aws ec2 describe-network-interfaces --region "$AWS_REGION" --network-interface-ids "$ENI_ID" \
  --query "NetworkInterfaces[0].Association.PublicIp" \
  --output text)
echo "http://$PUBLIC_IP"
```

```
http://54.210.13.77
```

> **Why:** `aws ec2 describe-network-interfaces` returns details about an ENI; `--network-interface-ids` tells it which one to describe. When `assignPublicIp=ENABLED` (Step 5), Fargate attaches a public IPv4 address to the ENI, exposed under `Association.PublicIp` — the `--query` extracts exactly that value. This is the address the internet uses to reach the container.

```bash
curl -I "http://$PUBLIC_IP"
```

```
HTTP/1.1 200 OK
Server: nginx
Content-Type: text/html
```

> **Why:** `curl` is a command-line HTTP client; `-I` sends a lightweight `HEAD` request that asks only for the response headers instead of downloading the whole page. A `200 OK` status line confirms the full path works end to end: the security-group rule from Step 4 lets the request in, the task is running, and the container is serving HTTP on port 80.

A `200 OK` response means the containerized application is deployed and publicly reachable.

## Best Practices

- **Separate task execution role from task role.** The **task execution role** (`ecsTaskExecutionRole`) grants permissions ECS needs *before* the task starts (pull image, write logs). The **task role** grants permissions the *application* needs at runtime (e.g. S3, DynamoDB). Keep them distinct and scoped to least privilege.
- **One task definition family and IAM role per service.** This limits how much each component can access in your account.
- **Serve production traffic through a load balancer.** Instead of a public task IP (which changes on every deployment), front the service with an Application Load Balancer for a stable endpoint, health checks, and TLS.
- **Enable logging.** Add a `logConfiguration` (awslogs driver) to the container definition so logs flow to CloudWatch — the execution role already permits this.

### 📚 Official Documentation

- [Best practices for IAM roles in Amazon ECS](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/security-iam-roles.html)
- [Amazon ECS task execution IAM role](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task_execution_IAM_role.html)
- [Use load balancing to distribute Amazon ECS service traffic](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/service-load-balancing.html)
- [Learn how to create an Amazon ECS Linux task for Fargate](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/getting-started-fargate.html)
