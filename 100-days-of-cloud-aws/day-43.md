# Day 43: Scaling and Managing Kubernetes Clusters with Amazon EKS

The Nautilus DevOps team has been tasked with preparing the infrastructure for a new Kubernetes-based application that will be deployed using Amazon EKS. The team is in the process of setting up an EKS cluster that meets their internal security and scalability standards. They require that the cluster be provisioned using the latest stable Kubernetes version to take advantage of new features and security improvements.

To minimize external exposure, the EKS cluster endpoint must be kept private. Additionally, the cluster needs to use the default VPC with availability zones a, b, and c to ensure high availability across different physical locations.

Your task is to create an EKS cluster named `datacenter-eks`, with Custom configuration, use IAM role for the cluster named `eksClusterRole`. Additionally, ensure that EKS Auto Mode is disabled and that the cluster endpoint access is set to private.

Finally, verify that the EKS cluster is successfully created with the correct configuration and is ready for workloads.

## Specific Requirements:

1. Provision the cluster using the latest stable Kubernetes version.
2. Keep the EKS cluster endpoint private.
3. Use the default VPC with availability zones `a`, `b`, and `c`.
4. Create an EKS cluster named `datacenter-eks` with Custom configuration.
5. Use an IAM role for the cluster named `eksClusterRole`.
6. Ensure that EKS Auto Mode is disabled and that the cluster endpoint access is set to private.
7. Verify that the EKS cluster is successfully created with the correct configuration and is ready for workloads.

## Solution

**Amazon EKS** (Elastic Kubernetes Service) is AWS's managed Kubernetes offering: AWS runs and patches the Kubernetes **control plane** (the brain of the cluster — API server, scheduler, etcd) for you, while you bring the networking and, later, the worker nodes. Two concepts drive this challenge:

- **EKS Auto Mode** is a newer EKS feature where AWS automatically provisions and manages compute, storage, and networking for your workloads. The challenge wants it **disabled**, meaning the "Custom configuration" path — you manage those pieces yourself instead of letting EKS do it automatically.
- **Cluster endpoint access** controls how you (or your tools) reach the Kubernetes API server. Setting it to **private-only** means the API server is reachable only from inside the VPC — nobody on the public internet can even attempt a connection, which is what "minimize external exposure" means here.

### 📦 Variables

Define all task variables in a single place before running any step.

```bash
AWS_REGION="us-east-1"
CLUSTER_NAME="datacenter-eks"
ROLE_NAME="eksClusterRole"
```

### 🔎 Step 1: Find the Latest Stable Kubernetes Version

```bash
aws eks describe-cluster-versions --region "$AWS_REGION" \
  --default-only \
  --cluster-type eks \
  --query "clusterVersions[0].clusterVersion" --output text
```

> **Why:** `aws eks describe-cluster-versions` lists the Kubernetes versions EKS offers. EKS supports several at once but always marks one as the **default** — the latest version AWS considers stable for new clusters. `--default-only` filters the list down to just that version, and `--cluster-type eks` scopes the query to standard EKS clusters (as opposed to other cluster types). The last two flags are **global AWS CLI options** you'll see on almost every command: `--query "clusterVersions[0].clusterVersion"` uses a JMESPath expression (client-side filtering built into the CLI) to pull a single field out of the full JSON response, and `--output text` prints it as plain text instead of JSON — together they let us capture the version straight into a shell variable instead of copy-pasting it.

```bash
K8S_VERSION=$(aws eks describe-cluster-versions --region "$AWS_REGION" \
  --default-only --cluster-type eks \
  --query "clusterVersions[0].clusterVersion" --output text)
```

### 🌐 Step 2: Resolve the Default VPC and Subnets in AZs a, b, and c

```bash
VPC_ID=$(aws ec2 describe-vpcs --region "$AWS_REGION" \
  --filters "Name=is-default,Values=true" \
  --query "Vpcs[0].VpcId" --output text)

SUBNET_IDS=$(aws ec2 describe-subnets --region "$AWS_REGION" \
  --filters "Name=vpc-id,Values=$VPC_ID" \
    "Name=availability-zone,Values=${AWS_REGION}a,${AWS_REGION}b,${AWS_REGION}c" \
  --query "Subnets[*].SubnetId" --output text | tr '\t' ',')

echo "Subnets: $SUBNET_IDS"
```

