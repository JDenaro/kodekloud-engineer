# Day 03: Secure Root SSH Access

Following security audits, the xFusionCorp Industries security team has rolled out new protocols, including the restriction of direct root SSH login.

## Specific Requirements:

1. Disable direct SSH root login on all app servers within the Stratos Datacenter.

## Solution

The change must be applied to all three application servers: `stapp01`, `stapp02`, and `stapp03`. On each server, the active `PermitRootLogin yes` directive in `/etc/ssh/sshd_config` is changed to `PermitRootLogin no`. The SSH configuration is then validated and reloaded so new connections use the security policy.

### 🔌 Step 1: Connect to App Server 1

```bash
ssh tony@stapp01
```

The first connection to `stapp01` asked us to confirm its SSH host key:

```text
thor@jump-host ~$ ssh tony@stapp01
The authenticity of host 'stapp01 (10.244.73.182)' can't be established.
ED25519 key fingerprint is SHA256:+t0sXlGIQOzp4IZ8ZAlcZWbXKqhU+8/nxfsFgjulzMg.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added 'stapp01' (ED25519) to the list of known hosts.
tony@stapp01's password:
[tony@stapp01 ~]$
```

> **Why:** `ssh` opens an encrypted remote-login session. `tony` is the lab user for App Server 1, and `stapp01` is that server's hostname. During the first connection, SSH displays the server's host-key fingerprint and asks for confirmation before adding the host to the local `known_hosts` file. Enter the lab password `Ir0nM@n` when prompted.

### 🔎 Step 2: Inspect the current root-login setting

```bash
sudo grep PermitRootLogin /etc/ssh/sshd_config
```

The initial output on `stapp01` showed both commented examples and the active setting:

```text
[tony@stapp01 ~]$ sudo grep PermitRootLogin /etc/ssh/sshd_config

We trust you have received the usual lecture from the local System
Administrator. It usually boils down to these three things:

    #1) Respect the privacy of others.
    #2) Think before you type.
    #3) With great power comes great responsibility.

[sudo] password for tony:
#PermitRootLogin prohibit-password
# the setting of "PermitRootLogin without-password".
PermitRootLogin yes
[tony@stapp01 ~]$
```

> **Why:** `sudo` allows the command to read the system configuration with administrator privileges. `grep` searches the file for the text `PermitRootLogin`. Lines beginning with `#` are comments and do not take effect; the active line was `PermitRootLogin yes`, which allows direct root SSH login.

### 🔒 Step 3: Disable root SSH login on App Server 1

```bash
sudo sed -i 's/PermitRootLogin yes/PermitRootLogin no/' /etc/ssh/sshd_config
```

The command returned to the prompt without an error:

```text
[tony@stapp01 ~]$ sudo sed -i 's/PermitRootLogin yes/PermitRootLogin no/' /etc/ssh/sshd_config
[tony@stapp01 ~]$
```

> **Why:** `sed` edits text in a file. Its `-i` option applies the edit directly to the original file, and the `s/old/new/` expression replaces the exact active setting `PermitRootLogin yes` with `PermitRootLogin no`. The `sshd_config` path is the main OpenSSH server configuration file. The value `no` prevents the `root` account from logging in directly through SSH.

### 🧪 Step 4: Validate the SSH configuration on App Server 1

```bash
sudo sshd -t
```

The command produced no error output:

```text
[tony@stapp01 ~]$ sudo sshd -t
[tony@stapp01 ~]$
```

> **Why:** `sshd` is the OpenSSH server daemon. The `-t` option checks the configuration file's syntax and key sanity without restarting the service. Returning to the prompt without an error means the edited configuration can be loaded safely.

### 🔄 Step 5: Reload SSH on App Server 1

```bash
sudo systemctl reload sshd
```

The service reloaded successfully without output:

```text
[tony@stapp01 ~]$ sudo systemctl reload sshd
[tony@stapp01 ~]$
```

