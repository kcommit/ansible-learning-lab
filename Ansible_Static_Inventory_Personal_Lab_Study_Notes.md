# Ansible Study Notes

## Part 3: Static Inventory 

These notes document the static Ansible inventory for  three-node managed environment.

## 1. Lab Nodes

| Inventory alias | Intended role | Private IP address |
| --- | --- | --- |
| `web-server` | Web server | `192.168.1.154` |
| `app-server` | Application server | `192.168.1.11` |
| `db-server` | Database server | `192.168.1.215` |

These are private LAN addresses. They are reachable only within my local network unless routing or remote access has been configured separately.

## 2. Create the Inventory File

My `ansible.cfg` contains:

```ini
[defaults]
inventory = ./inventory
```

Therefore, I will create the inventory file inside my `automation` project directory:

```bash
cd ~/automation
vim inventory
```

## 3. Final Inventory

```ini
[web]
web-server ansible_host=192.168.1.154

[app]
app-server ansible_host=192.168.1.11

[db]
db-server ansible_host=192.168.1.215

[three_tier_app:children]
web
app
db

[all:vars]
ansible_user=ansibleadmin
ansible_python_interpreter=/usr/bin/python3

# Optional:
# ansible_ssh_private_key_file=/home/ansibleadmin/.ssh/id_ed25519
```

```ini
# ============================================
# Ansible Static Inventory — INI Format
# ============================================

# Web server group
[web]
web-server ansible_host=192.168.1.154

# Application server group
[app]
app-server ansible_host=192.168.1.11

# Database server group
[db]
db-server ansible_host=192.168.1.215


# Parent group containing all three child groups
[three_tier_app:children]
web
app
db


# Variables inherited by every inventory host
[all:vars]

# SSH user on the managed nodes
ansible_user=ansibleadmin

# Python interpreter used to execute Ansible modules
ansible_python_interpreter=/usr/bin/python3

# Optional: explicitly specify the SSH private key
# Uncomment only if Ansible does not automatically select the key
# ansible_ssh_private_key_file=/home/ansibleadmin/.ssh/id_ed25519
```

## 4. Inventory File Syntax

This is an INI-format static inventory.

### Comments

A line beginning with `#` is a comment:

```ini
# Web server group
```

Comments are ignored by Ansible and are used to document the file.

### Group headers

A name inside square brackets defines a group:

```ini
[web]
```

Hosts listed below it become members of that group.

### Host lines

```ini
web-server ansible_host=192.168.1.154
```

This line contains two important parts:

| Part | Meaning |
| --- | --- |
| `web-server` | Inventory hostname or alias used in Ansible commands |
| `ansible_host=192.168.1.154` | Actual IP address used for the connection |

The inventory alias does not have to match the managed node's Linux hostname or DNS name.

For example, I can target the alias directly:

```bash
ansible web-server -m ping
```

### Equal-sign spacing in an INI inventory

The safest and most consistent inventory style is to write variable assignments with **no spaces around the equal sign**:

```ini
variable=value
```

Correct examples:

```ini
web-server ansible_host=192.168.1.154
ansible_user=ansibleadmin
ansible_python_interpreter=/usr/bin/python3
ansible_ssh_private_key_file=/home/ansibleadmin/.ssh/id_ed25519
```

I should avoid writing host-line variables like this:

```ini
web-server ansible_host = 192.168.1.154
```

On a host line, spaces separate the inventory hostname and its inline variables. Adding spaces around `=` may split one assignment into multiple fields and cause incorrect parsing or an inventory error.

Although some INI parsers may tolerate spaces in certain section-based assignments, using `key=value` consistently prevents ambiguity and matches standard Ansible inventory examples.

This differs from `ansible.cfg`, where spaces around `=` are normal and readable:

```ini
[defaults]
inventory = ./inventory
remote_user = ansibleadmin
```

Easy rule:

```text
ansible.cfg  → spaces around = are fine
inventory    → use key=value without spaces
```

## 5. Functional Groups

The three managed nodes are organized according to their intended roles:

```ini
[web]
web-server ansible_host=192.168.1.154
```

```ini
[app]
app-server ansible_host=192.168.1.11
```

