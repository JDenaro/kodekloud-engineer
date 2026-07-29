# Task 10: File Permission Correction

After conducting a security audit within the Stratos DC, the Nautilus security team discovered misconfigured permissions on critical files. To address this, corrective actions are being taken by the production support team. Specifically, the file named `/etc/resolv.conf` on Nautilus App 1 server requires adjustments to its Access Control Lists (ACLs) as follows:

1. The file's user owner and group owner should be set to `root`.

2. Others should possess read only permissions on the file.

3. User `mariyam` must not have any permissions on the file.

4. User `rod` should be granted read only permission on the file.

## Task Requirements

1. Work on App Server 1.
2. Set both the user owner and group owner of `/etc/resolv.conf` to `root`.
3. Give the `others` permission class read-only access.
4. Give `mariyam` no permissions through a named user ACL.
5. Give `rod` read-only access through a named user ACL.

## Solution

Traditional Linux permissions define access for the file owner, owning group, and all other users. Access Control Lists extend that model with entries for specific users and groups. This allows `rod` and `mariyam` to receive permissions that differ from the general `other::r--` entry.

The requested ownership, traditional permission, and named ACL entries are applied separately and then inspected together with `getfacl`.

### 🔌 Step 1: Connect to App Server 1

From the jump host, connect to App Server 1 as `tony`:

```bash
ssh tony@stapp01
```

The connection opened a shell on `stapp01`:

```text
thor@jump-host ~$ ssh tony@stapp01
tony@stapp01's password:
[tony@stapp01 ~]$
```

> **Why:** `ssh` opens a secure remote shell session. `tony` is the login account for App Server 1, and `stapp01` is the target hostname. The ownership and ACL changes must be applied to `/etc/resolv.conf` on the server specified by the challenge.

### 👥 Step 2: Set the user and group ownership

Set both owners to `root`:

```bash
sudo chown root:root /etc/resolv.conf
```

The command returned to the prompt without an error:

```text
[tony@stapp01 ~]$ sudo chown root:root /etc/resolv.conf
[tony@stapp01 ~]$
```

> **Why:** `sudo` provides the administrative privileges required to modify a critical system file. `chown` changes file ownership. In `root:root`, the value before the colon sets the user owner to `root`, and the value after the colon sets the group owner to `root`. `/etc/resolv.conf` is the target file.

### 🔐 Step 3: Give others read-only permission

Set the `others` permission class to read only:

```bash
sudo chmod o=r /etc/resolv.conf
```

The command completed silently:

```text
[tony@stapp01 ~]$ sudo chmod o=r /etc/resolv.conf
[tony@stapp01 ~]$
```

> **Why:** `chmod` changes file mode permissions. In the symbolic mode `o=r`, `o` selects users who are neither the owner nor members of the owning group, `=` replaces their current permissions, and `r` grants only read access. This removes any write or execute permission from the general `others` class without changing the owner permissions.

### 🚫 Step 4: Remove all permissions from mariyam

Create a named user ACL with no permissions:

```bash
sudo setfacl -m u:mariyam:--- /etc/resolv.conf
```

The command returned without an error:

```text
[tony@stapp01 ~]$ sudo setfacl -m u:mariyam:--- /etc/resolv.conf
[tony@stapp01 ~]$
```

> **Why:** `setfacl` sets or modifies Access Control Lists. The `-m` option modifies the existing ACL instead of replacing it. The entry `u:mariyam:---` targets the named user `mariyam` and grants no read, write, or execute permissions. A matching named-user ACL is evaluated specifically for that user, so `mariyam` does not fall back to the broader read permission assigned to `others`.

### 📖 Step 5: Give rod read-only permission

Add the requested named ACL:

```bash
sudo setfacl -m u:rod:r-- /etc/resolv.conf
```

The command completed successfully:

```text
[tony@stapp01 ~]$ sudo setfacl -m u:rod:r-- /etc/resolv.conf
[tony@stapp01 ~]$
```

> **Why:** The named-user entry `u:rod:r--` grants `rod` read permission while withholding write and execute access. By default, `setfacl` also recalculates the ACL mask when named entries are modified. The mask defines the maximum effective permissions available to named users other than the file owner and to group entries.

### ✅ Step 6: Verify the ownership and ACL

Display the complete ACL:

```bash
sudo getfacl /etc/resolv.conf
```

The command returned:

```text
[tony@stapp01 ~]$ sudo getfacl /etc/resolv.conf
getfacl: Removing leading '/' from absolute path names
# file: etc/resolv.conf
# owner: root
# group: root
user::rw-
user:mariyam:---
user:rod:r--
group::r--
mask::r--
other::r--

[tony@stapp01 ~]$
```

> **Why:** `getfacl` displays the filename, user owner, group owner, and all ACL entries. The ownership comments confirm `root:root`. `user:mariyam:---` denies every permission, `user:rod:r--` grants read only, and `other::r--` gives all remaining users read-only access. `mask::r--` allows at most read permission for the named-user and group-class entries, which is compatible with `rod`'s ACL and does not grant anything to `mariyam`. The message about removing the leading `/` is informational: by default, `getfacl` prints absolute pathnames without their leading slash in the output header.

## Best Practices

- **Use named ACLs for user-specific exceptions.** ACLs allow `mariyam` and `rod` to receive different access without changing the permissions for every other user.
- **Use exact assignment for restrictive permissions.** `o=r` replaces the complete `others` permission set with read only.
- **Understand ACL precedence.** A named user entry applies before the general `other` entry for that user.
- **Review the ACL mask.** The mask can restrict the effective permissions of named users and group entries even when their individual entries contain broader permissions.
- **Verify ownership and permissions together.** `getfacl` provides the owners, base permissions, named ACL entries, and mask in one output.
- **Apply the minimum necessary access.** Neither `mariyam` nor `rod` receives write or execute permission.

### 📚 Official Documentation

- [chown(1) Linux manual page](https://man7.org/linux/man-pages/man1/chown.1.html)
- [chmod(1) Linux manual page](https://man7.org/linux/man-pages/man1/chmod.1.html)
- [setfacl(1) Linux manual page](https://man7.org/linux/man-pages/man1/setfacl.1.html)
- [getfacl(1) Linux manual page](https://man7.org/linux/man-pages/man1/getfacl.1.html)
- [acl(5) Linux manual page](https://man7.org/linux/man-pages/man5/acl.5.html)
