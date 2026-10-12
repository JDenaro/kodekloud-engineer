# Day 78: Jenkins Conditional Pipeline

The development team of xFusionCorp Industries is working on to develop a new static website and they are planning to deploy the same on Nautilus App Server using Jenkins pipeline. They have shared their requirements with the DevOps team and accordingly we need to create a Jenkins pipeline job. Please find below more details about the task:

Click on the Jenkins button on the top bar to access the Jenkins UI. Login using username admin and password [redacted].

Similarly, click on the Gitea button on the top bar to access the Gitea UI. Login using username sarah and password [redacted]. There under user sarah you will find a repository named web_app that is already cloned on App Server 1 under /var/www/html. sarah is a developer who is working on this repository.

## Specific Requirements:

1. Add a slave node named App Server 1. It should be labeled as stapp01 and its remote root directory should be /home/sarah/jenkins_agent (the repository is cloned under /var/www/html).

2. We have already cloned repository on App Server 1 under /var/www/html.

3. Apache is already installed on the app server and is running on port 8080.

4. Create a Jenkins pipeline job named devops-webapp-job (it must not be a Multibranch pipeline) and configure it to:

   Add a string parameter named BRANCH.

   It should conditionally deploy the code from web_app repository under /var/www/html on App Server 1, as this is the document root of the app server. The pipeline should have a single stage named Deploy ( which is case sensitive ) to accomplish the deployment.

   The pipeline should be conditional, if the value master is passed to the BRANCH parameter then it must deploy the master branch, on the other hand if the value feature is passed to the BRANCH parameter then it must deploy the feature branch.

LB server is already configured. You should be able to see the latest changes you made by clicking on the App button. Please make sure the required content is loading on the main URL https://<LBR-URL> i.e there should not be a sub-directory like https://<LBR-URL>/web_app etc.

Note:

You might need to install some plugins and restart Jenkins service. So, we recommend clicking on Restart Jenkins when installation is complete and no jobs are running on plugin installation/update page i.e update centre. Also, Jenkins UI sometimes gets stuck when Jenkins service restarts in the back end. In this case, please make sure to refresh the UI page.

For these kind of scenarios requiring changes to be done in a web UI, please take screenshots so that you can share it with us for review in case your task is marked incomplete. You may also consider using a screen recording software such as loom.com to record and share your work.

> **Credential note:** Only temporary passwords have been redacted. Use the active lab's credentials.

## Solution

> **Java prerequisite for new agents:** In the captured App Server labs, the initial runtime was Java 11 (`11.0.20.1`). Installing Java 17 was necessary to connect a new Jenkins agent with the lab's remoting version. Before launching a new node, install `java-17-openjdk` on its App Server and confirm `java -version` selects Java 17. This is an agent prerequisite, not an instruction to change the Jenkins controller's runtime or reinstall Java for UI-only tasks. Check each target server; the captured Java 11 evidence is from `stapp01`, not a version measurement for every server. Other Jenkins versions may require newer Java.

A validated String parameter selects the branch to deploy into Apache's existing document root. Both branches use a single stage named `Deploy`. Java was upgraded from 11 to 17, and `withEnv` explicitly passed the Jenkins parameter to the shell after an empty-value failure during the first build.

### 🖥️ Step 1: Inspect the Clone and Prepare the Agent Directory

From the jump host:

```bash
ssh sarah@stapp01
cd /var/www/html
git status -sb
git branch -a
java -version
mkdir -p /home/sarah/jenkins_agent
```

The repository was clean and both branches existed:

```text
## feature
* feature
  master
  remotes/origin/feature
  remotes/origin/master
```

The initial Java version was `11.0.20.1`.

> **Why:** `ssh` connects as the agent account. `cd` selects the website clone. `git status -sb` combines short status and branch information, while `git branch -a` lists local and remote-tracking branches. `java -version` checks the runtime. `mkdir -p` creates the separate agent directory and accepts an already-existing directory.

### ☕ Step 2: Install Java 17

```bash
sudo yum install -y java-17-openjdk
java -version
```

The user joined these commands with `&&`; they are separated here for readability. The installation selected Java 17 automatically:

```text
openjdk version "17.0.20" 2026-07-21 LTS
OpenJDK Runtime Environment (Red_Hat-17.0.20.0.8-1) (build 17.0.20+8-LTS)
OpenJDK 64-Bit Server VM (Red_Hat-17.0.20.0.8-1) (build 17.0.20+8-LTS, mixed mode, sharing)
```

> **Why:** The Jenkins agent runs a Java process. `sudo` permits system installation, `yum install` installs the runtime and dependencies, and `-y` accepts the transaction prompt. The version check confirms the selected runtime. Java 17 worked for this lab; newer Jenkins versions may require newer Java.

### 🔌 Step 3: Configure the SSH Agent

Open Jenkins with the active lab credentials. Use a private/incognito window if the lab UI returns **HTTP ERROR 431 Request Header Fields Too Large**, as encountered in earlier labs.

If missing, install **SSH Build Agents** and **Pipeline** through **Manage Jenkins → Plugins → Available plugins**. If restarting, select **Restart Jenkins when installation is complete and no jobs are running**, then wait for the login page. Only seeing **Freestyle project** under **New Item** indicates missing Pipeline components.

Open **Manage Jenkins → Credentials → System → Global → Add Credentials**:

| Field | Value |
| --- | --- |
| Kind | Username with password |
| Username | `sarah` |
| Password | Sarah's SSH password from the active lab |
| ID | `stapp01-sarah` |

