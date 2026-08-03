# Day 10: Linux Bash Scripts

The production support team of xFusionCorp Industries is working on developing some bash scripts to automate different day to day tasks. One is to create a bash script for archiving website content files. They have a static website running on App Server 3 in Stratos Datacenter, and they need to create a bash script named `ecommerce_archive.sh` which should accomplish the following tasks. (Also remember to place the script under the `/scripts` directory on App Server 3).

a. Create a zip archive named `xfusioncorp_ecommerce.zip` of `/var/www/html/ecommerce` directory.

b. Save the archive in the `/archives/` directory on the App Server 3. This is a temporary storage, as archives from this location will be cleaned on a weekly basis. Therefore, the archive should also be copied to the Nautilus Storage Server so it can be retrieved later for validation purposes.

c. Copy the created archive to the Nautilus Storage Server server in the `/archives/` location.

d. Please make sure script won't ask for password while copying the archive file. Additionally, the respective server user (for example, `tony` in case of App Server 1) must be able to run it.

e. Do not use `sudo` inside the script.

**Note:**
The `zip` package must be installed on given App Server before executing the script. This package is essential for creating the zip archive of the website files. Install it manually outside the script.

## Specific Requirements:

1. Create `/scripts/ecommerce_archive.sh` on App Server 3.
2. Archive `/var/www/html/ecommerce` as `/archives/xfusioncorp_ecommerce.zip`.
3. Copy the archive to the Nautilus Storage Server at `/archives/`.
4. Configure password-less SSH authentication for the archive transfer.
5. Allow the App Server 3 user `banner` to run the script.
6. Install `zip` manually outside the script.
7. Do not use `sudo` inside the script.

## Solution

The script runs as `banner` on App Server 3 (`stapp03`). The archive is created locally and then copied to the Storage Server (`ststor01`) as user `natasha`.

Because the script must copy the archive without asking for a password, an RSA SSH key was generated for `banner` and its public key was installed for `natasha` on the Storage Server. The `zip` package was installed outside the script as required.

### 🔌 Step 1: Connect to App Server 3

From the Jump Host, connect as `banner`:

```bash
ssh banner@stapp03
```

The session opened on App Server 3:

```text
thor@jump-host ~$ ssh banner@stapp03
[banner@stapp03 ~]$
```

> **Why:** `ssh` opens a secure remote shell. `banner` is the user that must be able to run the script, and `stapp03` is the application server containing the website files.

### 📦 Step 2: Install the zip package outside the script

```bash
sudo yum install -y zip
```

The package was already available and the installation requirement was satisfied:

```text
Package zip-3.0-35.el9.x86_64 is already installed.
Dependencies resolved.
Nothing to do.
Complete!
```

> **Why:** `sudo` provides administrative privileges, `yum` manages packages on the server, `install` requests the `zip` package, and `-y` automatically confirms the package action. The package is installed outside the script because the challenge explicitly prohibits placing installation logic or `sudo` inside the script.

### 🔑 Step 3: Generate an SSH key for banner

```bash
ssh-keygen -t rsa -N "" -f ~/.ssh/id_rsa
```

The key pair was created under `banner`'s home directory:

```text
Created directory '/home/banner/.ssh'.
Your identification has been saved in /home/banner/.ssh/id_rsa
Your public key has been saved in /home/banner/.ssh/id_rsa.pub
The key fingerprint is:
SHA256:+vmpHVexp9kCpBPu96s0kfCdYjHduTC6vzGXNWr46RQ banner@stapp03
```

> **Why:** `ssh-keygen` generates SSH authentication keys. `-t rsa` selects RSA, `-N ""` creates the key without a passphrase so the script can run unattended, and `-f ~/.ssh/id_rsa` stores the private key at the standard path with the public key beside it as `id_rsa.pub`. The private key stays on App Server 3.

### 📤 Step 4: Install the public key on the Storage Server

```bash
ssh-copy-id natasha@ststor01
```

The public key was installed successfully:

```text
/usr/bin/ssh-copy-id: INFO: Source of key(s) to be installed: "/home/banner/.ssh/id_rsa.pub"
Number of key(s) added: 1
```

