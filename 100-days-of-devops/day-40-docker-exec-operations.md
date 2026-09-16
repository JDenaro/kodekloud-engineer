# Day 40: Docker EXEC Operations

One of the Nautilus DevOps team members was working to configure services on a `kkloud` container that is running on `App Server 1` in `Stratos Datacenter`. Due to some personal work he is on PTO for the rest of the week, but we need to finish his pending work ASAP. Please complete the remaining work as per details given below:

a. Install `apache2` in `kkloud` container using `apt` that is running on `App Server 1` in `Stratos Datacenter`.

b. Configure Apache to listen on port `5003` instead of default `http` port. Do not bind it to listen on specific IP or hostname only, i.e it should listen on localhost, 127.0.0.1, container ip, etc.

c. Make sure Apache service is up and running inside the container. Keep the container in running state at the end.

## Specific Requirements:

1. Install `apache2` in `kkloud` container using `apt` that is running on `App Server 1` in `Stratos Datacenter`.
2. Configure Apache to listen on port `5003` instead of default `http` port. Do not bind it to listen on specific IP or hostname only, i.e it should listen on localhost, 127.0.0.1, container ip, etc.
3. Make sure Apache service is up and running inside the container. Keep the container in running state at the end.

## Solution

The `kkloud` container runs Ubuntu 18.04 and does not use systemd as its init process. Apache was therefore installed with `apt` and started explicitly with the `service` command. Its generic `Listen 5003` directive binds Apache to all available interfaces, while the default virtual host was updated to use the same port.

### 🔐 Step 1: Connect to App Server 1 and enter the container

```bash
ssh tony@stapp01
docker exec -it kkloud bash
```

> **Why:** `ssh` opens a remote shell on App Server 1 as `tony`. `docker exec` runs a command inside an already running container. `-i` keeps standard input open, `-t` allocates an interactive terminal, `kkloud` selects the target container, and `bash` opens its shell so the remaining commands run inside the container.

### 📦 Step 2: Refresh package metadata and install Apache

```bash
apt update
apt install apache2 -y
```

> **Why:** `apt update` downloads current package metadata from the Ubuntu repositories. `apt install apache2 -y` installs Apache HTTP Server and its dependencies; `-y` accepts the installation prompt non-interactively. The package installation did not start Apache automatically because the container's policy prevents service autostart, so it is started explicitly after configuration.

### ⚙️ Step 3: Configure Apache to listen on port 5003

```bash
sed -i 's/^Listen 80$/Listen 5003/' /etc/apache2/ports.conf
sed -i 's/<VirtualHost \*:80>/<VirtualHost *:5003>/' /etc/apache2/sites-enabled/000-default.conf
grep -n '<VirtualHost' /etc/apache2/sites-enabled/000-default.conf
```

The virtual host configuration returned:

```text
1:<VirtualHost *:5003>
```

> **Why:** `sed -i` updates a file in place, which is useful in a minimal container without a text editor. The first command replaces only the standalone `Listen 80` directive with `Listen 5003`. Because no IP address or hostname precedes `5003`, Apache listens on every available interface rather than one specific address. The second command changes the default site's `<VirtualHost *:80>` declaration to `<VirtualHost *:5003>`, keeping the virtual host aligned with Apache's listener. `grep -n` confirms the resulting virtual-host line and displays its line number.

### ▶️ Step 4: Validate and start Apache

```bash
apache2ctl configtest
service apache2 restart
service apache2 status
```

The configuration test and service status returned:

```text
Syntax OK
* apache2 is running
```

> **Why:** `apache2ctl configtest` parses the Apache configuration without serving traffic; `Syntax OK` confirms that the edited directives are valid. `service apache2 restart` starts Apache or reloads it with the new port configuration. `service apache2 status` confirms that the Apache process remains active inside the container. The fully qualified domain name warning displayed during startup is informational and did not prevent Apache from running.

### ✅ Step 5: Verify the container remains running

```bash
exit
docker ps --filter "name=kkloud"
```

The Docker host showed:

```text
CONTAINER ID   IMAGE          COMMAND       CREATED          STATUS          PORTS     NAMES
7721cc9ea911   ubuntu:18.04   "/bin/bash"   12 minutes ago   Up 12 minutes             kkloud
```

> **Why:** `exit` returns from the container shell to App Server 1. `docker ps` lists running containers, while `--filter "name=kkloud"` limits the result to the required container. Its `Up` status confirms that configuring Apache did not stop the container.

## Best Practices

- **Validate configuration before restarting.** Run `apache2ctl configtest` before applying Apache configuration changes so syntax mistakes do not interrupt the service.
- **Use an all-interface listener when required.** `Listen 5003` permits connections through localhost and the container network interface without binding Apache to a single address.
- **Keep listener and virtual-host ports consistent.** Update both `ports.conf` and the active site's `<VirtualHost>` declaration when changing Apache's HTTP port.
- **Use non-interactive configuration tools in minimal images.** `sed -i` makes a targeted, repeatable change without installing a text editor solely for a one-line configuration update.
- **Check container state separately.** A working Apache process does not by itself prove that the container's main process is still running; verify both states.

### 📚 Official Documentation

- [Docker exec reference](https://docs.docker.com/reference/cli/docker/container/exec/)
- [Apache HTTP Server binding and addressing](https://httpd.apache.org/docs/2.4/bind.html)
- [Apache apachectl documentation](https://httpd.apache.org/docs/2.4/programs/apachectl.html)
- [APT user guide](https://www.debian.org/doc/manuals/apt-guide/)
