# HomeLab Gateway: Debian 12 to Local Machine Bridge

# Pre-requisite

```
1. Create the user
Switch to root (su -) if you aren't already, then run:

# Replace 'suriya' with your preferred username
adduser suriya

2. Install Sudo (Debian doesn't always include it)
apt update && apt install sudo

3. Add your user to the Sudo group
usermod -aG sudo suriya

4. Switch to your new user
su - suriya
```


# Phase 1: Resource Optimization (Swap)
Debian handles swap similarly, but it is critical for 512MB RAM instances to avoid "Out of Memory" (OOM) errors during apt upgrades or Certbot runs.

```
# 1. Create a 2GB swap file (1M * 2048 = 2GB)
sudo dd if=/dev/zero of=/swapfile bs=1M count=2048

# 2. Secure the file (only root should read/write it)
sudo chmod 600 /swapfile

# 3. Set up the swap area
sudo mkswap /swapfile

# 4. Enable the swap
sudo swapon /swapfile

# 5. Verify the new size
free -h
```

# Phase 2: Private Networking (Tailscale)
The installation script works perfectly on Debian.

```
# Install Tailscale
curl -fsSL https://tailscale.com/install.sh | sh

# Start and Authenticate
sudo tailscale up --advertise-exit-node
```

Now on the vps to allow forward so that you could use it as a exit node
```
# Update sysctl.conf
sudo nano /etc/sysctl.conf

# uncomment the following line
net.ipv4.ip_forward=1
net.ipv6.conf.all.forwarding=1

# Save and exit (Ctrl+O, Enter, Ctrl+X).

# Force apply
sudo sysctl -p
```


# Phase 3: Web Server & Monitoring
Debian's package names are slightly different for GoAccess dependencies.

```
# 1. Install Nginx, GoAccess, Nginx UI
sudo apt update
sudo apt install nginx goaccess -y
# curl -L -s https://githubusercontent.com | sudo bash -s -- install

# 2. Prepare Monitoring Directory
sudo mkdir -p /var/www/html/monitor
sudo chown www-data:www-data /var/www/html/monitor
```

# Phase 4: Reverse Proxy Configuration
The Nginx configuration remains mostly the same, but ensure you use www-data (the Debian default user) if you modify permissions.


First secure nginx using htpassd util,
```
sudo apt update
sudo apt install apache2-utils
sudo htpasswd -c /etc/nginx/.htpasswd your_username
```

Create config:
```
sudo nano /etc/nginx/conf.d/bridge.conf
```

Paste this template (Update the proxy_pass IP to your local home machine's Tailscale IP):
```
# Nginx
server {
    listen 80 default_server;
    listen [::]:80 default_server;
    server_name _;

    # These two lines protect the ENTIRE server
    auth_basic "Restricted Access - Home Lab";
    auth_basic_user_file /etc/nginx/.htpasswd;

    location / {
	# Force authentication here as well
        auth_basic "Restricted Access";
        auth_basic_user_file /etc/nginx/.htpasswd;

        proxy_pass http://100.82.34.11:8080; 
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

	# Force authentication here as well
        auth_basic "Restricted Access";
        auth_basic_user_file /etc/nginx/.htpasswd;

        alias /var/www/html/monitor/;
        index index.html;

        # Essential for Real-Time GoAccess
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "Upgrade";

        allow 100.82.34.11; # mac
	allow 100.10.196.108; # phone
        deny all;
    }
}

# Ctrl + o, Enter, Ctrl + x to save and exit

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
