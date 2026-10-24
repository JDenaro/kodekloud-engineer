# Day 90: Managing ACLs Using Ansible

The Nautilus DevOps team needs to create three files on application servers in Stratos DC. Each file must be owned by `root`, while an application-specific user or group receives additional access through an ACL. Complete all file and ACL changes with Ansible from the Jump Host.

## Specific Requirements

1. Create `/home/thor/ansible/playbook.yml` on the Jump Host. The `inventory` file already exists in `/home/thor/ansible`.
2. On `stapp01`, create the empty file `/opt/security/blog.txt`, owned by `root`, and grant read permission (`r`) to group `tony`.
3. On `stapp02`, create the empty file `/opt/security/story.txt`, owned by `root`, and grant read and write permissions (`rw`) to user `steve`.
4. On `stapp03`, create the empty file `/opt/security/media.txt`, owned by `root`, and grant read and write permissions (`rw`) to group `banner`.
5. The playbook must run with `ansible-playbook -i inventory playbook.yml` without extra arguments.

## Solution

The inventory contains the three application hosts as `stapp01`, `stapp02`, and `stapp03`. The playbook uses one play per server so each file and ACL is easy to match to its target. `become: true` runs the tasks as `root`; `ansible.builtin.file` creates each file, and `ansible.posix.acl` grants the requested named ACL entry.

### 📂 Step 1: Open the Ansible working directory

On the Jump Host console as `thor`, change to the Ansible working directory:

```bash
cd /home/thor/ansible
```

> **Why:** The validation command refers to `inventory` by its relative name, so run it from `/home/thor/ansible`, where the lab provides the inventory.

### 📝 Step 2: Create the playbook

```bash
vi playbook.yml
```

Enter and save this playbook:

```yaml
---
- name: Create blog.txt on app server 1
  hosts: stapp01
  become: true
  tasks:
    - name: Create blog.txt owned by root
      ansible.builtin.file:
        path: /opt/security/blog.txt
        state: touch
        owner: root

    - name: Grant read access to group tony
      ansible.posix.acl:
        path: /opt/security/blog.txt
        entity: tony
        etype: group
        permissions: r
        state: present

- name: Create story.txt on app server 2
  hosts: stapp02
  become: true
  tasks:
    - name: Create story.txt owned by root
      ansible.builtin.file:
        path: /opt/security/story.txt
        state: touch
        owner: root

    - name: Grant read and write access to user steve
      ansible.posix.acl:
        path: /opt/security/story.txt
        entity: steve
        etype: user
        permissions: rw
        state: present

- name: Create media.txt on app server 3
  hosts: stapp03
  become: true
  tasks:
    - name: Create media.txt owned by root
      ansible.builtin.file:
        path: /opt/security/media.txt
        state: touch
        owner: root

    - name: Grant read and write access to group banner
      ansible.posix.acl:
        path: /opt/security/media.txt
        entity: banner
        etype: group
        permissions: rw
        state: present
```

Save and exit `vi` with `Esc`, `:wq`, and `Enter`.

> **Why:** Each play targets one inventory hostname, keeping the server-to-file mapping explicit. `become: true` enables privilege escalation, which uses `root` by default. `state: touch` creates the file if it does not exist, and `owner: root` sets its owner. The ACL task identifies the exact user or group with `entity` and `etype`, grants the requested permissions, and uses `state: present` to ensure the entry exists.

### ✅ Step 3: Verify

Run the required validation command from `/home/thor/ansible`:

```bash
ansible-playbook -i inventory playbook.yml
```

The playbook completed successfully on all three application servers. The observed recap was:

```text
PLAY RECAP ********************************************************************************************************************
stapp01                    : ok=3    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
stapp02                    : ok=3    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
stapp03                    : ok=3    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

The KodeKloud lab was then marked successful. Each host reported two changes: file creation/ownership and ACL configuration.

## Best Practices

- **Keep ownership and ACLs explicit.** Set the file owner to `root`, then add the app-specific ACL as a separate task.
- **Target the intended host.** Use the exact hostnames shown by `ansible-inventory -i inventory --graph`.
- **Use fully qualified module names.** `ansible.builtin.file` and `ansible.posix.acl` make the module source clear.
- **Check ACL prerequisites.** The target filesystem must support ACLs, and `setfacl` and `getfacl` must be installed on the target hosts.
- **Use the required validation command.** Running the playbook with the supplied inventory verifies the same invocation the lab uses.

### 📚 Official Documentation

- [Ansible: file module](https://docs.ansible.com/projects/ansible/latest/collections/ansible/builtin/file_module.html)
- [Ansible: POSIX ACL module](https://docs.ansible.com/projects/ansible/latest/collections/ansible/posix/acl_module.html)
- [Ansible: privilege escalation with `become`](https://docs.ansible.com/projects/ansible-core/devel/playbook_guide/playbooks_privilege_escalation.html)
- [Ansible: creating and running a playbook](https://docs.ansible.com/projects/ansible/latest/getting_started/get_started_playbook.html)
- [Ansible: building an inventory](https://docs.ansible.com/projects/ansible/latest/inventory_guide/intro_inventory.html)
