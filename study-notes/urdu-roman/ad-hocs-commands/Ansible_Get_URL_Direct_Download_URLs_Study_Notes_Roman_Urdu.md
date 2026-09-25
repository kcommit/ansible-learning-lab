# Ansible `get_url`: Direct Download URLs - Roman Urdu Study Notes

## Fehrist (Table of Contents)

1. [`get_url` module kya hai?](#1-get_url-module-kya-hai)
2. [Kya `raw.githubusercontent.com` har URL ke liye zaroori hai?](#2-kya-rawgithubusercontentcom-har-url-ke-liye-zaroori-hai)
3. [GitHub webpage URL aur raw file URL ka farq](#3-github-webpage-url-aur-raw-file-url-ka-farq)
4. [Apni Ansible lab ka example](#4-apni-ansible-lab-ka-example)
5. [Doosri websites ke examples](#5-doosri-websites-ke-examples)
6. [URL ko use karne se pehle test kaise karein](#6-url-ko-use-karne-se-pehle-test-kaise-karein)
7. [`get_url` ke aham arguments](#7-get_url-ke-aham-arguments)
8. [Downloaded file verify karne ke commands](#8-downloaded-file-verify-karne-ke-commands)
9. [Aam ghaltiyan](#9-aam-ghaltiyan)
10. [Khulasa](#10-khulasa)

---

## 1. `get_url` module kya hai?

Ansible ka `get_url` module HTTP, HTTPS ya FTP URL se file download karke managed node par rakhta hai.

Basic syntax:

```bash
ansible <host-pattern> -m get_url -a \
'url=<DIRECT_FILE_URL> dest=<DESTINATION_PATH>'
```

Example:

```bash
ansible three_tier_app -m get_url -a \
'url=https://example.com/files/config.conf dest=/tmp/config.conf mode=0644'
```

Is command mein:

- `three_tier_app` inventory ka host group hai.
- `-m get_url` se `get_url` module select hota hai.
- `url=` source file ka URL batata hai.
- `dest=` managed node par destination path batata hai.
- `mode=0644` downloaded file ki permissions set karta hai.

---

## 2. Kya `raw.githubusercontent.com` har URL ke liye zaroori hai?

Nahi. `raw.githubusercontent.com` sirf GitHub se raw file contents download karne ke liye use hota hai. Yeh Ansible ki requirement nahi hai aur har website ke URL ke liye zaroori nahi.

Asal requirement yeh hai ke URL actual file content return kare, sirf file dikhane wala HTML webpage nahi.

| URL ki qisam | `get_url` ke liye theek hai? | Wazahat |
|---|---:|---|
| GitHub `blob` webpage | Aam tor par nahi | GitHub ka webpage aur interface return karta hai |
| GitHub raw URL | Haan | Actual file contents return karta hai |
| Doosri website ka direct file URL | Haan | File ko seedha return karta hai |
| Login ya download webpage | Aam tor par nahi | HTML, authentication page ya redirects return kar sakta hai |

> `usercontent` sirf GitHub ke raw-content domain name ka hissa hai. Yeh Ansible ka keyword, argument ya requirement nahi hai.

---

## 3. GitHub webpage URL aur raw file URL ka farq

### GitHub webpage URL

Yeh URL file ko GitHub ke webpage ke andar dikhata hai:

```text
https://github.com/kcommit/ansible-learning-lab/blob/main/misc/inventory/nodes.ini
```

Is URL mein `/blob/` maujood hai. Yeh aam tor par webpage URL hota hai, direct file URL nahi.

### GitHub raw file URL

Isi file ka direct-download URL yeh hai:

```text
https://raw.githubusercontent.com/kcommit/ansible-learning-lab/main/misc/inventory/nodes.ini
```

Raw URL GitHub interface ke baghair `nodes.ini` ka actual content return karta hai.

### GitHub par asaan tareeqa

1. GitHub par file open karein.
2. **Raw** button select karein.
3. Browser ke address bar se URL copy karein.
4. Is URL ko `get_url` module ke sath use karein.

---

## 4. Apni Ansible lab ka example

Pehle sirf `node1` par `nodes.ini` ko `/tmp` mein download karein:

```bash
ansible three_tier_app --limit node1 -m get_url -a \
'url=https://raw.githubusercontent.com/kcommit/ansible-learning-lab/main/misc/inventory/nodes.ini dest=/tmp/nodes.ini mode=0644'
```

Jab command `node1` par sahi kaam kare, phir teenon managed nodes par chalayein:

```bash
ansible three_tier_app -m get_url -a \
'url=https://raw.githubusercontent.com/kcommit/ansible-learning-lab/main/misc/inventory/nodes.ini dest=/tmp/nodes.ini mode=0644'
```

Agar destination par root permission chahiye ho to `-b` use karein:

```bash
ansible three_tier_app -b -m get_url -a \
'url=https://raw.githubusercontent.com/kcommit/ansible-learning-lab/main/misc/inventory/nodes.ini dest=/etc/ansible/nodes.ini owner=root group=root mode=0644'
```

> Kisi naye command ko tamam nodes par chalane se pehle `--limit node1` ke sath test karna achhi practice hai.

---

## 5. Doosri websites ke examples

GitHub ke ilawa kisi aur website ke liye us website ya file server ka direct file URL use karein.

```bash
ansible three_tier_app -m get_url -a \
'url=https://example.com/files/application.conf dest=/tmp/application.conf mode=0644'
```

Direct-download file in jagahon par bhi ho sakti hai:

- Internal web server
- Artifact repository
- Software vendor ka download server
- Object-storage URL
- GitLab ka raw file URL

Domain name mein `raw` ya `usercontent` hona zaroori nahi. Zaroori yeh hai ke URL intended file return kare.

---

## 6. URL ko use karne se pehle test kaise karein

### HTTP headers inspect karein

```bash
curl -I -L "https://example.com/files/config.conf"
```

- `-I` response headers mangta hai.
- `-L` redirects follow karta hai.

Successful response aam tor par is tarah hota hai:

```text
HTTP/2 200
```

### Content preview karein

```bash
curl -L "https://example.com/files/config.conf" | head
```

Agar output HTML se shuru ho, to mumkin hai ke aap actual file ki jagah webpage URL use kar rahe hain:

```html
<!DOCTYPE html>
<html>
```

### Local machine par test download karein

```bash
curl -L -o /tmp/test-download.conf \
"https://example.com/files/config.conf"
```

Phir file inspect karein:

```bash
file /tmp/test-download.conf
head /tmp/test-download.conf
```

---

## 7. `get_url` ke aham arguments

| Argument | Kaam | Example |
|---|---|---|
| `url` | Source URL specify karta hai | `url=https://example.com/file.txt` |
| `dest` | Destination path specify karta hai | `dest=/tmp/file.txt` |
| `mode` | File permissions set karta hai | `mode=0644` |
| `owner` | File owner set karta hai | `owner=root` |
| `group` | File group set karta hai | `group=root` |
| `force` | Zaroorat par file dobara download karta hai | `force=yes` |
| `checksum` | File ki integrity verify karta hai | `checksum=sha256:<HASH>` |
| `validate_certs` | HTTPS certificate verify karta hai | `validate_certs=yes` |
| `timeout` | Download timeout set karta hai | `timeout=30` |

Module ki complete documentation dekhein:

```bash
ansible-doc get_url
```

Short argument reference dekhein:

```bash
ansible-doc -s get_url
```

---

## 8. Downloaded file verify karne ke commands

Check karein ke file maujood hai:

```bash
ansible three_tier_app -m stat -a \
'path=/tmp/nodes.ini'
```

File ka content dekhein:

```bash
ansible three_tier_app -m command -a \
'head /tmp/nodes.ini'
```

File ki type check karein:

```bash
ansible three_tier_app -m command -a \
'file /tmp/nodes.ini'
```

Ownership aur permissions check karein:

```bash
ansible three_tier_app -m command -a \
'ls -l /tmp/nodes.ini'
```

---

## 9. Aam ghaltiyan

### Ghalti 1: GitHub ka `blob` URL use karna

Ghalat:

```text
https://github.com/OWNER/REPOSITORY/blob/main/file.txt
```

Sahi:

```text
https://raw.githubusercontent.com/OWNER/REPOSITORY/main/file.txt
```

### Ghalti 2: Yeh samajhna ke har direct URL mein `raw` hota hai

Sirf kuch platforms apne direct-file URL mein `raw` use karte hain. Doosri websites ka URL format mukhtalif ho sakta hai.

### Ghalti 3: Intended file ki jagah HTML download kar lena

Agar URL ke bare mein yaqeen na ho to `curl -L "URL" | head` se content inspect karein.

### Ghalti 4: Privilege escalation use na karna

`/etc` jaisi directories mein file likhne ke liye aam tor par root permissions chahiye hoti hain. Zaroorat par `-b` add karein.

### Ghalti 5: Bagair zaroorat certificate validation disable karna

HTTPS certificate verification enabled rakhein:

```text
validate_certs=yes
```

Isay sirf controlled lab mein temporary tor par disable karein, jab aap ko certificate failure ki wajah maloom ho.

---

## 10. Khulasa

- `get_url` managed nodes par files download karta hai.
- `raw.githubusercontent.com` GitHub ke raw file content ke liye hota hai.
- Yeh doosri websites ke URLs ke liye required nahi hai.
- Suitable URL actual file return kare, HTML webpage nahi.
- Uncertain URL ko `curl -I -L` aur `curl -L "URL" | head` se test karein.
- Pehle `--limit node1` ke sath test karein, phir tamam managed nodes par chalayein.
- Root-owned destination ke liye `-b` use karein.
- Result ko `stat`, `head`, `file` ya `ls -l` se verify karein.

### Ek-line definition

> Ansible ka `get_url` module direct URL se file download karke ek ya zyada managed nodes par rakhta hai.
