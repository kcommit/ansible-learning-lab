# Ansible Study Notes — Roman Urdu

## Part 1: Rocky Linux 9 par Control Node Setup

Yeh notes hamari ab tak ki complete practical activity ko record karti hain: Rocky Linux server check karne se lekar Ansible install aur locally test karne tak.

## Lab Environment

| Item | Value |
| --- | --- |
| Server ka role | Ansible control node |
| Hostname | `ansible-server` |
| Operating system | Rocky Linux 9.8 (Blue Onyx) |
| Architecture | `x86_64` |
| Virtualization | Xen |
| Administrative user | `ansibleadmin` |
| Python | Python 3.9.25 |
| Python ka path | `/usr/bin/python3` |
| Ansible community package | 7.7.0 |
| Ansible Core | 2.14.18 |

## 1. Agent aur Agentless kya hain?

**Agent-based** automation tool mein har managed server par ek khas agent software install aur continuously available hona zaroori hota hai.

Ansible aam tor par **agentless** hai. Linux managed nodes par Ansible ka koi agent ya daemon install karna zaroori nahi hota. Control node usually SSH ke zariye managed node se connect hota hai, required module transfer aur execute karta hai, result wapas leta hai aur disconnect ho jata hai.

```text
Ansible control node -- SSH --> Managed Linux node
```

Linux managed node par aam tor par yeh cheezen honi chahiye:

- SSH server
- Zyada tar Ansible modules ke liye Python 3
- Required permissions wala remote user
- Privileged task ke liye `sudo` access

Ansible package sirf control node par install hota hai.

## 2. Server ki Information Check Karna

Hostname, operating system, kernel, architecture aur virtualization check karein:

```bash
hostnamectl
```

Hamare lab server par Rocky Linux 9.8 mila aur is ko Ansible control node banaya gaya.

## 3. Dedicated Administrative User Banana

Daily Ansible ka kaam directly `root` user se nahi karna chahiye. Hum ne `ansibleadmin` naam ka dedicated user banaya.

Neeche diye gaye commands `root` user se run karein.

### User create karein

```bash
useradd -m -s /bin/bash ansibleadmin
```

Options ka matlab:

- `-m`: user ki home directory banata hai
- `-s /bin/bash`: Bash ko login shell assign karta hai

### Password set karein

```bash
passwd ansibleadmin
```

Kam az kam 8 characters ka strong password use karein. Agar `BAD PASSWORD` warning aaye lekin akhir mein `all authentication tokens updated successfully` likha ho, to password warning ke bawajood set ho gaya. Phir bhi strong password rakhna best practice hai.

### User ko wheel group mein add karein

```bash
usermod -aG wheel ansibleadmin
```

Options:

- `-a`: user ko group mein append karta hai aur purane supplementary groups remove nahi karta
- `-G`: supplementary group specify karta hai
- `wheel`: Rocky Linux ka standard administrative group hai

### User aur groups verify karein

```bash
id ansibleadmin
```

Hamare lab ka result:

```text
uid=1001(ansibleadmin) gid=1001(ansibleadmin) groups=1001(ansibleadmin),10(wheel)
```

`10(wheel)` confirm karta hai ke user wheel group ka member hai.

## 4. Normal Sudo Access Test Karna

Naye user par switch karein:

```bash
su - ansibleadmin
```

Sudo test karein:

```bash
sudo whoami
```

Password enter karne ke baad expected output:

```text
root
```

Is ka matlab yeh nahi ke login user permanently root ban gaya. Sirf `whoami` command root privileges ke saath execute hua.

## 5. Lab ke Liye Passwordless Sudo Configure Karna

Controlled Ansible lab mein passwordless sudo useful hai, kyun ke automated tasks har command par manually password enter karne ke liye ruk nahi sakte.

Yeh commands `root` shell se run karein:

```bash
echo "ansibleadmin ALL=(ALL) NOPASSWD: ALL" > /etc/sudoers.d/ansibleadmin
chmod 440 /etc/sudoers.d/ansibleadmin
visudo -cf /etc/sudoers.d/ansibleadmin
```

Sudo rule ka matlab:

| Field | Matlab |
| --- | --- |
| `ansibleadmin` | Rule is user par apply hoga |
| `ALL=(ALL)` | Har host par kisi bhi user ke taur par command chala sakta hai |
| `NOPASSWD:` | Sudo password nahi poochega |
| `ALL` | Tamam commands run kar sakta hai |

`chmod 440` file ko correct secure permission deta hai. Owner aur group file read kar sakte hain, lekin normal access se modify nahi kar sakte.

Expected validation:

```text
/etc/sudoers.d/ansibleadmin: parsed OK
```

### Redirection ka important lesson

Ordinary user yeh command directly run nahi kar sakta:

```bash
echo "ansibleadmin ALL=(ALL) NOPASSWD: ALL" > /etc/sudoers.d/ansibleadmin
```

Shell `>` redirection perform karta hai, aur ordinary user ko `/etc/sudoers.d` ke andar write permission nahi hoti. Hamare lab mein complete command ko root shell se run karna correct solution tha.

### Passwordless sudo test karein

```bash
sudo -n whoami
```

Expected output bina password prompt ke:

```text
root
```

`-n` ka matlab **non-interactive** hai. Yeh sudo ko kehta hai ke password mat poochna. Agar passwordless sudo configured na ho to command foran `sudo: a password is required` error ke saath fail ho jata hai.

