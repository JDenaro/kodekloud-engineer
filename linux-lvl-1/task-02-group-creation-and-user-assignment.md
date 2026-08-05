# Task 02: Group Creation and User Assignment

The system admin team at xFusionCorp Industries has streamlined access management by implementing group-based access control. Here's what you need to do:

a. Create a group named `nautilus_admin_users` across all App servers within the Stratos Datacenter.

b. Add the user `stark` into the `nautilus_admin_users` group on all App servers. If the user doesn't exist, create it as well.

Note: You can find the infrastructure details by clicking on the **Details of all Users and Servers** button on the top-right section of the page.

## Task Requirements

1. Create the group `nautilus_admin_users` on all App Servers in the Stratos Datacenter.
2. Check whether the user `stark` exists on each App Server.
3. Create `stark` when the account is missing.
4. Add `stark` to `nautilus_admin_users` on every App Server.

## Solution

Linux users and groups configured locally on one server are not automatically available on another server. Therefore, the group and user membership must be configured separately on `stapp01`, `stapp02`, and `stapp03`.

The user lookup is important because the task explicitly says to create `stark` only when the account does not exist. `getent passwd stark` provides a simple decision point:

- If it returns an account entry, use `usermod -aG nautilus_admin_users stark` to add the existing user to the group.
- If it returns no output, use `useradd -G nautilus_admin_users stark` to create the user and assign the supplementary group at the same time.

### 🖥️ Step 1: Configure App Server 1

Connect to App Server 1:

```bash
ssh tony@stapp01
```

The connection opened a shell on `stapp01`:

```text
thor@jump-host ~$ ssh tony@stapp01
tony@stapp01's password:
[tony@stapp01 ~]$
```

> **Why:** `ssh` opens a secure remote shell session. `tony` is the login account for App Server 1, and `stapp01` is the server hostname. The commands must run on the target server because local account databases are managed independently on each App Server.

Create the required group:

```bash
sudo groupadd nautilus_admin_users
```

The command returned to the prompt without an error:

```text
[tony@stapp01 ~]$ sudo groupadd nautilus_admin_users
[tony@stapp01 ~]$
```

> **Why:** `sudo` provides the administrative privileges required to modify system accounts. `groupadd` creates a new local group account, and `nautilus_admin_users` is the group name required by the task. A command that returns silently to the prompt has completed successfully.

Check whether `stark` already exists:

```bash
getent passwd stark
```

> **Why:** `getent` reads entries from system databases. The `passwd` database contains user-account records, and `stark` limits the lookup to that username. An account record means the user exists; no output means the user must be created.

If the lookup returns no output, create `stark` and add it to the group:

```bash
sudo useradd -G nautilus_admin_users stark
```

The creation command completed successfully during the lab:

```text
[tony@stapp01 ~]$ sudo useradd -G nautilus_admin_users stark
[tony@stapp01 ~]$
```

If the lookup returns an existing account instead, use this command rather than `useradd`:

```bash
sudo usermod -aG nautilus_admin_users stark
```

> **Why:** `useradd` creates a new account, and its `-G` option assigns a comma-separated list of supplementary groups to that new user. For an existing account, `usermod` changes the account instead. With `usermod`, `-G` selects the supplementary group list and `-a` appends the new group without removing the user's current supplementary memberships. Only one of these two commands is needed, according to the result returned by `getent`.

Return to the jump host:

```bash
exit
```

```text
[tony@stapp01 ~]$ exit
logout
Connection to stapp01 closed.
thor@jump-host ~$
```

> **Why:** `exit` closes the current remote shell and returns to `thor@jump-host`, from where the next App Server can be reached.

### 🖥️ Step 2: Configure App Server 2

Connect to App Server 2:

```bash
ssh steve@stapp02
```

```text
thor@jump-host ~$ ssh steve@stapp02
steve@stapp02's password:
[steve@stapp02 ~]$
```

Create the group:

```bash
sudo groupadd nautilus_admin_users
```

```text
[steve@stapp02 ~]$ sudo groupadd nautilus_admin_users
[steve@stapp02 ~]$
```

Check for the user before choosing the account-management command:

```bash
getent passwd stark
```

