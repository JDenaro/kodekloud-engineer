# Day 71: Configure Jenkins Job for Package Installation

Some new requirements have come up to install and configure some packages on the Nautilus infrastructure under Stratos Datacenter. The Nautilus DevOps team installed and configured a new Jenkins server so they wanted to create a Jenkins job to automate this task. Find below more details and complete the task accordingly:

## Specific Requirements:

1. Access the Jenkins UI by clicking on the `Jenkins` button in the top bar. Log in using the credentials: username `admin` and password `[redacted]`.

2. Create a new Jenkins job named `install-packages` and configure it with the following specifications:

   - Add a string parameter named `PACKAGE`.
   - Configure the job to install a package specified in the `$PACKAGE` parameter on the `storage server` (Stratos Datacenter).
   - Build the job at least once (e.g. with parameter `PACKAGE=vim-enhanced`) so the package is installed on the Storage server and can be verified.

`Note`:

1. Ensure to install any required plugins and restart the Jenkins service if necessary. Opt for `Restart Jenkins when installation is complete and no jobs are running` on the plugin installation/update page. Refresh the UI page if needed after restarting the service.

2. Verify that the Jenkins job runs successfully on repeated executions to ensure reliability.

3. Capture screenshots of your configuration for documentation and review purposes. Alternatively, use screen recording software like `loom.com` for comprehensive documentation and sharing.

> **Credential note:** The temporary password in the original challenge is redacted from this public guide. Enter the value shown in your active KodeKloud task.

## Solution

The important distinction is **where the command runs**. A normal Jenkins shell build step runs on the Jenkins server; it does not install a package on the Storage server. The successful job used the **Publish Over SSH** plugin to execute the installation on `ststor01` as `natasha`.

### 🔐 Step 1: Sign In and Install Publish Over SSH

1. Click **Jenkins** in the lab's top bar and sign in as `admin` with the password from the current lab prompt.
2. Open **Manage Jenkins → Plugins**, find **Publish Over SSH** under **Available plugins**, and install it.
3. If Jenkins requests a restart, select **Restart Jenkins when installation is complete and no jobs are running**. Wait for the login page to return before continuing.

> **Why:** Publish Over SSH adds a build step that opens an SSH connection to another server and executes a command there. Installing the plugin makes that step available to the Freestyle job. A restart may be needed before Jenkins loads the plugin.

### 🔎 Step 2: Check the Storage Server

From the jump host, connect to the Storage server using the lab's `natasha` password:

```bash
ssh natasha@ststor01
```

On `ststor01`, check the package manager and non-interactive sudo access:

```bash
command -v dnf
sudo -n id
```

In this lab, `dnf` was available at `/usr/bin/dnf`, and `sudo -n id` returned `uid=0(root)`. The server reported **CentOS Stream 9**.

> **Why:** `ssh` logs in to the machine where the package must actually be installed. `command -v dnf` checks that its package manager is available. `sudo -n id` verifies that `natasha` can run a privileged command without an interactive password prompt: `-n` makes sudo fail instead of waiting for input, which matters for an unattended Jenkins build. Installing packages requires administrator privileges.

### 🔗 Step 3: Configure the SSH Connection in Jenkins

1. Open **Manage Jenkins → System → Publish over SSH → SSH Servers** and add a server.
2. Set **Name** to `ststor01`, **Hostname** to `ststor01`, **Username** to `natasha`, and **Remote Directory** to `/tmp`.
3. Under the server's advanced authentication settings, provide `natasha`'s current lab password. Do not put this password in the job command or in this guide.
4. Select **Test Configuration** and save when the connection succeeds.

> **Why:** This named SSH configuration tells Jenkins which host to contact and which account to use. `/tmp` is a suitable remote directory for the plugin even though this job transfers no files. Testing the connection before configuring the build separates an SSH/authentication problem from a package-installation problem.

### 📦 Step 4: Create the Parameterized Job

1. Select **New Item**, enter `install-packages`, choose **Freestyle project**, and create the job.
2. In **Configure → General**, enable **This project is parameterized**.
3. Add a **String Parameter** named `PACKAGE`; use `vim-enhanced` as its default value.

> **Why:** A string parameter lets the person starting a build choose the package without editing the job each time. Jenkins makes `PACKAGE` available to the build as an environment variable. The default gives this lab an easy first test, but the build still accepts other package names.

### 🛠️ Step 5: Install the Package on `ststor01`

In **Build Steps**, add **Send files or execute commands over SSH**, select the `ststor01` SSH configuration, and enter this **Exec command**. Leave file-transfer fields empty:

```sh
set -e
hostname
sudo -n dnf install -y "$PACKAGE"
rpm -q "$PACKAGE"
```

Save the job.

> **Why:** The plugin runs this command through SSH on `ststor01`, not in the Jenkins workspace. `set -e` stops the remote shell if a command fails. `hostname` makes the target visible in command output. `sudo -n` runs `dnf` with administrator privileges without prompting; `dnf install -y` installs the selected package and automatically answers yes to confirmation prompts. `rpm -q` queries the installed RPM package and makes the step fail if it cannot find it. Jenkins expands the `PACKAGE` build parameter for the SSH command. Running `dnf install` again for an already-installed package is safe for this repeat-build check.

### ✅ Step 6: Verify

1. Select **Build with Parameters** and run the job with `PACKAGE=vim-enhanced`.
2. Open **Console Output** and confirm the SSH step uses `ststor01` and the build ends with **Finished: SUCCESS**.
3. Run the job a second time with the same parameter and check that it also finishes successfully. The second run in this lab reported:

   ```text
   SSH: Connecting with configuration [ststor01] ...
   SSH: EXEC: completed after 1,200 ms
   Build step 'Send files or execute commands over SSH' changed build result to SUCCESS
   Finished: SUCCESS
   ```

4. On the Storage server, confirm the package is installed:

   ```bash
   rpm -q vim-enhanced
   ```

   The lab returned `vim-enhanced-8.2.2637-41.el9.x86_64`.
5. Capture screenshots of the job parameter, SSH build step, and successful Console Output if required for review. Then submit the lab.

> **Why:** A green Jenkins build alone does not prove the package is on the required host. The SSH configuration in Console Output confirms the remote step was used, and `rpm -q` on `ststor01` confirms the package is present there. The second successful build checks that the job can run repeatedly. The lab reported success after these checks.

## Best Practices

- **Install on the target host.** A local Jenkins shell step would change the Jenkins server, not `ststor01`; use a remote execution step for this requirement.
- **Keep credentials in Jenkins configuration.** Do not paste SSH passwords into the job's command, screenshots, or a public repository.
- **Make remote failures fail the build.** `set -e` and `rpm -q` prevent an installation failure from being hidden by a later successful command.
- **Restrict who can choose packages.** A build parameter is user-controlled input. In production, limit who can run this privileged job and validate package names before using them in a shell command.
- **Verify repeatability and destination.** Re-running `dnf install` should succeed when the package is already installed, and the final package check must run on the Storage server.

### 📚 Official Documentation

- [Publish Over SSH plugin](https://plugins.jenkins.io/publish-over-ssh/)
- [Jenkins: Using environment variables](https://www.jenkins.io/doc/book/security/environment-variables/)
- [Jenkins: Managing plugins](https://www.jenkins.io/doc/book/managing/plugins/)
- [Red Hat Enterprise Linux 9: Managing software with DNF](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/managing_software_with_the_dnf_tool/index)
