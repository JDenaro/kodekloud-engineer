# Task 01: Custom Apache User Setup

In response to heightened security concerns, the xFusionCorp Industries security team has opted for custom Apache users for their web applications. Each user is tailored specifically for an application, enhancing security measures. Your task is to create a custom Apache user according to the outlined specifications:

a. Create a user named `jim` on App server 1 within the Stratos Datacenter.

b. Assign a unique UID `1310` and designate the home directory as `/var/www/jim`.

Note: You can find the infrastructure details by clicking on the **Details of all Users and Servers** button on the top-right section of the page.

## Task Requirements

1. Create a user named `jim` on App server 1 within the Stratos Datacenter.
2. Assign the user the unique UID `1310` and the home directory `/var/www/jim`.

## Solution

The account was created on App Server 1 with the requested username, UID, and home-directory path.

### 🔌 Step 1: Connect to App Server 1

From the jump host, connect to App Server 1 as `tony`:

```bash
ssh tony@stapp01
```

The connection opened a shell on `stapp01`:

```text
thor@jump-host ~$ ssh tony@stapp01
tony@stapp01's password:
[tony@stapp01 ~]$
```

> **Why:** `ssh` starts a secure remote shell session. `tony` is the account used to access the server, and `stapp01` is the hostname of App Server 1. The user must run the account-creation command on the target server, not on the jump host.

### 👤 Step 2: Create the user with the requested attributes

```bash
sudo useradd -u 1310 -d /var/www/jim jim
```

The command completed without an error:

```text
[tony@stapp01 ~]$ sudo useradd -u 1310 -d /var/www/jim jim
[tony@stapp01 ~]$
```

> **Why:** `-u 1310` assigns the requested numeric user ID (UID). `-d /var/www/jim` sets the account's home-directory path. The final `jim` argument supplies the required login name. The `-d` option sets the path stored for the account; it does not create the physical directory unless a separate home-directory creation option is used. No password was requested because the challenge only requires the account and its attributes.

### ✅ Step 3: Verify the account

Use `getent` to read the `passwd` account database entry for `jim`:

```bash
getent passwd jim
```

The command returned:

```text
[tony@stapp01 ~]$ getent passwd jim
jim:x:1310:1310::/var/www/jim:/bin/bash
[tony@stapp01 ~]$
```

The output confirms the required values:

- `jim` is the username.
- `1310` is the UID.
- `/var/www/jim` is the home-directory path.

The account also returned the following identity information when checked with `id`:

```text
[tony@stapp01 ~]$ id jim
uid=1310(jim) gid=1310(jim) groups=1310(jim)
[tony@stapp01 ~]$
```

> **Why:** `getent passwd jim` queries the system's account database and displays the complete `passwd` record for `jim`. The fields are separated by colons: username, password placeholder, UID, primary group ID, comment, home directory, and login shell. This single command is sufficient for this task because it confirms both the UID and the configured home-directory path. `id jim` is an additional identity check that shows the UID, primary group, and group membership.

## Best Practices

- **Use unique UIDs.** A unique numeric UID prevents the new account from sharing an identity with another local user.
- **Set an explicit home-directory path.** The requested application-specific path makes the account configuration clear and predictable.
- **Verify the account database entry.** `getent passwd jim` confirms the values stored by the system without requiring advanced filtering commands.
- **Separate account configuration from directory creation.** Setting `/var/www/jim` as the home path does not automatically create that directory with the command used in this lab.

### 📚 Official Documentation

- [useradd(8) Linux manual page](https://man7.org/linux/man-pages/man8/useradd.8.html)
- [getent(1) Linux manual page](https://man7.org/linux/man-pages/man1/getent.1.html)
