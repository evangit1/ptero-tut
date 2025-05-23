# Full Pterodactyl Setup: Home Server + VPS Reverse Proxy (Ubuntu, Tailscale, Cloudflare)

This guide walks you through installing the **Pterodactyl Panel** and **Wings** daemon on a home server, making it securely accessible via a VPS reverse proxy. We’ll use **Tailscale** for secure internal networking and **Cloudflare** for DNS + initial SSL, addressing common pitfalls such as redirect loops and Panel-Wings communication errors.

---

## Our Goal

* Run **Pterodactyl Panel** (e.g., `panel.yourdomain.com`) securely.  
* **Panel & Wings** hosted on a home server.  
* **VPS** acts as the public-facing reverse proxy.  
* **Cloudflare** manages DNS and provides SSL to the VPS.  
* **Tailscale** connects the VPS and home server.  

---

## Example IPs & Domains

| Component | Address | Notes |
|-----------|---------|-------|
| **Home Server** (Pterodactyl & Wings) | `100.60.10.1` | Replace with your home server’s Tailscale IP |
| **VPS** (Reverse Proxy) | Public IP `203.0.113.45`<br>Tailscale IP `100.60.20.2` | Replace with your VPS IPs |
| **Panel Domain** | `panel.yourdomain.com` | Replace with your actual domain |
*Note: this tutorial is long and you will want to do lots of copying & pasting, downloading this file and doing 'find and replace' to put in your own info will make it easier

---

## Section 0  Prerequisites – Cloudflare & Tailscale Setup

### 0.1  Cloudflare DNS Setup
1. Sign up / log in to Cloudflare.  
2. Add your domain (e.g., `yourdomain.com`).  
3. Update nameservers at your registrar to the ones Cloudflare gives.  
4. **Create an A Record** for your Panel domain:  
   * **Type:** A  
   * **Name:** `panel` (for `panel.yourdomain.com`)  
   * **IPv4:** `203.0.113.45` (VPS Public IP)  
   * **Proxy status:** *Proxied* (orange cloud).  
5. Save.

### 0.2  Tailscale Installation & Setup
On **both** Home Server & VPS:

