---
layout: default
title: OS Layer
---

# 13 - OS Layer Deep Dive (Linux for Server Deployment)

## Why This Matters
Every server runs Linux. Every deployment issue eventually traces back to an OS-level concept:
permissions, processes, services, disk, memory, networking. If you understand the OS layer,
you can debug anything.

---

## Linux Directory Structure (Know This Map)

```
/
├── bin/             # Essential commands (ls, cp, mv, cat)
├── sbin/            # System commands (iptables, fdisk) - need root
├── etc/             # ALL configuration files
│   ├── nginx/       #   Nginx config
│   ├── ssh/         #   SSH server config
│   ├── systemd/     #   Service definitions
│   ├── php/         #   PHP config
│   ├── mysql/       #   MySQL config
│   ├── ufw/         #   Firewall rules
│   ├── letsencrypt/ #   SSL certificates
│   ├── hosts        #   Local DNS overrides
│   ├── hostname     #   Server name
│   ├── fstab        #   Disk mount config (swap is here)
│   └── crontab      #   System-level scheduled tasks
│
├── var/             # Variable data (changes at runtime)
│   ├── www/         #   Web app files (YOUR CODE GOES HERE)
│   ├── log/         #   ALL log files
│   │   ├── nginx/   #     Nginx logs
│   │   ├── mysql/   #     MySQL logs
│   │   ├── auth.log #     SSH/login attempts
│   │   └── syslog   #     General system log
│   └── lib/         #   Database data files (MySQL, PostgreSQL)
│       ├── mysql/
│       └── postgresql/
│
├── home/            # User home directories
│   └── deploy/      #   Your deploy user's home
│       ├── .ssh/    #     SSH keys (authorized_keys)
│       ├── .bashrc  #     Shell config
│       ├── .nvm/    #     Node version manager
│       └── .pm2/    #     PM2 data and logs
│
├── root/            # Root user's home (avoid using)
├── tmp/             # Temporary files (cleared on reboot)
├── usr/             # User programs
│   ├── bin/         #   Installed program binaries
│   └── local/       #   Locally compiled programs
├── opt/             # Optional/third-party software
├── proc/            # Virtual filesystem (live system info)
├── dev/             # Device files
└── mnt/ or media/   # Mount points for external drives
```

### Key Paths to Memorize
```
YOUR CODE:        /var/www/
NGINX CONFIG:     /etc/nginx/
SSL CERTS:        /etc/letsencrypt/
SSH CONFIG:       /etc/ssh/sshd_config
LOGS:             /var/log/
USER HOME:        /home/deploy/
PHP CONFIG:       /etc/php/8.x/
MYSQL CONFIG:     /etc/mysql/
SYSTEMD SERVICES: /etc/systemd/system/
CRON JOBS:        crontab -e (per user)
HOSTS FILE:       /etc/hosts
```

---

## Users, Groups & Permissions

### Users That Matter on a Server

| User | Purpose | Notes |
|------|---------|-------|
| `root` | God mode | NEVER run apps as root |
| `deploy` | Your apps | You create this user |
| `www-data` | Nginx/PHP-FPM | Web server process user |
| `mysql` | MySQL process | Database process user |
| `postgres` | PostgreSQL process | Database process user |
| `nobody` | Minimal privilege | Used by some services |

### User Management
```bash
# Create user
adduser deploy                    # Interactive (asks for password)
useradd -m -s /bin/bash deploy    # Non-interactive

# Add to sudo group
usermod -aG sudo deploy

# Switch user
su - deploy                       # Switch to deploy user
sudo -u www-data whoami           # Run command as another user

# Check who you are
whoami
id                                # Shows user, groups, IDs

# Check who is logged in
who
w                                 # More detailed

# List all users
cat /etc/passwd | grep -v nologin | grep -v false
```

