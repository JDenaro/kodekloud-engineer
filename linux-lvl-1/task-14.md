# Task 14: Default GUI Boot Configuration

With the installation of new tools on the app servers within the Stratos Datacenter, certain functionalities now necessitate graphical user interface (GUI) access.

Adjust the default runlevel on all App servers in Stratos Datacenter to enable GUI booting by default. It's imperative not to initiate a server reboot after completing this task.

## Task Requirements

1. Configure the default runlevel on all App servers in the Stratos Datacenter.
2. Enable GUI booting by default.
3. Do not reboot any server after completing the configuration.

## Solution

Modern Linux systems managed by `systemd` use targets to represent boot modes. `graphical.target` starts the services required for a graphical login. The `systemctl set-default` command updates the persistent default target by changing the `default.target` symlink; it does not immediately reboot or change the current session.

### 🖥️ Step 1: Configure GUI boot on App Server 1

Connect from the Jump Host and set the default target:

```bash
ssh tony@stapp01
sudo systemctl set-default graphical.target
exit
```

The configuration completed successfully:

```text
[tony@stapp01 ~]$ sudo systemctl set-default graphical.target
Removed "/etc/systemd/system/default.target".
Created symlink /etc/systemd/system/default.target → /usr/lib/systemd/system/graphical.target.
[tony@stapp01 ~]$ exit
logout
Connection to stapp01 closed.
```

> **Why:** `ssh` opens the remote shell on App Server 1. `sudo` supplies the administrative privileges required to change a system-wide boot setting. `systemctl` controls the `systemd` service manager, `set-default` changes the target used for the next boot, and `graphical.target` selects the graphical boot mode. `exit` returns to the Jump Host. The command changes the default symlink and does not reboot the server.

### 🖥️ Step 2: Configure GUI boot on App Server 2

```bash
ssh steve@stapp02
sudo systemctl set-default graphical.target
exit
```

The configuration completed successfully:

```text
[steve@stapp02 ~]$ sudo systemctl set-default graphical.target
Removed "/etc/systemd/system/default.target".
Created symlink /etc/systemd/system/default.target → /usr/lib/systemd/system/graphical.target.
[steve@stapp02 ~]$ exit
logout
Connection to stapp02 closed.
```

> **Why:** App Server 2 requires the same persistent boot target. The `steve` account is the designated login account for `stapp02`, while `graphical.target` ensures that the next normal boot selects the GUI environment. No `reboot` command is included because the challenge explicitly prohibits restarting the server.

### 🖥️ Step 3: Configure GUI boot on App Server 3

```bash
ssh banner@stapp03
sudo systemctl set-default graphical.target
exit
```

The configuration completed successfully:

```text
[banner@stapp03 ~]$ sudo systemctl set-default graphical.target
Removed "/etc/systemd/system/default.target".
Created symlink /etc/systemd/system/default.target → /usr/lib/systemd/system/graphical.target.
[banner@stapp03 ~]$
```

> **Why:** The final App Server is configured with the same `graphical.target` default. `banner` is the designated login account for `stapp03`. The `Created symlink` message confirms that `default.target` now points to the graphical target. The change will be used on a future boot and does not require a reboot during this lab.

### ✅ Step 4: Verify the configured default target

If a read-only confirmation is desired, run the following command on each App Server:

```bash
systemctl get-default
```

Expected output on each server:

```text
graphical.target
```

> **Why:** `systemctl get-default` displays the target selected as the persistent default boot target. `graphical.target` confirms that GUI booting is configured. This command only reads the setting and does not start, stop, or restart any service.

## Best Practices

- **Use the appropriate systemd target.** `graphical.target` is the standard target for booting into a graphical environment.
- **Change the default without rebooting.** `set-default` updates the future boot configuration while respecting the maintenance requirement not to restart the servers.
- **Apply the change consistently.** Configure every App Server because the requirement applies to the entire application-server group.
- **Read the command output.** The created symlink confirms that `default.target` now points to `graphical.target`.
- **Use a read-only verification when needed.** `systemctl get-default` confirms the target without changing the system state.

### 📚 Official Documentation

- [systemctl(1) Linux manual page](https://man7.org/linux/man-pages/man1/systemctl.1.html)
