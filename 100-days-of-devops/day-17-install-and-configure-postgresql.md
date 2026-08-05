# Day 17: Install and Configure PostgreSQL

The Nautilus application development team has shared that they are planning to deploy one newly developed application on Nautilus infra in Stratos DC. The application uses PostgreSQL database, so as a pre-requisite we need to set up PostgreSQL database server as per requirements shared below:

PostgreSQL database server is already installed on the Nautilus database server.

a. Create a database user kodekloud_joy and set its password to dCV3szSGNA.

b. Create a database kodekloud_db9 and grant full permissions to user kodekloud_joy on this database.

Note: Please do not try to restart PostgreSQL server service.

## Specific Requirements:

1. PostgreSQL is already installed on the Nautilus database server.
2. Create the database user `kodekloud_joy` with the password provided in the task.
3. Create the database `kodekloud_db9`.
4. Grant full database privileges on `kodekloud_db9` to `kodekloud_joy`.
5. Do not restart the PostgreSQL service.

## Solution

The database server was `stdb01`, accessed with the `peter` user. PostgreSQL 13.23 was already installed. The administrative `postgres` operating-system account was used to open `psql`, the PostgreSQL interactive terminal.

The user was created without placing the password in a shell command. The `\password` command requested the password interactively, which avoids exposing it in shell history or in the process list. The database was then created and all database-level privileges were granted to the new user. PostgreSQL was not restarted.

### 🔌 Step 1: Connect to the database server

From the Jump Host, connect to `stdb01`:

~~~
ssh peter@stdb01
~~~

> **Why:** `ssh` opens a secure remote shell. `peter` is the login account for the database server, and `stdb01` is the host where PostgreSQL is installed.

### 🐘 Step 2: Open PostgreSQL as the administrative account

~~~
sudo -u postgres psql
~~~

The PostgreSQL prompt appeared:

~~~
psql (13.23)
Type "help" for help.

postgres=#
~~~

> **Why:** `sudo -u postgres` runs the following command as the operating-system user `postgres`. `psql` is PostgreSQL's interactive terminal. The `postgres` account has the administrative privileges needed to create roles, databases, and grants. This command opens a database session and does not restart the PostgreSQL service.

### 👤 Step 3: Create the database user

At the `postgres=#` prompt, run:

~~~sql
CREATE USER kodekloud_joy;
~~~

The command returned:

~~~
CREATE ROLE
~~~

Set the password interactively:

~~~sql
\password kodekloud_joy
~~~

When prompted, enter the password supplied by the task and enter it again for confirmation. The password is not displayed while it is typed.

> **Why:** `CREATE USER` creates a login-capable PostgreSQL role; in PostgreSQL, a user is implemented as a role with the `LOGIN` attribute. The command creates the account without granting superuser, database-creation, or role-management privileges. `\password` is a `psql` meta-command that prompts for a new password without placing the secret in the SQL command or shell history. Keeping credentials out of command arguments reduces accidental exposure in terminal history and process inspection.

### 🗄️ Step 4: Create the application database

Still at the `postgres=#` prompt, run:

~~~sql
CREATE DATABASE kodekloud_db9;
~~~

The command returned:

~~~
CREATE DATABASE
~~~

> **Why:** `CREATE DATABASE` creates a new database in the PostgreSQL cluster. Creating it as the administrative `postgres` account ensures the command has the required database-creation privilege. No service restart is needed for a new database to become available.

### 🔐 Step 5: Grant database privileges

Grant the requested privileges to the new user:

~~~sql
GRANT ALL PRIVILEGES ON DATABASE kodekloud_db9 TO kodekloud_joy;
~~~

The command returned:

~~~
GRANT
~~~

> **Why:** `GRANT ALL PRIVILEGES ON DATABASE` assigns all privileges available at the database level to the specified role. The database name identifies the target database, and `TO kodekloud_joy` identifies the role receiving the grant. This gives the application user the requested database-level access without making it a PostgreSQL superuser.

### 🚪 Step 6: Exit PostgreSQL

~~~
\q
~~~

The session returned to the server shell:

~~~
[peter@stdb01 ~]$
~~~

> **Why:** `\q` is the `psql` command for quitting the interactive terminal. It closes the client session only; it does not stop or restart the PostgreSQL server.

### ✅ Step 7: Final result

The required operations completed successfully:

~~~
CREATE ROLE
CREATE DATABASE
GRANT
~~~

The final state was:

| Resource | Result |
| --- | --- |
| PostgreSQL user | `kodekloud_joy` created |
| Database | `kodekloud_db9` created |
| Database privileges | All database-level privileges granted to `kodekloud_joy` |
| PostgreSQL service | Not restarted |

> **Why:** The command results confirm that the role, database, and grant were accepted by PostgreSQL. The service remained running throughout the task, respecting the instruction not to restart it.

## Best Practices

- **Use the PostgreSQL administrative account only for setup.** The `postgres` account was used to create the application role and database, not as the application credential.
- **Avoid exposing passwords in commands.** `\password` reads the password interactively instead of placing it in shell history or command arguments.
- **Do not grant superuser privileges unnecessarily.** `CREATE USER` creates a normal login role; the task-specific database grant is sufficient.
- **Grant privileges at the required scope.** The grant targets `kodekloud_db9` rather than granting broad access across the PostgreSQL cluster.
- **Do not restart a running database service without a requirement.** Creating roles, databases, and grants takes effect immediately.
- **Keep database and role names exact.** PostgreSQL identifiers are part of the application's connection configuration.

### 📚 Official Documentation

- [PostgreSQL 13 CREATE USER](https://www.postgresql.org/docs/13/sql-createuser.html)
- [PostgreSQL 13 CREATE ROLE](https://www.postgresql.org/docs/13/sql-createrole.html)
- [PostgreSQL 13 CREATE DATABASE](https://www.postgresql.org/docs/13/sql-createdatabase.html)
- [PostgreSQL 13 GRANT](https://www.postgresql.org/docs/13/sql-grant.html)
- [PostgreSQL 13 psql](https://www.postgresql.org/docs/13/app-psql.html)
- [PostgreSQL 13 privileges](https://www.postgresql.org/docs/13/ddl-priv.html)