### File Permissions Explained
```
-rwxr-xr-- 1 deploy www-data 4096 Jan 15 10:00 app.js
│├─┤├─┤├─┤   │      │
│ │   │  │   owner  group
│ │   │  └── Others: r-- (read only)
│ │   └───── Group:  r-x (read + execute)
│ └───────── Owner:  rwx (read + write + execute)
└──────────── File type: - (regular file), d (directory)

NUMERIC (chmod):
  r = 4, w = 2, x = 1
  rwx = 7, r-x = 5, r-- = 4, --- = 0

  chmod 755 = rwxr-xr-x  (owner: full, others: read+execute)
  chmod 644 = rw-r--r--  (owner: read+write, others: read only)
  chmod 600 = rw-------  (owner only - use for .env, keys)
  chmod 700 = rwx------  (owner only + execute - use for scripts, .ssh/)
```

### Permission Patterns for Deployment
```bash
# Web app directory (Node.js, Next.js)
sudo chown -R deploy:deploy /var/www/myapp
sudo chmod -R 755 /var/www/myapp
chmod 600 /var/www/myapp/.env        # Secrets: owner-only

# WordPress (needs www-data for PHP-FPM writes)
sudo chown -R www-data:www-data /var/www/wordpress
sudo find /var/www/wordpress -type d -exec chmod 755 {} \;
sudo find /var/www/wordpress -type f -exec chmod 644 {} \;

# SSH directory (MUST be strict or SSH refuses to work)
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
chmod 600 ~/.ssh/id_rsa              # Private key
chmod 644 ~/.ssh/id_rsa.pub          # Public key

# Upload directory (app writes, web server reads)
sudo chown -R deploy:www-data /var/www/myapp/uploads
sudo chmod -R 775 /var/www/myapp/uploads
```

### Why Permission Errors Happen
```
SCENARIO 1: "403 Forbidden" from Nginx
  Nginx runs as www-data.
  Your files are owned by deploy:deploy with 750.
  www-data is "others" -> no read permission.
  FIX: chmod 755 (give others read+execute)

SCENARIO 2: "EACCES" in Node.js trying to write logs
  App runs as deploy user.
  Log directory owned by root.
  FIX: sudo chown deploy:deploy /var/www/myapp/logs

SCENARIO 3: npm install fails with EACCES
  You ran "sudo npm install" once -> node_modules owned by root.
  FIX: sudo rm -rf node_modules && npm install (without sudo)

GOLDEN RULE: Never run npm/node with sudo.
```

---

## Process Management

### Understanding Processes
```bash
# View all processes
ps aux                             # Full list
ps aux | grep nginx                # Find specific process
ps aux | grep node

# Interactive process viewer
top                                # Basic (q to quit)
htop                               # Better (install: apt install htop)

# Process tree (see parent-child relationships)
pstree

# Find what's using a port
sudo lsof -i :3000                 # What process is on port 3000?
sudo ss -tlnp                      # All listening ports + processes

# Kill a process
kill PID                           # Graceful stop (SIGTERM)
kill -9 PID                        # Force kill (SIGKILL) - last resort
killall node                       # Kill all node processes
```

### Process States
```
RUNNING (R)  - Actively using CPU
SLEEPING (S) - Waiting for something (normal for servers)
ZOMBIE (Z)   - Finished but parent hasn't cleaned up (bad if many)
STOPPED (T)  - Paused (Ctrl+Z)

A healthy server mostly shows SLEEPING processes.
High CPU = too many RUNNING processes.
```

### Background vs Foreground
```bash
# Run in foreground (blocks terminal)
node app.js                        # Ctrl+C kills it, closing terminal kills it

# Run in background (basic - still dies when terminal closes)
node app.js &

# Run and survive terminal close
nohup node app.js &                # Output goes to nohup.out

# PROPER WAY: Use PM2 or systemd (they handle everything)
pm2 start app.js                   # PM2: auto-restart, logging, startup
```

---

## systemd - The Service Manager

