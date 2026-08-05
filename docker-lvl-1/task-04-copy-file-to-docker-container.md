# Task 04: Copy File to Docker Container

The Nautilus DevOps team possesses confidential data on App Server 3 in the Stratos Datacenter. A container named ubuntu_latest is running on the same server.

Copy an encrypted file /tmp/nautilus.txt.gpg from the docker host to the ubuntu_latest container located at /tmp/. Ensure the file is not modified during this operation.

## Task Requirements

1. Use App Server 3 as the Docker host.
2. Copy `/tmp/nautilus.txt.gpg` from the Docker host.
3. Copy the file into the `/tmp/` directory of the `ubuntu_latest` container.
4. Do not modify the encrypted file during the transfer.

## Solution

The file was copied directly from the Docker host into the existing `ubuntu_latest` container with `docker cp`. No text editor, decompression tool, or content-processing command was used, so the encrypted file remained unchanged.

### 🔌 Step 1: Connect to App Server 3

```bash
ssh banner@stapp03
```

The connection succeeded:

```text
thor@jump-host ~$ ssh banner@stapp03
The authenticity of host 'stapp03 (10.244.13.42)' can't be established.
ED25519 key fingerprint is SHA256:ImMyjbfw36NnIKpSmaxMAJ4aY/7PM5aHMXOtOMdlgGg.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added 'stapp03 (10.244.13.42)' (ED25519) to the list of known hosts.
banner@stapp03's password:
[banner@stapp03 ~]$
```

> **Why:** `ssh` opens a secure remote shell. `banner` is the Linux user used for App Server 3, and `stapp03` identifies the Docker host where both the source file and the target container are located.

### 📤 Step 2: Copy the encrypted file into the container

```bash
sudo docker cp /tmp/nautilus.txt.gpg ubuntu_latest:/tmp/
```

Docker reported a successful transfer:

```text
[banner@stapp03 ~]$ sudo docker cp /tmp/nautilus.txt.gpg ubuntu_latest:/tmp/

We trust you have received the usual lecture from the local System
Administrator. It usually boils down to these three things:

    #1) Respect the privacy of others.
    #2) Think before you type.
    #3) With great power comes great responsibility.

[sudo] password for banner:
Successfully copied 2.05kB to ubuntu_latest:/tmp/
[banner@stapp03 ~]$
```

> **Why:** `sudo` provides the privileges needed to access the Docker daemon. `docker cp` copies files between the Docker host and a container. `/tmp/nautilus.txt.gpg` is the source path on the host, `ubuntu_latest` identifies the target container, and `/tmp/` is the destination directory inside that container. Because the command performs a direct copy and no content-processing command was used, the encrypted file was not modified.

### ✅ Step 3: Confirm the lab result

The command returned the success message `Successfully copied 2.05kB to ubuntu_latest:/tmp/`, and the lab validator reported success. No additional command was run against the encrypted file, which avoided altering its contents.

## Best Practices

- **Run the copy on the correct Docker host.** Container names are local to a Docker host, so the command must run on App Server 3.
- **Use `docker cp` for direct file transfers.** It copies data between a host and a container without requiring an interactive shell inside the container.
- **Do not inspect or transform confidential encrypted files unnecessarily.** Avoid editors, decompression commands, or redirection when the requirement is to preserve the file exactly.
- **Use the complete destination path.** `ubuntu_latest:/tmp/` clearly identifies both the target container and the directory where the file must be placed.
- **Use `sudo` only when needed.** Administrative privileges may be required to access the Docker daemon or protected host files.

### 📚 Official Documentation

- [`docker container cp` reference](https://docs.docker.com/reference/cli/docker/container/cp/)
- [Docker Engine security](https://docs.docker.com/engine/security/)
