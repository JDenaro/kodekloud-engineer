# Day 43: Docker Ports Mapping

The Nautilus DevOps team is planning to host an application on a nginx-based container. There are number of tickets already been created for similar tasks. One of the tickets has been assigned to set up a nginx container on `Application Server 2` in `Stratos Datacenter`. Please perform the task as per details mentioned below:

a. Pull `nginx:alpine` docker image on `Application Server 2`.

b. Create a container named `games` using the image you pulled.

c. Map host port `6400` to container port `80`. Please keep the container in running state.

## Specific Requirements:

1. Pull `nginx:alpine` docker image on `Application Server 2`.
2. Create a container named `games` using the image you pulled.
3. Map host port `6400` to container port `80`. Please keep the container in running state.

## Solution

Docker port publishing exposes a container service through a port on the host. Nginx listens on port `80` inside the container, so Docker maps host port `6400` to that internal port while the container runs in the background.

### 🔐 Step 1: Connect to App Server 2

```bash
ssh steve@stapp02
```

> **Why:** `ssh` opens a remote shell on App Server 2 as `steve`, the authorized user for this Docker lab.

### 📥 Step 2: Pull the Nginx image

```bash
docker pull nginx:alpine
```

Docker downloaded the image with digest `sha256:72ba65eb42c10344912a84ff42408db7d34f2feb642204570ab8fc5ffd29f1d3`.

> **Why:** `docker pull` downloads an image from its registry to the Docker host. `nginx` is the image repository and `alpine` is the tag selecting the small Alpine Linux-based Nginx image required by the challenge.

### 🚀 Step 3: Run the container with a published port

```bash
docker run -d --name games -p 6400:80 nginx:alpine
```

Docker created the container with ID `4a3507287d72426294ce1e72c191b6235a7f2b20c2209e5aea5d7a4d559c82ad`.

> **Why:** `docker run` creates and starts a container from an image. `-d` runs it in detached mode, so it remains running after the terminal returns. `--name games` assigns the required container name. `-p 6400:80` publishes TCP port `6400` on the host and forwards incoming traffic to port `80` inside the container, where Nginx listens. `nginx:alpine` is the image pulled in the preceding step.

### ✅ Step 4: Verify the running container and port mapping

```bash
docker ps
```

The output confirmed that `games` was running and exposed the required mapping:

```text
CONTAINER ID   IMAGE          COMMAND                  CREATED         STATUS         PORTS                                   NAMES
4a3507287d72   nginx:alpine   "/docker-entrypoint…"   6 seconds ago   Up 5 seconds   0.0.0.0:6400->80/tcp, :::6400->80/tcp   games
```

> **Why:** `docker ps` lists running containers. `Up` confirms that `games` remains running. `0.0.0.0:6400->80/tcp` confirms IPv4 host traffic on port `6400` reaches container port `80`; `:::6400->80/tcp` shows the equivalent IPv6 binding.

## Best Practices

- **Use an explicit image tag.** `nginx:alpine` is predictable and avoids unintentionally pulling the default `latest` tag.
- **Publish only required ports.** Mapping `6400:80` exposes the application through the requested host port without opening additional container ports.
- **Use meaningful container names.** The `games` name makes later operational commands, such as `docker logs games`, easier to use.
- **Verify the runtime state.** Check `docker ps` after deployment to confirm both the running state and the actual port mapping.

### 📚 Official Documentation

- [Docker image pull reference](https://docs.docker.com/reference/cli/docker/image/pull/)
- [Docker container run reference](https://docs.docker.com/reference/cli/docker/container/run/)
- [Publishing and mapping ports](https://docs.docker.com/get-started/docker-concepts/running-containers/publishing-ports/)
- [Docker container ls reference](https://docs.docker.com/reference/cli/docker/container/ls/)
