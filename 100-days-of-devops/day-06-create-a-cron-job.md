# Day 06: Create a Cron Job

The Nautilus system admins team has prepared scripts to automate several day-to-day tasks. They want them to be deployed on all app servers in Stratos DC on a set schedule. Before that they need to test similar functionality with a sample cron job. Therefore, perform the steps below:

## Specific Requirements:

1. Install `cronie` package on all Nautilus app servers and start `crond` service.
2. Add a cron `*/5 * * * * echo hello > /tmp/cron_text` for root user.

## Solution

The cron service must be available on all three app servers: `stapp01`, `stapp02`, and `stapp03`. The `cronie` package provides the cron daemon, while the `crond` service reads users' crontab files and runs scheduled commands.

The job is added to `root`'s personal crontab on each server. The schedule `*/5 * * * *` means that the command runs every five minutes. Because the challenge only asks us to install and start the service and add the job, no additional status or file-content verification was required during the lab.

### 🔌 Step 1: Install and configure the cron job on App Server 1

Connect to App Server 1:

```bash
ssh tony@stapp01
```

The connection succeeded:

```text
thor@jump-host ~$ ssh tony@stapp01
The authenticity of host 'stapp01 (10.244.49.15)' can't be established.
ED25519 key fingerprint is SHA256:lZ49sLYKmh62kwHCh2cRYh7sp6FgE6WlFmUUVV+Ope0.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added 'stapp01' (ED25519) to the list of known hosts.
tony@stapp01's password:
[tony@stapp01 ~]$
```

Install the package and start the daemon:

```bash
sudo yum install -y cronie
sudo systemctl start crond
```

The installation completed successfully, and the service-start command returned without an error:

```text
[tony@stapp01 ~]$ sudo yum install -y cronie

We trust you have received the usual lecture from the local System
Administrator. It usually boils down to these three things:

    #1) Respect the privacy of others.
    #2) Think before you type.
    #3) With great power comes great responsibility.

[sudo] password for tony:
Last metadata expiration check: 0:05:03 ago on Wed Jul 22 12:55:24 2026.
Dependencies resolved.
Installing:
 cronie              x86_64      1.5.7-16.el9                  baseos      119 k
Installing dependencies:
 cronie-anacron      x86_64      1.5.7-16.el9                  baseos       31 k
 crontabs            noarch      1.11-26.20190603git.el9       baseos       19 k

Transaction Summary
=================================================================================
Install  3 Packages

Created symlink /etc/systemd/system/multi-user.target.wants/crond.service → /usr/lib/systemd/system/crond.service.

Installed:
  cronie-1.5.7-16.el9.x86_64                cronie-anacron-1.5.7-16.el9.x86_64
  crontabs-1.11-26.20190603git.el9.noarch

Complete!
[tony@stapp01 ~]$ sudo systemctl start crond
[tony@stapp01 ~]$
```

> **Why:** `sudo` runs a command with administrative privileges. `yum` is the package-management command available on the lab's RHEL-compatible host, `install` requests the named package, and `-y` automatically confirms the installation. The `cronie` package provides the cron daemon, and the transaction also installed `cronie-anacron` and `crontabs` as dependencies. `systemctl` controls services managed by `systemd`, `start` starts a service for the current session, and `crond` is the service that executes scheduled cron jobs. The `Complete!` message confirms that the package transaction finished successfully. The created systemd symlink shows that the package also enabled `crond` on this host, but the explicit `start` command is still required by the challenge.

Open the root user's crontab:

```bash
sudo crontab -u root -e
```

When the editor opens, press `i`, add the following line, press `Esc`, type `:wq`, and press `Enter` to save and exit:

```text
*/5 * * * * echo hello > /tmp/cron_text
```

The crontab command reported that the new root crontab was installed:

```text
[tony@stapp01 ~]$ sudo crontab -u root -e
no crontab for root - using an empty one
crontab: installing new crontab
[tony@stapp01 ~]$
```

