# Ansible Study Notes

## Part 1: Control Node Setup on Rocky Linux 9

These notes document the complete setup completed on the Ansible control node, from checking the operating system through validating Ansible locally.

## Lab Environment

| Item | Value |
| --- | --- |
| Role | Ansible control node |
| Hostname | `ansible-server` |
| Operating system | Rocky Linux 9.8 (Blue Onyx) |
| Architecture | `x86_64` |
| Virtualization | Xen |
| Administrative user | `ansibleadmin` |
| Python | Python 3.9.25 |
| Python path | `/usr/bin/python3` |
| Ansible community package | 7.7.0 |
| Ansible Core | 2.14.18 |

## 1. Ansible: Agent-Based vs Agentless

An **agent-based** automation tool requires special agent software to be installed and continuously available on every managed server.

Ansible is normally **agentless**. It does not require an Ansible agent or daemon on Linux managed nodes. The control node usually connects to managed nodes through SSH, transfers the required module, runs it, receives the result, and disconnects.

```text
Ansible control node -- SSH --> Managed Linux node
```

A Linux managed node normally needs:

- An SSH server
- Python 3 for most Ansible modules
- A remote user with the required permissions
- `sudo` access when a task needs privilege escalation

Only the control node needs the Ansible package installed.

### Why use Ansible?

Ansible is useful because it turns repeated administrative work into consistent, reusable automation.

- **Faster configuration and deployment:** The same task can be performed across many servers without configuring each one manually. This can reduce deployment time and help deliver changes faster.
- **Consistency:** Playbooks apply the same desired configuration to every selected host, reducing manual errors, missing packages, incorrect permissions, and configuration drift.
- **Scalability:** A command or playbook can target one host, a group of hosts, several groups, or the complete inventory.
- **Agentless operation:** Linux managed nodes normally do not need a permanent Ansible agent. Existing SSH access and Python are used for most tasks.
- **Repeatability:** Playbooks preserve the procedure as code, so the same work can be reviewed, reused, and stored in Git.
- **Readable automation:** Playbooks use YAML, which is generally easier to read than a long collection of manual shell commands.

### Push and pull models

In a **push model**, the control system initiates a connection and sends the required configuration to managed nodes. Ansible primarily uses this model: the control node connects to the inventory hosts and runs the requested tasks.

In a **pull model**, each managed node periodically contacts a central service or repository and retrieves its configuration.

Ansible is agentless and primarily push-based, but it also supports a pull-style workflow through `ansible-pull`. Therefore, saying that Ansible is *only* push-based would be incomplete.

### Idempotency

**Idempotency** means that running the same properly written task repeatedly should not keep making unnecessary changes. Ansible compares the current state with the requested state and changes the system only when required.

For example, if a playbook says that `httpd` must be installed and it is already installed, a suitable Ansible module normally reports `ok` instead of reinstalling it. Not every shell command is automatically idempotent, so purpose-built Ansible modules should be preferred when available.

### Common Ansible use cases

- Installing, updating, and removing packages
- Creating users and groups
- Managing files, templates, ownership, and permissions
- Starting, stopping, enabling, and restarting services
- Applying operating-system patches
- Deploying applications and configuration changes
- Provisioning cloud or infrastructure resources
- Performing repeatable compliance and validation tasks

### Ansible, Puppet, and Chef — common operating models

| Tool | Common/default model | Agent on managed node | Main configuration style |
| --- | --- | --- | --- |
| Ansible | Push by default; pull is available with `ansible-pull` | No permanent Ansible agent for normal Linux management | YAML playbooks |
| Puppet | Commonly agent-based pull from a Puppet server | Usually Puppet Agent | Puppet DSL |
| Chef | Commonly client-based pull from a Chef server | Usually Chef Infra Client | Ruby-based DSL |

These are the common architectures, not absolute limitations. Each product may provide additional components or workflows. The right choice depends on the environment, existing skills, scale, security requirements, and desired operating model.

