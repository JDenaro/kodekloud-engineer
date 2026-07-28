# Task 06: Linux User Data Transfer

Due to an accidental data mix-up, user data was unintentionally mingled on Nautilus App Server 2 at the `/home/usersdata` location by the Nautilus production support team in Stratos DC. To rectify this, specific user data needs to be filtered and relocated. Here are the details:

Locate all files (excluding directories) owned by user `rose` within the `/home/usersdata` directory on App Server 2. Copy these files while preserving the directory structure to the `/beta` directory.

## Task Requirements

1. Work on App Server 2.
2. Search recursively under `/home/usersdata`.
3. Select only regular files owned by `rose`, excluding directories.
4. Copy the selected files to `/beta` while preserving their directory structure.

## Solution

The `find` command can select filesystem entries by both type and owner. Its `-exec` action can then pass every matching file to `cp`. GNU `cp --parents` recreates the source path beneath the destination directory, which preserves the required hierarchy.

The lab contained a large number of matching files, so only a representative portion of the verification output is included below.

### 🔌 Step 1: Connect to App Server 2

From the jump host, connect to App Server 2 as `steve`:

```bash
ssh steve@stapp02
```

The connection opens a shell on `stapp02`:

```text
thor@jump-host ~$ ssh steve@stapp02
steve@stapp02's password:
[steve@stapp02 ~]$
```

> **Why:** `ssh` opens a secure remote shell session. `steve` is the login account for App Server 2, and `stapp02` is the target hostname. The search and copy must run on the server that contains `/home/usersdata` and `/beta`.

### 🔎 Step 2: Locate the files owned by rose

Search for the required files:

```bash
sudo find /home/usersdata -type f -user rose
```

The command prints every matching source path. The output was too large to retain in full during the lab.

> **Why:** `sudo` supplies the privileges needed to traverse all relevant directories. `find` recursively searches a directory hierarchy, beginning here at `/home/usersdata`. The `-type f` test selects only regular files, which excludes directories. The `-user rose` test limits the results to files owned by `rose`. With no explicit action, `find` prints every path that satisfies both tests.

### 📁 Step 3: Copy the matching files with their directory structure

Run the same search and execute `cp` for each result:

```bash
sudo find /home/usersdata -type f -user rose -exec cp --parents {} /beta \;
```

The copy operation completed without printing an error.

> **Why:** `-exec` tells `find` to run a command for each matching file. `cp` copies that file, and `--parents` reproduces its source path beneath `/beta` instead of placing every file directly in the destination root. The `{}` placeholder is replaced with the current path found by `find`. `/beta` is the destination, and `\;` terminates the `-exec` action; the backslash prevents the shell from treating the semicolon as its own command separator. Because the source paths are absolute, a source such as `/home/usersdata/wp-includes/load.php` becomes `/beta/home/usersdata/wp-includes/load.php`.

### ✅ Step 4: Verify the copied files

List the regular files under the destination:

```bash
sudo find /beta -type f
```

The command produced many lines. This excerpt shows that the original hierarchy was recreated below `/beta`:

```text
/beta/home/usersdata/wp-includes/Requests/src/Exception/Http/Status411.php
/beta/home/usersdata/wp-includes/Requests/src/Exception/Http/Status410.php
/beta/home/usersdata/wp-includes/Requests/src/Exception/Http/Status402.php
/beta/home/usersdata/wp-includes/Requests/src/Exception/Http/Status431.php
/beta/home/usersdata/wp-includes/Requests/src/Exception/Http/Status415.php
/beta/home/usersdata/wp-includes/Requests/src/Exception/Http/Status412.php
/beta/home/usersdata/wp-includes/Requests/src/Exception/Http/Status511.php
/beta/home/usersdata/wp-includes/Requests/src/Exception/Http/Status405.php
/beta/home/usersdata/wp-includes/Requests/src/Exception/Http/Status400.php
/beta/home/usersdata/wp-includes/Requests/src/Exception/Http/Status500.php
...
/beta/home/usersdata/wp-includes/SimplePie/src/Cache/Memcache.php
/beta/home/usersdata/wp-includes/SimplePie/src/Copyright.php
/beta/home/usersdata/wp-includes/SimplePie/src/Cache.php
```

> **Why:** This `find` command starts at `/beta` and uses `-type f` to print only copied regular files. Paths beginning with `/beta/home/usersdata/` demonstrate that `cp --parents` preserved the source hierarchy rather than flattening all files into one directory. A complete listing is unnecessary when the output is very large; a representative sample is sufficient to understand the resulting structure.

## Best Practices

- **Filter by both type and owner.** Combining `-type f` and `-user rose` prevents directories and other users' files from being copied.
- **Inspect the matches before copying.** Running the search by itself makes the selection criteria visible before the copy action is added.
- **Preserve hierarchy explicitly.** `cp --parents` avoids filename collisions and retains the context provided by the original directory tree.
- **Keep destination verification focused.** When thousands of files are involved, a representative path sample can confirm the structure without storing an impractically large console transcript.
- **Do not confuse structure with metadata.** `--parents` preserves path hierarchy; it does not request preservation of ownership, permissions, or timestamps because the challenge only requires the directory structure.

### 📚 Official Documentation

- [find(1) Linux manual page](https://man7.org/linux/man-pages/man1/find.1.html)
- [cp(1) Linux manual page](https://man7.org/linux/man-pages/man1/cp.1.html)
