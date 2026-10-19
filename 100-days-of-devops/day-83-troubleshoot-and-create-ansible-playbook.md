# Day 83: Troubleshoot and Create Ansible Playbook

An Ansible playbook needs completion on the `jump host`, where a team member left off. Below are the details:

## Specific Requirements:

1. The inventory file `/home/thor/ansible/inventory` requires adjustments. The playbook must run on `App Server 1` in `Stratos DC`. Update the inventory accordingly.
2. Create a playbook `/home/thor/ansible/playbook.yml`. Include a task to create an empty file `/tmp/file.txt` on `App Server 1`.

`Note:` Validation will run the playbook using the command `ansible-playbook -i inventory playbook.yml`. Ensure the playbook works without any additional arguments.

## Solution

The existing inventory targeted `stapp03` with user `banner`, while the challenge required `stapp01` with user `tony`. Its password value was the literal string `$pwd`: Ansible does not expand shell variables in an INI inventory. The solution corrected the target and credentials, then created a playbook using the built-in file module.

### 🔎 Step 1: Inspect the existing inventory

From the Jump Host session as `thor`:

```bash
cd /home/thor/ansible
ls -l
cat inventory
```

> **Why:** `cd` enters the required working directory. `ls -l` lists files with their permissions, ownership, and sizes; `cat inventory` displays the existing host configuration so we can identify the changes needed before editing it.

The directory initially contained only the inventory:

```text
total 4
-rw-r--r-- 1 thor thor 103 Oct 19 00:48 inventory
```

Its original contents were:

```ini
stapp03 ansible_user=banner ansible_ssh_pass=$pwd ansible_ssh_common_args='-o StrictHostKeyChecking=no'
```

`ansible_ssh_pass` supplies an SSH password, but `$pwd` here is a literal value, not a reference to a shell variable. Both the host and login user also belonged to App Server 3, so they had to be replaced for this challenge.

### 📝 Step 2: Correct the inventory

```bash
vi inventory
```

> **Why:** `vi` opens the inventory for editing. An inventory identifies the managed hosts and can provide the connection variables needed for unattended execution.

Replace the existing line with the following, substituting Tony's password from the current lab wiki for the placeholder:

```ini
stapp01 ansible_user=tony ansible_password='TONY_LAB_PASSWORD' ansible_ssh_common_args='-o StrictHostKeyChecking=no'
```

In `vi`, press `i` to enter insert mode. After editing, press `Esc`, type `:wq`, and press `Enter` to save and exit.

> **Why:** `stapp01` is both the inventory hostname and the destination Ansible connects to. `ansible_user` selects `tony` as the SSH login account, and `ansible_password` supplies its password without a command-line prompt. `ansible_ssh_common_args` passes additional options to SSH. The existing lab option `-o StrictHostKeyChecking=no` was retained: `-o` supplies an SSH setting, and this value permits first-time connections without an interactive host-key confirmation. The real password is intentionally omitted from this guide.

### 📄 Step 3: Create the playbook

```bash
vi playbook.yml
```

> **Why:** This creates the playbook at the required path, `/home/thor/ansible/playbook.yml`. A playbook describes the hosts to target and the tasks Ansible should perform on them.

Press `i` and enter:

```yaml
---
- name: Create the required file on App Server 1
  hosts: stapp01
  tasks:
    - name: Create an empty file
      ansible.builtin.file:
        path: /tmp/file.txt
        state: touch
```

Save and exit with `Esc`, `:wq`, and `Enter`.

> **Why:** `---` marks the start of a YAML document. The `name` fields provide readable labels for the play and task, while `hosts: stapp01` selects the required server and `tasks` lists its actions. `ansible.builtin.file` manages filesystem objects; `path` identifies the remote file, and `state: touch` creates an empty file when it does not exist. If a file already exists, this state updates its timestamps without clearing its contents. No privilege escalation was needed to create the file in `/tmp` as `tony`.

### 🔒 Step 4: Protect the inventory

```bash
chmod 600 inventory
```

> **Why:** `chmod` changes file permissions. Mode `600` gives the owner, `thor`, read and write access while denying access to the group and other users. This restricts access to the password stored in the inventory.

### ✅ Step 5: Verify

From `/home/thor/ansible/`, run exactly the command required by the challenge:

```bash
ansible-playbook -i inventory playbook.yml
```

> **Why:** `ansible-playbook` executes a playbook. `-i inventory` selects the corrected inventory, and `playbook.yml` selects the task definition. Using the exact validation command confirms that no additional login or privilege-escalation arguments are needed.

The actual execution returned:

```text
PLAY [Create the required file on App Server 1] *********************************

TASK [Gathering Facts] **********************************************************
ok: [stapp01]

TASK [Create an empty file] *****************************************************
changed: [stapp01]

PLAY RECAP **********************************************************************
stapp01                    : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

`ok=2` records successful fact gathering and task execution, while `changed=1` records the file task's change. `unreachable=0` and `failed=0` confirm successful connectivity and execution on `stapp01`. KodeKloud success was explicitly confirmed after validation.

## Best Practices

- **Match the inventory to the requested server.** Correct the hostname and its associated login account together when an inventory was left configured for another host.
- **Use actual connection values.** Shell-style placeholders such as `$pwd` are not expanded in INI inventory variables. Supply the current lab credentials and keep real passwords out of repository documentation.
- **Understand the selected file state.** `touch` creates an empty file when absent; it preserves existing contents and normally updates timestamps on subsequent runs.
- **Use only the permissions needed.** Creating this file in `/tmp` succeeded as `tony`, so the playbook did not require administrative privileges.
- **Keep SSH settings appropriate to the environment.** The existing host-key option was retained for this temporary lab. For persistent environments, manage trusted host keys and protect credentials with a supported secret-management mechanism.

### 📚 Official Documentation

- [Ansible: How to build your inventory](https://docs.ansible.com/projects/ansible/latest/inventory_guide/intro_inventory.html)
- [Ansible: file module](https://docs.ansible.com/projects/ansible/latest/collections/ansible/builtin/file_module.html)
- [Ansible: SSH connection plugin](https://docs.ansible.com/projects/ansible/latest/collections/ansible/builtin/ssh_connection.html)
