# Day 04: Script Execution Permissions

In a bid to automate backup processes, the xFusionCorp Industries sysadmin team has developed a new bash script named xfusioncorp.sh. While the script has been distributed to all necessary servers, it lacks executable permissions on App Server 1 within the Stratos Datacenter.

## Specific Requirements:

1. Grant executable permissions to the `/tmp/xfusioncorp.sh` script on App Server 1. Additionally, ensure that all users have the capability to execute it.

## Solution

The script is on App Server 1, `stapp01`, and is owned by `root`. The required permission mode is `755`: the owner gets read, write, and execute permissions, while the group and all other users get read and execute permissions. This allows every user to execute the script without allowing every user to modify it.

### Why a script needs both read and execute permissions

`xfusioncorp.sh` is a Bash script, which means it is a text file interpreted by a shell rather than a compiled binary. The execute permission allows a user to invoke the file, but the shell still needs to read the script's contents to process its commands. For direct execution, the operating system identifies the interpreter from the script's `#!` line and passes the script path to that interpreter; the interpreter then reads the file. If a script has execute permission but is not readable by the user, execution can fail even though the `x` bit is present.

That is why `755` is appropriate here: the owner receives `rwx`, while the group and all other users receive `r-x`. Every user can read and execute the script, but only the owner can write to it. A mode that grants only execute permission would not reliably run a text script, while `777` would give every user unnecessary write access. The Linux `execve` documentation describes how executable interpreter scripts are handed to their interpreter.

### 🔌 Step 1: Connect to App Server 1

```bash
ssh tony@stapp01
```

The lab connection succeeded and opened a shell as `tony`:

```text
thor@jump-host ~$ ssh tony@stapp01
tony@stapp01's password:
Last login: Wed Jul 22 04:03:04 2026 from 10.244.240.173
[tony@stapp01 ~]$
```

> **Why:** `ssh` opens an encrypted remote-login session. `tony` is the lab user for App Server 1, and `stapp01` is its hostname. Enter the lab password when prompted so the permission change is performed on the correct server.

### 🔍 Step 2: Attempt to change the permissions as the regular user

```bash
chmod 755 /tmp/xfusioncorp.sh
```

The first attempt failed because the file belongs to `root`:

```text
[tony@stapp01 ~]$ chmod 755 /tmp/xfusioncorp.sh
chmod: changing permissions of '/tmp/xfusioncorp.sh': Operation not permitted
[tony@stapp01 ~]$
```

> **Why:** `chmod` changes the access permissions of a file. The numeric mode `755` requests read, write, and execute for the owner, and read and execute for the group and other users. A regular user cannot change permissions on a file owned by `root`, so the system rejected this attempt with `Operation not permitted`.

### 🔐 Step 3: Grant executable permissions with administrative privileges

```bash
sudo chmod 755 /tmp/xfusioncorp.sh
```

The command succeeded after `sudo` supplied the required administrative privileges:

```text
[tony@stapp01 ~]$ sudo chmod 755 /tmp/xfusioncorp.sh

We trust you have received the usual lecture from the local System
Administrator. It usually boils down to these three things:

    #1) Respect the privacy of others.
    #2) Think before you type.
    #3) With great power comes great responsibility.

[sudo] password for tony:
[tony@stapp01 ~]$
```

> **Why:** `sudo` runs `chmod` with administrator privileges, allowing `tony` to modify a root-owned file without starting a full root shell. In `755`, the first digit `7` means read, write, and execute for the owner; each `5` means read and execute for the group and for all other users. The path `/tmp/xfusioncorp.sh` identifies the exact script required by the challenge.

### ✅ Step 4: Verify the permissions

```bash
ls -l /tmp/xfusioncorp.sh
```

The final output confirmed the required mode:

```text
[tony@stapp01 ~]$ ls -l /tmp/xfusioncorp.sh
-rwxr-xr-x 1 root root 40 Jul 22 03:55 /tmp/xfusioncorp.sh
[tony@stapp01 ~]$
```

> **Why:** `ls` lists information about files, and `-l` selects the long format so the permission bits, owner, group, size, timestamp, and path are displayed. The leading permission string `-rwxr-xr-x` means the owner can read, write, and execute; the group can read and execute; and all other users can read and execute. The `x` in all three permission groups confirms that every user can run the script.

## Best Practices

- **Use `755` for executable scripts that everyone must run.** The owner can modify the script, while other users can execute it without receiving write access.
- **Avoid `777` when it is unnecessary.** `777` would allow every user to modify the script, creating a security risk because a user could replace backup logic with malicious commands.
- **Use `sudo` only for the required operation.** Running only `chmod` with elevated privileges is safer than opening a complete root shell.
- **Check ownership when permissions changes fail.** The `Operation not permitted` message indicated that the file was owned by `root`, which explained why the regular-user command failed.
- **Verify the permission string after changes.** `ls -l` makes the effective read, write, and execute permissions visible and easy to compare with the requested mode.

### 📚 Official Documentation

- [Project Nautilus infrastructure](https://kodekloudhub.github.io/kodekloud-engineer/docs/projects/nautilus)
- [GNU Coreutils — `chmod` invocation](https://www.gnu.org/software/coreutils/manual/html_node/chmod-invocation.html)
- [GNU Coreutils — Numeric Modes](https://www.gnu.org/software/coreutils/manual/html_node/Numeric-Modes.html)
- [GNU Coreutils — `ls` invocation](https://www.gnu.org/software/coreutils/manual/html_node/ls-invocation.html)
- [`execve(2)` — execute program and interpreter scripts](https://www.man7.org/linux/man-pages/man2/execve.2.html)
