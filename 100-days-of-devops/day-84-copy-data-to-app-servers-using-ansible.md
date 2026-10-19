# Day 84: Copy Data to App Servers using Ansible

The Nautilus DevOps team needs to copy data from the `jump host` to all `application servers` in `Stratos DC` using Ansible. Execute the task with the following details:

## Specific Requirements:

1. Create an inventory file `/home/thor/ansible/inventory` on `jump_host` and add all application servers as managed nodes.
2. Create a playbook `/home/thor/ansible/playbook.yml` on the `jump host` to copy the `/usr/src/dba/index.html` file to all application servers, placing it at `/opt/dba`.

`Note:` Validation will run the playbook using the command `ansible-playbook -i inventory playbook.yml`. Ensure the playbook functions properly without any extra arguments.

## Solution

The Jump Host acts as the Ansible controller: it reads the source file and sends it to the managed application servers. The inventory groups `stapp01`, `stapp02`, and `stapp03`, and the playbook copies the file to `/opt/dba/index.html` on each server. Privilege escalation supplies the administrative access needed to manage this destination.

### 🔎 Step 1: Inspect the source file

From the Jump Host session as `thor`:

```bash
cd /home/thor/ansible
ls -l /usr/src/dba/index.html
```

> **Why:** `cd` enters the required working directory so the inventory and playbook are created in the right location. `ls -l` displays the source file's permissions, owner, and size. Ansible must be able to read this file on the controller before transferring it to the managed servers.

The source file was present and readable:

```text
-rw-r--r-- 1 root root 35 Oct 19 01:28 /usr/src/dba/index.html
```

### 🔑 Step 2: Establish SSH access to each server

```bash
ssh tony@stapp01 hostname
ssh steve@stapp02 hostname
ssh banner@stapp03 hostname
```

> **Why:** `ssh` opens an encrypted connection using the account before `@` and the server after it. The trailing `hostname` runs that command remotely, prints the server name, and returns to the Jump Host. These connections verify each login and allow the first-time host keys to be recorded in the user's `known_hosts` file before unattended Ansible execution.

At each first-time authenticity prompt, check the server identity and enter `yes` to accept its host key. Enter the corresponding password from the current lab wiki when prompted. The commands returned `stapp01`, `stapp02`, and `stapp03`, respectively.

The actual server addresses observed in this lab were:

| Server | Login user | Observed IP address |
| --- | --- | --- |
| `stapp01` | `tony` | `10.244.226.138` |
| `stapp02` | `steve` | `10.244.195.208` |
| `stapp03` | `banner` | `10.244.49.36` |

These addresses can change between lab runs; the inventory uses the server names.

### 📝 Step 3: Create the inventory

```bash
vi inventory
```

> **Why:** `vi` opens a text editor to create the inventory in `/home/thor/ansible/`. An inventory lists the managed nodes and their connection settings.

Press `i` and enter the following, replacing each password placeholder with the corresponding user's password from the current lab wiki:

```ini
[app_servers]
stapp01 ansible_user=tony ansible_password='TONY_LAB_PASSWORD' ansible_become_password='TONY_LAB_PASSWORD'
stapp02 ansible_user=steve ansible_password='STEVE_LAB_PASSWORD' ansible_become_password='STEVE_LAB_PASSWORD'
stapp03 ansible_user=banner ansible_password='BANNER_LAB_PASSWORD' ansible_become_password='BANNER_LAB_PASSWORD'
```

Save and exit with `Esc`, `:wq`, and `Enter`.

> **Why:** `[app_servers]` defines a group containing all three application servers. Each hostname also serves as the connection destination. `ansible_user` selects the SSH account, `ansible_password` supplies its login password, and `ansible_become_password` supplies the password for privilege escalation through `sudo`. These variables allow the required command to run without additional password arguments. Actual credentials are omitted from this guide.

The saved inventory was inspected with:

```bash
cat inventory
```

> **Why:** `cat` displays the saved inventory so the server names, group, and connection variables can be checked before running the playbook. Avoid sharing its real password values in terminal transcripts or repository files.

### 📄 Step 4: Create the copy playbook

```bash
vi playbook.yml
```

