# Day 46: Deploy an App on Docker Containers

The Nautilus Application development team recently finished development of one of the apps that they want to deploy on a containerized platform. The Nautilus Application development and DevOps teams met to discuss some of the basic pre-requisites and requirements to complete the deployment. The team wants to test the deployment on one of the app servers before going live and set up a complete containerized stack using a docker compose fie. Below are the details of the task:

On App Server 2 in Stratos Datacenter create a docker compose file /opt/itadmin/docker-compose.yml (should be named exactly).

The compose should deploy two services (web and DB), and each service should deploy a container as per details below:

For web service:

a. Container name must be php_web.

b. Use image php with any apache tag. Check here for more details.

c. Map php_web container's port 80 with host port 8088

d. Map php_web container's /var/www/html volume with host volume /var/www/html.

For DB service:

a. Container name must be mysql_web.

b. Use image mariadb with any tag (preferably latest). Check here for more details.

c. Map mysql_web container's port 3306 with host port 3306

d. Map mysql_web container's /var/lib/mysql volume with host volume /var/lib/mysql.

e. Set MYSQL_DATABASE=database_web and use any custom user ( except root ) with some complex password for DB connections.

After running docker-compose up you can access the app with curl command curl <server-ip or hostname>:8088/

For more details check here.

## Specific Requirements:

1. On App Server 2 in Stratos Datacenter create a docker compose file /opt/itadmin/docker-compose.yml (should be named exactly).
2. The compose should deploy two services (web and DB).
3. For web service, the container name must be php_web, use image php with any apache tag, map container port 80 with host port 8088, and map the container's /var/www/html volume with host volume /var/www/html.
4. For DB service, the container name must be mysql_web, use image mariadb with any tag (preferably latest), map container port 3306 with host port 3306, and map the container's /var/lib/mysql volume with host volume /var/lib/mysql.
5. Set MYSQL_DATABASE=database_web and use any custom user ( except root ) with some complex password for DB connections.
6. After running docker-compose up you can access the app with curl command curl <server-ip or hostname>:8088/.

## Solution

Docker Compose deployed a PHP Apache web service and a MariaDB database service on App Server 2. The lab run used a non-root database user and initialized `database_web`; the credential values are intentionally excluded from this guide. Hardcoding passwords in a Compose file creates a credential exposure risk, so the reusable configuration below injects them from the runtime environment instead.

### 🔐 Step 1: Connect to App Server 2 and open the Compose directory

```bash
ssh steve@stapp02
sudo su -
cd /opt/itadmin
```

> **Why:** `ssh` opens a remote shell on App Server 2 as `steve`. `sudo su -` opens a root shell, which has the permissions required to manage the Compose project and Docker daemon. `cd /opt/itadmin` selects the directory required for the exact `docker-compose.yml` path.

### 📝 Step 2: Define the web and database services

```bash
vi docker-compose.yml
```

Add the following content, then save and exit with `Esc`, `:wq`, and `Enter`:

```yaml
services:
  web:
    image: php:apache
    container_name: php_web
    ports:
      - "8088:80"
    volumes:
      - /var/www/html:/var/www/html

  db:
    image: mariadb:latest
    container_name: mysql_web
    ports:
      - "3306:3306"
    volumes:
      - /var/lib/mysql:/var/lib/mysql
    environment:
      MYSQL_DATABASE: database_web
      MYSQL_USER: ${MYSQL_USER:?set MYSQL_USER}
      MYSQL_PASSWORD: ${MYSQL_PASSWORD:?set MYSQL_PASSWORD}
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD:?set MYSQL_ROOT_PASSWORD}
```

