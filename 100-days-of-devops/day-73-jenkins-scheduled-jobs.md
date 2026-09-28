# Day 73: Jenkins Scheduled Jobs

The devops team of xFusionCorp Industries is working on to setup centralised logging management system to maintain and analyse server logs easily. Since it will take some time to implement, they wanted to gather some server logs on a regular basis. At least one of the app servers is having issues with the Apache server. The team needs Apache logs so that they can identify and troubleshoot the issues easily if they arise. So they decided to create a Jenkins job to collect logs from the server. Please create/configure a Jenkins job as per details mentioned below:

Click on the `Jenkins` button on the top bar to access the Jenkins UI. Login using username `admin` and password `[redacted]`

## Specific Requirements:

1. Create a Jenkins jobs named `copy-logs`.

2. Configure it to periodically build `every 7 minutes` to copy the Apache logs (both `access_log` and `error_log`) from **App Server 3** (stapp03) from the default logs location to location `/usr/src/dba` on the Storage Server.

3. Build the job at least once so that the logs are copied and can be verified.

`Note:`

1. You might need to install some plugins and restart Jenkins. We recommend selecting `Restart Jenkins when installation is complete and no jobs are running` in the update centre. Refresh the page if the UI gets stuck after a restart.

2. Define the cron expression as required (e.g. `*/10 * * * *` to run every 10 minutes).

3. For scenarios that require web UI changes, take screenshots or record your work (e.g. using loom.com) so you can share it for review if the task is marked incomplete.

> **Credential note:** The temporary Jenkins password in the original challenge is redacted from this public guide. Use the credentials shown in the active lab; do not record server passwords here either.

## Solution

This job needs two SSH connections: **Jenkins → `stapp03`** to run a command, then **`stapp03` → `ststor01`** to copy the files. The successful setup used the Publish Over SSH plugin for the first connection and a dedicated SSH key for the second. Neither connection can wait for someone to type a password during a scheduled build.

### 🔎 Step 1: Check the Log Files and Destination

From the jump host, connect to App Server 3:

```bash
ssh banner@stapp03
ls -l /var/log/httpd/access_log /var/log/httpd/error_log
exit
```

Both Apache logs existed under `/var/log/httpd/`. In this lab, `access_log` was 41 bytes and `error_log` was 732 bytes; both were readable without `sudo`.

Next, check the destination on the Storage Server:

```bash
ssh natasha@ststor01
ls -ld /usr/src/dba
sudo -n id
```

The directory did not yet exist, and `sudo -n id` returned `uid=0(root)`, showing that `natasha` could perform the one-time setup without an interactive sudo prompt.

> **Why:** The job must read the files from the real Apache log directory, not from the Jenkins workspace. `ls -l` shows their existence and read permissions; `ls -ld` checks whether the destination directory is already available. `sudo -n id` checks non-interactive administrator access: `-n` makes sudo fail rather than asking for a password. These checks explain why the source needs no privileged read while the destination needs initial creation.

### 📁 Step 2: Create the Storage Directory

While still logged in as `natasha` on `ststor01`, run:

```bash
sudo mkdir -p /usr/src/dba
sudo chown natasha:natasha /usr/src/dba
ls -ld /usr/src/dba
exit
```

The result showed `/usr/src/dba` owned by `natasha:natasha`.

> **Why:** `mkdir -p` creates the requested destination, including any missing parent directories. `chown` gives `natasha` permission to write the incoming log files without requiring sudo in every scheduled run. `ls -ld` verifies the directory itself rather than listing its contents. Only the new destination directory is changed; the source logs remain untouched.

### 🔑 Step 3: Allow Non-Interactive SSH from App Server 3 to Storage

Back on the jump host, connect to App Server 3 and generate a dedicated key pair:

```bash
ssh banner@stapp03
ssh-keygen -t ed25519 -f ~/.ssh/copy-logs-key -N ''
ssh-copy-id -i ~/.ssh/copy-logs-key.pub natasha@ststor01
```

Accept the verified Storage Server host key if prompted and enter `natasha`'s lab password for the one-time `ssh-copy-id` operation. The lab reported `Number of key(s) added: 1`. Then test that authentication no longer needs a password:

```bash
ssh -i ~/.ssh/copy-logs-key -o BatchMode=yes natasha@ststor01 hostname
exit
```

The test printed `ststor01` and returned without a password prompt.

> **Why:** `ssh-keygen` creates an SSH key pair. `-t ed25519` selects the key type, `-f` names the dedicated key file, and `-N ''` leaves the key without a passphrase so the scheduled job can use it unattended. `ssh-copy-id -i` places this key's public half in `natasha`'s authorized keys; the private half remains on `stapp03`. The final `ssh -i` selects that private key, while `-o BatchMode=yes` disables interactive password questions. The `hostname` output proves the connection reaches Storage.

