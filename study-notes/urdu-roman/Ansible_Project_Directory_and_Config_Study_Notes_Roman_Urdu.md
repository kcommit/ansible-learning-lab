# Ansible Study Notes — Roman Urdu

## Part 2: Project Directory aur `ansible.cfg`

Yeh notes `automation` working directory ke andar banayi gayi Ansible project configuration ko explain karti hain.

## 1. Project Directory

Project directory banayein aur us ke andar jayein:

```bash
mkdir -p ~/automation
cd ~/automation
```

Apni current location confirm karein:

```bash
pwd
```

Expected path:

```text
/home/ansibleadmin/automation
```

Hamari planned directory structure:

```text
automation/
├── ansible.cfg
├── inventory
└── playbooks/
```

- `ansible.cfg` mein project-specific Ansible settings hoti hain.
- `inventory` mein managed nodes ke names, IP addresses aur groups honge.
- `playbooks/` mein YAML automation playbooks rakhe jayenge.

## 2. `ansible.cfg` Banana

`automation` directory ke andar se:

```bash
vim ansible.cfg
```

Is mein yeh configuration add karein:

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

`ansible.cfg` INI-style syntax use karta hai.

### Section headers

Section ka naam square brackets ke andar likha jata hai:

```ini
[defaults]
```

```ini
[privilege_escalation]
```

### Key-value settings

Har setting aam tor par is structure mein hoti hai:

```ini
key = value
```

Example:

```ini
remote_user = ansibleadmin
```

### Comments

`#` se shuru hone wali line comment hoti hai:

```ini
# This is a comment
```

Comments configuration ko samjhate hain, lekin Ansible unhein execute nahi karta.

## 4. `[defaults]` Section

`[defaults]` section Ansible ka general behavior control karta hai.

### Inventory location

```ini
inventory = ./inventory
```

Yeh Ansible ko batata hai ke isi project directory mein maujood `inventory` file use karni hai.

Is setting ke baad hum short command chala sakte hain:

```bash
ansible all -m ping
```

Is setting ke baghair inventory manually deni parti:

```bash
ansible all -i inventory -m ping
```

### SSH host-key checking

```ini
host_key_checking = True
```

Yeh Ansible ko managed node ki SSH identity verify karne ka kehta hai. Is se kisi fake ya unexpectedly changed server ke saath silently connection banne ka risk kam hota hai.

Naye managed node ke saath pehli dafa manually trust establish karein:

```bash
ssh ansibleadmin@MANAGED_NODE_IP
```

Fingerprint check karein aur server correct hone par hi `yes` type karein.

Temporary private lab ke liye checking disable ki ja sakti hai:

```ini
host_key_checking = False
```

Lekin `True` rakhna zyada secure aur production jaisi practice hai.

### Remote SSH user

```ini
remote_user = ansibleadmin
```

Ansible har managed node par SSH ke zariye `ansibleadmin` user se connect karega. Is liye yeh account har managed node par maujood hona chahiye.

Yeh conceptually is command jaisa hai:

```bash
ssh ansibleadmin@MANAGED_NODE_IP
```

Command line par user override bhi kiya ja sakta hai:

```bash
ansible all -m ping -u another_user
```

### SSH password prompt

```ini
ask_pass = False
```

Ansible SSH login password nahi poochega. Is configuration mein control node aur managed nodes ke darmiyan SSH key authentication expected hai.

Agar temporary password authentication use karni ho:

```bash
ansible all -m ping --ask-pass
```

## 5. `[privilege_escalation]` Section

Ansible pehle managed node par `ansibleadmin` ke taur par connect hota hai. Privileged task phir `sudo` use karke `root` ke taur par execute ho sakta hai.

```text
Control node
     |
     | SSH
     v
Managed node par ansibleadmin
     |
     | sudo / become
     v
root privileges
```

### Become method

```ini
become_method = sudo
```

Yeh privilege escalation ke liye `sudo` method select karta hai.

### Become user

```ini
become_user = root
```

Jab privilege escalation request ki jaye gi, Ansible task ko `root` ke taur par execute karega.

Yeh setting target user select karti hai, lekin apne aap privilege escalation enable nahi karti.

### Become password prompt

```ini
become_ask_pass = False
```

Ansible sudo password nahi poochega. Is ke liye har managed node par is qisam ka `NOPASSWD` rule chahiye:

```text
ansibleadmin ALL=(ALL) NOPASSWD: ALL
```

Agar passwordless sudo configured na ho to `-K` ke saath become password request karein:

```bash
ansible all -b -K -m command -a "whoami"
```

## 6. SSH Password aur Become Password ka Farq

Dono settings different authentication stages ko control karti hain:

| Setting | Maqsad |
| --- | --- |
| `ask_pass` | Managed node ke saath SSH login ka password |
| `become_ask_pass` | Connection ke baad privilege escalation ka sudo password |

Hamare lab mein:

```text
ask_pass = False
```

