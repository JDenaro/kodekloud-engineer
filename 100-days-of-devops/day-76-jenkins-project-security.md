# Day 76: Jenkins Project Security

The xFusionCorp Industries has recruited some new developers. There are already some existing jobs on Jenkins and two of these new developers need permissions to access those jobs. The development team has already shared those requirements with the DevOps team, so as per details mentioned below grant required permissions to the developers.

Click on the `Jenkins` button on the top bar to access the Jenkins UI. Login using username `admin` and password `[redacted]`.

## Specific Requirements:

1. There is an existing Jenkins job named `Packages`, there are also two existing Jenkins users named `sam` with password `[redacted]` and `rohan` with password `[redacted]`.

2. Grant permissions to these users to access `Packages` job as per details mentioned below:

   a.) Make sure to select `Inherit permissions from parent ACL` under `inheritance strategy` for granting permissions to these users.

   b.) Grant mentioned permissions to `sam` user : `build`, `configure` and `read`.

   c.) Grant mentioned permissions to `rohan` user : `build`, `cancel`, `configure`, `read`, `update` and `tag`.

> **Credential note:** The temporary lab passwords are redacted from this public guide. Use the values shown in the active KodeKloud task; do not record them in screenshots or this repository.

## Solution

Jenkins has two relevant permission scopes here. The **global authorization matrix** lets `sam` and `rohan` sign in with **Overall → Read**. The **project matrix** grants the requested permissions only on the existing `Packages` job. The Matrix Authorization Strategy plugin was missing in this lab, so it had to be installed before the project-based strategy appeared.

### 🔌 Step 1: Install the Authorization Plugin

1. Open **Jenkins** from the lab toolbar and sign in as `admin` with the active lab password.
2. Go to **Manage Jenkins → Plugins → Available plugins**.
3. Find and install **Matrix Authorization Strategy**.
4. On the installation page, select **Restart Jenkins when installation is complete and no jobs are running** if offered. Wait for the login screen, then sign in again as `admin`.

> **Why:** The plugin provides **Project-based Matrix Authorization Strategy**. Without it, that choice is absent from Jenkins' authorization settings and the job cannot use a project-specific permission matrix. Wait for Jenkins to finish restarting before editing security settings or clicking the lab's Finish button.

### 🛡️ Step 2: Set the Global Authorization Strategy

Open **Manage Jenkins → Security**. On this page, find the section headed **Authorization**, below **Authentication** and **Security Realm**. In the **Authorization** dropdown, select **Project-based Matrix Authorization Strategy**. The strategy is **under Authorization**, not in the Authentication or Plugins section.

In the global permission matrix, retain or add these permissions:

| User | Global permission |
| --- | --- |
| `admin` | **Overall → Administer** |
| `sam` | **Overall → Read** |
| `rohan` | **Overall → Read** |

Use **Add user** for any missing row. Ensure `admin` has **Overall → Administer** *before saving*, then save the page. In the successful lab screenshot, Anonymous and Authenticated Users had no permissions.

> **Why:** **Overall → Read** allows each developer to enter Jenkins; it does not itself grant Build, Configure, or Read on every job. **Overall → Administer** preserves the administrator's ability to manage Jenkins and prevents an accidental lockout when changing strategies. Do not grant the requested **Job**, **Run**, or **SCM** permissions globally: that would apply beyond `Packages`. The screenshot confirmed the selected strategy and these three global rows in **Manage Jenkins → Security → Authorization**.

### 📦 Step 3: Enable Security on `Packages`

1. Open the existing **Packages** job and select **Configure**.
2. Under **General**, enable **Enable project-based security**.
3. In **Inheritance Strategy**, select **Inherit permissions from parent ACL** exactly.
4. Use **Add user** to add `sam` and `rohan` to this job's permission matrix.

> **Why:** This is the second, job-specific matrix. The required inheritance setting keeps the global permissions from the parent ACL, including `admin`'s administrator rights and the developers' Overall/Read access, while allowing additional permissions on `Packages` only. The **Add user** entries should match the Jenkins usernames, not a display name or password.

### 🎯 Step 4: Grant Only the Required Job Permissions

In the `Packages` matrix, select these exact permissions:

| User | Section | Permissions |
| --- | --- | --- |
| `sam` | **Job** | **Build**, **Configure**, **Read** |
| `rohan` | **Job** | **Build**, **Cancel**, **Configure**, **Read** |
| `rohan` | **Run** | **Update** |
| `rohan` | **SCM** | **Tag** |

Leave other selectable permissions unmarked, then save the job. Jenkins may display **Job → Discover** as *implied* when **Job → Read** is selected; an implied permission is not an extra checkbox to grant manually.

> **Why:** **Job → Build** starts a build, **Configure** changes the job's settings, **Read** allows viewing it, and **Cancel** stops a running build. **Run → Update** permits updating a build record, while **SCM → Tag** permits tagging through source-control integration. `sam` receives only the three requested job permissions; `rohan` receives those four Job permissions plus Run/Update and SCM/Tag. Keeping them in the project matrix limits this access to `Packages`.

### ✅ Step 5: Verify

Reopen **Manage Jenkins → Security → Authorization** and confirm that **Project-based Matrix Authorization Strategy** is selected, `admin` retains **Overall: Administer**, and both `sam` and `rohan` have **Overall: Read**. Then reopen **Packages → Configure → General** and check that **Enable project-based security** remains active, **Inheritance Strategy** reads **Inherit permissions from parent ACL**, and each user's selected permissions match the table in Step 4.

Capture screenshots of the global and job matrices for lab review without including passwords. The user confirmed that the lab was successful. The global screenshot showed the **Authorization** heading immediately above the strategy selector; that placement is important because the similarly named project matrix is configured on the job page.

> **Why:** A permission in the wrong scope can appear correct in one matrix but give access to other jobs or leave a user unable to see `Packages`. Reopening both saved pages checks the strategy, inheritance, and exact permission sets at their intended scopes.

## Best Practices

- **Preserve administrator access.** Confirm `admin` has Overall/Administer before saving a new global authorization strategy.
- **Keep job rights local.** Grant Build, Configure, Cancel, Read, Update, and Tag on the `Packages` job, not across all jobs.
- **Inspect inherited permissions.** Inheritance combines parent and project permissions; verify the global matrix does not already grant broader job access to the developers.
- **Distinguish explicit and implied permissions.** Jenkins may show implied permissions in gray; the selected boxes should still match the task's requested rights.
- **Protect lab credentials.** Use existing accounts and enter passwords only in the active Jenkins lab, not in this guide or screenshots.

### 📚 Official Documentation

- [Matrix Authorization Strategy plugin](https://plugins.jenkins.io/matrix-auth/)
- [Managing Jenkins plugins](https://www.jenkins.io/doc/book/managing/plugins/)
- [Managing security in Jenkins](https://www.jenkins.io/doc/book/security/managing-security/)
- [Jenkins permissions](https://www.jenkins.io/doc/book/security/access-control/permissions/)