```ini
[db]
db-server ansible_host=192.168.1.215
```

This allows me to target a specific role:

```bash
ansible web -m ping
ansible app -m ping
ansible db -m ping
```

The group name targets every host inside that group.

### Compact numeric IP ranges

When many managed nodes have consecutive IP addresses, Ansible can expand a numeric range instead of requiring every address to be written separately:

```ini
[database]
192.168.10.[1:20]
```

This creates 20 inventory hosts, starting with:

```text
192.168.10.1
192.168.10.2
192.168.10.3
...
192.168.10.20
```

The starting and ending values are both included. This is a compact inventory definition; it does not scan the subnet or verify that the addresses are active. When targeted, Ansible attempts to connect to every expanded address.

This syntax is useful only when the addresses form a genuine sequence. My current nodes—`192.168.1.154`, `192.168.1.11`, and `192.168.1.215`—are not consecutive, so listing them separately with meaningful aliases is better.

### Compact hostname ranges

When managed-node hostnames follow a numbered sequence, I can use:

```ini
[sandbox]
server[1:9].nehraclasses.local
```

This expands to:

```text
server1.nehraclasses.local
server2.nehraclasses.local
server3.nehraclasses.local
...
server9.nehraclasses.local
```

These names must resolve through DNS or another name-resolution source such as `/etc/hosts`. A hostname pattern generates inventory names; it does not create DNS records or Linux servers.

Leading zeros can be preserved when required:

```ini
[sandbox]
server[01:09].example.local
```

This produces `server01.example.local` through `server09.example.local`.

An optional third number defines the stride, or increment:

```ini
[odd_web]
web[01:09:2].example.local
```

This produces `web01`, `web03`, `web05`, `web07`, and `web09`.

After adding a range, inspect its expansion:

```bash
ansible-inventory --graph
ansible database --list-hosts
ansible sandbox --list-hosts
```

Range syntax is an optional scaling technique and should not be added to my active inventory unless those hosts actually exist.

## 6. Parent and Child Groups

```ini
[three_tier_app:children]
web
app
db
```

The `:children` modifier means that the entries below the header are group names—not individual hosts.

The parent group `three_tier_app` contains all hosts inherited from the `web`, `app`, and `db` child groups.

I can therefore target the complete environment:

```bash
ansible three_tier_app -m ping
```

Ansible also provides the built-in `all` group, which automatically includes every inventory host:

```bash
ansible all -m ping
```

## 7. Variables Shared by All Hosts

```ini
[all:vars]
```

Variables under this section apply to every host in the inventory.

The variables beginning with `ansible_` are special connection or behavioral variables understood by Ansible. They describe **how Ansible should reach or operate on a host**; they do not create Linux users, install Python, or generate SSH keys.

| Variable | Purpose | Location being described |
| --- | --- | --- |
| `ansible_host` | Real IP address or DNS name used to reach a managed node | Managed node |
| `ansible_user` | User used for the SSH login | Managed node account |
| `ansible_python_interpreter` | Python executable used to run modules | Managed node |
| `ansible_ssh_private_key_file` | Private key used to authenticate the SSH connection | Control node |

The distinction between the control-node and managed-node paths is important. The Python path must exist on the remote managed node, while the private-key path must exist on the Ansible control node.

### Actual host address

```ini
web-server ansible_host=192.168.1.154
```

`web-server` is the friendly inventory alias. `ansible_host` supplies the actual network destination used by SSH.

This means:

```bash
ansible web-server -m ping
```

connects to:

```text
192.168.1.154
```

`ansible_host` can contain either an IP address or a resolvable DNS name. It does not change the Linux hostname of the managed node.

### SSH connection user

```ini
ansible_user=ansibleadmin
```

Ansible connects to every managed node as `ansibleadmin`.

This is conceptually similar to:

```bash
ssh ansibleadmin@192.168.1.154
```

The `ansibleadmin` account must exist on every managed node.

My `ansible.cfg` already contains:

```ini
remote_user = ansibleadmin
```

Therefore, `ansible_user=ansibleadmin` is technically redundant in this inventory. I am keeping it temporarily so that the effect of `[all:vars]` is clear. An inventory variable can also override the general configuration when required.

### Python interpreter

