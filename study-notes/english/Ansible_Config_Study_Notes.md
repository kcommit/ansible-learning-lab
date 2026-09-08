# Ansible Study Notes

## Part 2: Project Directory and `ansible.cfg`

These notes explain the Ansible project configuration created inside the `automation` working directory.

## 1. Project Directory

Create and enter the project directory:

```bash
mkdir -p ~/automation
cd ~/automation
```

Confirm the current location:

```bash
pwd
```

Expected path:

```text
/home/ansibleadmin/automation
```

The planned directory structure is:

```text
automation/
├── ansible.cfg
├── inventory
└── playbooks/
```

- `ansible.cfg` contains project-specific Ansible settings.
- `inventory` will contain managed-node names, IP addresses, and groups.
- `playbooks/` will contain YAML automation playbooks.

## 2. Create `ansible.cfg`

From inside the `automation` directory:

```bash
vim ansible.cfg
```

Add the following configuration:

```ini
[defaults]
inventory = ./inventory
host_key_checking = False
remote_user = ansibleadmin
ask_pass = False

[privilege_escalation]
become_method = sudo
become_user = root
become_ask_pass = False
```

```ini
[defaults]

# Inventory file containing the managed nodes
inventory = ./inventory

# Verify managed nodes' SSH host keys for secure connections
host_key_checking = True

# Default user Ansible will use to connect to managed nodes through SSH
remote_user = ansibleadmin

# Do not ask for the SSH login password; use SSH key authentication
ask_pass = False


[privilege_escalation]

# Use sudo when privilege escalation is enabled
become_method = sudo

# Escalate privileges to the root user
become_user = root

# Do not ask for the sudo password; requires NOPASSWD sudo configuration
become_ask_pass = False
```

## 3. Configuration Syntax

The `ansible.cfg` file uses INI-style syntax.

### Section headers

A section name is written inside square brackets:

```ini
[defaults]
```

```ini
[privilege_escalation]
```

### Key-value settings

Every setting normally follows this structure:

```ini
key = value
```

Example:

```ini
remote_user = ansibleadmin
```

### Comments

A line beginning with `#` is a comment:

```ini
# This is a comment
```

Comments explain the configuration but are not executed by Ansible.

## 4. The `[defaults]` Section

The `[defaults]` section controls general Ansible behavior.

### Inventory location

```ini
inventory = ./inventory
```

This tells Ansible to use the file named `inventory` in the same project directory.

Because the inventory path is configured, this shorter command can be used:

```bash
ansible all -m ping
```

Without this setting, the inventory would need to be supplied manually:

```bash
ansible all -i inventory -m ping
```

### SSH host-key checking

```ini
host_key_checking = True
```

This tells Ansible to verify the managed node's SSH identity. It protects against connecting silently to an impersonated or unexpectedly changed host.

When connecting to a new managed node for the first time, establish trust manually:

```bash
ssh ansibleadmin@MANAGED_NODE_IP
```

Review the fingerprint and type `yes` only if the server is correct.

For a temporary private lab, host-key checking can be disabled:

```ini
host_key_checking = False
```

However, keeping it enabled is the safer and more realistic practice.

### Remote SSH user

```ini
remote_user = ansibleadmin
```

Ansible will connect to each managed node as `ansibleadmin`. The account must therefore exist on every managed node.

This is conceptually equivalent to:

```bash
ssh ansibleadmin@MANAGED_NODE_IP
```

The user can be overridden from the command line:

```bash
ansible all -m ping -u another_user
```

### SSH password prompt

```ini
ask_pass = False
```

Ansible will not ask for the SSH login password. This configuration expects SSH key authentication between the control node and managed nodes.

If password authentication must be used temporarily:

```bash
ansible all -m ping --ask-pass
```

## 5. The `[privilege_escalation]` Section

Ansible first connects to the managed node as `ansibleadmin`. A privileged task can then use `sudo` to execute as `root`.

```text
Control node
     |
     | SSH
     v
ansibleadmin on managed node
     |
     | sudo / become
     v
root privileges
```

### Become method

```ini
become_method = sudo
```

This chooses `sudo` as the method used for privilege escalation.

### Become user

```ini
become_user = root
```

When privilege escalation is requested, Ansible will execute the task as `root`.

This setting selects the target user but does not enable privilege escalation by itself.

### Become password prompt

```ini
become_ask_pass = False
```

Ansible will not request a sudo password. This requires the following type of `NOPASSWD` rule on every managed node:

```text
ansibleadmin ALL=(ALL) NOPASSWD: ALL
```

If passwordless sudo is not configured, request the become password using `-K`:

```bash
ansible all -b -K -m command -a "whoami"
```

