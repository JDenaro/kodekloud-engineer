# Day 85: Create Files on App Servers using Ansible

The Nautilus DevOps team is testing various Ansible modules on servers in `Stratos DC`. They're currently focusing on file creation on remote hosts using Ansible. Here are the details:

## Specific Requirements:

1. Create an inventory file `~/playbook/inventory` on `jump host` and include `all app servers`.

2. Create a playbook `~/playbook/playbook.yml` to create a blank file `/opt/data.txt` on `all app servers`.

3. Set the permissions of the `/opt/data.txt` file to `0755`.

4. Ensure the user/group owner of the `/opt/data.txt` file is `tony` on `app server 1`, `steve` on `app server 2` and `banner` on `app server 3`.

`Note:` Validation will execute the playbook using the command `ansible-playbook -i inventory playbook.yml`, so ensure the playbook functions correctly without any additional arguments.

## Solution

The Jump Host runs Ansible as the controller, and the three application servers are its managed nodes. A single file task creates `/opt/data.txt` and applies its permissions and ownership on each server. In this lab, each required owner and group matches that server's SSH login account, so the playbook uses the inventory variable `ansible_user` for both.

### 📁 Step 1: Prepare the working directory

From the Jump Host session as `thor`:

```bash
mkdir -p ~/playbook
cd ~/playbook
```

> **Why:** `mkdir` creates a directory; `-p` also creates missing parent directories and accepts an existing directory without an error. `~` refers to the current user's home directory, so the lab uses `/home/thor/playbook`. `cd` enters that directory so the inventory and playbook are saved at the required paths.

### 🔑 Step 2: Establish SSH access to each application server

```bash
ssh tony@stapp01 hostname
ssh steve@stapp02 hostname
ssh banner@stapp03 hostname
```

> **Why:** `ssh` opens an encrypted connection with the user before `@` and the server after it. The trailing `hostname` runs remotely, displays the server name, and returns to the Jump Host. These connections check the login credentials and allow the servers' SSH host keys to be recorded before Ansible connects without interactive prompts.

At each first-time authenticity prompt, verify the server identity and enter `yes` to accept its host key. Enter that user's password from the current lab wiki when prompted. The commands returned `stapp01`, `stapp02`, and `stapp03`, respectively.

The addresses observed in this lab were:

| Server | Login user | Observed IP address |
| --- | --- | --- |
| `stapp01` | `tony` | `10.244.247.233` |
| `stapp02` | `steve` | `10.244.244.137` |
| `stapp03` | `banner` | `10.244.195.6` |

These addresses can change between lab runs; the inventory uses the server names.

### 📝 Step 3: Create the inventory

```bash
vi inventory
```

> **Why:** `vi` opens the text editor to create the inventory in the current directory. An inventory lists Ansible's managed nodes and the variables needed to connect to them.

Press `i` to enter insert mode and add the following, replacing each password placeholder with the corresponding password from the current lab wiki:

```ini
[app_servers]
stapp01 ansible_user=tony ansible_password='TONY_LAB_PASSWORD' ansible_become_password='TONY_LAB_PASSWORD'
stapp02 ansible_user=steve ansible_password='STEVE_LAB_PASSWORD' ansible_become_password='STEVE_LAB_PASSWORD'
stapp03 ansible_user=banner ansible_password='BANNER_LAB_PASSWORD' ansible_become_password='BANNER_LAB_PASSWORD'
```

Save and exit with `Esc`, `:wq`, and `Enter`.

> **Why:** `[app_servers]` groups all three application servers. Each hostname is also its connection destination. `ansible_user` selects the SSH login account, `ansible_password` supplies its SSH password, and `ansible_become_password` supplies the password used for privilege escalation through `sudo`. Providing these variables lets the validation command run without additional password arguments. Actual credentials are omitted from this guide.

### 🔒 Step 4: Protect the inventory

```bash
chmod 600 inventory
```