> **Why:** This opens the required playbook file in the current directory. A playbook specifies the target hosts and the tasks Ansible executes on them.

Press `i` and enter:

```yaml
---
- name: Copy data to all application servers
  hosts: app_servers
  become: true
  tasks:
    - name: Ensure the destination directory exists
      ansible.builtin.file:
        path: /opt/dba
        state: directory
        mode: '0755'

    - name: Copy index.html from the jump host
      ansible.builtin.copy:
        src: /usr/src/dba/index.html
        dest: /opt/dba/index.html
        mode: '0644'
```

Save and exit with `Esc`, `:wq`, and `Enter`.

> **Why:** `---` starts the YAML document, `name` gives readable labels to the play and tasks, and `tasks` lists the actions to execute. `hosts: app_servers` selects the inventory group. `become: true` enables privilege escalation, using `root` as the default target user. The file module manages the `path`: `state: directory` ensures that it is a directory, creating it if needed, and `mode: '0755'` gives its owner full access and others read and traversal access. Quoting permission modes keeps them as strings for Ansible to interpret.

> **Why:** The copy module reads `src` on the controller by default and writes it to the remote `dest`. The explicit filename produces `/opt/dba/index.html` on each server. Its parent directory is managed first because copying to a file path does not automatically create missing parent directories. `mode: '0644'` gives the file owner read and write access and others read access.

### 🔒 Step 5: Protect the inventory

```bash
chmod 600 inventory
```

> **Why:** `chmod` changes file permissions. Mode `600` allows only the owner, `thor`, to read and modify the inventory, protecting the credentials it contains from other local users.

### ✅ Step 6: Verify

Run the exact validation command from `/home/thor/ansible/`:

```bash
ansible-playbook -i inventory playbook.yml
```

> **Why:** `ansible-playbook` executes the playbook, while `-i inventory` selects its managed hosts and connection variables. `playbook.yml` identifies the task file. Using the specified command verifies that no additional connection or privilege-escalation arguments are required.

The actual output was:

```text
PLAY [Copy data to all application servers] *************************************************

TASK [Gathering Facts] **********************************************************************
ok: [stapp01]
ok: [stapp03]
ok: [stapp02]

TASK [Ensure the destination directory exists] **********************************************
ok: [stapp01]
ok: [stapp03]
ok: [stapp02]

TASK [Copy index.html from the jump host] ***************************************************
changed: [stapp03]
changed: [stapp02]
changed: [stapp01]

PLAY RECAP **********************************************************************************
stapp01                    : ok=3    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
stapp02                    : ok=3    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
stapp03                    : ok=3    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

The directory task returned `ok` on all servers, indicating that the directories already met the requested state. The copy task returned `changed` on each server. Every recap shows `ok=3`, `changed=1`, `unreachable=0`, and `failed=0`, confirming successful execution on all three managed nodes. KodeKloud success was explicitly confirmed after validation.

## Best Practices

- **Distinguish the controller from managed nodes.** The source file is on the Jump Host, and each application server receives its own destination copy.
- **Group servers with a shared task.** A single inventory group lets the same play target all application servers consistently.
- **Manage destination directories before copying.** An explicit directory task handles missing directories and keeps the required permissions consistent.
- **Set readable permission modes explicitly.** Quote modes such as `'0755'` and `'0644'` so Ansible interprets them correctly.
- **Keep credentials private.** Restrict inventory access and use placeholders in guides. For persistent environments, store secrets with Ansible Vault or another supported secret-management mechanism.
- **Check every target's recap.** Success on one server does not establish that the file reached all servers; each target must show zero unreachable hosts and zero failures.

### 📚 Official Documentation

- [Ansible: How to build your inventory](https://docs.ansible.com/projects/ansible/latest/inventory_guide/intro_inventory.html)
- [Ansible: copy module](https://docs.ansible.com/projects/ansible/latest/collections/ansible/builtin/copy_module.html)
- [Ansible: file module](https://docs.ansible.com/projects/ansible/latest/collections/ansible/builtin/file_module.html)
- [Ansible: Understanding privilege escalation — become](https://docs.ansible.com/projects/ansible/latest/playbook_guide/playbooks_privilege_escalation.html)