> **Why:** An EKS cluster's control plane needs to attach network interfaces inside your VPC, and it needs subnets in **at least two Availability Zones (AZs)** to do so — an AZ is an isolated physical data center location, and spreading across three of them (`a`, `b`, `c`) is what gives this cluster high availability: if one physical location has an outage, the others keep working. `aws ec2 describe-vpcs` and `aws ec2 describe-subnets` are read-only lookups; `--filters` narrows what they return using `Name=<field>,Values=<match>` pairs — here `is-default,true` finds the account's default VPC, and `availability-zone,us-east-1a,us-east-1b,us-east-1c` keeps only subnets in those three AZs. The `--query "Subnets[*].SubnetId"` pulls just the subnet IDs, and because `--output text` returns them separated by tabs, `tr '\t' ','` (a shell tool that swaps one character for another) converts them into the single comma-separated list that the next command expects.

### 🔑 Step 3: Find or Create the IAM Role for the Cluster

Before creating anything, let's find out whether the lab already provisioned the `eksClusterRole` for us. We ask IAM to describe it: if the role is there we reuse it, and if it isn't we create it in the next commands.

```bash
aws iam get-role --role-name $ROLE_NAME --query "Role.Arn" --output text
```

> **Why:** `aws iam get-role` is a read-only lookup that returns a role's details, or fails with `NoSuchEntity` if the role doesn't exist. Running it first is the "check before you create" habit: the KodeKloud lab sometimes pre-creates named resources, and creating a role that already exists would just error out. `--role-name` names the role we're asking about, and `--query "Role.Arn"` with `--output text` pulls only the role's **ARN** (Amazon Resource Name — the globally unique identifier for the role), which is the single value the cluster-creation command in Step 4 needs.

If the command above returns an ARN, the role already exists — skip ahead and capture it into `ROLE_ARN` (the last command in this step). If instead it fails with `NoSuchEntity`, the role isn't there yet, so create it with the following commands.

```bash
cat <<EOF > eks-cluster-trust-policy.json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "eks.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

aws iam create-role \
  --role-name $ROLE_NAME \
  --assume-role-policy-document file://eks-cluster-trust-policy.json

aws iam attach-role-policy \
  --role-name $ROLE_NAME \
  --policy-arn arn:aws:iam::aws:policy/AmazonEKSClusterPolicy

rm eks-cluster-trust-policy.json
```

> **Why:** The EKS control plane itself needs permission to call other AWS APIs on your behalf (for example, to manage network interfaces for the cluster). This is exactly what an **IAM role** is for: an identity that a service — here, `eks.amazonaws.com` — can "assume" to get temporary permissions, instead of using a long-lived access key. `aws iam create-role` creates that role: `--role-name` sets its name (`eksClusterRole`), and `--assume-role-policy-document` receives the **trust policy** — the JSON that states *who* is allowed to assume the role (the EKS service). We write that JSON to a file with a `cat <<EOF` here-document and pass it with the `file://` prefix, which tells the CLI "read this argument's value from a local file." `aws iam attach-role-policy` then grants actual permissions: `--policy-arn` points at the AWS-managed `AmazonEKSClusterPolicy`, which bundles exactly the permissions EKS needs to operate the cluster. Finally, whichever path you took, capture the role's ARN into `ROLE_ARN` with `aws iam get-role`, since the next command needs it.

```bash
ROLE_ARN=$(aws iam get-role --role-name $ROLE_NAME --query "Role.Arn" --output text)
```

### 🚀 Step 4: Create the EKS Cluster (Custom Config, Auto Mode Disabled, Private Endpoint)

```bash
aws eks create-cluster \
  --name $CLUSTER_NAME \
  --kubernetes-version "$K8S_VERSION" \
  --role-arn "$ROLE_ARN" \
  --resources-vpc-config subnetIds=$SUBNET_IDS,endpointPublicAccess=false,endpointPrivateAccess=true \
  --region $AWS_REGION
```

