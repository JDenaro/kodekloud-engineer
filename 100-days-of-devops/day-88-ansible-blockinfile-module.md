# Day 88: Ansible Blockinfile Module

The Nautilus DevOps team wants to install and set up a simple httpd web server on all app servers in Stratos DC. Additionally, they want to deploy a sample web page for now using Ansible only. Therefore, write the required playbook to complete this task. Find more details about the task below.

We already have an inventory file under /home/thor/ansible directory on jump host. Create a playbook.yml under /home/thor/ansible directory on jump host itself.

Using the playbook, install httpd web server on all app servers. Additionally, make sure its service should up and running.

Using blockinfile Ansible module add some content in /var/www/html/index.html file. Below is the content:

Welcome to XfusionCorp!

This is  Nautilus sample file, created using Ansible!

Please do not modify this file manually!

The /var/www/html/index.html file's user and group owner should be apache on all app servers.

The /var/www/html/index.html file's permissions should be 0655 on all app servers.

Note:

i. Validation will try to run the playbook using command ansible-playbook -i inventory playbook.yml so please make sure the playbook works this way without passing any extra arguments.

ii. Do not use any custom or empty marker for blockinfile module.

## Specific Requirements:

1. We already have an inventory file under `/home/thor/ansible` directory on jump host. Create a `playbook.yml` under `/home/thor/ansible` directory on jump host itself.
2. Using the playbook, install `httpd` web server on all app servers. Additionally, make sure its service should up and running.
3. Using `blockinfile` Ansible module add some content in `/var/www/html/index.html` file. Below is the content:

   ```text
   Welcome to XfusionCorp!

   This is  Nautilus sample file, created using Ansible!

   Please do not modify this file manually!
   ```

4. The `/var/www/html/index.html` file's user and group owner should be `apache` on all app servers.
5. The `/var/www/html/index.html` file's permissions should be `0655` on all app servers.
6. Note:
   i. Validation will try to run the playbook using command `ansible-playbook -i inventory playbook.yml` so please make sure the playbook works this way without passing any extra arguments.
   ii. Do not use any custom or empty marker for `blockinfile` module.

## Solution

The existing inventory already lists the app servers, so the play can target `all` without adding connection arguments to the validation command. The play installs `httpd`, starts and enables the service, inserts the requested page with the default `blockinfile` markers, and sets the file owner, group, and mode on each server.

### 📂 Step 1: Open the Ansible working directory

```bash
cd /home/thor/ansible/
```

> **Why:** `cd` changes the current directory to the location of the existing inventory. Running the validator from here lets `-i inventory` resolve that file and lets Ansible find `playbook.yml` by its relative name. The lab session uses the `thor` account on the Jump Host.

### 📝 Step 2: Create the playbook

```bash
vi playbook.yml
```

Enter and save this playbook:

```yaml
---
- name: Install and configure the web server on all app servers
  hosts: all
  become: true
  tasks:
    - name: Install httpd
      ansible.builtin.yum:
        name: httpd
        state: present

    - name: Ensure httpd is running and enabled
      ansible.builtin.service:
        name: httpd
        state: started
        enabled: true

    - name: Add the sample page
      ansible.builtin.blockinfile:
        path: /var/www/html/index.html
        create: true
        block: |
          Welcome to XfusionCorp!

          This is  Nautilus sample file, created using Ansible!

          Please do not modify this file manually!

    - name: Set the sample page owner and permissions
      ansible.builtin.file:
        path: /var/www/html/index.html
        owner: apache
        group: apache
        mode: '0655'
```

Save and exit `vi` with `Esc`, `:wq`, and `Enter`.

> **Why:** `vi` creates the required playbook. `hosts: all` applies it to every host in the existing inventory. `become: true` runs privileged tasks with administrative access. The `yum` module ensures the `httpd` package is present. The `service` module makes sure the service is started and enabled at boot. `blockinfile` writes the requested multiline `block` to `path`; `create: true` allows it to create the file if needed. No `marker` parameter is set, so Ansible uses its standard begin and end markers. The `file` module sets the path's `owner`, `group`, and `mode`; quoting `'0655'` preserves the leading zero in the octal permissions value.

### ✅ Step 3: Verify

```bash
ansible-playbook -i inventory playbook.yml
```

> **Why:** `ansible-playbook` runs the playbook. The `-i` option selects the existing `inventory` file in the current directory, and `playbook.yml` names the playbook to run. This is the exact command required by validation and does not need extra arguments.

The play completed successfully on all three app servers. The observed recap was:

```text
PLAY RECAP *********************************************************************
stapp01 : ok=5    changed=4    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
stapp02 : ok=5    changed=4    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
stapp03 : ok=5    changed=4    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

`failed=0` and `unreachable=0` for every host confirm that all tasks ran successfully across the app servers. The package, service, content, and ownership tasks each reported changes on all three hosts.

## Best Practices

- **Use the provided inventory.** Target `all` when the existing inventory contains exactly the app servers required for the task.
- **Keep package and service state explicit.** `state: present` ensures the package exists, while `state: started` and `enabled: true` keep the web server running and configured to start after a reboot.
- **Use the default block markers when required.** Omitting `marker` lets `blockinfile` add its standard markers and manage the content on later playbook runs.
- **Set file ownership and permissions declaratively.** The `file` module makes the desired owner, group, and mode repeatable across every managed host.
- **Quote octal modes in YAML.** Writing `'0655'` keeps the leading zero and communicates the intended file permission mode.

### 📚 Official Documentation

- [Ansible: blockinfile module](https://docs.ansible.com/projects/ansible/latest/collections/ansible/builtin/blockinfile_module.html)
- [Ansible: yum module](https://docs.ansible.com/projects/ansible/latest/collections/ansible/builtin/yum_module.html)
- [Ansible: service module](https://docs.ansible.com/projects/ansible/latest/collections/ansible/builtin/service_module.html)
- [Ansible: file module](https://docs.ansible.com/projects/ansible/latest/collections/ansible/builtin/file_module.html)
- [Ansible: Understanding privilege escalation](https://docs.ansible.com/projects/ansible/latest/playbook_guide/playbooks_privilege_escalation.html)
- [Ansible: How to build your inventory](https://docs.ansible.com/projects/ansible/latest/inventory_guide/intro_inventory.html)