> **Why:** `chmod` changes file permissions. Mode `600` allows only the owner, `thor`, to read and modify the inventory, protecting its credentials from other local users.

### 📄 Step 5: Create the file playbook

```bash
vi playbook.yml
```

> **Why:** This creates the required playbook in `/home/thor/playbook`. A playbook defines which hosts to manage and which tasks to run on them.

Press `i` and enter:

```yaml
---
- name: Create a file on all application servers
  hosts: app_servers
  become: true
  tasks:
    - name: Create data.txt with the required permissions and ownership
      ansible.builtin.file:
        path: /opt/data.txt
        state: touch
        owner: "{{ ansible_user }}"
        group: "{{ ansible_user }}"
        mode: '0755'
```

Save and exit with `Esc`, `:wq`, and `Enter`.

> **Why:** `---` starts the YAML document, `name` supplies readable labels, and `tasks` lists the actions to execute. `hosts: app_servers` selects the inventory group. `become: true` enables privilege escalation to `root` by default, allowing the task to create a file in `/opt` and set its ownership.

> **Why:** `ansible.builtin.file` manages filesystem objects and their attributes. `path` identifies the target, and `state: touch` creates an empty file if it does not exist. `owner` selects its user and `group` its group. The expression `{{ ansible_user }}` resolves separately for each host: `tony`, `steve`, or `banner`. Mode `'0755'` grants the owner read, write, and execute access, and grants the group and others read and execute access. Quoting the mode lets Ansible interpret it as an octal permission string.

With `state: touch`, an existing file keeps its contents and receives updated access and modification times. Repeated runs can therefore report `changed` even when ownership and permissions already match.

### ✅ Step 6: Verify

Run the exact validation command from `~/playbook`:

```bash
ansible-playbook -i inventory playbook.yml
```

> **Why:** `ansible-playbook` executes the playbook. `-i inventory` selects the inventory file containing the hosts and connection variables, and `playbook.yml` identifies the playbook to execute. Using the exact required command checks that no additional connection or privilege-escalation arguments are needed.

The actual output was:

```text
PLAY [Create a file on all application servers] *********************************

TASK [Gathering Facts] **********************************************************
ok: [stapp03]
ok: [stapp01]
ok: [stapp02]

TASK [Create data.txt with the required permissions and ownership] **************
changed: [stapp01]
changed: [stapp03]
changed: [stapp02]

PLAY RECAP **********************************************************************
stapp01                    : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
stapp02                    : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
stapp03                    : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

Ansible gathered facts and completed the file task on all three servers. Each recap shows `ok=2`, `changed=1`, `unreachable=0`, and `failed=0`. KodeKloud success was explicitly confirmed after validation.

## Best Practices

- **Group hosts that share a task.** One inventory group lets the playbook manage all application servers consistently.
- **Keep ownership specific to each host.** Reusing `ansible_user` works here because the required user and group match the SSH account on every server. Other environments may require dedicated ownership variables.
- **Set permissions explicitly.** Use the required quoted mode `'0755'` rather than relying on default permissions.
- **Understand touch behavior.** It creates an empty file when absent, preserves existing contents, and updates timestamps on repeated runs.
- **Protect connection credentials.** Restrict inventory permissions and keep real passwords out of repository files. For persistent environments, use Ansible Vault or another supported secret-management mechanism.
- **Check every server's recap.** All three hosts must show zero unreachable hosts and zero failures.

### 📚 Official Documentation

- [Ansible: How to build your inventory](https://docs.ansible.com/projects/ansible/latest/inventory_guide/intro_inventory.html)
- [Ansible: file module](https://docs.ansible.com/projects/ansible/latest/collections/ansible/builtin/file_module.html)
- [Ansible: Understanding privilege escalation — become](https://docs.ansible.com/projects/ansible/latest/playbook_guide/playbooks_privilege_escalation.html)
- [Ansible: ansible-playbook command](https://docs.ansible.com/projects/ansible/latest/cli/ansible-playbook.html)
