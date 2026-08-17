# Laravel High Availability Setup with HAProxy

## Overview

This project implements a basic **High Availability (HA)** architecture for a Laravel application using:

- 1 Load Balancer server with HAProxy
- 3 Application servers running Laravel + Nginx
- 1 Dedicated MySQL database server
- UFW firewall rules
- Static IP addressing
- HAProxy round-robin load balancing
- HAProxy health checks
- Session persistence using cookies

## Server Architecture

| Server | Hostname | IP Address | Role |
|---|---|---|---|
| App 1 | `app1` | `192.168.8.134` | Laravel + Nginx |
| Database | `db` | `192.168.8.135` | MySQL |
| App 2 | `app2` | `192.168.8.136` | Laravel + Nginx |
| App 3 | `app3` | `192.168.8.137` | Laravel + Nginx |
| Load Balancer | `lb` | `192.168.8.138` | HAProxy |

Traffic flow:

```text
                    Windows Host
                         |
                         | forum.devops.com
                         v
              +----------------------+
              |   HAProxy / lb       |
              |   192.168.8.138      |
              |        :80            |
              +----------+-----------+
                         |
              Round-Robin Load Balancing
                  /       |        \
                 /        |         \
                v         v          v
        +---------+ +---------+ +---------+
        |  app1   | |  app2   | |  app3   |
        | .134:80 | | .136:80 | | .137:80 |
        +----+----+ +----+----+ +----+----+
             \          |           /
              \         |          /
               +--------v---------+
               |       db        |
               |  .135:3306      |
               |     MySQL       |
               +-----------------+
```

---

# 1. Database Server Setup

The database server was cloned from an existing server and configured with a new static IP.

## Static IP Configuration

File:

```bash
/etc/netplan/01-network-manager-all.yaml
```

Configuration:

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    ens33:
      dhcp4: false
      addresses:
        - 192.168.8.135/24
      routes:
        - to: default
          via: 192.168.8.2
      nameservers:
        addresses:
          - 8.8.8.8
          - 1.1.1.1
```

Apply the configuration:

```bash
netplan generate
netplan apply
```

Verify:

```bash
ip a show ens33
ip r
```

## Set Database Hostname

```bash
hostnamectl set-hostname db
hostname
```

Expected:

```text
db
```

---

# 2. Remove Unnecessary Web/PHP Packages from DB

The database server does not need Nginx or PHP.

Stop and remove Nginx:

```bash
systemctl status nginx
systemctl stop nginx
apt remove nginx -y
```

Check listening services:

```bash
netstat -tulpn
```

Remove PHP packages if they are not required on the DB server:

```bash
apt remove -y php7.4 php7.4-common php7.4-opcache php7.4-cli \
php7.4-gd php7.4-curl php7.4-mysql php7.4-xmlrpc php7.4-imap \
php7.4-mbstring php7.4-xml php7.4-fpm php7.4-zip php7.4-bcmath
```

---

# 3. MySQL Configuration on DB Server

The MySQL server was configured to listen for connections from the application servers.

Edit:

```bash
nano /etc/mysql/mysql.conf.d/mysqld.cnf
```

Change:

```ini
bind-address = 127.0.0.1
```

to an address that permits remote application-server connections, for example:

```ini
bind-address = 0.0.0.0
```

Restart MySQL:

```bash
systemctl restart mysql.service
```

Verify:

```bash
systemctl status mysql.service
netstat -tulpn
```

MySQL should be listening on port:

```text
3306
```

> Security note: Binding MySQL to `0.0.0.0` exposes the service on all interfaces. Access must therefore be restricted with UFW and MySQL user privileges.

---

# 4. Configure Laravel Application Database Connection

On the application server:

```bash
cd /var/www/forum-pipe
nano .env
```

The Laravel database configuration should point to the dedicated DB server:

```env
DB_CONNECTION=mysql
DB_HOST=192.168.8.135
DB_PORT=3306
DB_DATABASE=forumdb
DB_USERNAME=forum
DB_PASSWORD=Test@12345
```

After changing `.env`, clear Laravel configuration cache if required:

```bash
php artisan config:clear
php artisan cache:clear
```

---

# 5. Create MySQL Application User

Enter MySQL:

```bash
mysql
```

Remove the old localhost-only user:

```sql
DROP USER 'forum'@'localhost';
```

Create a user for the application network:

```sql
CREATE USER 'forum'@'192.168.8.%' IDENTIFIED BY 'Test@12345';
```

Grant database permissions:

```sql
GRANT ALL PRIVILEGES ON forumdb.* TO 'forum'@'192.168.8.%';
```

Apply privileges:

```sql
FLUSH PRIVILEGES;
```

> Note: MySQL account host patterns use `%` as the wildcard. `192.168.8.0/24` is not the normal MySQL account-host wildcard syntax.

---

# 6. Test Laravel Application

After configuring the database connection, the Laravel application was tested through:

```text
http://forum.devops.com/
```

The application was working successfully.

---

# 7. Database Firewall Configuration

Enable UFW on the DB server:

```bash
ufw status
ufw enable
```

Allow SSH only from the local network:

```bash
ufw allow from 192.168.8.0/24 to any port 22 proto tcp
```

Allow MySQL only from the application network:

```bash
ufw allow from 192.168.8.0/24 to any port 3306 proto tcp
```

Check:

```bash
ufw status
```

---

# 8. SSH Server Configuration on DB

Install OpenSSH server:

```bash
apt update
apt install openssh-server -y
```

Enable and start SSH:

```bash
systemctl enable --now ssh
```

From an application server, test SSH:

```bash
ssh soraz@192.168.8.135
```

---

# 9. Clone Application Servers

Two additional application servers were created by cloning the original application server.

Final application-server IPs:

```text
app1 = 192.168.8.134
app2 = 192.168.8.136
app3 = 192.168.8.137
```

---

# 10. App2 Network Configuration

Set hostname:

```bash
hostnamectl set-hostname app2
```

Edit the Netplan configuration:

```bash
cd /etc/netplan/
nano 01-network-manager-all.yaml
```

Change the server IP to:

```yaml
addresses:
  - 192.168.8.136/24
