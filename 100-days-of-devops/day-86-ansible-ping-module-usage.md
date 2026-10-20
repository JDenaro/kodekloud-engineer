# Day 86: Ansible Ping Module Usage

The Nautilus DevOps team is planning to test several Ansible playbooks on different app servers in `Stratos DC`. Before that, some pre-requisites must be met. Essentially, the team needs to set up a password-less SSH connection between Ansible controller and Ansible managed nodes. One of the tickets is assigned to you; please complete the task as per details mentioned below:

## Specific Requirements:

1. `Jump host` is our Ansible controller, and we are going to run Ansible playbooks through `thor` user from `jump host`.

2. There is an inventory file `/home/thor/ansible/inventory` on `jump host`. Using that inventory file test `Ansible ping` from `jump host` to `App Server 2`, make sure ping works.

## Solution

The Jump Host is the Ansible controller, and App Server 2 (`stapp02`) is the managed node for this task. The controller runs as `thor`, while the remote SSH account is `steve`. Install `thor`'s public key in `steve`'s account, configure the existing inventory entry, and run the Ansible ping module directly; no playbook is needed.

An inventory password can let Ansible connect without prompting, but that still uses password authentication. This challenge also requests password-less SSH access, so the successful solution configures authentication with an SSH key. Ansible ping checks remote login and Python execution rather than sending ICMP network ping packets.

### 🔎 Step 1: Inspect the controller and existing inventory

From the Jump Host session:

```bash
whoami
cd /home/thor/ansible
cat inventory
ls -la /home/thor/.ssh
```

> **Why:** `whoami` identifies the current user, which must be `thor`. `cd` enters the directory containing the required inventory. `cat` displays that inventory so the host entries and connection settings can be inspected. `ls` lists the SSH directory; `-l` includes permissions and ownership, and `-a` includes hidden entries. Checking the directory before generating a key avoids overwriting an existing identity.

The current user was `thor`. The inventory contained `stapp01`, `stapp02`, and `stapp03`, each with an `ansible_ssh_pass` password variable but no explicit remote user. The `.ssh` directory contained no keys. Actual inventory passwords are omitted from this guide.

### 🔑 Step 2: Generate an SSH key for thor

```bash
ssh-keygen -t ed25519
```

> **Why:** `ssh-keygen` generates an SSH key pair, and `-t ed25519` selects the Ed25519 key type. The private key stays on the controller, while the public key can be installed on a managed node to authorize login.

At the prompts:

1. Press `Enter` to accept `/home/thor/.ssh/id_ed25519` as the private-key path.
2. Press `Enter` at both passphrase prompts to leave the key without a passphrase for unattended use in this temporary lab.

The command created:

```text
Your identification has been saved in /home/thor/.ssh/id_ed25519
Your public key has been saved in /home/thor/.ssh/id_ed25519.pub
```

### 📤 Step 3: Install the public key on App Server 2

```bash
ssh-copy-id -i /home/thor/.ssh/id_ed25519.pub steve@stapp02
```

> **Why:** `ssh-copy-id` installs the public key in the remote user's `~/.ssh/authorized_keys`, which lists keys allowed to authenticate to that account. `-i` selects the public-key file. `steve@stapp02` specifies the remote user and server. The initial connection uses `steve`'s password from the current lab wiki to authorize installing the key; later connections can authenticate with the matching private key.

At the first-time host authenticity prompt, verify the server identity and enter `yes` to record its host key. Enter `steve`'s password when prompted. The observed address of `stapp02` was `10.244.195.35`; this address may change between lab runs.

The command reported:

```text
Number of key(s) added: 1
```

### 📝 Step 4: Update only the target inventory entry

```bash
vi inventory
```

> **Why:** `vi` opens the existing inventory for editing. Ansible needs the remote user for App Server 2, and its password variable is no longer needed for this host now that key authentication is configured.

Press `i` and replace only the `stapp02` line with:

```ini
stapp02 ansible_user=steve
```

Keep the other two host entries unchanged. Save and exit with `Esc`, `:wq`, and `Enter`.

> **Why:** `stapp02` remains the inventory hostname and connection destination. `ansible_user=steve` selects the remote SSH account. Removing `ansible_ssh_pass` from this entry lets the connection use the default SSH key created for `thor` instead of supplying a stored password.

### 🔐 Step 5: Check SSH access without password prompts

```bash
ssh -o BatchMode=yes steve@stapp02 hostname
```

> **Why:** `ssh` connects to the remote account. `-o` supplies an SSH configuration option, and `BatchMode=yes` disables interactive prompts, including password prompts. This checks that unattended authentication works. The trailing `hostname` runs remotely and returns the server name before closing the connection.

The command succeeded without a password prompt and returned:

```text
stapp02
```

### ✅ Step 6: Verify

```bash
ansible stapp02 -i /home/thor/ansible/inventory -m ansible.builtin.ping
```

> **Why:** `ansible` executes an ad hoc task without a playbook. `stapp02` targets only App Server 2, `-i` selects the inventory required by the challenge, and `-m` selects the module. `ansible.builtin.ping` verifies that Ansible can log in and execute Python on the remote host, returning `pong` on success. No privilege escalation is required for this test.

The actual output was:

```text
stapp02 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}
```

`SUCCESS` and `"ping": "pong"` confirm that the module executed successfully on App Server 2. Ansible discovered `/usr/bin/python3` as the remote Python interpreter. `"changed": false` indicates that the test reported no configuration changes. KodeKloud success was explicitly confirmed after validation.

## Best Practices

- **Create keys under the controller's execution account.** The task runs as `thor`, so that account must have access to the private key used for the remote connection.
- **Keep private keys private.** Install only the `.pub` file on the managed node and keep private-key contents out of transcripts and repositories.
- **Configure the remote user explicitly.** `thor` controls Ansible locally, while `steve` is the login account on App Server 2.
- **Verify unattended authentication separately.** A successful SSH command with `BatchMode=yes` demonstrates that login works without a password prompt before testing the Ansible module.
- **Distinguish stored passwords from key authentication.** Avoiding a prompt by storing a password does not establish password-less SSH authentication.
- **Limit changes and tests to the required host.** Preserve the other inventory entries and target `stapp02` rather than all application servers.
- **Protect reusable keys.** This temporary lab used an empty passphrase for unattended access. For persistent environments, protect keys with a passphrase and an appropriate SSH agent or automation credential system.

### 📚 Official Documentation

- [Ansible: ping module](https://docs.ansible.com/projects/ansible/latest/collections/ansible/builtin/ping_module.html)
- [Ansible: How to build your inventory](https://docs.ansible.com/projects/ansible/latest/inventory_guide/intro_inventory.html)
- [OpenSSH: ssh-keygen manual](https://man.openbsd.org/ssh-keygen)
- [OpenSSH: SSH client configuration, including BatchMode](https://man.openbsd.org/ssh_config)
