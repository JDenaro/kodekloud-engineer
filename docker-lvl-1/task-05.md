# Task 05: Troubleshoot Docker Container Issue

An issue has arisen with a static website running in a container named nautilus on App Server 1. To resolve the issue, investigate the following details:

Check if the container's volume /usr/local/apache2/htdocs is correctly mapped with the host's volume /var/www/html.

Verify that the website is accessible on host port 8085 on App Server 1. Confirm that the command curl http://localhost:8085/ works on App Server 1.

## Task Requirements

1. Check whether the container path `/usr/local/apache2/htdocs` is mapped to the host path `/var/www/html`.
2. Verify that the website is exposed on host port `8085`.
3. Confirm that `curl http://localhost:8085/` works on App Server 1.

## Solution

The volume and port configuration were already correct. The apparent volume issue was a distraction: the `nautilus` container was stopped with `Exited (137)`, so the website could not respond. The fix was to start the existing container without changing or recreating its configuration.

### 🔌 Step 1: Connect to App Server 1

```bash
ssh tony@stapp01
```

The connection succeeded:

```text
thor@jump-host ~$ ssh tony@stapp01
The authenticity of host 'stapp01 (10.244.221.70)' can't be established.
ED25519 key fingerprint is SHA256:Ip7vB8hGY+uSQQeMz5OGRFieyT6eSpOTC99XaFEr6SE.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added 'stapp01 (10.244.221.70)' (ED25519) to the list of known hosts.
tony@stapp01's password:
[tony@stapp01 ~]$
```

> **Why:** `ssh` opens a secure remote shell. `tony` is the Linux user for App Server 1, and `stapp01` identifies the host where the `nautilus` container is running.

### 🔎 Step 2: Check the container state

```bash
sudo docker ps -a
```

The container was present but stopped:

```text
[tony@stapp01 ~]$ sudo docker ps -a

We trust you have received the usual lecture from the local System
Administrator. It usually boils down to these three things:

    #1) Respect the privacy of others.
    #2) Think before you type.
    #3) With great power comes great responsibility.

[sudo] password for tony:
CONTAINER ID   IMAGE     COMMAND              CREATED         STATUS                       PORTS     NAMES
761df9f001b7   httpd     "httpd-foreground"   9 minutes ago   Exited (137) 9 minutes ago             nautilus
[tony@stapp01 ~]$
```

> **Why:** `sudo` provides the privileges needed to query Docker on this host. `docker ps` lists containers, and `-a` includes containers that are stopped. The `STATUS` value `Exited (137)` showed that `nautilus` was not running, which was enough to explain why the website was unavailable.

### 🔎 Step 3: Inspect the volume and port configuration

```bash
sudo docker inspect nautilus
```

The relevant configuration was:

```json
"State": {
    "Status": "exited",
    "Running": false,
    "ExitCode": 137
},
"HostConfig": {
    "Binds": [
        "/var/www/html:/usr/local/apache2/htdocs/"
    ],
    "PortBindings": {
        "80/tcp": [
            {
                "HostIp": "",
                "HostPort": "8085"
            }
        ]
    }
},
"Mounts": [
    {
        "Type": "bind",
        "Source": "/var/www/html",
        "Destination": "/usr/local/apache2/htdocs",
        "RW": true
    }
]
```

> **Why:** `docker inspect` displays the low-level configuration of a container. The `Binds` and `Mounts` entries confirmed the required host-to-container mapping: `/var/www/html` to `/usr/local/apache2/htdocs`. The `PortBindings` entry confirmed that host port `8085` maps to the container's HTTP port `80/tcp`. Since both settings were correct, no configuration change was needed. The inspection output did not establish the reason for exit code `137`, but it did identify that the container needed to be started.

### ▶️ Step 4: Start the existing container

```bash
sudo docker start nautilus
```

The container started successfully:

```text
[tony@stapp01 ~]$ sudo docker start nautilus
nautilus
[tony@stapp01 ~]$
```

> **Why:** `docker start` starts an existing stopped container using its saved image, volume, port, and command configuration. Starting it was safer and simpler than deleting and recreating it because the required mappings were already correct.

### ✅ Step 5: Verify the container and website

Check the container state:

```bash
docker ps -a
```

The container was running with the expected port mapping:

```text
[tony@stapp01 ~]$ docker ps -a
CONTAINER ID   IMAGE     COMMAND              CREATED          STATUS          PORTS                                   NAMES
761df9f001b7   httpd     "httpd-foreground"   13 minutes ago   Up 33 seconds   0.0.0.0:8085->80/tcp, :::8085->80/tcp   nautilus
[tony@stapp01 ~]$
```

Test the website locally on App Server 1:

```bash
curl http://localhost:8085/
```

The web server responded successfully:

```text
[tony@stapp01 ~]$ curl http://localhost:8085/
Welcome to xFusionCorp Industries![tony@stapp01 ~]$
```

> **Why:** `docker ps -a` confirmed that the container changed from `Exited` to `Up` and that host port `8085` maps to container port `80`. `curl` sends an HTTP request to `localhost:8085`; receiving the website text confirms that the container, port mapping, and static content are working together.

## Root Cause

The root cause was the stopped `nautilus` container:

```text
Before: Exited (137)
After:  Up 33 seconds
```

The volume mapping and port mapping were already correct. The volume requirement was a diagnostic check, not the actual fault that needed correction.

## Best Practices

- **Check container state before changing configuration.** A stopped container can make a correctly configured volume or port appear faulty.
- **Use `docker inspect` for detailed diagnosis.** It exposes mounts, port bindings, state, exit codes, and runtime configuration.
- **Prefer starting an existing container when its configuration is correct.** This preserves the container's current settings and avoids unnecessary recreation.
- **Verify both process state and application response.** `docker ps` confirms that the container is running, while `curl` confirms that the application is actually reachable.
- **Interpret exit codes carefully.** Exit code `137` indicates termination by `SIGKILL`, but this output alone did not prove the exact reason for the termination; the immediate actionable problem was that the container was not running.
- **Test from the target host.** `curl http://localhost:8085/` validates the host-to-container port path locally on App Server 1.

### 📚 Official Documentation

- [`docker container inspect` reference](https://docs.docker.com/reference/cli/docker/container/inspect/)
- [`docker container start` reference](https://docs.docker.com/reference/cli/docker/container/start/)
- [`docker container ls` reference](https://docs.docker.com/reference/cli/docker/container/ls/)
