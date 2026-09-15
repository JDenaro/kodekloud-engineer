# Day 34: Git Hook

The Nautilus application development team was working on a git repository `/opt/official.git` which is cloned under `/usr/src/kodekloudrepos` directory present on `Storage server` in `Stratos DC`. The team want to setup a hook on this repository, please find below more details:

- Merge the `feature` branch into the `master` branch, but before pushing your changes complete below point.

- Create a `post-update` hook in this git repository so that whenever any changes are pushed to the `master` branch, it creates a release tag with name `release-2023-06-15`, where `2023-06-15` is supposed to be the current date. For example if today is `20th June, 2023` then the release tag must be `release-2023-06-20`. Make sure you test the hook at least once and create a release tag for today's release.

- Finally remember to push your changes.

`Note:` Perform this task using the `natasha` user, and ensure the repository or existing directory permissions are not altered.

## Specific Requirements:

1. Merge the `feature` branch into the `master` branch, but before pushing your changes complete below point.
2. Create a `post-update` hook in this git repository so that whenever any changes are pushed to the `master` branch, it creates a release tag with name `release-2023-06-15`, where `2023-06-15` is supposed to be the current date. For example if today is `20th June, 2023` then the release tag must be `release-2023-06-20`. Make sure you test the hook at least once and create a release tag for today's release.
3. Finally remember to push your changes.
4. `Note:` Perform this task using the `natasha` user, and ensure the repository or existing directory permissions are not altered.

## Solution

The working copy was on `feature`, while `master` still pointed to the initial commit. The `post-update` hook was installed in the bare repository `/opt/official.git` before the merge was pushed. The hook checks which references were updated, creates a date-based release tag only when `master` changes, and therefore the required `git push origin master` also tests the hook.

### 🔌 Step 1: Connect to the Storage Server and inspect the repository

```bash
ssh natasha@ststor01
cd /usr/src/kodekloudrepos/official
git status --short
git branch --show-current
git log
```

The working tree was clean, and the repository was on `feature`:

```text
feature
```

The relevant history was:

```text
commit 3853705cfafb1b7d597cb6e348d990502f63e293 (HEAD -> feature, origin/feature)
Author: Admin admin@kodekloud.com
Date:   Tue Sep 15 02:17:14 2026 +0000

    Add feature

commit c6adcfce2a164584ca872ba7217a9d3825c0e94b (origin/master, master)
Author: Admin admin@kodekloud.com
Date:   Tue Sep 15 02:17:14 2026 +0000

    initial commit
```

> **Why:** `ssh` opens a remote shell on the Storage Server as `natasha`. `cd` enters the cloned repository. `git status --short` checks for uncommitted changes in compact form; no output confirms a clean working tree. `git branch --show-current` displays the active branch, and `git log` shows the commit history and branch references. The output shows that `feature` contains `Add feature` and is ahead of `master`.

### 🪝 Step 2: Create the `post-update` hook in the bare repository

```bash
sudo vi /opt/official.git/hooks/post-update
```

Press `i` in `vi`, enter the following content, then press `Esc`, type `:wq`, and press `Enter`:

```sh
#!/bin/sh
for ref in "$@"; do
  case "$ref" in
    refs/heads/master)
      tag="release-$(date +%F)"
      git tag -f "$tag" "$ref"
      ;;
  esac
done
```

Make only the new hook executable:

```bash
sudo chmod +x /opt/official.git/hooks/post-update
```

> **Why:** A `post-update` hook is a server-side Git hook that runs after a push updates one or more references. `/opt/official.git` is the bare repository that receives the push, so its hook directory is the correct location. `sudo` is needed to edit that repository's hook file, while `chmod +x` gives execution permission only to the required hook and does not alter the existing repository or directory permissions. `"$@"` contains the references updated by the push. The `case` pattern limits the action to `refs/heads/master`; pushes to other branches do nothing. `date +%F` returns the current date as `YYYY-MM-DD`, and `git tag -f` creates or updates the lightweight release tag at the new `master` commit.

### 🔀 Step 3: Merge `feature` into `master`

```bash
git switch master
git merge feature
```

The merge was a fast-forward:

```text
Switched to branch 'master'
Your branch is up to date with 'origin/master'.
Updating c6adcfc..3853705
Fast-forward
feature.txt | 1 +
1 file changed, 1 insertion(+)
create mode 100644 feature.txt
```

> **Why:** `git switch master` changes the working copy to the destination branch. `git merge feature` incorporates the feature branch into `master`. Because `master` was an ancestor of `feature`, Git advanced the `master` reference directly with a fast-forward and did not create an unnecessary merge commit.

### 🚀 Step 4: Push `master` and trigger the hook

```bash
git push origin master
```

The lab returned:

```text
Total 0 (delta 0), reused 0 (delta 0), pack-reused 0
To /opt/official.git
c6adcfc..3853705  master -> master
```

> **Why:** `git push` publishes the local `master` reference to the remote repository. `origin` is the configured remote name and `master` is the branch being pushed. Updating `master` invokes the `post-update` hook in `/opt/official.git`, which creates the release tag for the Storage Server's current date. This push is also the required test of the hook.

### ✅ Step 5: Verify the release tag

```bash
sudo git --git-dir=/opt/official.git tag --list "release-$(date +%F)"
```

The lab returned:

```text
release-2026-09-15
```

> **Why:** `git tag --list` lists tags matching the supplied pattern. `--git-dir=/opt/official.git` tells Git to inspect the bare receiving repository directly, and `$(date +%F)` builds the tag pattern using the current date. The returned tag confirms that the hook ran after the `master` push and created today's release tag.

## Best Practices

- **Install the hook before pushing.** A receive-side hook must already exist and be executable when the push updates the target reference.
- **Filter the updated reference.** Check for `refs/heads/master` so pushes to `feature` or other branches do not create unintended release tags.
- **Use a dynamic date.** Generate the `YYYY-MM-DD` suffix at hook execution time instead of hardcoding a date that will become stale.
- **Keep the permission change scoped.** Change only the executable bit on the new hook; do not alter owners or permissions on the repository and its existing directories.
- **Use a fast-forward when possible.** Merging a descendant feature branch into `master` preserves a simple history without adding an unnecessary merge commit.
- **Verify the tag in the receiving repository.** The hook creates the tag in the bare remote, so inspect `/opt/official.git` directly after pushing.

### 📚 Official Documentation

- [Git hooks documentation](https://git-scm.com/docs/githooks)
- [Git post-update hook documentation](https://git-scm.com/docs/githooks#_post_update)
- [Git switch documentation](https://git-scm.com/docs/git-switch)
- [Git merge documentation](https://git-scm.com/docs/git-merge)
- [Git push documentation](https://git-scm.com/docs/git-push)
- [Git tag documentation](https://git-scm.com/docs/git-tag)
