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

`hosts: stapp01` limits the play to App Server 1.

### 🧩 Step 3: Create the Jinja2 template

Create `role/httpd/templates/index.html.j2` with this line:

```text
This file was created using Ansible on {{ inventory_hostname }}
```

Ansible renders `inventory_hostname` as the current inventory name, so the template does not hard-code `stapp01`.

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

The `template` module finds `src` in the role's `templates/` directory. The inventory's `ansible_user` is the SSH connection user (`tony` for `stapp01`); privilege escalation to root does not require hard-coding that owner in the task.

### ✅ Step 5: Run the required playbook command

```bash
ansible-playbook -i inventory playbook.yml
```

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