> **Why:** `aws eks create-cluster` is the single API call that ties everything together. `--name` gives the cluster its identifier (`datacenter-eks`), and `--region` tells the CLI which AWS Region to create it in (every command in this guide targets `us-east-1` so all resources land together). `--kubernetes-version` pins the cluster to the latest stable version found in Step 1. `--role-arn` is the cluster's IAM identity from Step 3. Inside `--resources-vpc-config` — a compact `key=value` structure — `subnetIds` places the control plane's network interfaces in our three AZs, `endpointPublicAccess=false` turns off any internet-facing access to the API server, and `endpointPrivateAccess=true` keeps it reachable from inside the VPC; together this is the **private-only endpoint** the challenge requires. There is no separate "Auto Mode" flag to set here: by not requesting Auto Mode compute (`--compute-config`), the cluster is created in the traditional, self-managed ("Custom configuration") mode, which is what "EKS Auto Mode is disabled" means in practice.

### ⏳ Step 5: Wait for the Cluster to Become Active

```bash
aws eks wait cluster-active --name $CLUSTER_NAME --region $AWS_REGION
```

> **Why:** Provisioning a Kubernetes control plane takes several minutes — EKS is standing up API servers, etcd, and networking behind the scenes. This command blocks until the cluster's status reaches `ACTIVE`, so the next verification step isn't run against a cluster that's still `CREATING`.

### ✅ Step 6: Verify

```bash
aws eks describe-cluster --name $CLUSTER_NAME --region $AWS_REGION \
  --query "cluster.{Status:status,Version:version,Endpoint:endpoint,PrivateAccess:resourcesVpcConfig.endpointPrivateAccess,PublicAccess:resourcesVpcConfig.endpointPublicAccess,Subnets:resourcesVpcConfig.subnetIds,RoleArn:roleArn}" \
  --output table
```

> **Why:** `aws eks describe-cluster` returns the full configuration of an existing cluster. Here `--query` uses a JMESPath **object projection** — the `{Alias:path,...}` syntax — to build a small custom result containing only the fields we care about (status, version, endpoint access flags, subnets, role), and `--output table` renders it as an easy-to-read ASCII table instead of raw JSON. This turns a long JSON blob into a quick visual check.

Confirm the output shows: `Status: ACTIVE`, `Version` matching the latest stable release, `PrivateAccess: true`, `PublicAccess: false`, the three subnets from Step 2, and the `eksClusterRole` ARN. This confirms the cluster is ready for workloads with the exact configuration the challenge requires.

## Best Practices

- **Spread control plane subnets across at least 3 AZs when possible.** More AZs means more resilience against a single data-center failure; EKS requires a minimum of two.
- **Prefer private-only (or private-and-public with restricted CIDRs) over fully public endpoints.** A private-only endpoint removes the API server from the public attack surface entirely; if you need any public access, always restrict it to known CIDR blocks.
- **Use AWS-managed policies like `AmazonEKSClusterPolicy` for the cluster role** instead of writing custom permissions, unless you have a specific reason to further restrict them — AWS keeps these policies updated as EKS evolves.
- **Track the "default" Kubernetes version programmatically** (`--default-only`) rather than hardcoding a version string, so your automation naturally picks up new stable releases.
- **Understand the Auto Mode trade-off.** Auto Mode reduces operational overhead by having AWS manage compute/storage/networking, but Custom configuration (Auto Mode disabled) gives you full control — required here for stricter internal security and scalability standards.

### 📚 Official Documentation

- [Amazon EKS cluster IAM role](https://docs.aws.amazon.com/eks/latest/userguide/cluster-iam-role.html)
- [create-cluster — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/eks/create-cluster.html)
- [Disable EKS Auto Mode](https://docs.aws.amazon.com/eks/latest/userguide/auto-disable.html)
- [VPC and Subnet Considerations (EKS Best Practices)](https://docs.aws.amazon.com/eks/latest/best-practices/subnets.html)
- [describe-cluster-versions — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/v1/reference/eks/describe-cluster-versions.html)
