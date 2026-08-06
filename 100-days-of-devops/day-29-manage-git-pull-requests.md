# Day 29: Manage Git Pull Requests

Max want to push some new changes to one of the repositories but we don't want people to push directly to master branch, since that would be the final version of the code. It should always only have content that has been reviewed and approved. We cannot just allow everyone to directly push to the master branch. So, let's do it the right way as discussed below:

SSH into storage server using user max. Use the temporary password supplied by the lab. There you can find an already cloned repo under Max user's home.

Max has written his story about The 🦊 Fox and Grapes 🍇

Max has already pushed his story to remote git repository hosted on Gitea branch story/fox-and-grapes

Check the contents of the cloned repository. Confirm that you can see Sarah's story and history of commits by running git log and validate author info, commit message etc.

Max has pushed his story, but his story is still not in the master branch. Let's create a Pull Request(PR) to merge Max's story/fox-and-grapes branch into the master branch

Click on the Gitea UI button on the top bar. You should be able to access the Gitea page.

UI login info:

- Username: max

- Use the temporary password supplied by the lab.

PR title : Added fox-and-grapes story

PR pull from branch: story/fox-and-grapes (source)

PR merge into branch: master (destination)

Before we can add our story to the master branch, it has to be reviewed. So, let's ask tom to review our PR by assigning him as a reviewer

Add tom as reviewer through the Git Portal UI

Go to the newly created PR

Click on Reviewers on the right

Add tom as a reviewer to the PR

Now let's review and approve the PR as user Tom

Login to the portal with the user tom

Logout of Git Portal UI if logged in as max

UI login info:

- Username: tom

- Use the temporary password supplied by the lab.

PR title : Added fox-and-grapes story

Review and merge it.

Great stuff!! The story has been merged! 👏

Note: For these kind of scenarios requiring changes to be done in a web UI, please take screenshots so that you can share it with us for review in case your task is marked incomplete. You may also consider using a screen recording software such as loom.com to record and share your work.

## Specific Requirements:

1. SSH into storage server using user `max` with the password supplied by the lab. There you can find an already cloned repo under Max user's home.
2. Check the contents of the cloned repository. Confirm that you can see Sarah's story and history of commits by running `git log` and validate author info, commit message etc.
3. Create a Pull Request titled `Added fox-and-grapes story` from `story/fox-and-grapes` into `master`.
4. Add `tom` as reviewer through the Git Portal UI.
5. Log in to the portal with the user `tom`, approve the Pull Request, and merge it.
6. Capture screenshots of the UI workflow for review.

## Solution

This task uses a Pull Request to protect the final `master` branch. Max's branch is reviewed by Tom before it is merged. Temporary passwords are intentionally not stored in this guide, and the credential-bearing remote URL is redacted in the recorded output.

### 🔌 Step 1: Connect to the Storage Server

```bash
ssh max@ststor01
ls
cd story-blog
ls
```

Successful repository contents:

```text
story-blog
fox-and-grapes.txt  frogs-and-ox.txt  lion-and-mouse.txt
```

> **Why:** `ssh` opens a secure shell on the Storage Server as `max`. `ls` lists the home directory and then the repository contents. The files confirm that Max's story and Sarah's existing stories are present.

### 📜 Step 2: Inspect the commit history

```bash
git log
```

Relevant output from the lab:

```text
commit eccf8e03c8b059f6b21a52182f57e9ad7a76f445 (HEAD -> story/fox-and-grapes, origin/story/fox-and-grapes)
Author: Max <max@stratos.xfusioncorp.com>
Date:   Wed Aug 5 23:28:20 2026 +0000

    Added fox-and-grapes story

commit 39c34759250c54b849b4a077895ac1aeb9f2446a (origin/master, origin/HEAD, master)
Merge: ea307f2 66e1000
Author: sarah <sarah@stratos.xfusioncorp.com>
Date:   Wed Aug 5 23:28:18 2026 +0000

    Merge branch 'story/frogs-and-ox'
```

The remaining history showed Sarah's commits, including `Fix typo in story title`, `Completed frogs-and-ox story`, `Added the lion and mouse story`, and `Add incomplete frogs-and-ox story`.

> **Why:** `git log` displays the repository history, including commit hashes, branch references, authors, dates, and messages. This confirms that Max's story is on `story/fox-and-grapes` and that `master` still points to Sarah's history.

### 🔗 Step 3: Confirm the Gitea repository

```bash
git remote -v
```

The remote was the `sarah/story-blog` repository on the Gitea server. The password embedded in the temporary lab URL is intentionally omitted here.

> **Why:** `git remote -v` displays the repository's fetch and push destinations. It confirms which Gitea repository must be opened in the UI without exposing the credential-bearing URL in documentation.

### 🌐 Step 4: Open Gitea and create the Pull Request

Use the lab's **Gitea UI** button and sign in as `max` with the temporary password provided by the lab.

Create the Pull Request with these values:

```text
Repository: sarah/story-blog
Title: Added fox-and-grapes story
Source branch: story/fox-and-grapes
Destination branch: master
```

The created Pull Request was `Added fox-and-grapes story #1`.

> **Why:** A Pull Request provides a review boundary between a working branch and the protected final branch. Selecting `story/fox-and-grapes` as the source and `master` as the destination ensures that only the intended story is proposed for integration.

### 👀 Step 5: Assign Tom as reviewer

In the new Pull Request:

1. Open **Reviewers** in the right-hand sidebar.
2. Select `tom`.
3. Confirm that Tom appears in the reviewer list.

Capture a screenshot showing the Pull Request and Tom assigned as reviewer.

> **Why:** Assigning a reviewer establishes the required approval step before the change can be merged. The screenshot provides evidence that the correct reviewer was assigned through the UI.

### ✅ Step 6: Review and approve as Tom

1. Log out of Gitea as `max`.
2. Log in as `tom` with the temporary password supplied by the lab.
3. Open `Added fox-and-grapes story #1`.
4. Select **Review**, then choose **Approve**.

The UI showed that Tom approved the changes.

> **Why:** Reviewing as a separate user demonstrates that the Pull Request was approved by the assigned reviewer rather than by the author who created it.

### 🔀 Step 7: Merge the approved Pull Request

From the approved Pull Request, choose **Create merge commit** and confirm the merge. The final UI state showed:

```text
Merged
tom merged 1 commits from story/fox-and-grapes into master
Pull request successfully merged and closed
```

Capture a screenshot showing the merged state and the source and destination branches.

> **Why:** Merging the approved Pull Request integrates Max's story into `master` while preserving the review history. The closed and merged state confirms that the requested change reached the final branch.

## Best Practices

- **Protect the final branch.** Require Pull Requests and review approval instead of allowing direct pushes to `master`.
- **Review the source branch before creating the PR.** Check the author, commit message, and branch references so the correct change is proposed.
- **Use a separate reviewer account.** Independent approval provides stronger review evidence than self-approval.
- **Keep credentials out of repository documentation.** Use temporary lab credentials only in the lab UI and never copy them into remote URLs, logs, or committed files.
- **Keep screenshots outside the repository.** Save UI evidence for platform review without adding screenshots or other generated artifacts to the guide repository.

### 📚 Official Documentation

- [Gitea pull requests](https://docs.gitea.com/next/usage/issues-prs/pull-request/)
- [Git log documentation](https://git-scm.com/docs/git-log)
- [Git remote documentation](https://git-scm.com/docs/git-remote)