```

Apply:

```bash
netplan generate
netplan apply
```

Verify:

```bash
ip a
ip r
```

---

# 11. App3 Network Configuration

Set hostname:

```bash
hostnamectl set-hostname app3
```

Edit:

```bash
cd /etc/netplan/
nano 01-network-manager-all.yaml
```

Change the IP:

```yaml
addresses:
  - 192.168.8.137/24
```

Apply:

```bash
netplan generate
netplan apply
```

Verify:

```bash
ip a
ip r
```

---

# 12. Load Balancer Server

The cloned server was converted into the HAProxy load balancer.

Set hostname:

```bash
hostnamectl set-hostname lb
```

Configure static IP:

```text
192.168.8.138/24
```

Apply Netplan:

```bash
netplan generate
netplan apply
```

Verify:

```bash
ip a
ip r
```

---

# 13. Remove MySQL from Load Balancer

The load balancer does not require MySQL.

Stop MySQL:

```bash
systemctl stop mysql.service
```

Remove MySQL packages:

```bash
apt remove mysql-server mysql-client mysql-common -y
```

---

# 14. Load Balancer Firewall

Allow HTTP traffic:

```bash
ufw allow 80/tcp
```

Verify:

```bash
ufw status
```

---

# 15. Configure /etc/hosts on HAProxy

Edit:

```bash
vi /etc/hosts
```

Add:

```text
192.168.8.134 app1
192.168.8.135 db
192.168.8.136 app2
192.168.8.137 app3
192.168.8.138 lb
```

This allows the load balancer to resolve the internal server names.

---

# 16. Install HAProxy

Update packages:

```bash
apt-get update
```

Install HAProxy:

```bash
apt-get install haproxy -y
```

Check:

```bash
systemctl status haproxy.service
```

---

# 17. HAProxy Configuration

Configuration file:

```bash
/etc/haproxy/haproxy.cfg
```

Example configuration:

```haproxy
global
    log 127.0.0.1 local2 info
    chroot /var/lib/haproxy
    pidfile /var/run/haproxy.pid
    maxconn 1000
    user haproxy
    group haproxy
    daemon
    stats socket /var/lib/haproxy/stats

defaults
    mode http
    log global
    option httplog
    option dontlognull
    option http-server-close
    option forwardfor except 127.0.0.0/8
    option redispatch
    retries 3
    timeout http-request 600s
    timeout queue 1m
    timeout connect 10s
    timeout client 1m
    timeout server 600s
    timeout http-keep-alive 6000
    timeout check 10s
    maxconn 1000

frontend forumapp
    mode http
    bind *:80

    stats realm Haproxy\ Statistics
    stats show-legends
    stats refresh 60s
    stats enable
    stats auth admin:12345
    stats hide-version
    stats show-node
    stats uri /stats

    maxconn 1000
    default_backend forumback

backend forumback
    mode http
    option httpchk GET /
    http-check expect status 200

    balance roundrobin

    cookie SESSIONID insert

    server app1 192.168.8.134:80 cookie app1 check
    server app2 192.168.8.136:80 cookie app2 check
    server app3 192.168.8.137:80 cookie app3 check
