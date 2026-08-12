# 🚀 Laravel Application Deployment on Ubuntu 26.04

## 🧩 Nginx + PHP 7.4-FPM + Laravel + Custom Domain

> 📚 A step-by-step DevOps deployment guide for running a Laravel application with **Nginx** and **PHP 7.4-FPM** on **Ubuntu 26.04**.

![Ubuntu](https://img.shields.io/badge/Ubuntu-26.04-E95420?logo=ubuntu&logoColor=white)
![Nginx](https://img.shields.io/badge/Nginx-1.28.x-009639?logo=nginx&logoColor=white)
![PHP](https://img.shields.io/badge/PHP-7.4-777BB4?logo=php&logoColor=white)
![Laravel](https://img.shields.io/badge/Laravel-PHP%20Framework-FF2D20?logo=laravel&logoColor=white)

---

This documentation records the deployment process in the same order used during the setup.

## 🖥️ Environment

- OS: Ubuntu 26.04
- Web Server: Nginx
- PHP: 7.4
- PHP-FPM: 7.4
- Framework: Laravel
- Project: `forum-pipe`
- Project path: `/var/www/forum-pipe`
- Domain: `forum.devops.com`
- VM IP: `192.168.1.191`

---

# 1️⃣ Update Ubuntu

First update the package list:

```bash
sudo apt update
```

Upgrade installed packages:

```bash
sudo apt upgrade -y
```

---

# 2️⃣ Install Nginx

Install Nginx:

```bash
sudo apt install nginx -y
```

Check Nginx status:

```bash
sudo systemctl status nginx --no-pager
```

If Nginx is not running:

```bash
sudo systemctl start nginx
```

Enable Nginx to start automatically after reboot:

```bash
sudo systemctl enable nginx
```

Check the installed Nginx version:

```bash
nginx -v
```

In this setup, Nginx was:

```text
nginx/1.28.3
```

---

# 3️⃣ Install PHP Repository Dependencies

The following packages were installed before configuring PHP:

```bash
sudo apt install software-properties-common ca-certificates lsb-release apt-transport-https
```

These packages provide repository-management and HTTPS support.

---

# 4️⃣ Add PHP Repository

The Ondřej PHP repository was attempted with:

```bash
sudo LC_ALL=C.UTF-8 add-apt-repository ppa:ondrej/php
```

During the setup on Ubuntu 26.04 (`resolute`), the PPA returned:

```text
404 Not Found
```

Specifically, the repository did not provide a `resolute` Release file.

Therefore, this PPA should not be treated as a valid Ubuntu 26.04 source.

> **Important:** PHP 7.4-FPM was already available/installed in the working environment, so the deployment continued with the existing PHP 7.4 installation.

---

# 5️⃣ PHP 7.4 Packages Used

The PHP 7.4 packages used for the Laravel application were:

```bash
sudo apt install php7.4 php7.4-common php7.4-opcache php7.4-cli \
php7.4-gd php7.4-curl php7.4-mysql php7.4-xmlrpc php7.4-imap \
php7.4-mbstring php7.4-xml php7.4-fpm php7.4-zip php7.4-bcmath
```

> On the initial repository attempt, Ubuntu reported that these PHP 7.4 packages had no installation candidate because the required PHP 7.4 repository was not available for `resolute`. The final working environment nevertheless had PHP 7.4-FPM installed and running.

---

# 6️⃣ Verify PHP

Check PHP version:

```bash
php -v
```

Check PHP-FPM:

```bash
sudo systemctl status php7.4-fpm --no-pager
```

Expected:

```text
Active: active (running)
```

Enable PHP-FPM:

```bash
sudo systemctl enable php7.4-fpm
```

---

# 7️⃣ Check PHP-FPM Socket

Check the PHP sockets:

```bash
ls -l /run/php/
```

The working PHP-FPM socket was:

```text
/run/php/php7.4-fpm.sock
```

It can also be referenced as:

```text
/var/run/php/php7.4-fpm.sock
```

The socket showed:

```text
www-data www-data
```

---

# 8️⃣ Laravel Project Location

The Laravel project was located under:

```text
/var/www/forum-pipe
```

Check:

```bash
ls -la /var/www/
```

Expected project directory:

```text
forum-pipe/
```

Check the project:

```bash
ls -la /var/www/forum-pipe/
```

The project contained:

```text
app/
bootstrap/
config/
database/
public/
resources/
routes/
storage/
vendor/
artisan
composer.json
.env
```

---

# 9️⃣ Check Laravel Public Directory

Laravel's web root is the `public` directory.

Check:

```bash
ls -la /var/www/forum-pipe/public/
```

Important file:

```text
index.php
```

Therefore the Nginx document root must be:

```text
/var/www/forum-pipe/public
```

---

# 🔟 Create Nginx Virtual Host

Create the Nginx site configuration:

```bash
sudo nano /etc/nginx/sites-enabled/forum.devops.com.conf
```

Use:

```nginx
server {
    listen 80;
    server_name forum.devops.com;

    root /var/www/forum-pipe/public;

    add_header X-Frame-Options "SAMEORIGIN";
    add_header X-Content-Type-Options "nosniff";

    index index.php;
    charset utf-8;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location = /favicon.ico {
        access_log off;
        log_not_found off;
    }

    location = /robots.txt {
        access_log off;
        log_not_found off;
    }

    error_page 404 /index.php;

    location ~ \.php$ {
        fastcgi_pass unix:/var/run/php/php7.4-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
        include fastcgi_params;
    }

    location ~ /\.(?!well-known).* {
        deny all;
    }
}
```

### ✍️ Nano Save / Exit

After editing:

```text
Ctrl + X
Y
Enter
```

---

# 1️⃣1️⃣ ⚠️ Important Nginx Root Path Issue

Initially the configuration contained an incorrect path:

```nginx
root /var/www/html/forum-pipe/public;
```

But the actual Laravel project was:

```text
/var/www/forum-pipe/
```

and the actual public directory was:

```text
/var/www/forum-pipe/public/
```

Therefore the correct configuration is:

```nginx
root /var/www/forum-pipe/public;
```

This was the reason for the initial:

```text
HTTP/1.1 404 Not Found
```

response.

---

# 1️⃣2️⃣ Test Nginx Configuration

After creating/editing the configuration:

```bash
sudo nginx -t
```

Expected:

```text
nginx: configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

Reload Nginx:

```bash
sudo systemctl reload nginx
```

---

# 1️⃣3️⃣ Verify the Active Virtual Host

To make sure Nginx is actually using the correct configuration:

```bash
sudo nginx -T | grep -A 35 "server_name forum.devops.com"
```

Verify that the output contains:

```nginx
server_name forum.devops.com;
root /var/www/forum-pipe/public;
```

Also verify:

```nginx
fastcgi_pass unix:/var/run/php/php7.4-fpm.sock;
```

---

# 1️⃣4️⃣ Test the Application from Ubuntu

First test the domain:

```bash
curl -I http://forum.devops.com
```

Initially, before fixing the Nginx root, the result was:

```text
HTTP/1.1 404 Not Found
```

After correcting the root to:

```nginx
root /var/www/forum-pipe/public;
```

and reloading Nginx:

```bash
sudo nginx -t
sudo systemctl reload nginx
```

the result became:

```text
HTTP/1.1 200 OK
```

The final response included:

```text
Server: nginx/1.28.3 (Ubuntu)
Content-Type: text/html; charset=UTF-8
Cache-Control: no-cache, private
```

This confirmed that Laravel was successfully being served through Nginx and PHP-FPM.

---

# 1️⃣5️⃣ Test PHP-FPM Through Nginx

Another useful test:

```bash
curl -I -H "Host: forum.devops.com" http://127.0.0.1/index.php
```

This verifies that the Nginx virtual host is receiving the requested Host header and forwarding PHP requests to PHP-FPM.

---

# 1️⃣6️⃣ 🌐 Configure the Domain

The domain used for the application:

```text
forum.devops.com
```

The VM/server IP:

```text
192.168.1.191
```

Inside Ubuntu, the domain resolved successfully:

```bash
ping forum.devops.com
```

Result:

```text
PING forum.devops.com (192.168.1.191)
```

---

# 1️⃣7️⃣ Test the Domain from the Ubuntu VM

```bash
curl -I http://forum.devops.com
```

Successful result:

```text
HTTP/1.1 200 OK
```

At this point the application worked in the Ubuntu VM browser.

---

# 1️⃣8️⃣ 🪟 Make the Domain Work from the Windows Host

The VM browser could open:

```text
http://forum.devops.com
```

but the Windows host browser could not resolve:

```text
forum.devops.com
```

Windows initially returned:

```text
Ping request could not find host forum.devops.com.
```

The solution is to add the domain to the Windows `hosts` file.

Open **Command Prompt as Administrator**.

Run:

```cmd
notepad C:\Windows\System32\drivers\etc\hosts
```

Add:

```text
192.168.1.191    forum.devops.com
```

Save the file.

Flush DNS:

```cmd
ipconfig /flushdns
```

Then test:

```cmd
ping forum.devops.com
```

Expected:

```text
Pinging forum.devops.com [192.168.1.191]
```

Then open:

```text
http://forum.devops.com
```

in the Windows browser.

---

# 1️⃣9️⃣ Check the Windows Hosts File

To verify the hosts file:

```cmd
type C:\Windows\System32\drivers\etc\hosts
```

Or search for the domain:

```cmd
findstr /i "forum.devops.com" C:\Windows\System32\drivers\etc\hosts
```

Expected:

```text
192.168.1.191 forum.devops.com
```

If you see:

```text
Access is denied.
```

when modifying the hosts file, make sure Command Prompt or Notepad was started with:

```text
Run as administrator
```

---

# 2️⃣0️⃣ 🛠️ Useful Troubleshooting Commands

## Nginx

```bash
sudo systemctl status nginx --no-pager
```

```bash
sudo nginx -t
```

```bash
sudo nginx -T
```

```bash
sudo systemctl reload nginx
```

```bash
sudo systemctl restart nginx
```

## PHP

```bash
php -v
```

```bash
sudo systemctl status php7.4-fpm --no-pager
```

```bash
ls -l /run/php/
```

## Laravel

```bash
ls -la /var/www/forum-pipe/
```

```bash
ls -la /var/www/forum-pipe/public/
```

```bash
sudo tail -n 50 /var/www/forum-pipe/storage/logs/laravel.log
```

## Domain

```bash
ping forum.devops.com
```

```bash
curl -I http://forum.devops.com
```

---

# 2️⃣1️⃣ 🏗️ Final Architecture

```text
Windows Host Browser
        |
        | forum.devops.com
        |
        | Windows hosts file
        ↓
192.168.1.191
        |
        ↓
Ubuntu 26.04 VM
        |
        ↓
Nginx :80
        |
        ↓
/var/www/forum-pipe/public
        |
        ↓
index.php
        |
        ↓
PHP 7.4-FPM
        |
        ↓
Laravel Application
        |
        ↓
HTTP 200 OK
```

---

# 2️⃣2️⃣ 📋 Final Configuration Summary

| Component | Configuration |
|---|---|
| OS | Ubuntu 26.04 |
| Web Server | Nginx |
| Nginx Version | 1.28.3 |
| PHP | 7.4 |
| PHP-FPM | 7.4-FPM |
| PHP-FPM Socket | `/run/php/php7.4-fpm.sock` |
| Laravel Project | `/var/www/forum-pipe` |
| Laravel Public | `/var/www/forum-pipe/public` |
| Laravel Entry | `/var/www/forum-pipe/public/index.php` |
| Domain | `forum.devops.com` |
| VM IP | `192.168.1.191` |
| Nginx Config | `/etc/nginx/sites-enabled/forum.devops.com.conf` |
| Final HTTP Status | `200 OK` |

---

# 2️⃣3️⃣ 🎉 Result

The deployment successfully achieved:

```text
Ubuntu 26.04
      ↓
Nginx
      ↓
PHP 7.4-FPM
      ↓
Laravel
      ↓
forum.devops.com
      ↓
HTTP 200 OK
```

The main troubleshooting issue was the incorrect Nginx root path. Once it was changed from:

```nginx
/var/www/html/forum-pipe/public
```

to:

```nginx
/var/www/forum-pipe/public
```

the Laravel application started returning:

```text
HTTP/1.1 200 OK
```


---

## ⭐ Learning Outcome

By completing this deployment, you practice:

- 🐧 Ubuntu server administration
- 🌐 Nginx web server configuration
- 🐘 PHP 7.4-FPM configuration
- 🚀 Laravel deployment
- 🔗 Custom domain / hosts-file mapping
- 🛠️ Nginx and PHP troubleshooting
- 🔍 HTTP testing with `curl`
- 📦 Basic production-style application structure

---

### 👨‍💻 Author

**MD. JAKIUL RASHID KHAN**

⭐ If this documentation helps you, consider giving the repository a **Star** on GitHub!
