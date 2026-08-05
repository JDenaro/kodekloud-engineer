# Day 26: Git Manage Remotes

The xFusionCorp development team added updates to the project that is maintained under /opt/cluster.git repo and cloned under /usr/src/kodekloudrepos/cluster. Recently some changes were made on Git server that is hosted on Storage server in Stratos DC. The DevOps team added some new Git remotes, so we need to update remote on /usr/src/kodekloudrepos/cluster repository as per details mentioned below:

a. In /usr/src/kodekloudrepos/cluster repo add a new remote dev_cluster and point it to /opt/xfusioncorp_cluster.git repository.

b. There is a file /tmp/index.html on same server; copy this file to the repo and add/commit to master branch.

c. Finally push master branch to this new remote origin.

## Specific Requirements:

1. In /usr/src/kodekloudrepos/cluster repo add a new remote dev_cluster and point it to /opt/xfusioncorp_cluster.git repository.
2. There is a file /tmp/index.html on same server; copy this file to the repo and add/commit to master branch.
3. Finally push master branch to this new remote origin.

## Solution

The repository is owned by `root`, while the Storage Server login uses `natasha`. Entering a root login shell avoids repeating `sudo` for Git metadata and repository file operations. The challenge calls the final destination “origin”, but the new remote required in step a is named `dev_cluster`, so the successful push uses `dev_cluster` explicitly.

### 🔌 Step 1: Connect to the Storage Server

```bash
ssh natasha@ststor01
```

> **Why:** `ssh` opens a secure remote shell. `natasha` is the designated Storage Server user and `ststor01` is the server that contains the cloned repository and the source file.

### 🔑 Step 2: Open a root login shell

```bash
sudo su -
```

> **Why:** `sudo` runs a command with administrative privileges, and `su -` starts a login shell as `root`. This is required because the repository and its `.git` directory are owned by `root`, so Git must have permission to update repository metadata and the working tree.

### 📁 Step 3: Enter the repository and inspect its existing remote

```bash
cd /usr/src/kodekloudrepos/cluster
git remote -v
```

The repository initially had this remote:

```text
origin  /opt/cluster.git (fetch)
origin  /opt/cluster.git (push)
```

> **Why:** `cd` changes to the repository directory. `git remote -v` lists each configured remote and the URL used for fetching and pushing. The existing `origin` remains unchanged.

### 🔗 Step 4: Add the new remote

```bash
git remote add dev_cluster /opt/xfusioncorp_cluster.git
git remote -v
```

Successful output included:

```text
dev_cluster     /opt/xfusioncorp_cluster.git (fetch)
dev_cluster     /opt/xfusioncorp_cluster.git (push)
origin          /opt/cluster.git (fetch)
origin          /opt/cluster.git (push)
```

> **Why:** `git remote add` registers a new remote name and its repository URL. The name `dev_cluster` is used later to send the `master` branch to `/opt/xfusioncorp_cluster.git`, while `origin` continues to point to `/opt/cluster.git`.

### 🌿 Step 5: Select the master branch and copy the requested file

```bash
git switch master
cp /tmp/index.html index.html
```

Successful branch output:

```text
Already on 'master'
Your branch is up to date with 'origin/master'.
```

> **Why:** `git switch master` selects the required destination branch. `cp` copies `/tmp/index.html` into the root of the current repository, where it can be tracked by Git.

### 🧾 Step 6: Add and commit the file on master

```bash
git add index.html
git commit -m "add index.html"
```

Successful commit output:

```text
[master 054805b] add index.html
 1 file changed, 10 insertions(+)
 create mode 100644 index.html
```

> **Why:** `git add` places `index.html` in the staging area. `git commit -m` records the staged file in the history of the active `master` branch with the supplied commit message.

### ✅ Step 7: Push master to the new remote

```bash
git push dev_cluster master
```

Successful output:

```text
Enumerating objects: 6, done.
Counting objects: 100% (6/6), done.
Delta compression using up to 16 threads
Compressing objects: 100% (4/4), done.
Writing objects: 100% (6/6), 583 bytes | 583.00 KiB/s, done.
Total 6 (delta 0), reused 0 (delta 0), pack-reused 0 (from 0)
To /opt/xfusioncorp_cluster.git
 * [new branch]      master -> master
```

The new `master` branch was successfully published to `dev_cluster`.

> **Why:** `git push` transfers local commits to a remote repository. `dev_cluster` selects the new remote, and `master` specifies the branch to publish. The `[new branch] master -> master` message confirms that the destination remote received the branch.

## Best Practices

- **Use descriptive remote names.** `dev_cluster` identifies the purpose of the new destination and avoids confusing it with the existing `origin`.
- **Inspect remotes before changing them.** `git remote -v` confirms the current configuration and helps prevent accidental changes to an existing remote.
- **Commit only the requested file.** Adding `index.html` explicitly keeps unrelated working-tree changes out of the commit.
- **Specify the remote and branch when pushing.** `git push dev_cluster master` makes the destination and branch unambiguous.
- **Protect repository ownership.** Use the required administrative privileges without changing ownership or permissions on the existing repository.

### 📚 Official Documentation

- [Git remote documentation](https://git-scm.com/docs/git-remote)
- [Git switch documentation](https://git-scm.com/docs/git-switch)
- [Git add documentation](https://git-scm.com/docs/git-add)
- [Git commit documentation](https://git-scm.com/docs/git-commit)
- [Git push documentation](https://git-scm.com/docs/git-push)
