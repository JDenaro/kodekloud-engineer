# Day 79: Jenkins Deployment Job

The Nautilus development team had a meeting with the DevOps team where they discussed automating the deployment of one of their apps using Jenkins (the one in Stratos Datacenter). They want to auto deploy the new changes in case any developer pushes to the repository. As per the requirements mentioned below configure the required Jenkins job.

Click on the Jenkins button on the top bar to access the Jenkins UI. Login using username admin and [redacted] password.

Similarly, you can access the Gitea UI using Gitea button. Username and password for Git are sarah and [redacted]. Under user sarah you will find a repository named web that is already cloned on App Server 1 under sarah's home (/home/sarah/web). sarah is a developer who is working on this repository.

## Specific Requirements:

1. httpd is already installed and configured on the app server (listening on port 8080). Ensure the httpd service is running on App Server 1 (e.g. start it manually if needed). You can make starting/restarting httpd part of your Jenkins job if you prefer.

2. Create a Jenkins job named xfusion-app-deployment and configure it so that if anyone pushes any new change to the origin repository in master branch, the job should auto build and deploy the latest code on App Server 1 under /var/www/html directory.
Before deployment, ensure that the ownership of the /var/www/html directory is set to user sarah, so that Jenkins can successfully deploy files to that directory.

3. SSH into App Server 1 using sarah user credentials mentioned above. Under sarah user's home (/home/sarah/web) you will find a cloned Git repository named web. Under this repository there is an index.html file, update its content to Welcome to the xFusionCorp Industries, then push the changes to the origin into master branch. This push must trigger your Jenkins job and the latest changes must be deployed on the server, also make sure it deploys the entire repository content not only index.html file.

