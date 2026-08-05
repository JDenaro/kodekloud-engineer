# Day 22: Clone Git Repository on Storage Server

The DevOps team established a new Git repository last week, which remains unused at present. However, the Nautilus application development team now requires a copy of this repository on the Storage Server in the Stratos DC. Follow the provided details to clone the repository:

## Specific Requirements:

1. The repository to be cloned is located at /opt/beta.git
2. Clone this Git repository to the /usr/src/kodekloudrepos directory. Perform this task using the natasha user, and ensure that no modifications are made to the repository or existing directories, such as changing permissions or making unauthorized alterations.

## Solution

The challenge names `/usr/src/kodekloudrepos` as the destination directory, but the lab validator expects the cloned repository inside a child directory named after the repository. Because the source is `/opt/beta.git`, the successful destination is `/usr/src/kodekloudrepos/beta`. `git clone` creates this `beta` directory automatically.

### 🔌 Step 1: Connect to the Storage Server

From the Jump Host, connect using the required `natasha` user:

```bash
ssh natasha@ststor01
```

> **Why:** `ssh` opens a secure remote shell. `natasha` is the account required for this task, and `ststor01` is the Storage Server where the repository must be cloned.

### 📥 Step 2: Clone the repository into the named subdirectory

Clone the local repository into a `beta` subdirectory under `/usr/src/kodekloudrepos`:

```bash
git clone /opt/beta.git /usr/src/kodekloudrepos/beta
```

The successful lab produced:

```text
Cloning into '/usr/src/kodekloudrepos/beta'...
warning: You appear to have cloned an empty repository.
done.
```

> **Why:** `git clone` creates a complete working copy, including the Git metadata and the `origin` remote. `/opt/beta.git` is the local source repository, while `/usr/src/kodekloudrepos/beta` is the exact destination expected by the lab validator. Git creates the `beta` directory as part of the clone, so no manual directory creation, permission change, or `sudo` command is needed. The empty-repository warning is expected when the source has no commits yet; it does not mean that cloning failed.

### ✅ Step 3: Verify the clone

Confirm that the destination is a working-tree repository and that its origin points to the requested source:

```bash
git -C /usr/src/kodekloudrepos/beta rev-parse --is-inside-work-tree
git -C /usr/src/kodekloudrepos/beta remote get-url origin
```

Expected output:

```text
true
/opt/beta.git
```

> **Why:** The `-C` option runs each Git command from the cloned repository directory. `rev-parse --is-inside-work-tree` confirms that Git recognizes the destination as a normal working copy. `remote get-url origin` displays the source configured as the `origin` remote. These read-only checks do not modify the repository or its permissions.

## Best Practices

- **Use the repository-named subdirectory.** The parent path is `/usr/src/kodekloudrepos`, but this lab expects the clone at `/usr/src/kodekloudrepos/beta`.
- **Run the clone as the requested user.** Using `natasha` avoids unauthorized ownership or permission changes.
- **Do not modify the source repository.** A clone operation reads from `/opt/beta.git` and creates an independent working copy.
- **Treat an empty-repository warning correctly.** It indicates that the source has no commits yet; the clone can still be valid.

### 📚 Official Documentation

- [Git `clone` documentation](https://git-scm.com/docs/git-clone)
- [Git Basics: Getting a Git Repository](https://git-scm.com/book/en/v2/Git-Basics-Getting-a-Git-Repository)
