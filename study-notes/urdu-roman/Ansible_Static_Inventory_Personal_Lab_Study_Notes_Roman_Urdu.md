# Ansible Study Notes — Roman Urdu

## Part 3: Static Inventory

Yeh notes three-node Ansible managed environment ki static inventory ko document karti hain.

## 1. Lab Nodes

| Inventory alias | Intended role | Private IP address |
| --- | --- | --- |
| `web-server` | Web server | `192.168.1.154` |
| `app-server` | Application server | `192.168.1.11` |
| `db-server` | Database server | `192.168.1.215` |

Yeh private LAN addresses hain. Yeh meri local network ke andar reachable hain jab tak separate routing ya remote access configure na ki jaye.

## 2. Inventory File Banana

Meri `ansible.cfg` mein yeh setting hai:

```ini
[defaults]
inventory = ./inventory
```

Is liye main `automation` project directory ke andar inventory file banaunga:

```bash
cd ~/automation
vim inventory
```

## 3. Final Inventory

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

Yeh INI-format static inventory hai.

### Comments

`#` se shuru hone wali line comment hoti hai:

```ini
# Web server group
```

Ansible comments ko ignore karta hai. Comments file ko explain aur document karne ke liye use hote hain.

### Group headers

Square brackets ke andar naam ek group define karta hai:

```ini
[web]
```

Is ke neeche listed hosts us group ke members ban jate hain.

### Host lines

```ini
web-server ansible_host=192.168.1.154
```

Is line ke do important parts hain:

| Part | Matlab |
| --- | --- |
| `web-server` | Ansible commands mein use hone wala inventory hostname ya alias |
| `ansible_host=192.168.1.154` | Connection ke liye use hone wala actual IP address |

Inventory alias ka managed node ke actual Linux hostname ya DNS name ke saath match karna zaroori nahi.

Main alias ko directly target kar sakta hoon:

```bash
ansible web-server -m ping
```

### Equal sign ke aas paas spaces

Ansible INI inventory mein safest aur consistent style yeh hai ke `=` ke aas paas spaces na lagayein:

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

Host line par yeh style avoid karna chahiye:

```ini
web-server ansible_host = 192.168.1.154
```

Host line mein spaces inventory hostname aur inline variables ko separate karti hain. `=` ke aas paas spaces lagane se ek assignment multiple fields mein toot sakta hai, jis se parsing galat ho sakti hai ya inventory error aa sakta hai.

Kuch INI parsers section-based assignments mein spaces tolerate kar sakte hain, lekin har jagah `key=value` use karna ambiguity se bachata hai.

Yeh `ansible.cfg` se different hai. `ansible.cfg` mein spaces bilkul theek aur readable hain:

```ini
[defaults]
inventory = ./inventory
remote_user = ansibleadmin
```

Easy rule:

```text
ansible.cfg  → = ke aas paas spaces theek hain
inventory    → key=value without spaces use karein
```

## 5. Functional Groups

Teen managed nodes ko un ke intended roles ke mutabiq organize kiya gaya hai:

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

Main kisi specific role ko target kar sakta hoon:

```bash
ansible web -m ping
ansible app -m ping
ansible db -m ping
```

Group name us group ke tamam hosts ko target karta hai.

### Compact numeric IP ranges

Agar bohat se managed nodes ke IP addresses consecutive hon, to har address separately likhne ke bajaye Ansible numeric range expand kar sakta hai:

```ini
[database]
192.168.10.[1:20]
```

Yeh 20 inventory hosts banata hai:

```text
192.168.10.1
192.168.10.2
192.168.10.3
...
192.168.10.20
```

Starting aur ending dono values include hoti hain. Yeh sirf compact inventory definition hai; yeh subnet scan nahi karta aur yeh verify nahi karta ke addresses active hain. Group target karne par Ansible har expanded address ke saath connection try karega.

Yeh syntax tab useful hai jab addresses real sequence mein hon. Mere current nodes—`192.168.1.154`, `192.168.1.11`, aur `192.168.1.215`—consecutive nahi hain. Is liye unhein meaningful aliases ke saath separately list karna behtar hai.

### Compact hostname ranges

Agar managed-node hostnames numbered sequence follow karte hon, to main use kar sakta hoon:

```ini
[sandbox]
server[1:9].nitclasses.local
```

Yeh expand hoga:

```text
server1.nitclasses.local
server2.nitclasses.local
server3.nitclasses.local
...
server9.nitclasses.local
```

Yeh names DNS ya `/etc/hosts` jaisi kisi name-resolution source se resolve hone chahiye. Hostname pattern sirf inventory names generate karta hai; yeh DNS records ya Linux servers create nahi karta.

Agar leading zeros chahiye hon:

```ini
[sandbox]
server[01:09].example.local
```

Yeh `server01.example.local` se `server09.example.local` tak names banata hai.

Optional teesra number stride yani increment define karta hai:

```ini
[odd_web]
web[01:09:2].example.local
```

Yeh `web01`, `web03`, `web05`, `web07`, aur `web09` generate karta hai.

Range add karne ke baad expansion check karein:

```bash
ansible-inventory --graph
ansible database --list-hosts
ansible sandbox --list-hosts
```

Range syntax optional scaling technique hai. Isay active inventory mein tabhi add karna chahiye jab woh hosts actually exist karte hon.

## 6. Parent aur Child Groups

```ini
[three_tier_app:children]
web
app
db
```

`:children` modifier ka matlab hai ke header ke neeche entries individual hosts nahi, balke group names hain.

Parent group `three_tier_app`, `web`, `app` aur `db` child groups ke tamam hosts ko inherit karta hai.

Main complete environment ko target kar sakta hoon:

```bash
ansible three_tier_app -m ping
```

Ansible ka built-in `all` group automatically har inventory host ko include karta hai:

```bash
ansible all -m ping
```

## 7. Tamam Hosts ke Shared Variables

```ini
[all:vars]
```

Is section ke variables inventory ke har host par apply hote hain.

`ansible_` se shuru hone wale variables Ansible ke special connection ya behavioral variables hain. Yeh batate hain ke Ansible ko host tak kaise pohanchna ya us par kaam karna hai. Yeh Linux user create, Python install ya SSH key generate nahi karte.

| Variable | Maqsad | Location |
| --- | --- | --- |
| `ansible_host` | Managed node ka real IP ya DNS name | Managed node ka address |
| `ansible_user` | SSH login ke liye user | Managed node ka account |
| `ansible_python_interpreter` | Modules run karne ke liye Python executable | Managed node |
| `ansible_ssh_private_key_file` | SSH authentication ke liye private key | Control node |

Control-node aur managed-node paths ka farq important hai. Python ka path remote managed node par exist karna chahiye, jab ke private-key ka path Ansible control node par exist karna chahiye.

### Actual host address — `ansible_host`

```ini
web-server ansible_host=192.168.1.154
```

`web-server` friendly inventory alias hai. `ansible_host` woh actual network destination hai jis par SSH connection banta hai.

Is liye:

```bash
ansible web-server -m ping
```

asal mein is address ke saath connect karta hai:

```text
192.168.1.154
```

`ansible_host` mein IP address ya resolvable DNS name ho sakta hai. Yeh managed node ka Linux hostname change nahi karta.

### SSH connection user — `ansible_user`

```ini
ansible_user=ansibleadmin
```

Ansible har managed node ke saath `ansibleadmin` user ke taur par connect karega.

Yeh conceptually is command jaisa hai:

```bash
ssh ansibleadmin@192.168.1.154
```

`ansibleadmin` account har managed node par exist karna chahiye.

Meri `ansible.cfg` mein pehle hi yeh setting hai:

```ini
remote_user = ansibleadmin
```

Is liye inventory mein `ansible_user=ansibleadmin` technically duplicate hai. Main filhal isay rakh raha hoon taake `[all:vars]` ka effect clear rahe. Zaroorat par inventory variable general configuration ko override bhi kar sakta hai.

### Python interpreter — `ansible_python_interpreter`

```ini
ansible_python_interpreter=/usr/bin/python3
```

Yeh Ansible ko har managed node par Python-based modules ke liye `/usr/bin/python3` use karne ka kehta hai.

Yeh path **managed node** par check hota hai, control node par nahi. Agar different hosts par Python ke paths different hon, to variable ko host lines ya group-specific variables mein separately define kiya ja sakta hai.

Correct spelling:

```text
interpreter
```

