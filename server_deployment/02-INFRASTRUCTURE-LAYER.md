---
layout: default
title: Infrastructure Layer
---

# 19 - Infrastructure Layer (Layer 1) - Deep Dive

> The foundation: choosing, provisioning, and configuring your virtual server.

---

## What is the Infrastructure Layer?

```
Before you deploy anything, you need a machine to deploy TO.

Infrastructure = the virtual (or physical) server + its resources:
  - CPU cores
  - RAM
  - Disk (SSD)
  - Operating System
  - Network interface (public IP)
  - Geographic location (region)

This is Layer 1 because EVERYTHING else depends on it.
No server = no deployment.
```

---

## Choosing a Provider

### AWS EC2 vs DigitalOcean vs Others

```
+-------------------+------------------+------------------+------------------+
| Factor            | AWS EC2          | DigitalOcean     | Others (Vultr,   |
|                   |                  |                  | Linode, Hetzner) |
+-------------------+------------------+------------------+------------------+
| Complexity        | High             | Low              | Low-Medium       |
| Pricing           | Complex (hourly, | Simple ($6/mo,   | Simple           |
|                   | data transfer,   | $12/mo, etc.)    |                  |
|                   | hidden charges)  |                  |                  |
| Free tier         | 1yr free t2.micro| None (but $200   | Varies           |
|                   |                  | credit for new)  |                  |
| Scaling           | Excellent        | Good             | Good             |
| Extra services    | 200+ services    | ~20 services     | Fewer            |
| Best for          | Enterprise,      | Simple apps,     | Budget servers,  |
|                   | complex setups,  | startups, solo   | high performance |
|                   | compliance       | developers       | per dollar       |
+-------------------+------------------+------------------+------------------+

RECOMMENDATION:
  Learning / Side project  --> DigitalOcean (simple, predictable pricing)
  Company / Enterprise     --> AWS (ecosystem, compliance, scaling)
  Budget-conscious         --> Hetzner / Vultr (cheapest per resource)
```

---

## Choosing Server Size

### How Much RAM Do You Need?

```
+------------------------+----------+------+-----------------------------+
| What You're Running    | Min RAM  | CPU  | Notes                       |
+------------------------+----------+------+-----------------------------+
| Node.js API (small)    | 1 GB     | 1    | Add swap                    |
| Node.js API (moderate) | 2 GB     | 1-2  | Comfortable                 |
| React (static files)   | 512 MB   | 1    | Nginx serves files, minimal |
| Next.js (SSR)          | 2 GB     | 1-2  | Build needs more RAM        |
| WordPress (small)      | 2 GB     | 1    | PHP + MySQL = RAM hungry    |
| WordPress (medium)     | 4 GB     | 2    | With caching plugins        |
| PHP app + MySQL        | 2 GB     | 1-2  | Similar to WordPress        |
| Multiple apps          | 4 GB+    | 2+   | Each app adds ~500MB-1GB    |
+------------------------+----------+------+-----------------------------+

RULE OF THUMB:
  1 GB  = Tight. Single small app only. Must have swap.
  2 GB  = Comfortable for one app with database.
  4 GB  = Can run multiple apps or one busy app.
  8 GB+ = Production workloads with room to grow.
```

### Monthly Cost Comparison (as of 2024)
```
1 GB / 1 CPU:
  DigitalOcean:  $6/mo
  AWS t3.micro:  ~$8/mo (free tier first year)
  Vultr:         $6/mo
  Hetzner:       ~$4/mo

2 GB / 1 CPU:
  DigitalOcean:  $12/mo
  AWS t3.small:  ~$17/mo
  Vultr:         $12/mo
  Hetzner:       ~$5/mo

4 GB / 2 CPU:
  DigitalOcean:  $24/mo
  AWS t3.medium: ~$34/mo
  Vultr:         $24/mo
  Hetzner:       ~$7/mo
```

---

## Choosing an Operating System

