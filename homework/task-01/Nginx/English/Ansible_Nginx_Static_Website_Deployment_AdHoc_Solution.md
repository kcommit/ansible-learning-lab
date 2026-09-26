# Solution: Deploy a Static Website with Nginx Using Ansible Ad-Hoc Commands

## Table of Contents

1. [Deployment Flow](#1-deployment-flow)
2. [Environment](#2-environment)
3. [Prechecks](#3-prechecks)
4. [Pilot Deployment on node1](#4-pilot-deployment-on-node1)
5. [Deploy to All Managed Nodes](#5-deploy-to-all-managed-nodes)
6. [Verification](#6-verification)
7. [Idempotency Test](#7-idempotency-test)
8. [Troubleshooting](#8-troubleshooting)
9. [Cleanup](#9-cleanup)
10. [Module Review](#10-module-review)

---

## 1. Deployment Flow

```text
Prechecks
   |
Install Nginx and unzip
   |
Start and enable Nginx
   |
Open HTTP in firewalld
   |
Test default Nginx page
   |
Download and inspect ZIP archive
   |
Extract website into Nginx document root
   |
Apply ownership, permissions, and SELinux context
   |
Verify HTTP 200
   |
Repeat on all nodes
   |
Test idempotency and clean up
```

---

## 2. Environment

Run all commands from the Ansible project directory:

```bash
cd /home/ansibleadmin/automation
```

Target group:

```text
three_tier_app
```

Website archive:

```text
https://freewebsitetemplates.com/download/space-science/
```

Nginx document root on Rocky Linux:

```text
/usr/share/nginx/html
```

Expected website directory:

```text
/usr/share/nginx/html/space-science
```

---

## 3. Prechecks

### Step 1: Confirm the active Ansible configuration

```bash
ansible --version
```

Look for:

```text
config file = /home/ansibleadmin/automation/ansible.cfg
```

Display changed configuration values:

```bash
ansible-config dump --only-changed
```

### Step 2: Inspect the inventory

```bash
ansible-inventory --graph
```

List the target hosts:

```bash
ansible three_tier_app --list-hosts
```

Expected hosts:

```text
node1
node2
node3
```

### Step 3: Test Ansible connectivity

```bash
ansible three_tier_app -m ping
```

Each node should return:

```text
"ping": "pong"
```

---

## 4. Pilot Deployment on node1

Always test the full process on one node first.

### Step 1: Install Nginx

```bash
ansible three_tier_app --limit node1 -b -m dnf -a \
"name=nginx state=present"
```

### Step 2: Install unzip

```bash
ansible three_tier_app --limit node1 -b -m dnf -a \
"name=unzip state=present"
```

The managed node needs `unzip` because the downloaded template is a ZIP archive.

### Step 3: Start and enable Nginx

```bash
ansible three_tier_app --limit node1 -b -m service -a \
"name=nginx state=started enabled=yes"
```

- `state=started` starts Nginx now.
- `enabled=yes` starts Nginx automatically after reboot.

### Step 4: Confirm the service state

```bash
ansible three_tier_app --limit node1 -m command -a \
"systemctl is-active nginx"
```

Expected output:

```text
active
```

### Step 5: Allow HTTP through firewalld

```bash
ansible three_tier_app --limit node1 -b -m firewalld -a \
"service=http permanent=yes immediate=yes state=enabled"
```

This keeps the firewall running while allowing web traffic on TCP port 80.

### Step 6: Test the default Nginx page

```bash
ansible three_tier_app --limit node1 -m uri -a \
"url=http://localhost status_code=200"
```

Expected result:

```text
status: 200
```

Browser test:

```text
http://192.168.1.154
```

### Step 7: Download the website archive

```bash
ansible three_tier_app --limit node1 -b -m get_url -a \
"url=https://freewebsitetemplates.com/download/space-science/ dest=/tmp/space-science.zip mode=0644"
```

### Step 8: Confirm the archive exists

```bash
ansible three_tier_app --limit node1 -m stat -a \
"path=/tmp/space-science.zip"
```

Important result:

```text
exists: true
```

### Step 9: Confirm that it is a ZIP archive

```bash
ansible three_tier_app --limit node1 -m command -a \
"file /tmp/space-science.zip"
```

Expected output should include:

```text
Zip archive data
```

If the result says `HTML document`, the download URL returned a webpage instead of the archive.

### Step 10: Inspect the archive structure

```bash
ansible three_tier_app --limit node1 -m command -a \
"unzip -l /tmp/space-science.zip"
```

Locate `index.html`. The archive is expected to provide a top-level directory named `space-science`.

### Step 11: Extract the website

```bash
ansible three_tier_app --limit node1 -b -m unarchive -a \
"src=/tmp/space-science.zip dest=/usr/share/nginx/html remote_src=yes creates=/usr/share/nginx/html/space-science/index.html"
```

Explanation:

- `remote_src=yes` means the archive is already on the managed node.
- `creates=` prevents unnecessary repeated extraction after `index.html` exists.

> If archive inspection shows a different top-level directory, adjust the `creates` path and browser URL accordingly.

### Step 12: Confirm the deployed index file

```bash
ansible three_tier_app --limit node1 -b -m shell -a \
"find /usr/share/nginx/html -path '*/space-science/index.html' -print"
```

Expected path:

```text
/usr/share/nginx/html/space-science/index.html
```

### Step 13: Apply ownership and safe permissions

```bash
ansible three_tier_app --limit node1 -b -m file -a \
"path=/usr/share/nginx/html/space-science state=directory owner=root group=root mode=u=rwX,g=rX,o=rX recurse=yes"
```

The capital `X` gives execute permission to directories while keeping ordinary files non-executable unless they were already executable.

### Step 14: Apply the web-content SELinux type

```bash
ansible three_tier_app --limit node1 -b -m file -a \
"path=/usr/share/nginx/html/space-science state=directory setype=httpd_sys_content_t recurse=yes"
```

SELinux remains enabled; the website receives the context expected for read-only web content.

### Step 15: Test the custom website locally

```bash
ansible three_tier_app --limit node1 -m uri -a \
"url=http://localhost/space-science/ status_code=200"
```

### Step 16: Test from a browser

```text
http://192.168.1.154/space-science/
```

Static content does not require an Nginx reload because the server reads the files when requests arrive.

---

## 5. Deploy to All Managed Nodes

After `node1` succeeds, remove `--limit node1`.

### Install Nginx and unzip

```bash
ansible three_tier_app -b -m dnf -a \
"name=nginx state=present"

ansible three_tier_app -b -m dnf -a \
"name=unzip state=present"
```

### Start and enable Nginx

```bash
ansible three_tier_app -b -m service -a \
"name=nginx state=started enabled=yes"
```

### Allow HTTP

```bash
ansible three_tier_app -b -m firewalld -a \
"service=http permanent=yes immediate=yes state=enabled"
```

### Download the template

```bash
ansible three_tier_app -b -m get_url -a \
"url=https://freewebsitetemplates.com/download/space-science/ dest=/tmp/space-science.zip mode=0644"
```

### Extract the template

```bash
ansible three_tier_app -b -m unarchive -a \
"src=/tmp/space-science.zip dest=/usr/share/nginx/html remote_src=yes creates=/usr/share/nginx/html/space-science/index.html"
```

### Apply ownership and permissions

```bash
ansible three_tier_app -b -m file -a \
"path=/usr/share/nginx/html/space-science state=directory owner=root group=root mode=u=rwX,g=rX,o=rX recurse=yes"
```

### Apply SELinux context

```bash
ansible three_tier_app -b -m file -a \
"path=/usr/share/nginx/html/space-science state=directory setype=httpd_sys_content_t recurse=yes"
```

---

## 6. Verification

### Check Nginx on every node

```bash
ansible three_tier_app -m command -a \
"systemctl is-active nginx"
```

### Confirm TCP port 80 is listening

The pipe requires the `shell` module:

```bash
ansible three_tier_app -m shell -a \
"ss -tln | grep ':80 '"
```

### Confirm the index file

```bash
ansible three_tier_app -m stat -a \
"path=/usr/share/nginx/html/space-science/index.html"
```

### Check the SELinux context

```bash
ansible three_tier_app -m command -a \
"ls -Zd /usr/share/nginx/html/space-science"
```

Expected context includes:

```text
httpd_sys_content_t
```

### Test HTTP from every managed node

```bash
ansible three_tier_app -m uri -a \
"url=http://localhost/space-science/ status_code=200"
```

### Browser URLs

```text
http://192.168.1.154/space-science/
http://192.168.1.185/space-science/
http://192.168.1.190/space-science/
```

---

## 7. Idempotency Test

Run the deployment commands again.

The desired second-run behavior is:

- `dnf`: `changed=false` because packages are installed.
- `service`: `changed=false` because Nginx is running and enabled.
- `firewalld`: `changed=false` because HTTP is already enabled.
- `get_url`: normally `changed=false` when the remote file has not changed.
- `unarchive`: skipped or unchanged because the `creates` path exists.
- `file`: `changed=false` when ownership, permissions, and context are correct.
- `uri`: verification succeeds without changing the server.

Idempotency means repeated automation preserves the required state without making unnecessary changes.

---

## 8. Troubleshooting

### Problem: `get_url` downloads HTML

Check:

```bash
ansible three_tier_app --limit node1 -m command -a \
"file /tmp/space-science.zip"
```

The result should say `Zip archive data`, not `HTML document`.

### Problem: `index.html` is not at the expected path

Inspect:

```bash
ansible three_tier_app --limit node1 -m command -a \
"unzip -l /tmp/space-science.zip"
```

Then locate the extracted file:

```bash
ansible three_tier_app --limit node1 -b -m shell -a \
"find /usr/share/nginx/html -name index.html -print"
```

### Problem: Browser cannot connect

Check service and port:

```bash
ansible three_tier_app --limit node1 -m command -a \
"systemctl is-active nginx"

ansible three_tier_app --limit node1 -m command -a \
"ss -tln"
```

Check the firewall:

```bash
ansible three_tier_app --limit node1 -b -m command -a \
"firewall-cmd --list-services"
```

### Problem: HTTP 403 Forbidden

Check permissions and SELinux context:

```bash
ansible three_tier_app --limit node1 -m command -a \
"namei -l /usr/share/nginx/html/space-science/index.html"

ansible three_tier_app --limit node1 -m command -a \
"ls -lZ /usr/share/nginx/html/space-science/index.html"
```

### Check Nginx logs

```bash
ansible three_tier_app --limit node1 -b -m command -a \
"journalctl -u nginx --no-pager -n 30"
```

---

## 9. Cleanup

### Remove the custom website

```bash
ansible three_tier_app -b -m file -a \
"path=/usr/share/nginx/html/space-science state=absent"
```

### Remove the downloaded archive

```bash
ansible three_tier_app -b -m file -a \
"path=/tmp/space-science.zip state=absent"
```

### Optionally uninstall Nginx and unzip

```bash
ansible three_tier_app -b -m dnf -a \
"name=nginx state=absent"

ansible three_tier_app -b -m dnf -a \
"name=unzip state=absent"
```

### Optionally remove the firewall rule

Only do this when no other website needs HTTP access:

```bash
ansible three_tier_app -b -m firewalld -a \
"service=http permanent=yes immediate=yes state=disabled"
```

---

## 10. Module Review

| Module | Purpose in this lab |
|---|---|
| `ping` | Tests Ansible connectivity |
| `dnf` | Installs Nginx and unzip |
| `service` | Starts and enables Nginx |
| `firewalld` | Allows HTTP traffic |
| `get_url` | Downloads the website archive |
| `stat` | Checks files and metadata |
| `command` | Runs commands without shell features |
| `shell` | Supports pipes and wildcard-style searches |
| `unarchive` | Extracts the ZIP archive |
| `file` | Manages ownership, permissions, SELinux context, and cleanup |
| `uri` | Tests the HTTP response |