Incorrect spelling:

```text
interpretor
```

Har managed node par Python path verify karein:

```bash
which python3
python3 --version
```

### SSH private key — `ansible_ssh_private_key_file`

Optional setting:

```ini
ansible_ssh_private_key_file=/home/ansibleadmin/.ssh/id_ed25519
```

Yeh **Ansible control node** ka path hai. Control node apni private key se SSH authentication ke dauran identity prove karta hai.

Private key ko:

- Managed nodes par copy nahi karna chahiye.
- Git repository mein commit nahi karna chahiye.
- Kisi ke saath share nahi karna chahiye.

`.pem` key aksar AWS EC2 jaise cloud instances ke liye use hoti hai. Meri local Rocky Linux lab mein aam tor par OpenSSH key, jaise `id_ed25519`, use hogi.

Jab key standard SSH location mein ho, OpenSSH usay automatically select kar sakta hai. Us surat mein inventory variable ki zaroorat nahi.

Is setting ko uncomment karna useful hai jab:

- Main non-standard key path use karoon.
- Mere paas multiple SSH keys hon.
- SSH automatically correct key select na kare.

Configuration files mein `~` par depend karne ke bajaye full absolute path zyada clear hota hai.

### Parsed variables inspect karna

Inventory save karne ke baad ek host ke parsed variables dekhein:

```bash
ansible-inventory --host web-server
```

Expected information:

```json
{
    "ansible_host": "192.168.1.154",
    "ansible_python_interpreter": "/usr/bin/python3",
    "ansible_user": "ansibleadmin"
}
```

Agar private-key line commented rahe, to `ansible_ssh_private_key_file` parsed variables mein show nahi hoga.

## 8. Basic Network Connectivity Validate Karna

Control node se IP addresses ki reachability check karein:

```bash
ping -c 2 192.168.1.154
ping -c 2 192.168.1.11
ping -c 2 192.168.1.215
```

Network ping fail hone ka hamesha yeh matlab nahi ke server unavailable hai, kyun ke firewall ICMP block kar sakta hai. Ansible ke liye SSH connectivity zyada relevant test hai.

## 9. Inventory Validate Karna

Pehle project directory mein jayein:

```bash
cd ~/automation
```

Confirm karein ke Ansible ne project configuration select ki:

```bash
ansible --version
```

Expected configuration path:

```text
/home/ansibleadmin/automation/ansible.cfg
```

Inventory hierarchy display karein:

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

Complete parsed inventory data display karein:

```bash
ansible-inventory --list
```

Module run kiye baghair inventory hosts list karein:

```bash
ansible all --list-hosts
```

Expected list:

```text
hosts (3):
  app-server
  db-server
  web-server
```

## 10. Connectivity aur Group Tests

In commands se pehle managed nodes aur SSH authentication prepare honi chahiye.

### Ek inventory host test karein

```bash
ansible web-server -m ping
```

### Ek group test karein

```bash
ansible web -m ping
```

### Parent group test karein

```bash
ansible three_tier_app -m ping
```

### Tamam inventory hosts test karein

```bash
ansible all -m ping
```

Ansible ka `ping` module ICMP network ping nahi hai. Yeh check karta hai ke Ansible connect kar sakta hai, usable Python interpreter dhoond sakta hai, module execute kar sakta hai aur successful response wapas le sakta hai.

## 11. Privilege Escalation Test Karna

Privilege escalation ke baghair `whoami` run karein:

```bash
ansible all -m command -a "whoami"
```

Expected remote user:

```text
ansibleadmin
```

`-b` ke saath become use karein:

```bash
ansible all -b -m command -a "whoami"
```

Expected privileged user:

```text
root
```

Is test ke liye har managed node par passwordless sudo ya become-password workflow required hai.

## 12. Ping Test se Pehle Requirements

Har managed node par yeh cheezen honi chahiye:

- SSH service running ho
- `ansibleadmin` account exist karta ho
- Python 3 `/usr/bin/python3` par available ho
- Appropriate sudo permissions hon
- Control node ki SSH public key remote user ki `authorized_keys` mein ho

Ansible package sirf control node par installed rahega.

## Agla Step

Teenon managed nodes par `ansibleadmin` prepare karna aur control node se SSH key-based authentication configure karna.
