# Day 39: Create a Docker Image From Container

One of the Nautilus developer was working to test new changes on a container. He wants to keep a backup of his changes to the container. A new request has been raised for the DevOps team to create a new image from this container. Below are more details about it:

a. Create an image `beta:datacenter` on `Application Server 1` from a container `ubuntu_latest` that is running on same server.

## Specific Requirements:

a. Create an image `beta:datacenter` on `Application Server 1` from a container `ubuntu_latest` that is running on same server.

## Solution

The `ubuntu_latest` container was running on App Server 1. Docker committed its current filesystem state into a new local image named `beta:datacenter`, preserving the container's changes as an image that can be used later to create new containers.

### 🔐 Step 1: Connect to App Server 1

```bash
ssh tony@stapp01
```

> **Why:** `ssh` opens a remote shell on App Server 1 as `tony`, where the running source container is available.

### 📋 Step 2: Confirm the source container is running

```bash
docker ps
```

The source container was running:

```text
CONTAINER ID   IMAGE     COMMAND       CREATED          STATUS          PORTS     NAMES
d4e0c7f9bf1f   ubuntu    "/bin/bash"   20 minutes ago   Up 20 minutes             ubuntu_latest
```

> **Why:** `docker ps` lists running containers. The output confirms that `ubuntu_latest` exists and is active before Docker captures its current state into a new image.

### 📸 Step 3: Create an image from the container

```bash
docker commit ubuntu_latest beta:datacenter
```

Docker created the image with this digest:

```text
sha256:5482b53d0e6f8d45cf39fc808dd9bcbdc5b8c4b94e1a9ee95708fa5a1cf36218
```

> **Why:** `docker commit` creates a new local image from a container's current filesystem changes. `ubuntu_latest` is the source container. `beta:datacenter` assigns the new image the repository name `beta` and the tag `datacenter`. The operation creates the backup image without stopping or altering the source container.

### ✅ Step 4: Verify the backup image

```bash
docker images beta
```

The image was listed as follows:

```text
REPOSITORY   TAG          IMAGE ID       CREATED          SIZE
beta         datacenter   5482b53d0e6f   10 seconds ago   143MB
```

> **Why:** `docker images` lists locally available images, and `beta` limits the output to the requested repository. The `datacenter` tag and image ID confirm that Docker created the required backup image.

## Best Practices

- **Use `docker commit` for short-lived troubleshooting snapshots.** It is useful for preserving an interactive container's state during tests or investigation.
- **Prefer reproducible image builds for production.** A Dockerfile records image changes as version-controlled instructions, whereas `docker commit` captures state without documenting how it was produced.
- **Apply explicit repository and tag names.** `beta:datacenter` clearly identifies the image's purpose and makes it easier to reference later.
- **Verify the resulting image.** List the local image after committing to confirm its repository, tag, and image ID.

### 📚 Official Documentation

- [Docker commit reference](https://docs.docker.com/reference/cli/docker/container/commit/)
- [Docker ps reference](https://docs.docker.com/reference/cli/docker/container/ls/)
- [Docker images reference](https://docs.docker.com/reference/cli/docker/image/ls/)
