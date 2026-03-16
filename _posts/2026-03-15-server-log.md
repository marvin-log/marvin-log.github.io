---
layout: post
title: "Server Migration & Maintenance Log (Region A → US)"
date: 2026-03-15 12:00:00 +0800
---

Migration of LLC-A platform from an AWS region-A instance to a US instance, including Nginx, PHP-FPM, MySQL, TCP/HTTP services, and a new public-facing site. Summary below.

---

## 1. Backup and transfer

### 1.1 Source server (region A) backup

- **PHP application**: Full tree as `www.tar.gz` (main app, sub-apps, websocket services).
- **Nginx**: `/etc/nginx/` as `nginx.tar.gz`.
- **PHP-FPM**: `/etc/php/7.4/` (and related) as `php.tar.gz`.
- **MySQL**: Full dump as `all.sql`.
- **Let's Encrypt**: `/etc/letsencrypt/` (certs and config).
- **Cron**: `crontab -l` and `/etc/cron.d/` if needed.

### 1.2 Transfer to target server (US)

- Use **rsync** or **scp** from source public IP to US instance public IP.
- SSH key: `.pem` must be `chmod 600` or SSH will reject.
- On target: create `/backup/` and ensure write permission for the transfer user (e.g. `sudo chown ubuntu:ubuntu /backup`).

---

## 2. Target server setup

### 2.1 Paths

- **OS**: Ubuntu; web root: `/var/www/`.
- If archives unpack to something like `/var/var/www`, move contents into `/var/www/`.

### 2.2 Nginx

- Install: `apt install nginx`; restore config under `/etc/nginx/`.
- **User**: If config has `user nobody;`, on Ubuntu use `user nobody nogroup;` to avoid `getgrnam("nobody") failed`.
- **Modules**: If `modules-enabled` symlinks are broken, comment out in `nginx.conf`: `# include /etc/nginx/modules-enabled/*.conf;`.
- **Log paths**: Create any log files referenced in config (e.g. under `/usr/local/nginx/logs/`), then `chown nobody:nogroup` and `chmod 664`.
- Test and reload: `nginx -t` then `systemctl reload nginx`.

### 2.3 PHP-FPM (7.4)

- Install `php7.4-fpm`, `php7.4-mysql`, etc. (add Ondřej PPA if 7.4 is not in default repos).
- **Pool user**: Match Nginx — set in `/etc/php/7.4/fpm/pool.d/www.conf`:  
  `user = nobody`, `group = nogroup`, `listen.owner = nobody`, `listen.group = nogroup`.
- **Log dir**: Create `/var/log/php/` and `chown nobody:nogroup`.
- **Session dir**: For `/var/lib/php/sessions`, run  
  `sudo chown -R nobody:nogroup /var/lib/php/sessions` and `sudo chmod 1733 /var/lib/php/sessions`.
- Restart: `systemctl restart php7.4-fpm`.

### 2.4 MySQL

- Install MySQL, create DB(s), then `mysql < /backup/all.sql`.
- If you see `Access denied for user 'root'@'localhost'`: use `sudo mysql`, then  
  `ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'your_password';`  
  `FLUSH PRIVILEGES;`
- Update app config (e.g. `customConfig.php`, `database.php`) with correct host, DB name, and credentials.

### 2.5 Let's Encrypt

- If backup was extracted to `/letsencrypt/`, copy into Nginx's path:  
  `sudo cp -a /letsencrypt/accounts /letsencrypt/archive ... /etc/letsencrypt/`.
- Ensure `/etc/letsencrypt/live/your-domain/` contains `fullchain.pem` and `privkey.pem` and is readable by Nginx.

### 2.6 Web directory permissions

- With Nginx and PHP-FPM as **nobody:nogroup**:  
  `sudo chown -R nobody:nogroup /var/www/<app-root>` (and any other site roots).  
  `sudo chmod -R 755` (775 for writable dirs such as cache, logs).
- **application/cache**: If PHP reports write failures, ensure nobody can write:  
  `sudo chown -R nobody:nogroup /var/www/<app-root>/application/cache` and `chmod -R 775`.

---

## 3. TCP / HTTP services (device listener and internal HTTP)

### 3.1 Service scripts and paths

- Several `service.sh` scripts (for main app socket, sub-app sockets, websocket) may still reference old `user`, `phppath`, and `filepath`. Update to the new server, e.g.  
  `user="nobody"`, `phppath="/usr/bin/php7.4"`, `filepath="/var/www/<app-root>/socket"`.

### 3.2 Swoole extension

- **Swoole 4.8.x** (e.g. 4.8.13) is required. Install via `pecl install swoole-4.8.13` or `apt install php7.4-swoole` if available.
- Confirm load: `php7.4 -m | grep swoole` for both FPM and CLI.
- **Server script**: If you see `unsupported option`, comment out the offending options in `$serv->set()` (e.g. `request_slowlog_file`, `request_slowlog_timeout`, `trace_event_worker`).

### 3.3 Listen address and firewall

- **Device TCP port**: For external device access, set the TCP listen host to `0.0.0.0` in the service config. Open this port in AWS security group and, if used, in UFW.
- **Internal HTTP port**: If only used locally, keep the HTTP listener on `127.0.0.1`.
- Open required ports (80, 443, and the device TCP port) in the security group and run `ufw allow ...` then `ufw reload` if UFW is enabled.

### 3.4 Data flow and troubleshooting

- Flow: devices → TCP listener → receive handler → task workers **curl** the backend (e.g. internal API base URL paths for timezone, logic, and record endpoints).
- If the client app does not show sensor/state data: verify the TCP listener is bound (e.g. `ss -lntp | grep <port>`), that curl from the server to the backend URL returns 200, and check `/var/log/php/php_errors.log` for timeouts or fatals.
- **Redis**: If the internal HTTP API or other services depend on Redis, install it and match the connection settings used in the socket/API code.

---

## 4. New public site and domain

- Create site root, e.g. `sudo mkdir -p /var/www/public-site`.
- Add an Nginx server block (e.g. `/etc/nginx/conf.d/company.conf`) with correct `server_name`, `root`, `index`, and log paths.
- Unzip uploaded assets: `sudo unzip -o archive.zip -d /var/www/public-site`, then `chown -R nobody:nogroup /var/www/public-site`.
- HTTPS: After DNS for the domain points to the US server, run `certbot --nginx -d example.com -d www.example.com`.
- DNS: Create A records for the domain (and www) to the US instance public IP.

---

## 5. Other fixes and consistency

- **Unified process user**: Use **nobody:nogroup** for both Nginx and PHP-FPM to avoid permission mismatches.
- **Uploads via SFTP**: If uploading as user `ubuntu` into `/var/www/`, either add ubuntu to the appropriate group and use `chmod 775` on site dirs, or upload to `/home/ubuntu` and then `sudo cp` into `/var/www/`.
- **Captcha / session**: Captcha needs GD and writable dirs; session errors usually mean `session.save_path` is not owned by nobody.

---

## 6. Mobile app and backend

- After moving servers, update the app's API base URL and any hardcoded IPs (e.g. for device/Wi‑Fi hub configuration), then rebuild and redeploy the app package.

---

Above is a concise log for future reference and repeatability.
![Terminal commands log](4c253ffac2b0f487f9ca9d8659250157.png)
