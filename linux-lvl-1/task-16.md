# Task 16: Firewall Configuration

The Nautilus system administrators team has rolled out a web UI application for their backup utility on the Nautilus application server 2 within the Stratos Datacenter. This application runs on port 8089and appropriate firewall rules must be configured to allow incoming traffic. To achieve this, firewalld needs to be installed and configured on the application server. To ensure proper functionality, the following requirements have been identified:

Install and enable the firewalld service.
Allow all incoming connections on port 8089/tcp.
Ensure the zone is set to public.

## Task Requirements

1. Install and enable `firewalld` on App Server 2.
2. Allow incoming TCP connections on port `8089`.
3. Set the default firewall zone to `public`.

## Solution

The firewall is configured on App Server 2 using `firewalld`. The port rule is added to the `public` zone with `--permanent`, then `firewall-cmd --reload` applies the saved configuration to the active firewall without rebooting the server.

### 🔌 Step 1: Connect to App Server 2

Run the command from the Jump Host:

```bash
ssh steve@stapp02
```

The session opened on App Server 2:

```text
thor@jump-host ~$ ssh steve@stapp02
[steve@stapp02 ~]$
```

> **Why:** `ssh` opens a secure remote shell. `steve` is the login account for App Server 2, and `stapp02` is the server where the firewall must be installed and configured.

### 📦 Step 2: Install firewalld

```bash
sudo yum install -y firewalld
```

The package installation completed successfully:

```text
Installed:
  firewalld-1.3.4-19.el9.noarch
  firewalld-filesystem-1.3.4-19.el9.noarch
  nftables-1:1.0.9-8.el9.x86_64

Complete!
```

> **Why:** `sudo` provides the administrative privileges required to install system software. `yum` is the package manager used by this server. The `install` subcommand adds the requested package, and `-y` automatically confirms the installation prompt. `firewalld` provides the firewall daemon and `firewall-cmd` command-line client; its dependencies include the packet-filtering components used by the service.

### ▶️ Step 3: Enable and start the firewalld service

```bash
sudo systemctl enable --now firewalld
```

The command returned to the prompt without an error:

```text
[steve@stapp02 ~]$ sudo systemctl enable --now firewalld
[steve@stapp02 ~]$
```

> **Why:** `systemctl` controls services managed by `systemd`. `enable` configures `firewalld` to start automatically during future boots, while `--now` starts it immediately. Both actions are required so the firewall is active now and remains enabled after a restart.

### 🌐 Step 4: Set the default zone to public

```bash
sudo firewall-cmd --set-default-zone=public
```

The server reported that the requested zone was already selected and completed successfully:

```text
Warning: ZONE_ALREADY_SET: public
success
```

> **Why:** `firewall-cmd` is the command-line client for `firewalld`. `--set-default-zone=public` sets `public` as the default zone for connections and interfaces that do not have another zone assigned. `ZONE_ALREADY_SET` is informational here: it means the desired default was already `public`, so the requirement was still satisfied.

### 🔓 Step 5: Permanently allow port 8089/tcp

```bash
sudo firewall-cmd --permanent --zone=public --add-port=8089/tcp
```

The rule was added successfully:

```text
success
```

> **Why:** `--permanent` saves the rule in the persistent firewall configuration. `--zone=public` applies it specifically to the `public` zone. `--add-port=8089/tcp` allows incoming traffic to destination port `8089` using the TCP protocol. Because the application listens on `8089/tcp`, this is the required firewall rule.

### 🔄 Step 6: Reload firewalld

```bash
sudo firewall-cmd --reload
```

The reload completed successfully:

```text
success
```

> **Why:** `--reload` loads the permanent configuration into the active runtime configuration while preserving state information. This makes the new `8089/tcp` rule effective without restarting the server.

### ✅ Step 7: Verify the firewall configuration

```bash
sudo firewall-cmd --get-default-zone
sudo firewall-cmd --zone=public --list-ports
```

The lab returned:

```text
[steve@stapp02 ~]$ sudo firewall-cmd --get-default-zone
public
[steve@stapp02 ~]$ sudo firewall-cmd --zone=public --list-ports
8089/tcp
[steve@stapp02 ~]$
```

> **Why:** `--get-default-zone` prints the default zone, which must be `public`. `--zone=public --list-ports` lists the ports currently allowed in that zone. Seeing `8089/tcp` confirms that the application port is open in the active firewall configuration.

## Best Practices

- **Use the narrowest required rule.** Allow only `8089/tcp` instead of opening an entire range of ports.
- **Separate permanent and runtime configuration.** Add persistent rules with `--permanent`, then use `--reload` to apply them immediately.
- **Configure the correct zone.** The port must be allowed in the `public` zone required by the challenge.
- **Enable services for future boots.** `systemctl enable --now firewalld` makes the firewall active now and after future restarts.
- **Treat informational warnings correctly.** `ZONE_ALREADY_SET` indicates that the requested zone was already configured; it is not a failure.

### 📚 Official Documentation

- [firewall-cmd manual page](https://firewalld.org/documentation/man-pages/firewall-cmd.html)
- [firewalld manual page](https://firewalld.org/documentation/man-pages/firewalld.html)
