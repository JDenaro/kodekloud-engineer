# Task 05: Temporary User Setup with Expiry

As part of the temporary assignment to the Nautilus project, a developer named `rose` requires access for a limited duration. To ensure smooth access management, a temporary user account with an expiry date is needed. Here's what you need to do:

Create a user named `rose` on App Server 3 in Stratos Datacenter. Set the expiry date to `2027-04-15`, ensuring the user is created in lowercase as per standard protocol.

Note: You can find the infrastructure details by clicking on the **Details of all Users and Servers** button on the top-right section of the page.

## Task Requirements

1. Create a user named `rose` on App Server 3.
2. Set the account expiry date to `2027-04-15`.
3. Ensure the username is written in lowercase.

## Solution

The temporary account was created on App Server 3 with an account expiry date of April 15, 2027. Account expiration controls access to the entire account and is separate from password expiration, which can have its own policy and dates.

Because the task explicitly requires creating the user, no pre-creation lookup is needed. The configured expiration information is inspected after creation.

### 🔌 Step 1: Connect to App Server 3

From the jump host, connect to App Server 3 as `banner`:

```bash
ssh banner@stapp03
```

The SSH connection opened a shell on `stapp03`:

```text
thor@jump-host ~$ ssh banner@stapp03
banner@stapp03's password:
[banner@stapp03 ~]$
```

> **Why:** `ssh` opens a secure remote shell session. `banner` is the login account for App Server 3, and `stapp03` is the target hostname. The temporary account must be created on the App Server specified by the challenge rather than on the jump host.

### 👤 Step 2: Create the temporary user

Create the lowercase user `rose` with the required expiry date:

```bash
sudo useradd -e 2027-04-15 rose
```

The command returned to the prompt without an error:

```text
[banner@stapp03 ~]$ sudo useradd -e 2027-04-15 rose
[banner@stapp03 ~]$
```

> **Why:** `sudo` supplies the administrative privileges required to create a system account. `useradd` creates the local user, and the final `rose` argument provides the required lowercase login name. The `-e` option sets the date on which the account will be disabled. Its value, `2027-04-15`, uses the supported `YYYY-MM-DD` format. Setting the date during creation ensures that the temporary access restriction exists from the beginning of the account's lifecycle.

### ✅ Step 3: Verify the account expiry date

Display the account-aging information for `rose`:

```bash
sudo chage -l rose
```

The command returned:

```text
[banner@stapp03 ~]$ sudo chage -l rose
Last password change                                    : Jul 26, 2026
Password expires                                        : never
Password inactive                                       : never
Account expires                                         : Apr 15, 2027
Minimum number of days between password change          : 0
Maximum number of days between password change          : 99999
Number of days of warning before password expires       : 7
[banner@stapp03 ~]$
```

> **Why:** `chage` manages and displays account and password aging information. The `-l` option lists the current values for the `rose` account. The decisive line is `Account expires : Apr 15, 2027`, which confirms that `useradd -e` stored the required account expiration date. `Password expires : never` does not conflict with the task because password expiration and account expiration are independent controls; the entire account will still become unavailable on its configured expiry date.

## Best Practices

- **Set expiration during account creation.** This prevents temporary access from being created without its required end date.
- **Use the standard date format.** `YYYY-MM-DD` is explicit and avoids ambiguity between day and month positions.
- **Keep usernames consistently lowercase.** Using `rose` exactly as requested follows the naming protocol and avoids creating a different account through capitalization.
- **Distinguish account and password expiration.** An account expiry date disables account access even when the password itself is configured to never expire.
- **Verify the effective account-aging data.** `chage -l rose` confirms the date stored by the system rather than relying only on the creation command's lack of errors.

### 📚 Official Documentation

- [useradd(8) Linux manual page](https://man7.org/linux/man-pages/man8/useradd.8.html)
- [chage(1) Linux manual page](https://man7.org/linux/man-pages/man1/chage.1.html)
