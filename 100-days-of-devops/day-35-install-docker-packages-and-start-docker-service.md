# Day 35: Install Docker Packages and Start Docker Service

The Nautilus DevOps team aims to containerize various applications following a recent meeting with the application development team. They intend to conduct testing with the following steps:

1. Install `docker-ce` and `docker compose` packages on `App Server 2`.

2. Initiate the `docker` service.

## Specific Requirements:

1. Install `docker-ce` and `docker compose` packages on `App Server 2`.
2. Initiate the `docker` service.

## Solution

The default package repositories on App Server 2 did not provide `docker-ce` or the Compose plugin. The official Docker CE repository was added first, then Docker Engine and the Docker Compose plugin were installed. The plugin supplies the modern `docker compose` command.

### 🔐 Step 1: Connect to App Server 2

```bash
ssh steve@stapp02
```

> **Why:** `ssh` opens a remote shell on App Server 2 as `steve`, the account used to administer this host. The package installation and service-management commands in the next steps require `sudo` privileges.

### 📦 Step 2: Install the repository-management plugin

```bash
sudo yum install -y dnf-plugins-core
```

> **Why:** `yum install` installs an RPM package through the system package manager. `dnf-plugins-core` provides the `dnf config-manager` command used to add an external package repository. `sudo` runs the command with the privileges required to modify the system package configuration, and `-y` accepts the package manager confirmation automatically.

### ➕ Step 3: Add Docker's official CE repository

```bash
sudo dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
```

> **Why:** `dnf config-manager --add-repo` registers the repository definition at the supplied URL. The default repositories did not contain the requested Docker CE packages; adding Docker's official repository makes `docker-ce` and the Compose plugin available to `yum`.

### 🐳 Step 4: Install Docker Engine and Docker Compose

```bash
sudo yum install -y docker-ce docker-compose-plugin
```

> **Why:** `docker-ce` installs Docker Community Edition, including the `dockerd` daemon that builds and runs containers. `docker-compose-plugin` installs Docker Compose as the `docker compose` subcommand requested by the challenge. Required runtime dependencies, such as `containerd.io` and the Docker CLI, are resolved by `yum` automatically.

### ▶️ Step 5: Start the Docker service

```bash
sudo systemctl start docker
sudo systemctl status docker
```

> **Why:** `systemctl start docker` starts the Docker systemd service for the current boot. `systemctl status docker` displays the daemon's current state and recent startup messages. The service reached `Active: active (running)`, which means the Docker daemon started successfully. The challenge requires starting the service, not enabling it to start automatically after a reboot.

### ✅ Step 6: Verify Docker is active

```bash
sudo systemctl is-active docker
```

The command returned:

```text
active
```

> **Why:** `systemctl is-active` reports only the service's active state. `active` confirms that the Docker daemon is currently running and ready to accept Docker commands.

## Best Practices

- **Use the appropriate vendor repository.** Add Docker's official CE repository when the distribution repositories do not provide the required `docker-ce` packages.
- **Install Compose as a plugin.** `docker-compose-plugin` provides the current `docker compose` command and integrates Compose with the Docker CLI.
- **Start only what the requirement asks for.** `start` makes Docker available immediately; use `enable` only when a service must also start automatically after reboots.
- **Confirm service health after changes.** Check the service state after installation so package installation issues and daemon startup failures are caught separately.

### 📚 Official Documentation

- [Install Docker Engine on CentOS](https://docs.docker.com/engine/install/centos/)
- [Docker Compose plugin](https://docs.docker.com/compose/install/linux/)
- [Manage services with systemctl](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/configuring_basic_system_settings/assembly_managing-systemd_configuring-basic-system-settings)
