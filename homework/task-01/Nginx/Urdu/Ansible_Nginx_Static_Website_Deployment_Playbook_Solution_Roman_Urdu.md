# Roman Urdu Playbook Solution: Nginx par Static Website Deploy Karein

## Fehrist

1. [Playbook kyun?](#1-playbook-kyun)
2. [Project Structure](#2-project-structure)
3. [Deployment Playbook](#3-deployment-playbook)
4. [Playbook ki Wazahat](#4-playbook-ki-wazahat)
5. [Validate aur Run](#5-validate-aur-run)
6. [Pilot aur Full Deployment](#6-pilot-aur-full-deployment)
7. [Verification aur Idempotency](#7-verification-aur-idempotency)
8. [Cleanup Playbook](#8-cleanup-playbook)
9. [Troubleshooting](#9-troubleshooting)

---

## 1. Playbook kyun?

Ad-hoc commands demonstration aur one-time work ke liye useful hain. Repeatable deployment ke liye playbook behtar hai kyun ke:

- Desired state YAML file mein save hoti hai.
- Tasks defined order mein chalte hain.
- Variables aur conditions use hoti hain.
- Execution se pehle syntax check ho sakta hai.
- Pilot node par `--limit` lagaya ja sakta hai.
- Git mein version control ho sakta hai.

Is solution mein aap ki preference ke mutabiq short module names use hue hain.

---

## 2. Project Structure

```text
/home/ansibleadmin/automation/
├── ansible.cfg
├── inventory/
│   └── nodes
└── playbooks/
    ├── deploy-space-science.yml
    └── cleanup-space-science.yml
```

Directory banayein:

```bash
mkdir -p /home/ansibleadmin/automation/playbooks
cd /home/ansibleadmin/automation
```

---

## 3. Deployment Playbook

File banayein:

```text
/home/ansibleadmin/automation/playbooks/deploy-space-science.yml
```

Content:

```yaml
---
- name: Install Nginx and deploy the Space Science website
  hosts: three_tier_app
  become: true

  vars:
    website_url: "https://freewebsitetemplates.com/download/space-science/"
    archive_path: "/tmp/space-science.zip"
    nginx_document_root: "/usr/share/nginx/html"
    website_directory: "/usr/share/nginx/html/space-science"
    website_index: "/usr/share/nginx/html/space-science/index.html"
    website_uri: "http://localhost/space-science/"

  tasks:
    - name: Install Nginx
      dnf:
        name: nginx
        state: present

    - name: Install unzip
      dnf:
        name: unzip
        state: present

    - name: Start and enable Nginx
      service:
        name: nginx
        state: started
        enabled: true

    - name: Allow HTTP through firewalld
      firewalld:
        service: http
        permanent: true
        immediate: true
        state: enabled

    - name: Verify the default Nginx website
      uri:
        url: "http://localhost"
        status_code: 200
      changed_when: false

    - name: Download the website archive
      get_url:
        url: "{{ website_url }}"
        dest: "{{ archive_path }}"
        mode: "0644"

    - name: Extract the website archive
      unarchive:
        src: "{{ archive_path }}"
        dest: "{{ nginx_document_root }}"
        remote_src: true
        creates: "{{ website_index }}"

    - name: Set website ownership and permissions
      file:
        path: "{{ website_directory }}"
        state: directory
        owner: root
        group: root
        mode: "u=rwX,g=rX,o=rX"
        recurse: true

    - name: Apply the web-content SELinux type
      file:
        path: "{{ website_directory }}"
        state: directory
        setype: httpd_sys_content_t
        recurse: true

    - name: Verify that the website index exists
      stat:
        path: "{{ website_index }}"
      register: website_index_status

    - name: Stop when the website index is missing
      fail:
        msg: >-
          Expected index file {{ website_index }} par nahi mili.
          ZIP structure inspect karke paths update karein.
      when: not website_index_status.stat.exists

    - name: Verify the custom website
      uri:
        url: "{{ website_uri }}"
        status_code: 200
        return_content: false
      register: website_test
      changed_when: false

    - name: Display deployment result
      debug:
        msg: >-
          {{ inventory_hostname }} ne {{ website_uri }} ke liye
          HTTP {{ website_test.status }} return kiya.
```

---

## 4. Playbook ki Wazahat

### Target group

```yaml
hosts: three_tier_app
```

Yeh parent group ke zariye `node1`, `node2` aur `node3` ko target karta hai.

### Privilege escalation

```yaml
become: true
```

Package, firewall aur document-root tasks `sudo` ke sath chalenge.

### Variables

URL aur paths ek jagah store hain, is liye future changes asaan hain.

### `dnf`, `service` aur `firewalld`

- `dnf` required packages ko present rakhta hai.
- `service` Nginx ko running aur boot par enabled rakhta hai.
- `firewalld` firewall band kiye baghair HTTP allow karta hai.

### `get_url` aur `unarchive`

- `get_url` har managed node par ZIP download karta hai.
- `remote_src: true` batata hai ke ZIP managed node par hai.
- `creates` repeated extraction ko rokta hai jab expected index file maujood ho.

### Permissions aur SELinux

`file` tasks ownership, readable permissions aur `httpd_sys_content_t` context ensure karte hain.

### `stat`, `fail` aur `uri`

- `stat` expected file check karta hai.
- `fail` wrong archive structure par clear error deta hai.
- `uri` HTTP `200` verify karta hai.

### Nginx reload kyun nahi?

Sirf static files change hui hain, Nginx configuration nahi. Is liye reload zaroori nahi.

---

## 5. Validate aur Run

Syntax check:

```bash
ansible-playbook --syntax-check playbooks/deploy-space-science.yml
```

Target hosts aur tasks dekhein:

```bash
ansible-playbook playbooks/deploy-space-science.yml --list-hosts
ansible-playbook playbooks/deploy-space-science.yml --list-tasks
```

Optional check mode:

```bash
ansible-playbook playbooks/deploy-space-science.yml --check --limit node1
```

Network aur archive operations check mode mein completely simulate na bhi hon; actual verification phir bhi zaroori hai.

---

## 6. Pilot aur Full Deployment

Pehle sirf `node1`:

```bash
ansible-playbook playbooks/deploy-space-science.yml --limit node1
```

Browser test:

```text
http://192.168.1.154/space-science/
```

Pilot successful ho to all nodes:

```bash
ansible-playbook playbooks/deploy-space-science.yml
```

---

## 7. Verification aur Idempotency

```bash
ansible three_tier_app -m command -a "systemctl is-active nginx"
ansible three_tier_app -m stat -a "path=/usr/share/nginx/html/space-science/index.html"
ansible three_tier_app -m uri -a "url=http://localhost/space-science/ status_code=200"
```

Playbook dobara chalayein:

```bash
ansible-playbook playbooks/deploy-space-science.yml
```

Ideal second-run recap:

```text
changed=0
failed=0
unreachable=0
```

Yeh dikhata hai ke required state pehle se maujood hai aur unnecessary changes nahi hue.

---

## 8. Cleanup Playbook

File:

```text
/home/ansibleadmin/automation/playbooks/cleanup-space-science.yml
```

Content:

```yaml
---
- name: Remove the Space Science website lab
  hosts: three_tier_app
  become: true

  vars:
    website_directory: "/usr/share/nginx/html/space-science"
    archive_path: "/tmp/space-science.zip"
    remove_nginx: false

  tasks:
    - name: Remove the deployed website
      file:
        path: "{{ website_directory }}"
        state: absent

    - name: Remove the downloaded archive
      file:
        path: "{{ archive_path }}"
        state: absent

    - name: Stop and disable Nginx when requested
      service:
        name: nginx
        state: stopped
        enabled: false
      when: remove_nginx | bool

    - name: Remove Nginx when requested
      dnf:
        name: nginx
        state: absent
      when: remove_nginx | bool
```

Sirf website aur ZIP remove karein:

```bash
ansible-playbook playbooks/cleanup-space-science.yml
```

Nginx bhi remove karein:

```bash
ansible-playbook playbooks/cleanup-space-science.yml \
-e "remove_nginx=true"
```

Pehle sirf `node1` par cleanup test karein:

```bash
ansible-playbook playbooks/cleanup-space-science.yml --limit node1
```

---

## 9. Troubleshooting

ZIP structure dekhein:

```bash
ansible three_tier_app --limit node1 -m command -a "unzip -l /tmp/space-science.zip"
```

Actual index path dhoondein:

```bash
ansible three_tier_app --limit node1 -b -m shell -a "find /usr/share/nginx/html -name index.html -print"
```

HTTP 403 par permissions aur context check karein:

```bash
ansible three_tier_app --limit node1 -m command -a "namei -l /usr/share/nginx/html/space-science/index.html"
ansible three_tier_app --limit node1 -m command -a "ls -lZ /usr/share/nginx/html/space-science/index.html"
```

Browser connect na kare to Nginx aur firewall check karein:

```bash
ansible three_tier_app --limit node1 -m command -a "systemctl is-active nginx"
ansible three_tier_app --limit node1 -b -m command -a "firewall-cmd --list-services"
```

