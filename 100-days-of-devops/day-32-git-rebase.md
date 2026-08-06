# Day 32: Git Rebase

The Nautilus application development team has been working on a project repository /opt/news.git. This repo is cloned at /usr/src/kodekloudrepos on storage server in Stratos DC. They recently shared the following requirements with DevOps team:

One of the developers is working on feature branch and their work is still in progress, however there are some changes which have been pushed into the master branch, the developer now wants to rebase the feature branch with the master branch without loosing any data from the feature branch, also they don't want to add any merge commit by simply merging the master branch into the feature branch. Accomplish this task as per requirements mentioned.

Also remember to push your changes once done.

## Specific Requirements:

1. One of the developers is working on feature branch and their work is still in progress, however there are some changes which have been pushed into the master branch, the developer now wants to rebase the feature branch with the master branch without loosing any data from the feature branch, also they don't want to add any merge commit by simply merging the master branch into the feature branch. Accomplish this task as per requirements mentioned.
2. Also remember to push your changes once done.

## Solution

The repository started with `feature` behind `master`: `master` contained `Update info.txt`, while `feature` contained `Add new feature`. Rebasing `feature` onto `master` preserved the feature commit and produced a linear history without a merge commit. Because rebasing changes commit hashes, the updated `feature` branch required a protected force push using `--force-with-lease`.

### 🔌 Step 1: Connect to the Storage Server

```bash
ssh natasha@ststor01
```

> **Why:** `ssh` opens a secure remote shell on the Storage Server. `natasha` is the designated user for this server, and `ststor01` is the host containing the cloned repository.

### 🔑 Step 2: Open a root login shell

```bash
sudo su -
```

> **Why:** `sudo` runs a command with administrative privileges, and `su -` starts a root login shell. The repository and its Git metadata are managed by `root`, so this provides the permissions needed to rebase and push the repository.

### 🔎 Step 3: Inspect the branches and history

```bash
cd /usr/src/kodekloudrepos/news/
git status --short
git branch --list master feature
git log --oneline --decorate --all
```

The relevant initial history was:

```text
b155269 (origin/master, master) Update info.txt
c6613d5 (HEAD -> feature, origin/feature) Add new feature
903973b initial commit
```

`git status --short` produced no output, confirming that the working tree was clean.

> **Why:** `cd` enters the target repository. `git status --short` checks for uncommitted changes. `git branch --list master feature` confirms that both required branches exist. `git log --oneline --decorate --all` shows the branch references and commit relationship: `master` contains the newer update, while `feature` contains the developer's work.

### 🌿 Step 4: Rebase the feature branch onto master

```bash
git switch feature
git rebase master
git log --oneline --decorate --all
git status --short
```

Successful rebase output:

```text
Already on 'feature'
Successfully rebased and updated refs/heads/feature.
a33fa59 (HEAD -> feature) Add new feature
b155269 (origin/master, master) Update info.txt
c6613d5 (origin/feature) Add new feature
903973b initial commit
```

`git status --short` produced no output.

> **Why:** `git switch feature` selects the branch that must be updated. `git rebase master` temporarily replays the feature branch's commits on top of the current `master` tip. The feature work is preserved, but its commit receives a new hash, `a33fa59`, because its parent commit changed. Unlike a merge, rebase does not create a merge commit.

### 🚀 Step 5: Push the rebased feature branch

```bash
git push origin feature --force-with-lease
```

Successful output:

```text
Enumerating objects: 4, done.
Counting objects: 100% (4/4), done.
Delta compression using up to 16 threads
Compressing objects: 100% (2/2), done.
Writing objects: 100% (3/3), 296 bytes | 296.00 KiB/s, done.
Total 3 (delta 0), reused 0 (delta 0), pack-reused 0 (from 0)
To /opt/news.git
- c6613d5...a33fa59 feature -> feature (forced update)
```

> **Why:** `git push` publishes local commits to a remote repository. `origin` selects the configured remote, and `feature` selects the branch to publish. `--force-with-lease` is needed because the rebase changed the feature commit hash; unlike plain `--force`, it also checks that the remote branch has not changed unexpectedly since it was last observed.

### ✅ Step 6: Verify

```bash
git log --oneline --decorate --all
git status --short
```

Successful output:

```text
a33fa59 (HEAD -> feature, origin/feature) Add new feature
b155269 (origin/master, master) Update info.txt
903973b initial commit
```

`git status --short` produced no output, confirming that the working tree was clean. The final history is linear, `origin/feature` matches the local `feature` branch, and no merge commit was added.

> **Why:** `git log` confirms the final relationship between the local and remote branches. `git status --short` confirms that no uncommitted changes remain after the rebase and push.

## Best Practices

- **Inspect both branches before rebasing.** Confirm which branch contains the base changes and which branch contains the feature work.
- **Keep the working tree clean.** Commit or stash local changes before starting a rebase.
- **Use rebase to maintain linear history.** Rebasing integrates the latest base commits without creating a merge commit.
- **Prefer `--force-with-lease` after rebasing.** It updates the rewritten remote branch while protecting against overwriting unexpected remote changes.
- **Verify the remote-tracking branch.** Confirm that `origin/feature` points to the same commit as the local `feature` branch.

### 📚 Official Documentation

- [Git rebase documentation](https://git-scm.com/docs/git-rebase)
- [Git switch documentation](https://git-scm.com/docs/git-switch)
- [Git push documentation](https://git-scm.com/docs/git-push)
- [Git log documentation](https://git-scm.com/docs/git-log)
- [Git status documentation](https://git-scm.com/docs/git-status)
