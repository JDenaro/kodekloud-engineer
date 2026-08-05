# Day 18: Install and Configure DB Server

We need to setup a database server on Nautilus DB Server in Stratos Datacenter. Please perform the below given steps on DB Server:

a. Install/Configure MariaDB server.

b. Create a database named kodekloud_db4.

c. Create a user called kodekloud_sam and set its password to B4zNgHA7Ya.

d. Grant full permissions to user kodekloud_sam on database kodekloud_db4.

## Specific Requirements:

1. Install/Configure MariaDB server.
2. Create a database named kodekloud_db4.
3. Create a user called kodekloud_sam and set its password to B4zNgHA7Ya.
4. Grant full permissions to user kodekloud_sam on database kodekloud_db4.

## Solution

The database server was `stdb01`, accessed with the `peter` user. MariaDB was installed with the `mariadb-server` package, enabled, and started. The database and user were then created from the MariaDB interactive client, and the user received full permissions on the requested database.

The password was entered inside the interactive SQL client instead of being passed as a shell argument. This is suitable for the training lab; in a real environment, credentials should be managed through a secret manager and should not be committed to documentation or exposed in command history.

### 🔌 Step 1: Connect to the database server

From the Jump Host, connect to `stdb01`:

~~~
ssh peter@stdb01
~~~

> **Why:** `ssh` opens a secure remote shell. `peter` is the login account for the database server, and `stdb01` is the host where MariaDB must be installed and configured.

### 📦 Step 2: Install MariaDB Server

Install the server package:

~~~
sudo yum install -y mariadb-server
~~~

The installation completed successfully:

~~~
Installed:
  mariadb-server-3:10.5.29-4.el9.x86_64
  mariadb-3:10.5.29-4.el9.x86_64
  mariadb-server-utils-3:10.5.29-4.el9.x86_64
  mariadb-backup-3:10.5.29-4.el9.x86_64
  mariadb-connector-c-3.2.6-1.el9.x86_64

Complete!
~~~

> **Why:** `sudo` runs the package operation with administrative privileges. `yum install` installs MariaDB Server and its required dependencies; `-y` automatically confirms the package manager's prompts. `mariadb-server` provides the database server daemon, `mariadb` provides the client and core MariaDB components, `mariadb-server-utils` provides server administration utilities, `mariadb-backup` provides backup tooling, and `mariadb-connector-c` provides client connectivity libraries. The package manager also installed supporting dependencies required by these components.

### ▶️ Step 3: Enable and start MariaDB

Enable the service and start it immediately:

~~~
sudo systemctl enable mariadb --now
~~~

The command created the service links and completed without an error:

~~~
Created symlink /etc/systemd/system/mysql.service → /usr/lib/systemd/system/mariadb.service.
Created symlink /etc/systemd/system/mysqld.service → /usr/lib/systemd/system/mariadb.service.
Created symlink /etc/systemd/system/multi-user.target.wants/mariadb.service → /usr/lib/systemd/system/mariadb.service.
~~~

> **Why:** `systemctl` manages services through systemd. `enable` configures MariaDB to start automatically during future boots, while `--now` starts it immediately in the current session. The service is named `mariadb`; the `mysql` and `mysqld` links are compatibility aliases. This starts the database service without needing a separate restart command.

### 🖥️ Step 4: Open the MariaDB client

Start an administrative MariaDB session:

~~~
sudo mysql
~~~

The MariaDB prompt is displayed:

~~~
MariaDB [(none)]>
~~~

> **Why:** `sudo` runs the client with the local administrative privileges needed for initial database setup. `mysql` opens the MariaDB command-line client. SQL statements entered at the `MariaDB` prompt are interpreted by the running database server, not by the Linux shell.

### 🗄️ Step 5: Create the application database

At the `MariaDB [(none)]>` prompt, create the requested database:

~~~sql
CREATE DATABASE kodekloud_db4;
~~~

MariaDB returned:

~~~
Query OK, 1 row affected (0.000 sec)
~~~

> **Why:** `CREATE DATABASE` creates a new logical database inside the MariaDB server. `kodekloud_db4` is the exact database name required by the application and the lab.

### 👤 Step 6: Create the database user

Create the user and set its password:

~~~sql
CREATE USER 'kodekloud_sam'@'localhost' IDENTIFIED BY 'B4zNgHA7Ya';
~~~

MariaDB returned:

