# Day 28: Creating a Private ECR Repository

The Nautilus DevOps team has been tasked with setting up a containerized application. They need to create a private Amazon Elastic Container Registry (ECR) repository to store their Docker images. Once the repository is created, they will build a Docker image from a Dockerfile located on the `aws-client` host and push this image to the ECR repository. This process is essential for maintaining and deploying containerized applications in a streamlined manner.

Create a private ECR repository named `devops-ecr`. There is a Dockerfile under `/root/pyapp` directory on `aws-client` host, build a docker image using this Dockerfile and push the same to the newly created ECR repo, the image tag must be latest.

## Specific Requirements:

1. Create a private ECR repository named `devops-ecr`.
2. There is a Dockerfile under `/root/pyapp` directory on `aws-client` host.
3. Build a Docker image using this Dockerfile.
4. Push the image to the newly created ECR repository.
5. The image tag must be `latest`.

## Solution

The Docker image is built locally from `/root/pyapp/Dockerfile`, tagged with the ECR repository URI, and pushed to the private ECR registry. ECR returned the image digest `sha256:7a5b4cc8277dc199e4d105913a04536d55fcf70b863cedad27ebce4a8f901516`, which verifies that the `latest` tag reached the repository.

### 📦 Variables

```bash
AWS_REGION="us-east-1"
REPOSITORY_NAME="devops-ecr"
IMAGE_TAG="latest"
BUILD_CONTEXT="/root/pyapp"
DOCKERFILE_PATH="/root/pyapp/Dockerfile"
```

### 🔍 Step 1: Validate the Docker build context

```bash
test -d /root/pyapp
test -f /root/pyapp/Dockerfile
docker info
```

The Dockerfile was found:

```text
Dockerfile found: /root/pyapp/Dockerfile
```

> **Why:** `test -d` confirms that `/root/pyapp` is a directory, while `test -f` confirms that the required Dockerfile exists as a regular file. `docker info` checks that the Docker client can communicate with the Docker daemon before the build starts.

### 🔎 Step 2: Check for and create the private ECR repository

Check whether the repository already exists:

```bash
aws ecr describe-repositories \
  --repository-names devops-ecr \
  --query "repositories[0].repositoryUri" \
  --output text
```

The repository did not exist, so it was created:

```bash
aws ecr create-repository \
  --repository-name devops-ecr \
  --query "repository.repositoryUri" \
  --output text
```

The private repository URI was:

```text
818788518694.dkr.ecr.us-east-1.amazonaws.com/devops-ecr
```

> **Why:** `describe-repositories` looks up ECR repositories by name before creating anything. `--repository-names` identifies the repository to search. `create-repository` creates a private ECR repository; `--repository-name` assigns its name. `--query` extracts the repository URI, and `--output text` returns a clean value for later Docker commands.

### 🔐 Step 3: Authenticate Docker to the ECR registry

```bash
aws ecr get-login-password --region us-east-1 | docker login \
  --username AWS \
  --password-stdin 818788518694.dkr.ecr.us-east-1.amazonaws.com
```

Authentication succeeded:

```text
Login Succeeded
Authenticated Docker to 818788518694.dkr.ecr.us-east-1.amazonaws.com.
```

Docker also displayed this warning:

```text
WARNING! Your credentials are stored unencrypted in '/root/.docker/config.json'.
Configure a credential helper to remove this warning.
```

This warning did not affect the lab, but a persistent host should use a Docker credential helper. The password was passed through standard input and was not printed or embedded in the command line.

> **Why:** `get-login-password` retrieves an ECR authentication password. `--region` must match the registry region. The pipe sends the password directly to `docker login`; `--username AWS` selects the ECR login user, and `--password-stdin` avoids exposing the password in shell history or process arguments. The registry hostname is the account-level portion of the repository URI.

### 🏗️ Step 4: Build and tag the Docker image

Build the image from the Dockerfile:

```bash
docker build \
  --file /root/pyapp/Dockerfile \
  --tag devops-ecr:latest \
  /root/pyapp
```

The build completed successfully in `187.3s` and produced image ID:

```text
sha256:765a401cc9fb353baa91fbfb0aaa1e4c9a126d355f8e0d
```

The Dockerfile used `python:3.8-slim` as its base image, copied the application into `/app`, and installed the requirements successfully.

Tag the local image with the full ECR repository URI:

```bash
docker tag \
  devops-ecr:latest \
  818788518694.dkr.ecr.us-east-1.amazonaws.com/devops-ecr:latest
```

The image was ready for ECR:

```text
Built and tagged image: 818788518694.dkr.ecr.us-east-1.amazonaws.com/devops-ecr:latest
```

