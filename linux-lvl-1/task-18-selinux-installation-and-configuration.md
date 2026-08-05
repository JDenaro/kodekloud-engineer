# Task 18: SELinux Installation and Configuration

Following a security audit, the xFusionCorp Industries security team has opted to enhance application and server security with SELinux. To initiate testing, the following requirements have been established for App server 1 in the Stratos Datacenter:

Install the required SELinux packages.

Permanently disable SELinux for the time being; it will be re-enabled after necessary configuration changes.

No need to reboot the server, as a scheduled maintenance reboot is already planned for tonight.

Disregard the current status of SELinux via the command line; the final status after the reboot should be disabled.

## Task Requirements

1. Install the required SELinux packages on App Server 1.
2. Permanently disable SELinux by configuring `/etc/selinux/config`.
3. Do not reboot App Server 1.
4. Ignore the current SELinux status; the disabled state must take effect after the scheduled reboot.

## Solution

SELinux is a mandatory access control system that can restrict how processes access files, ports, and other resources. The required policy packages are installed first, and then the persistent setting is changed in `/etc/selinux/config`.

The current runtime state is intentionally not changed. Setting `SELINUX=disabled` in the configuration file affects the next boot, so the server may continue to report its current SELinux state until the scheduled maintenance reboot.

### 🔌 Step 1: Connect to App Server 1

Run the command from the Jump Host:

```bash
ssh tony@stapp01
```

The session opened on App Server 1:

```text
thor@jump-host ~$ ssh tony@stapp01
[tony@stapp01 ~]$
```

> **Why:** `ssh` opens a secure remote shell. `tony` is the login account for App Server 1, and `stapp01` is the server where SELinux must be installed and configured.

### 📦 Step 2: Install the required SELinux packages

```bash
sudo yum install -y selinux-policy selinux-policy-targeted
```

The installation completed successfully:

```text
Installed:
  selinux-policy-38.1.83-1.el9.noarch
  selinux-policy-targeted-38.1.83-1.el9.noarch

Complete!
```

> **Why:** `sudo` provides the administrative privileges required to install system packages. `yum` is the package manager on this server. Its `install` subcommand adds packages, and `-y` automatically accepts the installation prompt.

The two requested packages have different roles:

- `selinux-policy` provides the base SELinux policy framework and common policy definitions.
- `selinux-policy-targeted` provides the targeted policy, which applies SELinux protection to selected system services and processes while leaving unselected processes less restricted. `targeted` is the standard policy type used by this lab's configuration.

The package manager also installed supporting dependencies such as `policycoreutils`, `checkpolicy`, `libselinux`, `libsemanage`, `setools`, and audit-related libraries. These packages provide tools and libraries used to manage, compile, inspect, and audit SELinux policies; they were installed automatically to satisfy the policy packages' dependencies.

### ✏️ Step 3: Disable SELinux permanently with vi

Open the SELinux configuration file:

```bash
sudo vi /etc/selinux/config
```

Set the configuration line to:

```text
SELINUX=disabled
```

If the file currently contains `SELINUX=enforcing` or `SELINUX=permissive`, replace that value with `disabled`. Save and exit `vi`:

1. Press `Esc` to leave insert mode.
2. Type `:wq`.
3. Press `Enter`.

The editor returned to the shell successfully:

```text
[tony@stapp01 ~]$ sudo vi /etc/selinux/config
[tony@stapp01 ~]$
```

> **Why:** `sudo` is required because `/etc/selinux/config` is a system-owned file. `vi` edits the file directly. The `SELINUX=disabled` setting tells the system not to load an SELinux policy during the next boot. `:wq` writes the change and exits the editor.

### ⏳ Step 4: Leave the current runtime state unchanged

Do not run a reboot command:

```text
No reboot performed.
```

> **Why:** The challenge explicitly schedules the reboot for later. A persistent configuration change in `/etc/selinux/config` does not necessarily change the current runtime state immediately. Therefore, the current output of commands such as `getenforce` is intentionally disregarded, and the disabled state will be applied after the planned reboot.

### ✅ Step 5: Confirm the persistent configuration

The configuration can be checked without evaluating the current runtime state:

```bash
sudo grep '^SELINUX=' /etc/selinux/config
```

Expected output:

```text
SELINUX=disabled
```

> **Why:** `grep` displays the line beginning with `SELINUX=` from the configuration file. The expected value confirms the persistent setting requested by the challenge. This is a file check only; it does not reboot the server or change the current SELinux mode.

## Best Practices

- **Distinguish policy packages from policy modes.** Installing `selinux-policy` and `selinux-policy-targeted` provides policy content; it does not by itself enable or disable the runtime mode.
- **Understand targeted policy.** The targeted policy protects selected services and processes instead of applying the same restrictions to every process.
- **Change persistent configuration deliberately.** `/etc/selinux/config` controls the SELinux state selected during boot.
- **Do not use `setenforce` for this requirement.** `setenforce` changes only the current enforcing or permissive runtime state and does not permanently disable SELinux.
- **Respect the maintenance window.** Do not reboot when the challenge explicitly says that a scheduled reboot will occur later.
- **Use the least disruptive change.** The lab requires the configuration to be prepared for the next boot while leaving the current server running.

### 📚 Official Documentation

- [Changing SELinux states and modes](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/using_selinux/changing-selinux-states-and-modes_using-selinux)
- [Using SELinux in Red Hat Enterprise Linux 9](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html-single/using_selinux/index)