~~~
Query OK, 0 rows affected (0.000 sec)
~~~

> **Why:** `CREATE USER` creates a MariaDB account. `'kodekloud_sam'@'localhost'` identifies both the username and the host from which this account may authenticate; `localhost` restricts this account to local connections. `IDENTIFIED BY` assigns the password required by the task. The password is entered in the interactive client so it is not exposed as a shell command argument.

### 🔐 Step 7: Grant full permissions on the database

Grant all privileges on the requested database to the new user:

~~~sql
GRANT ALL PRIVILEGES ON kodekloud_db4.* TO 'kodekloud_sam'@'localhost';
~~~

MariaDB returned:

~~~
Query OK, 0 rows affected (0.000 sec)
~~~

Refresh the privilege tables:

~~~sql
FLUSH PRIVILEGES;
~~~

> **Why:** `GRANT` assigns privileges to an account. `ALL PRIVILEGES` gives the user all permissions available at the specified scope, and `kodekloud_db4.*` limits that scope to every object in the `kodekloud_db4` database. `TO` identifies the account receiving the grant. `FLUSH PRIVILEGES` reloads privilege data; it is harmless here and was included in the successful lab workflow. Modern `CREATE USER` and `GRANT` statements update MariaDB's privilege system directly, so a manual flush is generally not required when using those statements.

### 🚪 Step 8: Exit the MariaDB client

Leave the interactive database session:

~~~sql
exit
~~~

The prompt returns to the Linux shell:

~~~
[root@stdb01 ~]#
~~~

> **Why:** `exit` closes the MariaDB client session. It does not stop, restart, or reconfigure the MariaDB service.

### ✅ Step 9: Verify

The required database configuration completed successfully:

~~~
CREATE DATABASE kodekloud_db4;
Query OK, 1 row affected (0.000 sec)

CREATE USER 'kodekloud_sam'@'localhost' IDENTIFIED BY 'B4zNgHA7Ya';
Query OK, 0 rows affected (0.000 sec)

GRANT ALL PRIVILEGES ON kodekloud_db4.* TO 'kodekloud_sam'@'localhost';
Query OK, 0 rows affected (0.000 sec)
~~~

The final state was:

| Resource | Result |
| --- | --- |
| MariaDB Server | Installed on `stdb01` |
| MariaDB service | Enabled and started |
| Database | `kodekloud_db4` created |
| Database user | `kodekloud_sam` created for `localhost` |
| Password | Set as required by the lab |
| Permissions | Full privileges on `kodekloud_db4` granted to `kodekloud_sam` |

> **Why:** The `Query OK` responses confirm that MariaDB accepted each SQL statement. The enabled and started service makes the database available now and after a future reboot.

## Best Practices

- **Use a dedicated application account.** The application user `kodekloud_sam` receives access to its own database instead of using the MariaDB administrative account.
- **Restrict the connection host when possible.** The `@'localhost'` host qualifier limits this account to local connections in this lab.
- **Grant only the required scope.** The privilege target `kodekloud_db4.*` avoids granting access to unrelated databases.
- **Protect credentials.** Do not place real database passwords in shell history, source control, chat, or public documentation; use a secret manager or an approved credential-injection mechanism.
- **Use the package manager for installation.** Installing `mariadb-server` with `yum` resolves and records dependencies consistently.
- **Enable the service after installation.** Enabling MariaDB prevents the application from losing database availability after a planned reboot.
- **Avoid unnecessary service restarts.** Database creation, user creation, and privilege changes take effect without restarting MariaDB.

### 📚 Official Documentation

- [Installing MariaDB Server](https://mariadb.com/docs/server/mariadb-quickstart-guides/installing-mariadb-server-guide)
- [Starting and stopping MariaDB with systemd](https://mariadb.com/docs/server/server-management/starting-and-stopping-mariadb/systemd)
- [MariaDB CREATE USER](https://mariadb.com/docs/server/reference/sql-statements/account-management-sql-statements/create-user)
- [MariaDB GRANT](https://mariadb.com/docs/server/reference/sql-statements/account-management-sql-statements/grant)
- [MariaDB CREATE DATABASE](https://mariadb.com/docs/server/reference/sql-statements/data-definition/create)
- [MariaDB SHOW GRANTS](https://mariadb.com/docs/server/reference/sql-statements/administrative-sql-statements/show/show-grants)
