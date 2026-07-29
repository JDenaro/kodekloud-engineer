# Task 15: Timezone Alignment

In the daily standup, it was noted that the timezone settings across the Nautilus Application Servers in the Stratos Datacenter are inconsistent with the local datacenter's timezone, currently set to Pacific/Tongatapu.

Synchronize the timezone settings to match the local datacenter's timezone (Pacific/Tongatapu).

## Task Requirements

1. Configure the timezone on all Nautilus Application Servers.
2. Set the timezone to `Pacific/Tongatapu`.

## Solution

The timezone is configured with `timedatectl`, the systemd utility for querying and changing system time settings. The `set-timezone` command updates the system timezone and the `/etc/localtime` link. The change does not require a server reboot.

### 🌏 Step 1: Set the timezone on App Server 1

Connect from the Jump Host and apply the datacenter timezone:

```bash
ssh tony@stapp01
sudo timedatectl set-timezone Pacific/Tongatapu
exit
```

The command completed successfully and returned to the shell without an error:

```text
[tony@stapp01 ~]$ sudo timedatectl set-timezone Pacific/Tongatapu
[tony@stapp01 ~]$ exit
logout
Connection to stapp01 closed.
```

> **Why:** `ssh` opens a secure remote shell on App Server 1. `sudo` supplies the administrative privileges required to change a system-wide setting. `timedatectl` controls the system clock and date configuration, while `set-timezone` sets the timezone to the supplied value, `Pacific/Tongatapu`. `exit` returns to the Jump Host.

### 🌏 Step 2: Set the timezone on App Server 2

```bash
ssh steve@stapp02
sudo timedatectl set-timezone Pacific/Tongatapu
exit
```

The command completed successfully and returned to the shell without an error:

```text
[steve@stapp02 ~]$ sudo timedatectl set-timezone Pacific/Tongatapu
[steve@stapp02 ~]$ exit
logout
Connection to stapp02 closed.
```

> **Why:** App Server 2 must use the same timezone as the other application servers. `steve` is the designated login account for `stapp02`. Running `timedatectl set-timezone Pacific/Tongatapu` changes the persistent system timezone without changing the file contents of the application or restarting services.

### 🌏 Step 3: Set and inspect the timezone on App Server 3

```bash
ssh banner@stapp03
sudo timedatectl set-timezone Pacific/Tongatapu
timedatectl
```

The timezone was set and confirmed:

```text
[banner@stapp03 ~]$ sudo timedatectl set-timezone Pacific/Tongatapu
[banner@stapp03 ~]$ timedatectl
               Local time: Wed 2026-07-29 15:06:10 +13
           Universal time: Wed 2026-07-29 02:06:10 UTC
                 RTC time: n/a
          Time zone: Pacific/Tongatapu (+13, +1300)
System clock synchronized: yes
           NTP service: n/a
       RTC in local TZ: no
[banner@stapp03 ~]$
```

> **Why:** `banner` is the designated login account for `stapp03`. The `timedatectl` command without a subcommand displays the current system clock, timezone, and synchronization information. The `Time zone: Pacific/Tongatapu (+13, +1300)` line confirms that the requested timezone is active.

## Best Practices

- **Use the IANA timezone name.** `Pacific/Tongatapu` identifies the timezone unambiguously and includes its UTC offset rules.
- **Apply configuration consistently.** Set the same timezone on every App Server so timestamps are comparable across the application environment.
- **Use administrative privileges only where needed.** The timezone change requires `sudo`, while displaying the current settings does not.
- **Verify the resulting timezone.** `timedatectl` provides the active timezone and current offset in a readable format.
- **Avoid unnecessary reboots.** Changing the timezone does not require restarting the servers.

### 📚 Official Documentation

- [timedatectl(1) Linux manual page](https://man7.org/linux/man-pages/man1/timedatectl.1.html)
