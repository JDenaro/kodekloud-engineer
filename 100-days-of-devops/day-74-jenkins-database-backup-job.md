# Day 74: Jenkins Database Backup Job

There is a requirement to create a Jenkins job to automate the database backup. Below you can find more details to accomplish this task:

Click on the `Jenkins` button on the top bar to access the Jenkins UI. Login using username `admin` and password `[redacted]`.

## Specific Requirements:

1. Create a Jenkins job named `database-backup`.

2. Configure it to take a database dump of the `kodekloud_db01` database present on the **App server (stapp01)** in Stratos Datacenter, the database user is `kodekloud_roy` and password is `[redacted]`.

3. The dump should be named in `db_$(date +%F).sql` format, where `date +%F` is the current date.

4. Copy the `db_$(date +%F).sql` dump to the **Storage server (ststor01)** under location `/home/natasha/db_backups`.

5. Further, schedule this job to run periodically at `*/10 * * * *` (please use this exact schedule format).

`Note:`

1. You might need to install some plugins and restart Jenkins service. So, we recommend clicking on `Restart Jenkins when installation is complete and no jobs are running` on plugin installation/update page i.e `update centre`. Also, Jenkins UI sometimes gets stuck when Jenkins service restarts in the back end. In this case please make sure to refresh the UI page.

2. Please make sure to define you cron expression like this `*/10 * * * *` (this is just an example to run job every 10 minutes).

3. For these kind of scenarios requiring changes to be done in a web UI, please take screenshots so that you can share it for review in case your task is marked incomplete. You may also consider using a screen recording software such as loom.com to record and share your work.

> **Credential note:** Passwords from the temporary lab are redacted in this public guide. Use the credentials provided in your active lab; do not commit them to the repository.

## Solution

The backup crosses two connections: **Jenkins → `stapp01`** to start a script, then **`stapp01` → `ststor01`** to deliver the SQL file. Jenkins uses Publish Over SSH for the first connection; a dedicated SSH key allows the second connection to run without a password prompt.

### 🔎 Step 1: Check Database Access on App Server 1

From the jump host, connect to `stapp01` and check for a dump client:

```bash
ssh tony@stapp01
command -v mysqldump
mysql -u kodekloud_roy -p kodekloud_db01 -e "SHOW TABLES;"
```

Enter the database password from the active lab when prompted. In this run, `mysqldump` was at `/usr/bin/mysqldump`. The `SHOW TABLES` query returned no rows but no error: the database existed and was accessible, although it contained no tables yet.

> **Why:** `ssh` opens a session on the server that hosts the database. `command -v` locates the installed dump utility. The MySQL client uses `-u` for the database user, `-p` to prompt for the password without placing it in shell history, and `-e` to run one SQL statement against the named database. An empty table list is different from an authentication failure; even an empty database can have a valid backup of its structure.

### 🔐 Step 2: Store Database Credentials Privately

Still on `stapp01`, set private defaults and create an option file:

```bash
umask 077
vi /home/tony/.db-backup.cnf
```

Enter the following content, replacing the placeholder with the database password from the active lab:

```ini
[client]
user=kodekloud_roy
password=<database password from the active lab>
```

Then secure the file and test the connection:

```bash
chmod 600 /home/tony/.db-backup.cnf
mysql --defaults-extra-file=/home/tony/.db-backup.cnf -e "SELECT DATABASE();" kodekloud_db01
exit
```

The test returned `kodekloud_db01`.

> **Why:** `umask 077` makes newly created files private to their owner. The `[client]` section supplies credentials to MySQL-compatible command-line clients; `chmod 600` explicitly permits only `tony` to read and write the file. `--defaults-extra-file` points the client at that file and must appear before other client options. The `SELECT DATABASE()` query verifies that unattended commands can connect to the intended database without exposing the password in the Jenkins job or process arguments.

### 📁 Step 3: Prepare the Storage Destination

From the jump host, connect to Storage and create the requested directory:

```bash
ssh natasha@ststor01
mkdir -p /home/natasha/db_backups
ls -ld /home/natasha/db_backups
exit
```

The directory was created under `natasha`'s home and was owned by `natasha`.

> **Why:** `mkdir -p` creates the destination if it does not already exist. `ls -ld` inspects the directory itself, including ownership and permissions. The receiving SSH user needs write access there before the scheduled job can deliver a dump.

### 🔑 Step 4: Allow App Server 1 to Reach Storage Unattended

Reconnect to `stapp01` and create a dedicated SSH key:

```bash
ssh tony@stapp01
ssh-keygen -t ed25519 -f /home/tony/.ssh/db-backup-key -N ''
ssh-copy-id -i /home/tony/.ssh/db-backup-key.pub natasha@ststor01
```

The lab reported `Number of key(s) added: 1`. Verify that the connection works without an interactive password:

```bash
ssh -i /home/tony/.ssh/db-backup-key -o BatchMode=yes natasha@ststor01 hostname
```

The command printed `ststor01`.

> **Why:** `ssh-keygen` creates a key pair; `-t ed25519` selects its algorithm, `-f` fixes the key's path, and `-N ''` leaves it without a passphrase for unattended lab runs. `ssh-copy-id -i` installs only the public key for `natasha`, while the private key stays on `stapp01`. The test uses `ssh -i` to choose that private key and `-o BatchMode=yes` to fail instead of waiting for a password prompt. The `hostname` result confirms the connection reached Storage.

### 🗃️ Step 5: Create the Backup Script on App Server 1

While still logged in as `tony`, create `/home/tony/db-backup.sh`:

```bash
vi /home/tony/db-backup.sh
```

Use this script:

