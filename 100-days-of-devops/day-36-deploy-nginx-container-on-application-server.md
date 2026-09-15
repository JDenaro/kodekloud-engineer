# Day 36: Deploy Nginx Container on Application Server

The Nautilus DevOps team is conducting application deployment tests on selected application servers. They require a nginx container deployment on `Application Server 2`. Complete the task with the following instructions:

1. On `Application Server 2` create a container named `nginx_2` using the `nginx` image with the `alpine` tag. Ensure container is in a `running` state

## Specific Requirements:

1. On `Application Server 2` create a container named `nginx_2` using the `nginx` image with the `alpine` tag. Ensure container is in a `running` state

## Solution

Docker was already available to the `steve` user on App Server 2. Running the requested image with detached mode created and started the container in one command. Because `nginx:alpine` was not present locally, Docker downloaded it automatically before starting `nginx_2`.

### 🔐 Step 1: Connect to App Server 2

```bash
ssh steve@stapp02
```

> **Why:** `ssh` opens a remote shell on App Server 2 as `steve`, which is the account used to operate Docker in this lab.

### 🐳 Step 2: Create and start the Nginx container

```bash
docker run -d --name nginx_2 nginx:alpine
```

Docker returned the new container ID:

```text
31906807992b153312e25aaf4e559503efb5c583d92f435f7a83af4afff8517c
```

> **Why:** `docker run` creates a container from an image and starts it. `-d` runs it in detached mode, leaving the terminal available. `--name nginx_2` assigns the exact container name required by the challenge. `nginx:alpine` selects the official `nginx` image and its `alpine` tag; the tag identifies the Alpine Linux-based variant. Docker pulled the image because it was not yet available locally.

### ✅ Step 3: Verify that the container is running

```bash
docker ps
```

The output showed the requested container in the running state:

```text
CONTAINER ID   IMAGE          COMMAND                  CREATED          STATUS          PORTS     NAMES
31906807992b   nginx:alpine   "/docker-entrypoint.…"   34 seconds ago   Up 33 seconds   80/tcp    nginx_2
```

> **Why:** `docker ps` lists currently running containers. The `nginx:alpine` image, the `nginx_2` name, and the `Up 33 seconds` status confirm that the correct container exists and is running. Nginx listens on port `80/tcp` inside the container; no host port publication was required by this challenge.

## Best Practices

- **Use explicit image tags.** `nginx:alpine` is deterministic and avoids relying on the mutable `latest` tag.
- **Use meaningful container names.** A fixed name such as `nginx_2` makes containers easier to identify and manage.
- **Run services in detached mode.** Use `-d` for long-running services so they continue after the terminal returns.
- **Verify the runtime state.** `docker ps` distinguishes a successfully created container from one that started and then exited.

### 📚 Official Documentation

- [Docker run reference](https://docs.docker.com/reference/cli/docker/container/run/)
- [Docker ps reference](https://docs.docker.com/reference/cli/docker/container/ls/)
- [Nginx Docker Official Image](https://hub.docker.com/_/nginx)
