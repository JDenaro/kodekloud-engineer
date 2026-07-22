# Day 05: SELinux Installation and Configuration

Following a security audit, the xFusionCorp Industries security team has opted to enhance application and server security with SELinux. To initiate testing, the following requirements have been established for App server 2 in the Stratos Datacenter:

## Specific Requirements:

1. Install the required SELinux packages.
2. Permanently disable SELinux for the time being; it will be re-enabled after necessary configuration changes.
3. No need to reboot the server, as a scheduled maintenance reboot is already planned for tonight.
4. Disregard the current status of SELinux via the command line; the final status after the reboot should be disabled.

## Solution

The target server is App Server 2, `stapp02`, and the administrative user for this host is `steve`. The required SELinux packages are installed first. SELinux is then disabled in `/etc/selinux/config`, which controls the state that will be applied during the next boot.

The challenge specifically says not to reboot and to disregard the current command-line status. Therefore, this walkthrough does not run `getenforce`, does not use `setenforce`, and does not reboot the server. The important result is the persistent configuration value `SELINUX=disabled`.

### 🔌 Step 1: Connect to App Server 2

```bash
ssh steve@stapp02
```

The lab connection succeeded and opened a shell as `steve`:

```text
thor@jump-host ~$ ssh steve@stapp02
The authenticity of host 'stapp02 (10.244.29.219)' can't be established.
ED25519 key fingerprint is SHA256:i86BPiXEqSQKQzMmkGnWEjIVMx/8C1YUx9fX3V2syW8.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added 'stapp02' (ED25519) to the list of known hosts.
steve@stapp02's password:
[steve@stapp02 ~]$
```

> **Why:** `ssh` opens an encrypted remote shell session. `steve` is the user assigned to App Server 2, and `stapp02` is the server hostname. The first connection asks whether the server identity should be trusted; answering `yes` stores the host key in the jump host's known-hosts file. The password is then used to authenticate the lab user.

### 📦 Step 2: Install the SELinux packages

```bash
sudo yum install -y selinux-policy selinux-policy-targeted
```

The package installation completed successfully. The transaction installed the two requested SELinux packages and their dependencies, and it also upgraded packages required by the transaction:

