# Day 93: Using Ansible Conditionals

The Nautilus DevOps team needs to copy a different source file from the Jump Host to each application server. This playbook targets all app servers and uses Ansible's `when` conditionals to select the correct file and ownership for each host.

## Specific Requirements

1. Create `/home/thor/ansible/playbook.yml` on the Jump Host and use the existing inventory.
2. Copy `/usr/src/itadmin/blog.txt` to `/opt/itadmin/blog.txt` on `stapp01`, owned by `tony:tony` with mode `0744`.
3. Copy `/usr/src/itadmin/story.txt` to `/opt/itadmin/story.txt` on `stapp02`, owned by `steve:steve` with mode `0744`.
4. Copy `/usr/src/itadmin/media.txt` to `/opt/itadmin/media.txt` on `stapp03`, owned by `banner:banner` with mode `0744`.
5. Target all inventory hosts and use `when` conditions based on `ansible_nodename`.
6. Run the playbook with `ansible-playbook -i inventory playbook.yml` without extra arguments.

## Solution

The existing inventory contains all three application servers. One play targets `all`; it creates the destination directory if necessary, then runs only the matching file-copy task on each host. Ansible gathers facts by default, making `ansible_nodename` available to the conditions.

### 📂 Step 1: Open the Ansible working directory

On the Jump Host as `thor`, enter the directory containing the inventory:

```bash
cd /home/thor/ansible
```

> **Why:** The validation command uses the relative filename `inventory`, so running it from this directory lets Ansible find the provided inventory and the playbook at the required path.

### 📝 Step 2: Create the conditional playbook

Open `playbook.yml`:

```bash
vi playbook.yml
```

Enter and save:

```yaml
---
- name: Copy the correct file to each app server
  hosts: all
  become: true

  tasks:
    - name: Ensure the destination directory exists
      ansible.builtin.file:
        path: /opt/itadmin
        state: directory

    - name: Copy blog.txt to App Server 1
      ansible.builtin.copy:
        src: /usr/src/itadmin/blog.txt
        dest: /opt/itadmin/blog.txt
        owner: tony
        group: tony
        mode: '0744'
      when: ansible_nodename == "stapp01"

    - name: Copy story.txt to App Server 2
      ansible.builtin.copy:
        src: /usr/src/itadmin/story.txt
        dest: /opt/itadmin/story.txt
        owner: steve
        group: steve
        mode: '0744'
      when: ansible_nodename == "stapp02"

    - name: Copy media.txt to App Server 3
      ansible.builtin.copy:
        src: /usr/src/itadmin/media.txt
        dest: /opt/itadmin/media.txt
        owner: banner
        group: banner
        mode: '0744'
      when: ansible_nodename == "stapp03"
```

Save and exit `vi` with `Esc`, `:wq`, and `Enter`.

> **Why:** `hosts: all` lets Ansible evaluate these tasks on every inventory host. `become: true` grants the privileges needed to create the target directory and set file ownership. The `file` task ensures `/opt/itadmin` exists before the copies. Each `copy` task reads its `src` from the controller (the Jump Host), writes to the named `dest`, and sets the requested owner, group, and quoted octal mode. The `when` expressions compare the gathered `ansible_nodename` fact with the target server name; Ansible runs the matching task and skips the other two. Conditional expressions are written without `{{ }}`.

### ✅ Step 3: Run the required validation command

```bash
ansible-playbook -i inventory playbook.yml
```

> **Why:** `ansible-playbook` executes the automation, and `-i inventory` selects the lab's hosts and connection settings. This is the exact validation command, so the playbook must not depend on extra arguments.

The observed recap was:

```text
stapp01 : ok=3    changed=1    unreachable=0    failed=0    skipped=2
stapp02 : ok=3    changed=1    unreachable=0    failed=0    skipped=2
stapp03 : ok=3    changed=1    unreachable=0    failed=0    skipped=2
```

Each server had one copy task run and two tasks skipped. The user confirmed that KodeKloud marked the lab successful.

## Best Practices

- Target the full inventory when each host has a distinct task, then use clear `when` conditions to select the correct action.
- Keep source paths on the controller and destination paths on managed hosts explicit in `copy` tasks.
- Quote file modes such as `'0744'` so Ansible interprets them as octal permissions.

### 📚 Official Documentation

- [Ansible conditionals](https://docs.ansible.com/projects/ansible/latest/playbook_guide/playbooks_conditionals.html)
- [Ansible facts and variables](https://docs.ansible.com/projects/ansible/latest/playbook_guide/playbooks_vars_facts.html)
- [Ansible `copy` module](https://docs.ansible.com/projects/ansible/latest/collections/ansible/builtin/copy_module.html)
- [Ansible `file` module](https://docs.ansible.com/projects/ansible/latest/collections/ansible/builtin/file_module.html)