```ini
ansible_python_interpreter=/usr/bin/python3
```

This tells Ansible to use `/usr/bin/python3` for Python-based modules on every managed node.

The path `/usr/bin/python3` is checked on each **managed node**, not on the control node. If different hosts have Python in different locations, I can define this variable separately on their host lines or in group-specific variables.

The correct spelling is:

```text
interpreter
```

This spelling is incorrect:

```text
interpretor
```

Before relying on this setting, I should verify the Python path on every managed node:

```bash
which python3
python3 --version
```

### SSH private-key file

The optional setting is:

```ini
ansible_ssh_private_key_file=/home/ansibleadmin/.ssh/id_ed25519
```

This is a path on the **Ansible control node** because the control node uses its private key to prove its identity during SSH authentication. This private key must never be copied to managed nodes or committed to Git.

A `.pem` key is commonly used for cloud instances such as AWS EC2. My local Rocky Linux lab will normally use an OpenSSH key such as `id_ed25519`.

When a key exists in a standard SSH location, OpenSSH may select it automatically. In that case, this inventory variable is unnecessary.

I should uncomment the setting when:

- I use a non-standard key path.
- I maintain multiple SSH keys.
- SSH does not automatically select the correct key.

Using the full absolute path is clearer than relying on `~` expansion inside configuration files.

### Inspect the variables Ansible parsed

After saving the inventory, I can display the variables Ansible associated with one host:

```bash
ansible-inventory --host web-server
```

The result should include values such as:

```json
{
    "ansible_host": "192.168.1.154",
    "ansible_python_interpreter": "/usr/bin/python3",
    "ansible_user": "ansibleadmin"
}
```

If the private-key line remains commented, `ansible_ssh_private_key_file` will not appear in the parsed host variables.

## 8. Validate Basic Network Connectivity

From the control node, check whether the IP addresses are reachable:

```bash
ping -c 2 192.168.1.154
ping -c 2 192.168.1.11
ping -c 2 192.168.1.215
```

A failed network ping does not always prove the server is unavailable because a firewall may block ICMP. SSH connectivity is the more relevant Ansible test.

## 9. Validate the Inventory

Enter the project directory first:

```bash
cd ~/automation
```

Confirm that Ansible selected the project configuration:

```bash
ansible --version
```

The expected configuration path is:

```text
/home/ansibleadmin/automation/ansible.cfg
```

Display the inventory hierarchy:

```bash
ansible-inventory --graph
```

Expected general structure:

```text
@all:
  |--@three_tier_app:
  |  |--@app:
  |  |  |--app-server
  |  |--@db:
  |  |  |--db-server
  |  |--@web:
  |  |  |--web-server
```

Display complete parsed inventory data:

```bash
ansible-inventory --list
```

List the inventory hosts without running a module:

```bash
ansible all --list-hosts
```

Expected host list:

```text
hosts (3):
  app-server
  db-server
  web-server
```

## 10. Connectivity and Group Tests

These commands require the managed nodes and SSH authentication to be prepared.

### Test one inventory host

```bash
ansible web-server -m ping
```

### Test one group

```bash
ansible web -m ping
```

### Test the parent group

```bash
ansible three_tier_app -m ping
```

### Test every inventory host

```bash
ansible all -m ping
```

The Ansible `ping` module is not an ICMP ping. It tests whether Ansible can connect, find a usable Python interpreter, execute its module, and receive a successful response.

## 11. Test Privilege Escalation

Run `whoami` without privilege escalation:

```bash
ansible all -m command -a "whoami"
```

Expected remote user:

```text
ansibleadmin
```

Run it with `become` using `-b`:

```bash
ansible all -b -m command -a "whoami"
```

Expected privileged user:

```text
root
```

This requires passwordless sudo or a become-password workflow on each managed node.

## 12. Requirements Before the Ping Test

Every managed node must have:

- A running SSH service
- The `ansibleadmin` account
- Python 3 at `/usr/bin/python3`
- Appropriate sudo permissions
- The control node's SSH public key in the remote user's `authorized_keys`

Ansible itself remains installed only on the control node.

## Next Step

Prepare `ansibleadmin` on all three managed nodes and configure SSH key-based authentication from the control node.