```text
[steve@stapp02 ~]$ sudo yum install -y selinux-policy selinux-policy-targeted

We trust you have received the usual lecture from the local System
Administrator. It usually boils down to these three things:

    #1) Respect the privacy of others.
    #2) Think before you type.
    #3) With great power comes great responsibility.

[sudo] password for steve:
Last metadata expiration check: 0:20:16 ago on Wed Jul 22 04:17:46 2026.
Dependencies resolved.
=================================================================================
 Package                         Arch      Version            Repository    Size
=================================================================================
Installing:
 selinux-policy                  noarch    38.1.83-1.el9      baseos        42 k
 selinux-policy-targeted         noarch    38.1.83-1.el9      baseos       6.9 M
Upgrading:
 audit-libs                      x86_64    3.1.5-8.el9        baseos       121 k
 ima-evm-utils                   x86_64    1.6.2-4.el9        baseos        71 k
 openssl                         x86_64    1:3.5.7-2.el9      baseos       1.5 M
 openssl-devel                   x86_64    1:3.5.7-2.el9      appstream    4.8 M
 openssl-libs                    x86_64    1:3.5.7-2.el9      baseos       2.3 M
 policycoreutils                 x86_64    3.6-7.el9          baseos       238 k
 python3-rpm                     x86_64    4.16.1.3-40.el9    baseos        64 k
 rpm                             x86_64    4.16.1.3-40.el9    baseos       535 k
 rpm-build-libs                 x86_64    4.16.1.3-40.el9    baseos        88 k
 rpm-libs                        x86_64    4.16.1.3-40.el9    baseos       307 k
 rpm-sign-libs                   x86_64    4.16.1.3-40.el9    baseos        20 k
Installing dependencies:
 checkpolicy                     x86_64    3.6-1.el9          baseos       353 k
 flatpak-selinux                 noarch    1.12.9-1.el9       appstream     22 k
 openssl-fips-provider           x86_64    1:3.5.7-2.el9      baseos       812 k
 policycoreutils-python-utils    noarch    3.6-7.el9          baseos        75 k
 python3-audit                   x86_64    3.1.5-8.el9        baseos        82 k
 python3-distro                  noarch    1.5.0-7.el9        baseos        37 k
 python3-libselinux              x86_64    3.6-1.el9          appstream    188 k
 python3-libsemanage             x86_64    3.6-1.el9          appstream     80 k
 python3-policycoreutils         noarch    3.6-7.el9          baseos       2.1 M
 python3-setools                 x86_64    4.4.4-1.el9        baseos       605 k
 python3-setuptools              noarch    53.0.0-15.el9      baseos        936 k
 rpm-plugin-selinux              x86_64    4.16.1.3-40.el9    baseos        15 k

Transaction Summary
=================================================================================
Install  14 Packages
Upgrade  11 Packages

Total download size: 22 M
Downloading Packages:
(1/25): policycoreutils-python-utils-3.6-7.el9.n 271 kB/s |  75 kB     00:00
(2/25): python3-audit-3.1.5-8.el9.x86_64.rpm     1.2 MB/s |  82 kB     00:00
(3/25): checkpolicy-3.6-1.el9.x86_64.rpm         863 kB/s | 353 kB     00:00
(4/25): python3-distro-1.5.0-7.el9.noarch.rpm    526 kB/s |  37 kB     00:00
(5/25): openssl-fips-provider-3.5.7-2.el9.x86_64 1.5 MB/s | 812 kB     00:00
(6/25): python3-setools-4.4.4-1.el9.x86_64.rpm   3.0 MB/s | 605 kB     00:00
(7/25): rpm-plugin-selinux-4.16.1.3-40.el9.x86_6 230 kB/s |  15 kB     00:00
(8/25): selinux-policy-38.1.83-1.el9.noarch.rpm  623 kB/s |  42 kB     00:00
(9/25): python3-setuptools-53.0.0-15.el9.noarch. 2.3 MB/s | 936 kB     00:00
(10/25): flatpak-selinux-1.12.9-1.el9.noarch.rpm 112 kB/s |  22 kB     00:00
(11/25): python3-policycoreutils-3.6-7.el9.noarc 2.7 MB/s | 2.1 MB     00:00
(12/25): python3-libselinux-3.6-1.el9.x86_64.rpm 1.3 MB/s | 188 kB     00:00
(13/25): python3-libsemanage-3.6-1.el9.x86_64.rpm 928 kB/s |  80 kB     00:00
(14/25): audit-libs-3.1.5-8.el9.x86_64.rpm       1.7 MB/s | 121 kB     00:00
(15/25): ima-evm-utils-1.6.2-4.el9.x86_64.rpm    1.0 MB/s |  71 kB     00:00
(16/25): selinux-policy-targeted-38.1.83-1.el9.n 8.1 MB/s | 6.9 MB     00:00
(17/25): openssl-libs-3.5.7-2.el9.x86_64.rpm     8.7 MB/s | 2.3 MB     00:00
(18/25): policycoreutils-3.6-7.el9.x86_64.rpm    3.3 MB/s | 238 kB     00:00
(19/25): python3-rpm-4.16.1.3-40.el9.x86_64.rpm  963 kB/s |  64 kB     00:00
(20/25): rpm-build-libs-4.16.1.3-40.el9.x86_64.r 1.3 MB/s |  88 kB     00:00
(21/25): rpm-4.16.1.3-40.el9.x86_64.rpm          7.0 MB/s | 535 kB     00:00
(22/25): rpm-sign-libs-4.16.1.3-40.el9.x86_64.rpm 290 kB/s |  20 kB     00:00
(23/25): rpm-libs-4.16.1.3-40.el9.x86_64.rpm     4.2 MB/s | 307 kB     00:00
(24/25): openssl-devel-3.5.7-2.el9.x86_64.rpm     23 MB/s | 4.8 MB     00:00
(25/25): openssl-3.5.7-2.el9.x86_64.rpm          1.6 MB/s | 1.5 MB     00:00
---------------------------------------------------------------------------------
Total                                            7.6 MB/s |  22 MB     00:02
Running transaction check
Transaction check succeeded.
Running transaction test
Transaction test succeeded.
Running transaction
  Running scriptlet: selinux-policy-targeted-38.1.83-1.el9.noarch            1/1
  Preparing        :                                                         1/1
  Upgrading        : audit-libs-3.1.5-8.el9.x86_64                          1/36
  Installing       : python3-libselinux-3.6-1.el9.x86_64                    2/36
  Installing       : python3-setuptools-53.0.0-15.el9.noarch                3/36
  Installing       : python3-distro-1.5.0-7.el9.noarch                      4/36
  Installing       : python3-setools-4.4.4-1.el9.x86_64                     5/36
  Installing       : python3-libsemanage-3.6-1.el9.x86_64                   6/36
  Installing       : python3-audit-3.1.5-8.el9.x86_64                       7/36
  Upgrading        : openssl-libs-1:3.5.7-2.el9.x86_64                      8/36
  Installing       : openssl-fips-provider-1:3.5.7-2.el9.x86_64             9/36
  Upgrading        : rpm-4.16.1.3-40.el9.x86_64                            10/36
  Upgrading        : rpm-libs-4.16.1.3-40.el9.x86_64                       11/36
  Upgrading        : policycoreutils-3.6-7.el9.x86_64                      12/36
  Running scriptlet: policycoreutils-3.6-7.el9.x86_64                      12/36
  Installing       : selinux-policy-38.1.83-1.el9.noarch                   13/36
  Running scriptlet: selinux-policy-38.1.83-1.el9.noarch                   13/36
  Running scriptlet: selinux-policy-targeted-38.1.83-1.el9.noarch          14/36
  Installing       : selinux-policy-targeted-38.1.83-1.el9.noarch          14/36
  Running scriptlet: selinux-policy-targeted-38.1.83-1.el9.noarch          14/36
  Upgrading        : rpm-build-libs-4.16.1.3-40.el9.x86_64                 15/36
  Upgrading        : ima-evm-utils-1.6.2-4.el9.x86_64                      16/36
  Upgrading        : rpm-sign-libs-4.16.1.3-40.el9.x86_64                  17/36
  Installing       : checkpolicy-3.6-1.el9.x86_64                          18/36
  Installing       : python3-policycoreutils-3.6-7.el9.noarch              19/36
  Installing       : policycoreutils-python-utils-3.6-7.el9.noarch         20/36
  Installing       : flatpak-selinux-1.12.9-1.el9.noarch                  21/36
  Running scriptlet: flatpak-selinux-1.12.9-1.el9.noarch                   21/36
  Upgrading        : python3-rpm-4.16.1.3-40.el9.x86_64                    22/36
  Installing       : rpm-plugin-selinux-4.16.1.3-40.el9.x86_64             23/36
  Upgrading        : openssl-1:3.5.7-2.el9.x86_64                          24/36
  Upgrading        : openssl-devel-1:3.5.7-2.el9.x86_64                    25/36
  Cleanup          : openssl-1:3.2.2-2.el9.x86_64                           26/36
  Cleanup          : python3-rpm-4.16.1.3-30.el9.x86_64                    27/36
  Cleanup          : openssl-devel-1:3.2.2-2.el9.x86_64                    28/36
  Cleanup          : rpm-sign-libs-4.16.1.3-30.el9.x86_64                  29/36
  Cleanup          : rpm-build-libs-4.16.1.3-30.el9.x86_64                 30/36
  Cleanup          : ima-evm-utils-1.5-2.el9.x86_64                        31/36
  Running scriptlet: policycoreutils-3.6-2.1.el9.x86_64                    32/36
  Cleanup          : policycoreutils-3.6-2.1.el9.x86_64                     32/36
  Cleanup          : rpm-libs-4.16.1.3-30.el9.x86_64                       33/36
  Cleanup          : rpm-4.16.1.3-30.el9.x86_64                            34/36
  Cleanup          : audit-libs-3.1.2-2.el9.x86_64                         35/36
  Cleanup          : openssl-libs-1:3.2.2-2.el9.x86_64                     36/36
  Running scriptlet: rpm-4.16.1.3-40.el9.x86_64                            36/36
  Running scriptlet: selinux-policy-targeted-38.1.83-1.el9.noarch          36/36
  Running scriptlet: openssl-libs-1:3.2.2-2.el9.x86_64                     36/36
  Verifying        : checkpolicy-3.6-1.el9.x86_64                           1/36
  Verifying        : openssl-fips-provider-1:3.5.7-2.el9.x86_64             2/36
  Verifying        : policycoreutils-python-utils-3.6-7.el9.noarch          3/36
  Verifying        : python3-audit-3.1.5-8.el9.x86_64                       4/36
  Verifying        : python3-distro-1.5.0-7.el9.noarch                      5/36
  Verifying        : python3-policycoreutils-3.6-7.el9.noarch               6/36
  Verifying        : python3-setools-4.4.4-1.el9.x86_64                     7/36
  Verifying        : python3-setuptools-53.0.0-15.el9.noarch                8/36
  Verifying        : rpm-plugin-selinux-4.16.1.3-40.el9.x86_64              9/36
  Verifying        : selinux-policy-38.1.83-1.el9.noarch                   10/36
  Verifying        : selinux-policy-targeted-38.1.83-1.el9.noarch           11/36
  Verifying        : flatpak-selinux-1.12.9-1.el9.noarch                   12/36
  Verifying        : python3-libselinux-3.6-1.el9.x86_64                    13/36
  Verifying        : python3-libsemanage-3.6-1.el9.x86_64                    14/36
  Verifying        : audit-libs-3.1.5-8.el9.x86_64                          15/36
  Verifying        : audit-libs-3.1.2-2.el9.x86_64                          16/36
  Verifying        : ima-evm-utils-1.6.2-4.el9.x86_64                      17/36
  Verifying        : ima-evm-utils-1.5-2.el9.x86_64                         18/36
  Verifying        : openssl-1:3.5.7-2.el9.x86_64                           19/36
  Verifying        : openssl-1:3.2.2-2.el9.x86_64                           20/36
  Verifying        : openssl-libs-1:3.5.7-2.el9.x86_64                      21/36
  Verifying        : openssl-libs-1:3.2.2-2.el9.x86_64                      22/36
  Verifying        : policycoreutils-3.6-7.el9.x86_64                       23/36
  Verifying        : policycoreutils-3.6-2.1.el9.x86_64                     24/36
  Verifying        : python3-rpm-4.16.1.3-40.el9.x86_64                    25/36
  Verifying        : python3-rpm-4.16.1.3-30.el9.x86_64                    26/36
  Verifying        : rpm-4.16.1.3-40.el9.x86_64                             27/36
  Verifying        : rpm-4.16.1.3-30.el9.x86_64                             28/36
  Verifying        : rpm-build-libs-4.16.1.3-40.el9.x86_64                 29/36
  Verifying        : rpm-build-libs-4.16.1.3-30.el9.x86_64                 30/36
  Verifying        : rpm-libs-4.16.1.3-40.el9.x86_64                       31/36
  Verifying        : rpm-libs-4.16.1.3-30.el9.x86_64                       32/36
  Verifying        : rpm-sign-libs-4.16.1.3-40.el9.x86_64                   33/36
  Verifying        : rpm-sign-libs-4.16.1.3-30.el9.x86_64                   34/36
  Verifying        : openssl-devel-1:3.5.7-2.el9.x86_64                    35/36
  Verifying        : openssl-devel-1:3.2.2-2.el9.x86_64                     36/36

Upgraded:
  audit-libs-3.1.5-8.el9.x86_64            ima-evm-utils-1.6.2-4.el9.x86_64
  openssl-1:3.5.7-2.el9.x86_64             openssl-devel-1:3.5.7-2.el9.x86_64
  openssl-libs-1:3.5.7-2.el9.x86_64        policycoreutils-3.6-7.el9.x86_64
  python3-rpm-4.16.1.3-40.el9.x86_64       rpm-4.16.1.3-40.el9.x86_64
  rpm-build-libs-4.16.1.3-40.el9.x86_64    rpm-libs-4.16.1.3-40.el9.x86_64
  rpm-sign-libs-4.16.1.3-40.el9.x86_64
Installed:
  checkpolicy-3.6-1.el9.x86_64
  flatpak-selinux-1.12.9-1.el9.noarch
  openssl-fips-provider-1:3.5.7-2.el9.x86_64
  policycoreutils-python-utils-3.6-7.el9.noarch
  python3-audit-3.1.5-8.el9.x86_64
  python3-distro-1.5.0-7.el9.noarch
  python3-libselinux-3.6-1.el9.x86_64
  python3-libsemanage-3.6-1.el9.x86_64
  python3-policycoreutils-3.6-7.el9.noarch
  python3-setools-4.4.4-1.el9.x86_64
  python3-setuptools-53.0.0-15.el9.noarch
  rpm-plugin-selinux-4.16.1.3-40.el9.x86_64
  selinux-policy-38.1.83-1.el9.noarch
  selinux-policy-targeted-38.1.83-1.el9.noarch

Complete!
[steve@stapp02 ~]$
```

