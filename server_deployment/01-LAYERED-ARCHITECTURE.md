---
layout: default
title: Layered Architecture
---

# 01 - Server Deployment Layered Architecture

## The 7 Core Layers (Every Deployment Has These)

```
Layer 7: APPLICATION        (Your code - Node, PHP, WordPress, React)
Layer 6: PROCESS MANAGEMENT (PM2, systemd, supervisor)
Layer 5: WEB SERVER         (Nginx, Apache - reverse proxy, static files)
Layer 4: RUNTIME            (Node.js, PHP-FPM, Python)
Layer 3: SECURITY           (Firewall, SSH, SSL, fail2ban)
Layer 2: NETWORK            (IP, DNS, Ports, Domain)
Layer 1: INFRASTRUCTURE     (VPS/EC2, OS, Disk, RAM)
```

---

## Layer 1: INFRASTRUCTURE

### What it is
The physical/virtual machine and operating system.

### Key Decisions
| Decision | Options | When to Choose |
|----------|---------|---------------|
| Provider | AWS EC2, DigitalOcean, Linode, Vultr | AWS = enterprise/complex. DO = simple/fast |
| OS | Ubuntu 22.04 LTS, Debian 12, Amazon Linux 2 | Ubuntu = best community support for beginners |
| Size | 1GB RAM / 1 vCPU minimum | WordPress: 2GB min. Node API: 1GB ok. Next.js: 2GB+ |
| Region | Closest to your users | Use latency testing tools to decide |
| Storage | SSD always | 20GB minimum, 50GB+ for WordPress with media |

### What You Must Know
```bash
# Check your server resources
free -h              # RAM usage
df -h                # Disk usage
nproc                # CPU cores
lsb_release -a       # OS version
uptime               # How long server has been running
top                  # Live resource usage (q to quit)
htop                 # Better version of top (install: apt install htop)
```

### Common Mistakes
- Choosing too small a server (1GB for WordPress = pain)
- Not enabling swap on small servers
- Picking a region far from users
- Using a non-LTS Ubuntu version (odd numbers like 23.04 expire fast)

### Setup Swap (Important for 1-2GB Servers)
```bash
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

---

## Layer 2: NETWORK

### What it is
How traffic reaches your server.

### The Flow
```
User types example.com
       |
       v
DNS Resolver (Cloudflare/Route53) --> Finds IP: 143.198.x.x
       |
       v
Internet routes to your server IP
       |
       v
Server firewall checks: Is port 80/443 open? --> YES
       |
       v
Nginx listening on port 80/443 receives request
       |
       v
Nginx forwards to your app (localhost:3000)
```

### Key Concepts
- **IP Address**: Your server's address on the internet
- **Ports**: Like doors on your server (80=HTTP, 443=HTTPS, 22=SSH)
- **DNS**: Maps domain names to IP addresses
- **A Record**: domain.com -> IP address
- **CNAME**: subdomain.domain.com -> another domain
- **Firewall**: Controls which ports are open/closed

### See `02-NETWORK-LAYER.md` for deep dive

---

## Layer 3: SECURITY

### What it is
Protecting your server from unauthorized access.

### Minimum Security (Do This Always)
```bash
# 1. SSH key-only login (disable password auth)
sudo nano /etc/ssh/sshd_config
# Set: PasswordAuthentication no
# Set: PermitRootLogin no
sudo systemctl restart sshd

# 2. Create a non-root user
adduser deploy
usermod -aG sudo deploy

# 3. Firewall
sudo ufw allow OpenSSH
sudo ufw allow 'Nginx Full'    # or: sudo ufw allow 80 && sudo ufw allow 443
sudo ufw enable
sudo ufw status

# 4. Auto security updates
sudo apt install unattended-upgrades
sudo dpkg-reconfigure -plow unattended-upgrades

# 5. Fail2ban (blocks brute force SSH)
sudo apt install fail2ban
sudo systemctl enable fail2ban
```

### Security Layers Explained
```
[Internet]
    |
    v
[Cloud Firewall]      -- AWS Security Groups / DO Firewall (Layer 1 - provider level)
    |
    v
[OS Firewall - ufw]   -- Controls port access (Layer 2 - server level)
    |
    v
[Fail2ban]             -- Blocks repeated failed logins (Layer 3 - brute force)
    |
    v
[SSH Key Auth]         -- Only key holders can connect (Layer 4 - authentication)
    |
    v
[Non-root User]       -- Limits damage if compromised (Layer 5 - authorization)
    |
    v
[App Security]         -- Input validation, CORS, rate limiting (Layer 6 - application)
```

---

## Layer 4: RUNTIME

### What it is
The language/platform that runs your code.

### Per Stack
| Stack | Runtime | Install Method |
|-------|---------|---------------|
| Node.js | Node + npm | Use `nvm` (never apt install node) |
| PHP | PHP-FPM | `apt install php8.x-fpm` |
| WordPress | PHP-FPM + MySQL/MariaDB | PHP + database |
| Python | Python + pip | Use `pyenv` |
| Next.js | Node.js | Same as Node.js |
| React (static) | None (just files) | Build locally, serve static |

### Node.js Setup (The Right Way)
```bash
# Install nvm
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.0/install.sh | bash
source ~/.bashrc

# Install Node
nvm install 20       # LTS version
nvm use 20
nvm alias default 20

