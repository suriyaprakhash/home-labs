# HomeLab Gateway: Debian 12 to Local Machine Bridge


# Phase 1: Resource Optimization (Swap)
Debian handles swap similarly, but it is critical for 512MB RAM instances to avoid "Out of Memory" (OOM) errors during apt upgrades or Certbot runs.

```
# 1. Create a 1GB swap file
sudo dd if=/dev/zero of=/swapfile bs=1M count=1024
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile

# 2. Persist swap across reboots

echo '/swapfile swap swap defaults 0 0' | sudo tee -a /etc/fstab

# 3. Verify
free -h
```

# Phase 2: Private Networking (Tailscale)
The installation script works perfectly on Debian.

```
# Install Tailscale
curl -fsSL https://tailscale.com/install.sh | sh

# Start and Authenticate
sudo tailscale up
```


# Phase 3: Web Server & Monitoring
Debian's package names are slightly different for GoAccess dependencies.

```
# 1. Install Nginx and GoAccess
sudo apt update
sudo apt install nginx goaccess -y

# 2. Prepare Monitoring Directory
sudo mkdir -p /var/www/html/monitor
sudo chown www-data:www-data /var/www/html/monitor
```

# Phase 4: Reverse Proxy Configuration
The Nginx configuration remains mostly the same, but ensure you use www-data (the Debian default user) if you modify permissions.

Create config: sudo nano /etc/nginx/conf.d/bridge.conf

Paste this template (Update the proxy_pass IP to your local home machine's Tailscale IP):

```
# Nginx
server {
    listen 80;
    server_name _; 

    location / {
        proxy_pass http://100.x.y.z:8080; 
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # Buffer tweaks for 512MB RAM
        proxy_buffer_size 12k;
        proxy_buffers 4 24k;
        proxy_busy_buffers_size 24k;
    }

    location /monitor {
        alias /var/www/html/monitor/;
        index index.html;
        allow 100.0.0.0/8; # Tailscale Only
        deny all;
    }
}

# Test & Reload:
sudo nginx -t
sudo systemctl reload nginx
```

# Phase 5: Security Hardening (Debian Style)
Debian typically uses ufw or nftables rather than firewalld.


```
# Firewall (UFW):
sudo apt install ufw -y
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw allow 41641/udp
sudo ufw enable
```

Intrusion Prevention (Fail2Ban):
On Debian, Fail2Ban is available directly via apt (no need for pip).
```
sudo apt install fail2ban -y
# It starts automatically with a default SSH jail
sudo systemctl enable fail2ban
```

SSL (Certbot):
```
sudo apt install certbot python3-certbot-nginx -y
sudo certbot --nginx
```

# Phase 6: Active Monitoring
Run GoAccess as a daemon to keep the dashboard updated.

```
sudo goaccess /var/log/nginx/access.log -o /var/www/html/monitor/index.html --log-format=COMBINED --real-time-html --daemonize
```
