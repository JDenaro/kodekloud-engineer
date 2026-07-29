# Task 13: Restrict Cron Access

In alignment with security compliance standards, the Nautilus project team has opted to impose restrictions on crontab access. Specifically, only designated users will be permitted to create or update cron jobs.

Configure crontab access on App Server 3 as follows: Allow crontab access to `ammar` user while denying access to the `rod` user.

## Task Requirements

1. Configure crontab access on App Server 3.
2. Allow crontab access to user `ammar`.
3. Deny crontab access to user `rod`.

## Solution

The access policy is configured with `/etc/cron.allow` and `/etc/cron.deny`. When `/etc/cron.allow` exists, only users listed in that file are allowed to use `crontab`. The explicit `rod` entry in `/etc/cron.deny` records the requested denial as well.

The primary method below uses `vi` so the files can be edited manually. An alternative using `echo` and `tee` is included afterward for cases where the file should contain exactly one username without opening an editor.

### 🔌 Step 1: Connect to App Server 3

Run the command from the Jump Host:

```bash
ssh banner@stapp03
```

The session opened on App Server 3:

```text
thor@jump-host ~$ ssh banner@stapp03
[banner@stapp03 ~]$
```

> **Why:** `ssh` opens a secure remote shell. `banner` is the login account for App Server 3, and `stapp03` is the target server where the cron access files must be configured.

### ✏️ Step 2: Allow ammar to use crontab with vi

Open the allow list:

```bash
sudo vi /etc/cron.allow
```

Inside `vi`:

1. Press `i` to enter insert mode.
2. Type `ammar` on its own line.
3. Press `Esc` to leave insert mode.
4. Type `:wq` and press `Enter` to save and exit.

The command returned to the shell without an error:

```text
[banner@stapp03 ~]$ sudo vi /etc/cron.allow
[banner@stapp03 ~]$
```

> **Why:** `vi` is a terminal text editor. The `i` key allows text insertion, `Esc` returns to command mode, and `:wq` writes the changes and quits the editor. `/etc/cron.allow` must contain `ammar`, because the existence of this file limits crontab access to users listed in it.

### 🚫 Step 3: Deny rod access to crontab with vi

Open the deny list:

```bash
sudo vi /etc/cron.deny
```

Inside `vi`:

1. Press `i` to enter insert mode.
2. Type `rod` on its own line.
3. Press `Esc` to leave insert mode.
4. Type `:wq` and press `Enter` to save and exit.

The command returned to the shell without an error:

```text
[banner@stapp03 ~]$ sudo vi /etc/cron.deny
[banner@stapp03 ~]$
```

> **Why:** This writes `rod` to the deny list. `cron.deny` rejects crontab access for users listed in it when no allow list exists. In this challenge, `/etc/cron.allow` takes precedence and already restricts access to `ammar`; keeping `rod` in `/etc/cron.deny` also records the requested explicit denial.

### 🔁 Alternative method: Write both files without vi

Instead of opening an editor, the files can be written directly:

```bash
echo ammar | sudo tee /etc/cron.allow
echo rod | sudo tee /etc/cron.deny
```

Expected output:

```text
ammar
rod
```

> **Why:** `echo` sends each username to standard output, and the pipe `|` passes it to `tee`. `tee` writes the username to the specified file and also displays it in the terminal. This method replaces the file contents, ensuring that `/etc/cron.allow` contains only `ammar` and `/etc/cron.deny` contains `rod`.

### ✅ Step 4: Verify the access lists

Display both files:

```bash
sudo cat /etc/cron.allow
sudo cat /etc/cron.deny
```

The lab produced:

```text
[banner@stapp03 ~]$ sudo cat /etc/cron.allow
ammar
[banner@stapp03 ~]$ sudo cat /etc/cron.deny
rod
[banner@stapp03 ~]$
```

> **Why:** `cat` displays the contents of each file. The result confirms that `ammar` is in the allow list and `rod` is in the deny list. Since `/etc/cron.allow` exists, users not listed there are not permitted to use `crontab`.

## Best Practices

- **Use an allow list for least privilege.** Listing only `ammar` in `/etc/cron.allow` prevents unspecified users from creating or updating personal crontabs.
- **Keep the policy explicit.** Recording `rod` in `/etc/cron.deny` makes the requested restriction clear even though the allow list has precedence.
- **Save vi changes deliberately.** Use `Esc`, then `:wq`, and press `Enter` so the edited file is written before leaving the editor.
- **Write exact file contents when appropriate.** The `tee` alternative avoids accidental extra entries or formatting left behind during manual editing.
- **Remember the scope of these files.** `cron.allow` and `cron.deny` control who may use `crontab`; they do not stop cron jobs that already exist from executing.

### 📚 Official Documentation

- [crontab(1) Linux manual page](https://man7.org/linux/man-pages/man1/crontab.1.html)
