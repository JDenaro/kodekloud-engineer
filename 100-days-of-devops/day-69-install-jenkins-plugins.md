# Day 69: Install Jenkins Plugins

The Nautilus DevOps team has recently setup a Jenkins server, which they want to use for some CI/CD jobs. Before that they want to install some plugins which will be used in most of the jobs. Please find below more details about the task

## Specific Requirements:

1. Click on the Jenkins button on the top bar to access the Jenkins UI. Login using username `admin` and password `[redacted]`.

2. Once logged in, install the `Git` and `GitLab` plugins. You may need to restart Jenkins to complete the plugin installation; if required, opt to **Restart Jenkins when installation is complete and no jobs are running** on the plugin installation/update page (Update Centre).

`Note:`

1. After restarting Jenkins, wait for the login page to reappear before proceeding.

2. For tasks involving web UI changes, capture screenshots to share for review or consider using screen recording software like loom.com for documentation and sharing.

> **Credential note:** The password from the original challenge is redacted from this public guide. Use the credentials shown in your active KodeKloud lab.

## Solution

The Git plugin initially reported that it could not load its **Credentials Binding** dependency. Installing Credentials Binding also showed an error during dynamic loading. After Jenkins was restarted, the required plugins appeared as installed. This outcome is consistent with plugin downloads completing while their activation still required a restart; the error alone did not establish a deeper dependency or version problem.

### 🕶️ Step 1: Open Jenkins in an Incognito Window

1. Open a private or Incognito browser window.
2. In the lab, click **Jenkins** on the top bar and open its URL in that private window.
3. Log in as `admin` using the password from the current lab prompt.

> **Why:** This lab's regular browser session could return `HTTP ERROR 431 Request Header Fields Too Large`. Incognito uses a separate browser session and allowed the Jenkins UI to load without changing the server. The error indicates oversized request headers, but this lab did not identify which header caused it. The lab credentials are temporary and should not be copied into a public guide.

### 🧩 Step 2: Select Git and GitLab

1. Go to **Manage Jenkins → Plugins → Available plugins**.
2. Search for **Git** and select the plugin named **Git**.
3. Search for **GitLab** and select the plugin named **GitLab**.
4. Start the installation from the plugin page and watch the installation progress.

> **Why:** The plugin manager downloads selected plugins and their dependencies from the Jenkins Update Center. **Git** enables Jenkins jobs to work with Git repositories; **GitLab** adds GitLab integration. Select the exact plugin names because searches can also return other Git-related plugins.

### 🔗 Step 3: Handle the Credentials Binding Dependency

1. If the Git installation reports `Failed to load: Credentials Binding Plugin`, return to **Manage Jenkins → Plugins → Available plugins**.
2. Find **Credentials Binding** and install it.
3. If dynamic loading still reports an error, use the installation/update page's restart option in the next step instead of repeatedly reinstalling the plugins.

> **Why:** Git depends on Credentials Binding for credential bindings used by authenticated Git operations. In this lab, both the Git installation and the separate Credentials Binding installation reported loading errors before the restart. The eventual success after restarting means the initial error was not evidence that the downloaded plugins were unusable.

### 🔄 Step 4: Restart Jenkins and Wait for Login

On the plugin installation/update page, select **Restart Jenkins when installation is complete and no jobs are running** if that option is available. Wait until Jenkins returns to its login page, then sign in again as `admin`.

> **Why:** A restart lets Jenkins load installed plugins and resolve dependencies during startup. This was the step that made the plugins appear correctly installed in the successful lab. Waiting for the login page confirms that Jenkins has finished restarting before verification.

### ✅ Step 5: Verify

1. Open **Manage Jenkins → Plugins → Installed plugins**.
2. Search for **Git**, **GitLab**, and **Credentials Binding**.
3. Confirm that the requested **Git** and **GitLab** plugins are installed and enabled. Capture a screenshot of the result if evidence is needed for review.

> **Why:** A completed download is not enough if a plugin failed to load. The installed-and-enabled view after the restart is the relevant UI check. In this lab, the plugins appeared correctly installed and the task was reported successful.

## Best Practices

- **Resolve dependency errors before repeating installations.** A failed parent plugin may be reporting that one of its dependencies did not load yet.
- **Allow a restart when activation requires one.** Some plugins cannot be loaded dynamically even after their files have downloaded; verify them after Jenkins is back online.
- **Use a separate browser session for this lab's HTTP 431.** Incognito avoided the oversized-header error without changing Jenkins or Jetty settings.
- **Keep credentials and screenshots private.** Do not place lab passwords, authenticated browser data, or screenshots containing secrets in a public repository.

### 📚 Official Documentation

- [Managing Jenkins plugins](https://www.jenkins.io/doc/book/managing/plugins/)
- [Git plugin](https://plugins.jenkins.io/git/)
- [GitLab plugin](https://plugins.jenkins.io/gitlab-plugin/)
- [Credentials Binding plugin](https://plugins.jenkins.io/credentials-binding/)
- [Browse in Incognito mode in Chrome](https://support.google.com/chrome/answer/95464?hl=en)
