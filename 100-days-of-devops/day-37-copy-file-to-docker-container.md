# Day 37: Copy File to Docker Container

The Nautilus DevOps team possesses confidential data on `App Server 2` in the `Stratos Datacenter`. A container named `ubuntu_latest` is running on the same server.

Copy an encrypted file `/tmp/nautilus.txt.gpg` from the docker host to the `ubuntu_latest` container located at `/home/`. Ensure the file is not modified during this operation.

## Specific Requirements:

1. Copy an encrypted file `/tmp/nautilus.txt.gpg` from the docker host to the `ubuntu_latest` container located at `/home/`.
2. Ensure the file is not modified during this operation.

## Solution

The destination container was already running on App Server 2. The file was copied with `docker cp` and its SHA-256 checksum was compared before and after the transfer. The matching hashes confirmed that the encrypted file reached the container without modification.

### 🔐 Step 1: Connect to App Server 2

```bash
ssh steve@stapp02
```

> **Why:** `ssh` opens a remote shell on App Server 2 as `steve`, where both the source file and the running Docker container are available.

### 📋 Step 2: Confirm the destination container

```bash
docker ps
```

The running container was listed as follows:

```text
CONTAINER ID   IMAGE     COMMAND       CREATED         STATUS         PORTS     NAMES
231a1c4ea7c0   ubuntu    "/bin/bash"   3 minutes ago   Up 3 minutes             ubuntu_latest
```

> **Why:** `docker ps` lists the containers that are currently running. The output confirms that `ubuntu_latest` is available as the destination before the file is copied.

### 🔒 Step 3: Calculate the source file checksum

```bash
sha256sum /tmp/nautilus.txt.gpg
```

The source checksum was:

```text
f231f021af07b2bc36c86b3880df0c6a6856c1b37971c3484511e7a026044c7a  /tmp/nautilus.txt.gpg
```

> **Why:** `sha256sum` calculates a SHA-256 cryptographic hash for a file. This hash acts as a fingerprint: if the file contents change, its calculated value changes. Recording it before the copy establishes the value used to verify the encrypted file's integrity.

### 📥 Step 4: Copy the encrypted file into the container

```bash
docker cp /tmp/nautilus.txt.gpg ubuntu_latest:/home/
```

Docker reported:

```text
Successfully copied 2.05kB to ubuntu_latest:/home/
```

> **Why:** `docker cp` transfers files between the Docker host and a container without opening an interactive shell in the container. The first path is the source on the Docker host. `ubuntu_latest:/home/` identifies the destination container and its `/home/` directory. Because the destination is a directory, Docker preserves the source filename and creates `/home/nautilus.txt.gpg`.

### ✅ Step 5: Verify the copied file's integrity

```bash
docker exec ubuntu_latest sha256sum /home/nautilus.txt.gpg
```

The container returned:

```text
f231f021af07b2bc36c86b3880df0c6a6856c1b37971c3484511e7a026044c7a  /home/nautilus.txt.gpg
```

> **Why:** `docker exec` runs a command in an already running container. Here it runs `sha256sum` on the copied file. Its SHA-256 hash exactly matches the source hash, confirming that `/home/nautilus.txt.gpg` was copied intact and was not modified during the operation.

## Best Practices

- **Verify sensitive-file integrity.** Compare a cryptographic checksum before and after transferring confidential or encrypted files.
- **Use explicit container and destination paths.** `ubuntu_latest:/home/` makes both the target container and final directory unambiguous.
- **Avoid unnecessary file processing.** Use `docker cp` directly instead of opening, decrypting, or rewriting an encrypted file during its transfer.
- **Confirm the final location.** Check the file inside the container, not only the success message from the host-side copy command.

### 📚 Official Documentation

- [Docker cp reference](https://docs.docker.com/reference/cli/docker/container/cp/)
- [Docker exec reference](https://docs.docker.com/reference/cli/docker/container/exec/)
- [GNU Coreutils: sha2 utilities](https://www.gnu.org/software/coreutils/manual/html_node/sha2-utilities.html)