### What systemd Does
```
systemd is the "process manager of the OS."
It starts services on boot, restarts them if they crash,
manages dependencies (start MySQL before app), and handles logging.

EVERYTHING on a modern Linux server is a systemd service:
  - Nginx: nginx.service
  - MySQL: mysql.service
  - SSH:   sshd.service
  - PHP:   php8.2-fpm.service
  - Your app: myapp.service (you create this)
```

### Essential systemd Commands
```bash
# Status
sudo systemctl status nginx        # Is it running? Any errors?
sudo systemctl status mysql
sudo systemctl status php8.2-fpm

# Start / Stop / Restart
sudo systemctl start nginx
sudo systemctl stop nginx
sudo systemctl restart nginx       # Stop then start
sudo systemctl reload nginx        # Reload config without downtime

# Enable / Disable (auto-start on boot)
sudo systemctl enable nginx        # Start on boot
sudo systemctl disable nginx       # Don't start on boot

# List all services
sudo systemctl list-units --type=service
sudo systemctl list-units --type=service --state=running   # Only running

# Check if a service is enabled
sudo systemctl is-enabled nginx
```

### Creating a Custom Service
```bash
sudo nano /etc/systemd/system/myapp.service
```
```ini
[Unit]
Description=My Node.js Application
After=network.target              # Start after network is ready
# After=mysql.service             # Uncomment if app needs MySQL

[Service]
Type=simple
User=deploy                       # Run as deploy, NOT root
Group=deploy
WorkingDirectory=/var/www/myapp
ExecStart=/home/deploy/.nvm/versions/node/v20.11.0/bin/node app.js
Restart=on-failure                 # Restart if it crashes
RestartSec=10                      # Wait 10 seconds before restart
StandardOutput=journal             # Logs go to journalctl
StandardError=journal
Environment=NODE_ENV=production
Environment=PORT=3000
# EnvironmentFile=/var/www/myapp/.env  # Load .env file

[Install]
WantedBy=multi-user.target        # Start in normal multi-user mode
```
```bash
sudo systemctl daemon-reload       # Reload service files
sudo systemctl start myapp         # Start your service
sudo systemctl enable myapp        # Auto-start on boot
sudo systemctl status myapp        # Verify

# View logs
journalctl -u myapp -f             # Follow logs in real-time
journalctl -u myapp --since "1 hour ago"
journalctl -u myapp --since today
```

### PM2 vs systemd for Node.js
```
PM2:
  + Cluster mode (use all CPU cores)
  + Built-in log management
  + Easy ecosystem config
  + pm2 monit (built-in monitor)
  + Simpler syntax
  - Extra dependency (npm package)

systemd:
  + Built into every Linux system
  + No extra software
  + Standardized (same as nginx, mysql)
  + Better for non-Node.js apps
  - No cluster mode
  - Log management via journalctl

RECOMMENDATION:
  Node.js apps -> PM2 (better features)
  PHP/Python/other -> systemd
  If you want one tool for everything -> systemd
```

---

## Package Management (apt)

### How apt Works
```
apt = Advanced Package Tool
It downloads and installs software from Ubuntu's repositories.
Like npm for the operating system.
```

### Essential apt Commands
```bash
# Update package list (always do first)
sudo apt update

# Upgrade all installed packages
sudo apt upgrade -y

# Install a package
sudo apt install nginx -y
sudo apt install htop curl git -y          # Multiple packages

# Remove a package
sudo apt remove nginx                      # Remove but keep config
sudo apt purge nginx                       # Remove + delete config
sudo apt autoremove                        # Remove unused dependencies

# Search for packages
apt search redis
apt show nginx                             # Package details

# List installed packages
apt list --installed
apt list --installed | grep php            # Filter

# Fix broken installations
sudo apt --fix-broken install
```

### Adding External Repositories (PPAs)
```bash
# Some software isn't in default Ubuntu repos
# Example: Adding PHP 8.3 when Ubuntu only has 8.1

sudo add-apt-repository ppa:ondrej/php     # Add PHP repo
sudo apt update                            # Refresh package list
sudo apt install php8.3-fpm                # Now available

# Example: Adding Node.js repo (alternative to nvm)
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install nodejs
# NOTE: nvm is still recommended over this method
```

