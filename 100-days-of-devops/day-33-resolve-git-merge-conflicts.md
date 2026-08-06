# Day 33: Resolve Git Merge Conflicts

Sarah and Max were working on writting some stories which they have pushed to the repository. Max has recently added some new changes and is trying to push them to the repository but he is facing some issues. Below you can find more details:

SSH into storage server using user max and the temporary password supplied by the lab. Under /home/max you will find the story-blog repository. Try to push the changes to the origin repo and fix the issues. The story-index.txt must have titles for all 4 stories. Additionally, there is a typo in The Lion and the Mooose line where Mooose should be Mouse.

Click on the Gitea UI button on the top bar. You should be able to access the Gitea page. You can login to Gitea server from UI using username sarah and the temporary password supplied by the lab or username max and the temporary password supplied by the lab.

Note: For these kind of scenarios requiring changes to be done in a web UI, please take screenshots so that you can share it with us for review in case your task is marked incomplete. You may also consider using screen recording software such as loom.com to record and share your work.

## Specific Requirements:

1. SSH into storage server using user max and the temporary password supplied by the lab. Under /home/max you will find the story-blog repository.
2. Try to push the changes to the origin repo and fix the issues.
3. The story-index.txt must have titles for all 4 stories.
4. Correct the typo in The Lion and the Mooose line so that Mooose becomes Mouse.
5. Access the Gitea UI and capture screenshots of the completed workflow for review.

## Solution

The local repository contained Max's story and a typo in `story-index.txt`. After the typo was corrected, the first push was rejected because the remote `master` branch had newer commits. Pulling with rebase produced an `add/add` conflict in `story-index.txt`. The conflict was resolved by keeping all four required titles, then the local commits were rebased and pushed successfully. Temporary passwords and credential-bearing URLs are intentionally omitted from this guide.

### 🔌 Step 1: Connect to the Storage Server and inspect the repository

```bash
ssh max@ststor01
cd /home/max/story-blog/
git status --short
git branch --show-current
git log --oneline --decorate --all
ls
cat story-index.txt
```

Relevant initial output:

```text
master
0f139c7 (HEAD -> master) Added the fox and grapes story
1eee234 (origin/master, origin/HEAD) Merge branch 'story/frogs-and-ox'
03864cf Fix typo in story title
f4ed20f Completed frogs-and-ox story
8665809 Added the lion and mouse story
ed1b72e Add incomplete frogs-and-ox story
fox-and-grapes.txt  frogs-and-ox.txt  lion-and-mouse.txt  story-index.txt
1. The Lion and the Mooose
2. The Frogs and the Ox
3. The Fox and the Grapes
4. The Donkey and the Dog
```

`git status --short` produced no output, confirming that the working tree was clean.

> **Why:** `ssh` opens a secure shell on the Storage Server. `cd` enters Max's cloned repository. `git status --short` checks for local changes. `git branch --show-current` displays the active branch. `git log --oneline --decorate --all` shows the commit history, branch references, and remote-tracking references. `ls` lists the story files, and `cat` displays the index that must be corrected.

### ✏️ Step 2: Correct the typo and commit the local fix

Open `story-index.txt` with `vi` and change only `Mooose` to `Mouse`. Keep all four title lines.

```bash
vi story-index.txt
git diff -- story-index.txt
git add story-index.txt
git commit -m "fix story typo"
```

The corrected file contained:

```text
1. The Lion and the Mouse
2. The Frogs and the Ox
3. The Fox and the Grapes
4. The Donkey and the Dog
```

Successful commit output:

```text
[master fe37972] fix story typo
 1 file changed, 2 insertions(+), 2 deletions(-)
```

> **Why:** `vi` edits the file without replacing the other titles. `git diff -- story-index.txt` reviews only that file. `git add` stages the correction, and `git commit -m` records it with a descriptive message. The `-m` option supplies the commit message directly.

### 🔄 Step 3: Reconcile the local and remote histories

The initial `git push origin master` was rejected because `origin/master` had commits that were not present locally. Use a rebase pull to bring those commits in without creating a merge commit:

```bash
git pull --rebase origin master
git status --short
```

The pull fetched a newer remote tip and reported:

```text
1eee234..5dfd972  master     -> origin/master
CONFLICT (add/add): Merge conflict in story-index.txt
```

The status showed `story-index.txt` as conflicted.

> **Why:** `git pull` fetches remote changes and integrates them into the current branch. `origin` selects the configured Gitea remote, and `master` selects the remote branch. `--rebase` replays Max's local commits on top of the updated remote history, preserving a linear history. Git paused because both histories added `story-index.txt`.

