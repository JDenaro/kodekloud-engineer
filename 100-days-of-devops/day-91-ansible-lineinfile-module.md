# Day 91: Ansible Lineinfile Module

The Nautilus DevOps team needs Apache HTTP Server installed and running on every application server in Stratos DC. Ansible must create a sample web page, add a welcome line at the beginning, and set the required owner and permissions.

## Specific Requirements

1. Create `/home/thor/ansible/playbook.yml` on the Jump Host and use the existing inventory.
2. Install `httpd` and ensure its service is running on all application servers.
3. Create `/var/www/html/index.html` with `This is a Nautilus sample file, created using Ansible!`.
4. Add `Welcome to xFusionCorp Industries!` at the beginning of the file with the `lineinfile` module.
5. Set the file owner and group to `apache`, and permissions to `0644`.
6. Make the playbook work with `ansible-playbook -i inventory playbook.yml` without extra arguments.

## Solution

The inventory lists `stapp01`, `stapp02`, and `stapp03` under `all`, so one play can configure all three hosts.

1. Open the playbook on the Jump Host:

   ```bash
   cd /home/thor/ansible
   vi playbook.yml
   ```

2. Add the following YAML and save the file:

   ```yaml
   ---
   - name: Configure Apache on all application servers
     hosts: all
     become: true

     tasks:
       - name: Install httpd
         ansible.builtin.package:
           name: httpd
           state: present

       - name: Ensure httpd is running
         ansible.builtin.service:
           name: httpd
           state: started

       - name: Create the sample web page
         ansible.builtin.copy:
           dest: /var/www/html/index.html
           content: "This is a Nautilus sample file, created using Ansible!\n"
           owner: apache
           group: apache
           mode: '0644'

       - name: Add the welcome line at the beginning
         ansible.builtin.lineinfile:
           path: /var/www/html/index.html
           line: Welcome to xFusionCorp Industries!
           insertbefore: BOF
   ```

   **Why:** `hosts: all` targets every server in this inventory. `package` installs `httpd`, `service` starts it, and `copy` creates the page with its owner, group, and mode. `lineinfile` with `insertbefore: BOF` places the welcome line at the beginning.

3. Run the command required by the lab:

   ```bash
   ansible-playbook -i inventory playbook.yml
   ```

   The observed recap reported `failed=0` for `stapp01`, `stapp02`, and `stapp03`.

## Best Practices

- Use `state: present` to ensure a package is installed and `state: started` to keep a service running.
- Quote file modes such as `'0644'` so YAML and Ansible interpret the value as intended.
- Keep task names descriptive; they make playbook output easier to follow.

### 📚 Official Documentation

- [Ansible `package` module](https://docs.ansible.com/projects/ansible/latest/collections/ansible/builtin/package_module.html)
- [Ansible `service` module](https://docs.ansible.com/projects/ansible/latest/collections/ansible/builtin/service_module.html)
- [Ansible `copy` module](https://docs.ansible.com/projects/ansible/latest/collections/ansible/builtin/copy_module.html)
- [Ansible `lineinfile` module](https://docs.ansible.com/projects/ansible/latest/collections/ansible/builtin/lineinfile_module.html)