> **Why:** `systemctl` controls services managed by `systemd`. The `reload` action asks `sshd` to reread its configuration without stopping the service or closing the current SSH session. The new root-login restriction will apply to new SSH connections.

### 🚪 Step 6: Return to the jump host

```bash
exit
```

```text
[tony@stapp01 ~]$ exit
logout
Connection to stapp01 closed.
thor@jump-host ~$
```

> **Why:** `exit` closes the current SSH session and returns to the jump host. From there, we connect separately to App Server 2 and apply the same security change.

### 🔌 Step 7: Connect to App Server 2

```bash
ssh steve@stapp02
```

The first connection to `stapp02` asked us to confirm its host key:

```text
thor@jump-host ~$ ssh steve@stapp02
The authenticity of host 'stapp02 (10.244.195.200)' can't be established.
ED25519 key fingerprint is SHA256:K6My+2M5u7njosX8nMRr4QBAf6Wnf2CxGhrJPRJf53U.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added 'stapp02' (ED25519) to the list of known hosts.
steve@stapp02's password:
[steve@stapp02 ~]$
```

> **Why:** `ssh` opens the encrypted session to App Server 2. `steve` is the lab user for `stapp02`; enter the lab password `Am3ric@` when prompted. The host-key confirmation is only required the first time this jump host connects to `stapp02`.

### 🔒 Step 8: Disable root SSH login on App Server 2

```bash
sudo sed -i 's/PermitRootLogin yes/PermitRootLogin no/' /etc/ssh/sshd_config
```

The command completed without an error:

```text
[steve@stapp02 ~]$ sudo sed -i 's/PermitRootLogin yes/PermitRootLogin no/' /etc/ssh/sshd_config

We trust you have received the usual lecture from the local System
Administrator. It usually boils down to these three things:

    #1) Respect the privacy of others.
    #2) Think before you type.
    #3) With great power comes great responsibility.

[sudo] password for steve:
[steve@stapp02 ~]$
```

> **Why:** This is the same exact replacement used on `stapp01`: it changes the active root-login setting from `yes` to `no` in `/etc/ssh/sshd_config`. `sudo` is required because the file belongs to the system and regular users cannot modify it directly.

### 🧪 Step 9: Validate the SSH configuration on App Server 2

```bash
sudo sshd -t
```

```text
[steve@stapp02 ~]$ sudo sshd -t
[steve@stapp02 ~]$
```

> **Why:** `sshd -t` checks that the edited configuration is valid before the running SSH daemon reloads it. No output indicates that the test completed successfully.

### 🔄 Step 10: Reload SSH on App Server 2

```bash
sudo systemctl reload sshd
```

```text
[steve@stapp02 ~]$ sudo systemctl reload sshd
[steve@stapp02 ~]$
```

> **Why:** Reloading `sshd` makes the new `PermitRootLogin no` policy active for future SSH connections while preserving the current session.

### 🚪 Step 11: Return to the jump host

```bash
exit
```

```text
[steve@stapp02 ~]$ exit
logout
Connection to stapp02 closed.
thor@jump-host ~$
```

> **Why:** We return to the jump host before connecting to the final application server, `stapp03`.

### 🔌 Step 12: Connect to App Server 3

```bash
ssh banner@stapp03
```

The first connection to `stapp03` asked us to confirm its host key:

```text
thor@jump-host ~$ ssh banner@stapp03
The authenticity of host 'stapp03 (10.244.81.43)' can't be established.
ED25519 key fingerprint is SHA256:6yiAzvEJK5yZO6MDXGrHYxzp7zc/Ko8e9A3iYp3JWaY.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added 'stapp03' (ED25519) to the list of known hosts.
banner@stapp03's password:
[banner@stapp03 ~]$
```

> **Why:** `ssh` opens the encrypted session to App Server 3. `banner` is the lab user for `stapp03`; enter the lab password `BigGr33n` when prompted. The fingerprint confirmation establishes trust for this lab host on the jump host.

