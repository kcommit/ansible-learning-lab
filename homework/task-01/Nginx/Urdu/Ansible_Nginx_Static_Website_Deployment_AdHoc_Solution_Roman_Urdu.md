# Roman Urdu Solution: Nginx par Static Website Ansible Ad-Hoc Commands se Deploy Karein

## Fehrist

1. [Deployment Flow](#1-deployment-flow)
2. [Prechecks](#2-prechecks)
3. [node1 par Pilot Deployment](#3-node1-par-pilot-deployment)
4. [Teenon Nodes par Deployment](#4-teenon-nodes-par-deployment)
5. [Verification](#5-verification)
6. [Idempotency](#6-idempotency)
7. [Troubleshooting](#7-troubleshooting)
8. [Cleanup](#8-cleanup)

---

## 1. Deployment Flow

```text
Prechecks
  -> Nginx aur unzip install
  -> Nginx start aur enable
  -> Firewall mein HTTP allow
  -> Default page test
  -> ZIP download aur inspect
  -> Website extract
  -> Permissions aur SELinux context
  -> HTTP 200 verification
  -> All nodes deployment
  -> Idempotency aur cleanup
```

Project directory mein jayein:

```bash
cd /home/ansibleadmin/automation
```

---

## 2. Prechecks

### Active configuration dekhein

```bash
ansible --version
ansible-config dump --only-changed
```

### Inventory graph aur target hosts dekhein

```bash
ansible-inventory --graph
ansible three_tier_app --list-hosts
```

Expected hosts:

```text
node1
node2
node3
```

### Connectivity test karein

```bash
ansible three_tier_app -m ping
```

Har node se `pong` milna chahiye.

---

## 3. `node1` par Pilot Deployment

### Step 1: Nginx install karein

```bash
ansible three_tier_app --limit node1 -b -m dnf -a \
"name=nginx state=present"
```

### Step 2: unzip install karein

```bash
ansible three_tier_app --limit node1 -b -m dnf -a \
"name=unzip state=present"
```

ZIP archive extract karne ke liye managed node par `unzip` chahiye.

### Step 3: Nginx start aur enable karein

```bash
ansible three_tier_app --limit node1 -b -m service -a \
"name=nginx state=started enabled=yes"
```

- `state=started`: service abhi running ho.
- `enabled=yes`: reboot ke baad automatically start ho.

### Step 4: Service verify karein

```bash
ansible three_tier_app --limit node1 -m command -a \
"systemctl is-active nginx"
```

Expected:

```text
active
```

### Step 5: Firewall mein HTTP allow karein

```bash
ansible three_tier_app --limit node1 -b -m firewalld -a \
"service=http permanent=yes immediate=yes state=enabled"
```

### Step 6: Default Nginx page test karein

```bash
ansible three_tier_app --limit node1 -m uri -a \
"url=http://localhost status_code=200"
```

Browser:

```text
http://192.168.1.154
```

### Step 7: Website ZIP download karein

```bash
ansible three_tier_app --limit node1 -b -m get_url -a \
"url=https://freewebsitetemplates.com/download/space-science/ dest=/tmp/space-science.zip mode=0644"
```

### Step 8: Download verify karein

```bash
ansible three_tier_app --limit node1 -m stat -a \
"path=/tmp/space-science.zip"

ansible three_tier_app --limit node1 -m command -a \
"file /tmp/space-science.zip"
```

Result mein `Zip archive data` aana chahiye. Agar `HTML document` aaye to URL actual archive return nahi kar raha.

### Step 9: ZIP structure inspect karein

```bash
ansible three_tier_app --limit node1 -m command -a \
"unzip -l /tmp/space-science.zip"
```

`index.html` aur top-level directory ka naam note karein.

### Step 10: Website extract karein

```bash
ansible three_tier_app --limit node1 -b -m unarchive -a \
"src=/tmp/space-science.zip dest=/usr/share/nginx/html remote_src=yes creates=/usr/share/nginx/html/space-science/index.html"
```

- `remote_src=yes`: ZIP managed node par maujood hai.
- `creates=`: `index.html` maujood ho to unnecessary extraction nahi hogi.

> Agar ZIP ka directory name mukhtalif ho to `creates` path aur website URL adjust karein.

### Step 11: `index.html` locate karein

```bash
ansible three_tier_app --limit node1 -b -m shell -a \
"find /usr/share/nginx/html -name index.html -print"
```

### Step 12: Ownership aur permissions set karein

```bash
ansible three_tier_app --limit node1 -b -m file -a \
"path=/usr/share/nginx/html/space-science state=directory owner=root group=root mode=u=rwX,g=rX,o=rX recurse=yes"
```

Capital `X` directories ko traverse permission deta hai aur normal files ko bila-wajah executable nahi banata.

### Step 13: SELinux web context apply karein

```bash
ansible three_tier_app --limit node1 -b -m file -a \
"path=/usr/share/nginx/html/space-science state=directory setype=httpd_sys_content_t recurse=yes"
```

### Step 14: Custom website test karein

```bash
ansible three_tier_app --limit node1 -m uri -a \
"url=http://localhost/space-science/ status_code=200"
```

Browser:

```text
http://192.168.1.154/space-science/
```

Static files deploy karne par Nginx reload zaroori nahi, kyun ke configuration change nahi hui.

---

## 4. Teenon Nodes par Deployment

`node1` successful hone ke baad `--limit node1` hata dein:

```bash
ansible three_tier_app -b -m dnf -a "name=nginx state=present"
ansible three_tier_app -b -m dnf -a "name=unzip state=present"
ansible three_tier_app -b -m service -a "name=nginx state=started enabled=yes"
ansible three_tier_app -b -m firewalld -a "service=http permanent=yes immediate=yes state=enabled"
ansible three_tier_app -b -m get_url -a "url=https://freewebsitetemplates.com/download/space-science/ dest=/tmp/space-science.zip mode=0644"
ansible three_tier_app -b -m unarchive -a "src=/tmp/space-science.zip dest=/usr/share/nginx/html remote_src=yes creates=/usr/share/nginx/html/space-science/index.html"
ansible three_tier_app -b -m file -a "path=/usr/share/nginx/html/space-science state=directory owner=root group=root mode=u=rwX,g=rX,o=rX recurse=yes"
ansible three_tier_app -b -m file -a "path=/usr/share/nginx/html/space-science state=directory setype=httpd_sys_content_t recurse=yes"
```

---

## 5. Verification

```bash
ansible three_tier_app -m command -a "systemctl is-active nginx"
ansible three_tier_app -m shell -a "ss -tln | grep ':80 '"
ansible three_tier_app -m stat -a "path=/usr/share/nginx/html/space-science/index.html"
ansible three_tier_app -m command -a "ls -Zd /usr/share/nginx/html/space-science"
ansible three_tier_app -m uri -a "url=http://localhost/space-science/ status_code=200"
```

Browser URLs:

```text
http://192.168.1.154/space-science/
http://192.168.1.185/space-science/
http://192.168.1.190/space-science/
```

---

## 6. Idempotency

Deployment commands dobara chalayein. Expected behavior:

- Packages already installed hon to `changed=false`.
- Nginx running aur enabled ho to `changed=false`.
- HTTP pehle se allowed ho to `changed=false`.
- Remote archive change na ho to `get_url` aam tor par `changed=false`.
- `creates` path ki wajah se extraction repeat nahi hogi.
- Correct permissions aur context hon to `file` task `changed=false`.
- `uri` sirf verification karega.

Idempotency ka matlab hai ke repeated automation required state ko maintain kare aur unnecessary changes na kare.

---

## 7. Troubleshooting

### ZIP ki jagah HTML download ho

```bash
ansible three_tier_app --limit node1 -m command -a "file /tmp/space-science.zip"
```

### `index.html` expected path par na ho

```bash
ansible three_tier_app --limit node1 -m command -a "unzip -l /tmp/space-science.zip"
ansible three_tier_app --limit node1 -b -m shell -a "find /usr/share/nginx/html -name index.html -print"
```

### Browser connect na kare

```bash
ansible three_tier_app --limit node1 -m command -a "systemctl is-active nginx"
ansible three_tier_app --limit node1 -m command -a "ss -tln"
ansible three_tier_app --limit node1 -b -m command -a "firewall-cmd --list-services"
```

### HTTP 403 aaye

```bash
ansible three_tier_app --limit node1 -m command -a "namei -l /usr/share/nginx/html/space-science/index.html"
ansible three_tier_app --limit node1 -m command -a "ls -lZ /usr/share/nginx/html/space-science/index.html"
```

### Nginx logs dekhein

```bash
ansible three_tier_app --limit node1 -b -m command -a "journalctl -u nginx --no-pager -n 30"
```

---

## 8. Cleanup

Website aur ZIP remove karein:

```bash
ansible three_tier_app -b -m file -a "path=/usr/share/nginx/html/space-science state=absent"
ansible three_tier_app -b -m file -a "path=/tmp/space-science.zip state=absent"
```

Optional Nginx aur unzip removal:

```bash
ansible three_tier_app -b -m dnf -a "name=nginx state=absent"
ansible three_tier_app -b -m dnf -a "name=unzip state=absent"
```

HTTP rule sirf tab remove karein jab koi doosri website port 80 use na kar rahi ho:

```bash
ansible three_tier_app -b -m firewalld -a \
"service=http permanent=yes immediate=yes state=disabled"
```

