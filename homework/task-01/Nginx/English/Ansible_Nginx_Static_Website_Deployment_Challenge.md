# Ansible Challenge: Deploy a Static Website with Nginx

## Table of Contents

1. [Scenario](#1-scenario)
2. [Lab Environment](#2-lab-environment)
3. [Learning Objectives](#3-learning-objectives)
4. [Rules](#4-rules)
5. [Challenge Tasks](#5-challenge-tasks)
6. [Hints](#6-hints)
7. [Required Evidence](#7-required-evidence)
8. [Success Criteria](#8-success-criteria)
9. [Optional Advanced Tasks](#9-optional-advanced-tasks)

---

## 1. Scenario

Your organization wants to publish a static **Space Science** website on three Rocky Linux web servers. Use Ansible from the control node to install Nginx, download the website archive, deploy it, verify it, demonstrate idempotency, and clean up the lab.

Template preview:

```text
https://freewebsitetemplates.com/preview/space-science/index.html
```

Direct ZIP download:

```text
https://freewebsitetemplates.com/download/space-science/
```

> Review the template provider's current usage and attribution terms before publishing the website publicly.

---

## 2. Lab Environment

### Control node

| Item | Value |
|---|---|
| Hostname | `ansible-server` |
| IP address | `192.168.1.233` |
| User | `ansibleadmin` |
| Project directory | `/home/ansibleadmin/automation` |

### Managed nodes

| Inventory name | IP address | Group |
|---|---|---|
| `node1` | `192.168.1.154` | `web` |
| `node2` | `192.168.1.185` | `app` |
| `node3` | `192.168.1.190` | `db` |

Parent group:

```text
three_tier_app
```

---

## 3. Learning Objectives

After completing this challenge, you should be able to:

- Install packages with the `dnf` module.
- Start and enable a service with the `service` module.
- Open HTTP access with the `firewalld` module.
- Download an archive with the `get_url` module.
- Inspect a ZIP archive before deployment.
- Extract an archive with the `unarchive` module.
- Apply suitable ownership, permissions, and SELinux context.
- Test a website with the `uri` module.
- Use `--limit node1` for a safe pilot deployment.
- Demonstrate Ansible idempotency.
- Remove deployed resources with the `file` module.

---

## 4. Rules

1. Run all Ansible commands from `/home/ansibleadmin/automation`.
2. Confirm the active configuration and inventory before making changes.
3. Test the complete deployment on `node1` first.
4. Deploy to `three_tier_app` only after `node1` passes verification.
5. Use Ansible modules whenever a suitable module exists.
6. Use privilege escalation only for tasks that require root access.
7. Do not disable SELinux or stop the firewall.
8. Run the deployment twice and record the second result.
9. Keep screenshots or command output as evidence.

---

## 5. Challenge Tasks

### Task 1: Perform prechecks

Confirm:

- Which `ansible.cfg` file is active.
- Which inventory file is active.
- Whether `node1`, `node2`, and `node3` are members of `three_tier_app`.
- Whether Ansible can reach all three nodes.

### Task 2: Install Nginx on `node1`

Use an Ansible ad-hoc command to:

- Install Nginx.
- Start Nginx immediately.
- Enable Nginx at boot.
- Confirm that the service is active.

### Task 3: Allow HTTP traffic

Use Ansible to enable the `http` service permanently and immediately in `firewalld`.

Do not stop or disable the firewall.

### Task 4: Verify the default Nginx website

Use Ansible to request:

```text
http://localhost
```

The task must confirm HTTP status code `200`.

Also open the default page from a browser using the IP address of `node1`.

### Task 5: Download and inspect the template

Use Ansible to:

- Install the tool required to read ZIP archives.
- Download the template to `/tmp/space-science.zip`.
- Set file mode `0644`.
- Verify that the downloaded object is a ZIP archive.
- List the archive contents.
- Identify the location of `index.html` inside the archive.

### Task 6: Deploy the custom website

Use Ansible to:

- Extract the archive under the Nginx document root.
- Ensure the website files are owned by `root:root`.
- Ensure directories are traversable and files are readable.
- Restore or assign the correct SELinux context.
- Confirm the deployed `index.html` path.

Expected website location:

```text
/usr/share/nginx/html/space-science/
```

### Task 7: Verify the custom website

Use Ansible to verify:

- Nginx is active.
- TCP port 80 is listening.
- The website returns status code `200`.
- The deployed `index.html` exists.

Expected browser URL for `node1`:

```text
http://192.168.1.154/space-science/
```

### Task 8: Deploy to all three nodes

After the pilot deployment succeeds, repeat the deployment against:

```text
three_tier_app
```

Verify the following URLs:

```text
http://192.168.1.154/space-science/
http://192.168.1.185/space-science/
http://192.168.1.190/space-science/
```

### Task 9: Demonstrate idempotency

Run the deployment commands a second time.

Document:

- Which tasks report `changed=false`?
- Does the website remain available?
- Why should a repeatable deployment avoid unnecessary changes?

### Task 10: Clean up the lab

Use Ansible to remove:

- `/usr/share/nginx/html/space-science`
- `/tmp/space-science.zip`

Optionally uninstall Nginx after collecting evidence.

Verify that the custom website is no longer available.

---

## 6. Hints

### Hint 1: Pilot deployment

Use:

```text
--limit node1
```

### Hint 2: Required modules

You will probably need:

```text
ping
dnf
service
firewalld
get_url
stat
command
unarchive
file
uri
shell
```

### Hint 3: Nginx document root on Rocky Linux

```text
/usr/share/nginx/html
```

### Hint 4: Remote archive

When the ZIP archive already exists on a managed node, the `unarchive` module needs:

```text
remote_src=yes
```

### Hint 5: HTTP verification

The `uri` module can verify:

```text
status_code=200
```

### Hint 6: SELinux

Do not disable SELinux. Content served by Nginx normally needs the `httpd_sys_content_t` context.

### Hint 7: Idempotency

`state=present`, `state=started`, and `enabled=yes` describe the required final state rather than blindly repeating commands.

---

## 7. Required Evidence

Submit or record:

1. Output of `ansible three_tier_app -m ping`.
2. Output confirming Nginx is active.
3. Output showing the downloaded file is a ZIP archive.
4. Output listing the ZIP archive contents.
5. Output showing the deployed `index.html` path.
6. Output from the `uri` module showing status `200`.
7. Browser screenshot of the custom website.
8. Second-run output demonstrating idempotency.
9. Cleanup verification.

---

## 8. Success Criteria

The challenge is complete when:

- Nginx is installed, active, and enabled.
- HTTP is allowed through `firewalld`.
- The archive downloads successfully.
- The custom website is deployed on all three nodes.
- Each website returns HTTP `200`.
- SELinux remains enabled.
- The second run does not make unnecessary changes.
- Cleanup can remove the lab resources safely.

---

## 9. Optional Advanced Tasks

1. Convert the ad-hoc commands into a playbook.
2. Place deployment values in variables.
3. Add tags such as `packages`, `deploy`, `verify`, and `cleanup`.
4. Use a handler where a real Nginx configuration change requires a reload.
5. Add a checksum to verify the downloaded archive.
6. Back up the existing document root before deployment.
7. Create a dedicated Nginx server block for the site.

