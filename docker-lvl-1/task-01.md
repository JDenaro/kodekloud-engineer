# Task 01: Install Docker Packages and Start Docker Service

The Nautilus DevOps team aims to containerize various applications following a recent meeting with the application development team. They intend to conduct testing with the following steps:

## Task Requirements

1. Install docker-ce and docker compose packages on App Server 3.
2. Initiate the docker service.

## Solution

The first installation attempt failed because App Server 3 did not have Docker's official package repository configured. The enabled CentOS and EPEL repositories did not contain `docker-ce` or the legacy `docker-compose` package.

The successful solution used the official Docker repository for CentOS Stream 9. It installed Docker Engine, the Docker CLI, the container runtime, Buildx, and the modern Docker Compose plugin. The Compose plugin is used with the command `docker compose`; it is the current replacement for the legacy `docker-compose` executable.

### 🔌 Step 1: Connect to App Server 3

```bash
ssh banner@stapp03
```

The connection succeeded:

```text
thor@jump-host ~$ ssh banner@stapp03
The authenticity of host 'stapp03 (10.244.244.143)' can't be established.
ED25519 key fingerprint is SHA256:R0vsOyTtAEiHRHuCEUZlpvHXbTgEHh8Nizqh2Oyb1zM.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added 'stapp03 (10.244.244.143)' (ED25519) to the list of known hosts.
banner@stapp03's password:
[banner@stapp03 ~]$
```

> **Why:** `ssh` opens a secure remote shell, `banner` is the remote Linux user, and `stapp03` is App Server 3. The Docker packages must be installed on the target app server, not on the jump host.

### 🔎 Step 2: Identify the missing repository

The initial package installation was attempted with:

```bash
sudo yum install -y docker-ce docker-compose
```

The package manager could not find either requested package:

```text
[banner@stapp03 ~]$ sudo yum install -y docker-ce docker-compose

We trust you have received the usual lecture from the local System
Administrator. It usually boils down to these three things:

    #1) Respect the privacy of others.
    #2) Think before you type.
    #3) With great power comes great responsibility.

[sudo] password for banner:
No match for argument: docker-ce
No match for argument: docker-compose
Error: Unable to find a match: docker-ce docker-compose
[banner@stapp03 ~]$
```

> **Why:** `sudo` runs the package command with administrative privileges. `yum install` asks the enabled repositories for the named packages, and `-y` accepts the installation prompts automatically. `No match for argument` means that the currently enabled repositories do not provide those package names; it does not mean Docker itself is unavailable.

### 🧰 Step 3: Install repository management tools

```bash
sudo dnf -y install dnf-plugins-core
```

The tool was already installed, but the lab upgraded it to the available version:

```text
[banner@stapp03 ~]$ sudo dnf -y install dnf-plugins-core
[sudo] password for banner:
Package dnf-plugins-core-4.3.0-25.el9.noarch is already installed.
Dependencies resolved.
Upgrading:
 dnf-plugins-core              noarch      4.3.0-26.el9
 python3-dnf-plugins-core      noarch      4.3.0-26.el9
 yum-utils                     noarch      4.3.0-26.el9

Complete!
[banner@stapp03 ~]$
```

> **Why:** `dnf` is the package manager used by modern RPM-based distributions such as CentOS Stream 9. `-y` confirms the transaction without an additional prompt, and `install dnf-plugins-core` provides `dnf config-manager`, which can add external repositories. The package was already present, so DNF upgraded it rather than installing it from scratch.

### 🖥️ Step 4: Confirm the operating system

```bash
cat /etc/*release*
```

The server is running CentOS Stream 9:

```text
[banner@stapp03 ~]$ cat /etc/*release*
CentOS Stream release 9
NAME="CentOS Stream"
VERSION="9"
ID="centos"
ID_LIKE="rhel fedora"
VERSION_ID="9"
PLATFORM_ID="platform:el9"
PRETTY_NAME="CentOS Stream 9"
ANSI_COLOR="0;31"
LOGO="fedora-logo-icon"
CPE_NAME="cpe:/o:centos:centos:9"
HOME_URL="https://centos.org/"
BUG_REPORT_URL="https://issues.redhat.com/"
REDHAT_SUPPORT_PRODUCT="Red Hat Enterprise Linux 9"
REDHAT_SUPPORT_PRODUCT_VERSION="CentOS Stream"
CentOS Stream release 9
CentOS Stream release 9
cpe:/o:centos:centos:9
[banner@stapp03 ~]$
```

> **Why:** `cat` prints file contents. The `/etc/*release*` pattern selects the operating-system release files, and the output confirms that the CentOS installation instructions are the appropriate ones for this host.

