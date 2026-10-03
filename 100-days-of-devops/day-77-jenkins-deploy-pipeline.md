# Day 77: Jenkins Deploy Pipeline

The development team of xFusionCorp Industries is working on to develop a new static website and they are planning to deploy the same on Nautilus App Server using Jenkins pipeline. They have shared their requirements with the DevOps team and accordingly we need to create a Jenkins pipeline job. Please find below more details about the task:

Click on the `Jenkins` button on the top bar to access the Jenkins UI. Login using username `admin` and password `[redacted]`.

Similarly, click on the `Gitea` button on the top bar to access the Gitea UI. Login using username `sarah` and password `[redacted]`. There under user `sarah` you will find a repository named `web_app` that is already cloned on **App Server 1** under `/var/www/html`. sarah is a developer who is working on this repository.

## Specific Requirements:

1. Add a slave node named `App Server 1`. It should be labeled as `stapp01` and its remote root directory should be `/home/sarah/jenkins_agent` (the repository is cloned under `/var/www/html`; the agent uses a separate directory so it does not pollute the repo).

2. We have already cloned repository on **App Server 1** under `/var/www/html`.

3. Apache is already installed on the app server and is running on port `8080`.

4. Create a Jenkins pipeline job named `datacenter-webapp-job` (it must not be a `Multibranch pipeline`) and configure it to:

   - Deploy the code from `web_app` repository under `/var/www/html` on **App Server 1**, as this is the document root of the app server. The pipeline should have a single stage named `Deploy` ( which is case sensitive ) to accomplish the deployment.

LB server is already configured. You should be able to see the latest changes you made by clicking on the `App` button. Please make sure the required content is loading on the main URL `https://<LBR-URL>` i.e there should not be a sub-directory like `https://<LBR-URL>/web_app` etc.

`Note:`

1. You might need to install some plugins and restart Jenkins service. So, we recommend clicking on `Restart Jenkins when installation is complete and no jobs are running` on plugin installation/update page i.e `update centre`. Also, Jenkins UI sometimes gets stuck when Jenkins service restarts in the back end. In this case, please make sure to refresh the UI page.

2. For these kind of scenarios requiring changes to be done in a web UI, please take screenshots so that you can share it with us for review in case your task is marked incomplete. You may also consider using a screen recording software such as loom.com to record and share your work.

> **Credential note:** Passwords from the challenge and the captured Git remote URL are redacted in this public guide. Use the credentials supplied by the active lab.

## Solution

The Jenkins controller schedules the deployment on an SSH agent running as `sarah` on `stapp01`. The agent has its own workspace under `/home/sarah/jenkins_agent`, while the website remains in `/var/www/html`, Apache's document root. Updating the existing clone publishes the static files at the main application URL.

Three details mattered in this run: Java 11 needed upgrading to Java 17, the Pipeline plugin was missing, and a Jenkins `dir` block caused a temporary-directory permission error. The successful pipeline uses `cd` inside `sh` to update the repository while keeping Jenkins' temporary files in the agent workspace.

### 🖥️ Step 1: Inspect the Existing Clone and Prepare the Agent Directory

From the jump host:

```bash
ssh sarah@stapp01
cd /var/www/html
git status -sb
git remote -v
java -version
mkdir -p /home/sarah/jenkins_agent
```

The repository status was:

```text
## master...origin/master
```

No changed files followed that line. The existing `origin` pointed to `http://gitea:3000/sarah/web_app.git`, with temporary credentials embedded in the actual lab URL. The initial Java version was `11.0.20.1`.

> **Why:** `ssh` opens a session as the account Jenkins will use. `cd` selects the existing repository. `git status -sb` combines short status with branch information, confirming a clean working tree and an upstream branch. `git remote -v` shows the fetch and push URLs. `java -version` identifies the agent runtime. `mkdir -p` creates the requested remote root directory and accepts an already-existing directory.

