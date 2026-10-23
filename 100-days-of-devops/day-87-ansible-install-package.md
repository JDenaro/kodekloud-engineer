# Day 87: Ansible Install Package

The Nautilus Application development team wanted to test some applications on app servers in `Stratos Datacenter`. They shared some pre-requisites with the DevOps team, and packages need to be installed on app servers. Since we are already using Ansible for automating such tasks, please perform this task using Ansible as per details mentioned below:

## Specific Requirements:

1. Create an inventory file `/home/thor/playbook/inventory` on `jump host` and add all app servers in it.
2. Create an Ansible playbook `/home/thor/playbook/playbook.yml` to install `logrotate` package on `all  app servers` using Ansible `yum` module.
3. Make sure user `thor` should be able to run the playbook on `jump host`.

`Note:` Validation will try to run playbook using command `ansible-playbook -i inventory playbook.yml` so please make sure playbook works this way, without passing any extra arguments.

## Solution

The Jump Host is the Ansible controller, and the three App Servers are managed hosts. The inventory groups the servers and provides each one its remote login credentials. The playbook uses privilege escalation to install `logrotate` with Ansible's `yum` module. Create and run both files as `thor` so the required validation command can find and read them from `/home/thor/playbook`.

### 📁 Step 1: Create the playbook directory

```bash
mkdir -p /home/thor/playbook
cd /home/thor/playbook
```

> **Why:** `mkdir -p` creates the required directory and does not fail if it already exists. `cd` changes into that directory so the inventory and playbook are saved where the validator expects them. Run these commands as `thor` so that user owns and can run the files.

### 🗂️ Step 2: Create the inventory

```bash
vi inventory
```

Enter the following inventory, replacing each password placeholder with the credential for the current lab. Do not include actual passwords in this guide:

```ini
[app_servers]
stapp01 ansible_user=tony ansible_password=<tony-password> ansible_become_password=<tony-password>
stapp02 ansible_user=steve ansible_password=<steve-password> ansible_become_password=<steve-password>
stapp03 ansible_user=banner ansible_password=<banner-password> ansible_become_password=<banner-password>
```

Save and exit `vi` with `Esc`, `:wq`, and `Enter`.

> **Why:** `vi` creates or edits the inventory file. The `[app_servers]` group lets the playbook target all three managed hosts together. Each `ansible_user` selects the SSH account for that server; `ansible_password` supplies its SSH password, and `ansible_become_password` supplies the password Ansible needs when escalating privileges. Lab passwords can change, so use the values shown for the current lab.

### 📝 Step 3: Create the playbook

```bash
vi playbook.yml
```

Enter and save this playbook:

```yaml
---
- name: Install logrotate on all app servers
  hosts: app_servers
  become: true
  tasks:
    - name: Ensure logrotate is installed
      ansible.builtin.yum:
        name: logrotate
        state: present
```

Save and exit `vi` with `Esc`, `:wq`, and `Enter`.

> **Why:** `vi` creates the playbook at the required path. `hosts: app_servers` selects the inventory group. `become: true` asks Ansible to use administrative privileges for package installation. The `ansible.builtin.yum` module manages packages on hosts that use YUM; `name` identifies `logrotate`, and `state: present` ensures it is installed while leaving an already-installed package in place.

### ✅ Step 4: Verify

```bash
ansible-playbook -i inventory playbook.yml
```

> **Why:** `ansible-playbook` runs the playbook. `-i inventory` selects the inventory in the current directory, and `playbook.yml` is the playbook file. This matches the validation command and needs no additional arguments. The successful lab was confirmed by the user. The recap should show all three app servers with `failed=0`.

## Best Practices

- **Run as the expected controller user.** Create and run the files as `thor` so the validator can access them on the Jump Host.
- **Use an inventory group for shared work.** A named group allows one play to apply the package task to every app server.
- **Keep credentials specific to the current lab.** Use the current server login and privilege escalation passwords; do not reuse credentials from another lab.
- **Make package tasks repeatable.** `state: present` keeps the playbook safe to run again without reinstalling an already-present package.

### 📚 Official Documentation

- [Ansible: yum module](https://docs.ansible.com/projects/ansible/latest/collections/ansible/builtin/yum_module.html)
- [Ansible: How to build your inventory](https://docs.ansible.com/projects/ansible/latest/inventory_guide/intro_inventory.html)
- [Ansible: Understanding privilege escalation](https://docs.ansible.com/projects/ansible/latest/playbook_guide/playbooks_privilege_escalation.html)