## 6. Python Check Karna

Ansible Linux systems par zyada tar modules execute karne ke liye Python 3 use karta hai.

```bash
python3 --version
which python3
```

Hamare results:

```text
Python 3.9.25
/usr/bin/python3
```

Modern Rocky Linux par explicit `python3` command use karein. `python` command missing ho sakta hai ya kisi different version ki taraf point kar sakta hai.

Agar Python 3 missing ho:

```bash
sudo dnf install python3 -y
```

## 7. Rocky Linux Update Karna

Lab control node ke tamam installed packages update karein:

```bash
sudo dnf update -y
```

Update ke dauran naya kernel install hua. Naye kernel ko activate karne ke liye reboot karein:

```bash
sudo reboot
```

Server se dobara connect hone ke baad active kernel verify karein:

```bash
uname -r
```

Production environment mein updates ko pehle review karein aur approved maintenance window mein patching karein:

```bash
sudo dnf check-update
```

## 8. EPEL Enable aur Verify Karna

Rocky Linux mein full Ansible community package EPEL yani **Extra Packages for Enterprise Linux** repository se milta hai.

```bash
sudo dnf install epel-release -y
```

Agar EPEL pehle se installed ho to DNF yeh result de sakta hai:

```text
Package epel-release is already installed.
Nothing to do.
Complete!
```

Enabled EPEL repositories verify karein:

```bash
sudo dnf repolist | grep -i epel
```

Hamare lab mein:

```text
epel                Extra Packages for Enterprise Linux 9 - x86_64
epel-cisco-openh264 Extra Packages for Enterprise Linux 9 openh264 (From Cisco) - x86_64
```

## 9. Ansible Package Check aur Install Karna

Install karne se pehle available package ki information dekhein:

```bash
sudo dnf info ansible
```

Lab mein EPEL se Ansible 7.7.0 available mila.

Full Ansible package install karein:

```bash
sudo dnf install ansible -y
```

Is command ne `ansible-core` ke saath required dependencies bhi install ki, jaise Python libraries, `git-core` aur `sshpass`.

### Ansible package aur Ansible Core mein farq

- `ansible-core` main automation engine, command-line programs aur built-in functionality hai.
- `ansible` bara community package hai jis mein Ansible Core aur curated collections shamil hoti hain.

Isi liye yeh different version numbers bilkul normal hain:

```text
ansible package: 7.7.0
ansible-core:     2.14.18
```

## 10. Installation Verify Karna

```bash
ansible --version
ansible-playbook --version
```

Important values:

```text
ansible [core 2.14.18]
config file = /etc/ansible/ansible.cfg
ansible python module location = /usr/lib/python3.9/site-packages/ansible
executable location = /usr/bin/ansible
python version = 3.9.25
```

`ansible` ad-hoc tasks run karta hai, jab ke `ansible-playbook` reusable YAML playbooks run karta hai.

## 11. Local Ansible Test Karna

Control node ko khud target bana kar Ansible ping module test karein:

```bash
ansible localhost -m ping
```

Result:

```text
localhost | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

Yeh normal ICMP network ping nahi hai. Yeh test confirm karta hai ke Ansible apna Python-based `ping` module load aur execute karke result wapas de sakta hai.

Ab `whoami` command ko Ansible ke zariye run karein:

```bash
ansible localhost -m command -a "whoami"
```

Result:

```text
localhost | CHANGED | rc=0 >>
ansibleadmin
```

Command breakdown:

| Part | Matlab |
| --- | --- |
| `ansible` | Ad-hoc Ansible task chalata hai |
| `localhost` | Target host hai |
| `-m command` | Command module select karta hai |
| `-a "whoami"` | Module ko `whoami` argument deta hai |
| `rc=0` | Command successfully complete hua |
| `ansibleadmin` | Command is user ke taur par run hua |

Command module aksar command execute karne par `CHANGED` report karta hai, chahe `whoami` jaisi read-only command ne system ko actually change na kiya ho. Baad mein playbook ke andar `changed_when` se is behavior ko control karna seekhenge.

## 12. Zaroorat Par Ansible Uninstall Karna

Full Ansible package remove karein:

```bash
sudo dnf remove ansible -y
```

Remaining Ansible packages check karein:

```bash
rpm -qa | grep -i ansible
```

Agar zaroorat ho to Ansible Core separately remove karein:

```bash
sudo dnf remove ansible-core -y
```

Sirf Ansible remove karne ki wajah se EPEL remove na karein, kyun ke doosre packages bhi is repository ko use kar sakte hain. `dnf autoremove` chalane se pehle proposed package list ko carefully review karein.

## Final Status

Control node ki installation aur validation complete hai:

- Dedicated `ansibleadmin` account create ho gaya
- Wheel membership verify ho gayi
- Passwordless sudo configure aur validate ho gaya
- Rocky Linux update aur reboot ho gaya
- Python 3 verify ho gaya
- EPEL enabled hai
- Full Ansible package install ho gaya
- `ansible` aur `ansible-playbook` verify ho gaye
- Local ping aur command-module tests pass ho gaye

## Agla Stage

Lab ke next part mein hum seekhenge:

1. Managed nodes prepare karna
2. Remote administrative user banana
3. SSH keys generate aur distribute karna
4. Ansible inventory banana
5. `ansible all -m ping` run karna
