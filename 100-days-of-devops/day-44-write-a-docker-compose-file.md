# Day 44: Write a Docker Compose File

The Nautilus application development team shared static website content that needs to be hosted on the `httpd` web server using a containerised platform. The team has shared details with the DevOps team, and we need to set up an environment according to those guidelines. Below are the details:

a. On `App Server 3` in `Stratos DC` create a container named `httpd` using a docker compose file `/opt/docker/docker-compose.yml` (please use the exact name for file).

b. Use `httpd` (preferably `latest` tag) image for container and make sure container is named as `httpd`; you can use any name for service.

c. Map `80` number port of container with port `8082` of docker host.

d. Map container's `/usr/local/apache2/htdocs` volume with `/opt/security` volume of docker host which is already there. (please do not modify any data within these locations).

## Specific Requirements:

1. On `App Server 3` in `Stratos DC` create a container named `httpd` using a docker compose file `/opt/docker/docker-compose.yml` (please use the exact name for file).
2. Use `httpd` (preferably `latest` tag) image for container and make sure container is named as `httpd`; you can use any name for service.
3. Map `80` number port of container with port `8082` of docker host.
4. Map container's `/usr/local/apache2/htdocs` volume with `/opt/security` volume of docker host which is already there. (please do not modify any data within these locations).

## Solution

Docker Compose defines a multi-container application's configuration in YAML, even when the application contains only one container. The compose file creates the requested Apache HTTP Server container, publishes the required port, and bind-mounts the pre-existing static website content without changing it.

### 🔐 Step 1: Connect to App Server 3

```bash
ssh banner@stapp03
```

> **Why:** `ssh` opens a remote shell on App Server 3 as `banner`, the user authorized to manage the Docker environment for this lab.

### 📝 Step 2: Create the Compose file

```bash
cd /opt/docker
sudo vi docker-compose.yml
```

Add the following content, then save and exit with `Esc`, `:wq`, and `Enter`:

```yaml
services:
  web:
    image: httpd:latest
    container_name: httpd
    ports:
      - "8082:80"
    volumes:
      - /opt/security:/usr/local/apache2/htdocs:ro
```

> **Why:** `cd /opt/docker` changes to the directory required by the challenge. `sudo vi docker-compose.yml` creates or edits the exact Compose filename with elevated permissions. `services` contains the Compose-managed containers; `web` is an arbitrary service name. `image: httpd:latest` selects the requested Apache HTTP Server image and its `latest` tag. `container_name: httpd` assigns the exact required container name. Under `ports`, `"8082:80"` forwards host port `8082` to port `80` inside the container. Under `volumes`, `/opt/security` is bind-mounted at Apache's document root, `/usr/local/apache2/htdocs`. The `:ro` suffix makes this mount read-only inside the container, protecting the supplied content from application writes.

### 🚀 Step 3: Create and start the Compose service

```bash
sudo docker compose up -d
```

Compose pulled `httpd:latest`, created its default network, and created the `httpd` container.

> **Why:** `docker compose up` creates the resources declared in `docker-compose.yml` and starts the service. Docker pulls `httpd:latest` automatically when it is not already present. `-d` runs the service in detached mode, keeping the container running after the command returns. `sudo` provides the permissions needed to communicate with the Docker daemon on this host.

### ✅ Step 4: Verify the container and port mapping

```bash
docker ps
```

The running container showed the expected configuration:

```text
CONTAINER ID   IMAGE          COMMAND              CREATED         STATUS         PORTS                                   NAMES
8089d06ac6b5   httpd:latest   "httpd-foreground"   7 seconds ago   Up 5 seconds   0.0.0.0:8082->80/tcp, :::8082->80/tcp   httpd
```

> **Why:** `docker ps` lists running containers. `Up` confirms that the `httpd` container remains running. `0.0.0.0:8082->80/tcp` confirms the requested IPv4 host-to-container port mapping, while `:::8082->80/tcp` shows the equivalent IPv6 binding.

## Best Practices

- **Use declarative container definitions.** Keeping image, port, and volume settings in `docker-compose.yml` makes the deployment repeatable and easier to review.
- **Protect supplied static content.** Mounting `/opt/security` with `:ro` lets Apache serve the files while preventing container processes from writing to them.
- **Publish only required ports.** Exposing `8082` only for the HTTP service limits unnecessary network exposure.
- **Verify the effective configuration.** Inspect `docker ps` after startup to confirm both the container state and the actual published port.

### 📚 Official Documentation

- [Docker Compose file reference](https://docs.docker.com/reference/compose-file/)
- [Docker Compose `up` reference](https://docs.docker.com/reference/cli/docker/compose/up/)
- [Docker bind mounts](https://docs.docker.com/engine/storage/bind-mounts/)
- [Publishing and mapping ports](https://docs.docker.com/get-started/docker-concepts/running-containers/publishing-ports/)
