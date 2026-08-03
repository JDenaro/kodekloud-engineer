# Day 09: MariaDB Troubleshooting

There is a critical issue going on with the Nautilus application in Stratos DC. The production support team identified that the application is unable to connect to the database. After digging into the issue, the team found that mariadb service is down on the database server.

Look into the issue and fix the same.

## Specific Requirements:

1. Troubleshoot the `mariadb` service on the database server.
2. Fix the issue preventing MariaDB from starting.
3. Leave the `mariadb` service running.

## Solution

The database server was `stdb01`, accessed with the `peter` account. The service initially appeared inactive, but a normal start attempt failed. The service logs identified that MariaDB could not initialize its data directory because `/var/lib/mysql` was missing or had been created with the wrong ownership.

MariaDB runs as the `mysql` user. Creating `/var/lib/mysql` and assigning ownership to `mysql:mysql` allowed the service preparation step to initialize the database and start successfully.

### 🔌 Step 1: Connect to the database server

From the Jump Host, connect to `stdb01`:

```bash
ssh peter@stdb01
```

The session opened on the database server:

```text
thor@jump-host ~$ ssh peter@stdb01
[peter@stdb01 ~]$
```

> **Why:** `ssh` opens a secure remote shell. `peter` is the database-server login account and `stdb01` is the server hosting MariaDB. All diagnostics and corrections must be performed on that host.

### 🔎 Step 2: Check the MariaDB service state

```bash
sudo systemctl status mariadb --no-pager
```

The service was inactive:

```text
○ mariadb.service - MariaDB 10.5 database server
     Loaded: loaded (/usr/lib/systemd/system/mariadb.service; disabled; preset:>
     Active: inactive (dead)
```

> **Why:** `sudo` provides the privileges required to inspect a system service. `systemctl` controls services managed by `systemd`, `status` displays the current service state, and `--no-pager` prints the complete output directly in the terminal. `inactive (dead)` confirmed why the application could not connect to the database.

### 🧪 Step 3: Attempt to start MariaDB and capture the failure

```bash
sudo systemctl start mariadb
```

The start operation failed, so the detailed service logs were collected:

```bash
sudo journalctl -xeu mariadb.service --no-pager
```

The important diagnostic message was:

```text
Database MariaDB is not initialized, but the directory /var/lib/mysql is not empty, so initialization cannot be done.
Make sure the /var/lib/mysql is empty before running mariadb-prepare-db-dir.
```

> **Why:** `start` attempts to launch the service. `journalctl` reads logs from the systemd journal; `-x` adds explanatory context, `-e` jumps to the newest messages, `-u mariadb.service` filters the output to the MariaDB unit, and `--no-pager` keeps the output in the terminal. The log showed that the initialization preparation step could not work with the state of `/var/lib/mysql`.

### 🔍 Step 4: Inspect the MariaDB data directory and configuration

Check the configured data directory:

```bash
sudo cat /etc/my.cnf.d/mariadb-server.cnf
```

The server configuration specified:

```text
[mysqld]
datadir=/var/lib/mysql
socket=/var/lib/mysql/mysql.sock
```

Inspect the directory:

```bash
sudo ls -ld /var/lib/mysql
```

The directory had been created with root ownership:

```text
drwxr-xr-x 2 root root 4096 Jul 30 22:01 /var/lib/mysql
```

> **Why:** `cat` displays the MariaDB server configuration, confirming the `datadir` that MariaDB uses for its database files. `ls -ld` displays the directory's metadata without listing its contents. The `root root` owner and group were the problem: the MariaDB service runs as `mysql`, so its data directory must be owned by `mysql:mysql`.

### 📁 Step 5: Create the data directory when it is missing

The directory was created manually because the configured data path was absent during the investigation:

```bash
sudo mkdir -p /var/lib/mysql
```

> **Why:** `mkdir` creates directories, and `-p` also creates any missing parent directories while avoiding an error if the directory already exists. The target path matches the `datadir` from `mariadb-server.cnf`.

### 👤 Step 6: Assign MariaDB ownership

```bash
sudo chown mysql:mysql /var/lib/mysql
```

The ownership correction completed without an error:

```text
[peter@stdb01 ~]$ sudo chown mysql:mysql /var/lib/mysql
[peter@stdb01 ~]$
```

> **Why:** `chown` changes ownership. In `mysql:mysql`, the first value assigns the user owner and the second assigns the group owner. MariaDB can now initialize and write its database files as the `mysql` service account instead of trying to write to a root-owned directory.

### ▶️ Step 7: Start MariaDB after correcting the root cause

```bash
sudo systemctl start mariadb
```

The service started successfully:

```text
[peter@stdb01 ~]$ sudo systemctl start mariadb
[peter@stdb01 ~]$
```

> **Why:** The same `systemctl start` command is retried after correcting the data-directory ownership. This tests the hypothesis directly: if ownership was the root cause, MariaDB should now initialize and start.

### ✅ Step 8: Verify MariaDB is running

```bash
sudo systemctl status mariadb --no-pager
```

The final status confirmed success:

```text
● mariadb.service - MariaDB 10.5 database server
     Loaded: loaded (/usr/lib/systemd/system/mariadb.service; disabled; preset: disabled)
     Active: active (running)
     Status: "Taking your SQL requests now..."
     Main PID: 42504 (mariadbd)
     Process: 42440 ExecStartPre=/usr/libexec/mariadb-prepare-db-dir (code=exited, status=0/SUCCESS)
```

> **Why:** `Active: active (running)` confirms that the service is operating. `ExecStartPre ... status=0/SUCCESS` confirms that the database-directory preparation step completed successfully, and `Taking your SQL requests now...` confirms that MariaDB is ready to accept database connections. The service remained `disabled` for automatic boot, but the challenge required the service to be running; no additional enablement was necessary for this lab.

## Best Practices

- **Read the service logs before changing files.** The `journalctl` output identified the failing initialization step and prevented guesswork.
- **Match data-directory ownership to the service account.** MariaDB runs as `mysql`, so `/var/lib/mysql` must be owned by `mysql:mysql`.
- **Use the configured `datadir`.** Check the MariaDB configuration before creating or moving database directories.
- **Change one variable at a time.** The directory was created and ownership was corrected before retrying the service, making the successful fix attributable to the ownership correction.
- **Do not delete database files blindly.** Inspect the data directory and logs first; removing database contents can cause irreversible data loss.
- **Distinguish service state from boot enablement.** `active (running)` means MariaDB is running now; `enabled` controls whether it starts automatically after a reboot.

### 📚 Official Documentation

- [MariaDB systemd service documentation](https://mariadb.com/docs/server/server-management/starting-and-stopping-mariadb/systemd)
- [mariadb-install-db documentation](https://mariadb.com/docs/server/clients-and-utilities/deployment-tools/mariadb-install-db)
- [systemctl(1) Linux manual page](https://man7.org/linux/man-pages/man1/systemctl.1.html)
