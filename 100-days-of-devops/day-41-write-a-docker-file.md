# Day 41: Write a Docker File

As per recent requirements shared by the Nautilus application development team, they need custom images created for one of their projects. Several of the initial testing requirements are already been shared with DevOps team. Therefore, create a docker file `/opt/docker/Dockerfile` (please keep `D` capital of Dockerfile) on `App server 1` in `Stratos DC` and configure to build an image with the following requirements:

a. Use `ubuntu:24.04` as the base image.

b. Install `apache2` and configure it to work on `6200` port. (do not update any other Apache configuration settings like document root etc).

## Specific Requirements:

1. Use `ubuntu:24.04` as the base image.
2. Install `apache2` and configure it to work on `6200` port. (do not update any other Apache configuration settings like document root etc).

## Solution

The `/opt/docker` directory already existed on App Server 1. A capitalized `Dockerfile` was created there. It installs Apache on top of `ubuntu:24.04`, changes only the Apache listener and default virtual-host port to `6200`, and starts Apache in the foreground when a container is created from the image.

### 🔐 Step 1: Connect to App Server 1 and enter the build context

```bash
ssh tony@stapp01
cd /opt/docker/
```

> **Why:** `ssh` opens a remote shell on App Server 1 as `tony`. `cd /opt/docker/` enters the directory specified by the challenge. This directory is the Docker build context and must contain the file named exactly `Dockerfile`, with a capital `D`.

### 📝 Step 2: Create the Dockerfile

```bash
sudo vi Dockerfile
```

Press `i` in `vi`, enter the following content, then press `Esc`, type `:wq`, and press `Enter`:

```Dockerfile
FROM ubuntu:24.04

RUN apt-get update && \
    apt-get install -y apache2 && \
    sed -i 's/^Listen 80$/Listen 6200/' /etc/apache2/ports.conf && \
    sed -i 's/<VirtualHost \*:80>/<VirtualHost *:6200>/' /etc/apache2/sites-available/000-default.conf

EXPOSE 6200

CMD ["apachectl", "-D", "FOREGROUND"]
```

> **Why:** `sudo` is needed because the Dockerfile is created in `/opt`. `FROM ubuntu:24.04` selects the exact base image required. The `RUN` instruction executes commands while building the image: `apt-get update` refreshes package metadata, then `apt-get install -y apache2` installs Apache without an interactive prompt. The first `sed -i` changes only `Listen 80` to `Listen 6200`, leaving Apache free to bind on all container interfaces. The second `sed -i` changes only the default virtual host from `*:80` to `*:6200`, while preserving settings such as the document root. `EXPOSE 6200` declares the intended application port as image metadata; it does not publish the port on the host by itself. `CMD` starts Apache with `FOREGROUND` so it remains the container's primary running process.

### ✅ Step 3: Verify the final Dockerfile

```bash
cat Dockerfile
```

The file contained:

```Dockerfile
FROM ubuntu:24.04

RUN apt-get update && \
    apt-get install -y apache2 && \
    sed -i 's/^Listen 80$/Listen 6200/' /etc/apache2/ports.conf && \
    sed -i 's/<VirtualHost \*:80>/<VirtualHost *:6200>/' /etc/apache2/sites-available/000-default.conf

EXPOSE 6200

CMD ["apachectl", "-D", "FOREGROUND"]
```

> **Why:** `cat` prints the Dockerfile without changing it. Seeing the `ubuntu:24.04` base, the Apache installation command, port `6200`, and the capitalized filename in `/opt/docker/` confirms that the build instructions meet the challenge requirements. No image name was provided, so the task requires only the Dockerfile rather than a build with an arbitrary tag.

## Best Practices

- **Use explicit base-image tags.** `ubuntu:24.04` makes the image build reproducible and satisfies the requested operating-system version.
- **Keep configuration changes scoped.** Change only Apache's listener and matching virtual-host port when the requirement excludes document-root or other Apache changes.
- **Run the web server in the foreground.** Containers remain alive while their primary process runs; `apachectl -D FOREGROUND` makes Apache that process.
- **Treat `EXPOSE` as documentation.** Publish port mappings deliberately when running a container; declaring a port in a Dockerfile does not expose it outside the container on its own.
- **Use the exact Dockerfile casing.** Docker's default build behavior searches for `Dockerfile`, so capitalization matters on Linux filesystems.

### 📚 Official Documentation

- [Dockerfile reference](https://docs.docker.com/reference/dockerfile/)
- [Dockerfile EXPOSE instruction](https://docs.docker.com/reference/dockerfile/#expose)
- [Apache HTTP Server binding and addressing](https://httpd.apache.org/docs/2.4/bind.html)
- [Apache apachectl documentation](https://httpd.apache.org/docs/2.4/programs/apachectl.html)
