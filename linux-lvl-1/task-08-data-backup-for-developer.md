# Task 08: Data Backup for Developer

The jump host server hosts a directory named `/data`, serving as a repository for various developers non-confidential data. Developer `james` has requested a copy of their data stored in `/data/james`. The System Admin team has provided the following steps to fulfill this request:

a. Create a compressed archive named `james.tar.gz` of the `/data/james` directory.

b. Transfer the archive to the `/home` directory on the Jump Host Server.

## Task Requirements

1. Create a gzip-compressed archive of `/data/james`.
2. Name the archive `james.tar.gz`.
3. Place the archive in `/home` on the Jump Host Server.

## Solution

Both the source directory and destination are on the Jump Host, so no SSH connection or network transfer is required. The archive can be created directly as `/home/james.tar.gz`, satisfying the creation and destination requirements in one operation.

Using `tar` with `-C /data` stores the directory as `james/` inside the archive rather than recording an absolute source path.

### 📦 Step 1: Create the compressed archive

Run the command directly from `thor@jump-host`:

```bash
sudo tar -czf /home/james.tar.gz -C /data james
```

The command returned to the prompt without an error:

```text
thor@jump-host ~$ sudo tar -czf /home/james.tar.gz -C /data james
thor@jump-host ~$
```

> **Why:** `sudo` provides permission to read all required source files and create an archive directly under `/home`. `tar` combines files and directories into an archive. The `-c` option creates a new archive, `-z` compresses it with gzip, and `-f` tells `tar` that the next argument, `/home/james.tar.gz`, is the archive filename. The `.tar.gz` extension communicates that this is a tar archive compressed with gzip. `-C /data` changes `tar`'s working directory to `/data` before collecting content, and the final `james` argument selects `/data/james`. This causes the archive members to begin with `james/` and avoids storing an absolute path.

The absence of verbose output is expected because the `-v` option was not used. Avoiding verbose mode keeps the console readable when a directory contains many files.

### ✅ Step 2: Verify the archive in /home

Display the resulting file:

```bash
sudo ls -lh /home/james.tar.gz
```

The command returned:

```text
thor@jump-host ~$ sudo ls -lh /home/james.tar.gz
-rw-r--r-- 1 root root 184 Jul 28 03:09 /home/james.tar.gz
thor@jump-host ~$
```

An additional listing of `/home` also showed the archive in the requested destination:

```bash
sudo ls -lh /home
```

```text
total 16K
drwx------ 2 ansible ansible 4.0K Jun 10 09:18 ansible
-rw-r--r-- 1 root    root     184 Jul 28 03:09 james.tar.gz
drwx------ 1 thor    thor    4.0K Jun 10 09:19 thor
```

> **Why:** `ls` displays filesystem entries. The `-l` option uses a long listing that includes permissions, ownership, size, and timestamp, while `-h` formats sizes in human-readable units. The first command verifies the exact archive path. The second confirms that `james.tar.gz` is located directly inside `/home`. The archive size may vary with its contents; the important result is that `/home/james.tar.gz` exists.

## Best Practices

- **Create the archive at its final destination.** Writing directly to `/home/james.tar.gz` avoids an unnecessary intermediate file and move operation.
- **Store relative archive paths.** Using `-C /data` keeps the archive rooted at `james/`, making future extraction cleaner and more predictable.
- **Use the expected filename extension.** `.tar.gz` clearly identifies a tar archive compressed with gzip.
- **Avoid unnecessary verbose output.** Omitting `-v` prevents large directory trees from producing an excessive console transcript.
- **Verify the exact destination.** Listing `/home/james.tar.gz` confirms both the required name and location.

### 📚 Official Documentation

- [tar(1) Linux manual page](https://man7.org/linux/man-pages/man1/tar.1.html)
- [ls(1) Linux manual page](https://man7.org/linux/man-pages/man1/ls.1.html)
