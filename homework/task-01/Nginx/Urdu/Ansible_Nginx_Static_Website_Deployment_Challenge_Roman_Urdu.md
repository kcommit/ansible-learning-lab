# Ansible Challenge: Nginx ke Sath Static Website Deploy Karein

## Fehrist

1. [Scenario](#1-scenario)
2. [Lab Environment](#2-lab-environment)
3. [Learning Objectives](#3-learning-objectives)
4. [Rules](#4-rules)
5. [Challenge Tasks](#5-challenge-tasks)
6. [Hints](#6-hints)
7. [Required Evidence](#7-required-evidence)
8. [Success Criteria](#8-success-criteria)

---

## 1. Scenario

Aap ki organization teen Rocky Linux servers par **Space Science** static website publish karna chahti hai. Control node se Ansible use karte hue Nginx install karein, template download aur deploy karein, website verify karein, idempotency demonstrate karein aur akhir mein lab cleanup karein.

Template preview:

```text
https://freewebsitetemplates.com/preview/space-science/index.html
```

Direct ZIP download:

```text
https://freewebsitetemplates.com/download/space-science/
```

> Public website publish karne se pehle provider ki current usage aur attribution terms check karein.

---

## 2. Lab Environment

### Control node

| Item | Value |
|---|---|
| Hostname | `ansible-server` |
| IP | `192.168.1.233` |
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

Is challenge ke baad aap yeh kar sakenge:

- `dnf` se packages install karna.
- `service` se Nginx start aur enable karna.
- `firewalld` se HTTP allow karna.
- `get_url` se ZIP archive download karna.
- Deployment se pehle archive inspect karna.
- `unarchive` se website extract karna.
- Ownership, permissions aur SELinux context manage karna.
- `uri` se HTTP response test karna.
- `--limit node1` ke sath safe pilot deployment karna.
- Idempotency demonstrate karna.
- `file` module se cleanup karna.

---

## 4. Rules

1. Commands `/home/ansibleadmin/automation` se chalayein.
2. Changes se pehle active configuration aur inventory verify karein.
3. Pehle complete deployment sirf `node1` par test karein.
4. `node1` successful hone ke baad `three_tier_app` par deploy karein.
5. Jahan suitable Ansible module ho, wahi module use karein.
6. Root access ke liye hi `-b` use karein.
7. SELinux ya firewall disable na karein.
8. Deployment do martaba chala kar second-run output record karein.
9. Screenshots ya command output evidence ke liye rakhein.

---

## 5. Challenge Tasks

### Task 1: Prechecks

Confirm karein:

- Kaunsi `ansible.cfg` active hai?
- Kaunsi inventory active hai?
- Kya teenon nodes `three_tier_app` mein hain?
- Kya Ansible teenon nodes tak pohanch sakta hai?

### Task 2: `node1` par Nginx install karein

Ad-hoc commands se:

- Nginx install karein.
- Service abhi start karein.
- Boot par enable karein.
- Confirm karein ke service active hai.

### Task 3: HTTP traffic allow karein

`firewalld` mein `http` service ko permanently aur immediately enable karein. Firewall ko stop na karein.

### Task 4: Default Nginx page verify karein

Ansible se yeh URL request karein:

```text
http://localhost
```

Status code `200` confirm karein aur browser se `node1` ka default page dekhein.

### Task 5: Template download aur inspect karein

Ansible se:

- ZIP read karne wala package install karein.
- Archive `/tmp/space-science.zip` mein download karein.
- Mode `0644` set karein.
- Verify karein ke file ZIP archive hai, HTML page nahi.
- Archive ke contents list karein.
- Archive ke andar `index.html` locate karein.

### Task 6: Custom website deploy karein

Ansible se:

- Archive ko Nginx document root mein extract karein.
- Ownership `root:root` set karein.
- Safe permissions set karein.
- Correct SELinux context apply karein.
- Deployed `index.html` ka path confirm karein.

Expected directory:

```text
/usr/share/nginx/html/space-science/
```

### Task 7: Website verify karein

Confirm karein:

- Nginx active hai.
- TCP port 80 listen kar raha hai.
- Website HTTP `200` return karti hai.
- `index.html` maujood hai.

Expected URL:

```text
http://192.168.1.154/space-science/
```

### Task 8: Teenon nodes par deploy karein

Pilot successful hone ke baad `three_tier_app` par repeat karein aur verify karein:

```text
http://192.168.1.154/space-science/
http://192.168.1.185/space-science/
http://192.168.1.190/space-science/
```

### Task 9: Idempotency demonstrate karein

Deployment commands dobara chalayein aur note karein:

- Kaun se tasks `changed=false` dikhate hain?
- Kya website available rehti hai?
- Repeatable deployment unnecessary changes kyun avoid karti hai?

### Task 10: Cleanup

Ansible se remove karein:

```text
/usr/share/nginx/html/space-science
/tmp/space-science.zip
```

Evidence ke baad Nginx uninstall karna optional hai.

---

## 6. Hints

### Pilot test

```text
--limit node1
```

### Mumkin modules

```text
ping, dnf, service, firewalld, get_url, stat, command,
shell, unarchive, file, uri
```

### Rocky Linux ka Nginx document root

```text
/usr/share/nginx/html
```

### Managed node par pehle se archive ho

```text
remote_src=yes
```

### HTTP verification

```text
status_code=200
```

### SELinux

SELinux disable na karein. Read-only web content ke liye aam context:

```text
httpd_sys_content_t
```

---

## 7. Required Evidence

1. `ansible three_tier_app -m ping` ka output.
2. Nginx active hone ka output.
3. ZIP archive verification.
4. Archive contents ka output.
5. Deployed `index.html` ka path.
6. `uri` module ka HTTP `200` result.
7. Browser mein custom website ka screenshot.
8. Second run ka idempotency output.
9. Cleanup verification.

---

## 8. Success Criteria

Challenge tab complete hai jab:

- Nginx installed, active aur enabled ho.
- `firewalld` mein HTTP allowed ho.
- Archive successfully download ho.
- Website teenon nodes par deploy ho.
- Har website HTTP `200` return kare.
- SELinux enabled rahe.
- Second run unnecessary changes na kare.
- Cleanup lab resources safely remove kar sake.

