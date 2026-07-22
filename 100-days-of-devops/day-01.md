# Day 01: Linux User Setup with Non-Interactive Shell

To accommodate the backup agent tool's specifications, the system admin team at xFusionCorp Industries requires the creation of a user with a non-interactive shell. Here's your task:

## Specific Requirements:

1. Create a user named `mariyam` with a non-interactive shell on App Server 2.

## Solution

The work happens on App Server 2, `stapp02`. The account is created with `/sbin/nologin`, a shell that refuses interactive logins while still allowing the account to exist for tools or services that need a dedicated user identity.

### 🔌 Step 1: Connect to App Server 2

```bash
ssh steve@stapp02
```

> **Why:** `ssh` opens an encrypted remote-login session. `steve` is the user provided for App Server 2, and `stapp02` is that server's hostname in the Stratos Datacenter. Run the remaining commands after the prompt changes to the `steve@stapp02` session. When prompted, enter the lab password `Am3ric@`.

### 👤 Step 2: Create `mariyam` with a non-interactive shell

```bash
sudo useradd -s /sbin/nologin mariyam
```

The command completed successfully in the lab and returned to the prompt without an error:

```text
[steve@stapp02 ~]$ sudo useradd -s /sbin/nologin mariyam

We trust you have received the usual lecture from the local System
Administrator. It usually boils down to these three things:

    #1) Respect the privacy of others.
    #2) Think before you type.
    #3) With great power comes great responsibility.

[sudo] password for steve:
[steve@stapp02 ~]$
```

> **Why:** `sudo` runs the account-creation operation with administrator privileges. `useradd` creates a new local user account. The `-s` option sets the user's login shell to `/sbin/nologin`, and `/sbin/nologin` refuses interactive login sessions. The final argument, `mariyam`, is the required login name. No password is assigned to `mariyam` because the challenge only asks for the account and its shell.

### ✅ Step 3: Verify

The challenge was completed after the `useradd` command returned to the prompt without an error, so no separate verification command was required during the lab. For a manual check, run:

```bash
getent passwd mariyam
```

> **Why:** `getent` reads entries from the system's configured Name Service Switch databases. The `passwd` database contains user account records, and `mariyam` limits the result to that account. The final field should be `/sbin/nologin`, confirming the required non-interactive shell.

## Best Practices

- **Use a non-interactive shell for service accounts.** `/sbin/nologin` prevents direct interactive sessions while allowing software to run under a dedicated user identity when needed.
- **Use least privilege.** Run only the account-management command with `sudo` instead of working as `root` for the entire session.
- **Match the requested account values exactly.** The username `mariyam` and shell `/sbin/nologin` are the values the challenge expects.
- **Keep a verification step available.** Even when a lab grader accepts a successful command immediately, checking the resulting account record is a useful operational habit.

### 📚 Official Documentation

- [Project Nautilus infrastructure](https://kodekloudhub.github.io/kodekloud-engineer/docs/projects/nautilus)
- [ssh(1) — OpenSSH remote login client](https://man7.org/linux/man-pages/man1/ssh.1.html)
- [useradd(8) — create a new user](https://man7.org/linux/man-pages/man8/useradd.8.html)
- [nologin(8) — politely refuse a login](https://man7.org/linux/man-pages/man8/nologin.8.html)
- [getent(1) — get entries from Name Service Switch libraries](https://man7.org/linux/man-pages/man1/getent.1.html)
