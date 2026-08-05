# Task 12: Secure Data Transfer

A Nautilus developer has stored confidential data on the jump host within Stratos DC. To ensure security and compliance, this data must be transferred to one of the app servers. Given developers lack direct access to these servers, the system admin team has been enlisted for assistance.

Copy `/tmp/nautilus.txt.gpg` file from jump server to App Server 2 placing it in the directory `/home/code`.

## Task Requirements

1. Copy `/tmp/nautilus.txt.gpg` from the Jump Host.
2. Transfer it to App Server 2.
3. Place the copied file directly inside `/home/code`.
4. Use the App Server 2 login account `steve`.

## Solution

The destination directory allowed `steve` to write the file directly, so the transfer required only one `scp` command. No temporary remote location, separate SSH session, or privileged move was necessary.

`scp` transfers the file through an SSH-secured connection and uses the same remote authentication mechanism as an interactive SSH login.

### 🔐 Step 1: Copy the encrypted file directly to App Server 2

Run the command from `thor@jump-host`:

```bash
scp /tmp/nautilus.txt.gpg steve@stapp02:/home/code/
```

The command authenticated to App Server 2 and completed the transfer:

```text
thor@jump-host ~$ scp /tmp/nautilus.txt.gpg steve@stapp02:/home/code/
The authenticity of host 'stapp02 (10.244.81.25)' can't be established.
ED25519 key fingerprint is SHA256:9kVEdG6YLssI8HITuXe9k97ogsSZ3+HhnELaF3xETQA.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added 'stapp02' (ED25519) to the list of known hosts.
steve@stapp02's password:
nautilus.txt.gpg  100%  105  294.7KB/s  00:00
thor@jump-host ~$
```

> **Why:** `scp` securely copies files between hosts over an SSH connection. `/tmp/nautilus.txt.gpg` is the local source file on the Jump Host. The remote destination follows the format `user@host:path`: `steve` is the App Server 2 login account, `stapp02` is its hostname, and `/home/code/` is the destination directory. The trailing slash identifies the destination as a directory, so the original filename `nautilus.txt.gpg` is retained. Because the file is already GPG-encrypted and `scp` transports its bytes without transforming the contents, the encrypted data remains intact.

The first connection displayed the server's ED25519 host-key fingerprint. Answering `yes` accepted that key and stored it in the Jump Host's known-hosts file, allowing SSH to identify the same server during later connections.

### ✅ Step 2: Confirm the transfer result

The `scp` progress line provides the completion evidence:

```text
nautilus.txt.gpg  100%  105  294.7KB/s  00:00
```

> **Why:** `100%` indicates that the entire 105-byte file was sent to the remote destination. The displayed transfer rate and elapsed time are informational and can vary between lab runs. Returning to the `thor@jump-host` prompt without an error confirms that `scp` completed successfully. Since the KodeKloud lab also validated successfully, no additional remote move or copy operation was required.

## Best Practices

- **Use encrypted transport for confidential files.** `scp` uses an SSH-secured connection rather than sending data over an unprotected channel.
- **Copy directly when destination permissions allow it.** A direct destination avoids unnecessary temporary copies and privileged operations.
- **Review host fingerprints on first connection.** The host-key prompt helps establish the identity used for future SSH connections.
- **Use an explicit absolute destination path.** `/home/code/` clearly identifies the required location on App Server 2.
- **Preserve the encrypted artifact.** Transferring the `.gpg` file directly avoids decrypting or otherwise transforming confidential content.
- **Check the transfer progress.** The `100%` indicator confirms that the complete source file was transmitted.

### 📚 Official Documentation

- [OpenSSH scp(1) manual page](https://man.openbsd.org/scp)
- [OpenSSH ssh(1) manual page](https://man.openbsd.org/ssh)