### 📦 Step 5: Add the official Docker repository

```bash
sudo dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
```

The Docker repository was added successfully:

```text
[banner@stapp03 ~]$ sudo dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
Adding repo from: https://download.docker.com/linux/centos/docker-ce.repo
[banner@stapp03 ~]$
```

> **Why:** `config-manager` manages DNF repository configuration, `--add-repo` adds a repository definition, and the URL points to Docker's official CentOS repository. Adding this repository makes Docker's official packages discoverable by DNF.

### 🐳 Step 6: Install Docker Engine and Docker Compose

```bash
sudo dnf install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

The installation completed successfully. The important packages installed were:

```text
Installing:
 containerd.io               x86_64   2.2.6-1.el9       docker-ce-stable
 docker-buildx-plugin        x86_64   0.35.0-1.el9      docker-ce-stable
 docker-ce                   x86_64   3:29.6.2-1.el9    docker-ce-stable
 docker-ce-cli               x86_64   1:29.6.2-1.el9    docker-ce-stable
 docker-compose-plugin       x86_64   5.3.1-1.el9       docker-ce-stable

Complete!
[banner@stapp03 ~]$
```

> **Why:** `dnf install` installs packages from the configured repositories, and `-y` accepts the transaction automatically. `docker-ce` is Docker Engine, `docker-ce-cli` is the command-line client, `containerd.io` provides the container runtime, `docker-buildx-plugin` provides the Buildx builder, and `docker-compose-plugin` adds the modern `docker compose` command. The additional dependencies provide container security and networking support. Docker's official documentation recommends installing these packages from the Docker repository on CentOS Stream 9.

### ▶️ Step 7: Start the Docker service

```bash
sudo systemctl start docker
```

The command returned without an error:

```text
[banner@stapp03 ~]$ sudo systemctl start docker
[banner@stapp03 ~]$
```

> **Why:** `systemctl` controls services managed by systemd, and `start docker` starts the Docker daemon for the current boot. The task requires the service to be initiated; enabling it to start automatically at boot was not required.

### ✅ Step 8: Verify the Docker service

```bash
sudo systemctl status docker
```

The final status confirmed that Docker was active and running:

```text
[banner@stapp03 ~]$ sudo systemctl status docker
● docker.service - Docker Application Container Engine
     Loaded: loaded (/usr/lib/systemd/system/docker.service; disabled; preset: d>
     Active: active (running) since Fri 2026-07-24 11:44:29 UTC; 9s ago
TriggeredBy: ● docker.socket
       Docs: https://docs.docker.com
   Main PID: 31202 (dockerd)
     CGroup: /system.slice/docker.service
             └─31202 /usr/bin/dockerd -H fd:// --containerd=/run/containerd/cont>
Jul 24 11:44:29 stapp03 systemd[1]: Started Docker Application Container Engine.
lines 1-22/22 (END)
```

> **Why:** `status` displays the current state of a systemd service. `Active: active (running)` confirms that Docker is running. The `disabled` value in the `Loaded` line means the service is not configured to start automatically at boot, but that was acceptable because the task only required starting the service now.

## Root Cause

The first package installation failed because App Server 3 only had the standard CentOS and EPEL repositories enabled. Docker's official packages were not available until Docker's CentOS repository was added.

The package name for current Docker Compose on Linux is `docker-compose-plugin`, and its command is `docker compose`. The legacy standalone package and command are written as `docker-compose`.

## Best Practices

- **Use the official Docker repository.** It provides Docker's maintained Engine, CLI, runtime, Buildx, and Compose packages for the supported CentOS Stream release.
- **Install packages on the target server.** The Docker service must run on the application server that will host the containers, not on the jump host.
- **Prefer the Compose plugin.** The current command is `docker compose`; the standalone `docker-compose` installation is legacy and intended mainly for backward compatibility.
- **Start the service after installation.** Installing Docker does not necessarily start the daemon automatically on RPM-based systems.
- **Use `systemctl status` when troubleshooting.** It shows whether the service is loaded, running, and reporting errors.
- **Do not use the convenience script blindly in production.** Docker documents repository installation as the maintainable approach because packages can receive normal updates.

### 📚 Official Documentation

- [Install Docker Engine on CentOS](https://docs.docker.com/engine/install/centos/)
- [Install the Docker Compose plugin on Linux](https://docs.docker.com/compose/install/linux/)
- [Docker Compose installation overview](https://docs.docker.com/compose/install/)
- [Start the Docker daemon](https://docs.docker.com/engine/daemon/start/)