```

---

# 18. HAProxy Configuration Test

Before restarting HAProxy, validate the configuration:

```bash
haproxy -c -f /etc/haproxy/haproxy.cfg
```

Expected result:

```text
Configuration file is valid
```

Restart:

```bash
systemctl restart haproxy.service
```

Check:

```bash
systemctl status haproxy.service
```

Check listening ports:

```bash
netstat -tulpn
```

HAProxy should listen on:

```text
*:80
```

---

# 19. HAProxy Statistics Dashboard

The HAProxy statistics dashboard is available at:

```text
http://192.168.8.138/stats
```

Credentials configured in HAProxy:

```text
Username: admin
Password: 12345
```

The dashboard can be used to monitor:

- app1 status
- app2 status
- app3 status
- Number of active sessions
- Request traffic
- Backend health
- Server availability

---

# 20. Windows Hosts File

To access the application using the domain name from Windows, edit the Windows hosts file as Administrator.

Open:

```powershell
Start-Process notepad "C:\Windows\System32\drivers\etc\hosts" -Verb RunAs
```

Add:

```text
192.168.8.138 forum.devops.com
```

Flush DNS:

```powershell
ipconfig /flushdns
```

Test:

```powershell
ping forum.devops.com
```

Open:

```text
http://forum.devops.com
```

The request now goes to:

```text
forum.devops.com
        |
        v
192.168.8.138 (HAProxy)
        |
        +----> app1 192.168.8.134
        |
        +----> app2 192.168.8.136
        |
        +----> app3 192.168.8.137
```

---

# 21. How High Availability Works

The setup provides application-level high availability.

HAProxy distributes requests between:

```text
app1
app2
app3
```

using:

```text
balance roundrobin
```

HAProxy also performs health checks:

```haproxy
option httpchk GET /
http-check expect status 200
```

If one application server becomes unavailable, HAProxy can detect the failed health check and stop sending normal traffic to that server.

For example:

```text
Normal:

HAProxy
  ├── app1  UP
  ├── app2  UP
  └── app3  UP


If app2 fails:

HAProxy
  ├── app1  UP
  ├── app2  DOWN
  └── app3  UP
```

Traffic can continue through the healthy application servers.

---

# 22. What Was Achieved

This deployment completed the following tasks:

- Configured a dedicated MySQL database server.
- Removed unnecessary Nginx/PHP packages from the DB server.
- Configured MySQL for remote application-server connections.
- Created a dedicated Laravel MySQL user.
- Restricted DB access using UFW.
- Created three application servers.
- Assigned unique static IP addresses to each server.
- Configured separate hostnames: `app1`, `app2`, `app3`.
- Created a dedicated HAProxy load-balancer server.
- Removed MySQL from the load-balancer server.
- Installed and configured HAProxy.
- Added three Laravel application servers to the HAProxy backend.
- Enabled round-robin load balancing.
- Enabled application health checks.
- Enabled HAProxy statistics dashboard.
- Configured Windows DNS resolution through the hosts file.
- Successfully accessed the Laravel application through `forum.devops.com`.

---

# 23. Important Security Improvements

The lab setup works, but the following should be improved for production:

## MySQL

Instead of allowing the complete `/24` network:

```bash
ufw allow from 192.168.8.0/24 to any port 3306 proto tcp
```

allow only the application servers:

```bash
ufw allow from 192.168.8.134 to any port 3306 proto tcp
ufw allow from 192.168.8.136 to any port 3306 proto tcp
ufw allow from 192.168.8.137 to any port 3306 proto tcp
```

## HAProxy Statistics

Do not use a simple password such as:

```text
admin:12345
```

Use a strong password in production.

## MySQL Password

Replace the example password:

```text
Test@12345
```

with a strong secret and preferably manage secrets securely.

## Laravel Sessions

For true multi-server Laravel HA, application sessions should be stored centrally, such as:

- Redis
- Database
- Another shared session store

Otherwise, users may experience session problems when requests move between application servers.

## Uploaded Files

If Laravel allows user uploads, local files on one application server are not automatically available on the other servers.

Consider:

- S3/object storage
- Shared storage
- Centralized file storage

## Database High Availability

This setup has **one database server**, so the application tier is redundant but the database is still a single point of failure.

For full infrastructure HA, consider:

- MySQL replication
- MySQL InnoDB Cluster
- MySQL Group Replication
- Managed database service

---

# 24. Final Architecture

```text
                    Client
                      |
                      | HTTP :80
                      v
              +----------------+
              |    HAProxy     |
              | 192.168.8.138  |
              +-------+--------+
                      |
              Round-Robin
          +-----------+-----------+
          |           |           |
          v           v           v
     +---------+ +---------+ +---------+
     |  app1   | |  app2   | |  app3   |
     | .134    | | .136    | | .137    |
     | Nginx   | | Nginx   | | Nginx   |
     | Laravel | | Laravel | | Laravel |
     +----+----+ +----+----+ +----+----+
          \           |           /
           \          |          /
            +---------v---------+
            |        DB        |
            |   192.168.8.135  |
            |      MySQL       |
            +------------------+
```

## Result

The Laravel application is now accessible through:

```text
http://forum.devops.com
```

with HAProxy acting as the single entry point and distributing traffic across three application servers.
