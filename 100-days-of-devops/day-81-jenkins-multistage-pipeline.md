# Day 81: Jenkins Multistage Pipeline

The development team of xFusionCorp Industries is working on to develop a new static website and they are planning to deploy the same on Nautilus App Server using Jenkins pipeline. They have shared their requirements with the DevOps team and accordingly we need to create a Jenkins pipeline job. Please find below more details about the task:

Click on the Jenkins button on the top bar to access the Jenkins UI. Login using username admin and password [redacted].

Similarly, click on the Gitea button on the top bar to access the Gitea UI. Login using username sarah and password [redacted].

## Specific Requirements:

1. There is a repository named sarah/web in Gitea that is already cloned on App Server 1 under /var/www/html directory.

2. Update the content of the file index.html under the same repository to Welcome to xFusionCorp Industries and push the changes to the origin into the master branch.

3. Apache is already installed on the app server and is running on port 8080.

4. Add App Server 1 as a Jenkins agent (slave) node: name App Server 1, label stapp01, remote root directory /home/sarah/jenkins_agent, launch via SSH with host stapp01 and credentials for user sarah. Install java-17-openjdk on App Server 1 if needed.

5. Create a Jenkins pipeline job named deploy-job (it must not be a Multibranch pipeline job) and pipeline should have two stages Deploy and Test ( names are case sensitive ). Configure these stages as per details mentioned below.

   a. The Deploy stage should deploy the code from web repository under /var/www/html on App Server 1, as this is the document root of the app server.

   b. The pipeline should run on the App Server 1 node (e.g. use label stapp01).

   c. The Test stage should just test if the app is working fine and website is accessible. Its up to you how you design this stage to test it out, you can simply add a curl command as well to run a curl against the LBR URL (http://stlb01:8091) to see if the website is working or not. Make sure this stage fails in case the website/app is not working or if the Deploy stage fails.

Click on the App button on the top bar to see the latest changes you deployed. Please make sure the required content is loading on the main URL http://stlb01:8091 i.e there should not be a sub-directory like http://stlb01:8091/web etc.

Note:

You might need to install some plugins and restart Jenkins service. So, we recommend clicking on Restart Jenkins when installation is complete and no jobs are running on plugin installation/update page i.e update centre. Also, Jenkins UI sometimes gets stuck when Jenkins service restarts in the back end. In this case, please make sure to refresh the UI page.

For these kind of scenarios requiring changes to be done in a web UI, please take screenshots so that you can share it with us for review in case your task is marked incomplete. You may also consider using a screen recording software such as loom.com to record and share your work.

> **Credential note:** Only temporary passwords have been redacted. Use the active lab's credentials.

## Solution

Using Jenkins **2.541.2** as the user-confirmed reference, run a Declarative Pipeline on the SSH agent. `Deploy` updates the existing document-root clone; `Test` requests the main load-balancer URL and fails on connection or HTTP errors. If deployment fails, the Pipeline fails and skips the later normal stage rather than reporting a successful test.

### 🖥️ Step 1: Inspect and Prepare App Server 1

From the jump host:

```bash
ssh sarah@stapp01
cd /var/www/html
git status -sb
git remote -v
ls -ld /var/www/html
java -version
sudo systemctl status httpd
mkdir -p /home/sarah/jenkins_agent
sudo yum install -y java-17-openjdk
java -version
```

The captured server IP was `10.244.164.39`. Results included:

```text
## master...origin/master
drwxr-xr-x 1 sarah sarah 4096 Oct 12 21:27 /var/www/html
openjdk version "11.0.20.1" 2023-08-24 LTS
Active: active (running)
openjdk version "17.0.20" 2026-07-21 LTS
```

The remote was `http://gitea:3000/sarah/web.git`; its embedded password is omitted.

> **Why:** `ssh` connects as Sarah, the Linux user who owns the clone. `cd` selects that repository. `git status -sb` shows the branch and changes concisely; `git remote -v` shows fetch and push destinations. `ls -ld` inspects the directory's ownership. `java -version` revealed Java 11; `yum install -y` installed Java 17 with automatic package confirmation, and the second version check confirmed it was selected. Java 17 is the agent prerequisite used in this lab, not a request to change the controller's runtime. `sudo systemctl status` checks Apache; leave its pager with `q`. `mkdir -p` creates the independent agent directory and tolerates an existing directory. No Apache restart or ownership change was needed.

### 📝 Step 2: Publish the Required Content

```bash
cd /var/www/html
vi index.html
```

Set the file to exactly:

```text
Welcome to xFusionCorp Industries
```

```bash
git diff
git add index.html
git commit -m "Update welcome message"
git push origin master
```

Captured results:

```text
[master 78acf45] Update welcome message
1 file changed, 1 insertion(+), 1 deletion(-)
6f4e60a..78acf45  master -> master
```

> **Why:** `vi` edits the source, `git diff` reviews the change, `git add` stages it, and `git commit -m` records it with a message. `git push origin master` publishes the master branch to Gitea. The required text has no “the” before xFusionCorp. Because this lab's clone is already the document root, the manual edit is immediately local; the Pipeline still performs the required pull from origin.

### 🔌 Step 3: Configure the SSH Agent

Open Jenkins in a private/incognito tab; this avoided HTTP 431 from the regular browser session in earlier labs. Ensure **SSH Build Agents** and **Pipeline** are installed under **Manage Jenkins → Plugins**. If installation needs a restart, wait for the login page to return.

At **Manage Jenkins → Credentials → System → Global → Add Credentials**, create **Username with password** credentials for Sarah using the active lab password and ID `stapp01-sarah`.

Under **Manage Jenkins → Nodes → New Node**, configure:

| Field | Value |
| --- | --- |
| Node name | App Server 1 |
| Type | Permanent Agent |
| Number of executors | 1 |
| Remote root directory | /home/sarah/jenkins_agent |
| Labels | stapp01 |
| Launch method | Launch agents via SSH |
| Host | stapp01 |
| Credentials | stapp01-sarah |
| Host Key Verification Strategy | Manually trusted key Verification Strategy |

Save and click **Launch Agent**. Verify any host-key approval against the SSH connection's fingerprint before trusting it. Wait until the node is online.

> **Why:** SSH Build Agents starts the remote agent as Sarah; Pipeline provides the job type and execution steps. The label selects the server, while the separate remote root keeps Jenkins workspaces outside the public site. Jenkins admin credentials manage Jenkins; Sarah's credentials provide Linux SSH access.

### 🚀 Step 4: Create the Two-Stage Pipeline

Choose **New Item → deploy-job → Pipeline**, not Multibranch Pipeline. Under **Pipeline → Definition → Pipeline script**, paste:

```groovy
pipeline {
    agent {
        label 'stapp01'
    }

    options {
        disableConcurrentBuilds()
        skipDefaultCheckout()
    }

    stages {
        stage('Deploy') {
            steps {
                sh '''
                    cd /var/www/html
                    git pull --ff-only origin master
                '''
            }
        }

        stage('Test') {
            steps {
                sh '''
                    curl --fail --show-error --silent --location --max-time 30 http://stlb01:8091/
                '''
            }
        }
    }
}
```

> **Why:** `agent` selects `stapp01` for both stages. `disableConcurrentBuilds()` prevents simultaneous changes to the shared clone; `skipDefaultCheckout()` avoids an additional automatic checkout in the workspace. `sh` runs a shell command and fails on a nonzero exit. `git pull --ff-only origin master` fetches and updates from master without creating an unexpected merge. Shell `cd`, rather than Jenkins `dir('/var/www/html')`, keeps Jenkins temporary files in the agent workspace. `curl --fail` rejects HTTP errors such as 404 and 500; `--silent` hides progress, `--show-error` preserves error messages, `--location` follows redirects, and `--max-time 30` bounds the request. Connection errors also fail the stage. Sequential stages stop normal execution after deployment failure, so Test is skipped and the overall build remains failed.

### ✅ Step 5: Verify

Save and select **Build Now**. The captured Console Output included:

```text
Running on App Server 1
in /home/sarah/jenkins_agent/workspace/deploy-job
[Pipeline] { (Deploy)
+ cd /var/www/html
+ git pull --ff-only origin master
Already up to date.
[Pipeline] { (Test)
+ curl --fail --show-error --silent --location --max-time 30 http://stlb01:8091/
Welcome to xFusionCorp Industries
Finished: SUCCESS
```

Open **App** and verify the content on the main URL without `/web`. The user confirmed the lab was successful.

> **Why:** The node and workspace prove remote execution. “Already up to date” is expected because the developer already pushed from this clone. The Test response proves access through the load balancer, not just localhost. This test checks HTTP availability; it prints but does not automatically assert the exact response text. A 200 response containing incorrect content would require a separate content assertion in a stricter production test.

## Best Practices

- **Preserve stage names.** Use exactly `Deploy` and `Test` as required by the validator.
- **Propagate failures.** Do not suppress shell errors or use `returnStatus` without checking it; deployment and HTTP failures must fail the Pipeline.
- **Separate agent files from website files.** Keep the remote root under Sarah's home and operate on the site with shell `cd`.
- **Avoid unnecessary privileged commands.** Sarah already owns this clone and Apache is running; neither passwordless sudo nor service changes are needed in the Pipeline.
- **Bound and serialize work.** Limit HTTP wait time and prevent concurrent updates of the same repository.
- **Check runtime compatibility.** Java 11 was replaced by Java 17 in this lab; consult the relevant controller and agent requirements before applying that choice to other Jenkins versions.

### 📚 Official Documentation

- [Jenkins: Pipeline syntax](https://www.jenkins.io/doc/book/pipeline/syntax/)
- [Jenkins: Using agents](https://www.jenkins.io/doc/book/using/using-agents/)
- [Git: git pull](https://git-scm.com/docs/git-pull)
- [curl: Command-line manual](https://curl.se/docs/manpage.html)
