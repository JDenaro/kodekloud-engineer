# Task 02: Deploy Nginx Container on Application Server

The Nautilus DevOps team is conducting application deployment tests on selected application servers. They require a nginx container deployment on Application Server 2. Complete the task with the following instructions:

On Application Server 2 create a container named nginx_2 using the nginx image with the alpine tag. Ensure container is in a running state.

## Task Requirements

1. Create the container on Application Server 2.
2. Name the container `nginx_2`.
3. Use the `nginx:alpine` image.
4. Ensure that the container is running.

## Solution

The container was created on App Server 2 with `docker run` in detached mode. The `nginx:alpine` image was not present locally, so Docker pulled it automatically from Docker Hub before starting the container.

### 🔌 Step 1: Connect to Application Server 2

```bash
ssh steve@stapp02
```

The connection succeeded:

```text
thor@jump-host ~$ ssh steve@stapp02
The authenticity of host 'stapp02 (10.244.195.219)' can't be established.
ED25519 key fingerprint is SHA256:yjJxOv8hpYFDSwNpz8b/6LVLtuL9jb2tePcssXMt9zE.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added 'stapp02 (10.244.195.219)' (ED25519) to the list of known hosts.
steve@stapp02's password:
[steve@stapp02 ~]$
```

> **Why:** `ssh` opens a secure remote shell. `steve` is the Linux user used for App Server 2, and `stapp02` identifies the target server. The container must be created on App Server 2 rather than on the jump host.

### 🐳 Step 2: Create and start the Nginx container

```bash
sudo docker run -d --name nginx_2 nginx:alpine
```

Docker pulled the image and started the container:

```text
[steve@stapp02 ~]$ sudo docker run -d --name nginx_2 nginx:alpine

We trust you have received the usual lecture from the local System
Administrator. It usually boils down to these three things:

    #1) Respect the privacy of others.
    #2) Think before you type.
    #3) With great power comes great responsibility.

[sudo] password for steve:
Unable to find image 'nginx:alpine' locally
alpine: Pulling from library/nginx
55afa1ecc21d: Pull complete
3cd534fe98c6: Pull complete
1223f016b4e4: Pull complete
62bec68d7c31: Pull complete
46f977ee452f: Pull complete
d0008c891db4: Pull complete
390dc935348d: Pull complete
46519e7231d2: Pull complete
Digest: sha256:4a73073bd557c65b759505da037898b61f1be6cbcc3c2c3aeac22d2a470c1752
Status: Downloaded newer image for nginx:alpine
4d72ddd1f26f326b4106f162e9d2cb8cd5debdd44841cfc31109e413e873999a
[steve@stapp02 ~]$
```

> **Why:** `sudo` provides the administrative privileges required to access the Docker daemon on this host. `docker run` creates and starts a container. `-d` runs it in detached mode so it remains in the background, `--name nginx_2` assigns the required container name, and `nginx:alpine` selects the official Nginx image with the `alpine` tag. Because the image was not local, Docker pulled it automatically before creating the container.

### ✅ Step 3: Verify that the container is running

```bash
sudo docker ps
```

The container appeared with an `Up` status:

```text
[steve@stapp02 ~]$ sudo docker ps
CONTAINER ID   IMAGE          COMMAND                  CREATED         STATUS         PORTS     NAMES
4d72ddd1f26f   nginx:alpine   "/docker-entrypoint.…"   7 seconds ago   Up 5 seconds   80/tcp    nginx_2
[steve@stapp02 ~]$
```

> **Why:** `docker ps` lists running containers. The `IMAGE` column confirms `nginx:alpine`, the `STATUS` column shows `Up`, and the `NAMES` column confirms the required name `nginx_2`. The `80/tcp` value shows that Nginx listens on port 80 inside the container; no host-port mapping was required by this task.

## Best Practices

- **Run commands on the requested server.** A container created on another host will not satisfy the application-server requirement.
- **Use explicit image tags.** `nginx:alpine` identifies the requested lightweight Nginx image variant instead of relying on the default `latest` tag.
- **Assign meaningful container names.** `nginx_2` makes the container easy to reference in later Docker commands.
- **Use detached mode for services.** `-d` keeps the Nginx process running in the background while returning control of the terminal.
- **Check the running state after creation.** `docker ps` confirms that the container is active and shows the image and name used.
- **Publish ports only when required.** The task required Nginx to run, but it did not require access from outside the Docker host, so no `-p` option was added.

### 📚 Official Documentation

- [`docker container run` reference](https://docs.docker.com/reference/cli/docker/container/run/)
- [`docker container ls` reference](https://docs.docker.com/reference/cli/docker/container/ls/)
- [Nginx Official Image](https://hub.docker.com/_/nginx)
