# Day 28: Git Cherry Pick

The Nautilus application development team has been working on a project repository /opt/cluster.git. This repo is cloned at /usr/src/kodekloudrepos on storage server in Stratos DC. They recently shared the following requirements with the DevOps team:

There are two branches in this repository, master and feature. One of the developers is working on the feature branch and their work is still in progress, however they want to merge one of the commits from the feature branch to the master branch, the message for the commit that needs to be merged into master is Update info.txt. Accomplish this task for them, also remember to push your changes eventually.

## Specific Requirements:

1. There are two branches in this repository, master and feature. One of the developers is working on the feature branch and their work is still in progress, however they want to merge one of the commits from the feature branch to the master branch, the message for the commit that needs to be merged into master is Update info.txt. Accomplish this task for them, also remember to push your changes eventually.

## Solution

The commit with the message `Update info.txt` was `650bd8d`. The later `Update welcome.txt` commit remained only on `feature`, so cherry-picking the required commit avoided merging the developer's unfinished work.

### 🔌 Step 1: Connect to the Storage Server

```bash
ssh natasha@ststor01
```

> **Why:** `ssh` opens a secure remote shell. `natasha` is the designated user for the Storage Server, and `ststor01` is the server containing the cloned repository.

### 🔑 Step 2: Open a root login shell

```bash
sudo su -
```

> **Why:** `sudo` runs a command with administrative privileges, and `su -` starts a root login shell. The repository is owned by `root`, so this provides the permissions Git needs to update its metadata.

### 🔎 Step 3: Identify the required commit

```bash
cd /usr/src/kodekloudrepos/cluster
git branch --list master feature
git log --oneline --decorate feature -n 5
```

The relevant output was:

```text
- feature
  master
1b6931a (HEAD -> feature, origin/feature) Update welcome.txt
650bd8d Update info.txt
a4664eb (origin/master, master) Add welcome.txt
4768ddc initial commit
```

> **Why:** `cd` enters the cloned repository. `git branch --list` confirms that `master` and `feature` exist. `git log --oneline --decorate feature -n 5` shows the five most recent commits on `feature` in a compact format, including the abbreviated commit hash and branch references. The required commit is `650bd8d`, while the newer `Update welcome.txt` commit must not be merged.

### 🌿 Step 4: Select the master branch

```bash
git switch master
```

Successful output:

```text
Switched to branch 'master'
Your branch is up to date with 'origin/master'.
```

> **Why:** `git switch master` changes the active branch to `master`, which is the destination for the selected commit.

### 🍒 Step 5: Cherry-pick the required commit

```bash
git cherry-pick 650bd8d
```

Successful output:

```text
[master 2d2cbaf] Update info.txt
 Date: Wed Aug 5 20:00:38 2026 +0000
 1 file changed, 1 insertion(+), 1 deletion(-)
```

> **Why:** `git cherry-pick` applies the changes introduced by one existing commit to the current branch. Because `master` was active, only the changes from `650bd8d` were copied to `master`; the unfinished `Update welcome.txt` commit was not included.

### 🚀 Step 6: Push master to the remote repository

```bash
git push origin master
```

Successful output:

```text
Enumerating objects: 5, done.
Counting objects: 100% (5/5), done.
Delta compression using up to 16 threads
Compressing objects: 100% (2/2), done.
Writing objects: 100% (3/3), 315 bytes | 315.00 KiB/s, done.
Total 3 (delta 0), reused 0 (delta 0), pack-reused 0 (from 0)
To /opt/cluster.git
   a4664eb..2d2cbaf  master -> master
```

> **Why:** `git push` sends local commits to a remote repository. `origin` selects the repository's existing remote, and `master` specifies the branch to publish.

### ✅ Step 7: Verify

```bash
git log --oneline --decorate -n 4
git status --short
```

Successful log output:

```text
2d2cbaf (HEAD -> master, origin/master) Update info.txt
a4664eb Add welcome.txt
4768ddc initial commit
```

`git status --short` produced no output, confirming that the working tree was clean. The log also confirms that `master` and `origin/master` point to the new cherry-picked commit and that `Update welcome.txt` was not added to `master`.

> **Why:** `git log` confirms the commit history and remote-tracking reference. `git status --short` reports uncommitted changes in a compact format; no output means there are no pending changes.

## Best Practices

- **Cherry-pick only the required commit.** This transfers one reviewed change without merging unrelated or unfinished work from another branch.
- **Inspect the source branch first.** Reviewing the commit history prevents selecting the wrong change.
- **Switch to the destination branch before cherry-picking.** Git applies the selected commit to whichever branch is currently active.
- **Push the destination branch explicitly.** `git push origin master` makes the target remote and branch clear.
- **Verify the remote-tracking branch.** Seeing `origin/master` at the same commit as `HEAD` confirms that the push succeeded.

### 📚 Official Documentation

- [Git cherry-pick documentation](https://git-scm.com/docs/git-cherry-pick)
- [Git switch documentation](https://git-scm.com/docs/git-switch)
- [Git log documentation](https://git-scm.com/docs/git-log)
- [Git push documentation](https://git-scm.com/docs/git-push)
- [Git branch documentation](https://git-scm.com/docs/git-branch)
