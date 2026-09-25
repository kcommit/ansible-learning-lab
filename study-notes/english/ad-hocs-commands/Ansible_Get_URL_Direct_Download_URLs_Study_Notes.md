# Ansible `get_url`: Direct Download URLs Study Notes

## Table of Contents

1. [What is the `get_url` module?](#1-what-is-the-get_url-module)
2. [Is `raw.githubusercontent.com` required for every URL?](#2-is-rawgithubusercontentcom-required-for-every-url)
3. [GitHub webpage URL vs raw file URL](#3-github-webpage-url-vs-raw-file-url)
4. [Example for the personal Ansible lab](#4-example-for-the-personal-ansible-lab)
5. [Examples from other websites](#5-examples-from-other-websites)
6. [How to test a URL before using it](#6-how-to-test-a-url-before-using-it)
7. [Important `get_url` arguments](#7-important-get_url-arguments)
8. [Verification commands](#8-verification-commands)
9. [Common mistakes](#9-common-mistakes)
10. [Quick summary](#10-quick-summary)

---

## 1. What is the `get_url` module?

The Ansible `get_url` module downloads a file from an HTTP, HTTPS, or FTP URL to a managed node.

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

In this command:

- `three_tier_app` is the inventory host group.
- `-m get_url` selects the `get_url` module.
- `url=` specifies the source URL.
- `dest=` specifies the destination on each managed node.
- `mode=0644` sets the downloaded file's permissions.

---

## 2. Is `raw.githubusercontent.com` required for every URL?

No. `raw.githubusercontent.com` is used for downloading raw file contents from GitHub. It is not an Ansible requirement and is not needed for every website.

The important requirement is that the URL must return the actual file contents, rather than an HTML webpage displaying the file.

| URL type | Suitable for `get_url`? | Explanation |
|---|---:|---|
| GitHub `blob` webpage | Usually no | Returns the GitHub webpage and interface |
| GitHub raw URL | Yes | Returns the actual file contents |
| Direct file URL from another website | Yes | Returns the requested file directly |
| Login or download webpage | Usually no | May return HTML, authentication, or redirects |

> `usercontent` is simply part of GitHub's raw-content domain name. It is not an Ansible keyword or required option.

---

## 3. GitHub webpage URL vs raw file URL

### GitHub webpage URL

This URL displays the file inside GitHub's website:

```text
https://github.com/kcommit/ansible-learning-lab/blob/main/misc/inventory/nodes.ini
```

Notice the `/blob/` component. This is normally a webpage URL, not a direct file URL.

### GitHub raw file URL

The corresponding direct-download URL is:

```text
https://raw.githubusercontent.com/kcommit/ansible-learning-lab/main/misc/inventory/nodes.ini
```

The raw URL returns the actual content of `nodes.ini` without the GitHub interface.

### Easy GitHub method

1. Open the file on GitHub.
2. Select **Raw**.
3. Copy the URL from the browser.
4. Use that URL with `get_url`.

---

## 4. Example for the personal Ansible lab

Download `nodes.ini` to `/tmp` on `node1` first:

```bash
ansible three_tier_app --limit node1 -m get_url -a \
'url=https://raw.githubusercontent.com/kcommit/ansible-learning-lab/main/misc/inventory/nodes.ini dest=/tmp/nodes.ini mode=0644'
```

After confirming that it works on `node1`, run it on all three managed nodes:

```bash
ansible three_tier_app -m get_url -a \
'url=https://raw.githubusercontent.com/kcommit/ansible-learning-lab/main/misc/inventory/nodes.ini dest=/tmp/nodes.ini mode=0644'
```

If the destination requires root permission, add `-b`:

```bash
ansible three_tier_app -b -m get_url -a \
'url=https://raw.githubusercontent.com/kcommit/ansible-learning-lab/main/misc/inventory/nodes.ini dest=/etc/ansible/nodes.ini owner=root group=root mode=0644'
```

> Start with `--limit node1` before applying a new command to every managed node.

---

## 5. Examples from other websites

For websites other than GitHub, use the direct URL provided by that website or file server.

```bash
ansible three_tier_app -m get_url -a \
'url=https://example.com/files/application.conf dest=/tmp/application.conf mode=0644'
```

Other possible direct-download locations include:

- An internal web server
- An artifact repository
- A software vendor's download server
- An object-storage URL
- A GitLab raw file URL

The domain does not need to contain `raw` or `usercontent`. The response only needs to provide the intended file.

---

## 6. How to test a URL before using it

### Inspect HTTP headers

```bash
curl -I -L "https://example.com/files/config.conf"
```

- `-I` requests response headers.
- `-L` follows redirects.

Look for a successful response such as:

```text
HTTP/2 200
```

### Preview the content

```bash
curl -L "https://example.com/files/config.conf" | head
```

If the output begins with HTML such as the following, the URL is probably a webpage rather than the requested file:

```html
<!DOCTYPE html>
<html>
```

### Download locally for testing

```bash
curl -L -o /tmp/test-download.conf \
"https://example.com/files/config.conf"
```

Then inspect it:

```bash
file /tmp/test-download.conf
head /tmp/test-download.conf
```

---

## 7. Important `get_url` arguments

| Argument | Purpose | Example |
|---|---|---|
| `url` | Source URL | `url=https://example.com/file.txt` |
| `dest` | Destination path | `dest=/tmp/file.txt` |
| `mode` | File permissions | `mode=0644` |
| `owner` | File owner | `owner=root` |
| `group` | File group | `group=root` |
| `force` | Download again when required | `force=yes` |
| `checksum` | Verify file integrity | `checksum=sha256:<HASH>` |
| `validate_certs` | Verify the HTTPS certificate | `validate_certs=yes` |
| `timeout` | Set download timeout | `timeout=30` |

View the module documentation:

```bash
ansible-doc get_url
```

View a shorter argument reference:

```bash
ansible-doc -s get_url
```

---

## 8. Verification commands

Check that the file exists:

```bash
ansible three_tier_app -m stat -a \
'path=/tmp/nodes.ini'
```

Display the file contents:

```bash
ansible three_tier_app -m command -a \
'head /tmp/nodes.ini'
```

Check the file type:

```bash
ansible three_tier_app -m command -a \
'file /tmp/nodes.ini'
```

Check its ownership and permissions:

```bash
ansible three_tier_app -m command -a \
'ls -l /tmp/nodes.ini'
```

---

## 9. Common mistakes

### Mistake 1: Using a GitHub `blob` URL

Incorrect:

```text
https://github.com/OWNER/REPOSITORY/blob/main/file.txt
```

Correct:

```text
https://raw.githubusercontent.com/OWNER/REPOSITORY/main/file.txt
```

### Mistake 2: Assuming every direct URL contains `raw`

Only some platforms use `raw` in their direct-file URL. Other websites may use entirely different URL formats.

### Mistake 3: Downloading HTML instead of the intended file

Always inspect the URL with `curl -L ... | head` when uncertain.

### Mistake 4: Missing privilege escalation

Writing to directories such as `/etc` normally requires root privileges. Add `-b` when necessary.

### Mistake 5: Disabling certificate validation unnecessarily

Keep HTTPS certificate verification enabled:

```text
validate_certs=yes
```

Only disable it temporarily in a controlled lab when you understand why certificate verification is failing.

---

## 10. Quick summary

- `get_url` downloads files to managed nodes.
- `raw.githubusercontent.com` is specific to GitHub raw file content.
- It is not required for URLs from other websites.
- A suitable URL must return the actual file, not an HTML webpage.
- Test uncertain URLs with `curl -I -L` and `curl -L URL | head`.
- Use `--limit node1` before applying an unfamiliar download command to every node.
- Use `-b` when the destination directory requires root permission.
- Verify the result with `stat`, `head`, `file`, or `ls -l`.

### One-line definition

> The Ansible `get_url` module downloads a file from a direct URL to one or more managed nodes.