## 6. SSH Password vs Become Password

These settings control two different authentication stages:

| Setting | Purpose |
| --- | --- |
| `ask_pass` | SSH login password for connecting to a managed node |
| `become_ask_pass` | Sudo password for privilege escalation after connecting |

In this lab:

```text
ask_pass = False
```

because SSH keys will be used, and:

```text
become_ask_pass = False
```

because `ansibleadmin` will have `NOPASSWD` sudo access.

## 7. Why `become = True` Is Not Global

The configuration intentionally does not contain:

```ini
become = True
```

This prevents every task from automatically running with root privileges. Privilege escalation should be requested only when required.

Enable it for an ad-hoc command with `-b`:

```bash
ansible all -b -m command -a "whoami"
```

Or enable it in a playbook:

```yaml
---
- name: Configure web servers
  hosts: webservers
  become: true

  tasks:
    - name: Install Apache
      ansible.builtin.dnf:
        name: httpd
        state: present
```

This method clearly shows which commands or playbooks require administrative access.

## 8. Validate the Configuration

From inside `~/automation`, check which configuration file Ansible selected:

```bash
ansible --version
```

The expected line is:

```text
config file = /home/ansibleadmin/automation/ansible.cfg
```

Display the selected configuration file:

```bash
ansible-config view
```

Show settings that differ from Ansible's defaults:

```bash
ansible-config dump --only-changed
```

After creating the inventory, display its structure:

```bash
ansible-inventory --graph
```

Test all managed nodes:

```bash
ansible all -m ping
```

## 9. Important Configuration-File Behavior

Ansible selects **only one** configuration file. It checks the following locations in order and stops as soon as it finds the first valid file:

| Priority | Location | Explanation |
| --- | --- | --- |
| 1 — Highest | `$ANSIBLE_CONFIG` | A configuration path explicitly exported as an environment variable |
| 2 | `./ansible.cfg` | Configuration file in the present working directory (`pwd`) |
| 3 | `~/.ansible.cfg` | Hidden configuration file in the current user's home directory |
| 4 — Lowest | `/etc/ansible/ansible.cfg` | System-wide default configuration file |

The preference order is therefore:

```text
$ANSIBLE_CONFIG
       ↓
./ansible.cfg in PWD
       ↓
~/.ansible.cfg
       ↓
/etc/ansible/ansible.cfg
```

### Priority 1: Exported `ANSIBLE_CONFIG` value

You can explicitly tell Ansible which configuration file to use:

```bash
export ANSIBLE_CONFIG=/home/ansibleadmin/automation/ansible.cfg
```

Check the exported value:

```bash
echo "$ANSIBLE_CONFIG"
```

This has the highest priority. While the variable is set, Ansible uses that file instead of searching the other locations.

Remove the exported value from the current shell when it is no longer required:

```bash
unset ANSIBLE_CONFIG
```

### Priority 2: Configuration in the present working directory

`pwd` means **present working directory**. Check it with:

```bash
pwd
```

If the result is:

```text
/home/ansibleadmin/automation
```

Ansible next looks for:

```text
/home/ansibleadmin/automation/ansible.cfg
```

This is the method used in our lab. A project-level `ansible.cfg` allows each project to have its own inventory and behavior.

Security note: Ansible may ignore `ansible.cfg` in the current directory if that directory is world-writable, because another user could place a malicious configuration there.

### Priority 3: Configuration in the user's home directory

The tilde `~` represents the current user's home directory:

```bash
echo "$HOME"
```

For `ansibleadmin`, this normally produces:

```text
/home/ansibleadmin
```

Therefore:

```text
~/.ansible.cfg
```

means:

```text
/home/ansibleadmin/.ansible.cfg
```

This configuration applies to the user when no higher-priority configuration file is selected.

### Priority 4: System-wide configuration

The final location is:

```text
/etc/ansible/ansible.cfg
```

This is the system-wide configuration. Ansible uses it only when none of the first three locations supplies a configuration file.

### Confirm the selected configuration

Run `ansible --version` whenever you need to confirm which configuration file is active.

```bash
ansible --version
```

Look for this line:

```text
config file = /home/ansibleadmin/automation/ansible.cfg
```

You can also use:

```bash
ansible-config view
ansible-config dump --only-changed
```

If you leave the `automation` directory and Ansible can no longer find this project configuration, return to it:

```bash
cd ~/automation
```

## Final Configuration

```ini
[defaults]
inventory = ./inventory
host_key_checking = True
remote_user = ansibleadmin
ask_pass = False

[privilege_escalation]
become_method = sudo
become_user = root
become_ask_pass = False
```

## Next Step

The next step is to prepare the managed nodes and create the `inventory` file inside `~/automation`.
