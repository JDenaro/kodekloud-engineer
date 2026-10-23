# Day 89: Ansible Manage Services

Developers are looking for dependencies to be installed and run on Nautilus app servers in Stratos DC. They have shared some requirements with the DevOps team. Because we are now managing packages installation and services management using Ansible, some playbooks need to be created and tested. As per details mentioned below please complete the task:

a. On `jump host` create an Ansible playbook `/home/thor/ansible/playbook.yml` and configure it to install `httpd` on all app servers.

b. After installation make sure to start and enable `httpd` service on all app servers.

c. The inventory `/home/thor/ansible/inventory` is already there on `jump host`.

d. Make sure user `thor` should be able to run the playbook on `jump host`.

`Note:` Validation will try to run playbook using command `ansible-playbook -i inventory playbook.yml` so please make sure playbook works this way, without passing any extra arguments.

## Specific Requirements:

1. On `jump host` create an Ansible playbook `/home/thor/ansible/playbook.yml` and configure it to install `httpd` on all app servers.
2. After installation make sure to start and enable `httpd` service on all app servers.
3. The inventory `/home/thor/ansible/inventory` is already there on `jump host`.
4. Make sure user `thor` should be able to run the playbook on `jump host`.
5. `Note:` Validation will try to run playbook using command `ansible-playbook -i inventory playbook.yml` so please make sure playbook works this way, without passing any extra arguments.

## Solution

The existing inventory contains the three app servers and their SSH connection settings. The playbook targets every inventory host, installs `httpd`, and starts and enables its service. Create and run the playbook as `thor` from `/home/thor/ansible` so the required validation command can use the existing inventory and the playbook without extra arguments. Inventory passwords are omitted from this guide.

### 📂 Step 1: Enter the Ansible directory

```bash
cd /home/thor/ansible/
```

> **Why:** `cd` changes the current directory to the location of the existing inventory. The validation command uses the relative filename `inventory`, so running it from this directory lets Ansible find the expected file. The lab runs on the Jump Host as `thor`.

### 📝 Step 2: Create the playbook

```bash
vi playbook.yml
```

Enter and save this playbook:

```yaml
---
- name: Install and manage httpd on all app servers
  hosts: all
  become: true
  tasks:
    - name: Install httpd
      ansible.builtin.yum:
        name: httpd
        state: present

    - name: Start and enable httpd
      ansible.builtin.service:
        name: httpd
        state: started
        enabled: true
```

Save and exit `vi` with `Esc`, `:wq`, and `Enter`.

> **Why:** `vi` creates the required playbook. `hosts: all` targets every server in the existing inventory. `become: true` gives Ansible the administrative privileges needed to install a package and manage a system service. The `yum` module ensures `httpd` is installed with `name` and `state: present`. The `service` module uses `state: started` to start the service and `enabled: true` to configure it to start at boot.

### ✅ Step 3: Verify

```bash
ansible-playbook -i inventory playbook.yml
```

> **Why:** `ansible-playbook` runs the playbook. The `-i` option selects the existing inventory file in the current directory, and `playbook.yml` identifies the playbook. This is the exact validation command and requires no extra arguments.

The play completed successfully on all three app servers. The observed recap was:

```text
PLAY RECAP *********************************************************************
stapp01 : ok=3    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
stapp02 : ok=3    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
stapp03 : ok=3    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

`failed=0` and `unreachable=0` for each host confirm the play ran successfully. Each server reported changes for package installation and service management.

## Best Practices

- **Use the existing inventory.** Keeping the provided inventory avoids duplicating connection details and lets the play target the intended app servers as a group.
- **Declare the desired package state.** `state: present` makes the playbook safe to run repeatedly and ensures `httpd` is installed.
- **Declare service state and startup behavior.** `started` starts the service now, while `enabled` keeps it configured to start after a reboot.
- **Run the playbook as the required controller user.** Creating and executing the files as `thor` ensures the required user can run the validation command.
- **Keep credentials out of documentation.** The inventory contains temporary lab authentication details, which are not needed to understand the playbook.

### 📚 Official Documentation

- [Ansible: yum module](https://docs.ansible.com/projects/ansible/latest/collections/ansible/builtin/yum_module.html)
- [Ansible: service module](https://docs.ansible.com/projects/ansible/latest/collections/ansible/builtin/service_module.html)
- [Ansible: Understanding privilege escalation](https://docs.ansible.com/projects/ansible/latest/playbook_guide/playbooks_privilege_escalation.html)
- [Ansible: How to build your inventory](https://docs.ansible.com/projects/ansible/latest/inventory_guide/intro_inventory.html)