# Verify
node -v
npm -v
```

### PHP Setup
```bash
sudo apt install php8.2-fpm php8.2-mysql php8.2-xml php8.2-curl php8.2-mbstring php8.2-zip php8.2-gd
sudo systemctl status php8.2-fpm
```

---

## Layer 5: WEB SERVER (Nginx/Apache)

### Why You Need It
Your app runs on port 3000/5000/etc. Users expect port 80 (HTTP) and 443 (HTTPS).
Nginx sits in front and forwards requests.

### What Nginx Does
```
Internet Request (port 443)
       |
       v
    [Nginx]
       |-- Handles SSL/HTTPS
       |-- Serves static files directly (fast)
       |-- Forwards dynamic requests to your app
       |-- Can load balance between multiple app instances
       |
       v
    [Your App on localhost:3000]
```

### Nginx vs Apache
| Feature | Nginx | Apache |
|---------|-------|--------|
| Performance | Better for high concurrency | Good for .htaccess heavy setups |
| Config | Server blocks | Virtual hosts |
| Use case | Node/Next/React/API | WordPress/PHP legacy |
| Static files | Excellent | Good |
| Recommendation | **Default choice** | Only if WordPress needs .htaccess |

### Basic Nginx Config (Node.js App)
```nginx
# /etc/nginx/sites-available/myapp
server {
    listen 80;
    server_name example.com www.example.com;

    location / {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
    }
}
```

### Basic Nginx Config (React/Static)
```nginx
server {
    listen 80;
    server_name example.com;

    root /var/www/myapp/build;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;  # SPA routing
    }

    # Cache static assets
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg)$ {
        expires 30d;
        add_header Cache-Control "public, immutable";
    }
}
```

### Enable and Test Config
```bash
sudo ln -s /etc/nginx/sites-available/myapp /etc/nginx/sites-enabled/
sudo nginx -t            # Test config syntax
sudo systemctl reload nginx
```

---

## Layer 6: PROCESS MANAGEMENT

### Why You Need It
If your Node.js app crashes at 3 AM, who restarts it? PM2 does.

### PM2 (For Node.js)
```bash
# Install
npm install -g pm2

# Start app
pm2 start app.js --name "myapp"
pm2 start npm --name "myapp" -- start    # For npm start
pm2 start "next start" --name "myapp"    # For Next.js

# Essential commands
pm2 list                  # See all running apps
pm2 logs myapp            # View logs
pm2 restart myapp         # Restart
pm2 stop myapp            # Stop
pm2 delete myapp          # Remove from PM2

# CRITICAL: Make PM2 survive reboot
pm2 startup               # Generates startup script (run the command it gives you)
pm2 save                  # Saves current process list

# Ecosystem file (recommended for production)
# ecosystem.config.js
module.exports = {
  apps: [{
    name: 'myapp',
    script: 'app.js',
    instances: 'max',      # Use all CPU cores
    exec_mode: 'cluster',  # Cluster mode
    env: {
      NODE_ENV: 'production',
      PORT: 3000
    }
  }]
};
# Then: pm2 start ecosystem.config.js
```

### systemd (For PHP/Python/anything)
```bash
# Create service file
sudo nano /etc/systemd/system/myapp.service
```
```ini
[Unit]
Description=My Application
After=network.target

[Service]
Type=simple
User=deploy
WorkingDirectory=/var/www/myapp
ExecStart=/usr/bin/node app.js
Restart=on-failure
RestartSec=10
Environment=NODE_ENV=production
Environment=PORT=3000

[Install]
WantedBy=multi-user.target
```
```bash
sudo systemctl daemon-reload
sudo systemctl start myapp
sudo systemctl enable myapp    # Start on boot
sudo systemctl status myapp    # Check status
```

---

## Layer 7: APPLICATION

### What it is
Your actual code and its dependencies.

### Deployment Flow (Same for All Stacks)
```
1. Get code onto server (git clone / git pull / scp / rsync)
2. Install dependencies (npm install / composer install)
3. Build if needed (npm run build)
4. Set environment variables (.env file)
5. Run database migrations if any
6. Start/restart the application
7. Verify it's working
```

### Environment Variables
```bash
# NEVER commit .env to git
# On server, create .env manually or use:
# - AWS Parameter Store
# - DigitalOcean App Platform env vars
# - Vault (advanced)

# Example .env
DATABASE_URL=mysql://user:pass@localhost:3306/mydb
NODE_ENV=production
PORT=3000
JWT_SECRET=your-secret-here
```

---

## How Layers Connect - Full Request Flow

```
User: https://myapp.com/api/users
         |
    [DNS: Route53/Cloudflare]  --> Resolves to 143.198.50.100
         |
    [Cloud Firewall]           --> Port 443 allowed
         |
    [Server Firewall (ufw)]    --> Port 443 allowed
         |
    [Nginx :443]               --> SSL terminates here
         |                         Forwards to localhost:3000
    [PM2]                      --> Manages Node.js process
         |
    [Node.js App :3000]        --> Processes request
         |
    [Database]                 --> Queries data
         |
    [Response flows back up through all layers]
```

---

## Layer Checklist (Use Before Every Deployment)

- [ ] Infrastructure: Server provisioned, OS updated, swap enabled
- [ ] Network: IP noted, DNS pointed, ports identified
- [ ] Security: SSH keys, firewall, non-root user, fail2ban
- [ ] Runtime: Correct version installed (nvm/apt)
- [ ] Web Server: Nginx configured, tested, enabled
- [ ] Process Mgmt: PM2/systemd configured, startup enabled
- [ ] Application: Code deployed, deps installed, built, env vars set
- [ ] SSL: Certbot installed, certificate issued
- [ ] Monitoring: Logs accessible, basic health check in place
