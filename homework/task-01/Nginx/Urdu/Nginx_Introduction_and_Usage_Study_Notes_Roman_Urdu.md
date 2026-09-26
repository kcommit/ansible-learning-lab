# Nginx: Introduction aur Usage — Roman Urdu Study Notes

## Fehrist (Table of Contents)

1. [Nginx kya hai?](#1-nginx-kya-hai)
2. [Nginx kaise kaam karta hai?](#2-nginx-kaise-kaam-karta-hai)
3. [Nginx ke aam istemal](#3-nginx-ke-aam-istemal)
4. [Hamare Ansible lab mein Nginx](#4-hamare-ansible-lab-mein-nginx)
5. [Rocky Linux par aham Nginx paths](#5-rocky-linux-par-aham-nginx-paths)
6. [Nginx ko manually install aur manage karna](#6-nginx-ko-manually-install-aur-manage-karna)
7. [Ansible se Nginx install aur manage karna](#7-ansible-se-nginx-install-aur-manage-karna)
8. [Website ko test karna](#8-website-ko-test-karna)
9. [Restart aur reload ka farq](#9-restart-aur-reload-ka-farq)
10. [Nginx as a reverse proxy](#10-nginx-as-a-reverse-proxy)
11. [Nginx as a load balancer](#11-nginx-as-a-load-balancer)
12. [Logs aur troubleshooting](#12-logs-aur-troubleshooting)
13. [Ansible mein idempotency](#13-ansible-mein-idempotency)
14. [Removal aur cleanup](#14-removal-aur-cleanup)
15. [Practice exercises](#15-practice-exercises)
16. [Quick revision](#16-quick-revision)

---

## 1. Nginx kya hai?

**Nginx** ko **Engine-X** pronounce kiya jata hai. Yeh ek fast aur lightweight web-server application hai. Yeh clients—aksar web browsers—se HTTP ya HTTPS requests receive karta hai aur website ka content wapas bhejta hai, jaise HTML, CSS, JavaScript, images aur download files.

**Ek line ki definition:** Nginx ek high-performance web server hai jo websites host karne, application traffic ko reverse proxy karne, load balance karne, HTTPS handle karne aur content cache karne ke liye use hota hai.

```text
Browser → HTTP/HTTPS request → Nginx → Website content
```

Jab koi user yeh address open karta hai:

```text
http://192.168.1.154
```

to Nginx aam tor par TCP port `80` par request receive karta hai aur apne document root se website content serve karta hai.

---

## 2. Nginx kaise kaam karta hai?

1. Client HTTP ya HTTPS request bhejta hai.
2. Nginx aam tor par port `80` ya `443` par request sunta hai.
3. Nginx apni configuration check karta hai ke request ko kaise handle karna hai.
4. Yeh static file bhej sakta hai, request backend application ko forward kar sakta hai, ya kisi doosre server ko de sakta hai.
5. Nginx client ko HTTP response wapas bhej deta hai.

| Port | Protocol | Maqsad |
|---:|---|---|
| `80` | HTTP | Baghair encryption web traffic |
| `443` | HTTPS | TLS-encrypted web traffic |

---

## 3. Nginx ke aam istemal

| Istemal | Wazahat |
|---|---|
| Web server | Static HTML, CSS, JavaScript, images aur downloads host karta hai |
| Reverse proxy | Request receive karke backend application ko forward karta hai |
| Load balancer | Requests ko multiple application servers mein distribute karta hai |
| HTTPS termination | TLS certificates aur encrypted connections handle karta hai |
| Caching | Bar-bar mangwaya jane wala content save karke performance improve karta hai |
| Access control | IP address, password ya doosre rules se access restrict karta hai |

### Static website hosting

Nginx document root mein rakhi files ko seedha serve kar sakta hai:

```text
/usr/share/nginx/html/
```

### Reverse proxy

Nginx port `80` ya `443` par request receive karke usay kisi application ke port, jaise `8080`, par forward kar sakta hai.

### Load balancing

Nginx incoming requests ko multiple backend servers ke darmiyan distribute karta hai. Is se availability aur capacity improve hoti hai.

---

## 4. Hamare Ansible lab mein Nginx

| Role | Host ya group |
|---|---|
| Ansible control node | `ansible-server` |
| Managed nodes | `node1`, `node2`, `node3` |
| Inventory group | `three_tier_app` |
| Remote user | `ansibleadmin` |
| Operating system | Rocky Linux 9 |

Ansible tamam managed nodes par Nginx install kar sakta hai, service start kar sakta hai, firewall open kar sakta hai, website deploy kar sakta hai aur HTTP response verify kar sakta hai.

---

## 5. Rocky Linux par aham Nginx paths

| Path | Maqsad |
|---|---|
| `/etc/nginx/nginx.conf` | Nginx ki main configuration file |
| `/etc/nginx/conf.d/` | Additional configuration files ki directory |
| `/usr/share/nginx/html/` | Default website document root |
| `/var/log/nginx/access.log` | Client requests ka record |
| `/var/log/nginx/error.log` | Errors aur warnings ka record |
| `/usr/lib/systemd/system/nginx.service` | Package ke sath aane wali systemd service unit |

Packaged systemd unit ko seedha edit na karein. Agar override chahiye ho to yeh use karein:

```bash
sudo systemctl edit nginx
```

---

## 6. Nginx ko manually install aur manage karna

### Nginx install karein

```bash
sudo dnf install nginx -y
```

### Service start aur enable karein

```bash
sudo systemctl enable --now nginx
```

`--now` do kaam karta hai:

- Nginx ko foran start karta hai.
- Nginx ko system boot par automatically start hone ke liye enable karta hai.

### Service check karein

```bash
systemctl status nginx
systemctl is-active nginx
systemctl is-enabled nginx
```

### Firewalld mein HTTP allow karein

```bash
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --reload
sudo firewall-cmd --list-services
```

### Simple website banayein

```bash
echo '<h1>Welcome to my Nginx lab</h1>' | sudo tee /usr/share/nginx/html/index.html
```

### Configuration syntax test karein

```bash
sudo nginx -t
```

---

## 7. Ansible se Nginx install aur manage karna

In examples mein aapki preference ke mutabiq short module names use kiye gaye hain.

### Connectivity confirm karein

```bash
ansible three_tier_app -m ping
```

### Nginx install karein

```bash
ansible three_tier_app -b -m dnf \
  -a "name=nginx state=present"
```

- `three_tier_app`: target inventory group hai.
- `-b`: privilege escalation, yani `sudo`, use karta hai.
- `-m dnf`: `dnf` module select karta hai.
- `name=nginx`: Nginx package ko manage karta hai.
- `state=present`: package missing ho to install karta hai.

### Nginx start aur enable karein

```bash
ansible three_tier_app -b -m service \
  -a "name=nginx state=started enabled=yes"
```

### Firewall mein HTTP open karein

```bash
ansible three_tier_app -b -m firewalld \
  -a "service=http permanent=yes immediate=yes state=enabled"
```

### Simple home page deploy karein

```bash
ansible three_tier_app -b -m copy \
  -a 'content="<h1>Welcome to my Ansible Nginx lab</h1>\n" dest=/usr/share/nginx/html/index.html owner=root group=root mode=0644'
```

### Configuration validate karein

```bash
ansible three_tier_app -b -m command -a "nginx -t"
```

### Service state check karein

```bash
ansible three_tier_app -b -m command \
  -a "systemctl is-active nginx"
```

---

## 8. Website ko test karna

### Control node se `curl` ke zariye test

```bash
curl http://192.168.1.154
curl http://192.168.1.185
curl http://192.168.1.190
```

### Ansible `uri` module se test

```bash
ansible three_tier_app -m uri \
  -a 'url=http://{{ ansible_host }} status_code=200 return_content=yes'
```

HTTP status code `200` ka matlab hai request successfully complete hui.

### Browser mein test

```text
http://192.168.1.154
http://192.168.1.185
http://192.168.1.190
```

---

## 9. Restart aur reload ka farq

| Operation | Matlab | Kab use karein? |
|---|---|---|
| `restart` | Service ko stop karke dobara start karta hai | Recovery ya full restart wali change par |
| `reload` | Service ko band kiye baghair configuration dobara read karta hai | Valid configuration change ke baad |

```bash
sudo systemctl restart nginx
sudo systemctl reload nginx
```

Aham baatein:

- HTML, CSS, JavaScript ya image change karne par aam tor par restart ya reload ki zaroorat nahi hoti.
- Nginx configuration change karne ke baad pehle `nginx -t` chalayein, phir reload karein.

Safe sequence:

```bash
sudo nginx -t && sudo systemctl reload nginx
```

---

## 10. Nginx as a reverse proxy

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

Is example mein:

- Client Nginx ke port `80` se connect karta hai.
- Nginx request application ke port `8080` ko forward karta hai.
- Proxy headers backend ko client aur protocol ki useful information dete hain.

Configuration ko `/etc/nginx/conf.d/` mein save karne ke baad:

```bash
sudo nginx -t
sudo systemctl reload nginx
```

---

## 11. Nginx as a load balancer

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

Nginx `upstream` block mein defined servers ko requests bhejta hai. Default tor par round-robin method use hota hai, yani requests bari bari servers ko milti hain.

---

## 12. Logs aur troubleshooting

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

| Problem | Kya check karein? |
|---|---|
| Website open nahi hoti | Nginx service, firewall, IP address aur listening ports |
| `403 Forbidden` | File permissions, directory permissions aur SELinux context |
| `404 Not Found` | URL aur document-root file path |
| Nginx start nahi hota | `nginx -t` aur `journalctl -u nginx` |
| Purana page nazar aata hai | Browser/proxy cache aur correct file path |
| Reverse proxy par `502` | Backend application running aur reachable hai ya nahi |

### SELinux checks

```bash
getenforce
ls -lZ /usr/share/nginx/html
sudo restorecon -Rv /usr/share/nginx/html
```

Troubleshooting ke pehle step mein SELinux disable na karein. Pehle labels check aur correct karein.

---

## 13. Ansible mein idempotency

Idempotency ka matlab hai ke same automation bar-bar chalane par system required state mein rahe aur unnecessary changes na hon.

```bash
ansible three_tier_app -b -m dnf \
  -a "name=nginx state=present"
```

Expected behavior:

- Pehli run: Nginx install hoga, is liye result aam tor par `CHANGED` hoga.
- Doosri run: Nginx pehle se installed hoga, is liye result aam tor par `SUCCESS` aur `changed=false` hoga.

`dnf`, `service`, `copy`, `file` aur `firewalld` jaise modules state manage karne ke liye banaye gaye hain. Arbitrary `shell` commands aksar har run par `CHANGED` show karte hain jab tak change detection khud define na ki jaye.

---

## 14. Removal aur cleanup

### Test page remove karein

```bash
ansible three_tier_app -b -m file \
  -a "path=/usr/share/nginx/html/index.html state=absent"
```

### Nginx stop aur disable karein

```bash
ansible three_tier_app -b -m service \
  -a "name=nginx state=stopped enabled=no"
```

### Nginx package remove karein

```bash
ansible three_tier_app -b -m dnf \
  -a "name=nginx state=absent"
```

### HTTP firewall rule remove karein

```bash
ansible three_tier_app -b -m firewalld \
  -a "service=http permanent=yes immediate=yes state=disabled"
```

Cleanup commands sirf tab use karein jab lab mein complete removal required ho. Package remove karne ke baad kuch modified configuration ya content files reh sakti hain, is liye verify zaroor karein.

---

## 15. Practice exercises

### Exercise 1: Manual installation

`node1` par:

1. Nginx install karein.
2. Isay enable aur start karein.
3. Firewall mein HTTP open karein.
4. Custom `index.html` banayein.
5. `curl` se test karein.

### Exercise 2: Ansible ad-hoc deployment

Ansible se wohi kaam `three_tier_app` group ke tamam nodes par karein.

### Exercise 3: Idempotency

Package, service, firewall aur copy commands do martaba chalayein. `changed=true` aur `changed=false` ke results compare karein.

### Exercise 4: Troubleshooting

`node1` par Nginx stop karein, website test karein, failure identify karein aur service restore karein:

```bash
ansible node1 -b -m service -a "name=nginx state=stopped"
ansible node1 -m uri -a 'url=http://{{ ansible_host }} status_code=200'
ansible node1 -b -m service -a "name=nginx state=started"
```

Playbook ko sirf ek managed node par safely test karne ke liye `--limit node1` use karein:

```bash
ansible-playbook deploy-nginx.yml --limit node1
```

### Exercise 5: Sawalat

1. Nginx kya hai?
2. Document root kya hota hai?
3. `restart` aur `reload` mein kya farq hai?
4. Reload se pehle `nginx -t` kyun chalana chahiye?
5. Reverse proxy kya hota hai?
6. Package aur service management ke liye `--become` kyun chahiye?
7. HTTP error ke liye kaun se logs check karenge?
8. Idempotency ka kya matlab hai?

---

## 16. Quick revision

| Sawal | Mukhtasar jawab |
|---|---|
| Nginx ko kaise pronounce karte hain? | Engine-X |
| Iska main maqsad kya hai? | Web content serve aur requests proxy karna |
| Default HTTP port? | `80` |
| Default HTTPS port? | `443` |
| Rocky Linux ka default document root? | `/usr/share/nginx/html` |
| Main configuration file? | `/etc/nginx/nginx.conf` |
| Configuration test command? | `nginx -t` |
| Access log? | `/var/log/nginx/access.log` |
| Error log? | `/var/log/nginx/error.log` |
| Lab mein installation module? | `dnf` |
| `service` module ka maqsad? | Nginx ko start, stop aur enable karna |
| `firewalld` module ka maqsad? | HTTP/HTTPS traffic allow karna |
| `uri` module ka maqsad? | HTTP/HTTPS endpoint test karna |

---

## Final khulasa

Nginx sirf ek simple web server nahi hai. Yeh static websites serve kar sakta hai, application traffic ko protect aur forward kar sakta hai, backend servers ke darmiyan requests distribute kar sakta hai, HTTPS handle kar sakta hai, content cache kar sakta hai aur web traffic ke logs rakh sakta hai. Ansible ke zariye Nginx package, service, firewall access, website content aur validation ko multiple servers par ek jaisi aur repeatable configuration ke sath manage kiya ja sakta hai.
