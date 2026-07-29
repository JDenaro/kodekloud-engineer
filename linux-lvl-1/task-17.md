# Task 17: Process Limit Adjustment

In the Stratos Datacenter, our application server 1 is encountering performance degradation due to excessive processes held by the nfsuser user. To mitigate this issue, we need to enforce limitations on its maximum processes. Please set the maximum process limits as specified below:

a. Set the soft limit to 1027

b. Set the hard limit to 2024

## Task Requirements

1. Configure the process limits for user `nfsuser` on App Server 1.
2. Set the soft `nproc` limit to `1027`.
3. Set the hard `nproc` limit to `2024`.

## Solution

User resource limits are configured in `/etc/security/limits.conf`. The `nproc` item controls the maximum number of processes, `soft` defines the initial active limit, and `hard` defines the maximum value to which the user may raise the soft limit. The limits are applied when the user starts a new login session.

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

> **Why:** `ssh` opens a secure remote shell. `tony` is the login account for App Server 1, and `stapp01` is the target server where the process limits must be configured.

### ✏️ Step 2: Edit the user limits with vi

Open the limits configuration file:

```bash
sudo vi /etc/security/limits.conf
```

Add these two lines to the file:

```text
nfsuser soft nproc 1027
nfsuser hard nproc 2024
```

Save and exit `vi`:

1. Press `Esc` to leave insert mode.
2. Type `:wq`.
3. Press `Enter`.

The editor returned to the shell successfully:

```text
[tony@stapp01 ~]$ sudo vi /etc/security/limits.conf
[tony@stapp01 ~]$
```

> **Why:** `sudo` provides the administrative privileges required to modify a file under `/etc`. `vi` opens the file for manual editing. Each limits line follows the format `<domain> <type> <item> <value>`: `nfsuser` is the user, `soft` or `hard` identifies the limit type, `nproc` selects the maximum process count, and the final number is the requested value. `:wq` writes the changes and quits the editor.

The resulting policy gives `nfsuser` an active soft limit of `1027` processes. The user may move that soft limit up or down within the hard limit, but cannot raise it above `2024` without administrative privileges. The hard limit is enforced by the kernel.

### Increasing the soft limit temporarily

After starting a new session, `nfsuser` begins with a process limit of `1027`. If the workload requires more processes, the user can raise the active soft limit up to the hard limit with `ulimit`:

```bash
ulimit -u 2024
```

The user can also lower the active limit:

```bash
ulimit -u 500
```

However, this attempt would fail for a normal user because it exceeds the hard limit:

```bash
ulimit -u 3000
```

> **Why:** `ulimit` changes resource limits for the current shell and processes started from it. The `-u` option selects the maximum number of user processes, which corresponds to `nproc`. A user may raise the soft limit only up to the existing hard limit. Raising the hard limit itself requires administrative privileges, and changes made with `ulimit` do not replace the persistent values in `/etc/security/limits.conf`.

### ✅ Step 3: Verify the configured limits

Display the lines for `nfsuser`:

```bash
sudo grep nfsuser /etc/security/limits.conf
```

Expected output:

```text
nfsuser soft nproc 1027
nfsuser hard nproc 2024
```

> **Why:** `grep` displays lines matching the supplied username. The result confirms that both the soft and hard `nproc` limits were written for `nfsuser`. The limits will be loaded for a new login session; they do not retroactively change the limits of an already-running session.

## Best Practices

- **Use a soft limit for normal operation.** The soft limit is the initial active process limit applied to the user.
- **Set a higher hard ceiling deliberately.** The hard limit defines the maximum value the user can request for the soft limit.
- **Raise the soft limit only when necessary.** A user can temporarily raise the active soft limit with `ulimit -u`, but never above the configured hard limit.
- **Use the correct resource item.** `nproc` limits the maximum number of processes, not CPU, memory, or disk usage.
- **Apply limits to the correct user.** The domain `nfsuser` makes the rule specific to the affected account.
- **Start a new session after changing limits.** Settings in `/etc/security/limits.conf` are applied by PAM during login, so an existing session may retain its previous values.
- **Keep the limits persistent.** `/etc/security/limits.conf` provides a persistent configuration instead of changing only one shell with `ulimit`.

### 📚 Official Documentation

- [limits.conf(5) Linux manual page](https://man7.org/linux/man-pages/man5/limits.conf.5.html)
