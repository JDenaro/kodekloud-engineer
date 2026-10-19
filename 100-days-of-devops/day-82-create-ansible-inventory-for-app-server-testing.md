# Day 82: Create Ansible Inventory for App Server Testing

The Nautilus DevOps team is testing Ansible playbooks on various servers within their stack. They've placed some playbooks under `/home/thor/playbook/` directory on the `jump host` and now intend to test them on `app server 1` in `Stratos DC`. However, an inventory file needs creation for Ansible to connect to the respective app. Here are the requirements:

## Specific Requirements:

1. Create an ini type Ansible inventory file `/home/thor/playbook/inventory` on `jump host`.
2. Include `App Server 1` in this inventory along with necessary variables for proper functionality.
3. Ensure the inventory hostname corresponds to the `server name` as per the wiki, for example `stapp01` for `app server 1` in `Stratos DC`.

`Note:` Validation will execute the playbook using the command `ansible-playbook -i inventory playbook.yml`. Ensure the playbook functions properly without any extra arguments.

## Solution

An inventory tells Ansible which servers to manage and how to connect to them. This lab required one host, `stapp01`, with the SSH login user `tony`. The provided playbook already enabled privilege escalation to `root`, so the inventory supplied both SSH and privilege-escalation credentials without requiring interactive password flags.

### 🔎 Step 1: Inspect the provided playbook

From the Jump Host session as `thor`:

```bash
cd /home/thor/playbook
ls -l
cat playbook.yml
```

> **Why:** `cd` enters the required working directory so subsequent relative filenames refer to the correct files. `ls -l` lists its contents with permissions, ownership, and sizes. `cat` displays the existing playbook so we can identify its target hosts and privilege requirements before creating the inventory.

The directory contained the pre-provided configuration and playbook:

```text
total 8
-rw-r--r-- 1 thor thor  36 Oct 19 00:14 ansible.cfg
-rw-r--r-- 1 thor thor 250 Oct 19 00:14 playbook.yml
```

The playbook contained:

```yaml
---
- hosts: all
  become: yes
  become_user: root
  tasks:
    - name: Install httpd package
      yum:
        name: httpd
        state: installed

    - name: Start service httpd
      service:
        name: httpd
        state: started
```

`hosts: all` selects every host in the supplied inventory. `become: yes` enables privilege escalation, and `become_user: root` selects the administrative account. The tasks install the `httpd` package, which provides the Apache web server, and start its service. The supplied playbook did not need modification.

### 📝 Step 2: Create the INI inventory

```bash
vi inventory
```

> **Why:** `vi` opens a text editor to create the required `inventory` file in `/home/thor/playbook/`. An INI inventory can list a host and its connection variables on one line; a named group is unnecessary because the playbook targets `all`.

Press `i` to enter insert mode and add this line, replacing both password placeholders with Tony's password from the current lab wiki before saving:

```ini
stapp01 ansible_host=stapp01 ansible_user=tony ansible_password='TONY_LAB_PASSWORD' ansible_become_password='TONY_LAB_PASSWORD'
```

Press `Esc`, type `:wq`, and press `Enter` to save and exit.

> **Why:** The first `stapp01` is the inventory hostname, matching the server name required by the challenge. `ansible_host` specifies the destination used for the connection, and `ansible_user` selects the SSH login account. `ansible_password` supplies its SSH password. `ansible_become_password` supplies the password used for privilege escalation through `sudo`. The playbook already enables escalation, so no additional `ansible_become` variable is needed. The password values are intentionally replaced with placeholders in this guide.

### 🔒 Step 3: Restrict access to the inventory

```bash
chmod 600 inventory
```

> **Why:** `chmod` changes file permissions. Mode `600` grants read and write access to the owner, `thor`, and no access to the group or other users. This limits access to the credentials stored in the lab inventory.

### ✅ Step 4: Verify

Run the exact command specified by the challenge from `/home/thor/playbook/`:

```bash
ansible-playbook -i inventory playbook.yml
```

> **Why:** `ansible-playbook` executes the tasks defined in a playbook. `-i inventory` selects the inventory file created in this directory, and `playbook.yml` selects the provided playbook. Running this exact command verifies that the inventory supplies the connection and escalation information without extra arguments.

The actual output was:

```text
PLAY [all] **********************************************************************

TASK [Gathering Facts] **********************************************************
ok: [stapp01]

TASK [Install httpd package] ****************************************************
changed: [stapp01]

TASK [Start service httpd] ******************************************************
changed: [stapp01]

PLAY RECAP **********************************************************************
stapp01                    : ok=3    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

The recap identifies the host as `stapp01`. `ok=3` confirms that fact gathering and both tasks completed; `changed=2` records the package and service changes. `unreachable=0` and `failed=0` confirm that the connection and task execution succeeded. KodeKloud success was also explicitly confirmed after validation.

## Best Practices

- **Keep the required inventory hostname.** Use `stapp01` as the host entry so Ansible identifies the server by its wiki name, even when a separate connection address is supplied through `ansible_host`.
- **Provide credentials for both operations.** SSH login and privilege escalation are separate operations; the playbook needs access to the server and permission to perform administrative tasks.
- **Protect lab credentials.** Restrict the inventory permissions and keep actual passwords out of repository guides. For persistent environments, use Ansible Vault or another supported secret-management approach.
- **Verify the exact validation command.** Successful execution without password prompts or additional arguments demonstrates that the inventory meets the challenge's invocation requirement.

### 📚 Official Documentation

- [Ansible: How to build your inventory](https://docs.ansible.com/projects/ansible/latest/inventory_guide/intro_inventory.html)
- [Ansible: Understanding privilege escalation — become](https://docs.ansible.com/projects/ansible/latest/playbook_guide/playbooks_privilege_escalation.html)
- [Ansible: ansible-playbook command](https://docs.ansible.com/projects/ansible/latest/cli/ansible-playbook.html)