If no account entry is returned, create the user with the required supplementary group:

```bash
sudo useradd -G nautilus_admin_users stark
```

The command completed without an error during the lab:

```text
[steve@stapp02 ~]$ sudo useradd -G nautilus_admin_users stark
[steve@stapp02 ~]$
```

If an account entry is returned, add the existing user instead:

```bash
sudo usermod -aG nautilus_admin_users stark
```

> **Why:** App Server 2 has its own local user and group databases, so the configuration from `stapp01` does not satisfy the requirement on `stapp02`. The same `getent` decision is made here: `useradd -G` handles a missing account, while `usermod -aG` safely adds an existing account to the new group.

Return to the jump host:

```bash
exit
```

```text
[steve@stapp02 ~]$ exit
logout
Connection to stapp02 closed.
thor@jump-host ~$
```

### 🖥️ Step 3: Configure App Server 3

Connect to App Server 3:

```bash
ssh banner@stapp03
```

```text
thor@jump-host ~$ ssh banner@stapp03
banner@stapp03's password:
[banner@stapp03 ~]$
```

Create the group:

```bash
sudo groupadd nautilus_admin_users
```

```text
[banner@stapp03 ~]$ sudo groupadd nautilus_admin_users
[banner@stapp03 ~]$
```

Check whether `stark` exists:

```bash
getent passwd stark
```

If the lookup produces no output, create the account and assign the supplementary group:

```bash
sudo useradd -G nautilus_admin_users stark
```

The correct creation command completed without an error:

```text
[banner@stapp03 ~]$ sudo useradd -G nautilus_admin_users stark
[banner@stapp03 ~]$
```

If the account already exists, use the modification command instead:

```bash
sudo usermod -aG nautilus_admin_users stark
```

> **Why:** The same configuration must be applied on App Server 3. Checking first prevents an unnecessary attempt to recreate an existing account and determines whether `useradd` or `usermod` is appropriate.

### ✅ Step 4: Verify the user and group membership on every server

On each App Server, confirm that the account exists:

```bash
getent passwd stark
```

The App Server 3 verification returned:

```text
[banner@stapp03 ~]$ getent passwd stark
stark:x:1002:1003::/home/stark:/bin/bash
[banner@stapp03 ~]$
```

The account record confirms the username, numeric UID, primary GID, home directory, and login shell. The numeric IDs may differ between servers.

Then confirm that the user belongs to the required supplementary group:

```bash
id stark
```

The output must include `nautilus_admin_users` in the `groups` list. Run both verification commands on `stapp01`, `stapp02`, and `stapp03` before leaving each server.

> **Why:** `getent passwd stark` proves that the account exists in the server's user database. `id stark` displays the user's UID, primary group, and all supplementary groups, making it possible to confirm that `nautilus_admin_users` was assigned successfully. The actual numeric IDs are allocated independently by each server, so the group name is the important value to verify.

After verifying App Server 3, return to the jump host:

```bash
exit
```

```text
[banner@stapp03 ~]$ exit
logout
Connection to stapp03 closed.
thor@jump-host ~$
```

## Best Practices

- **Check before creating an account.** `getent passwd stark` determines whether account creation or account modification is appropriate.
- **Preserve existing group memberships.** Use `usermod -aG` for an existing user because `-a` appends the new group instead of replacing the current supplementary group list.
- **Configure every target independently.** Local users and groups must be created or modified separately on all three App Servers.
- **Verify both existence and membership.** `getent` confirms the account record, while `id` confirms the effective group assignments.
- **Use the exact requested names.** Linux treats different spellings as different users and groups, so `stark` and `nautilus_admin_users` must be entered exactly.

### 📚 Official Documentation

- [groupadd(8) Linux manual page](https://man7.org/linux/man-pages/man8/groupadd.8.html)
- [useradd(8) Linux manual page](https://man7.org/linux/man-pages/man8/useradd.8.html)
- [usermod(8) Linux manual page](https://man7.org/linux/man-pages/man8/usermod.8.html)
- [getent(1) Linux manual page](https://man7.org/linux/man-pages/man1/getent.1.html)
- [id(1) Linux manual page](https://man7.org/linux/man-pages/man1/id.1.html)