Click on the App button on the top bar to access the app. Please make sure the required content is loading on the main URL (e.g. http://stlb01:8091) i.e there should not be any sub-directory like http://stlb01:8091/web etc.

Note:

1. You might need to install some plugins and restart Jenkins service. So, we recommend clicking on Restart Jenkins when installation is complete and no jobs are running on plugin installation/update page i.e update centre. Also some times Jenkins UI gets stuck when Jenkins service restarts in the back end so in such case please make sure to refresh the UI page.
2. Make sure Jenkins job passes even on repetitive runs as validation may try to build the job multiple times.
3. Deployment related tasks should be done by sudo user on the destination server to avoid any permission issues so make sure to configure your Jenkins job accordingly.
4. For these kind of scenarios requiring changes to be done in a web UI, please take screenshots so that you can share it with us for review in case your task is marked incomplete. You may also consider using a screen recording software such as loom.com to record and share your work.

> **Credential note:** Temporary passwords and API tokens are redacted. Use the active lab's credentials; never publish a webhook URL containing a real token.

## Solution

A Gitea push webhook starts a Freestyle job on an SSH agent running as Sarah on App Server 1. Jenkins checks out `master` and exports its committed files into Apache's document root. The successful lab used a webhook, not scheduled SCM polling.

### 🖥️ Step 1: Inspect App Server 1

From the jump host:

```bash
ssh sarah@stapp01
cd /home/sarah/web
git status -sb
git remote -v
java -version
sudo systemctl status httpd
```

Captured results:

```text
stapp01: 10.244.244.164
## master...origin/master
openjdk version "11.0.20.1" 2023-08-24 LTS
Active: active (running)
```

The remote pointed to `http://gitea:3000/sarah/web.git`; its embedded password is omitted here.

> **Why:** `ssh` opens a session as the server's designated user. `cd` selects the existing development clone. `git status -sb` shows the branch and changes in a short format; `git remote -v` shows the fetch and push destinations. `java -version` checks the runtime needed by the Jenkins agent. `sudo systemctl status httpd` inspects Apache with elevated privileges; press `q` to leave its pager. Sarah is a Linux SSH user with sudo access. Jenkins's `admin` account manages Jenkins, but does not automatically provide Linux SSH access.

### 🔧 Step 2: Prepare Permissions and Java

```bash
mkdir -p /home/sarah/jenkins_agent
sudo chown -R sarah:sarah /var/www/html
ls -ld /var/www/html
sudo systemctl start httpd
curl http://localhost:8080/
sudo yum install -y java-17-openjdk
java -version
```

Captured results included:

```text
drwxr-xr-x 1 sarah sarah 4096 Oct  6 23:28 /var/www/html
Welcome to KodeKloud
openjdk version "17.0.20" 2026-07-21 LTS
```

> **Why:** `mkdir -p` creates the separate agent directory and tolerates an existing directory. `chown -R` changes the document root and its contents to Sarah's user and group; `ls -ld` displays the directory itself and its ownership. `systemctl start` ensures Apache is running without restarting an already active service. `curl` retrieves the local site on port 8080. `yum install -y` installs Java 17 and accepts package installation prompts. This lab initially supplied Java 11, and Java 17 was needed for the agent used in these labs. Verify that `java` selects 17 after installation; this is not an instruction to downgrade or change the Jenkins controller's runtime, and other Jenkins releases may require newer Java.

### 🔌 Step 3: Install Plugins and Connect the Agent

Open Jenkins in an **incognito/private tab**, and log in as `admin`. In earlier labs, a normal browser session returned **HTTP 431 Request Header Fields Too Large**; private browsing allowed access without changing Jenkins configuration.

Under **Manage Jenkins → Plugins**, install **Git** and **SSH Build Agents** if absent. Both plugins needed installation in this run. If Jenkins requests a restart, select **Restart Jenkins when installation is complete and no jobs are running**, then wait for the login page to return.

Go to **Manage Jenkins → Credentials → System → Global → Add Credentials**:

| Field | Value |
| --- | --- |
| Kind | Username with password |
| Username | sarah |
| Password | Sarah's current lab password |
| ID | stapp01-sarah |

Under **Manage Jenkins → Nodes → New Node**, create:

| Field | Value |
| --- | --- |
| Node name | App Server 1 |
| Type | Permanent Agent |
| Number of executors | 1 |
| Remote root directory | /home/sarah/jenkins_agent |
| Labels | stapp01 |
| Usage | Only build jobs with label expressions matching this node |
| Launch method | Launch agents via SSH |
| Host | stapp01 |
| Credentials | stapp01-sarah |
| Host Key Verification Strategy | Manually trusted key Verification Strategy |

Save and use **Launch Agent**. Compare any key approval prompt with the SSH fingerprint before trusting it. The captured fingerprint was `SHA256:yEyN8qvzhNxfcKVE+H05zwQPmQMKCXj4JyGWuOP1HIg`.

> **Why:** Git provides the repository checkout, and SSH Build Agents connects Jenkins to the application server. The label routes the deployment to the correct host. A separate remote root keeps agent files and Jenkins workspaces away from the development clone and public website. The agent runs as Sarah; `Running as SYSTEM` in the build log describes Jenkins's internal build identity, not root privileges on the remote server.

### 📦 Step 4: Configure the Deployment Job

Create **xfusion-app-deployment** through **New Item → Freestyle project**.

| Section / Field | Value |
| --- | --- |
| General → Restrict where this project can be run | stapp01 |
| Source Code Management | Git |
| Repository URL | http://gitea:3000/sarah/web.git |
| Credentials | stapp01-sarah |
| Branch Specifier | */master |

In **Build Steps → Execute shell**, enter:

```bash
set -eu
git archive --format=tar --output=deployment.tar HEAD
tar -xf deployment.tar -C /var/www/html
rm deployment.tar
```

Save and run **Build Now** to establish a working deployment. Repeat it to check reliability before relying on automatic execution.

> **Why:** The Git plugin checks out the selected branch in the agent workspace. `set -e` stops the script on a command failure, and `-u` rejects unset variables. `git archive` exports the selected commit (`HEAD`); `--format=tar` chooses an archive format and `--output` writes it to a file. With no path restriction, the committed tree is exported, including tracked hidden files, but not Git's internal `.git` directory. `tar -x` extracts, `-f` selects the archive, and `-C` places its contents directly into Apache's document root without a `web` subdirectory. `rm` removes only the generated workspace archive. Static file updates do not require restarting Apache.

**Permission lessons from the earlier attempt:** Calling `sudo` inside the job failed because it requested a password without an interactive terminal. Prepare ownership manually as in Step 2 instead. Copying the entire checkout with `.git` also failed when overwriting read-only Git objects; exporting the committed tree avoids publishing or overwriting those internal objects. The final successful run used the script above, not the failed copy commands.

### 🔑 Step 5: Enable Remote Builds and Generate an API Token

In **job → Configure → Triggers**, enable **Trigger builds remotely (e.g., from scripts)**. Set **Authentication Token** to your chosen job token; the lab used `day79-deploy`. Save.

For the user API token, go to **Manage Jenkins → Users → admin → Security → API Token → Add new Token**. Name it `gitea-web-deployment`, generate it, and keep its value private.

The user page can also be reached at `/user/admin/security/` under the current Jenkins URL.

> **Why:** The job token identifies this remote trigger, while the user's API token authenticates the HTTP request to Jenkins. The token's name is only a label; the generated value is the credential. This is the **admin user's Security page**, not **Manage Jenkins → Security**, which controls global settings. Do not enable legacy API tokens or use **Git plugin notifyCommit access tokens**: neither is the user API token needed here.

### 🔔 Step 6: Configure Gitea's Push Webhook

Log in to Gitea as Sarah. Open **sarah/web → Settings → Webhooks → Add Webhook → Gitea**.

Use this template for **Target URL**, replacing the uppercase placeholders privately:

```text
https://admin:TOKEN_API@HOST_JENKINS/job/xfusion-app-deployment/build?token=day79-deploy
```

The captured Jenkins host was `8080-port-aqskga3kuja5nkvw.labs.kodekloud.com`. `HOST_JENKINS` is the domain only: do not insert another `https://` after `@`, and use one slash before `job`.

| Field | Value |
| --- | --- |
| HTTP Method | POST |
| POST Content Type | application/json |
| Branch filter | master |
| Trigger On | Push Events |
| Active | Enabled |

Save with **Add Webhook**, or **Update Webhook** when editing. This lab's UI showed **Recent Deliveries** but no **Test Delivery** button; the actual push in the next step tested the integration.

> **Why:** A webhook is an HTTP notification sent by Gitea when a selected repository event occurs. The branch filter restricts notifications to `master`. Jenkins's `/job/xfusion-app-deployment/build` endpoint accepts a POST that queues a build. JSON is the selected webhook payload format. The URL credentials authenticate as `admin` using the API token; the query token matches the job's remote trigger configuration. This lab used credentials in the URL, so treat the entire URL as a secret and revoke any token accidentally shared. Prefer header-based authentication with a least-privilege service identity for production rather than putting an administrator credential in a URL.

### 🚀 Step 7: Update the Development Clone and Push

As Sarah on App Server 1:

```bash
cd /home/sarah/web
vi index.html
```

Replace its content with:

```text
Welcome to the xFusionCorp Industries
```

Save and run:

```bash
git diff
git add index.html
git commit -m "Update welcome message"
git push origin master
```

Captured results:

```text
[master 9538b3c] Update welcome message
1 file changed, 1 insertion(+), 1 deletion(-)
0531d98..9538b3c  master -> master
```

> **Why:** `vi` edits the developer's source file. `git diff` reviews the change before publishing; `git add` stages the file and `git commit -m` records it with a descriptive message. `git push origin master` publishes the commit to the original repository's master branch. Make this change only after configuring the webhook, and do not click **Build Now** afterward: the push itself must start the deployment.

### ✅ Step 8: Verify

In Gitea, open the webhook's **Recent Deliveries**. The captured response was **HTTP 201**, with a `Location` ending in `/queue/item/3/`.

In Jenkins, the resulting build contained:

```text
Started by remote host 65.108.255.62
Building remotely on App Server 1
(stapp01) in workspace /home/sarah/jenkins_agent/workspace/xfusion-app-deployment
Checking out Revision 9538b3c77284ee4e56a848fe5ebaa93e3d2c80b4 (refs/remotes/origin/master)
Commit message: "Update welcome message"
Finished: SUCCESS
```

From App Server 1:

```bash
curl http://localhost:8080/
```

Captured response:

```text
Welcome to the xFusionCorp Industries
```

Open **App** and check the same content at the main URL, without `/web`. The user subsequently confirmed that the lab passed.

> **Why:** HTTP 201 and the queue location show that Jenkins accepted the webhook request, not that deployment had already finished. The remotely started build, matching commit ID, successful result, and HTTP content together prove the complete push-to-deployment path. A webhook-triggered build can say `Started by remote host`; it does not need to say `Started by an SCM change`, which was the expectation for the earlier polling approach.

## Best Practices

- **Keep Git metadata private.** Export the committed application tree instead of copying `.git` into a public document root. Check for `export-ignore` attributes or submodules before reusing this approach in a different project.
- **Separate runtime identities.** Sarah owns the target and runs the agent; Jenkins admin controls the UI. Do not assume that an admin UI account gives remote root access.
- **Limit elevated privileges.** Prepare ownership interactively with sudo rather than embedding a password or granting unrestricted passwordless sudo to make a job pass.
- **Verify the event, not just a manual build.** Confirm webhook delivery, automatic build completion, deployed commit, and public content before finishing the lab.
- **Protect tokens.** Redact webhook credentials from logs and screenshots and revoke exposed tokens. Administrator tokens in URLs are a temporary lab shortcut, not a production credential-management pattern.
- **Understand repeatability limits.** Re-extracting the archive supports repeated builds, but does not remove files deleted from newer commits. Production deployments should use a reviewed synchronization or release-directory strategy rather than broad deletion commands.

### 📚 Official Documentation

- [Jenkins: Using agents](https://www.jenkins.io/doc/book/using/using-agents/)
- [Jenkins: Remote Access API](https://www.jenkins.io/doc/book/using/remote-access-api/)
- [Gitea: Webhooks](https://docs.gitea.com/usage/repository/webhooks/)
- [Git: git archive](https://git-scm.com/docs/git-archive)