### ☕ Step 2: Install Java 17

In the same `sarah` session:

```bash
sudo yum install -y java-17-openjdk
java -version
```

The successful installation selected Java 17 automatically:

```text
openjdk version "17.0.20" 2026-07-21 LTS
OpenJDK Runtime Environment (Red_Hat-17.0.20.0.8-1) (build 17.0.20+8-LTS)
OpenJDK 64-Bit Server VM (Red_Hat-17.0.20.0.8-1) (build 17.0.20+8-LTS, mixed mode, sharing)
```

No manual alternatives selection was needed in this run.

> **Why:** Jenkins agents run a Java process, so SSH access alone is insufficient. `sudo` grants the privileges needed for package installation; `yum install` installs the requested runtime and its dependencies, and `-y` accepts the package prompt. The second version check confirms the runtime that the launcher will find. Java 17 worked for this lab; check Jenkins' Java support policy before choosing a runtime for another controller version.

### 🔌 Step 3: Install the Jenkins Plugins

Sign in to Jenkins with the active lab's `admin` credentials. Use a private/incognito browser window if the lab UI returns **HTTP ERROR 431 Request Header Fields Too Large**, as encountered in earlier Jenkins labs.

In **Manage Jenkins → Plugins → Available plugins**, install the required plugins if missing:

| Plugin | Purpose |
| --- | --- |
| **SSH Build Agents** (`ssh-slaves`) | Adds the SSH launch method for permanent agents. |
| **Pipeline** (`workflow-aggregator`) | Adds Pipeline jobs and their supporting components. |

In this run, **New Item** initially offered only **Freestyle project**. Installing **Pipeline** made the required job type available. Select **Restart Jenkins when installation is complete and no jobs are running** when necessary, then wait for the login page and sign in again.

> **Why:** Jenkins capabilities are supplied by plugins. The SSH launcher connects the controller to the app server; Pipeline supplies the job type needed to execute the deployment script. A Freestyle project would not meet the challenge's Pipeline requirement.

### 🔐 Step 4: Add the SSH Credential

The path shown in the lab UI was **Manage Jenkins → Credentials → System → Global → Add Credentials**.

Select **Username with password** and enter:

| Field | Value |
| --- | --- |
| Username | `sarah` |
| Password | The active lab's SSH password for `sarah` |
| ID | `stapp01-sarah` |
| Description | `SSH access to App Server 1` |

Save the credential. **SSH Username with private key** and **Certificate** were also available, but the lab used password authentication.

> **Why:** Jenkins needs an SSH identity to start the remote agent. The credential ID lets the node configuration select that identity. This credential authenticates the agent connection; the existing Git clone has its own remote authentication configuration. Do not assume an SSH password and a Gitea password are interchangeable.

### 🧩 Step 5: Create the Permanent Agent

Open **Manage Jenkins → Nodes → New Node**. Enter `App Server 1`, choose **Permanent Agent**, and configure:

| Field | Value |
| --- | --- |
| Number of executors | `1` |
| Remote root directory | `/home/sarah/jenkins_agent` |
| Labels | `stapp01` |
| Launch method | **Launch agents via SSH** |
| Host | `stapp01` |
| Credentials | `stapp01-sarah` |
| Host Key Verification Strategy | **Manually trusted key Verification Strategy** |

Save the node, verify the server key when prompted, and click **Launch Agent** if needed. Confirm that the agent is online before building the job.

> **Why:** The node name identifies the agent and its label lets the pipeline select App Server 1. One executor allows one job at a time. The remote root stores Jenkins' agent files and workspaces outside the website clone. Host-key verification checks the identity of the SSH server.

### 🚀 Step 6: Create the Deploy Pipeline

Select **New Item**, name the job `datacenter-webapp-job`, and choose **Pipeline**. Under **Pipeline → Definition**, select **Pipeline script** and enter:

