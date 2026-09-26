# Playbook Solution: Deploy a Static Website with Nginx

## Table of Contents

1. [Why Use a Playbook?](#1-why-use-a-playbook)
2. [Project Structure](#2-project-structure)
3. [Deployment Playbook](#3-deployment-playbook)
4. [Playbook Explanation](#4-playbook-explanation)
5. [Validate and Run](#5-validate-and-run)
6. [Pilot Deployment](#6-pilot-deployment)
7. [Full Deployment](#7-full-deployment)
8. [Verification](#8-verification)
9. [Idempotency Test](#9-idempotency-test)
10. [Cleanup Playbook](#10-cleanup-playbook)
11. [Troubleshooting](#11-troubleshooting)
12. [Recommended Improvements](#12-recommended-improvements)

---

## 1. Why Use a Playbook?

Ad-hoc commands are useful for demonstrations and one-time operations. A playbook is better for a repeatable deployment because it:

- Stores the desired state as YAML.
- Runs tasks in a defined order.
- Supports variables and conditions.
- Can be checked before execution.
- Can be limited to one pilot node.
- Provides consistent results across all managed nodes.
- Can be stored in Git and reviewed later.

This solution intentionally uses short module names to match the current learning style.

---

## 2. Project Structure

Recommended structure:

```text
/home/ansibleadmin/automation/
├── ansible.cfg
├── inventory/
│   └── nodes
└── playbooks/
    ├── deploy-space-science.yml
    └── cleanup-space-science.yml
```

Create the playbook directory:

```bash
mkdir -p /home/ansibleadmin/automation/playbooks
```

Move into the project directory:

```bash
cd /home/ansibleadmin/automation
```

---

## 3. Deployment Playbook

Create:

```text
/home/ansibleadmin/automation/playbooks/deploy-space-science.yml
```

Add the following content:

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
          The expected index file was not found at {{ website_index }}.
          Inspect the ZIP structure and adjust website_directory and website_index.
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
          {{ inventory_hostname }} successfully returned HTTP
          {{ website_test.status }} for {{ website_uri }}
```

---

## 4. Playbook Explanation

### Play name

```yaml
- name: Install Nginx and deploy the Space Science website
```

Provides a readable description of the automation.

### Target hosts

```yaml
hosts: three_tier_app
```

Targets `node1`, `node2`, and `node3` through the parent inventory group.

### Privilege escalation

```yaml
become: true
```

Runs privileged tasks through `sudo` because package installation, firewall management, and document-root changes require root access.

### Variables

The `vars` section stores values that might change later, such as the download URL and deployment paths.

### Package tasks

The two `dnf` tasks ensure that Nginx and unzip are installed. Repeating the playbook does not reinstall packages unnecessarily.

### Service task

```yaml
state: started
enabled: true
```

Ensures Nginx is running now and after future reboots.

### Firewall task

Allows HTTP traffic while keeping `firewalld` enabled.

### Default-page test

The first `uri` task proves that Nginx works before deploying the custom site.

### Download task

`get_url` downloads the ZIP archive directly to every managed node.

### Extraction task

`remote_src: true` tells Ansible that the archive already exists on the managed node. `creates` improves repeatability by preventing extraction after the expected `index.html` exists.

### Permission and SELinux tasks

The `file` tasks ensure that:

- Files belong to `root:root`.
- Directories are traversable.
- Website files are readable.
- SELinux recognizes the directory as web content.

### `stat`, `fail`, and `uri`

- `stat` checks whether the expected file exists.
- `fail` stops deployment with a clear message when the archive structure differs.
- `uri` verifies that Nginx returns HTTP `200`.

### Why there is no reload task

This playbook deploys static files but does not change Nginx configuration. Nginx reads static files when requests arrive, so a reload is unnecessary.

---

## 5. Validate and Run

### Step 1: Confirm YAML syntax

```bash
ansible-playbook --syntax-check \
playbooks/deploy-space-science.yml
```

### Step 2: List targeted hosts

```bash
ansible-playbook playbooks/deploy-space-science.yml \
--list-hosts
```

### Step 3: List tasks

```bash
ansible-playbook playbooks/deploy-space-science.yml \
--list-tasks
```

### Step 4: Optional check mode

```bash
ansible-playbook playbooks/deploy-space-science.yml \
--check --limit node1
```

Some network and archive operations may not fully simulate in check mode. Treat check mode as a preview, not final verification.

---

## 6. Pilot Deployment

Deploy only to `node1`:

```bash
ansible-playbook playbooks/deploy-space-science.yml \
--limit node1
```

Browser test:

```text
http://192.168.1.154/space-science/
```

Do not continue to all nodes until the pilot deployment succeeds.

---

## 7. Full Deployment

After successful pilot testing:

```bash
ansible-playbook playbooks/deploy-space-science.yml
```

Because `hosts: three_tier_app` is already defined, all three managed nodes are targeted.

---

## 8. Verification

### Check Nginx

```bash
ansible three_tier_app -m command -a \
"systemctl is-active nginx"
```

### Verify the index file

```bash
ansible three_tier_app -m stat -a \
"path=/usr/share/nginx/html/space-science/index.html"
```

### Verify HTTP status

```bash
ansible three_tier_app -m uri -a \
"url=http://localhost/space-science/ status_code=200"
```

### Verify from a browser

```text
http://192.168.1.154/space-science/
http://192.168.1.185/space-science/
http://192.168.1.190/space-science/
```

---

## 9. Idempotency Test

Run the playbook again:

```bash
ansible-playbook playbooks/deploy-space-science.yml
```

On the second run, the play recap should ideally show:

```text
changed=0
failed=0
unreachable=0
```

The `uri`, `stat`, and `debug` tasks do not change the system. The state-management tasks should also remain unchanged when the desired state already exists.

---

## 10. Cleanup Playbook

Create:

```text
/home/ansibleadmin/automation/playbooks/cleanup-space-science.yml
```

Add:

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

### Remove only the website and ZIP file

```bash
ansible-playbook playbooks/cleanup-space-science.yml
```

### Also stop and uninstall Nginx

```bash
ansible-playbook playbooks/cleanup-space-science.yml \
-e "remove_nginx=true"
```

### Test cleanup on node1 first

```bash
ansible-playbook playbooks/cleanup-space-science.yml \
--limit node1
```

---

## 11. Troubleshooting

### Archive structure differs

Inspect the ZIP archive:

```bash
ansible three_tier_app --limit node1 -m command -a \
"unzip -l /tmp/space-science.zip"
```

Update these playbook variables if necessary:

```yaml
website_directory: "/actual/extracted/directory"
website_index: "/actual/extracted/directory/index.html"
website_uri: "http://localhost/actual-directory/"
```

### Website returns HTTP 403

Check permissions and SELinux:

```bash
ansible three_tier_app --limit node1 -m command -a \
"namei -l /usr/share/nginx/html/space-science/index.html"

ansible three_tier_app --limit node1 -m command -a \
"ls -lZ /usr/share/nginx/html/space-science/index.html"
```

### Website is unreachable from the browser

Check Nginx and firewall:

```bash
ansible three_tier_app --limit node1 -m command -a \
"systemctl is-active nginx"

ansible three_tier_app --limit node1 -b -m command -a \
"firewall-cmd --list-services"
```

---

## 12. Recommended Improvements

After mastering this version, improve the project by:

1. Moving variables into `group_vars/all.yml`.
2. Adding tags for `packages`, `deploy`, `verify`, and `cleanup`.
3. Adding a SHA-256 checksum for the downloaded archive.
4. Creating an Nginx server block using a Jinja2 template.
5. Using a handler only when the Nginx configuration changes.
6. Converting the playbook into an Ansible role.
7. Storing the playbook in GitHub and launching it from AWX.

