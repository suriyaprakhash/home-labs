# HomeLab Gateway: Lightsail to Local Machine Bridge

**Target OS:** Amazon Linux 2023 (AL2023)  
**Hardware Profile:** 512 MB RAM / 1 vCPU  
**Core Stack:** Tailscale + Nginx + GoAccess + Fail2Ban

This guide provides a memory-efficient way to use an AWS Lightsail instance as a public gateway for services running on a local machine (e.g., a NAS or Home Server) via a secure Tailscale tunnel.

---

## Phase 1: Resource Optimization (The Safety Net)
**Reasoning:** AL2023 with 512MB RAM will crash during package installations or SSL generation without virtual memory. We must enable Swap first.

```bash
# 1. Create a 1GB swap file
sudo dd if=/dev/zero of=/swapfile bs=1M count=1024
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile

# 2. Persist swap across reboots
echo '/swapfile swap swap defaults 0 0' | sudo tee -a /etc/fstab

# 3. Verify memory status
free -h
```

## Phase 2: Private Networking (Tailscale)
**Reasoning:** Tailscale creates an encrypted tunnel between your AWS VPS and your home machine without opening ports on your home router.

```
# 1. Install Tailscale
curl -fsSL [https://tailscale.com/install.sh](https://tailscale.com/install.sh) | sh

# 2. Enable and start the daemon
sudo systemctl enable --now tailscaled

# 3. Connect to your Tailnet
# Follow the URL provided in the terminal to authenticate
sudo tailscale up

# 4. Identify IPs
# Note the 100.x.y.z IP of this VPS and your HOME machine
tailscale status
```

## Phase 3: Web Server & Monitoring Tools
**Reasoning:** We use Nginx as the reverse proxy and GoAccess for a lightweight, real-time dashboard that uses significantly less RAM than heavy GUI alternatives.

```
# 1. Install Nginx and GoAccess
sudo dnf update -y
sudo dnf install nginx goaccess -y

# 2. Start Nginx
sudo systemctl enable --now nginx

# 3. Prepare Monitoring Directory
sudo mkdir -p /var/www/html/monitor
sudo chown nginx:nginx /var/www/html/monitor
```

## Phase 4: Reverse Proxy Configuration
**Reasoning:** Configure Nginx to route traffic from the Public IP to your local Tailscale IP.

- Create a new config: sudo nano /etc/nginx/conf.d/bridge.conf
- Paste the following template (Replace 100.x.y.z:8080 with your actual local machine's Tailscale IP/Port):

```
server {
    listen 80;
    server_name _; # Responds to your Lightsail Static IP

    # Primary Bridge to Home Lab
    location / {
        proxy_pass [http://100.](http://100.)x.y.z:8080; 
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # Optimization for low-RAM instances
        proxy_buffer_size 12k;
        proxy_buffers 4 24k;
        proxy_busy_buffers_size 24k;
    }

    # Monitoring Dashboard (Protected: Tailscale Only)
    location /monitor {
        alias /var/www/html/monitor/;
        index index.html;
        allow 100.0.0.0/8; # Allow only Tailscale network
        deny all;
    }
}
```

- Test and Reload:

```
sudo nginx -t
sudo systemctl reload nginx
```

## Phase 5: Security Hardening
**Reasoning:** Public IPs are targets for bots. These steps block unauthorized access.

- Internal Firewall (Firewalld)

```
sudo dnf install firewalld -y
sudo systemctl enable --now firewalld

# Open Web ports
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
# Allow Tailscale communication
sudo firewall-cmd --permanent --add-port=41641/udp

sudo firewall-cmd --reload
```

- Intrusion Prevention (Fail2Ban)

```
# Install via Python's package manager for AL2023 compatibility
sudo dnf install python3-pip -y
sudo pip3 install fail2ban

# Start basic protection
sudo fail2ban-client -x start
```

- SSL Encryption (Certbot)

```
sudo dnf install certbot python3-certbot-nginx -y
sudo certbot --nginx
```


## Phase 6: Active Monitoring
**Action:** Start the GoAccess background process to generate the live web dashboard.

```
sudo goaccess /var/log/nginx/access.log -o /var/www/html/monitor/index.html --log-format=COMBINED --real-time-html --daemonize
```

## Maintenance & Verification

- Check Bridge Health: tailscale status (Ensure home node is active; direct).
- RAM Health: free -h (Verify swap is functioning).
- Clear DNF Cache: sudo dnf clean all (Reduces disk and RAM overhead).
- Access Monitor: Connect via Tailscale and visit http://[Lightsail-Tailscale-IP]/monitor.
