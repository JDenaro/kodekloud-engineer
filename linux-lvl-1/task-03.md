# Task 03: Linux User Setup with Non-Interactive Shell

To accommodate the backup agent tool's specifications, the system admin team at xFusionCorp Industries requires the creation of a user with a non-interactive shell. Here's your task:

Create a user named `javed` with a non-interactive shell on App Server 3.

Note: You can find the infrastructure details by clicking on the **Details of all Users and Servers** button on the top-right section of the page.

## Task Requirements

1. Create a user named `javed` on App Server 3.
2. Configure the account with a non-interactive shell.

## Solution

The `javed` account was created on App Server 3 with `/sbin/nologin` as its login shell. This allows the operating system and automated tools to use the account while preventing it from opening an interactive shell session.

Because the task explicitly requires creating the user, no pre-creation lookup is necessary. The account is verified after creation to confirm that the requested shell was stored correctly.

### 🔌 Step 1: Connect to App Server 3

From the jump host, connect to App Server 3 as `banner`:

```bash
ssh banner@stapp03
```

The SSH connection opened a shell on `stapp03`:

```text
thor@jump-host ~$ ssh banner@stapp03
banner@stapp03's password:
[banner@stapp03 ~]$
```

> **Why:** `ssh` opens a secure remote shell session. `banner` is the login account for App Server 3, and `stapp03` is the target hostname. The user must be created on the specified App Server rather than on the jump host.

### 👤 Step 2: Create the user with a non-interactive shell

Create `javed` and assign `/sbin/nologin` as the account's shell:

```bash
sudo useradd -s /sbin/nologin javed
```

The command returned to the prompt without an error:

```text
[banner@stapp03 ~]$ sudo useradd -s /sbin/nologin javed
[banner@stapp03 ~]$
```

> **Why:** `sudo` runs the command with the administrative privileges needed to create a system account. `useradd` creates the local user, and the final `javed` argument supplies the login name. The `-s` option sets the user's login shell to the following value, `/sbin/nologin`. When the system attempts to start that shell for an interactive login, `nologin` refuses access and exits. This is appropriate for an account intended for an automated backup agent rather than a human login session.

### ✅ Step 3: Verify the account and shell

Read the account entry from the system user database:

```bash
getent passwd javed
```

The command returned:

```text
[banner@stapp03 ~]$ getent passwd javed
javed:x:1001:1001::/home/javed:/sbin/nologin
[banner@stapp03 ~]$
```

> **Why:** `getent` queries system databases, `passwd` selects the user-account database, and `javed` selects the specific account. The colon-separated fields contain the username, password placeholder, UID, primary GID, comment, home directory, and login shell. The final field is `/sbin/nologin`, confirming that the account has the required non-interactive shell. The automatically assigned UID and GID do not need specific values because the task does not define them.

## Best Practices

- **Use non-interactive shells for service accounts.** Accounts used by automated tools usually do not need interactive terminal access.
- **Follow the requested server scope.** Creating the account on `stapp03` ensures that the change is applied only to App Server 3.
- **Set the shell during creation.** Using `useradd -s` applies the intended access restriction as soon as the account is created.
- **Verify the stored account entry.** `getent passwd javed` confirms both that the user exists and that `/sbin/nologin` is configured as the shell.
- **Avoid unnecessary pre-checks for explicit creation tasks.** When the challenge directly instructs us to create a resource, we treat it as absent unless the challenge states otherwise.

### 📚 Official Documentation

- [useradd(8) Linux manual page](https://man7.org/linux/man-pages/man8/useradd.8.html)
- [nologin(8) Linux manual page](https://man7.org/linux/man-pages/man8/nologin.8.html)
- [getent(1) Linux manual page](https://man7.org/linux/man-pages/man1/getent.1.html)