### Keeping System Updated
```bash
# Manual update (do monthly at minimum)
sudo apt update && sudo apt upgrade -y

# Automatic security updates (set up once)
sudo apt install unattended-upgrades
sudo dpkg-reconfigure -plow unattended-upgrades

# Check if reboot needed after updates
cat /var/run/reboot-required         # If exists, reboot needed
sudo reboot                          # Reboot (causes downtime!)
```

---

## Disk & Storage

### Disk Commands
```bash
# Disk usage overview
df -h                              # How full are disks?
#   Filesystem      Size  Used Avail Use% Mounted on
#   /dev/vda1        50G   12G   36G  25% /

# Directory sizes
du -sh /var/www/*                  # Size of each app
du -sh /var/log/*                  # Size of each log dir
du -sh /home/deploy/*              # Size of user files

# Find large files
sudo find / -type f -size +100M -exec ls -lh {} \;

# Find what's eating disk space
sudo du -h / --max-depth=1 | sort -hr | head -20
```

### Common Disk Space Problems
```bash
# Problem: Disk full (100%)
# Check what's using space:
sudo du -h /var/log --max-depth=1 | sort -hr

# Fix 1: Clear old logs
sudo journalctl --vacuum-time=3d              # Keep only 3 days of journal
sudo truncate -s 0 /var/log/nginx/access.log  # Empty Nginx access log
pm2 flush                                      # Clear PM2 logs

# Fix 2: Remove old packages
sudo apt autoremove
sudo apt clean                                 # Clear apt cache

# Fix 3: Clear old Docker data (if using Docker)
docker system prune -a

# Fix 4: Find and remove old backups
ls -lh /home/deploy/backups/
# Delete old ones manually

# Prevention: Set up log rotation
# Nginx already has logrotate configured at /etc/logrotate.d/nginx
# For custom apps, create /etc/logrotate.d/myapp:
```
```
/var/www/myapp/logs/*.log {
    daily
    rotate 7
    compress
    delaycompress
    missingok
    notifempty
    create 640 deploy deploy
}
```

---

## Memory (RAM) & Swap

### Understanding Memory
```bash
free -h
#               total    used    free    shared  buff/cache   available
# Mem:          1.9Gi   800Mi   200Mi    20Mi      960Mi       1.0Gi
# Swap:         2.0Gi    50Mi   1.9Gi

# IMPORTANT: Look at "available", not "free"
# Linux uses free RAM for disk cache (buff/cache) - this is GOOD
# "available" = truly available for new processes

# If "available" is < 100MB -> you need more RAM or swap
```

### Swap Management
```bash
# Check current swap
swapon --show
free -h

# Create swap (if none exists)
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile

# Make permanent (survives reboot)
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab

# Adjust swappiness (how eagerly Linux uses swap)
cat /proc/sys/vm/swappiness          # Default: 60
sudo sysctl vm.swappiness=10         # Lower = prefer RAM (good for servers)
echo 'vm.swappiness=10' | sudo tee -a /etc/sysctl.conf   # Permanent

# SWAP SIZING GUIDE:
#   1GB RAM  -> 2GB swap
#   2GB RAM  -> 2GB swap
#   4GB RAM  -> 2GB swap
#   8GB+ RAM -> 1GB swap (or none)
```

### Diagnosing Memory Issues
```bash
# What's using the most RAM?
ps aux --sort=-%mem | head -10

# Watch memory in real-time
watch -n 2 free -h

# OOM (Out of Memory) killer - did Linux kill your app?
dmesg | grep -i "killed process"
sudo journalctl | grep -i "out of memory"

# If OOM keeps killing your app:
# Option 1: Add more swap
# Option 2: Upgrade server (more RAM)
# Option 3: Optimize app (fix memory leaks)
# Option 4: Limit PM2 memory: --max-memory-restart 300M
```