Under **Manage Jenkins → Nodes → New Node**, create a **Permanent Agent** named `App Server 1`:

| Field | Value |
| --- | --- |
| Number of executors | `1` |
| Remote root directory | `/home/sarah/jenkins_agent` |
| Labels | `stapp01` |
| Launch method | Launch agents via SSH |
| Host | `stapp01` |
| Credentials | `stapp01-sarah` |
| Host Key Verification Strategy | Manually trusted key Verification Strategy |

Save, verify the server key when prompted, and click **Launch Agent** if necessary. Confirm the agent is online.

> **Why:** The plugins provide SSH launching and Pipeline jobs. The credential authenticates the agent connection, not Git. The label selects App Server 1, and the remote root keeps Jenkins files outside the website. One executor limits parallel work on this agent; host-key verification checks the server's identity.

### 🚀 Step 4: Create the Conditional Pipeline

Create `devops-webapp-job` using **New Item → Pipeline**, not Multibranch Pipeline. Select **Pipeline → Definition → Pipeline script** and paste the complete script:

```groovy
pipeline {
    agent {
        label 'stapp01'
    }

    options {
        disableConcurrentBuilds()
    }

    parameters {
        string(
            name: 'BRANCH',
            defaultValue: 'master',
            description: 'Branch to deploy: master or feature'
        )
    }

    stages {
        stage('Deploy') {
            steps {
                script {
                    if (params.BRANCH != 'master' && params.BRANCH != 'feature') {
                        error('BRANCH must be master or feature')
                    }

                    withEnv(["BRANCH=${params.BRANCH}"]) {
                        sh '''
                            cd /var/www/html
                            git fetch origin
                            git switch "$BRANCH"
                            git pull --ff-only origin "$BRANCH"
                        '''
                    }
                }
            }
        }
    }
}
```

> **Why:** `agent` selects the server label. `disableConcurrentBuilds()` serializes this job's builds. The String parameter defaults to `master`; `script` permits Groovy validation and `error` rejects other values. `withEnv` explicitly passes the parameter to the shell. `sh` runs the multiline commands, and the shell expands the quoted `"$BRANCH"`. `git fetch origin` refreshes remote references, `git switch` selects the branch, and `git pull --ff-only origin` integrates its remote changes only when a fast-forward is possible. Exactly one stage is declared.

Keep `cd` inside `sh`. In Day 77, wrapping the shell in `dir('/var/www/html')` caused an `/var/www/html@tmp` permission error. This script keeps Jenkins' control files in its own workspace without broadening directory permissions.

#### First-Build Parameter Registration

Save the job. If **Build with Parameters** is absent, run **Build Now** once to process the parameter definition, then refresh the job page.

The initial script passed parameter validation, but the shell received an empty value:

```text
+ git switch ''
fatal: invalid reference:
```

The final script explicitly transfers `params.BRANCH` through `withEnv`. Jenkins parameter access and shell environment access are different contexts; this avoids relying on automatic propagation during the first execution. Paste the complete script when updating it so its closing braces remain balanced.

### ✅ Step 5: Verify

Run **Build with Parameters** first with `BRANCH=master`, then with `BRANCH=feature`.

The successful master build included:

```text
Running on App Server 1
 in /home/sarah/jenkins_agent/workspace/devops-webapp-job
[Pipeline] { (Deploy)
+ cd /var/www/html
+ git fetch origin
+ git switch master
Switched to branch 'master'
Your branch is up to date with 'origin/master'.
+ git pull --ff-only origin master
From http://gitea:3000/sarah/web_app
 * branch            master     -> FETCH_HEAD
Already up to date.
Finished: SUCCESS
```

The successful feature build included:

```text
[Pipeline] { (Deploy)
+ cd /var/www/html
+ git fetch origin
+ git switch feature
Switched to branch 'feature'
+ git pull --ff-only origin feature
From http://gitea:3000/sarah/web_app
 * branch            feature    -> FETCH_HEAD
Already up to date.
Finished: SUCCESS
```

> **Why:** Both logs show the parameter selecting the correct branch on App Server 1. `Already up to date.` means no additional remote commits needed downloading; switching branches still selects that branch's files. The final deployment uses `feature`, the last successful execution.

Open **App** at the main load-balancer URL, without appending `/web_app`. Capture the agent settings, Pipeline configuration, parameters, and successful output for review. The user confirmed that the lab passed.

## Best Practices

- **Validate inputs.** Accept only the required branches before changing the deployed working tree.
- **Pass parameters explicitly.** Use `withEnv` to connect Jenkins parameters and shell variables clearly.
- **Serialize deployments.** Parallel builds must not switch the same website clone at the same time.
- **Preserve local changes.** Investigate dirty or diverged history rather than forcing a checkout or destructive reset.
- **Separate agent and website files.** Keep Jenkins' root under Sarah's home and website content in Apache's document root.
- **Check Java compatibility.** Java 17 worked here, but runtime requirements depend on the Jenkins version.
- **Protect credentials.** Use managed credentials and avoid printing passwords or credential-bearing Git URLs.

### 📚 Official Documentation

- [Declarative Pipeline syntax](https://www.jenkins.io/doc/book/pipeline/syntax/)
- [Pipeline Basic Steps: withEnv and error](https://www.jenkins.io/doc/pipeline/steps/workflow-basic-steps/)
- [Jenkins Java support policy](https://www.jenkins.io/doc/book/platform-information/support-policy-java/)
- [Git switch](https://git-scm.com/docs/git-switch)
- [Git pull](https://git-scm.com/docs/git-pull)