```bash
bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

* Follow the authentication URL.
* Find your Tailscale IPs:

```bash
bash
sudo tailscale status
```

* Test connectivity:

```bash
bash
# From VPS
ping 100.60.10.1
# From Home Server
ping 100.60.20.2
```

**Troubleshooting Tailscale**
* Not connecting ➜ ensure `sudo tailscale up` ran and shows logged-in status.  
* Over-restrictive host firewall ➜ relax rules or allow Tailscale.  

### 0.3  Initial Server Preparation & Firewall (UFW)
On **both** servers:

```bash
bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y software-properties-common curl apt-transport-https ca-certificates gnupg ufw
```

#### UFW Rules – Home Server
```bash
bash
sudo ufw allow ssh          # or your custom SSH port
sudo ufw allow http         # Panel served locally
sudo ufw allow 8080/tcp     # Wings API
sudo ufw allow 2022/tcp     # Wings SFTP
# Add game ports later (e.g., 25565/tcp)
sudo ufw enable
sudo ufw status
```

#### UFW Rules – VPS
```bash
bash
sudo ufw allow ssh
sudo ufw allow http
sudo ufw allow https
# Add game ports later if proxying (e.g., 25565)
sudo ufw enable
sudo ufw status
```

---

## Section 1  Pterodactyl Panel Installation (Home Server)

### 1.1  Install LEMP Stack

#### A. Nginx
```bash
bash
sudo apt install -y nginx
sudo systemctl start nginx
sudo systemctl enable nginx
```

#### B. MariaDB
```bash
bash
sudo apt install -y mariadb-server
sudo systemctl start mariadb
sudo systemctl enable mariadb
sudo mysql_secure_installation
```

#### C. PHP (8.3 example)
```bash
bash
sudo add-apt-repository ppa:ondrej/php -y
sudo apt update
sudo apt install -y php8.3-fpm php8.3-common php8.3-mysql php8.3-gd php8.3-mbstring \
php8.3-bcmath php8.3-xml php8.3-curl php8.3-zip php8.3-intl php8.3-cli php8.3-opcache
```

### 1.2  Create Pterodactyl Database
```bash
bash
sudo mysql -u root -p
```
```sql
CREATE DATABASE panel CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'pterodactyl'@'127.0.0.1' IDENTIFIED BY 'YOUR_PANEL_DB_PASSWORD';
GRANT ALL PRIVILEGES ON panel.* TO 'pterodactyl'@'127.0.0.1' WITH GRANT OPTION;
FLUSH PRIVILEGES;
EXIT;
```

### 1.3  Download & Configure Panel

#### A. Directories & Download
```bash
bash
sudo mkdir -p /var/www/pterodactyl
cd /var/www/pterodactyl
curl -Lo panel.tar.gz https://github.com/pterodactyl/panel/releases/latest/download/panel.tar.gz
sudo tar -xzvf panel.tar.gz
sudo rm panel.tar.gz
sudo chmod -R 755 storage/* bootstrap/cache/
```

#### B. Composer & Dependencies
```bash
bash
curl -sS https://getcomposer.org/installer | sudo php -- --install-dir=/usr/local/bin --filename=composer
sudo cp .env.example .env
sudo composer install --no-dev --optimize-autoloader
```

#### C. Edit `.env`
```bash
bash
sudo nano .env
```
Essential lines to update:
```
APP_ENV=production
APP_DEBUG=false
APP_URL=https://panel.yourdomain.com
DB_HOST=127.0.0.1
DB_DATABASE=panel
DB_USERNAME=pterodactyl
DB_PASSWORD=YOUR_PANEL_DB_PASSWORD
SESSION_DRIVER=file      # or redis
CACHE_DRIVER=file        # or redis
QUEUE_CONNECTION=database # or redis
SESSION_SECURE_COOKIE=true
```

#### D. Application Setup
```bash
bash
sudo php artisan key:generate --force
sudo php artisan migrate --seed --force
sudo php artisan p:user:make          # create admin user
```

#### E. Permissions
```bash
bash
sudo chown -R www-data:www-data /var/www/pterodactyl/*
```

#### F. Cronjob
```bash
bash
sudo crontab -e
```
Add:
```
* * * * * php /var/www/pterodactyl/artisan schedule:run >> /dev/null 2>&1
```

#### G. pteroq (service)
```bash
bash
sudo nano /etc/systemd/system/pteroq.service
```
```ini
[Unit]
Description=Pterodactyl Queue Worker
After=network.target

[Service]
User=www-data
Group=www-data
Restart=always
ExecStart=/usr/bin/php /var/www/pterodactyl/artisan queue:work --queue=high,standard,low --sleep=3 --tries=3
StartLimitInterval=180
StartLimitBurst=30
RestartSec=5s

[Install]
WantedBy=multi-user.target
```
```bash
bash
sudo systemctl daemon-reload
sudo systemctl enable --now pteroq.service
```

### 1.4  Nginx (Local HTTP-only) for Panel
```bash
bash
sudo nano /etc/nginx/sites-available/pterodactyl
```
```nginx
# Nginx
server {
    listen 80;
    server_name localhost;            # or 100.60.10.1
    root /var/www/pterodactyl/public;
    index index.php;

    access_log /var/log/nginx/pterodactyl.app-access.log;
    error_log  /var/log/nginx/pterodactyl.app-error.log error;

    client_max_body_size 100m;
    client_body_timeout 120s;
    sendfile off;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_pass unix:/run/php/php8.3-fpm.sock;
        fastcgi_index index.php;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_read_timeout 3600;
    }

    location ~ /\.ht {
        deny all;
    }
}
```
```bash
bash
sudo rm /etc/nginx/sites-enabled/default || true
sudo ln -s /etc/nginx/sites-available/pterodactyl /etc/nginx/sites-enabled/pterodactyl
sudo nginx -t && sudo systemctl reload nginx
```

**Troubleshooting (Section 1)**  
* *500 errors* ➜ check `storage/logs/laravel-*.log` & permissions.  
* *PHP files download* ➜ PHP-FPM socket or Nginx `location ~ \.php` mis-set.  
* *pteroq fails* ➜ `sudo journalctl -u pteroq`.

---

## Section 2  Pterodactyl Wings Installation (Home Server)

### 2.1  Docker
```bash
bash
sudo apt install -y docker.io
sudo systemctl start docker
sudo systemctl enable docker
```

### 2.2  Wings Binary
```bash
bash
sudo mkdir -p /etc/pterodactyl
curl -L -o /usr/local/bin/wings "https://github.com/pterodactyl/wings/releases/latest/download/wings_linux_amd64"
sudo chmod u+x /usr/local/bin/wings
```

### 2.3  Configure Wings via Panel & `config.yml`
* Panel ➜ **Admin → Locations** ➜ create *Home*.  
* Panel ➜ **Admin → Nodes** ➜ *Create New* with:  
  * **FQDN:** `panel.yourdomain.com`  
  * **Communicate Over SSL:** YES  
  * **Behind Proxy:** ✓  
  * **Daemon Port:** 443  
  * **Daemon SFTP Port:** 2022  

Copy generated config, then on Home Server:

```bash
bash
sudo nano /etc/pterodactyl/config.yml
```
Make sure:
```yaml
api:
  host: 0.0.0.0
  port: 8080
  ssl:
    enabled: false
...
remote: 'https://panel.yourdomain.com'
```

### 2.4  Systemd Service for Wings
```bash
bash
sudo nano /etc/systemd/system/wings.service
```
```ini
[Unit]
Description=Pterodactyl Wings Daemon
After=docker.service
Requires=docker.service

[Service]
User=root
WorkingDirectory=/etc/pterodactyl
LimitNOFILE=4096
PIDFile=/var/run/wings/daemon.pid
ExecStart=/usr/local/bin/wings
Restart=on-failure
StartLimitInterval=180
StartLimitBurst=30
RestartSec=5s

[Install]
WantedBy=multi-user.target
```
```bash
bash
sudo systemctl daemon-reload
sudo systemctl enable wings   # (do NOT start yet)
```

**Troubleshooting Wings** ➜ `sudo journalctl -u wings`.

---

## Section 3  Reverse Proxy Configuration (VPS)

### 3.1  Install Nginx & Certbot
```bash
bash
sudo apt update
sudo apt install -y nginx certbot python3-certbot-nginx
```

### 3.2  VPS Nginx for Panel + Wings API
```bash
bash
sudo nano /etc/nginx/sites-available/panel.yourdomain.com
```
```nginx
# Nginx
server {
    listen 80;
    server_name panel.yourdomain.com;

    location /.well-known/acme-challenge/ {
        root /var/www/html;
    }

    location / {
        return 301 https://$host$request_uri;
    }
}

server {
    listen 443 ssl http2;
    server_name panel.yourdomain.com;

    ssl_certificate     /etc/letsencrypt/live/panel.yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/panel.yourdomain.com/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

    # Security Headers
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header X-Robots-Tag "none" always;
    add_header Content-Security-Policy "frame-ancestors 'self'" always;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;

    # Panel API (Application/Client/Remote)
    location ~ ^/api/(application|client|remote)(/.*)?$ {
        proxy_pass http://100.60.10.1:80;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
    }

    # All other /api/ ➜ Wings
    location /api/ {
        proxy_pass http://100.60.10.1:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;

        # WebSocket
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_connect_timeout 600s;
        proxy_send_timeout 600s;
        proxy_read_timeout 600s;
    }

    # Everything else ➜ Panel
    location / {
        proxy_pass http://100.60.10.1:80;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
    }
}
```
```bash
bash
sudo ln -s /etc/nginx/sites-available/panel.yourdomain.com /etc/nginx/sites-enabled/
sudo nginx -t
sudo certbot --nginx -d panel.yourdomain.com   # choose redirect
sudo systemctl reload nginx
```

**Troubleshooting VPS Nginx**
* Certbot fails ➜ confirm DNS points to VPS & port 80 open.  
* 502 Bad Gateway ➜ VPS ↔ Home Server via Tailscale down.  
* Redirect loops ➜ missing `X-Forwarded-Proto https` header.  

---

## Section 4  Final Tweaks & Communication Fixes

### 4.1  Cloudflare SSL/TLS Mode
* Cloudflare → **SSL/TLS → Overview → Full (Strict)**.

### 4.2  `trustedproxy.php`
```bash
bash
sudo nano /var/www/pterodactyl/config/trustedproxy.php
```
```php
<?php

use Illuminate\Http\Request;

return [
    'proxies' => [
        '100.60.20.2',      // VPS Tailscale IP
        // Cloudflare IPv4 ranges
        '173.245.48.0/20',
        '103.21.244.0/22',
        '103.22.200.0/22',
        '103.31.4.0/22',
        '141.101.64.0/18',
        '108.162.192.0/18',
        '190.93.240.0/20',
        '188.114.96.0/20',
        '197.234.240.0/22',
        '198.41.128.0/17',
        '162.158.0.0/15',
        '104.16.0.0/13',
        '104.24.0.0/14',
        '172.64.0.0/13',
        '131.0.72.0/22',
    ],
    'headers' => Request::HEADER_X_FORWARDED_FOR
              | Request::HEADER_X_FORWARDED_HOST
              | Request::HEADER_X_FORWARDED_PORT
              | Request::HEADER_X_FORWARDED_PROTO,
];
```

### 4.3  Re-configure Wings
Generate token in Panel → copy the command, then:

```bash
bash
sudo systemctl stop wings
sudo wings configure --panel-url https://panel.yourdomain.com --token <TOKEN> --node <NODE_ID>
```

Fix `config.yml` if it changed `api.port` or `api.ssl.enabled`.

### 4.4  Clear Panel Caches & Start Wings
```bash
bash
cd /var/www/pterodactyl
sudo php artisan config:clear
sudo php artisan cache:clear
sudo php artisan route:clear
sudo php artisan view:clear
sudo systemctl restart php8.3-fpm
sudo systemctl start wings
```

### 4.5  Test
* Visit `https://panel.yourdomain.com` → heart icon should be green.  
* Create a server ➜ should succeed.

**If issues:**  
* Panel logs ➜ `/var/www/pterodactyl/storage/logs/laravel-*.log`  
* Wings logs ➜ `sudo journalctl -u wings`  
* VPS Nginx ➜ `/var/log/nginx/error.log`  

---

## Section 5  Proxy Game Server Traffic (Optional)

### DNS
* `mc.yourdomain.com` ➜ VPS Public IP, *DNS Only* (grey cloud).

### Nginx Stream on VPS
```bash
bash
sudo apt install -y nginx-full      # if not already
```
Add to `/etc/nginx/nginx.conf` **outside** `http{}`:
```nginx
# Nginx
stream {
    include /etc/nginx/streams-enabled/*.conf;
    include /etc/nginx/sites-enabled/*.stream;
}
```
```bash
bash
sudo nano /etc/nginx/sites-available/minecraft.stream
```
```nginx
# Nginx
server {
    listen 203.0.113.45:25565;
    proxy_pass 100.60.10.1:25565;
    proxy_timeout 10m;
    proxy_connect_timeout 5s;
}
server {
    listen 203.0.113.45:25565 udp;
    proxy_pass 100.60.10.1:25565;
    proxy_timeout 10m;
    proxy_connect_timeout 1s;
}
```
```bash
bash
sudo ln -s /etc/nginx/sites-available/minecraft.stream /etc/nginx/streams-enabled/minecraft.stream
sudo nginx -t && sudo systemctl reload nginx
sudo ufw allow 25565
```
Allocate `mc.yourdomain.com:25565` in Panel.

---

## Final Notes

Common failure points:

1. **SSL termination** must happen only on the outer proxy (VPS).  
2. **`X-Forwarded-Proto`** headers must reach Panel.  
3. **`trustedproxy.php`** must include Cloudflare & VPS IPs.  
4. **Node FQDN / Ports** in Panel must match proxied path.  
5. **Nginx location order** on VPS is critical.  
6. **Wings `api.port` & `api.ssl.enabled`** must reflect local-only HTTP.  
7. Always clear caches after config changes and check logs.  

**Useful Logs**

| Service | Command/Path |
|---------|--------------|
| Panel | `/var/www/pterodactyl/storage/logs/laravel-*.log` |
| Wings | `sudo journalctl -u wings --no-pager -n 200` |
| VPS Nginx | `/var/log/nginx/error.log` |
| Home Nginx | `/var/log/nginx/*.log` |

Good luck—everything should now run smoothly end-to-end!
