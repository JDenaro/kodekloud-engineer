# Task 04: Service User Creation without Home Directory

In response to the latest tool implementation at xFusionCorp Industries, the system admins require the creation of a service user account. Here are the specifics:

Create a user named `javed` in App Server 2 without a home directory.

Note: You can find the infrastructure details by clicking on the **Details of all Users and Servers** button on the top-right section of the page.

## Task Requirements

1. Create a user named `javed` on App Server 2.
2. Do not create a home directory for the user.

## Solution

The `javed` service account was created on App Server 2 with the `useradd -M` option. The account still has `/home/javed` recorded as its default login-directory path, but the physical directory was not created.

Because the task explicitly requires creating the user, no pre-creation lookup is needed. The account and the absence of its home directory are verified after creation.

### 🔌 Step 1: Connect to App Server 2

From the jump host, connect to App Server 2 as `steve`:

```bash
ssh steve@stapp02
```

The SSH connection opened a shell on `stapp02`:

```text
thor@jump-host ~$ ssh steve@stapp02
steve@stapp02's password:
[steve@stapp02 ~]$
```

> **Why:** `ssh` opens a secure remote shell session. `steve` is the login account for App Server 2, and `stapp02` is the target hostname. The account must be created on the server specified by the challenge rather than on the jump host.

### 👤 Step 2: Create the user without a home directory

Create `javed` while preventing home-directory creation:

```bash
sudo useradd -M javed
```

The command returned to the prompt without an error:

```text
[steve@stapp02 ~]$ sudo useradd -M javed
[steve@stapp02 ~]$
```

> **Why:** `sudo` provides the administrative privileges needed to create a local account. `useradd` creates the user, and the final `javed` argument supplies the login name. The `-M` option means "do not create the user's home directory," even when the server's global account defaults would normally create one. This is useful for a service account that does not need personal files or an interactive working directory.

### ✅ Step 3: Verify the account and directory

Confirm that the account exists:

```bash
getent passwd javed
```

The command returned:

```text
[steve@stapp02 ~]$ getent passwd javed
javed:x:1001:1001::/home/javed:/bin/bash
[steve@stapp02 ~]$
```

> **Why:** `getent` reads entries from system databases, `passwd` selects the user-account database, and `javed` selects the account. The returned record confirms that the user exists. `/home/javed` is still stored in the account's home-directory field because `-M` controls directory creation; it does not remove or empty the configured path.

Check whether the physical home directory exists:

```bash
ls -ld /home/javed
```

The server reported that the path does not exist:

```text
[steve@stapp02 ~]$ ls -ld /home/javed
ls: cannot access '/home/javed': No such file or directory
[steve@stapp02 ~]$
```

> **Why:** `ls` displays information about filesystem paths. The `-l` option requests the long format, while `-d` makes `ls` inspect the directory entry itself instead of listing its contents. The `No such file or directory` result is expected in this verification and confirms that `/home/javed` was not physically created. It is evidence that the `-M` requirement was satisfied, not a failed account-creation command.

## Best Practices

- **Avoid unnecessary home directories for service accounts.** This reduces unused filesystem content and keeps the account aligned with its limited purpose.
- **Distinguish account metadata from filesystem objects.** A home path can appear in the user database even when the corresponding directory does not exist.
- **Set the behavior during account creation.** `useradd -M` makes the no-home-directory requirement explicit and independent of server defaults.
- **Verify both parts of the requirement.** `getent` confirms the account, while `ls -ld` confirms that the physical home directory is absent.
- **Treat expected negative checks correctly.** A missing `/home/javed` path is the successful result required by this challenge.

### 📚 Official Documentation

- [useradd(8) Linux manual page](https://man7.org/linux/man-pages/man8/useradd.8.html)
- [getent(1) Linux manual page](https://man7.org/linux/man-pages/man1/getent.1.html)
- [ls(1) Linux manual page](https://man7.org/linux/man-pages/man1/ls.1.html)
