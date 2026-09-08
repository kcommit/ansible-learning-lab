# Deploy a Static Website with NGINX on Rocky Linux

## Objective

Download a website template, inspect it safely, place the website files in `/var/www/html`, configure NGINX, and test the deployment.

For the first practice deployment, use port `8080`. This avoids conflict if Apache, another NGINX server block, or a container is already using port `80`.

## Deployment Flow

1. Download the ZIP archive into a temporary directory.
2. Verify and inspect the archive.
3. Extract the website into a staging directory.
4. Back up the existing document root.
5. Copy only the website files to `/var/www/html`.
6. Apply safe ownership, permissions, and SELinux labels.
7. Configure NGINX on port `8080`.
8. Test the configuration, reload NGINX, and verify the response.

## 1. Check for Existing Web Services

Before deployment, check which processes are using ports `80` and `8080`:

```bash
sudo ss -ltnp 'sport = :80 or sport = :8080'
```

Also check the services:

```bash
sudo systemctl status nginx
sudo systemctl status httpd
```

Merely installing Apache does not create a port conflict. A conflict occurs when Apache is running and listening on the same IP address and port that NGINX wants to use.

## 2. Create a Temporary Working Directory

```bash
work_dir=$(mktemp -d)
cd "$work_dir"
```

Check the generated location:

```bash
pwd
```

`mktemp -d` creates a unique temporary directory, which prevents accidental filename collisions with earlier downloads.

## 3. Download the Website Archive

```bash
wget -O lawfirm.zip \
"https://freewebsitetemplates.com/preview/lawfirm/"
```

Command explanation:

| Part | Meaning |
| --- | --- |
| `wget` | Downloads content using HTTP or HTTPS |
| `-O` | Writes the response using the specified output filename |
| `lawfirm.zip` | Exact filename to create |
| Quoted URL | Protects special URL characters from shell interpretation |

Uppercase `-O` selects the output file. Lowercase `-o` writes Wget messages to a log file, so the two options are not interchangeable.

### Important overwrite warning

If `lawfirm.zip` already exists, `wget -O lawfirm.zip` can overwrite or truncate it. Using a unique temporary directory helps avoid this problem.

## 4. Verify the Download

Check its type:

```bash
file lawfirm.zip
```

Expected result should identify ZIP archive data.

View the file size:

```bash
ls -lh lawfirm.zip
```

Test archive integrity without extracting it:

```bash
unzip -t lawfirm.zip
```

The final output should report that no errors were detected.

List its contents without extraction:

```bash
unzip -l lawfirm.zip | less
```

Before publishing a third-party template publicly, check its license and usage terms.

## 5. Extract into a Staging Directory

```bash
mkdir extracted
unzip lawfirm.zip -d extracted
```

Find the main page:

```bash
find extracted -type f -name index.html -print
```

Possible results include:

```text
extracted/index.html
```

or:

```text
extracted/lawfirm/index.html
```

The directory containing `index.html` is normally the directory whose contents should be copied to the NGINX document root.

## 6. Back Up the Existing Document Root

Check the existing directory:

```bash
sudo ls -lah /var/www/html
```

If it already exists and contains files, create a timestamped backup:

```bash
sudo cp -a /var/www/html \
"/var/www/html.backup-$(date '+%Y-%m-%d-%H-%M-%S')"
```

Create the document root if necessary:

```bash
sudo mkdir -p /var/www/html
```

## 7. Copy the Website Files

If `index.html` is located at `extracted/lawfirm/index.html`, run:

```bash
sudo cp -a extracted/lawfirm/. /var/www/html/
```

If it is located at `extracted/index.html`, run:

```bash
sudo cp -a extracted/. /var/www/html/
```

The `/.` means copy the directory's contents, including hidden files, rather than nesting the source directory itself inside `/var/www/html`.

Verify the result:

```bash
sudo find /var/www/html -maxdepth 2 -type f | head -30
```

You should normally see items such as:

```text
/var/www/html/index.html
/var/www/html/css/...
/var/www/html/images/...
```

## 8. Apply Safe Ownership and Permissions

For a static website, NGINX needs read access but does not normally need ownership or write access:

```bash
sudo chown -R root:root /var/www/html
sudo find /var/www/html -type d -exec chmod 755 {} +
sudo find /var/www/html -type f -exec chmod 644 {} +
```

Meaning:

| Setting | Purpose |
| --- | --- |
| Directories `755` | Owner can modify; everyone can enter and read |
| Files `644` | Owner can modify; everyone can read |
| `root:root` | Web content cannot normally be modified by the NGINX worker |

Do not use:

```bash
chmod -R 777 /var/www/html
```

Mode `777` gives every local user write permission and is unnecessary for serving static files.

## 9. Restore SELinux Labels on Rocky Linux

```bash
sudo restorecon -Rv /var/www/html
```

Inspect the labels:

```bash
ls -ldZ /var/www/html
ls -lZ /var/www/html/index.html
```

Web content should normally have an appropriate HTTP content context, such as `httpd_sys_content_t`.

Do not disable SELinux as the first response to an access problem. Inspect permissions, labels, and logs first.

## 10. Create the NGINX Server Configuration

Create the configuration file:

```bash
sudo vi /etc/nginx/conf.d/lawfirm.conf
```

Add:

```nginx
server {
    listen 8080;
    listen [::]:8080;

    server_name _;

    root /var/www/html;
    index index.html;

    access_log /var/log/nginx/lawfirm_access.log;
    error_log  /var/log/nginx/lawfirm_error.log;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

### Directive explanation

| Directive | Purpose |
| --- | --- |
| `listen 8080` | Accepts IPv4 HTTP requests on port `8080` |
| `listen [::]:8080` | Accepts IPv6 HTTP requests on port `8080` |
| `server_name _` | Provides a catch-all name for this practice port |
| `root /var/www/html` | Sets the document root |
| `index index.html` | Uses `index.html` for directory requests |
| `access_log` | Records incoming HTTP requests |
| `error_log` | Records NGINX errors for this site |
| `try_files` | Looks for the requested file or directory; otherwise returns `404` |

## 11. Validate and Reload NGINX

Test the entire NGINX configuration:

```bash
sudo nginx -t
```

Expected result:

```text
syntax is ok
test is successful
```

`nginx -t` checks configuration syntax and referenced files. It does not prove that the website content is correct or that a requested port is reachable from another computer.

If the test succeeds, reload NGINX gracefully:

```bash
sudo systemctl reload nginx
```

Check the service:

```bash
sudo systemctl status nginx
```

Confirm that port `8080` is listening:

```bash
sudo ss -ltnp 'sport = :8080'
```

## 12. Test the Website Locally

Request only the response headers:

```bash
curl -I http://127.0.0.1:8080/
```

Expected status:

```text
HTTP/1.1 200 OK
```

Download and display part of the page:

```bash
curl -s http://127.0.0.1:8080/ | head
```

## 13. Allow Port 8080 Through firewalld

Check the active firewall configuration:

```bash
sudo firewall-cmd --list-all
```

Allow port `8080` temporarily for the current runtime:

```bash
sudo firewall-cmd --add-port=8080/tcp
```

For a permanent rule:

```bash
sudo firewall-cmd --permanent --add-port=8080/tcp
sudo firewall-cmd --reload
```

Verify:

```bash
sudo firewall-cmd --query-port=8080/tcp
```

Expected output:

```text
yes
```

## 14. Test from Windows

Open a browser and enter:

```text
http://SERVER_IP:8080/
```

Example:

```text
http://192.168.1.167:8080/
```

You can also test from Windows PowerShell:

```powershell
Test-NetConnection 192.168.1.167 -Port 8080
curl.exe -I http://192.168.1.167:8080/
```

## 15. Troubleshooting Workflow

### NGINX fails to start or reload

```bash
sudo nginx -t
sudo systemctl status nginx --no-pager
sudo journalctl -u nginx -n 50 --no-pager
```

### Port is already in use

```bash
sudo ss -ltnp 'sport = :8080'
```

Choose another port or safely stop/reconfigure the process currently using it.

### HTTP 403 Forbidden

Check directory traversal permissions:

```bash
namei -l /var/www/html/index.html
```

Check normal and SELinux permissions:

```bash
ls -ldZ /var /var/www /var/www/html
ls -lZ /var/www/html/index.html
```

Read the NGINX error log:

```bash
sudo tail -n 50 /var/log/nginx/lawfirm_error.log
```

Common causes include:

- Missing `index.html`
- Incorrect `root` directive
- Missing execute permission on a parent directory
- Incorrect SELinux context
- An NGINX `deny` rule

### HTTP 404 Not Found

Confirm that the requested file exists beneath the configured root:

```bash
sudo ls -l /var/www/html/index.html
sudo nginx -T | grep -nE 'listen|server_name|root|index|try_files'
```

### It works locally but not from Windows

Check:

```bash
ip -br address
sudo ss -ltnp 'sport = :8080'
sudo firewall-cmd --query-port=8080/tcp
```

Also confirm that the VM network adapter and any external firewall/security rules permit the connection.

## 16. Moving the Site to Port 80 Later

First check whether anything already uses port `80`:

```bash
sudo ss -ltnp 'sport = :80'
```

If the port is safely available, change the two `listen` directives:

```nginx
listen 80;
listen [::]:80;
```

Then validate and reload:

```bash
sudo nginx -t
sudo systemctl reload nginx
```

Allow standard HTTP through firewalld:

```bash
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --reload
```

Test:

```bash
curl -I http://127.0.0.1/
```

## 17. Rollback

Before rollback, locate the exact timestamped backup:

```bash
sudo find /var/www -maxdepth 1 -type d \
    -name 'html.backup-*' -print
```

Stop and verify the exact backup directory before replacing any content. Restore only the intended backup, then run:

```bash
sudo restorecon -Rv /var/www/html
sudo nginx -t
sudo systemctl reload nginx
```

To remove the practice configuration after confirming the exact target:

```bash
sudo rm /etc/nginx/conf.d/lawfirm.conf
sudo nginx -t
sudo systemctl reload nginx
```

Remove the firewall rule if it is no longer required:

```bash
sudo firewall-cmd --permanent --remove-port=8080/tcp
sudo firewall-cmd --reload
```

## Interview-Style Summary

> I would download the website archive into a temporary staging directory instead of downloading directly into the document root. I would verify the file type and archive integrity, inspect its structure, and back up the existing site before copying the correct files into `/var/www/html`. I would apply read-only static-content permissions, restore SELinux labels, and configure a dedicated NGINX server block. Finally, I would run `nginx -t`, reload NGINX, confirm the listening port, test locally with `curl`, verify the firewall, and then test from the client.

## Key Points

- Download and inspect content outside the live web root.
- Use uppercase `wget -O` to select the output filename.
- Locate `index.html` before copying files.
- Back up the existing document root before deployment.
- Use `755` for directories and `644` for static files.
- Do not use `chmod 777` as a shortcut.
- Restore SELinux labels on Rocky Linux.
- Start on port `8080` to avoid disrupting an existing port `80` service.
- Run `nginx -t` before every reload.
- Verify the deployment with both `curl` and a remote client.