> **Why:** `sudo` runs the package operation with administrative privileges. `yum` is the compatibility command for the DNF package manager on this RHEL 9 host. `install` adds packages from the configured repositories, and `-y` automatically answers yes to the confirmation prompt. `selinux-policy` provides the SELinux policy framework, while `selinux-policy-targeted` provides the targeted policy package. The package manager may also install dependencies and upgrade related packages as part of the transaction. The `Complete!` message confirms that the transaction finished successfully.

### 🔧 Step 3: Permanently disable SELinux for the next reboot

```bash
sudo sed -i 's/^SELINUX=.*/SELINUX=disabled/' /etc/selinux/config
```

The command completed without an error and returned to the shell prompt:

```text
[steve@stapp02 ~]$ sudo sed -i 's/^SELINUX=.*/SELINUX=disabled/' /etc/selinux/config
[steve@stapp02 ~]$
```

> **Why:** `sed` edits text from the command line. Its `-i` option saves the change directly to the file. The expression searches for the active configuration line that starts with `SELINUX=`, matches the rest of that line with `.*`, and replaces it with `SELINUX=disabled`. The `/etc/selinux/config` path is SELinux's persistent configuration file. This changes what will happen at the next boot; it does not require or perform an immediate reboot.

### ✅ Step 4: Verify the persistent SELinux configuration