```
+-------------------+----------------------------+----------------------------+
| OS                | Pros                       | When to Use                |
+-------------------+----------------------------+----------------------------+
| Ubuntu 22.04 LTS  | Most tutorials, largest    | DEFAULT CHOICE             |
|                   | community, easy to learn   | Best for beginners         |
+-------------------+----------------------------+----------------------------+
| Ubuntu 24.04 LTS  | Latest LTS, newest         | If starting fresh in 2024+ |
|                   | packages                   |                            |
+-------------------+----------------------------+----------------------------+
| Debian 12         | Stable, lightweight,       | If you want stability over |
|                   | minimal bloat              | newest packages            |
+-------------------+----------------------------+----------------------------+
| Amazon Linux 2023 | Optimized for AWS,         | Only on AWS, when using    |
|                   | integrated with AWS tools  | AWS-specific features      |
+-------------------+----------------------------+----------------------------+

RULE: Always use LTS (Long Term Support) versions.
  Ubuntu 22.04 LTS = supported until 2027
  Ubuntu 24.04 LTS = supported until 2029

NEVER use non-LTS (23.04, 23.10) on servers -- they expire in 9 months.
```

---

## Choosing a Region

```
Pick the region CLOSEST to your users.

US East Coast users     --> us-east-1 (Virginia) / NYC1
US West Coast users     --> us-west-2 (Oregon) / SFO1
European users          --> eu-west-1 (Ireland) / AMS3 / FRA1
Indian users            --> ap-south-1 (Mumbai) / BLR1
Asian users             --> ap-southeast-1 (Singapore) / SGP1

Multiple regions of users? Use a CDN (Cloudflare) for static content
and choose the region where most of your users are for the server.

Test latency: Use cloudping.info or ping from different locations.
```

---

## Provisioning a Server

### DigitalOcean (Droplet)

```
Step 1: Create Droplet
  - Choose region (closest to users)
  - Choose image: Ubuntu 22.04 or 24.04 LTS
  - Choose size: Start with $12/mo (2GB / 1 CPU)
  - Choose authentication: SSH Key (NOT password)
  - Add your SSH public key from local machine

Step 2: Note the IP
  - After creation, you get an IP like: 143.198.50.100

Step 3: First SSH
  ssh root@143.198.50.100
```

### AWS EC2

```
Step 1: Launch Instance
  - Choose AMI: Ubuntu 22.04 LTS
  - Choose type: t3.small (2GB) or t3.micro (1GB, free tier)
  - Configure security group:
    - SSH (22) from your IP
    - HTTP (80) from anywhere
    - HTTPS (443) from anywhere
  - Create or select key pair (.pem file)

Step 2: Note the public IP (or Elastic IP)
  - Elastic IP = static IP that doesn't change on restart
  - Without it, IP changes every time you stop/start

Step 3: First SSH
  ssh -i ~/.ssh/mykey.pem ubuntu@ec2-xx-xx-xx-xx.compute-1.amazonaws.com
```

### AWS Elastic IP (Important!)
```bash
# Without Elastic IP:
# Stop instance -> Start instance -> NEW IP ADDRESS
# Your DNS breaks, SSH config breaks, everything breaks

# With Elastic IP:
# IP stays the same forever (until you release it)
# Free when attached to a running instance
# $3.65/mo if NOT attached (AWS charges for unused IPs)

# ALWAYS attach an Elastic IP to production servers on AWS
```

---

## First Server Setup (After Provisioning)

### The First 10 Minutes Script
```bash
# Run these commands right after SSH-ing into a new server

# 1. Update everything
sudo apt update && sudo apt upgrade -y

# 2. Create deploy user (don't run apps as root!)
adduser deploy
usermod -aG sudo deploy

# 3. Copy SSH key to deploy user
mkdir -p /home/deploy/.ssh
cp ~/.ssh/authorized_keys /home/deploy/.ssh/
chown -R deploy:deploy /home/deploy/.ssh
chmod 700 /home/deploy/.ssh
chmod 600 /home/deploy/.ssh/authorized_keys

# 4. Test deploy user login (in a NEW terminal, don't close this one!)
# ssh deploy@your-server-ip

# 5. Set timezone
sudo timedatectl set-timezone UTC        # UTC is standard for servers
# Or your local: sudo timedatectl set-timezone America/New_York

# 6. Set hostname
sudo hostnamectl set-hostname myapp-prod

# 7. Install essential packages
sudo apt install -y \
    curl \
    wget \
    git \
    vim \
    htop \
    unzip \
    build-essential \
    software-properties-common

# 8. Setup swap (for 1-2GB servers)
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab

# 9. Verify swap
free -h
# Should show swap: 2.0G

# 10. Setup firewall (see Security Layer for full details)
sudo ufw allow OpenSSH
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable
```

