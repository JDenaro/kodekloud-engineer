# Day 24: Git Create Branches

Nautilus developers are actively working on one of the project repositories, /usr/src/kodekloudrepos/cluster. Recently, they decided to implement some new features in the application, and they want to maintain those new changes in a separate branch. Below are the requirements that have been shared with the DevOps team:

On Storage server in Stratos DC create a new branch xfusioncorp_cluster from master branch in /usr/src/kodekloudrepos/cluster git repo.

Please do not try to make any changes in the code.

## Specific Requirements:

1. On Storage server in Stratos DC create a new branch xfusioncorp_cluster from master branch in /usr/src/kodekloudrepos/cluster git repo.
2. Please do not try to make any changes in the code.

## Solution

The repository is owned by `root`, while the lab connection uses `natasha`. Git can read the repository after the path is marked as trusted, but creating a branch writes metadata inside `.git`. Because `natasha` does not have write permission there, the branch-creation command must be run with `sudo`.

### 🔌 Step 1: Connect to the Storage Server

From the Jump Host, connect to the Storage Server as `natasha`:

```bash
ssh natasha@ststor01
```

> **Why:** `ssh` opens a secure remote shell. `natasha` is the account used for the lab, and `ststor01` is the Storage Server that contains the repository.

### 📁 Step 2: Enter the repository and inspect its ownership

```bash
cd /usr/src/kodekloudrepos/cluster
ls -la
```

The relevant ownership from the lab was:

```text
drwxr-xr-x 3 root root 4096 Aug  5 03:06 .
drwxr-xr-x 3 root root 4096 Aug  5 03:06 ..
drwxr-xr-x 7 root root 4096 Aug  5 03:06 .git
-rw-r--r-- 1 root root   34 Aug  5 03:06 data.txt
-rw-r--r-- 1 root root   34 Aug  5 03:06 info.txt
```

> **Why:** `cd` changes to the repository directory. `ls -la` displays hidden entries such as `.git` and shows their owners and permissions. The files and Git metadata belong to `root`, so `natasha` can read them but cannot create the lock file and branch metadata required by a branch operation.

### 🛡️ Step 3: Mark the known repository as trusted

```bash
git config --global --add safe.directory /usr/src/kodekloudrepos/cluster
```

> **Why:** Git's `safe.directory` setting identifies a repository path that the current user trusts even when its owner differs from the current user. This addresses Git's ownership safety check for this specific lab path. It does not grant write permissions and does not change the repository files.

### 🌿 Step 4: Create the branch with `git switch` (primary method)

```bash
sudo git -C /usr/src/kodekloudrepos/cluster switch -c xfusioncorp_cluster master
```

Successful output:

```text
Switched to a new branch 'xfusioncorp_cluster'
```

> **Why:** `sudo` runs this Git operation with the privileges of `root`, the owner of `.git`; this is necessary because creating a branch writes Git metadata and `natasha` cannot write to that directory. The `-C` option tells Git which repository to use. `switch -c` creates a new branch and switches to it, while `master` supplies the starting point. The working-tree files are not edited.

### 🔁 Alternative: Create the branch with `git checkout`

In a fresh lab, the equivalent command is:

```bash
sudo git -C /usr/src/kodekloudrepos/cluster checkout -b xfusioncorp_cluster master
```

Expected output:

```text
Switched to a new branch 'xfusioncorp_cluster'
```

> **Why:** `checkout -b` is the traditional Git syntax for creating and switching to a branch. It has the same result as `switch -c` for this task. Use this command instead of the `git switch` command, not after it, because the branch should be created only once.

### ✅ Step 5: Verify

```bash
sudo git -C /usr/src/kodekloudrepos/cluster branch --show-current
```

Expected output:

```text
xfusioncorp_cluster
```

This confirms that the new branch is the active branch in the requested repository.

## Best Practices

- **Use a specific trusted path.** Add only `/usr/src/kodekloudrepos/cluster` to `safe.directory`; do not trust every repository with a wildcard.
- **Use `sudo` only for the Git operation that needs it.** This allows Git to update root-owned metadata without changing ownership or permissions.
- **Prefer `git switch` for new workflows.** `switch -c` clearly communicates that the command creates and selects a branch.
- **Preserve the working tree.** Creating a branch changes Git metadata, not the application files, so no code changes are needed for this challenge.

### 📚 Official Documentation

- [Git `switch` documentation](https://git-scm.com/docs/git-switch)
- [Git `checkout` documentation](https://git-scm.com/docs/git-checkout)
- [Git `config` documentation](https://git-scm.com/docs/git-config)