### 🧩 Step 4: Resolve the add/add conflict

Open the conflicted file:

```bash
vi story-index.txt
```

Remove the conflict markers and leave exactly:

```text
1. The Lion and the Mouse
2. The Frogs and the Ox
3. The Fox and the Grapes
4. The Donkey and the Dog
```

Then stage the resolved files and continue the rebase:

```bash
git add story-index.txt fox-and-grapes.txt
git rebase --continue
```

Successful output included:

```text
[detached HEAD 73886ae] Added the fox and grapes story
 2 files changed, 23 insertions(+), 1 deletion(-)
 create mode 100644 fox-and-grapes.txt
dropping fe37972acf2b4af928fb205b96bbfbc62cae71dc fix story typo -- patch contents already upstream
Successfully rebased and updated refs/heads/master.
```

> **Why:** Conflict markers identify the competing versions and must not remain in the committed file. `git add` marks both the resolved index and the story file as ready for the rebase to continue. `git rebase --continue` finishes replaying the local work. Git dropped the separate typo-fix commit because the corrected content was already included in the remote changes.

### 🚀 Step 5: Push the resolved history

```bash
git push origin master
```

Successful output:

```text
Enumerating objects: 6, done.
Counting objects: 100% (6/6), done.
Delta compression using up to 16 threads
Compressing objects: 100% (4/4), done.
Writing objects: 100% (4/4), 870 bytes | 870.00 KiB/s, done.
Total 4 (delta 0), reused 0 (delta 0), pack-reused 0 (from 0)
To http://gitea:3000/sarah/story-blog.git
   5dfd972..73886ae  master -> master
```

> **Why:** The rebase made the local history compatible with the updated remote history, so a normal `git push` was sufficient. `origin` identifies the remote repository, and `master` identifies the branch to publish.

### ✅ Step 6: Verify locally

```bash
git log --oneline --decorate master origin/master
git status --short
cat story-index.txt
```

Successful final state:

```text
73886ae (HEAD -> master, origin/master, origin/HEAD) Added the fox and grapes story
5dfd972 Added Index
1eee234 Merge branch 'story/frogs-and-ox'

1. The Lion and the Mouse
2. The Frogs and the Ox
3. The Fox and the Grapes
4. The Donkey and the Dog
```

`git status --short` produced no output, confirming that the working tree was clean.

> **Why:** `git log` confirms that local `master` and `origin/master` point to the same commit. `git status --short` confirms that no unresolved or uncommitted changes remain. `cat` provides the final content check for all four story titles and the corrected spelling.

### 🖥️ Step 7: Verify through Gitea UI

Open the lab's **Gitea UI** button, sign in with a temporary lab account, open `sarah/story-blog`, and select the `master` branch. Open `story-index.txt` and confirm the four titles, including `The Lion and the Mouse`. Capture a screenshot showing the file and the final commit `Added the fox and grapes story`.

The Gitea UI confirmed the corrected file on `master` and the successful commit.

> **Why:** The UI check confirms that the pushed result is visible in the remote repository, not only in Max's local clone. Keep credentials out of screenshots and documentation.

## Best Practices

- **Inspect before changing files.** Confirm the current branch, remote-tracking branch, and working tree state before attempting a push.
- **Use `git pull --rebase` for a linear history.** It incorporates remote commits without creating an unnecessary merge commit.
- **Resolve conflicts by preserving the required final content.** Remove every conflict marker before staging the file.
- **Avoid duplicate commits after conflict resolution.** Git may drop a commit when its patch is already present upstream; this is expected and prevents duplicate changes.
- **Verify both local and remote state.** Confirm matching branch tips, a clean working tree, and the final file contents.
- **Keep credentials out of documentation and screenshots.** Use temporary lab credentials only for the task and never commit them to the repository.

### 📚 Official Documentation

- [Git pull documentation](https://git-scm.com/docs/git-pull)
- [Git rebase documentation](https://git-scm.com/docs/git-rebase)
- [Git merge documentation](https://git-scm.com/docs/git-merge)
- [Git add documentation](https://git-scm.com/docs/git-add)
- [Git commit documentation](https://git-scm.com/docs/git-commit)
- [Git push documentation](https://git-scm.com/docs/git-push)
- [Git status documentation](https://git-scm.com/docs/git-status)
- [Gitea repository documentation](https://docs.gitea.com/next/usage/repository)
