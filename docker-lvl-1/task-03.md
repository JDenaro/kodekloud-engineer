# Task 03: Delete Docker Container

A container named kke-container was created by one of the Nautilus project developers on App Server 1. It was solely for testing purposes and now requires deletion. Execute the following task:

Delete the kke-container on App Server 1 in Stratos DC.

## Task Requirements

1. Delete the container named `kke-container`.
2. Perform the deletion on App Server 1.

## Solution

The testing container was running on App Server 1, so it was removed with `docker rm -f`. The `-f` option stops the container if necessary and then removes it in one command.

### 🔌 Step 1: Connect to App Server 1

```bash
ssh tony@stapp01
```

The connection succeeded:

```text
thor@jump-host ~$ ssh tony@stapp01
The authenticity of host 'stapp01 (10.244.13.25)' can't be established.
ED25519 key fingerprint is SHA256:PMyUhIH8cxFteejrTc+r5mwkxu4tOWoOzxSv94yB24A.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added 'stapp01 (10.244.13.25)' (ED25519) to the list of known hosts.
tony@stapp01's password:
[tony@stapp01 ~]$
```

> **Why:** `ssh` opens a secure remote shell. `tony` is the Linux user used for App Server 1, and `stapp01` identifies the server where the container exists. Docker commands must be run on the host that owns the container.

### 🔎 Step 2: Confirm the target container

```bash
docker ps
```

The target container was running:

```text
[tony@stapp01 ~]$ docker ps
CONTAINER ID   IMAGE     COMMAND               CREATED              STATUS          PORTS     NAMES
ed65fbb3646e   busybox   "tail -f /dev/null"   About a minute ago   Up 58 seconds             kke-container
[tony@stapp01 ~]$
```

> **Why:** `docker ps` lists running containers. The `NAMES` column confirmed that the container to remove was `kke-container`, and the `STATUS` column showed that it was currently running.

### 🗑️ Step 3: Remove the container

```bash
sudo docker rm -f kke-container
```

The container was removed successfully after the correct `sudo` password was entered:

```text
[tony@stapp01 ~]$ sudo docker rm -f kke-container

We trust you have received the usual lecture from the local System
Administrator. It usually boils down to these three things:

    #1) Respect the privacy of others.
    #2) Think before you type.
    #3) With great power comes great responsibility.

[sudo] password for tony:

Sorry, try again.
[sudo] password for tony:
Sorry, try again.
[sudo] password for tony:
kke-container
[tony@stapp01 ~]$
```

> **Why:** `sudo` runs the Docker command with administrative privileges. `docker rm` removes a container, and `-f` forcibly stops a running container before removing it. The command prints the removed container name when the operation succeeds. The failed password attempts did not execute the command; the third attempt authenticated successfully.

### ✅ Step 4: Verify the container is gone

```bash
docker ps
```

The command returned an empty container list:

```text
[tony@stapp01 ~]$ docker ps
CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES
[tony@stapp01 ~]$
```

> **Why:** Running `docker ps` again confirms that no containers are currently running on App Server 1. Since `kke-container` no longer appears, the requested deletion was completed.

## Best Practices

- **Target the correct Docker host.** Container names are local to each Docker host, so the same name on another server would be a different resource.
- **Confirm the name before deleting.** `docker ps` helps avoid removing the wrong running container.
- **Use `-f` deliberately.** It is convenient for a disposable test container, but it stops a running workload immediately and should not be used casually in production.
- **Prefer graceful shutdown for important workloads.** For production containers, stop the application and confirm its replacement or maintenance plan before removing it.
- **Use exact container names.** `kke-container` was targeted explicitly, avoiding broad cleanup commands that could remove unrelated containers.

### 📚 Official Documentation

- [`docker container rm` reference](https://docs.docker.com/reference/cli/docker/container/rm/)
- [`docker container ls` reference](https://docs.docker.com/reference/cli/docker/container/ls/)