### 🔒 Step 13: Disable root SSH login on App Server 3

```bash
sudo sed -i 's/PermitRootLogin yes/PermitRootLogin no/' /etc/ssh/sshd_config
```

The command completed without an error:

```text
[banner@stapp03 ~]$ sudo sed -i 's/PermitRootLogin yes/PermitRootLogin no/' /etc/ssh/sshd_config

We trust you have received the usual lecture from the local System
Administrator. It usually boils down to these three things:

    #1) Respect the privacy of others.
    #2) Think before you type.
    #3) With great power comes great responsibility.

[sudo] password for banner:
[banner@stapp03 ~]$
```

> **Why:** The same exact replacement is applied to the third application server so all app servers enforce the same root SSH policy. `PermitRootLogin no` disables direct SSH login for `root` while leaving normal administrative access through named users and `sudo` available.

### 🧪 Step 14: Validate the SSH configuration on App Server 3

```bash
sudo sshd -t
```

```text
[banner@stapp03 ~]$ sudo sshd -t
[banner@stapp03 ~]$
```

> **Why:** `sshd -t` confirms that the configuration on `stapp03` is syntactically valid. The command returned with no output, so the daemon accepted the edited file.

### 🔄 Step 15: Reload SSH on App Server 3

```bash
sudo systemctl reload sshd
```

```text
[banner@stapp03 ~]$ sudo systemctl reload sshd
[banner@stapp03 ~]$
```

> **Why:** Reloading `sshd` applies the root-login restriction to new connections without interrupting the current session.

### 🚪 Step 16: Return to the jump host

```bash
exit
```

```text
[banner@stapp03 ~]$ exit
logout
Connection to stapp03 closed.
thor@jump-host ~$
```

> **Why:** The successful return to `thor@jump-host` confirms that the session to `stapp03` was closed after the configuration was applied.

### ✅ Step 17: Verify

The lab was completed after the configuration was validated with `sshd -t` and reloaded successfully on all three app servers. No additional verification command was required during the guided lab. For a simple manual check on each server, run:

```bash
sudo grep PermitRootLogin /etc/ssh/sshd_config
```

> **Why:** `grep` searches the SSH configuration file for the `PermitRootLogin` directive. The active line should read `PermitRootLogin no`; any line beginning with `#` is only a comment. This optional check was not run during the lab.

## Best Practices

- **Disable direct root SSH login.** Setting `PermitRootLogin no` prevents remote attackers from attempting to authenticate directly as the all-powerful `root` account.
- **Use named administrative accounts.** Administrators should connect with individual accounts such as `tony`, `steve`, or `banner`, then use `sudo` when elevated privileges are required. This improves accountability through user-specific authentication and logs.
- **Validate before reloading.** Always run `sshd -t` after editing `/etc/ssh/sshd_config`; a syntax error could prevent the daemon from loading the new configuration.
- **Reload instead of restarting when possible.** `systemctl reload sshd` applies the configuration without unnecessarily interrupting active sessions.
- **Apply security settings consistently.** Hardening only one app server leaves the remaining servers exposed, so the same policy must be applied to `stapp01`, `stapp02`, and `stapp03`.
- **Keep an existing administrative session open.** When changing SSH access remotely, retain the current session until the configuration has been validated and reloaded so a mistake does not immediately lock out the administrator.

### 📚 Official Documentation

- [Project Nautilus infrastructure](https://kodekloudhub.github.io/kodekloud-engineer/docs/projects/nautilus)
- [Red Hat — Using secure communications between two systems with OpenSSH](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/8/html/securing_networks/assembly_using-secure-communications-between-two-systems-with-openssh_securing-networks)
- [sshd_config(5) — OpenSSH daemon configuration file](https://man.openbsd.org/sshd_config)
- [sshd(8) — OpenSSH daemon](https://man.openbsd.org/sshd)
- [systemctl(1) — systemd system and service manager](https://man7.org/linux/man-pages/man1/systemctl.1.html)
