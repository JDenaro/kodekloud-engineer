# Day 68: Set Up Jenkins Server

The DevOps team at xFusionCorp Industries is initiating the setup of CI/CD pipelines and has decided to utilize Jenkins as their server. Execute the task according to the provided requirements:

> **Credential note:** The lab passwords in the original challenge are redacted from this public guide. Read them from your active KodeKloud task; do not reuse credentials from another lab session.

## Specific Requirements:

1. Install `Jenkins` on the jenkins server using the `apt` utility only, and start it using the `service` command.

   - If you face a timeout issue while starting the Jenkins service, first check the service status with `service jenkins status`
   - Then review the logs in `/var/log/jenkins/jenkins.log` to identify the cause.

2. Jenkin's admin user name should be `theadmin`, password should be `[redacted]`, full name should be `Javed` and email should be `javed@jenkins.stratos.xfusioncorp.com`.

`Note:`

1. To access the `jenkins` server, connect from the jump host using the `root` user with the password `[redacted]`.

2. After Jenkins server installation, click the `Jenkins` button on the top bar to access the Jenkins UI and follow on-screen instructions to create an admin user.

## Solution

Install Java before Jenkins, then start Jenkins with `service` as the lab requires. The Jenkins web page returned `HTTP ERROR 431 Request Header Fields Too Large` in the regular browser session during this lab. Opening the lab's Jenkins URL in an Incognito tab allowed the setup wizard to continue. The exact oversized header was not identified, so there was no reason to change the Jenkins server configuration.

### 🔐 Step 1: Connect to the Jenkins Server

From the jump host:

```bash
ssh root@jenkins
```

Enter the root password shown in the current lab prompt when SSH asks for it.

> **Why:** `ssh` opens a remote shell on the `jenkins` host. Connecting as `root` follows the task instructions and lets the subsequent package and service commands run without `sudo`. The password is entered interactively rather than placed in the command or this guide.

### 📦 Step 2: Update the Package Index

```bash
apt update
```

> **Why:** `apt update` refreshes the package index so `apt` knows which package versions are available from the configured repositories. It does not install Jenkins by itself.

### ☕ Step 3: Install Java

```bash
apt install -y fontconfig openjdk-21-jre
```

> **Why:** Jenkins runs on Java. `openjdk-21-jre` supplies the Java runtime, and `fontconfig` is included in the official Debian/Ubuntu installation prerequisites. Installing Java first avoids a service-start failure caused by a missing compatible runtime. `-y` automatically confirms the package installation prompt.

### 🧩 Step 4: Install Jenkins

```bash
apt install -y jenkins
```

> **Why:** This installs Jenkins through `apt`, the package utility required by the challenge. The lab environment already provides the Jenkins package repository, so no extra repository configuration is needed in this successful path.

### 🚀 Step 5: Start the Service

```bash
service jenkins start
service jenkins status
```

> **Why:** `service jenkins start` launches the Jenkins server using the command specified in the task. `service jenkins status` checks whether it is running before attempting the browser setup. The expected status is `jenkins is running`.

If starting Jenkins times out, follow the task's diagnostic order before changing anything:

```bash
service jenkins status
cat /var/log/jenkins/jenkins.log
```

> **Why:** The status shows whether Jenkins actually started despite the timeout. `cat` displays the service log so the cause can be identified. This is a conditional troubleshooting path, not a claim that a timeout occurred in the successful run.

### 🔑 Step 6: Get the One-Time Unlock Password

```bash
cat /var/lib/jenkins/secrets/initialAdminPassword
```

> **Why:** A fresh Jenkins installation generates a one-time password for the **Unlock Jenkins** page. `cat` displays it in your terminal. Use the value from this lab only, and do not paste it into notes, screenshots, or the repository.

### 🕶️ Step 7: Open Jenkins in an Incognito Tab

1. Click the **Jenkins** button in the lab's top bar.
2. Copy the URL of the Jenkins page.
3. Open an **Incognito** browser tab or window and paste that URL there.
4. Wait for the **Unlock Jenkins** page.

> **Why:** The normal browser session returned `HTTP 431`, which means the server rejected request headers that were too large. Incognito starts a separate browser session without reusing the regular session's stored site data. In this lab it allowed Jenkins to load. This result points toward browser-session headers, but it does not prove which particular header caused the error. Changing Jetty's header limits was unnecessary.

### 👤 Step 8: Create the Administrator in the UI

1. Paste the one-time unlock password from Step 6 into **Unlock Jenkins** and select **Continue**.
2. Choose **Install suggested plugins** and wait for the installation to finish.
3. On **Create First Admin User**, enter these values:

   | Field | Value |
   | --- | --- |
   | Username | `theadmin` |
   | Password and confirmation | Use the admin password in the current lab prompt |
   | Full name | `Javed` |
   | Email address | `javed@jenkins.stratos.xfusioncorp.com` |

4. Select **Save and Finish**, then **Start using Jenkins**.

> **Why:** The initial password only unlocks the setup wizard. Installing the suggested plugins provides Jenkins' standard starting features, and the final form creates the named administrator required by the task. The password is deliberately not reproduced in this guide.

### ✅ Step 9: Verify

On the Jenkins server, check the service once more:

```bash
service jenkins status
```

The expected result is `jenkins is running`. In the Incognito tab, confirm that the Jenkins dashboard opens and the administrator account is `theadmin`. The lab reported success after the UI setup was completed.

> **Why:** A running service confirms the server-side requirement; reaching the dashboard as the new user confirms the UI-based administrator setup. Both checks are needed because package installation alone does not create the requested admin account.

## Best Practices

- **Install a compatible Java runtime first.** Jenkins depends on Java and may not start without a supported version.
- **Keep lab credentials out of documentation.** Use the passwords from the active challenge prompt and avoid copying the generated unlock password into files or screenshots.
- **Diagnose browser errors before changing the server.** A `431` concerns request headers. Trying a separate browser session helps isolate client-side state without loosening server limits.
- **Check service status before reading logs.** If a start command times out, the service may still have reached a running state; inspect its status before making configuration changes.

### 📚 Official Documentation

- [Installing Jenkins on Linux](https://www.jenkins.io/doc/book/installing/linux/)
- [Managing Jenkins users](https://www.jenkins.io/doc/book/managing/users/)
- [HTTP 431 Request Header Fields Too Large](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status/431)
- [Browse in Incognito mode in Chrome](https://support.google.com/chrome/answer/95464/browse-in-private-computer?hl=en-GB)