> **Why:** `crontab` manages a user's scheduled commands. The `-u root` option selects the `root` user's crontab, and `-e` opens it for editing. If the user has no existing crontab, the command starts with an empty one; saving the editor creates it. The five cron fields are minute, hour, day of month, month, and day of week. `*/5` in the minute field means every five minutes, while each `*` allows every value in the remaining fields. `echo hello` writes the word `hello`, and `>` redirects that output to `/tmp/cron_text`, replacing the file contents each time the job runs.

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

### 🔌 Step 2: Install and configure the cron job on App Server 2

Connect to App Server 2:

```bash
ssh steve@stapp02
```

The first password attempt failed, but the second attempt connected successfully:

```text
thor@jump-host ~$ ssh steve@stapp02
The authenticity of host 'stapp02 (10.244.29.245)' can't be established.
ED25519 key fingerprint is SHA256:jD6en5XjWFzz+VNT/PN4EyJZl/VK3SENgYyiZNSKNF0.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added 'stapp02' (ED25519) to the list of known hosts.
steve@stapp02's password:
Permission denied, please try again.
steve@stapp02's password:
Last failed login: Wed Jul 22 13:02:22 UTC 2026 from 10.244.240.138 on ssh:notty
There was 1 failed login attempt since the last successful login.
[steve@stapp02 ~]$
```

Install `cronie` and start `crond`:

```bash
sudo yum install -y cronie
sudo systemctl start crond
```

The package transaction completed and the service-start command returned without an error:

```text
[steve@stapp02 ~]$ sudo yum install -y cronie

We trust you have received the usual lecture from the local System
Administrator. It usually boils down to these three things:

    #1) Respect the privacy of others.
    #2) Think before you type.
    #3) With great power comes great responsibility.

[sudo] password for steve:
Last metadata expiration check: 0:09:26 ago on Wed Jul 22 12:53:21 2026.
Dependencies resolved.
Installing:
 cronie              x86_64      1.5.7-16.el9                  baseos      119 k
Installing dependencies:
 cronie-anacron      x86_64      1.5.7-16.el9                  baseos       31 k
 crontabs            noarch      1.11-26.20190603git.el9       baseos       19 k

Transaction Summary
=================================================================================
Install  3 Packages

Created symlink /etc/systemd/system/multi-user.target.wants/crond.service → /usr/lib/systemd/system/crond.service.

Installed:
  cronie-1.5.7-16.el9.x86_64                cronie-anacron-1.5.7-16.el9.x86_64
  crontabs-1.11-26.20190603git.el9.noarch

Complete!
[steve@stapp02 ~]$ sudo systemctl start crond
[steve@stapp02 ~]$
```

> **Why:** The same package and service preparation is required on every app server. Installing the package locally ensures that `crond` and the `crontab` command are available on App Server 2; starting `crond` makes the scheduler active in the current session.

Open the root user's crontab and add the same schedule:

```bash
sudo crontab -u root -e
```

Add this line in the editor, then save with `Esc`, `:wq`, and `Enter`:

```text
*/5 * * * * echo hello > /tmp/cron_text
```

The root crontab was created successfully:

```text
[steve@stapp02 ~]$ sudo crontab -u root -e
no crontab for root - using an empty one
crontab: installing new crontab
[steve@stapp02 ~]$
```

> **Why:** Selecting `root` with `-u root` is important because each Linux user has a separate crontab. A job saved in `steve`'s crontab would run as `steve`, not as `root`, and would not satisfy the challenge.

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

### 🔌 Step 3: Install and configure the cron job on App Server 3

Connect to App Server 3:

```bash
ssh banner@stapp03
```

The connection succeeded:

```text
thor@jump-host ~$ ssh banner@stapp03
The authenticity of host 'stapp03 (10.244.49.47)' can't be established.
ED25519 key fingerprint is SHA256:ER8hjtjcXzQLTiIQb6b2calEnBiFzKDyQ9DBU0H8Tp8.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added 'stapp03' (ED25519) to the list of known hosts.
banner@stapp03's password:
[banner@stapp03 ~]$
```

Install `cronie` and start `crond`:

```bash
sudo yum install -y cronie
sudo systemctl start crond
```

The first `sudo` password attempt was rejected, but the command completed after the password was entered again:

