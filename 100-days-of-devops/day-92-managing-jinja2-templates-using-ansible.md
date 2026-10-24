# Day 92: Managing Jinja2 Templates Using Ansible

The Nautilus DevOps team has an existing `httpd` role that installs and starts Apache. Update the playbook to run the role on App Server 1, then use a Jinja2 template to deploy its web page with the required owner and permissions.

## Specific Requirements

1. Update `/home/thor/ansible/playbook.yml` to run the existing `role/httpd` on `stapp01`.
2. Create `role/httpd/templates/index.html.j2` with the server name rendered from `inventory_hostname`, not hard-coded.
3. Add a role task that deploys the template to `/var/www/html/index.html`.
4. Set the file owner and group to the server's inventory connection user and permissions to `0644`.
5. Ensure `ansible-playbook -i inventory playbook.yml` runs without extra arguments.

## Solution

### 📂 Step 1: Open the Ansible directory

```bash
cd /home/thor/ansible
```

> **Why:** The inventory is in this directory, and the lab's validation command refers to it with the relative name `inventory`. Working here also places the playbook and role edits under the required Ansible directory.

### 📝 Step 2: Target App Server 1 in the playbook

Edit `playbook.yml`:

```yaml
---
- hosts: stapp01
  become: yes
  become_user: root
  roles:
    - role/httpd
```

> **Why:** `hosts: stapp01` limits the role to App Server 1, the only target in this task. `become: yes` and `become_user: root` allow the role to install and configure a system service with administrative privileges.

### 🧩 Step 3: Create the Jinja2 template

Create `role/httpd/templates/index.html.j2` with this line:

```text
This file was created using Ansible on {{ inventory_hostname }}
```

> **Why:** Ansible renders `inventory_hostname` as the current inventory name when it processes the template. This produces the required server-specific line without hard-coding `stapp01` in the template.

### ⚙️ Step 4: Add the template task to the role

Keep the existing `yum` and `service` tasks in `role/httpd/tasks/main.yml`, then add:

```yaml
- name: Deploy the index page from a Jinja2 template
  template:
    src: index.html.j2
    dest: /var/www/html/index.html
    owner: "{{ ansible_user }}"
    group: "{{ ansible_user }}"
    mode: '0644'
```

> **Why:** The role's `templates` directory is where Ansible looks for `src`. The `template` module renders the Jinja2 variable and writes the result to `dest`. The inventory's `ansible_user` is the SSH connection user (`tony` for `stapp01`), so using it for `owner` and `group` applies the required ownership without hard-coding it. The quoted mode `'0644'` sets the requested permissions.

### ✅ Step 5: Run the required playbook command

```bash
ansible-playbook -i inventory playbook.yml
```

> **Why:** `ansible-playbook` executes the play, and `-i inventory` selects the provided hosts and connection settings. Running this exact command confirms the role works without additional arguments, as required by validation.

The observed run completed on `stapp01`: all four tasks succeeded, with `failed=0` and `unreachable=0`. The user confirmed that KodeKloud marked the lab successful.

## Best Practices

- Use `inventory_hostname` for host-specific text so one template can serve different hosts.
- Set the owner, group, and quoted file mode explicitly in the template task.
- Keep the play's host pattern narrow when a lab targets only one server.

### 📚 Official Documentation

- [Ansible roles](https://docs.ansible.com/projects/ansible/latest/playbook_guide/playbooks_reuse_roles.html)
- [Ansible `template` module](https://docs.ansible.com/projects/ansible/latest/collections/ansible/builtin/template_module.html)
- [Ansible magic variables, including `inventory_hostname`](https://docs.ansible.com/projects/ansible/latest/playbook_guide/playbooks_vars_facts.html)
- [Ansible connection variables, including `ansible_user`](https://docs.ansible.com/projects/ansible/latest/plugins/connection.html)
- [Ansible `yum` module](https://docs.ansible.com/projects/ansible/latest/collections/ansible/builtin/yum_module.html)
- [Ansible `service` module](https://docs.ansible.com/projects/ansible/latest/collections/ansible/builtin/service_module.html)
