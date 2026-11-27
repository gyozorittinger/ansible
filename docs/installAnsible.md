# Install Ansible and required collections

## Overview

This guide describes how to install Ansible in an isolated
Python virtual environment on Oracle Linux 9 and 10.

Using a virtual environment allows multiple Ansible versions
to coexist without modifying the system Python or the
RPM-based Ansible installation.

### Requirements

- Oracle Linux 9 or 10 control node
- Python 3.12
- Ansible Core 2.19.x
- Internet access to PyPI and Ansible Galaxy
- SSH access to the managed hosts

## 1. Install Python

```bash
dnf install -y python3.12 python3.12-pip
```

Verify the Python version:

```bash
python3.12 --version
```

## 2. Create a virtual environment

```bash
mkdir -p /opt/ansible
/usr/bin/python3.12 -m venv /opt/ansible/2.19
```

## 3. Install Ansible Core

Upgrade pip inside the virtual environment:

```bash
/opt/ansible/2.19/bin/python -m pip install --upgrade pip
```

Install Ansible Core:

```bash
/opt/ansible/2.19/bin/python -m pip install \
  'ansible-core>=2.19,<2.20'
```

Verify the installation:

```bash
/opt/ansible/2.19/bin/ansible --version
```

## 4. Install required Ansible collections

```bash
/opt/ansible/2.19/bin/ansible-galaxy collection install community.general
/opt/ansible/2.19/bin/ansible-galaxy collection install ansible.posix
```

List installed collections:

```bash
/opt/ansible/2.19/bin/ansible-galaxy collection list
```

## 5. Using the virtual environment

### Option A: Activate the virtual environment

```bash
source /opt/ansible/2.19/bin/activate
```

Verify the active environment:

```bash
which python
python --version
which ansible
ansible --version
```

To leave the virtual environment:

```bash
deactivate
```

### Option B: Use absolute paths

Ansible can also be executed without activating
the virtual environment:

```bash
/opt/ansible/2.19/bin/ansible --version
/opt/ansible/2.19/bin/ansible-playbook --version
```

### Option C: Configure shell aliases

Add the following lines to `~/.bashrc`:

```bash
alias ansible219='/opt/ansible/2.19/bin/ansible'
alias ansible-playbook219='/opt/ansible/2.19/bin/ansible-playbook'
alias ansible-galaxy219='/opt/ansible/2.19/bin/ansible-galaxy'
```

Reload the shell configuration:

```bash
source ~/.bashrc
```

## 6. Clone the Ansible repository

```bash
git clone https://github.com/gyozorittinger/ansible.git
cd ansible
```

Configure the roles path from the repository root:

```bash
export ANSIBLE_ROLES_PATH="$PWD/roles"
```

## 7. Test connectivity

Create an inventory file containing the managed hosts.

Example:

```yaml
all:
  hosts:
    server01:
      ansible_host: 192.0.2.10
    server02:
      ansible_host: 192.0.2.11
  vars:
    ansible_user: root
    ansible_python_interpreter: /usr/bin/python3
```

Save the inventory as `inventory/test.yaml`.

Test SSH connectivity and remote Python execution:

```bash
/opt/ansible/2.19/bin/ansible \
  -i inventory/test.yaml all -m ping
```

## 8. Execute a playbook

Check the playbook syntax before execution:

```bash
/opt/ansible/2.19/bin/ansible-playbook \
  -i inventory/test.yaml \
  playbooks/example.yml --syntax-check
```

Execute the playbook:

```bash
/opt/ansible/2.19/bin/ansible-playbook \
  -i inventory/test.yaml \
  playbooks/example.yml
```

Replace the example inventory and playbook paths
with the actual files.

## 9. Update Ansible

Update Ansible Core within the selected release branch:

```bash
/opt/ansible/2.19/bin/python -m pip install --upgrade \
  'ansible-core>=2.19,<2.20'
```

Update the installed collections:

```bash
/opt/ansible/2.19/bin/ansible-galaxy collection install \
  --upgrade community.general

/opt/ansible/2.19/bin/ansible-galaxy collection install \
  --upgrade ansible.posix
```

Verify the installed version after the update:

```bash
/opt/ansible/2.19/bin/ansible --version
```

For a new Ansible major or minor release, create a
separate virtual environment instead of replacing
the existing environment.

## 10. Keep local configuration files outside Git

Local inventory files can be excluded from Git
without modifying the repository's `.gitignore` file.

From the repository root:

```bash
echo '/inventory/test.yaml' >> .git/info/exclude
```

This setting applies only to the local Git clone.

Never commit passwords, private SSH keys or other
sensitive configuration data to the repository.

## Troubleshooting

### Python compatibility error

Older Ansible versions may fail when managing
hosts running Python 3.12.

Example error:

```text
HTTPSConnection.__init__() got an unexpected
keyword argument 'cert_file'
```

Use the Python 3.12 virtual environment and
verify that the correct Ansible executable is used:

```bash
/opt/ansible/2.19/bin/ansible --version
```

### pip cannot find a compatible Ansible version

Verify that pip is running inside the correct
Python environment:

```bash
/opt/ansible/2.19/bin/python --version
/opt/ansible/2.19/bin/python -m pip --version
```

Always use the virtual environment's Python
executable when installing or updating packages.

## Notes

- Do not replace the system Python interpreter.
- Do not install Ansible into the system Python
  environment using pip.
- The RPM-based Ansible installation can coexist
  with the virtual environment.
- Ansible collections are installed into the
  configured collection paths and are not necessarily
  isolated by the Python virtual environment.
- Validate collection compatibility before upgrading.