---

## Understanding Server Resources

### CPU
```bash
# How many cores?
nproc                    # Number of CPU cores
lscpu                   # Detailed CPU info

# Current CPU usage
top                      # Live view (q to quit)
htop                     # Better live view (install: apt install htop)
uptime                   # Load averages (1min, 5min, 15min)

# Load average interpretation:
# Load 1.0 on 1-core = 100% utilized
# Load 2.0 on 2-core = 100% utilized
# Load 4.0 on 2-core = 200% utilized (overloaded!)
# RULE: Load should be < number of cores
```

### RAM
```bash
# Check RAM usage
free -h

# Output explanation:
#               total    used    free    shared  buff/cache  available
# Mem:          2.0G     600M    200M    50M     1.2G        1.1G

# DON'T PANIC about low "free" -- Linux uses free RAM for disk cache
# Look at "available" -- that's what's actually available for new apps
# "available" < 200MB = you're running low

# Find what's using RAM
ps aux --sort=-%mem | head -10    # Top 10 memory consumers
```

### Disk
```bash
# Check disk usage
df -h

# Output that matters:
# Filesystem      Size  Used  Avail  Use%  Mounted on
# /dev/vda1       50G   15G   33G    31%   /

# RULE: Keep below 80%. Above 90% = emergency

# Find large files/directories
du -sh /var/*                    # Size of each directory in /var
du -sh /var/log/*                # Logs can grow huge
sudo find / -type f -size +100M  # Files larger than 100MB

# Clean up
sudo apt autoremove -y           # Remove unused packages
sudo apt clean                   # Clear apt cache
sudo journalctl --vacuum-size=100M  # Trim system logs to 100MB
```

### Network
```bash
# Check network interfaces
ip addr show

# Check internet connectivity
ping -c 3 google.com

# Check DNS resolution
dig example.com +short

# Bandwidth usage (install: apt install iftop)
sudo iftop
```

---

## Swap (Virtual Memory)

### What is Swap?
```
Swap = disk space used as emergency RAM.
When RAM is full, Linux moves inactive data to swap.

It's MUCH slower than RAM (disk vs memory speed).
But it prevents your server from crashing when RAM runs out.

Without swap on a 1GB server:
  App uses 1.1GB -> OOM Killer kills your app -> Site goes down

With swap on a 1GB server:
  App uses 1.1GB -> 0.1GB moves to swap -> App continues (slower but alive)
```

### Swap Setup
```bash
# Check current swap
sudo swapon --show
free -h

# Create swap file
sudo fallocate -l 2G /swapfile   # Size: 1x-2x your RAM
sudo chmod 600 /swapfile          # Security: only root can read
sudo mkswap /swapfile             # Format as swap
sudo swapon /swapfile             # Enable swap

# Make permanent (survives reboot)
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab

# Tune swappiness (how aggressively Linux uses swap)
# Default: 60 (too aggressive for servers)
# Recommended: 10 (only swap when really needed)
echo 'vm.swappiness=10' | sudo tee -a /etc/sysctl.conf
sudo sysctl -p

# Verify
free -h
cat /proc/sys/vm/swappiness       # Should show 10
```

### Swap Size Guide
```
Server RAM    Swap Size
512 MB        1 GB
1 GB          2 GB
2 GB          2 GB
4 GB          2-4 GB
8 GB+         2-4 GB (swap becomes less important)
```

---

## Server Monitoring Basics

