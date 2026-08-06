# Day 31: Git Stash

The Nautilus application development team was working on a git repository /usr/src/kodekloudrepos/media present on Storage server in Stratos DC. One of the developers stashed some in-progress changes in this repository, but now they want to restore some of the stashed changes. Find below more details to accomplish this task:

Look for the stashed changes under /usr/src/kodekloudrepos/media git repository, and restore the stash with stash@{1} identifier. Further, commit and push your changes to the origin.

## Specific Requirements:

1. Look for the stashed changes under /usr/src/kodekloudrepos/media git repository, and restore the stash with stash@{1} identifier. Further, commit and push your changes to the origin.

## Solution

The repository was on the `master` branch with a clean working tree. The requested `stash@{1}` restored a new `welcome.txt` file. The change was committed as `restore stash@{1}` and pushed to `origin/master`.

### 🔌 Step 1: Connect to the Storage Server

```bash
ssh natasha@ststor01
```

> **Why:** `ssh` opens a secure remote shell on the Storage Server. `natasha` is the designated user for this server, and `ststor01` is the host containing the cloned repository.

### 🔑 Step 2: Open a root login shell

```bash
sudo su -
```

> **Why:** `sudo` runs a command with administrative privileges, and `su -` starts a root login shell. The repository and its Git metadata are managed by `root`, so this provides the permissions needed to restore the stash and update the repository.

### 🔎 Step 3: Inspect the repository and available stashes

```bash
cd /usr/src/kodekloudrepos/media
git branch --show-current
git status --short
git stash list
```

Successful output:

```text
master
stash@{0}: WIP on master: 6c47570 initial commit
stash@{1}: WIP on master: 6c47570 initial commit
```

The empty `git status --short` output confirmed that the working tree was clean before restoring the stash.

> **Why:** `cd` enters the repository. `git branch --show-current` displays the active branch. `git status --short` checks for uncommitted changes in compact form. `git stash list` displays the saved work and its identifiers, allowing us to select the exact `stash@{1}` requested by the challenge.

### ♻️ Step 4: Restore the requested stash

```bash
git stash apply stash@{1}
git status --short
git diff --stat
```

Relevant output:

```text
On branch master
Your branch is up to date with 'origin/master'.

Changes to be committed:
  (use "git restore --staged <file>..." to unstage)
        new file:   welcome.txt

A  welcome.txt
```

`git diff --stat` produced no output because the restored file was already staged by the stash.

> **Why:** `git stash apply` restores the changes from the selected stash without removing the stash entry. `git status --short` confirms that `welcome.txt` is staged for commit. `git diff --stat` normally summarizes unstaged changes; it is empty here because the restored change is already in the staging area.

### 📝 Step 5: Commit the restored changes

```bash
git commit -m "restore stash@{1}"
```

Successful output:

```text
[master 2aceedc] restore stash@{1}
 1 file changed, 1 insertion(+)
 create mode 100644 welcome.txt
```

> **Why:** `git commit` records the staged changes in the repository history. The `-m` option supplies the commit message, making it clear that the change came from `stash@{1}`.

### 🚀 Step 6: Push the commit to the origin

```bash
git push origin master
```

Successful output:

```text
Enumerating objects: 4, done.
Counting objects: 100% (4/4), done.
Delta compression using up to 16 threads
Compressing objects: 100% (2/2), done.
Writing objects: 100% (3/3), 308 bytes | 308.00 KiB/s, done.
Total 3 (delta 0), reused 0 (delta 0), pack-reused 0 (from 0)
To /opt/media.git
   6c47570..2aceedc  master -> master
```

> **Why:** `git push` publishes local commits to a remote repository. `origin` selects the repository's configured remote, and `master` selects the branch to update.

### ✅ Step 7: Verify

```bash
git log --oneline --decorate -n 3
git status --short
```

Successful output:

```text
2aceedc (HEAD -> master, origin/master) restore stash@{1}
6c47570 initial commit
```

`git status --short` produced no output, confirming that the working tree is clean. The log shows that both local `master` and `origin/master` point to the new commit.

> **Why:** `git log` confirms the final commit history and the remote-tracking branch. The `-n 3` option limits the output to the three most recent commits. `git status --short` confirms that no uncommitted changes remain.

## Best Practices

- **Inspect stash identifiers before restoring.** Stash references are positional, so list them immediately before selecting the requested entry.
- **Use `apply` when preserving the stash is useful.** It restores the changes without deleting the stash entry; use `pop` only when removing the stash is intentional.
- **Check the working tree before applying a stash.** A clean tree reduces the chance of mixing existing work with the restored changes.
- **Verify both the local and remote branches.** Matching `HEAD` and `origin/master` confirms that the restored commit was pushed successfully.

### 📚 Official Documentation

- [Git stash documentation](https://git-scm.com/docs/git-stash)
- [Git commit documentation](https://git-scm.com/docs/git-commit)
- [Git push documentation](https://git-scm.com/docs/git-push)
- [Git status documentation](https://git-scm.com/docs/git-status)
- [Git log documentation](https://git-scm.com/docs/git-log)