> **Why:** `vi` creates the exact Compose file required by the task. `web` and `db` are the two service definitions. `php:apache` supplies PHP with Apache listening on container port `80`; `container_name: php_web` assigns the required name; and `"8088:80"` forwards host port `8088` to the web container's port `80`. `/var/www/html:/var/www/html` bind-mounts the host's supplied web content at the same path in the container. `mariadb:latest` supplies the database image; `container_name: mysql_web` assigns its required name; `"3306:3306"` publishes its database port; and `/var/lib/mysql:/var/lib/mysql` persists database files on the host. `MYSQL_DATABASE` initializes the required database. The other values create the non-root database account and set the MariaDB root password. The `${VARIABLE:?message}` form requires a value at startup and prevents credentials from being stored in the Compose file.

### 🔒 Step 3: Provide credentials at runtime

```bash
export MYSQL_USER="kodekloud"
read -r -s -p "Password for MYSQL_USER: " MYSQL_PASSWORD
echo
export MYSQL_PASSWORD
read -r -s -p "MariaDB root password: " MYSQL_ROOT_PASSWORD
echo
export MYSQL_ROOT_PASSWORD
```

> **Why:** `export MYSQL_USER` provides the required non-root user name. `read -r -s -p` reads each password without echoing it to the terminal: `-r` preserves the input literally, `-s` suppresses display, and `-p` shows a prompt. Each subsequent `export` makes the value available to Docker Compose. This keeps passwords out of the Compose file, terminal history, and this repository. Use distinct, randomly generated passwords in a real environment and retrieve them from a secret-management system rather than entering them manually.

### 🚀 Step 4: Create and start the stack

```bash
docker compose up -d
```

Compose pulled `php:apache` and `mariadb:latest`, created the project network, and created both containers.

> **Why:** `docker compose up` creates the resources declared in `docker-compose.yml` and starts each service. `-d` runs the stack in detached mode, leaving the web and database containers running after the command finishes.

### ✅ Step 5: Verify both running services and port mappings

```bash
docker compose ps
```

The Compose project reported both required containers as running:

```text
NAME        IMAGE            COMMAND                  SERVICE   CREATED         STATUS         PORTS
mysql_web   mariadb:latest   "docker-entrypoint.s…"   db        7 seconds ago   Up 6 seconds   0.0.0.0:3306->3306/tcp, [::]:3306->3306/tcp
php_web     php:apache       "docker-php-entrypoi…"   web       7 seconds ago   Up 6 seconds   0.0.0.0:8088->80/tcp, [::]:8088->80/tcp
```

> **Why:** `docker compose ps` displays the state of the services in the current Compose project. `Up` confirms that both `php_web` and `mysql_web` are running. The published-port entries confirm the required `8088:80` web mapping and `3306:3306` database mapping.

## Best Practices

- **Inject secrets instead of hardcoding them.** Do not put database passwords in `docker-compose.yml`, command history, or version control. Use runtime environment injection for a lab and a managed secrets service in real deployments. Rotate any password that was written directly into a configuration file.
- **Use a non-root database account.** Applications should connect with an account that has only the permissions they require; reserve the MariaDB root account for administration.
- **Keep database storage persistent.** The `/var/lib/mysql` bind mount keeps database files beyond the lifecycle of the `mysql_web` container. Back up that host path before destructive changes.
- **Limit database exposure outside the lab.** This task explicitly requires publishing port `3306`; in a real stack, keep the database on a private network and do not publish its port unless external access is necessary.
- **Use explicit image versions in long-lived deployments.** The lab accepts `latest`, but a specific PHP and MariaDB version makes production deployments repeatable.

### 📚 Official Documentation

- [Docker Compose file reference](https://docs.docker.com/reference/compose-file/)
- [Docker Compose environment variables](https://docs.docker.com/compose/how-tos/environment-variables/variable-interpolation/)
- [Docker Compose `up` reference](https://docs.docker.com/reference/cli/docker/compose/up/)
- [Docker bind mounts](https://docs.docker.com/engine/storage/bind-mounts/)
- [MariaDB Docker Official Image](https://hub.docker.com/_/mariadb)
