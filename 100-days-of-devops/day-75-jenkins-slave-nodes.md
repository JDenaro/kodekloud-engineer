# Day 75: Jenkins Slave Nodes

The Nautilus DevOps team has installed and configured new Jenkins server in Stratos DC which they will use for CI/CD and for some automation tasks. There is a requirement to add all app servers as slave nodes in Jenkins so that they can perform tasks on these servers using Jenkins. Find below more details and accomplish the task accordingly.

Click on the `Jenkins` button on the top bar to access the Jenkins UI. Login using username `admin` and password `[redacted]`.

## Specific Requirements:

1. Add all app servers as SSH build agent/slave nodes in Jenkins. Slave node name for `app server 1`, `app server 2` and `app server 3` must be `App_server_1`, `App_server_2`, `App_server_3` respectively.

2. Add labels as below:

   `App_server_1 : stapp01`

   `App_server_2 : stapp02`

   `App_server_3 : stapp03`

3. Remote root directory for `App_server_1` must be `/home/tony/jenkins`, for `App_server_2` must be `/home/steve/jenkins` and for `App_server_3` must be `/home/banner/jenkins`.

4. Make sure slave nodes are online and working properly.

`Note:`

1. You might need to install some plugins and restart Jenkins service. So, we recommend clicking on `Restart Jenkins when installation is complete and no jobs are running` on plugin installation/update page i.e `update centre`. Also, Jenkins UI sometimes gets stuck when Jenkins service restarts in the back end. In this case, please make sure to refresh the UI page.

2. For these kind of scenarios requiring changes to be done in a web UI, please take screenshots so that you can share it for review in case your task is marked incomplete. You may also consider using a screen recording software such as loom.com to record and share your work.

> **Credential note:** The temporary Jenkins password is redacted from this public guide. Use the credentials supplied by the active lab, and do not put server passwords in screenshots or this repository.

## Solution

A Jenkins **controller** schedules work; an **agent** runs that work on another machine. This lab needs one permanent SSH agent per app server. A successful SSH login alone is not enough: each server must also run a Java version that can start Jenkins' `remoting.jar`. On `stapp01`, Java 11 caused the agent launch to fail; installing Java 17 resolved it.

### 🖥️ Step 1: Prepare Each App Server

From the jump host, connect to each server as its designated user. Create the requested remote work directory and check Java:

```bash
ssh tony@stapp01
mkdir -p /home/tony/jenkins
java -version
exit
```

```bash
ssh steve@stapp02
mkdir -p /home/steve/jenkins
java -version
exit
```

```bash
ssh banner@stapp03
mkdir -p /home/banner/jenkins
java -version
exit
```

The captured output for `stapp01` showed Java `11.0.20.1` before the fix. The lab was ultimately successful for all three agents, but no Java-version output was captured for `stapp02` or `stapp03`; inspect each server rather than assuming it has the same version.

> **Why:** `ssh` opens a shell on the target host. `mkdir -p` creates the agent's working directory without failing if it already exists; the login user can write under its own home directory. `java -version` shows the runtime that the SSH launcher will use. Jenkins copies `remoting.jar` to this directory and starts it with Java, so both the writable directory and a compatible Java runtime are necessary. `exit` returns to the jump host.

### 🔌 Step 2: Enable SSH Build Agents in Jenkins

1. Open **Jenkins** from the lab toolbar and sign in as `admin` with the active lab password.
2. In **Manage Jenkins → Plugins**, check whether **SSH Build Agents** is installed. Install it from **Available plugins** if needed.
3. If Jenkins requests a restart, select **Restart Jenkins when installation is complete and no jobs are running**, wait for the login page, and sign in again.

> **Why:** **SSH Build Agents** supplies the **Launch agents via SSH** launch method for nodes. It is different from the **SSH Agent** plugin, which exposes SSH credentials inside jobs, and from **Publish Over SSH**, which runs remote build commands but does not register a Jenkins node.

### 🔐 Step 3: Add SSH Credentials

In **Manage Jenkins → Credentials → System → Global credentials → Add Credentials**, create **Username with password** credentials for the three designated app-server users. Enter the active lab's SSH password for each user in Jenkins, not in a guide or shell command.

| Suggested credential ID | Username | Server |
| --- | --- | --- |
| `stapp01-tony` | `tony` | `stapp01` |
| `stapp02-steve` | `steve` | `stapp02` |
| `stapp03-banner` | `banner` | `stapp03` |

The `App_server_1` launch log confirmed that Jenkins selected credential `stapp01-tony` and authenticated successfully.

> **Why:** The controller must log in to each app server to copy and run its agent process. Separate credentials let each node use the correct account and own its remote directory. The node form's **Credentials** selector appears only after choosing **Launch agents via SSH** under **Launch method**.

### 🧩 Step 4: Create the Three Permanent Agents

Open **Manage Jenkins → Nodes → New Node**. For each server, enter its exact node name, select **Permanent Agent**, and fill in these fields:

