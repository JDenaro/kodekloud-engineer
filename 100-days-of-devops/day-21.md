# Day 21: Set Up Git Repository on Storage Server

The Nautilus development team has provided requirements for a new application development project, specifically requesting the establishment of a Git repository. Follow the instructions below to create the Git repository on the Storage server in the Stratos DC:

## Specific Requirements:

1. Utilize yum to install the git package on the Storage Server.
2. Create a bare repository named /opt/official.git (ensure exact name usage).

## Solution

The repository is created as a bare repository because the Storage Server will act as a central Git remote. A bare repository stores Git history and references but does not contain a working tree for editing files directly.

### 🔌 Step 1: Connect to the Storage Server

From the Jump Host, connect to the Storage Server:

```bash
ssh natasha@ststor01
```

> **Why:** `ssh` opens a secure remote shell. `natasha` is the Storage Server login account and `ststor01` identifies the target server where the repository must be created.

### 📦 Step 2: Install Git

Install the Git package with `yum`:

```bash
sudo yum install -y git
```

> **Why:** `sudo` runs the package installation with administrative privileges. `yum install` downloads and installs the requested package and its dependencies. The `-y` option automatically confirms the package manager prompts. On this Enterprise Linux system, `yum` provides the expected compatibility interface for the DNF package manager.

### 🗃️ Step 3: Create the bare repository

Create the repository using the exact required path:

```bash
sudo git init --bare /opt/official.git
```

Expected output:

```text
Initialized empty Git repository in /opt/official.git/
```

> **Why:** `git init` initializes a new Git repository. The `--bare` option creates a repository without a working tree, which is the appropriate format for a shared central remote. `/opt/official.git` is an absolute path and preserves the exact repository name required by the challenge. `sudo` is needed because `/opt` is an administrative system location.

### ✅ Step 4: Verify

Confirm that Git recognizes the repository as bare:

```bash
sudo git -C /opt/official.git rev-parse --is-bare-repository
```

Expected output:

```text
true
```

> **Why:** The `-C` option makes Git run as if it had first changed to `/opt/official.git`. `rev-parse --is-bare-repository` asks Git whether the target repository is bare. The value `true` confirms that the central repository was initialized in the required format.

## Best Practices

- **Use a bare repository for shared remotes.** It avoids a working tree on the server and lets developers push and fetch safely.
- **Use the exact absolute path.** `/opt/official.git` is the required repository location and name.
- **Keep administrative locations protected.** Use elevated privileges only for creating or managing repositories under `/opt`.

### 📚 Official Documentation

- [Git `init` documentation](https://git-scm.com/docs/git-init)
- [Git on the Server: Setting Up the Server](https://git-scm.com/book/en/v2/Git-on-the-Server-Setting-Up-the-Server)
- [Red Hat Enterprise Linux 9: Installing packages with DNF](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/managing_software_with_the_dnf_tool/assembly_installing-rhel-9-content_managing-software-with-the-dnf-tool)
