# Task 09: Script Execution Permissions

In a bid to automate backup processes, the xFusionCorp Industries sysadmin team has developed a new bash script named `xfusioncorp.sh`. While the script has been distributed to all necessary servers, it lacks executable permissions on App Server 3 within the Stratos Datacenter.

Your task is to grant executable permissions to the `/tmp/xfusioncorp.sh` script on App Server 3. Additionally, ensure that all users have the capability to execute it.

## Task Requirements

1. Work on App Server 3.
2. Update the permissions of `/tmp/xfusioncorp.sh`.
3. Give every user the read and execute permissions needed to run the Bash script.

## Solution

The script is owned by `root`, so administrative privileges are required to modify it. The symbolic mode `a+rx` adds read and execute permissions for the owner, group, and all other users without granting write access.

A Bash script needs to be readable by the interpreter as well as executable. Giving every permission class both `r` and `x` ensures that all users can run the script directly.

### 🔌 Step 1: Connect to App Server 3

From the jump host, connect to App Server 3 as `banner`:

```bash
ssh banner@stapp03
```

The connection opened a shell on `stapp03`:

```text
thor@jump-host ~$ ssh banner@stapp03
banner@stapp03's password:
[banner@stapp03 ~]$
```

> **Why:** `ssh` opens a secure remote shell session. `banner` is the login account for App Server 3, and `stapp03` is the target hostname. The permission change must be applied to the script on the server specified by the challenge.

### 🔐 Step 2: Grant read and execute permissions to all users

Add the required permissions:

```bash
sudo chmod a+rx /tmp/xfusioncorp.sh
```

The command returned to the prompt without an error:

```text
[banner@stapp03 ~]$ sudo chmod a+rx /tmp/xfusioncorp.sh
[banner@stapp03 ~]$
```

> **Why:** `sudo` provides the administrative privileges required to modify a file owned by `root`. `chmod` changes file mode bits. In the symbolic mode `a+rx`, `a` selects all permission classes: the file owner, members of its group, and other users. The `+` operator adds permissions without removing existing mode bits, `r` grants read access, and `x` grants execute access. `/tmp/xfusioncorp.sh` is the target script. Read access allows the Bash interpreter to process the script contents, while execute access permits direct execution.

### ✅ Step 3: Verify the resulting permissions

Display the script's long file listing:

```bash
ls -l /tmp/xfusioncorp.sh
```

The command returned:

```text
[banner@stapp03 ~]$ ls -l /tmp/xfusioncorp.sh
-r-xr-xr-x 1 root root 40 Jul 28 22:12 /tmp/xfusioncorp.sh
[banner@stapp03 ~]$
```

The permission string can be read as:

```text
-  r-x  r-x  r-x
   owner group others
```

> **Why:** `ls` displays filesystem entries, and `-l` selects the long format containing permissions, ownership, size, and timestamp. The first `-` identifies a regular file. Each `r-x` block grants read and execute access without write access. Because `r-x` appears for the owner, group, and others, every user can read and execute the script as required. The owner, size, and timestamp provide context but are not values that the challenge asks us to change.

## Best Practices

- **Grant only the permissions required.** `a+rx` enables script execution without unnecessarily adding write access.
- **Account for interpreted scripts.** Shell scripts must be readable by their interpreter in addition to having the execute bit.
- **Use symbolic modes for targeted changes.** The `+` operator adds only the named permissions and preserves unrelated existing mode bits.
- **Use elevated privileges for protected files.** A non-owner should use `sudo chmod` when changing a root-owned script.
- **Verify all three permission classes.** The owner, group, and others must each show `r-x` for universal execution capability.

### 📚 Official Documentation

- [chmod(1) Linux manual page](https://man7.org/linux/man-pages/man1/chmod.1.html)
- [ls(1) Linux manual page](https://man7.org/linux/man-pages/man1/ls.1.html)