---

## Cron Jobs (Scheduled Tasks)

### Cron Syntax
```
* * * * *  command
│ │ │ │ │
│ │ │ │ └── Day of week (0-7, 0 and 7 = Sunday)
│ │ │ └──── Month (1-12)
│ │ └────── Day of month (1-31)
│ └──────── Hour (0-23)
└────────── Minute (0-59)

Examples:
  0 * * * *      Every hour (at minute 0)
  0 2 * * *      Every day at 2:00 AM
  0 2 * * 0      Every Sunday at 2:00 AM
  */5 * * * *    Every 5 minutes
  0 0 1 * *      First day of every month at midnight
```

### Managing Cron Jobs
```bash
# Edit cron jobs for current user
crontab -e

# List cron jobs
crontab -l

# Example cron jobs:
# Daily database backup at 2 AM
0 2 * * * /home/deploy/backup.sh >> /home/deploy/backup.log 2>&1

# Clean old logs every Sunday at 3 AM
0 3 * * 0 find /home/deploy/backups -mtime +30 -delete

# Health check every 5 minutes
*/5 * * * * curl -s http://localhost:3000/health > /dev/null || pm2 restart myapp

# SSL renewal check (twice daily, certbot recommends)
0 0,12 * * * certbot renew --quiet
```

### Cron Debugging
```bash
# Check if cron is running
sudo systemctl status cron

# View cron logs
sudo grep CRON /var/log/syslog | tail -20

# Common issues:
# 1. PATH not set in cron (cron has minimal PATH)
#    Fix: Use full paths (/usr/bin/node instead of node)
#
# 2. Environment variables not available
#    Fix: Source them in script or set in crontab
#
# 3. Permission denied
#    Fix: chmod +x your-script.sh
```

---

## Environment Variables

### How They Work
```bash
# Set for current session
export NODE_ENV=production
export PORT=3000

# Check value
echo $NODE_ENV
env                                # List all environment variables
env | grep NODE                    # Filter

# Set permanently for a user (add to .bashrc)
echo 'export NODE_ENV=production' >> ~/.bashrc
source ~/.bashrc                   # Reload

# Set for a single command
NODE_ENV=production node app.js
PORT=3000 npm start
```

### .env Files
```bash
# .env file format (no export, no spaces around =)
DATABASE_URL=postgresql://user:pass@localhost:5432/mydb
NODE_ENV=production
PORT=3000
JWT_SECRET=your-secret-here
REDIS_URL=redis://localhost:6379

# Node.js reads via: dotenv package or built-in (Node 20.6+)
# Your app code: process.env.DATABASE_URL

# Security:
chmod 600 .env                     # Only owner can read
# Add to .gitignore ALWAYS
```

### Environment Variables in systemd
```ini
# In service file:
Environment=NODE_ENV=production
Environment=PORT=3000

# Or load from file:
EnvironmentFile=/var/www/myapp/.env
```

### Environment Variables in PM2
```javascript
// ecosystem.config.js
module.exports = {
  apps: [{
    name: 'myapp',
    script: 'app.js',
    env: {
      NODE_ENV: 'production',
      PORT: 3000
    }
  }]
};
```

---

## Networking at OS Level

### Network Commands
```bash
# Your server's IP addresses
ip addr show                       # All network interfaces
hostname -I                        # Just IP addresses

# Check connectivity
ping google.com                    # Can server reach internet?
ping 10.0.0.5                     # Can server reach another server?

# DNS resolution
dig example.com +short
nslookup example.com
cat /etc/resolv.conf               # DNS servers configured

# Active connections
sudo ss -tlnp                     # Listening TCP ports
sudo ss -tnp                      # Active TCP connections
sudo netstat -tlnp                 # Alternative

# Route table
ip route show                      # How traffic is routed

# Hosts file (local DNS override)
sudo nano /etc/hosts
# 127.0.0.1  myapp.local           # Useful for testing
```