```bash
sudo grep SELINUX /etc/selinux/config
```

The output confirmed that the persistent SELinux state is disabled:

```text
[steve@stapp02 ~]$ sudo grep SELINUX /etc/selinux/config
# SELINUX= can take one of these three values:
# NOTE: Up to RHEL 8 release included, SELINUX=disabled would also
SELINUX=disabled
# SELINUXTYPE= can take one of these three values:
SELINUXTYPE=targeted
[steve@stapp02 ~]$
```

> **Why:** `grep` searches the file for lines containing `SELINUX`, allowing us to inspect the persistent configuration without checking the current runtime state. The important line is `SELINUX=disabled`. `SELINUXTYPE=targeted` identifies the policy type that is configured for future SELinux use, but SELinux will remain disabled after the reboot because the `SELINUX` setting takes precedence. No `getenforce` check or reboot was performed because the challenge explicitly says to disregard the current command-line status and wait for the scheduled maintenance reboot.

## Best Practices

- **Install the required policy packages.** Install both the base `selinux-policy` package and the `selinux-policy-targeted` package so the host has the policy components needed for future SELinux configuration.
- **Separate persistent configuration from runtime state.** Changing `/etc/selinux/config` controls the next boot; it does not necessarily change the mode of a running system immediately.
- **Do not reboot outside the maintenance window.** The challenge explicitly provides a scheduled reboot, so the change was prepared without interrupting the server.
- **Avoid disabling SELinux as a general security practice.** Red Hat recommends using an appropriate SELinux mode instead of permanently disabling the security control; this lab requires disabling it temporarily for a later configuration phase.
- **Use `sudo` for the smallest required operation.** Administrative privileges are needed to install packages and modify `/etc/selinux/config`, but no full root shell is necessary.

### 📚 Official Documentation

- [Project Nautilus infrastructure](https://kodekloudhub.github.io/kodekloud-engineer/docs/projects/nautilus)
- [Red Hat Enterprise Linux 9 — Managing software with the DNF tool](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/htmlsingle/managing_software_with_the_dnf_tool/index)
- [Red Hat Enterprise Linux 8 — Changing SELinux states and modes](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/8/html/using_selinux/changing-selinux-states-and-modes_using-selinux)
- [GNU `sed` manual](https://www.gnu.org/software/sed/manual/sed.html)
- [GNU `grep` manual](https://www.gnu.org/software/grep/manual/grep.html)