kyun ke SSH keys use hongi, aur:

```text
become_ask_pass = False
```

kyun ke `ansibleadmin` ke paas `NOPASSWD` sudo access hoga.

## 7. Global `become = True` Kyun Nahi Rakha?

Configuration mein jaan boojh kar yeh line nahi rakhi gayi:

```ini
become = True
```

Is se har task automatically root privileges ke saath run nahi hota. Privilege escalation sirf zaroorat par request karna behtar hai.

Ad-hoc command ke liye `-b` use karein:

```bash
ansible all -b -m command -a "whoami"
```

Ya playbook mein enable karein:

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

Is method se clearly nazar aata hai ke kaun sa command ya playbook administrative access mangta hai.

## 8. Configuration Validate Karna

`~/automation` directory ke andar se check karein ke Ansible ne kaunsi configuration file select ki:

```bash
ansible --version
```

Expected line:

```text
config file = /home/ansibleadmin/automation/ansible.cfg
```

Selected configuration file display karein:

```bash
ansible-config view
```

Sirf woh settings dikhayein jo Ansible defaults se different hain:

```bash
ansible-config dump --only-changed
```

Inventory create karne ke baad us ki structure dekhein:

```bash
ansible-inventory --graph
```

Tamam managed nodes test karein:

```bash
ansible all -m ping
```

## 9. Configuration File ka Important Behavior

Ansible ek waqt mein **sirf ek** configuration file select karta hai. Woh neeche di gayi locations ko isi order mein check karta hai aur pehli valid file milte hi searching rok deta hai:

| Priority | Location | Explanation |
| --- | --- | --- |
| 1 — Sab se zyada | `$ANSIBLE_CONFIG` | Environment variable ke zariye explicitly export kiya gaya configuration path |
| 2 | `./ansible.cfg` | Present working directory (`pwd`) ki configuration file |
| 3 | `~/.ansible.cfg` | Current user ki home directory mein hidden configuration file |
| 4 — Sab se kam | `/etc/ansible/ansible.cfg` | System-wide default configuration file |

Preference order:

```text
$ANSIBLE_CONFIG
       ↓
PWD ke andar ./ansible.cfg
       ↓
~/.ansible.cfg
       ↓
/etc/ansible/ansible.cfg
```

### Priority 1: Exported `ANSIBLE_CONFIG` value

Aap Ansible ko explicitly bata sakte hain ke kaunsi configuration file use karni hai:

```bash
export ANSIBLE_CONFIG=/home/ansibleadmin/automation/ansible.cfg
```

Exported value check karein:

```bash
echo "$ANSIBLE_CONFIG"
```

Is ki priority sab se zyada hai. Jab tak yeh variable set hai, Ansible isi file ko use karega aur baqi locations search nahi karega.

Jab exported setting ki zaroorat na rahe to current shell se remove karein:

```bash
unset ANSIBLE_CONFIG
```

### Priority 2: Present working directory ki configuration

`pwd` ka matlab **present working directory** hai. Isay check karein:

```bash
pwd
```

Agar output yeh ho:

```text
/home/ansibleadmin/automation
```

to Ansible is file ko check karega:

```text
/home/ansibleadmin/automation/ansible.cfg
```

Hum apne lab mein yehi method use kar rahe hain. Project-level `ansible.cfg` ki wajah se har project ki apni inventory aur behavior ho sakta hai.

Security note: Agar current directory world-writable ho, to Ansible us directory ki `ansible.cfg` ko ignore kar sakta hai. Is ki wajah yeh hai ke koi doosra user malicious configuration file rakh sakta hai.

### Priority 3: User ki home directory

Tilde `~` current user ki home directory ko represent karta hai:

```bash
echo "$HOME"
```

`ansibleadmin` ke liye aam tor par output hoga:

```text
/home/ansibleadmin
```

Is liye:

```text
~/.ansible.cfg
```

ka complete matlab hai:

```text
/home/ansibleadmin/.ansible.cfg
```

Yeh configuration user ke liye apply hoti hai jab koi higher-priority configuration select na hui ho.

### Priority 4: System-wide configuration

Akhri location hai:

```text
/etc/ansible/ansible.cfg
```

Yeh poore system ki configuration hai. Ansible isay tab use karta hai jab pehli teen locations se koi configuration file select na ho.

### Active configuration confirm karein

Yeh confirm karne ke liye ke kaunsi configuration active hai, run karein:

```bash
ansible --version
```

Output mein is line ko dekhein:

```text
config file = /home/ansibleadmin/automation/ansible.cfg
```

Yeh commands bhi use kar sakte hain:

```bash
ansible-config view
ansible-config dump --only-changed
```

Agar aap `automation` directory se bahar chale jayein aur Ansible ko project configuration na mile, to wapas directory mein aayein:

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

## Agla Step

Ab hum managed nodes prepare karenge aur `~/automation` directory ke andar `inventory` file banayenge.
