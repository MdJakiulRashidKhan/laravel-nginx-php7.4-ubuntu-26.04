# 🚀 Laravel Application Deployment on Ubuntu 26.04

## Nginx + PHP 7.4-FPM + MySQL + Laravel + Composer + Custom Domain

> A complete step-by-step guide for deploying the `forum-pipe` Laravel application on Ubuntu 26.04 using Nginx, MySQL, PHP 7.4-FPM, Composer, and a custom local domain.

---

## 🖥️ Environment

| Component | Configuration |
|---|---|
| OS | Ubuntu 26.04 |
| Web Server | Nginx |
| Database | MySQL |
| PHP | 7.4 |
| PHP-FPM | 7.4-FPM |
| Dependency Manager | Composer |
| Framework | Laravel |
| Project | `forum-pipe` |
| Project Path | `/var/www/forum-pipe` |
| Domain | `forum.devops.com` |
| Server / VM IP | `192.168.8.134` |
| Database | `forumdb` |
| Database User | `forum` |
| Nginx Config | `/etc/nginx/sites-available/forum.devops.com.conf` |

---

# 1️⃣ Update Ubuntu

Update the package list:

```bash
sudo apt update
```

---

# 2️⃣ Install Nginx

Install Nginx:

```bash
sudo apt install nginx -y
```

Nginx is the web server. It receives HTTP requests for `forum.devops.com` and passes PHP requests to PHP-FPM.

Check Nginx:

```bash
systemctl status nginx --no-pager
```

Check the version:

```bash
nginx -v
```

---

# 3️⃣ Install MySQL

Install MySQL Server:

```bash
apt install mysql-server -y
```

Run the MySQL security configuration:

```bash
mysql_secure_installation
```

During the interactive setup, the following choices were used:

```text
Remove anonymous users?                 Y
Disallow root login remotely?           Y
Remove test database and access to it?  Y
Reload privilege tables now?            Y
```

## Verify MySQL

Open MySQL:

```bash
mysql
```

Check databases:

```sql
show databases;
```

---

# 4️⃣ Create the Laravel Database and User

Inside MySQL:

```sql
create database forumdb;

create user 'forum'@'localhost' identified by 'Test@12345';

grant all privileges on forumdb.* to 'forum'@'localhost';

flush privileges;
```

The Laravel application will use:

```text
Database: forumdb
Username: forum
Password: Test@12345
Host: 127.0.0.1
Port: 3306
```

> ⚠️ Security: `Test@12345` is the password used in this setup. For a real production server, use a strong unique password and never commit real credentials to GitHub.

---

# 5️⃣ Install Repository Dependencies for PHP 7.4

Install the required packages:

```bash
apt install -y lsb-release ca-certificates curl apt-transport-https
```

Create the APT keyring directory:

```bash
mkdir -p /etc/apt/keyrings
```

Add the Sury PHP repository key:

```bash
curl -fsSL https://packages.sury.org/php/apt.gpg | sudo tee /etc/apt/keyrings/sury-php.gpg > /dev/null
```

Add the PHP repository:

```bash
echo "deb [signed-by=/etc/apt/keyrings/sury-php.gpg] https://packages.sury.org/php/ $(lsb_release -sc) main" | sudo tee /etc/apt/sources.list.d/php.list
```

Update APT:

```bash
apt update
```

---

# 6️⃣ Install PHP 7.4 + Required Extensions

Install PHP 7.4 and the extensions used by the Laravel application:

```bash
sudo apt install -y php7.4 php7.4-common php7.4-opcache php7.4-cli php7.4-gd php7.4-curl php7.4-mysql php7.4-xmlrpc php7.4-imap php7.4-mbstring php7.4-xml php7.4-fpm php7.4-zip php7.4-bcmath
```

Verify PHP:

```bash
php -v
```

---

# 7️⃣ Install Composer

Composer is the PHP dependency manager used by Laravel.

Go to `/tmp`:

```bash
cd /tmp/
```

Download and install Composer:

```bash
curl -sS https://getcomposer.org/installer | php
sudo mv composer.phar /usr/local/bin/composer
```

Verify Composer:

```bash
composer
```

The installer should report:

```text
All settings correct for using Composer
```

---

# 8️⃣ Verify PHP-FPM

Check PHP 7.4-FPM:

```bash
systemctl status php7.4-fpm.service
```

PHP-FPM is the service that executes PHP code for Nginx.

The Nginx configuration later uses this socket:

```text
/var/run/php/php7.4-fpm.sock
```

---

# 9️⃣ Clone the Laravel Project

Go to `/var/www`:

```bash
cd /var/www/
```

Clone the project:

```bash
git clone https://github.com/MdJakiulRashidKhan/forum-pipe.git
```

The project path is:

```text
/var/www/forum-pipe
```

Verify:

```bash
ls -la /var/www/
```