| Field | App Server 1 | App Server 2 | App Server 3 |
| --- | --- | --- | --- |
| Node name | `App_server_1` | `App_server_2` | `App_server_3` |
| Remote root directory | `/home/tony/jenkins` | `/home/steve/jenkins` | `/home/banner/jenkins` |
| Labels | `stapp01` | `stapp02` | `stapp03` |
| Host | `stapp01` | `stapp02` | `stapp03` |
| Credentials | `stapp01-tony` | `stapp02-steve` | `stapp03-banner` |

Set **Number of executors** to `1`, **Launch method** to **Launch agents via SSH**, and **Availability** to **Keep this agent online as much as possible**. The default SSH port is `22`. For host-key checking, use **Manually trusted key Verification Strategy** and approve a new key only after confirming it belongs to the intended server. Save each node.

> **Why:** The node name uniquely identifies an agent; its label lets a job request that server. **Remote root directory** is local to the app server, not to Jenkins. One executor permits one job at a time on that node. **Launch agents via SSH** starts the Java-based agent over the SSH connection, while host-key verification helps ensure Jenkins is contacting the intended host. **Keep this agent online** tells Jenkins to reconnect it when needed.

### ☕ Step 5: Fix the Java Version Mismatch on `stapp01`

The first `App_server_1` launch proved that SSH and file transfer were working, but Java could not load the agent:

```text
[SSH] Authentication successful.
[SSH] Copying latest remoting.jar...
java.lang.UnsupportedClassVersionError: hudson/remoting/Launcher has been compiled by a more recent version of the Java Runtime (class file version 61.0), this version of the Java Runtime only recognizes class file versions up to 55.0
Agent JVM has terminated. Exit code=1
```

From the jump host, update Java on `stapp01`:

```bash
ssh tony@stapp01
java -version
sudo yum install -y java-17-openjdk
java -version
exit
```

The first check showed `11.0.20.1`; the final check showed `17.0.20`. Java class-file version **55** corresponds to Java **11**, and version **61** to Java **17**. If another node shows the same error, check that server's Java version and install a compatible runtime there; do not assume all three need a package change. If Java 17 is installed but `java -version` still selects an older version, choose the correct Java binary for that node before launching it again.

> **Why:** `sudo yum install -y java-17-openjdk` installs a Java 17 runtime on the affected Red Hat-family server; `-y` accepts the package-manager prompt. The SSH connection itself was already healthy, so changing credentials, host-key settings, or Jenkins plugins would not fix this particular error. The `java -version` checks establish both the cause and the result. Jenkins agents need a Java version supported by the controller and its remoting library.

### ✅ Step 6: Verify

In **Manage Jenkins → Nodes**, select an offline node and click **Launch Agent**. This was the button label shown in the lab UI. Inspect its **Log** and return to the Nodes list. For `App_server_1`, the successful retry ended with:

```text
[SSH] Authentication successful.
[SSH] Starting agent process: cd "/home/tony/jenkins" && java -jar remoting.jar -workDir /home/tony/jenkins
This is a Unix agent
Agent successfully connected and online
```

Confirm that `App_server_1`, `App_server_2`, and `App_server_3` all show **online**, and that each configuration has its required label and remote root directory. Capture a screenshot of the Nodes page for lab review. The user confirmed that the lab passed; detailed launch logs for the second and third agents were not captured in the conversation.

> **Why:** The final log line demonstrates more than a successful SSH login: Java started `remoting.jar`, Jenkins established its remoting channel, and the node became available to run jobs. The Nodes page verifies the requirement across all three servers. The exact button text can vary by Jenkins version; in this lab it was **Launch Agent**, not **Relaunch Agent**.

## Best Practices

- **Check the agent JVM early.** A successful SSH authentication can still be followed by a Java compatibility failure. Read the node launch log before changing credentials or network settings.
- **Keep identities distinct.** Use `tony`, `steve`, and `banner` for their respective servers, each with a writable remote root under that user's home directory.
- **Verify host keys.** The manually trusted strategy is safer than disabling host-key verification; approve the key only after confirming the server identity.
- **Protect credentials.** Store SSH passwords in Jenkins credentials and omit them from screenshots and documentation. For a production deployment, prefer managed SSH keys and appropriate credential scope.
- **Verify all three nodes.** One online agent does not satisfy a three-node requirement; confirm names, labels, paths, and online state individually.

### 📚 Official Documentation

- [Using Jenkins agents](https://www.jenkins.io/doc/book/using/using-agents/)
- [SSH Build Agents plugin](https://plugins.jenkins.io/ssh-slaves/)
- [Configuring the SSH Build Agents plugin](https://github.com/jenkinsci/ssh-agents-plugin/blob/main/doc/CONFIGURE.md)
- [Jenkins Java support policy](https://www.jenkins.io/doc/book/platform-information/support-policy-java/)
- [Oracle Java class-file versions](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/lang/classfile/ClassFile.html)
- [Red Hat: Installing OpenJDK 17](https://docs.redhat.com/en/documentation/red_hat_build_of_openjdk/17/html/installing_and_using_red_hat_build_of_openjdk_17_on_rhel/installing-openjdk-on-rhel_openjdk)
