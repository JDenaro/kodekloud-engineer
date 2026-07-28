# Task 07: Secure Root SSH Access

Following security audits, the xFusionCorp Industries security team has rolled out new protocols, including the restriction of direct root SSH login.

Your task is to disable direct SSH root login on all app servers within the Stratos Datacenter.

## Task Requirements

1. Disable direct SSH login for the `root` account.
2. Apply the configuration on all three App Servers: `stapp01`, `stapp02`, and `stapp03`.
3. Validate the SSH configuration before reloading the service.

## Solution

OpenSSH reads server settings from `/etc/ssh/sshd_config`. Setting `PermitRootLogin no` prevents clients from authenticating directly as `root` through SSH. Administrators can still connect with their named accounts and use `sudo` when elevated privileges are required.

The lab was completed by editing the configuration manually with `vi`. A non-interactive `sed` command is documented afterward as an alternative; it replaces the `vi` editing step and should not be run in addition to it.

### 🖥️ Step 1: Configure App Server 1 with vi

Connect to App Server 1:

```bash
ssh tony@stapp01
```

Open the SSH daemon configuration file:

```bash
sudo vi /etc/ssh/sshd_config
```

Inside `vi`, use these actions:

1. Type `/PermitRootLogin` and press `Enter` to locate the setting.
2. Change the active setting to `PermitRootLogin no`.
3. Press `Esc`, type `:wq`, and press `Enter` to save and close the file.

The relevant configuration block must look like this:

```text
#LoginGraceTime 2m
PermitRootLogin no
#StrictModes yes
#MaxAuthTries 6
#MaxSessions 10
```

> **Why:** `sudo` provides the privileges needed to modify a root-owned system configuration file. `vi` is a terminal text editor, and `/etc/ssh/sshd_config` is the OpenSSH server configuration file. Lines beginning with `#` are comments and do not define active settings. The uncommented `PermitRootLogin no` line instructs `sshd`, the SSH server daemon, to refuse direct SSH authentication for `root`. In `vi`, `/PermitRootLogin` searches for the setting, `Esc` returns to normal mode, and `:wq` writes the changes and quits.

Validate the file, reload SSH, and return to the jump host:

```bash
sudo sshd -t
sudo systemctl reload sshd
exit
```

The commands completed without an error:

```text
[tony@stapp01 ~]$ sudo sshd -t
[tony@stapp01 ~]$ sudo systemctl reload sshd
[tony@stapp01 ~]$ exit
logout
Connection to stapp01 closed.
thor@jump-host ~$
```

> **Why:** `sshd -t` checks the validity of the configuration and the sanity of the server keys without starting another daemon. Successful validation produces no output. `systemctl` controls services managed by `systemd`; `reload sshd` asks the running SSH daemon to reread its configuration without a full restart. The reload is performed only after the syntax check succeeds. `exit` closes the remote session and returns to the jump host.

### 🖥️ Step 2: Configure App Server 2 with vi

Connect to App Server 2 and open the same configuration file:

```bash
ssh steve@stapp02
sudo vi /etc/ssh/sshd_config
```

Locate `PermitRootLogin`, set the active line to the following value, and save with `Esc`, `:wq`, and `Enter`:

```text
PermitRootLogin no
```

Validate and apply the configuration, then return to the jump host:

```bash
sudo sshd -t
sudo systemctl reload sshd
exit
```

The commands completed successfully:

```text
[steve@stapp02 ~]$ sudo sshd -t
[steve@stapp02 ~]$ sudo systemctl reload sshd
[steve@stapp02 ~]$ exit
logout
Connection to stapp02 closed.
thor@jump-host ~$
```

> **Why:** App Server 2 has its own `/etc/ssh/sshd_config` and running `sshd` service, so the change made on App Server 1 does not affect it. The same edit, syntax validation, and reload are required locally on `stapp02`.

### 🖥️ Step 3: Configure App Server 3 with vi

Connect to App Server 3 and open the configuration:

```bash
ssh banner@stapp03
sudo vi /etc/ssh/sshd_config
```

Set the active directive to:

```text
PermitRootLogin no
```

Save the file, validate it, reload SSH, and exit:

```bash
sudo sshd -t
sudo systemctl reload sshd
exit
```

The final server completed the same sequence successfully:

```text
[banner@stapp03 ~]$ sudo sshd -t
[banner@stapp03 ~]$ sudo systemctl reload sshd
[banner@stapp03 ~]$ exit
logout
Connection to stapp03 closed.
thor@jump-host ~$
```

> **Why:** The security requirement covers every App Server in the Stratos Datacenter. Applying and reloading the setting on `stapp03` completes the three-server scope.

### 🛠️ Step 4: Alternative editing method with sed

Instead of opening `vi`, the active setting can be replaced directly with `sed` on each App Server:

```bash
sudo sed -i 's/^PermitRootLogin yes$/PermitRootLogin no/' /etc/ssh/sshd_config
```

After using this alternative, the configuration must still be validated and reloaded:

```bash
sudo sshd -t
sudo systemctl reload sshd
```

> **Why:** `sed` is a stream editor, and `-i` edits the specified file in place. The substitution has the form `s/current/replacement/`. The `^` symbol requires the match to begin at the start of the line, while `$` requires it to end after `yes`; this targets the active `PermitRootLogin yes` directive and avoids commented examples. The replacement writes `PermitRootLogin no`. This is an alternative to the manual `vi` edit, not an additional required change. Testing with `sshd -t` before reloading remains essential regardless of how the file was edited.

### ✅ Step 5: Verify the configured directive

On each App Server, the active line can be displayed with:

```bash
sudo grep '^PermitRootLogin' /etc/ssh/sshd_config
```

The expected output is:

```text
PermitRootLogin no
```

> **Why:** `grep` prints lines that match a pattern. The leading `^` restricts the match to lines that begin with `PermitRootLogin`, excluding commented lines such as `#PermitRootLogin`. Seeing `PermitRootLogin no` confirms the stored directive, while the successful `sshd -t` and `systemctl reload sshd` commands confirm that the configuration is valid and was applied to the running service.

## Best Practices

- **Use named administrator accounts.** Connecting with an individual account and elevating with `sudo` provides better accountability than sharing direct root access.
- **Validate before reloading.** `sshd -t` reduces the risk of applying a malformed configuration that could prevent future SSH connections.
- **Reload instead of restarting when supported.** A reload applies the updated configuration without performing a full service restart.
- **Change every server in scope.** Security settings stored locally must be updated separately on `stapp01`, `stapp02`, and `stapp03`.
- **Use one editing method.** Choose either `vi` or `sed`, then validate and reload; running both editing methods is unnecessary.
- **Keep the current session until validation succeeds.** Do not close the active SSH connection before confirming that the configuration syntax is valid and the reload completes.

### 📚 Official Documentation

- [OpenSSH sshd_config(5) manual page](https://man.openbsd.org/sshd_config)
- [OpenSSH sshd(8) manual page](https://man.openbsd.org/sshd)
- [systemctl(1) Linux manual page](https://man7.org/linux/man-pages/man1/systemctl.1.html)
- [Vim user manual: Getting Started](https://vimhelp.org/usr_01.txt.html)
- [GNU sed manual](https://www.gnu.org/software/sed/manual/sed.html)
- [grep(1) Linux manual page](https://man7.org/linux/man-pages/man1/grep.1.html)
