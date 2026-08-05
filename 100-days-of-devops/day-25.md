# Day 25: Git Merge Branches

The Nautilus application development team has been working on a project repository /opt/games.git. This repo is cloned at /usr/src/kodekloudrepos on storage server in Stratos DC. They recently shared the following requirements with DevOps team:

Create a new branch xfusion in /usr/src/kodekloudrepos/games repo from master and copy the /tmp/index.html file (present on storage server itself) into the repo. Further, add/commit this file in the new branch and merge back that branch into master branch. Finally, push the changes to the origin for both of the branches.

## Specific Requirements:

1. Create a new branch xfusion in /usr/src/kodekloudrepos/games repo from master and copy the /tmp/index.html file (present on storage server itself) into the repo.
2. Further, add/commit this file in the new branch and merge back that branch into master branch.
3. Finally, push the changes to the origin for both of the branches.

## Solution

The repository and its `.git` directory are owned by `root`, while the initial connection is made with `natasha`. The simplest approach is to enter a root login shell once with `sudo su -`; after that, the Git and file commands can be run without repeating `sudo` on every line. The only application-file change is the requested `/tmp/index.html` file.

### 🔌 Step 1: Connect to the Storage Server

```bash
ssh natasha@ststor01
```

> **Why:** `ssh` opens a secure remote shell. `natasha` is the account used to access the Storage Server, and `ststor01` is the server that contains the cloned repository.

### 🔑 Step 2: Open a root login shell

```bash
sudo su -
```

> **Why:** `sudo` runs a command with administrative privileges. `su -` starts a login shell as `root`, so the remaining commands can write to the root-owned repository without a separate `sudo` prefix on every command. This is needed because Git must write branch, index, and commit metadata inside `.git`, and the repository is not writable by `natasha`.

### 📁 Step 3: Enter the repository

```bash
cd /usr/src/kodekloudrepos/games
ls -la
```

The repository ownership in the lab was:

```text
drwxr-xr-x 3 root root 4096 Aug  5 03:22 .
drwxr-xr-x 3 root root 4096 Aug  5 03:22 ..
drwxr-xr-x 7 root root 4096 Aug  5 03:22 .git
-rw-r--r-- 1 root root   34 Aug  5 03:22 info.txt
-rw-r--r-- 1 root root   26 Aug  5 03:22 welcome.txt
```

> **Why:** `cd` changes to the repository directory. `ls -la` lists regular and hidden files, including `.git`, and displays ownership and permissions. The output explains why administrative privileges are required for the remaining operations.

### 🌿 Step 4: Create the feature branch from master

```bash
git switch master
git switch -c xfusion
```

Successful output:

```text
Already on 'master'
Your branch is up to date with 'origin/master'.
Switched to a new branch 'xfusion'
```

> **Why:** The first command makes `master` the starting branch. `switch -c` creates the new `xfusion` branch at the current `master` commit and switches to it. Creating a branch changes Git metadata, not the application files.

### 📄 Step 5: Copy the requested file into the repository

```bash
cp /tmp/index.html index.html
```

> **Why:** `cp` copies the supplied file from `/tmp` into the root of the current repository. Because the shell is running as `root`, the command can write to the root-owned working tree.

### 🧾 Step 6: Add and commit the file on xfusion

```bash
git add index.html
git commit -m "add index.html"
```

Successful commit output:

```text
[xfusion f9e93bc] add index.html
 1 file changed, 1 insertion(+)
 create mode 100644 index.html
```

> **Why:** `git add` places the new file in Git's staging area, which is the set of changes selected for the next commit. `git commit -m` records that staged snapshot in the current `xfusion` branch with a descriptive message.

### 🔀 Step 7: Merge xfusion into master

```bash
git switch master
git merge xfusion
```

Successful output:

```text
Switched to branch 'master'
Your branch is up to date with 'origin/master'.
Updating 6ef350d..f9e93bc
Fast-forward
 index.html | 1 +
 1 file changed, 1 insertion(+)
 create mode 100644 index.html
```

> **Why:** `git switch master` selects the destination branch. `git merge xfusion` incorporates the commit from `xfusion` into `master`. Git used a fast-forward merge because `master` had no new commits after `xfusion` was created, so no merge commit was necessary.

### 🚀 Step 8: Push both branches to origin

```bash
git push origin master
git push origin xfusion
```

Successful output for master included:

```text
To /opt/games.git
   6ef350d..f9e93bc  master -> master
```

Successful output for xfusion included:

```text
To /opt/games.git
 * [new branch]      xfusion -> xfusion
```

> **Why:** `git push` publishes local commits to a remote repository. `origin` is the remote name configured for this clone, and naming each branch explicitly ensures that both `master` and `xfusion` are updated in `/opt/games.git`.

### ✅ Step 9: Verify

```bash
git branch --list master xfusion
git status --short
```

Expected result:

```text
* master
  xfusion
```

The asterisk confirms that `master` is active, `xfusion` exists, and an empty `git status --short` result confirms there are no uncommitted changes.

## Best Practices

- **Use one administrative shell.** Entering sudo su - once keeps the workflow readable while avoiding unnecessary ownership or permission changes.
- **Create the branch before changing files.** This ensures the requested file is committed first on xfusion, not directly on master.
- **Merge only after committing.** A clean commit gives Git a clear change set to fast-forward into master.
- **Push branches explicitly.** Push master and xfusion separately so both required refs are present in origin.
- **Keep the change focused.** Only copy and commit the requested index.html; do not alter the existing repository files.

### 📚 Official Documentation

- [Git switch documentation](https://git-scm.com/docs/git-switch)
- [Git add documentation](https://git-scm.com/docs/git-add)
- [Git commit documentation](https://git-scm.com/docs/git-commit)
- [Git merge documentation](https://git-scm.com/docs/git-merge)
- [Git push documentation](https://git-scm.com/docs/git-push)