### Quick Health Check Commands
```bash
# One-liner: CPU, RAM, Disk in one view
echo "=== CPU ===" && uptime && echo "=== RAM ===" && free -h && echo "=== DISK ===" && df -h /

# System load
uptime
# 12:00:00 up 30 days, load average: 0.50, 0.75, 0.60
#                                     1min  5min  15min

# Process overview
htop                    # Interactive (q to quit)

# Disk I/O
iostat -x 1 3           # 3 samples, 1 second apart (install: apt install sysstat)

# Network connections
ss -tlnp                # Listening TCP ports
ss -s                   # Connection summary
```

### Alerts: When to Worry
```
+------------------+-------------+------------------+------------------+
| Metric           | Normal      | Warning          | Critical         |
+------------------+-------------+------------------+------------------+
| CPU Load (1 min) | < cores     | 1-2x cores       | > 2x cores       |
| RAM Available    | > 500 MB    | 200-500 MB       | < 200 MB         |
| Disk Usage       | < 70%       | 70-85%           | > 85%            |
| Swap Usage       | < 100 MB    | 100 MB - 1 GB    | > 1 GB           |
+------------------+-------------+------------------+------------------+
```

---

## Storage Management

### Application Directories
```bash
# Standard Linux layout for web apps:
/var/www/                          # Web applications root
/var/www/myapp/                    # Your app
/var/www/wordpress/                # WordPress

# Logs
/var/log/nginx/                    # Nginx logs
/var/log/mysql/                    # MySQL logs
/var/log/pm2/                      # PM2 logs (if configured)

# Backups (create this)
/backups/                          # Database and file backups
sudo mkdir -p /backups
sudo chown deploy:deploy /backups
```

### Preventing Disk Full
```bash
# Common disk hogs:
# 1. Logs (especially Nginx access logs on busy sites)
# 2. Old deployments / node_modules
# 3. Database files
# 4. Package manager cache

# Log rotation (usually configured, but check)
ls /etc/logrotate.d/               # Should have nginx, mysql entries

# Manual log cleanup
sudo truncate -s 0 /var/log/nginx/access.log   # Empty a log file
sudo journalctl --vacuum-size=100M              # Trim system journal

# Clear old package cache
sudo apt clean
sudo apt autoremove -y
```

---

## Server Hardening (Beyond Security Layer)

### Kernel Tuning for Web Servers
```bash
# /etc/sysctl.conf additions for web servers

# Increase max open files
fs.file-max = 65535

# Network performance
net.core.somaxconn = 65535                  # Max connection queue
net.ipv4.tcp_max_syn_backlog = 65535        # SYN queue size
net.ipv4.ip_local_port_range = 1024 65535   # More ephemeral ports

# Reuse connections
net.ipv4.tcp_tw_reuse = 1

# Keepalive
net.ipv4.tcp_keepalive_time = 600
net.ipv4.tcp_keepalive_intvl = 60
net.ipv4.tcp_keepalive_probes = 3

# Apply
sudo sysctl -p
```

### Ulimits (Open File Limits)
```bash
# Check current limits
ulimit -n                          # Open files (default: 1024, too low)

# Increase for deploy user
sudo nano /etc/security/limits.conf
# Add:
deploy soft nofile 65535
deploy hard nofile 65535

# Also for www-data (PHP-FPM/Nginx)
www-data soft nofile 65535
www-data hard nofile 65535

# Verify after re-login:
ulimit -n
```

---

## Quick Reference

```bash
# --- SERVER INFO ---
lsb_release -a                     # OS version
uname -r                           # Kernel version
nproc                              # CPU cores
free -h                            # RAM usage
df -h                              # Disk usage
uptime                             # Uptime and load

# --- RESOURCE MONITORING ---
htop                               # Interactive process viewer
iostat -x 1                        # Disk I/O stats
ss -tlnp                           # Listening ports
iftop                              # Network bandwidth

# --- DISK CLEANUP ---
sudo apt autoremove -y             # Remove unused packages
sudo apt clean                     # Clear package cache
sudo journalctl --vacuum-size=100M # Trim logs
du -sh /var/* | sort -h            # Find large directories

# --- SWAP ---
free -h                            # Check swap usage
sudo swapon --show                 # Swap file details
cat /proc/sys/vm/swappiness        # Swap aggressiveness
```
