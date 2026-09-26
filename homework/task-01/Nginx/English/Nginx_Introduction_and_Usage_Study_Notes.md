# Nginx: Introduction and Usage — Study Notes

## Table of Contents

1. [What is Nginx?](#1-what-is-nginx)
2. [How Nginx works](#2-how-nginx-works)
3. [Common uses of Nginx](#3-common-uses-of-nginx)
4. [Nginx in an Ansible lab](#4-nginx-in-an-ansible-lab)
5. [Important Nginx paths on Rocky Linux](#5-important-nginx-paths-on-rocky-linux)
6. [Install and manage Nginx manually](#6-install-and-manage-nginx-manually)
7. [Install and manage Nginx with Ansible](#7-install-and-manage-nginx-with-ansible)
8. [Test the website](#8-test-the-website)
9. [Restart versus reload](#9-restart-versus-reload)
10. [Nginx as a reverse proxy](#10-nginx-as-a-reverse-proxy)
11. [Nginx as a load balancer](#11-nginx-as-a-load-balancer)
12. [Logs and troubleshooting](#12-logs-and-troubleshooting)
13. [Idempotency in Ansible](#13-idempotency-in-ansible)
14. [Removal and cleanup](#14-removal-and-cleanup)
15. [Practice exercises](#15-practice-exercises)
16. [Quick revision](#16-quick-revision)

---

## 1. What is Nginx?

**Nginx**—pronounced **Engine-X**—is a fast and lightweight web server. It receives HTTP or HTTPS requests from clients and returns website content such as HTML, CSS, JavaScript, images, and downloadable files.

**One-line definition:** Nginx is a high-performance web server commonly used to host websites, reverse-proxy application traffic, balance load, handle HTTPS, and cache content.

Example:

```text
Browser → HTTP/HTTPS request → Nginx → Website content
```

When a user opens the following address:

```text
http://192.168.1.154
```

Nginx normally receives the request on TCP port `80` and serves content from its document root.

---

## 2. How Nginx works

1. A client sends an HTTP or HTTPS request.
2. Nginx listens for the request, normally on port `80` or `443`.
3. Nginx checks its configuration to decide how to handle the request.
4. It may return a static file, forward the request to an application, or distribute it to another server.
5. Nginx sends an HTTP response back to the client.

Common ports:

| Port | Protocol | Purpose |
|---:|---|---|
| `80` | HTTP | Unencrypted web traffic |
| `443` | HTTPS | TLS-encrypted web traffic |

---

## 3. Common uses of Nginx

| Usage | Description |
|---|---|
| Web server | Hosts static HTML, CSS, JavaScript, images, and downloads |
| Reverse proxy | Receives requests and forwards them to backend applications |
| Load balancer | Distributes requests among multiple application servers |
| HTTPS termination | Handles TLS certificates and encrypted connections |
| Caching | Stores frequently requested content to improve performance |
| Access control | Restricts access by IP address, password, or other rules |

### Static website hosting

Nginx can directly serve files located under its document root:

```text
/usr/share/nginx/html/
```

### Reverse proxy

Nginx can accept requests on port `80` or `443` and forward them to an application running on a different port, such as `8080`.

### Load balancing

Nginx can distribute incoming requests among several backend servers, improving availability and capacity.

---

## 4. Nginx in an Ansible lab

This study lab uses:

| Role | Host or group |
|---|---|
| Ansible control node | `ansible-server` |
| Managed nodes | `node1`, `node2`, `node3` |
| Inventory group | `three_tier_app` |
| Remote user | `ansibleadmin` |
| Operating system | Rocky Linux 9 |

Ansible can install Nginx, start its service, open the firewall, deploy website files, and verify the HTTP response across all managed nodes.

---

## 5. Important Nginx paths on Rocky Linux

| Path | Purpose |
|---|---|
| `/etc/nginx/nginx.conf` | Main Nginx configuration file |
| `/etc/nginx/conf.d/` | Directory for additional configuration files |
| `/usr/share/nginx/html/` | Default website document root |
| `/var/log/nginx/access.log` | Records client requests |
| `/var/log/nginx/error.log` | Records errors and warnings |
| `/usr/lib/systemd/system/nginx.service` | Packaged systemd service unit |

Avoid editing the packaged systemd unit directly. Use `systemctl edit nginx` if a systemd override is required.

---

## 6. Install and manage Nginx manually

### Install Nginx

```bash
sudo dnf install nginx -y
```

### Start and enable the service

```bash
sudo systemctl enable --now nginx
```

`--now` performs both actions:

- Starts Nginx immediately.
- Enables Nginx to start automatically during boot.

### Check the service

```bash
systemctl status nginx
systemctl is-active nginx
systemctl is-enabled nginx
```

### Allow HTTP through firewalld

```bash
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --reload
sudo firewall-cmd --list-services
```

### Create a simple website

```bash
echo '<h1>Welcome to my Nginx lab</h1>' | sudo tee /usr/share/nginx/html/index.html
```

### Test configuration syntax

```bash
sudo nginx -t
```

---

## 7. Install and manage Nginx with Ansible

The following examples use short module names as preferred in this lab.

### Confirm connectivity

```bash
ansible three_tier_app -m ping
```

### Install Nginx

```bash
ansible three_tier_app -b -m dnf \
  -a "name=nginx state=present"
```

- `three_tier_app`: target inventory group
- `-b`: use privilege escalation (`sudo`)
- `-m dnf`: use the `dnf` module
- `name=nginx`: manage the Nginx package
- `state=present`: install it if it is missing

### Start and enable Nginx

```bash
ansible three_tier_app -b -m service \
  -a "name=nginx state=started enabled=yes"
```

### Open HTTP in the firewall

```bash
ansible three_tier_app -b -m firewalld \
  -a "service=http permanent=yes immediate=yes state=enabled"
```

### Deploy a simple home page

```bash
ansible three_tier_app -b -m copy \
  -a 'content="<h1>Welcome to my Ansible Nginx lab</h1>\n" dest=/usr/share/nginx/html/index.html owner=root group=root mode=0644'
```

### Validate the configuration

```bash
ansible three_tier_app -b -m command -a "nginx -t"
```

### Check the service state

```bash
ansible three_tier_app -b -m command \
  -a "systemctl is-active nginx"
```

---

## 8. Test the website

### Test from the control node with curl

```bash
curl http://192.168.1.154
curl http://192.168.1.185
curl http://192.168.1.190
```

### Test with the Ansible `uri` module

```bash
ansible three_tier_app -m uri \
  -a 'url=http://{{ ansible_host }} status_code=200 return_content=yes'
```

An HTTP status code of `200` means the request was successful.

### Test in a browser

```text
http://192.168.1.154
http://192.168.1.185
http://192.168.1.190
```

---

## 9. Restart versus reload

| Operation | Meaning | When to use it |
|---|---|---|
| `restart` | Stops and starts the service | Recovery or changes requiring a full restart |
| `reload` | Rereads configuration with minimal interruption | After a valid configuration change |

Commands:

```bash
sudo systemctl restart nginx
sudo systemctl reload nginx
```

Important points:

- Changing HTML, CSS, JavaScript, or image files normally requires neither restart nor reload.
- After changing Nginx configuration, run `nginx -t` before reloading.

Safe sequence:

```bash
sudo nginx -t && sudo systemctl reload nginx
```

---

## 10. Nginx as a reverse proxy

Example configuration:

```nginx
server {
    listen 80;
    server_name app.example.com;

    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

In this example:

- Clients connect to Nginx on port `80`.
- Nginx forwards requests to the application on port `8080`.
- Proxy headers pass useful client and protocol information to the backend.

After saving a configuration file under `/etc/nginx/conf.d/`:

```bash
sudo nginx -t
sudo systemctl reload nginx
```

---

## 11. Nginx as a load balancer

Example:

```nginx
upstream application_servers {
    server 192.168.1.185:8080;
    server 192.168.1.190:8080;
}

server {
    listen 80;

    location / {
        proxy_pass http://application_servers;
    }
}
```

Nginx sends requests to the servers defined in the `upstream` block. By default, it uses round-robin distribution.

---

## 12. Logs and troubleshooting

### Useful commands

```bash
sudo nginx -t
systemctl status nginx
sudo journalctl -u nginx --no-pager -n 50
sudo tail -f /var/log/nginx/access.log
sudo tail -f /var/log/nginx/error.log
sudo ss -tlnp | grep ':80\|:443'
sudo firewall-cmd --list-all
curl -I http://localhost
```

### Common problems

| Problem | Checks |
|---|---|
| Website does not open | Check Nginx service, firewall, IP address, and listening ports |
| `403 Forbidden` | Check file permissions, directory permissions, and SELinux context |
| `404 Not Found` | Check the URL and document-root file path |
| Nginx will not start | Run `nginx -t` and inspect `journalctl -u nginx` |
| Old page appears | Check browser/proxy cache and confirm the correct file was updated |
| Reverse proxy gives `502` | Confirm the backend application is running and reachable |

### SELinux checks

```bash
getenforce
ls -lZ /usr/share/nginx/html
sudo restorecon -Rv /usr/share/nginx/html
```

Do not disable SELinux as the first troubleshooting step. Check and correct labels instead.

---

## 13. Idempotency in Ansible

Idempotency means that running the same automation repeatedly keeps the required state without making unnecessary changes.

For example:

```bash
ansible three_tier_app -b -m dnf \
  -a "name=nginx state=present"
```

Expected behavior:

- First run: Nginx is installed, so the result is normally `CHANGED`.
- Second run: Nginx is already installed, so the result should normally be `SUCCESS` with `changed=false`.

Modules such as `dnf`, `service`, `copy`, `file`, and `firewalld` are designed to manage state. Arbitrary `shell` commands commonly report `CHANGED` unless change detection is explicitly defined.

---

## 14. Removal and cleanup

### Remove the deployed test page

```bash
ansible three_tier_app -b -m file \
  -a "path=/usr/share/nginx/html/index.html state=absent"
```

### Stop and disable Nginx

```bash
ansible three_tier_app -b -m service \
  -a "name=nginx state=stopped enabled=no"
```

### Remove the Nginx package

```bash
ansible three_tier_app -b -m dnf \
  -a "name=nginx state=absent"
```

### Remove the firewall rule if it is no longer required

```bash
ansible three_tier_app -b -m firewalld \
  -a "service=http permanent=yes immediate=yes state=disabled"
```

Use cleanup commands only when the lab requires complete removal. Removing the package may leave some modified configuration or content files, so verify afterward.

---

## 15. Practice exercises

### Exercise 1: Manual installation

On `node1`:

1. Install Nginx.
2. Enable and start it.
3. Open HTTP in the firewall.
4. Create a custom `index.html`.
5. Test it with `curl`.

### Exercise 2: Ansible ad-hoc deployment

Use Ansible to perform the same tasks on `three_tier_app`.

### Exercise 3: Idempotency

Run the package, service, firewall, and copy commands twice. Compare `changed=true` with `changed=false`.

### Exercise 4: Troubleshooting

Stop Nginx on `node1`, test the website, identify the failure, and restore the service:

```bash
ansible node1 -b -m service -a "name=nginx state=stopped"
ansible node1 -m uri -a 'url=http://{{ ansible_host }} status_code=200'
ansible node1 -b -m service -a "name=nginx state=started"
```

Use `--limit node1` when testing a playbook safely on only one managed node:

```bash
ansible-playbook deploy-nginx.yml --limit node1
```

### Exercise 5: Documentation questions

Answer these questions:

1. What is Nginx?
2. What is a document root?
3. What is the difference between `restart` and `reload`?
4. Why should `nginx -t` be run before a reload?
5. What is a reverse proxy?
6. Why is `--become` required for package and service management?
7. Which logs would you inspect for an HTTP error?
8. What does idempotency mean?

---

## 16. Quick revision

| Question | Short answer |
|---|---|
| How is Nginx pronounced? | Engine-X |
| What is its main purpose? | Serving web content and proxying requests |
| Default HTTP port? | `80` |
| Default HTTPS port? | `443` |
| Default Rocky Linux document root? | `/usr/share/nginx/html` |
| Main configuration file? | `/etc/nginx/nginx.conf` |
| Configuration test command? | `nginx -t` |
| Access log? | `/var/log/nginx/access.log` |
| Error log? | `/var/log/nginx/error.log` |
| Install module used in this lab? | `dnf` |
| Service module purpose? | Starts, stops, and enables Nginx |
| Firewall module purpose? | Permits HTTP/HTTPS traffic |
| `uri` module purpose? | Tests an HTTP/HTTPS endpoint |

---

## Final summary

Nginx is more than a simple web server. It can serve static sites, protect and forward traffic to applications, distribute requests among backend servers, handle HTTPS, cache content, and record web traffic. In an Ansible environment, its package, service, firewall access, website content, and validation can all be managed consistently across many servers.