```sh
#!/bin/sh
set -eu
umask 077
backup_file="/home/tony/db_$(date +%F).sql"
mysqldump --defaults-extra-file=/home/tony/.db-backup.cnf --databases kodekloud_db01 > "$backup_file"
test -s "$backup_file"
scp -i /home/tony/.ssh/db-backup-key -o BatchMode=yes "$backup_file" natasha@ststor01:/home/natasha/db_backups/
```

Check its syntax and restrict access:

```bash
chmod 700 /home/tony/db-backup.sh
sh -n /home/tony/db-backup.sh
exit
```

`sh -n` produced no output, indicating that the shell accepted the script's syntax.

> **Why:** `set -e` stops the script if a command fails; `set -u` rejects unset variables. `umask 077` keeps the new SQL file private. `date +%F` produces a `YYYY-MM-DD` date, and the shell substitutes it into the required `db_...sql` filename. `mysqldump` writes the database dump to that file; its first option loads the private credentials, and `--databases` includes the database definition even when there are no tables. `test -s` requires a nonempty dump before transfer. `scp` copies the file using the dedicated key and noninteractive SSH to the exact Storage path. `chmod 700` allows only `tony` to read or execute the script; `sh -n` checks syntax without running a backup.

### 🔗 Step 6: Configure the Jenkins-to-App-Server Connection

1. Open **Jenkins** from the lab toolbar and sign in as `admin` with the active lab password.
2. If needed, install **Publish Over SSH** from **Manage Jenkins → Plugins**. Select **Restart Jenkins when installation is complete and no jobs are running** if requested, then wait for the login page and sign in again.
3. Open **Manage Jenkins → System → Publish over SSH → SSH Servers**. Add a server with **Name** `stapp01`, **Hostname** `stapp01`, **Username** `tony`, and **Remote Directory** `/tmp`.
4. In the server's advanced settings, use password authentication with `tony`'s active lab password. Leave **Disable exec** off. Run **Test Configuration**, then save once it succeeds.

> **Why:** Publish Over SSH lets Jenkins start a command on `stapp01` without copying the database through the Jenkins workspace. The saved SSH server configuration handles the first connection, while the script's key handles the separate Storage connection. `/tmp` is a valid remote directory for the plugin even though this job sends no files. **Test Configuration** catches connection or authentication problems before the first build.

### ⏰ Step 7: Create and Schedule `database-backup`

1. Select **New Item**, enter `database-backup`, choose **Freestyle project**, and create it.
2. Under **Build Triggers**, enable **Build periodically** and enter the exact schedule:

   ```text
   */10 * * * *
   ```

3. Under **Build Steps**, add **Send files or execute commands over SSH**. Select `stapp01`, add a transfer set if needed, leave **Source files** empty, and set **Exec command** to:

   ```sh
   /home/tony/db-backup.sh
   ```

4. Save the job and select **Build Now** at least once.

> **Why:** A Freestyle job is sufficient for this scheduled command. The five cron fields represent minute, hour, day of month, month, and day of week; `*/10 * * * *` triggers at minutes 0, 10, 20, 30, 40, and 50 of every hour. The plugin invokes the script on `stapp01`. **Source files** stays empty because the script, not the plugin, copies the dump to Storage. **Build Now** proves the configuration works without waiting for the next scheduled time.

### ✅ Step 8: Verify

Open the build's **Console Output** in Jenkins. The successful run reported:

```text
SSH: Connecting with configuration [stapp01] ...
SSH: EXEC: completed after 601 ms
SSH: Transferred 0 file(s)
Build step 'Send files or execute commands over SSH' changed build result to SUCCESS
Finished: SUCCESS
```

Then verify the generated file on Storage:

```bash
ssh natasha@ststor01
ls -l /home/natasha/db_backups/
```

The lab produced `db_2026-09-29.sql` on `ststor01`: it was 1514 bytes, owned by `natasha`, and had private `-rw-------` permissions. The lab was marked successful.

> **Why:** The Jenkins result confirms that the remote script exited successfully. **Transferred 0 file(s)** only means the plugin itself did not upload a file; the script's `scp` made the required second-hop transfer. `ls -l` confirms the dated file exists in the exact destination with the expected ownership and restrictive permissions. The filename date comes from the App Server's clock, which may differ from a local workstation's date around midnight.

## Best Practices

- **Protect credentials.** Keep the database password in a private option file or a managed secret store, never in the Jenkins job command, shell history, screenshots, or a public repository.
- **Protect the SSH key.** The passphrase-free key enables unattended execution in this lab. In production, restrict access to the private key, scope the authorized public key to this task, and rotate it.
- **Check the destination, not just Jenkins.** A green build is useful, but the actual dated SQL file on Storage proves that the second SSH hop succeeded.
- **Account for same-day reruns.** The required date-only filename is overwritten by later successful runs on the same day. For production retention, use a unique timestamp or versioned storage without changing this lab's required filename.
- **Test restore procedures.** A dump file is only a useful backup if it can be restored; periodically test restores in an isolated environment.

### 📚 Official Documentation

- [Publish Over SSH plugin](https://plugins.jenkins.io/publish-over-ssh/)
- [Jenkins cron syntax](https://www.jenkins.io/doc/book/pipeline/syntax/#jenkins-cron-syntax)
- [MariaDB dump client](https://mariadb.com/docs/server/clients-and-utilities/backup-restore-and-import-clients/mariadb-dump)
- [MariaDB option files](https://mariadb.com/docs/server/server-management/install-and-upgrade-mariadb/configuring-mariadb/configuring-mariadb-with-option-files)
- [Red Hat: Using secure communications between two systems with OpenSSH](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/configuring_basic_system_settings/assembly_using-secure-communications-between-two-systems-with-openssh_configuring-basic-system-settings)
- [OpenBSD scp manual](https://man.openbsd.org/scp.1)