> **Why:** `docker build` creates an image from the Dockerfile and build context. `--file` selects the Dockerfile, `--tag` gives the local image its name and `latest` tag, and `/root/pyapp` supplies the files available to the build. `docker tag` adds a second name to the same local image; the full ECR URI tells Docker which registry and repository will receive it.

### 📤 Step 5: Push the image to ECR

```bash
docker push 818788518694.dkr.ecr.us-east-1.amazonaws.com/devops-ecr:latest
```

The image layers were uploaded and ECR returned this digest:

```text
latest: digest: sha256:7a5b4cc8277dc199e4d105913a04536d55fcf70b863cedad27ebce4a8f901516 size: 1783
Pushed image: 818788518694.dkr.ecr.us-east-1.amazonaws.com/devops-ecr:latest
```

> **Why:** `docker push` uploads the locally tagged image to ECR. The repository URI and `:latest` tag identify the destination. The returned digest is content-addressed, so it provides a stable verification value even though the `latest` tag can later move to another image.

### ✅ Step 6: Verify

Verify the ECR repository configuration:

```bash
aws ecr describe-repositories \
  --repository-names devops-ecr \
  --query "repositories[0].{RepositoryName:repositoryName,RepositoryUri:repositoryUri,ImageTagMutability:imageTagMutability,EncryptionType:encryptionConfiguration.encryptionType}" \
  --output table
```

```text
-----------------------------------------------------------------------------------
|                              DescribeRepositories                               |
+---------------------+-----------------------------------------------------------+
|  EncryptionType     |  AES256                                                   |
|  ImageTagMutability |  MUTABLE                                                  |
|  RepositoryName     |  devops-ecr                                               |
|  RepositoryUri      |  818788518694.dkr.ecr.us-east-1.amazonaws.com/devops-ecr  |
+---------------------+-----------------------------------------------------------+
```

Verify the pushed image by its required tag:

```bash
aws ecr describe-images \
  --repository-name devops-ecr \
  --image-ids "imageTag=latest" \
  --query "imageDetails[0].{RepositoryName:repositoryName,ImageTag:imageTags[0],ImageDigest:imageDigest,ImageSizeInBytes:imageSizeInBytes,ImagePushedAt:imagePushedAt,ScanStatus:imageScanStatus.status}" \
  --output table
```

```text
-------------------------------------------------------------------------------------------------
|                                        DescribeImages                                         |
+------------------+----------------------------------------------------------------------------+
|  ImageDigest     |  sha256:7a5b4cc8277dc199e4d105913a04536d55fcf70b863cedad27ebce4a8f901516   |
|  ImagePushedAt   |  1784647457.173                                                            |
|  ImageSizeInBytes|  49724249                                                                  |
|  ImageTag        |  latest                                                                    |
|  RepositoryName  |  devops-ecr                                                                |
|  ScanStatus      |  None                                                                      |
+------------------+----------------------------------------------------------------------------+
```

The repository is private, the image tag is `latest`, and the digest confirms that the image was pushed successfully. `ScanStatus` is `None` because scanning was not enabled for this lab repository; this does not indicate a failed push.

> **Why:** `describe-repositories` verifies the repository URI, encryption type, and tag mutability. `describe-images` retrieves image metadata; `--repository-name` selects `devops-ecr`, `--image-ids imageTag=latest` selects the required tag, `--query` projects the digest and evidence fields, and `--output table` formats the result for verification.

## Best Practices

- **Verify by digest.** Tags such as `latest` can move, while the image digest identifies the exact content stored in ECR.
- **Use a credential helper.** Docker warned that credentials were stored unencrypted in `/root/.docker/config.json`; configure a secure credential store on persistent hosts.
- **Avoid exposing registry passwords.** Pipe `get-login-password` to `docker login --password-stdin` instead of placing credentials in command arguments or scripts.
- **Enable image scanning.** This challenge did not require scanning, but production repositories should enable scan-on-push and review findings before deployment.
- **Consider immutable tags.** The lab requires `latest`, but immutable release tags prevent accidental overwrites in production workflows.
- **Keep build context minimal.** A focused `.dockerignore` reduces transfer time and prevents unrelated files or secrets from entering the image build context.

### 📚 Official Documentation

- [create-repository — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/ecr/create-repository.html)
- [describe-repositories — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/ecr/describe-repositories.html)
- [get-login-password — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/ecr/get-login-password.html)
- [describe-images — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/ecr/describe-images.html)
- [Creating a container image for use on Amazon ECS](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/create-container-image.html)
- [docker build](https://docs.docker.com/reference/cli/docker/build/)
- [docker tag](https://docs.docker.com/reference/cli/docker/tag/)
- [docker push](https://docs.docker.com/reference/cli/docker/push/)
- [docker login](https://docs.docker.com/reference/cli/docker/login/)
- [Docker credential stores](https://docs.docker.com/go/credential-store/)