```text
[banner@stapp03 ~]$ sudo yum install -y cronie

We trust you have received the usual lecture from the local System
Administrator. It usually boils down to these three things:

    #1) Respect the privacy of others.
    #2) Think before you type.
    #3) With great power comes great responsibility.

[sudo] password for banner:
Sorry, try again.
[sudo] password for banner:
Dependencies resolved.
Installing:
 cronie              x86_64      1.5.7-16.el9                  baseos      119 k
Installing dependencies:
 cronie-anacron      x86_64      1.5.7-16.el9                  baseos       31 k
 crontabs            noarch      1.11-26.20190603git.el9       baseos       19 k

Transaction Summary
=================================================================================
Install  3 Packages

Created symlink /etc/systemd/system/multi-user.target.wants/crond.service → /usr/lib/systemd/system/crond.service.

Installed:
  cronie-1.5.7-16.el9.x86_64                cronie-anacron-1.5.7-16.el9.x86_64
  crontabs-1.11-26.20190603git.el9.noarch

Complete!
[banner@stapp03 ~]$ sudo systemctl start crond
[banner@stapp03 ~]$
```

> **Why:** A rejected password does not change the system; the command only runs after successful `sudo` authentication. Once the package installation completed with `Complete!`, `systemctl start crond` started the scheduler on App Server 3.

Open the root user's crontab and add the requested job:

```bash
sudo crontab -u root -e
```

Add this line in the editor, then save with `Esc`, `:wq`, and `Enter`:

```text
*/5 * * * * echo hello > /tmp/cron_text
```

The root crontab was installed successfully:

```text
[banner@stapp03 ~]$ sudo crontab -u root -e
no crontab for root - using an empty one
crontab: installing new crontab
[banner@stapp03 ~]$
```

> **Why:** The schedule is identical on all three app servers so the sample automation behaves consistently across the Stratos application tier. Saving the editor writes the entry into `root`'s crontab for `crond` to read.

Return to the jump host:

```bash
exit
```

```text
[banner@stapp03 ~]$ exit
logout
Connection to stapp03 closed.
thor@jump-host ~$
```

### ✅ Step 4: Verify the root crontab entries

The lab validator reported success after the package, service, and crontab changes. The lab session did not run an additional verification command. If a manual check is needed, run the following command on each app server:

```bash
sudo crontab -u root -l
```

The expected entry is:

```text
*/5 * * * * echo hello > /tmp/cron_text
```

> **Why:** `crontab` manages scheduled commands, `-u root` selects the root user's crontab, and `-l` lists its current entries. The displayed line confirms that the job was stored for the correct user with the requested five-minute schedule.

## Best Practices

- **Install the scheduler on every target host.** A crontab entry cannot run if the `crond` service is not installed and started on that server.
- **Use the correct user crontab.** `sudo crontab -u root -e` ensures that the command runs with `root`'s permissions, as required by the challenge.
- **Preserve existing scheduled jobs.** Using `crontab -e` edits the selected user's crontab without replacing the complete file with a new one.
- **Understand output redirection.** The `>` operator replaces `/tmp/cron_text` each time the job runs. Use `>>` only when a task specifically requires appending a history or log.
- **Keep the schedule readable.** The five-field expression is compact, but each field should be understood before the job is deployed to production.
- **Use the smallest required privilege.** `sudo` is used only for package installation, service control, and editing the root crontab; no persistent root shell is opened.

### 📚 Official Documentation

- [Project Nautilus infrastructure](https://kodekloudhub.github.io/kodekloud-engineer/docs/projects/nautilus)
- [Red Hat Enterprise Linux 7 — Automating system tasks](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/7/html/system_administrators_guide/ch-automating_system_tasks)
- [Red Hat Enterprise Linux 9 — Managing systemd](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/configuring_basic_system_settings/managing-systemd_configuring-basic-system-settings)
- [`crontab(1)` Linux manual page](https://man7.org/linux/man-pages/man1/crontab.1.html)
- [`crontab(5)` Linux manual page](https://man7.org/linux/man-pages/man5/crontab.5.html)
