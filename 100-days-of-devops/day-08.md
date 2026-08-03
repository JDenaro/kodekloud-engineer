# Day 08: Install Ansible

During the weekly meeting, the Nautilus DevOps team discussed the automation and configuration management solutions that they wanted to implement. While considering several options, the team decided to go with Ansible for now due to its simple setup and minimal prerequisites. The team wanted to start testing using Ansible, so they decided to use the Jump Host as an Ansible controller to test different kinds of tasks on the rest of the servers.

Install ansible version `4.9.0` on the Jump Host using `pip3` only. Make sure the Ansible binary is available globally on this system, i.e. all users on this system are able to run Ansible commands.

## Specific Requirements:

1. Install Ansible version `4.9.0` on the Jump Host.
2. Use `pip3` as the installation method.
3. Make the Ansible binary available globally to all users.

## Solution

The Jump Host acts as the Ansible control node. Ansible is installed globally with `sudo pip3`, which places the package and its command-line binaries in the system Python installation rather than only in `thor`'s home directory.

The `ansible` package version `4.9.0` installs a compatible `ansible-core` version as a dependency. In this lab, the resulting core version was `2.11.12`.

### 📦 Step 1: Install Ansible globally with pip3

Run the command directly from `thor@jump-host`:

```bash
sudo pip3 install ansible==4.9.0
```

The installation completed successfully:

```text
Collecting ansible==4.9.0
Collecting ansible-core<2.12,>=2.11.6
Installing collected packages: pycparser, MarkupSafe, cffi, resolvelib, PyYAML, packaging, jinja2, cryptography, ansible-core, ansible
Successfully installed MarkupSafe-3.0.3 PyYAML-6.0.3 ansible-4.9.0 ansible-core-2.11.12 cffi-2.0.0 cryptography-49.0.0 jinja2-3.1.6 packaging-26.2 pycparser-2.23 resolvelib-0.5.4
```

> **Why:** `sudo` gives the installation administrative privileges and makes the package available to all users through the system Python environment. `pip3` is Python 3's package installer. `install` requests a package installation, and `ansible==4.9.0` pins the exact Ansible community package version required by the challenge. The dependency constraint `ansible-core<2.12,>=2.11.6` resolved to `ansible-core-2.11.12`, which provides the Ansible runtime.

The installation also added supporting Python packages such as `Jinja2` for templating, `PyYAML` for YAML parsing, `cryptography` for cryptographic operations, `packaging` for version handling, and `resolvelib` for dependency resolution. These were installed automatically because Ansible requires them.

The pip warning about running as `root` is expected in this lab because the requirement is a global system installation. In production, a virtual environment is usually safer because it avoids conflicts with the operating system's Python packages.

### ✅ Step 2: Confirm the globally available Ansible binary

```bash
ansible --version
```

The command returned:

```text
ansible [core 2.11.12]
  config file = None
  configured module search path = ['/home/thor/.ansible/plugins/modules', '/usr/share/ansible/plugins/modules']
  ansible collection location = /home/thor/.ansible/collections:/usr/share/ansible/collections
  executable location = /usr/local/bin/ansible
  python version = 3.9.19
```

> **Why:** `ansible --version` prints the installed Ansible core version, Python environment, module paths, collection paths, and executable location. The path `/usr/local/bin/ansible` confirms that the binary was installed in a system-wide location rather than only under `/home/thor`. The command displays `core 2.11.12` because Ansible `4.9.0` is the community package version and uses `ansible-core` as its runtime.

## Best Practices

- **Pin the requested version.** `ansible==4.9.0` prevents pip from selecting a different Ansible release.
- **Install globally only when required.** `sudo pip3` satisfies this lab's requirement that every user can run the command, but virtual environments are safer for isolated production applications.
- **Keep the control node lightweight.** Ansible does not require an agent or daemon on managed nodes; the control node connects to them remotely, normally over SSH.
- **Check the executable path.** `/usr/local/bin/ansible` demonstrates that the command is available system-wide.
- **Distinguish Ansible from ansible-core.** The `ansible` package includes the runtime and curated collections, while `ansible-core` is the smaller runtime and module base underneath it.
- **Treat root pip warnings seriously outside the lab.** A global pip installation can conflict with packages managed by the operating system.

### 📚 Official Documentation

- [Installing Ansible](https://docs.ansible.com/projects/ansible/4/installation_guide/intro_installation.html)
- [Installing Ansible with pip](https://docs.ansible.com/projects/ansible/4/installation_guide/intro_installation.html#installing-ansible-with-pip)
- [pip install command](https://pip.pypa.io/en/stable/cli/pip_install/)