---

# 🔟 Configure Nginx Virtual Hosting

Go to the Nginx sites-available directory:

```bash
cd /etc/nginx/sites-available/
```

Create the configuration file:

```bash
nano forum.devops.com.conf
```

> ⚠️ Important: The correct filename is `forum.devops.com.conf`.

Use this configuration:

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

### Why is the Nginx root `/var/www/forum-pipe/public`?

Laravel's web-accessible entry point is inside:

```text
/var/www/forum-pipe/public/index.php
```

Therefore Nginx must point to:

```text
/var/www/forum-pipe/public
```

Do **not** use:

```text
/var/www/forum-pipe
```

or:

```text
/var/www/html/forum-pipe/public
```

for this setup.

---

# 1️⃣1️⃣ Configure Nginx Site Enablement

Create a symbolic link from `sites-available` to `sites-enabled`:

```bash
cd /etc/nginx/sites-enabled/
```

```bash
ln -s /etc/nginx/sites-available/forum.devops.com.conf /etc/nginx/sites-enabled/
```

Remove the default Nginx site:

```bash
rm -f default
```

Test the Nginx configuration:

```bash
nginx -t
```

Expected result:

```text
syntax is ok
test is successful
```

Restart Nginx:

```bash
systemctl restart nginx
```

---

# 1️⃣2️⃣ Set Laravel Project Permissions

Go to the Nginx configuration directory:

```bash
cd /etc/nginx/
```

Set file permissions:

```bash
find /var/www/forum-pipe -type f -exec chmod 644 {} \;
```

Set directory permissions:

```bash
find /var/www/forum-pipe -type d -exec chmod 755 {} \;
```

Set ownership:

```bash
chown -R www-data:www-data /var/www/forum-pipe
```

This makes the Laravel project owned by the Nginx/PHP-FPM user:

```text
www-data
```

---

# 1️⃣3️⃣ Configure Laravel `.env`

Go to the Laravel project:

```bash
cd /var/www/forum-pipe/
```

Create the environment file:

```bash
cp .env.example .env
```

Edit it:

```bash
nano .env
```

Use the database-related configuration:

```env
APP_NAME=forumapp
APP_ENV=production
APP_KEY=
APP_DEBUG=false
APP_URL=http://forum.devops.com

LOG_CHANNEL=stack
LOG_LEVEL=debug

DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=forumdb
DB_USERNAME=forum
DB_PASSWORD="Test@12345"

BROADCAST_DRIVER=log
CACHE_DRIVER=file
QUEUE_CONNECTION=sync
SESSION_DRIVER=file
SESSION_LIFETIME=120

MEMCACHED_HOST=127.0.0.1

REDIS_HOST=127.0.0.1
REDIS_PASSWORD=null
REDIS_PORT=6379

MAIL_MAILER=smtp
MAIL_HOST=mailhog
MAIL_PORT=1025
MAIL_USERNAME=null
MAIL_PASSWORD=null
MAIL_ENCRYPTION=null
MAIL_FROM_ADDRESS=null
MAIL_FROM_NAME="${APP_NAME}"

AWS_ACCESS_KEY_ID=
AWS_SECRET_ACCESS_KEY=
AWS_DEFAULT_REGION=us-east-1
AWS_BUCKET=

PUSHER_APP_ID=
PUSHER_APP_KEY=
PUSHER_APP_SECRET=
PUSHER_APP_CLUSTER=mt1

MIX_PUSHER_APP_KEY="${PUSHER_APP_KEY}"
MIX_PUSHER_APP_CLUSTER="${PUSHER_APP_CLUSTER}"
```

The important database values are:

```env
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=forumdb
DB_USERNAME=forum
DB_PASSWORD="Test@12345"
```

> ⚠️ Never commit the real `.env` file or real passwords to GitHub.

---

# 1️⃣4️⃣ Install Laravel Dependencies and Run Migration

Go to the project:

```bash
cd /var/www/forum-pipe
```

Install/update Composer dependencies:

```bash
composer update --optimize-autoloader --no-dev
```

Generate the Laravel application key:

```bash
php artisan key:generate
```

Run database migrations:

```bash
php artisan migrate
```

The migration connects Laravel to:

```text
forumdb
```

using the MySQL user:

```text
forum
```

---

# 1️⃣5️⃣ Final Nginx Configuration Check

After making all Nginx changes:

```bash
nginx -t
```

Then restart Nginx:

```bash
systemctl restart nginx
```

The important configuration values are:

```text
server_name forum.devops.com;
root /var/www/forum-pipe/public;
fastcgi_pass unix:/var/run/php/php7.4-fpm.sock;
```

---

# 1️⃣6️⃣ Configure Windows Hosts File

The Ubuntu VM/server IP used in this setup is:

```text
192.168.8.134
```

On Windows, open **Command Prompt as Administrator**.