## 2. Inspect the Server

Display the hostname, operating system, kernel, architecture, and virtualization information:

```bash
hostnamectl
```

The lab server was identified as Rocky Linux 9.8 and selected as the Ansible control node.

## 3. Create a Dedicated Administrative User

Routine Ansible work should not be performed directly as `root`. A dedicated user named `ansibleadmin` was created.

Run these commands as `root`.

### Create the user

```bash
useradd -m -s /bin/bash ansibleadmin
```

Options used:

- `-m`: creates the user's home directory
- `-s /bin/bash`: assigns Bash as the login shell

### Set a password

```bash
passwd ansibleadmin
```

Use a strong password of at least eight characters. If `passwd` reports `BAD PASSWORD` but then reports that all authentication tokens were updated successfully, the password was accepted despite the warning. A stronger password should still be assigned.

### Add the user to the wheel group

```bash
usermod -aG wheel ansibleadmin
```

Options used:

- `-a`: appends the user without removing existing supplementary groups
- `-G`: specifies supplementary groups
- `wheel`: the standard administrative group on Rocky Linux

### Verify the account

```bash
id ansibleadmin
```

Observed result:

```text
uid=1001(ansibleadmin) gid=1001(ansibleadmin) groups=1001(ansibleadmin),10(wheel)
```

The presence of `10(wheel)` confirms membership in the wheel group.

## 4. Test Normal Sudo Access

Switch to the new account:

```bash
su - ansibleadmin
```

Test sudo:

```bash
sudo whoami
```

Expected result after entering the user's password:

```text
root
```

This does not mean the login user became root permanently. It means only the `whoami` command was executed with root privileges.

## 5. Configure Passwordless Sudo for the Lab

Passwordless sudo is convenient for a controlled Ansible lab because automated tasks cannot stop to enter a password interactively.

Run the following commands as `root`:

```bash
echo "ansibleadmin ALL=(ALL) NOPASSWD: ALL" > /etc/sudoers.d/ansibleadmin
chmod 440 /etc/sudoers.d/ansibleadmin
visudo -cf /etc/sudoers.d/ansibleadmin
```

Meaning of the sudo rule:

| Field | Meaning |
| --- | --- |
| `ansibleadmin` | User to whom the rule applies |
| `ALL=(ALL)` | May run commands on all hosts as any user |
| `NOPASSWD:` | Do not request a sudo password |
| `ALL` | May run all commands |

The `440` permission allows the owner and group to read the file but prevents modification through normal access.

Expected validation:

```text
/etc/sudoers.d/ansibleadmin: parsed OK
```

### Important redirection lesson

This command cannot be run directly by an ordinary user:

```bash
echo "ansibleadmin ALL=(ALL) NOPASSWD: ALL" > /etc/sudoers.d/ansibleadmin
```

The shell performs the `>` redirection, and the ordinary user cannot write inside `/etc/sudoers.d`. In this lab, the correct solution was to run the complete command from the root shell.

### Test passwordless sudo

```bash
sudo -n whoami
```

Expected result without a password prompt:

```text
root
```

The `-n` option means **non-interactive**. It tells sudo not to ask for a password. If passwordless sudo is unavailable, the command fails immediately with `sudo: a password is required`.

## 6. Check Python

Ansible uses Python 3 to execute most modules on Linux systems.

```bash
python3 --version
which python3
```

Observed results:

```text
Python 3.9.25
/usr/bin/python3
```

On modern Rocky Linux, use the explicit `python3` command. The command `python` might not exist or could be mapped differently.

If Python 3 is missing:

```bash
sudo dnf install python3 -y
```

## 7. Update Rocky Linux

For this lab control node, update all installed packages:

```bash
sudo dnf update -y
```

The update installed a newer kernel. Reboot to start using it:

```bash
sudo reboot
```

