# Day 02: Temporary User Setup with Expiry

As part of the temporary assignment to the Nautilus project, a developer named siva requires access for a limited duration. To ensure smooth access management, a temporary user account with an expiry date is needed. Here's what you need to do:

## Specific Requirements:

1. Create a user named `siva` on App Server 3 in Stratos Datacenter. Set the expiry date to `2026-12-07`, ensuring the user is created in lowercase as per standard protocol.

## Solution

The work happens on App Server 3, `stapp03`. The account is created with the requested lowercase username and an account expiry date of `2026-12-07`. The `useradd -e` option stores the date on which the account will be disabled.

### 🔌 Step 1: Connect to App Server 3

```bash
ssh banner@stapp03
```

The first connection to the lab server asked us to confirm its SSH host key:

```text
thor@jump-host ~$ ssh banner@stapp03
The authenticity of host 'stapp03 (10.244.97.185)' can't be established.
ED25519 key fingerprint is SHA256:zcX6Xs1KgJvL14NOcu6F6FMxrGlve4mjlL0zlqo/UH0.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added 'stapp03' (ED25519) to the list of known hosts.
banner@stapp03's password:
[banner@stapp03 ~]$
```

> **Why:** `ssh` opens an encrypted remote-login session. `banner` is the lab user for App Server 3, and `stapp03` is that server's hostname. On a first connection, SSH displays the server's host-key fingerprint so the client can confirm the server identity before saving it in the local `known_hosts` file. Enter the lab password `BigGr33n` when prompted.

### 👤 Step 2: Create `siva` with an expiry date

```bash
sudo useradd -e 2026-12-07 siva
```

The command completed successfully in the lab and returned to the prompt without an error:

```text
[banner@stapp03 ~]$ sudo useradd -e 2026-12-07 siva

We trust you have received the usual lecture from the local System
Administrator. It usually boils down to these three things:

    #1) Respect the privacy of others.
    #2) Think before you type.
    #3) With great power comes great responsibility.

[sudo] password for banner:
[banner@stapp03 ~]$
```

> **Why:** `sudo` runs the account-creation operation with administrator privileges. `useradd` creates a new user account. The `-e` option, also written `--expiredate`, sets the date on which the account will be disabled; `2026-12-07` uses the required `YYYY-MM-DD` format. The final argument, `siva`, is the required lowercase login name. No password is assigned because the challenge only asks for the user and its expiry date.

### ✅ Step 3: Verify

The challenge was completed after the `useradd` command returned to the prompt without an error, so no separate verification command was required during the lab. For a manual check, run:

```bash
sudo chage -l siva
```

> **Why:** `chage` displays or changes password-aging information. The `-l` option lists the account-aging fields, and `siva` selects the account to inspect. The output should show an account expiry corresponding to `2026-12-07`. This optional check was not run during the guided lab.

## Best Practices

- **Use account expiry for temporary access.** An explicit expiry date limits how long a temporary account remains usable without relying on a later manual cleanup task.
- **Use the standard date format.** `YYYY-MM-DD` is unambiguous and is the format accepted by `useradd` for the account expiry date.
- **Keep usernames lowercase.** Lowercase names follow the naming convention requested by the challenge and avoid inconsistent account references.
- **Review temporary accounts.** Even with an expiry date, periodically review temporary accounts and remove them when the assignment is permanently complete.

### 📚 Official Documentation

- [Project Nautilus infrastructure](https://kodekloudhub.github.io/kodekloud-engineer/docs/projects/nautilus)
- [ssh(1) — OpenSSH remote login client](https://man7.org/linux/man-pages/man1/ssh.1.html)
- [useradd(8) — create a new user](https://man7.org/linux/man-pages/man8/useradd.8.html)
- [chage(1) — change user password expiry information](https://man7.org/linux/man-pages/man1/chage.1.html)