### Private vs Public IPs
```
PUBLIC IP:   Accessible from internet (what users connect to)
             Example: 143.198.50.100
             Check: curl ifconfig.me

PRIVATE IP:  Only accessible within same network/VPC
             Example: 10.0.0.5, 172.16.x.x, 192.168.x.x
             Used for: Server-to-server communication (app -> database)

WHY IT MATTERS:
  - Database should listen on private IP only (not public!)
  - Load balancer -> app servers use private IPs
  - You SSH via public IP
```

---

## System Monitoring

### Quick Health Check Commands
```bash
# One-liner system overview
echo "=== UPTIME ===" && uptime && \
echo "=== MEMORY ===" && free -h && \
echo "=== DISK ===" && df -h / && \
echo "=== CPU LOAD ===" && cat /proc/loadavg && \
echo "=== LISTENING PORTS ===" && sudo ss -tlnp

# Load average explained (from uptime output)
# load average: 0.50, 0.75, 1.00
#               1min  5min  15min
# Compare to number of CPUs (nproc)
# If load avg > nproc consistently -> CPU bottleneck
# Example: 2 CPU server with load avg 3.0 = overloaded

# Disk I/O monitoring
iostat -x 1 5                      # Install: apt install sysstat

# Network traffic
vnstat                             # Install: apt install vnstat
iftop                              # Install: apt install iftop (live traffic)
```

### Important Log Files
```bash
# System logs
sudo tail -f /var/log/syslog       # General system events
sudo journalctl -f                 # systemd journal (all services)

# Auth/Security logs
sudo tail -f /var/log/auth.log     # SSH attempts, sudo usage

# Application logs
pm2 logs                           # All PM2 apps
pm2 logs myapp --lines 100         # Specific app, last 100 lines
journalctl -u myapp -f             # If using systemd

# Web server logs
sudo tail -f /var/log/nginx/access.log
sudo tail -f /var/log/nginx/error.log

# Database logs
sudo tail -f /var/log/mysql/error.log
sudo tail -f /var/log/postgresql/postgresql-16-main.log
```

---

## Essential OS Troubleshooting

### Server Won't Start / Can't SSH
```
1. Check cloud provider dashboard - is instance running?
2. Check Security Group/Firewall - is port 22 open?
3. Try: ssh -v deploy@server-ip (verbose output)
4. If locked out:
   - AWS: Use EC2 Serial Console or Instance Connect
   - DO: Use Recovery Console from dashboard
   - Fix sshd_config and restart sshd
```

### High CPU Usage
```bash
top                                # Find process using CPU (sort by %CPU)
# press 'P' to sort by CPU in top

# If it's your app: likely infinite loop or expensive operation
# If it's MySQL: slow queries, check slow query log
# If it's system: check for runaway cron jobs
```

### High Memory Usage
```bash
ps aux --sort=-%mem | head -5      # Top memory consumers
free -h                            # Overall memory

# If PM2/Node: possible memory leak
pm2 restart myapp                  # Quick fix
# Long-term: investigate with --max-memory-restart
```

### Server Suddenly Slow
```bash
# Check in this order:
uptime                             # Load average high?
free -h                            # RAM exhausted?
df -h                              # Disk full?
sudo ss -tnp | wc -l              # Too many connections?
dmesg | tail -20                   # Kernel errors?
```

---

## OS Hardening Summary

```
1. Keep system updated                sudo apt update && sudo apt upgrade
2. Use non-root user for apps         adduser deploy
3. SSH key auth only                   PasswordAuthentication no
4. Firewall on                         ufw enable
5. Fail2ban running                    blocks brute force
6. Auto security updates               unattended-upgrades
7. Proper file permissions             chmod 600 for secrets
8. Log rotation configured             prevent disk fill
9. Swap configured                     prevent OOM crashes
10. Timezone set to UTC                 consistent log timestamps
```
