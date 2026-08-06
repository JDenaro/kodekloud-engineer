# Day 30: Git Hard Reset

The Nautilus application development team was working on a git repository /usr/src/kodekloudrepos/cluster present on Storage server in Stratos DC. This was just a test repository and one of the developers just pushed a couple of changes for testing, but now they want to clean this repository along with the commit history/work tree, so they want to point back the HEAD and the branch itself to a commit with message add data.txt file. Find below more details:

In /usr/src/kodekloudrepos/cluster git repository, reset the git commit history so that there are only two commits in the commit history i.e initial commit and add data.txt file.

Also make sure to push your changes.

## Specific Requirements:

1. In /usr/src/kodekloudrepos/cluster git repository, reset the git commit history so that there are only two commits in the commit history i.e initial commit and add data.txt file.
2. Also make sure to push your changes.

## Solution

The target commit was `c280aa0 add data.txt file`. A hard reset moved the local `master` branch and working tree back to that commit, removing the ten later test commits. Because the remote branch still contained those commits, the rewritten history had to be force-pushed.

### 🔌 Step 1: Connect to the Storage Server

```bash
ssh natasha@ststor01
```

> **Why:** `ssh` opens a secure remote shell on the Storage Server. `natasha` is the designated user for this server, and `ststor01` is the host containing the cloned repository.

### 🔑 Step 2: Open a root login shell

```bash
sudo su -
```

> **Why:** `sudo` runs a command with administrative privileges, and `su -` starts a root login shell. The repository and its `.git` metadata are owned by `root`, so this provides the permissions Git needs to rewrite the repository history and working tree.

### 🔎 Step 3: Inspect the repository history

```bash
cd /usr/src/kodekloudrepos/cluster
git log --oneline --decorate
```

The relevant initial output was:

```text
8db75d5 (HEAD -> master, origin/master) Test Commit10
4563c8a Test Commit9
acbe8e8 Test Commit8
4be4cf2 Test Commit7
6e44f9b Test Commit6
dab0eb8 Test Commit5
d116b9f Test Commit4
83d9aaf Test Commit3
1f67334 Test Commit2
9043e5f Test Commit1
c280aa0 add data.txt file
ca77805 initial commit
```

> **Why:** `cd` enters the target repository. `git log` displays the commit history, while `--oneline` uses a compact one-line format and `--decorate` shows branch and remote-tracking references. This identifies `c280aa0` as the required target and confirms that `master` currently contains ten later test commits.

### 🧹 Step 4: Reset the local branch and working tree

```bash
git reset --hard c280aa0
```

Successful output:

```text
HEAD is now at c280aa0 add data.txt file
```

> **Why:** `git reset` moves the current branch reference to a selected commit. The `--hard` option also updates `HEAD`, the staging area, and the working tree to match that commit. This is intentionally destructive here because the challenge requires removing the test commits and resetting the work tree.

### 🚀 Step 5: Push the rewritten history

```bash
git push origin master --force
```

Successful output:

```text
Total 0 (delta 0), reused 0 (delta 0), pack-reused 0 (from 0)
To /opt/cluster.git
- 8db75d5...c280aa0 master -> master (forced update)
```

> **Why:** `git push` publishes local commits to a remote repository. `origin` selects the existing remote, and `master` selects the branch to update. The `--force` option is required because the reset rewrote history and the new branch tip is an ancestor of the current remote tip, so a normal fast-forward push cannot update the remote. Force-pushing should only be used when replacing the remote history is intentional.

### ✅ Step 6: Verify

```bash
git log --oneline --decorate
git status --short
```

Successful output:

```text
c280aa0 (HEAD -> master, origin/master) add data.txt file
ca77805 initial commit
```

`git status --short` produced no output, confirming that the working tree is clean. The log shows exactly two commits, and `HEAD`, `master`, and `origin/master` all point to `c280aa0`.

> **Why:** `git log` confirms the final local and remote-tracking history. `git status --short` reports pending changes in compact form; no output confirms that the hard reset left no uncommitted work.

## Best Practices

- **Inspect before resetting.** Identify the exact target commit and confirm the branch before using a destructive reset.
- **Use `--hard` deliberately.** It changes the branch, index, and working tree, so it should only be used when discarding later changes is intentional.
- **Force-push only after rewriting intentionally.** A force-push can replace remote history and should not be used casually on a shared branch.
- **Verify both history and working-tree state.** Check that the remote-tracking branch matches the intended commit and that `git status --short` is clean.

### 📚 Official Documentation

- [Git reset documentation](https://git-scm.com/docs/git-reset)
- [Git push documentation](https://git-scm.com/docs/git-push)
- [Git log documentation](https://git-scm.com/docs/git-log)
- [Git status documentation](https://git-scm.com/docs/git-status)