Open the hosts file:

```cmd
notepad C:\Windows\System32\drivers\etc\hosts
```

Add:

```text
192.168.8.134 forum.devops.com
```

Save the file.

---

# 1️⃣7️⃣ Flush Windows DNS Cache

Run:

```cmd
ipconfig /flushdns
```

Expected:

```text
Successfully flushed the DNS Resolver Cache.
```

---

# 1️⃣8️⃣ Open the Laravel Application

You can open the application directly in a browser:

```text
http://forum.devops.com
```

Or from Command Prompt:

```cmd
start http://forum.devops.com
```

> ⚠️ Correct command:
>
> `start http://forum.devops.com`
>
> Do not type `C:\Windows\System32>start` as part of the command. The `C:\Windows\System32>` text is only the Command Prompt prompt.

---

# 1️⃣9️⃣ Complete Deployment Flow

The complete request flow is:

```text
Windows Browser
       |
       | http://forum.devops.com
       |
       v
Windows hosts file
       |
       | 192.168.8.134
       |
       v
Ubuntu 26.04 Server / VM
       |
       v
Nginx :80
       |
       | /var/www/forum-pipe/public
       |
       v
Laravel public/index.php
       |
       v
PHP 7.4-FPM
       |
       v
Laravel Application
       |
       v
MySQL
       |
       v
forumdb
```

---

# 2️⃣0️⃣ Important Paths

### Laravel Project

```text
/var/www/forum-pipe
```

### Laravel Public Directory

```text
/var/www/forum-pipe/public
```

### Laravel Entry Point

```text
/var/www/forum-pipe/public/index.php
```

### Laravel Environment File

```text
/var/www/forum-pipe/.env
```

### Nginx Configuration

```text
/etc/nginx/sites-available/forum.devops.com.conf
```

### Nginx Enabled Configuration

```text
/etc/nginx/sites-enabled/forum.devops.com.conf
```

### PHP-FPM Socket

```text
/var/run/php/php7.4-fpm.sock
```

### Windows Hosts File

```text
C:\Windows\System32\drivers\etc\hosts
```

---

# 2️⃣1️⃣ Useful Troubleshooting Commands

## Nginx

```bash
systemctl status nginx --no-pager
```

```bash
nginx -t
```

```bash
nginx -T
```

```bash
systemctl restart nginx
```

---

## PHP

```bash
php -v
```

```bash
systemctl status php7.4-fpm.service
```

```bash
ls -l /run/php/
```

---

## MySQL

```bash
systemctl status mysql --no-pager
```

```bash
mysql
```

```sql
show databases;
```

---

## Laravel

```bash
cd /var/www/forum-pipe
```

```bash
ls -la
```

```bash
ls -la public/
```

```bash
php artisan key:generate
```

```bash
php artisan migrate
```

---

## Test the Domain

From Ubuntu:

```bash
curl -I http://forum.devops.com
```

From Windows:

```cmd
ping forum.devops.com
```

Then open:

```text
http://forum.devops.com
```

---

# 2️⃣2️⃣ Final Configuration Summary

| Component | Final Configuration |
|---|---|
| OS | Ubuntu 26.04 |
| Web Server | Nginx |
| Database | MySQL |
| PHP | 7.4 |
| PHP-FPM | 7.4-FPM |
| Composer | Installed globally |
| Laravel Project | `/var/www/forum-pipe` |
| Laravel Public | `/var/www/forum-pipe/public` |
| Database | `forumdb` |
| Database User | `forum` |
| Domain | `forum.devops.com` |
| Server / VM IP | `192.168.8.134` |
| Nginx Config | `/etc/nginx/sites-available/forum.devops.com.conf` |
| PHP-FPM Socket | `/var/run/php/php7.4-fpm.sock` |

---

# 2️⃣3️⃣ Result

The Laravel application is deployed using:

```text
Ubuntu 26.04
      ↓
Nginx
      ↓
PHP 7.4-FPM
      ↓
Laravel
      ↓
MySQL
      ↓
forumdb
```

The custom local domain:

```text
http://forum.devops.com
```

is mapped from Windows to:

```text
192.168.8.134
```

through the Windows hosts file.

---

## ⭐ Learning Outcome

By completing this deployment, you practiced:

- 🐧 Ubuntu server administration
- 🌐 Nginx installation and virtual hosting
- 🗄️ MySQL installation and database/user creation
- 🐘 PHP 7.4 installation
- ⚙️ PHP-FPM configuration
- 📦 Composer installation
- 🚀 Laravel deployment
- 🔐 Linux file permissions and ownership
- 🔗 Windows hosts-file domain mapping
- 🛠️ Nginx configuration and troubleshooting
- 🗃️ Laravel database migration
- 🌐 Custom local domain configuration

---

## 👨‍💻 Author

**MD. JAKIUL RASHID KHAN**
