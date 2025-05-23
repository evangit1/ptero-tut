```markdown
# Full Pterodactyl Setup: Home Server + VPS Reverse Proxy (Ubuntu, Tailscale, Cloudflare)

This guide walks you through installing the Pterodactyl Panel and Wings daemon on a home server, making it securely accessible via a VPS reverse proxy. We'll use Tailscale for secure internal networking and Cloudflare for DNS and initial SSL.

**Our Goal:**
* Run Pterodactyl Panel (e.g., `panel.yourdomain.com`) securely.
* Panel & Wings hosted on a home server.
* VPS acts as the public-facing reverse proxy.
* Cloudflare manages DNS and provides SSL to the VPS.
* Tailscale connects the VPS and home server.

**Example IPs & Domains:**
* **Home Server (Pterodactyl & Wings):**
    * Tailscale IP: `100.60.10.1` (replace with your home server's Tailscale IP)
* **VPS (Reverse Proxy):**
    * Public IP: `203.0.113.45` (replace with your VPS's public IP)
    * Tailscale IP: `100.60.20.2` (replace with your VPS's Tailscale IP)
* **Panel Domain:** `panel.yourdomain.com` (replace with your actual domain)

---

## Section 0: Prerequisites - Cloudflare & Tailscale Setup

### 0.1. Cloudflare DNS Setup
1. Add your domain to Cloudflare and point your registrar to Cloudflare's nameservers.
2. Create an A record for the panel:
   - **Name:** `panel`
   - **IPv4:** VPS Public IP (`203.0.113.45`)
   - **Proxy:** Proxied (Orange Cloud)

### 0.2. Tailscale Installation & Setup
```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
sudo tailscale status
```
Note Tailscale IPs (100.x.x.x range) on each device.

### 0.3. Initial Server Prep
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y software-properties-common curl apt-transport-https ca-certificates gnupg ufw
```
Set up UFW rules per your scenario (SSH, 80, 443, 8080, 2022).

---

## Section 1: Pterodactyl Panel Installation (Home Server)

### 1.1. LEMP Stack
Install Nginx, MariaDB, PHP 8.3 and dependencies.

### 1.2. Database Setup
```sql
CREATE DATABASE panel CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'pterodactyl'@'127.0.0.1' IDENTIFIED BY 'YOUR_STRONG_PASSWORD_HERE';
GRANT ALL PRIVILEGES ON panel.* TO 'pterodactyl'@'127.0.0.1' WITH GRANT OPTION;
FLUSH PRIVILEGES;
```

### 1.3. Download & Configure Panel
```bash
curl -Lo panel.tar.gz https://github.com/pterodactyl/panel/releases/latest/download/panel.tar.gz
tar -xzvf panel.tar.gz
composer install --no-dev --optimize-autoloader
php artisan migrate --seed --force
php artisan p:user:make
```

Set cronjob and systemd service for queue worker.

---

## Section 2: Wings Installation (Home Server)

### 2.1. Install Docker
```bash
sudo apt install -y docker.io
```

### 2.2. Install Wings
```bash
curl -L -o /usr/local/bin/wings "https://github.com/pterodactyl/wings/releases/latest/download/wings_linux_amd64"
chmod u+x /usr/local/bin/wings
```

### 2.3. Configure Wings
Use config from the Panel UI, modify:
```yaml
api:
  port: 8080
  ssl:
    enabled: false

sftp:
  bind_port: 2022
remote: 'https://panel.yourdomain.com'
```

Enable Wings as systemd service.

---

## Section 3: Reverse Proxy on VPS

### 3.1. Install Nginx & Certbot
```bash
sudo apt install -y nginx certbot python3-certbot-nginx
```

### 3.2. Nginx Proxy Config
Proxy `/api/` to Wings (port 8080) and other traffic to Panel (port 80) over Tailscale. Use Cloudflare certificates.

```nginx
server {
    listen 443 ssl http2;
    server_name panel.yourdomain.com;

    ssl_certificate /etc/letsencrypt/live/panel.yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/panel.yourdomain.com/privkey.pem;

    location ~ ^/api/(system|servers/.*/ws)$ {
        proxy_pass http://100.60.10.1:8080;
        proxy_set_header Host $host;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }

    location / {
        proxy_pass http://100.60.10.1:80;
        proxy_set_header Host $host;
    }
}
```

Enable config and reload Nginx.

---

## Section 4: Fix Mixed Content / Red Heart Issue

### 4.1. Cloudflare
Set SSL mode to **Full (Strict)**.

### 4.2. `trustedproxy.php`
Whitelist:
- VPS Tailscale IP
- All Cloudflare IPs

### 4.3. Clear Pterodactyl Caches
```bash
php artisan config:clear
php artisan cache:clear
php artisan view:clear
```

### 4.4. Panel Node Settings
- FQDN: `panel.yourdomain.com`
- Communicate Over SSL: Yes
- Port: 443
- Behind Proxy: Yes

---

## Section 5: Proxy Game Server Traffic (Optional)

Use Nginx Stream:
```nginx
stream {
    server {
        listen 25565;
        proxy_pass 100.60.10.1:25565;
    }
}
```
Allow port in firewall. Point DNS (e.g., `mc.yourdomain.com`) to VPS.

---

## Final Notes

Troubleshoot with logs:
```bash
sudo journalctl -u wings
tail -f /var/log/nginx/*.log
```

Test from browser in incognito. Good luck!
```
