# Day 70: Configure Jenkins User Access

The Nautilus team is integrating Jenkins into their CI/CD pipelines. After setting up a new Jenkins server, they're now configuring user access for the development team, Follow these steps:

## Specific Requirements:

1. Click on the `Jenkins` button on the top bar to access the Jenkins UI. Login with username `admin` and password `[redacted]`.
2. Create a jenkins user named `rose` with the password `[redacted]`. Their full name should match `Rose`.

3. Utilize the `Project-based Matrix Authorization Strategy` to assign `overall read` permission to the `rose` user.

4. Remove all permissions for `Anonymous` users (if any) ensuring that the `admin` user retains overall `Administer` permissions.

5. For the existing job, grant `rose` user only `read` permissions, disregarding other permissions such as Agent, SCM etc.

`Note:`

1. You may need to install plugins and restart Jenkins service. After plugins installation, select `Restart Jenkins when installation is complete and no jobs are running` on plugin installation/update page.

2. After restarting the Jenkins service, wait for the Jenkins login page to reappear before proceeding. Avoid clicking `Finish` immediately after restarting the service.

3. Capture screenshots of your configuration for review purposes. Consider using screen recording software like `loom.com` for documentation and sharing.

> **Credential note:** The temporary lab passwords in the original challenge are redacted from this public guide. Enter the values shown in your active KodeKloud task.

## Solution

Jenkins separates **authentication** (who the user is) from **authorization** (what the user may do). The successful setup gave `rose` global **Overall/Read** access, then gave her **Job/Read** only on the existing `Helloworld` job. These are two different matrices. In this lab, the Matrix Authorization Strategy plugin also had to be installed and Jenkins restarted before the strategy could be selected.

### 🔐 Step 1: Sign In and Install the Authorization Plugin

1. Click **Jenkins** in the lab's top bar and sign in as `admin` with the current lab password.
2. Open **Manage Jenkins → Plugins** and install **Matrix Authorization Strategy** from **Available plugins**.
3. On the installation/update page, select **Restart Jenkins when installation is complete and no jobs are running**.
4. Wait for the login page to return, then sign in again as `admin`. Do not click the lab's **Finish** button while Jenkins is restarting.

> **Why:** The plugin provides the project-based matrix authorization strategy. Downloading it is not enough if Jenkins has not yet loaded it; the restart completes activation. Waiting for the login page confirms Jenkins is available before changing access rules.

### 👤 Step 2: Create the User

1. Open **Manage Jenkins → Users → Create User**.
2. Enter `rose` as the username and `Rose` as the full name.
3. Enter and confirm the password from the current lab prompt, then select **Create User**.

> **Why:** Creating the account establishes the identity Jenkins will later match in its permission matrices. The username is `rose`; the interface may display the full name `Rose` in those matrices. The password is entered in Jenkins, not stored in this guide.

### 🛡️ Step 3: Configure Global Authorization

1. Open **Manage Jenkins → Security**.
2. Under **Authorization**, select **Project-based Matrix Authorization Strategy**.
3. Use **Add user** to add `admin`, then grant **Overall → Administer**.
4. Use **Add user** to add `rose`, then grant **Overall → Read** only.
5. Leave **Anonymous** and **Authenticated Users** with no permissions, and save.

> **Why:** **Overall/Read** lets `rose` enter Jenkins; it does not grant access to every job. **Overall/Administer** keeps `admin` able to manage the server and implies the other permissions shown in gray in Jenkins. The global matrix initially displayed no permissions for Anonymous or Authenticated Users, so nothing had to be added for them. Do not grant **Job/Read** globally: project-based permissions granted here would apply to all jobs rather than only `Helloworld`.

### 📂 Step 4: Grant Read Access on `Helloworld`

1. Open the existing **Helloworld** job and select **Configure**.
2. Under **General**, enable **Enable project-based security**.
3. Leave **Inheritance Strategy** at **Inherit permissions from parent ACL**.
4. Add the `rose` user to this job's matrix and select only **Job → Read**.
5. Leave **Anonymous** and **Authenticated Users** with no permissions on the job, then save.

> **Why:** A project-specific permission applies to this job instead of every Jenkins job. With the inherited global matrix from Step 3, `rose` receives Overall/Read plus this job's Job/Read, while `admin` remains an administrator. Jenkins displays **Job/Discover** as *implied* by Job/Read; it is not another permission that must be selected. Do not select Job/Build, Job/Configure, Agent, SCM, or other permissions for `rose`.

### ✅ Step 5: Verify

1. Reopen **Manage Jenkins → Security** and confirm `admin` has **Overall/Administer**, `rose` has only **Overall/Read**, and Anonymous has no permissions.
2. Reopen **Helloworld → Configure** and confirm project-based security remains enabled and the `Rose` row summarizes **Job: Read**.
3. Confirm **Discover** is only marked *implied*, with no other job permissions selected for `Rose`.
4. Capture screenshots of the global and job matrices for the lab review, without exposing passwords.

> **Why:** Reopening both screens checks that the permissions were saved at the correct scopes. The successful lab showed `admin` with Overall/Administer, `Rose` with Overall/Read globally, and `Rose` with Job/Read on `Helloworld`; the task then reported success.

## Best Practices

- **Preserve administrator access before saving.** Add `admin` with Overall/Administer before changing the global authorization strategy to avoid locking administrators out.
- **Grant job access at the job scope.** Project-based rules let a user read one job without granting Job/Read across the entire Jenkins instance.
- **Watch inherited permissions.** A job's permissions are combined with global permissions by default; empty Anonymous and Authenticated Users rows prevent unintended access in this lab.
- **Distinguish implied from selected permissions.** Overall/Administer and Job/Read imply other capabilities in the UI. Gray *implied* labels do not mean you granted each one separately.
- **Keep lab credentials out of the repository.** Screenshots should show the permission matrices, not passwords or session secrets.

### 📚 Official Documentation

- [Managing Jenkins users](https://www.jenkins.io/doc/book/managing/users/)
- [Managing Jenkins plugins](https://www.jenkins.io/doc/book/managing/plugins/)
- [Matrix Authorization Strategy plugin](https://plugins.jenkins.io/matrix-auth/)
- [Managing security in Jenkins](https://www.jenkins.io/doc/book/security/managing-security/)
- [Jenkins permissions](https://www.jenkins.io/doc/book/security/access-control/permissions/)