Reconnect and verify the active kernel:

```bash
uname -r
```

In production, first review updates and perform patching within an approved maintenance window:

```bash
sudo dnf check-update
```

## 8. Enable and Verify EPEL

Rocky Linux obtains the full Ansible community package from EPEL, the Extra Packages for Enterprise Linux repository.

```bash
sudo dnf install epel-release -y
```

If EPEL is already installed, DNF may report:

```text
Package epel-release is already installed.
Nothing to do.
Complete!
```

Verify enabled EPEL repositories:

```bash
sudo dnf repolist | grep -i epel
```

Observed repositories:

```text
epel                Extra Packages for Enterprise Linux 9 - x86_64
epel-cisco-openh264 Extra Packages for Enterprise Linux 9 openh264 (From Cisco) - x86_64
```

## 9. Inspect and Install Ansible

View the available package before installing it:

```bash
sudo dnf info ansible
```

The lab found Ansible 7.7.0 in EPEL.

Install the full Ansible package:

```bash
sudo dnf install ansible -y
```

The installation also installed `ansible-core` and required dependencies such as Python libraries, `git-core`, and `sshpass`.

### Ansible package vs Ansible Core

- `ansible-core` is the main automation engine, command-line programs, and built-in functionality.
- `ansible` is the larger community package containing Ansible Core plus a curated collection set.

Therefore, these different version numbers are normal:

```text
ansible package: 7.7.0
ansible-core:     2.14.18
```

## 10. Verify the Installation

```bash
ansible --version
ansible-playbook --version
```

Important values observed:

```text
ansible [core 2.14.18]
config file = /etc/ansible/ansible.cfg
ansible python module location = /usr/lib/python3.9/site-packages/ansible
executable location = /usr/bin/ansible
python version = 3.9.25
```

`ansible` runs ad-hoc tasks, while `ansible-playbook` runs reusable YAML playbooks.

## 11. Perform a Local Ansible Test

Test the Ansible ping module against the control node itself:

```bash
ansible localhost -m ping
```

Observed result:

```text
localhost | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

This is not an ICMP network ping. The test confirms that Ansible can load and execute its Python-based `ping` module and return a result.

Run the `whoami` command through Ansible:

```bash
ansible localhost -m command -a "whoami"
```

Observed result:

```text
localhost | CHANGED | rc=0 >>
ansibleadmin
```

Command breakdown:

| Part | Meaning |
| --- | --- |
| `ansible` | Runs an ad-hoc Ansible task |
| `localhost` | Target host |
| `-m command` | Uses the command module |
| `-a "whoami"` | Passes `whoami` as the module argument |
| `rc=0` | Successful command exit status |
| `ansibleadmin` | Account under which the command ran |

The command module commonly reports `CHANGED` whenever it executes a command, even if a read-only command such as `whoami` did not modify the system. This behavior can later be controlled with `changed_when` in a playbook.

## 12. Uninstall Ansible if Required

Remove the full Ansible package:

```bash
sudo dnf remove ansible -y
```

Check for remaining Ansible packages:

```bash
rpm -qa | grep -i ansible
```

If required, remove Ansible Core separately:

```bash
sudo dnf remove ansible-core -y
```

Do not remove EPEL merely because Ansible was removed; other software may use that repository. Review the proposed package list carefully before using `dnf autoremove`.

## Final Status

The control node installation is complete and validated:

- Dedicated `ansibleadmin` account created
- Wheel membership verified
- Passwordless sudo configured and validated
- Rocky Linux updated and rebooted
- Python 3 verified
- EPEL enabled
- Full Ansible package installed
- `ansible` and `ansible-playbook` verified
- Local ping and command-module tests passed

## Next Stage

The next part of the lab will cover:

1. Preparing managed nodes
2. Creating the remote administrative account
3. Generating and distributing SSH keys
4. Creating an Ansible inventory
5. Running `ansible all -m ping`
