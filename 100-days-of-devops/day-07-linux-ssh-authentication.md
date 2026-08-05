# Day 07: Linux SSH Authentication

The system admins team of xFusionCorp Industries has set up some scripts on jump host that run on regular intervals and perform operations on all app servers in Stratos Datacenter. To make these scripts work properly we need to make sure the thor user on jump host has password-less SSH access to all app servers through their respective sudo users (i.e tony for app server 1). Based on the requirements, perform the following:

Set up a password-less authentication from user `thor` on jump host to all app servers through their respective sudo users.

## Specific Requirements:

1. Configure password-less SSH authentication for user `thor` on the Jump Host.
2. Grant access to App Server 1 through user `tony`.
3. Grant access to App Server 2 through user `steve`.
4. Grant access to App Server 3 through user `banner`.

## Solution

SSH public-key authentication uses a private key on the client and a matching public key authorized on the remote server. The private key remains on the Jump Host, while its public key is copied to each remote user's `authorized_keys` file.

The key was generated without a passphrase because automated scripts must connect without interactive input. The same `thor` key was installed for all three application-server accounts.

### 🔑 Step 1: Generate an RSA key pair for thor

Run this command as `thor` on the Jump Host:

```bash
ssh-keygen -t rsa -N "" -f ~/.ssh/id_rsa
```

The key pair was created successfully:

```text
Generating public/private rsa key pair.
Created directory '/home/thor/.ssh'.
Your identification has been saved in /home/thor/.ssh/id_rsa
Your public key has been saved in /home/thor/.ssh/id_rsa.pub
The key fingerprint is:
SHA256:y6BGnCp4zUgsHigqU9KgDC1EdFtBsBk6HlfLlly9Ygo thor@jump-host
```

> **Why:** `ssh-keygen` generates and manages SSH authentication keys. `-t rsa` selects the RSA key type, `-N ""` sets an empty passphrase so scheduled scripts do not pause for user input, and `-f ~/.ssh/id_rsa` sets the private-key path. The public key is saved beside it as `/home/thor/.ssh/id_rsa.pub`. The private key must remain on the Jump Host and must not be copied to the application servers.

### 📤 Step 2: Install thor's public key on App Server 1

```bash
ssh-copy-id tony@stapp01
```

The key was installed:

```text
/usr/bin/ssh-copy-id: INFO: Source of key(s) to be installed: "/home/thor/.ssh/id_rsa.pub"
Number of key(s) added: 1
```

> **Why:** `ssh-copy-id` logs in to the remote host and adds the local public key to the remote user's authorized-key file. `tony@stapp01` identifies user `tony` on App Server 1. The password is used only for this initial key installation; later SSH sessions use the key pair.

### 📤 Step 3: Install thor's public key on App Server 2

```bash
ssh-copy-id steve@stapp02
```

The key was installed:

```text
/usr/bin/ssh-copy-id: INFO: Source of key(s) to be installed: "/home/thor/.ssh/id_rsa.pub"
Number of key(s) added: 1
```

> **Why:** This copies the same public key to the `steve` account on `stapp02`. The account-specific destination is important because the SSH server checks the `authorized_keys` file belonging to the account being used for login.

### 📤 Step 4: Install thor's public key on App Server 3

```bash
ssh-copy-id banner@stapp03
```

The key was installed:

```text
/usr/bin/ssh-copy-id: INFO: Source of key(s) to be installed: "/home/thor/.ssh/id_rsa.pub"
Number of key(s) added: 1
```

> **Why:** This completes the public-key installation for user `banner` on `stapp03`. All three sudo users now have the same `thor` public key authorized for SSH access.

### ✅ Step 5: Verify password-less access

Test each connection while explicitly disabling password authentication:

```bash
ssh -o PasswordAuthentication=no tony@stapp01
exit
ssh -o PasswordAuthentication=no steve@stapp02
exit
ssh -o PasswordAuthentication=no banner@stapp03
exit
```

The connections opened without requesting a password:

```text
thor@jump-host ~$ ssh -o PasswordAuthentication=no tony@stapp01
[tony@stapp01 ~]$ exit
logout
Connection to stapp01 closed.
thor@jump-host ~$ ssh -o PasswordAuthentication=no steve@stapp02
[steve@stapp02 ~]$ exit
logout
Connection to stapp02 closed.
thor@jump-host ~$ ssh -o PasswordAuthentication=no banner@stapp03
[banner@stapp03 ~]$ exit
logout
Connection to stapp03 closed.
```

> **Why:** `ssh` opens the remote session. The `-o` option supplies a connection setting, and `PasswordAuthentication=no` prevents SSH from falling back to password authentication. Therefore, each successful connection proves that the public-key authentication path works. `exit` closes the remote session and returns to the Jump Host.

## Best Practices

- **Keep private keys private.** Only the public key should be copied to remote servers; `/home/thor/.ssh/id_rsa` must remain on the Jump Host.
- **Use a passphrase for interactive keys.** The empty passphrase is appropriate for this lab's unattended scripts but is less secure for a human-operated production account.
- **Install keys for the correct remote users.** `tony`, `steve`, and `banner` are separate accounts, so each must receive the public key in its own home directory.
- **Test without password fallback.** `PasswordAuthentication=no` confirms that the connection is genuinely using public-key authentication.
- **Protect the `.ssh` directory.** SSH expects private keys and authorization files to have restrictive ownership and permissions.
- **Use least privilege.** Password-less SSH authentication grants login access as the specified account; it does not automatically grant root privileges.

### 📚 Official Documentation

- [OpenSSH manual pages](https://www.openssh.org/manual.html)
- [ssh-keygen(1) manual page](https://man.openbsd.org/ssh-keygen)
- [ssh(1) manual page](https://man.openbsd.org/ssh)
- [ssh-copy-id(1) manual page](https://man7.org/linux/man-pages/man1/ssh-copy-id.1.html)