> **Why:** `ssh-copy-id` logs in to the Storage Server and appends the local public key to `natasha`'s `~/.ssh/authorized_keys` file. The password is required only during this initial setup. Future SSH and `scp` connections from `banner` can use the private key without a password prompt.

### ✅ Step 5: Confirm password-less Storage Server access

```bash
ssh -o PasswordAuthentication=no natasha@ststor01
exit
```

The connection opened without asking for a password:

```text
[banner@stapp03 ~]$ ssh -o PasswordAuthentication=no natasha@ststor01
[natasha@ststor01 ~]$ exit
logout
Connection to ststor01 closed.
[banner@stapp03 ~]$
```

> **Why:** `ssh` attempts the remote login, and `-o PasswordAuthentication=no` prevents password fallback. A successful connection therefore proves that public-key authentication works. `exit` closes the temporary remote session.

### 📝 Step 6: Create the archive script

Create `/scripts/ecommerce_archive.sh` with this content:

```bash
#!/bin/bash

zip -r /archives/xfusioncorp_ecommerce.zip /var/www/html/ecommerce
scp /archives/xfusioncorp_ecommerce.zip natasha@ststor01:/archives/
```

> **Why:** The shebang `#!/bin/bash` tells the operating system to execute the file with Bash. `zip -r` recursively includes the website directory and its contents in `/archives/xfusioncorp_ecommerce.zip`. `scp` securely copies that local archive to `/archives/` on `ststor01` as `natasha`. The script contains no `sudo`; it relies on `banner`'s file permissions and the configured SSH key.

### ▶️ Step 7: Make the script executable for banner

```bash
chmod +x /scripts/ecommerce_archive.sh
```

The script was executable by `banner`:

```text
-rwxr-xr-x 1 banner banner 135 Jul 31 01:23 /scripts/ecommerce_archive.sh
```

> **Why:** `chmod` changes file permissions, and `+x` adds execute permission. The resulting mode allows the owner `banner` to read, write, and execute the script. The script can therefore be run directly without `sudo`.

### ✅ Step 8: Run the archive script

```bash
/scripts/ecommerce_archive.sh
```

The script created the ZIP archive and transferred it without a password prompt:

```text
  adding: var/www/html/ecommerce/ (stored 0%)
  adding: var/www/html/ecommerce/.gitkeep (stored 0%)
  adding: var/www/html/ecommerce/index.html (stored 0%)
xfusioncorp_ecommerce.zip                      100%  623     1.1MB/s   00:00
[banner@stapp03 ~]$
```

> **Why:** The `adding:` lines show that the website directory and its files were included in the archive. The `scp` progress line reaching `100%` confirms that the archive was copied to `natasha@ststor01:/archives/`. Since no password prompt appeared during execution, the SSH key configuration worked as required.

## Best Practices

- **Install dependencies outside automation scripts.** The `zip` package was installed manually, keeping package-management privileges out of the script.
- **Use password-less keys for unattended transfers.** Public-key authentication allows the scheduled script to copy archives without interactive input.
- **Keep private keys on the source host.** Only `id_rsa.pub` belongs on the Storage Server.
- **Use recursive archiving for directories.** The `-r` option ensures the website directory and its contents are included.
- **Make scripts executable for the intended user.** `banner` owns and can execute `/scripts/ecommerce_archive.sh` without `sudo`.
- **Retain a durable copy remotely.** The local `/archives/` directory is temporary, so copying the archive to the Storage Server preserves it for later validation.
- **Avoid `0.0.0.0`-style broad access for SSH keys.** Restrict authorized keys and remote accounts to the exact automation scope in production.

### 📚 Official Documentation

- [Info-ZIP project](https://infozip.sourceforge.net/)
- [OpenSSH manual pages](https://www.openssh.org/manual.html)
- [ssh-keygen(1) manual page](https://man.openbsd.org/ssh-keygen)
- [ssh-copy-id(1) Linux manual page](https://man7.org/linux/man-pages/man1/ssh-copy-id.1.html)
- [scp(1) OpenBSD manual page](https://man.openbsd.org/scp)