### 🔗 Step 4: Connect Jenkins to App Server 3

1. Click **Jenkins** in the lab's top bar and sign in as `admin` with the active lab password.
2. Install **Publish Over SSH** from **Manage Jenkins → Plugins** if it is not already available. If Jenkins requests a restart, choose **Restart Jenkins when installation is complete and no jobs are running** and wait for the login page to return.
3. Open **Manage Jenkins → System → Publish over SSH → SSH Servers** and add a server with **Name** `stapp03`, **Hostname** `stapp03`, **Username** `banner`, and **Remote Directory** `/tmp`.
4. In the advanced server settings, enable password authentication and enter `banner`'s lab password. Leave **Disable exec** off. Select **Test Configuration** and save after it succeeds.

> **Why:** This plugin executes a build command remotely on `stapp03`. Its server configuration stores the first SSH connection separately from the job command, keeping the password out of the script. `/tmp` is a usable remote directory even though the plugin will transfer no files in this job. **Test Configuration** checks that Jenkins can reach and authenticate to App Server 3 before the first build.

### ⏰ Step 5: Create and Schedule `copy-logs`

1. Select **New Item**, enter `copy-logs`, choose **Freestyle project**, and create it.
2. Under **Build Triggers**, enable **Build periodically** and enter:

   ```text
   */7 * * * *
   ```

3. Under **Build Steps**, add **Send files or execute commands over SSH**. Choose the `stapp03` server, add a transfer set if needed, leave **Source files** empty, and enter this **Exec command**:

   ```sh
   scp -i /home/banner/.ssh/copy-logs-key -o BatchMode=yes /var/log/httpd/access_log /var/log/httpd/error_log natasha@ststor01:/usr/src/dba/
   ```

4. Save the job.

> **Why:** Jenkins uses five cron fields: minute, hour, day of month, month, and day of week. `*/7 * * * *` selects minutes 0, 7, 14, and so on in every hour. Because 60 is not divisible by 7, the interval across an hour boundary is shorter; this is a seven-minute step within each hour, not a perfect rolling seven-minute timer. The plugin runs the `scp` command on `stapp03`: `-i` selects the dedicated private key, `-o BatchMode=yes` prevents password prompts, the two `/var/log/httpd/` paths identify the source files, and `natasha@ststor01:/usr/src/dba/` identifies the destination. Leaving **Source files** empty is intentional because Jenkins itself does not transfer the logs.

### ✅ Step 6: Verify

Select **Build Now** at least once, then inspect **Build History → Console Output**. The successful lab run showed:

```text
SSH: Connecting with configuration [stapp03] ...
SSH: EXEC: completed after 601 ms
SSH: Transferred 0 file(s)
Build step 'Send files or execute commands over SSH' changed build result to SUCCESS
Finished: SUCCESS
```

From the jump host, verify both files on Storage:

```bash
ssh natasha@ststor01
ls -l /usr/src/dba/access_log /usr/src/dba/error_log
```

The copied `access_log` was 41 bytes and `error_log` was 732 bytes, matching the source sizes. The lab was marked successful.

> **Why:** `Finished: SUCCESS` confirms that Jenkins executed the remote command successfully. **Transferred 0 file(s)** refers only to the plugin's own file-transfer field; the remote `scp` command performed the copy. Listing both files on `ststor01` confirms they reached the required destination. Capture screenshots of the trigger, SSH build step, and successful output if the lab requests review evidence.

## Best Practices

- **Keep the two SSH hops distinct.** Jenkins authenticates to `stapp03`; the dedicated key authenticates `banner` to `natasha@ststor01`. A green Jenkins connection alone does not prove the second hop works.
- **Do not disable host-key checking.** The lab's two hosts presented the same SSH host-key fingerprint, which is unusual for independent servers. In a real environment, verify each fingerprint against a trusted source before accepting it.
- **Protect the private key.** The passphrase-free key is a lab convenience for unattended execution. In production, limit who can read it, restrict its authorized use, and manage rotation.
- **Keep credentials out of job scripts.** Store the Jenkins-to-App-Server password in Jenkins configuration, not in the `scp` command or screenshots.
- **Verify the destination.** Check the actual files on Storage after the first build; a successful scheduler setting alone does not demonstrate that logs were copied.

### 📚 Official Documentation

- [Publish Over SSH plugin](https://plugins.jenkins.io/publish-over-ssh/)
- [Jenkins cron syntax](https://www.jenkins.io/doc/book/pipeline/syntax/#jenkins-cron-syntax)
- [Red Hat: Using secure communications between two systems with OpenSSH](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/configuring_basic_system_settings/assembly_using-secure-communications-between-two-systems-with-openssh_configuring-basic-system-settings)
- [OpenBSD scp manual](https://man.openbsd.org/scp.1)
