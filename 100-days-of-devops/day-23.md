# Day 23: Fork a Git Repository

There is a Git server utilized by the Nautilus project teams. Recently, a new developer named Jon joined the team and needs to begin working on a project. To begin, he must fork an existing Git repository. Follow the steps below:

## Specific Requirements:

1. Click on the Gitea UI button located on the top bar to access the Gitea page.
2. Login to Gitea server using username jon and password Jon_pass123.
3. Once logged in, locate the Git repository named sarah/story-blog and fork it under the jon user.
4. Note: For tasks requiring web UI changes, screenshots are necessary for review purposes. Additionally, consider utilizing screen recording software such as loom.com to record and share your task completion process.

## Solution

This task is completed entirely through the Gitea web UI. Its layout and fork workflow are very similar to GitHub: sign in, open the source repository, choose **Fork**, select the destination owner, and confirm the new copy.

### 🌐 Step 1: Open the Gitea UI

Use the lab's top bar to open **Gitea UI**:

```text
Open the Gitea UI button from the top bar.
```

> **Why:** The Gitea UI is the web interface for the Git server. This challenge requires a server-side fork, so the operation must be performed in the provided web interface rather than with local Git commands.

### 🔐 Step 2: Sign in as Jon

Enter the credentials provided by the challenge:

```text
Username: jon
Password: Jon_pass123
```

> **Why:** Signing in as `jon` ensures that the fork is created under Jon's account and that he becomes the owner of the new repository.

### 🍴 Step 3: Fork the repository

In the Gitea interface, open the repository and create the fork:

```text
Source repository: sarah/story-blog
Fork owner: jon
```

Choose **Fork**, select `jon` as the owner if prompted, and confirm the operation. The resulting repository should be owned by `jon` and retain `story-blog` as its repository name.

> **Why:** A fork is an independent server-side copy of a repository that keeps a relationship with its source. Forking `sarah/story-blog` under `jon` gives Jon his own repository space for development without changing Sarah's original repository.

### ✅ Step 4: Capture the result for review

After the fork is created, capture a screenshot showing the resulting repository page:

```text
Expected UI state:
Owner: jon
Repository: story-blog
Source: sarah/story-blog
```

> **Why:** The screenshot provides review evidence that the fork was created under the correct account and from the correct source repository. A screen recording can also be shared when the lab review process requires a complete activity record.

## Best Practices

- **Use the web UI for UI-based tasks.** Follow the lab's Gitea interface instead of making unrequested changes through the command line.
- **Verify the fork owner.** Confirm that the new repository belongs to `jon`, not to the source owner or another account.
- **Preserve the source repository.** Forking creates a separate copy and should not modify `sarah/story-blog`.
- **Keep review evidence.** Capture the final repository page with the owner and source relationship visible.

### 📚 Official Documentation

- [Gitea: Fork a repository](https://docs.gitea.com/usage/pull-request)
- [GitHub: About permissions and visibility of forks](https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/working-with-forks/about-permissions-and-visibility-of-forks)