```groovy
pipeline {
    agent {
        label 'stapp01'
    }

    stages {
        stage('Deploy') {
            steps {
                sh '''
                    cd /var/www/html
                    git pull --ff-only
                '''
            }
        }
    }
}
```

Save the job. The clone already tracks `origin/master`, so no branch name or additional checkout is needed in this script.

> **Why:** `pipeline` defines a Declarative Pipeline. `agent` selects an executor using the `stapp01` label. `stages` contains the single required stage, `Deploy`, and `steps` lists its actions. `sh` executes the multiline shell script. `cd` selects Apache's document root; `git pull` fetches and integrates changes from the configured upstream. `--ff-only` accepts a fast-forward update and fails if the histories diverge instead of creating a merge commit. Because the repository is already cloned at the document root, the site stays at the main URL.

The initial approach wrapped `sh` in `dir('/var/www/html')`. It failed before Git ran:

```text
java.nio.file.AccessDeniedException: /var/www/html@tmp
```

Jenkins tried to create its shell-control directory beside the selected directory. Creating `/var/www/html@tmp` requires write access to `/var/www`, which the agent account did not have. The successful script above leaves Jenkins' working directory under the agent root and changes directory only inside the shell process. No broader permissions or ownership changes were needed.

### ✅ Step 7: Verify

Click **Build Now**, then open **Console Output**. The successful run included:

```text
Running on App Server 1 in /home/sarah/jenkins_agent/workspace/datacenter-webapp-job
[Pipeline] stage
[Pipeline] { (Deploy)
[Pipeline] sh
+ cd /var/www/html
+ git pull --ff-only
Already up to date.
[Pipeline] End of Pipeline
Finished: SUCCESS
```

`Already up to date.` means the clone already contained the latest upstream commit; the deployment command still ran successfully. The user confirmed that **App** loaded the website and that the lab passed. Open the main application URL without adding `/web_app`. Capture the node configuration, pipeline configuration, and successful build for review.

> **Why:** The console confirms the selected server, the agent workspace, the required stage, and successful Git execution. Checking the website verifies the user-visible result through the existing load balancer; a successful job alone does not prove the site is reachable.

## Best Practices

- **Separate workspace and website files.** Keep Jenkins' remote root under the agent user's home and the static content under Apache's document root.
- **Diagnose the failing path.** This permission error affected `/var/www/html@tmp`, not Git authentication or Java. Moving the shell's directory change into the script resolved it without broadening filesystem access.
- **Keep deployment history predictable.** A clean clone and `--ff-only` avoid unexpected deployment merge commits. Investigate diverged history rather than forcing a reset.
- **Protect credentials.** The lab's Git URL embedded a temporary password. For production, use managed credentials and avoid printing credential-bearing remotes in build logs or screenshots.
- **Match the agent runtime to Jenkins.** Java 17 worked here; supported Java versions depend on the controller and remoting versions.
- **Treat shared deployment paths carefully.** One agent executor limits jobs on that agent, but other users or agents could still modify the clone. Coordinate deployments to a shared document root in production.

### 📚 Official Documentation

- [Using Jenkins agents](https://www.jenkins.io/doc/book/using/using-agents/)
- [SSH Build Agents plugin](https://plugins.jenkins.io/ssh-slaves/)
- [Pipeline plugin](https://plugins.jenkins.io/workflow-aggregator/)
- [Using Jenkins credentials](https://www.jenkins.io/doc/book/using/using-credentials/)
- [Declarative Pipeline syntax](https://www.jenkins.io/doc/book/pipeline/syntax/)
- [Pipeline steps: dir, sh, node, and workspace](https://www.jenkins.io/doc/pipeline/steps/workflow-durable-task-step/)
- [Jenkins Java support policy](https://www.jenkins.io/doc/book/platform-information/support-policy-java/)
- [Git pull and fast-forward behavior](https://git-scm.com/docs/git-pull)
