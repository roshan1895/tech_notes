---
layout: default
title: Manual Deployment
---

# 03 - Manual Deployment Mastery

## Why Master Manual First
- You understand EVERY layer (no magic black boxes)
- You can debug anything because you built it yourself
- Automation is just scripting what you already do manually
- You can't automate what you don't understand

---

## Universal Manual Deployment Flow (All Stacks)

```
Phase 1: SERVER SETUP (One time)
  1. Provision server
  2. Secure server (SSH, firewall, user)
  3. Install runtime (Node/PHP)
  4. Install web server (Nginx)
  5. Configure web server
  6. Setup SSL

Phase 2: APP DEPLOYMENT (Every deploy)
  1. SSH into server
  2. Pull latest code (git pull)
  3. Install dependencies
  4. Build (if needed)
  5. Update env vars (if changed)
  6. Run migrations (if any)
  7. Restart app
  8. Verify

Phase 3: POST-DEPLOY (Verify)
  1. Check app is running
  2. Check logs for errors
  3. Test key endpoints
  4. Monitor for 5-10 minutes
```

---

## Phase 1: Complete Server Setup (Step by Step)

### Step 1: Provision Server

**AWS EC2:**
```
1. Launch Instance
2. Choose AMI: Ubuntu Server 22.04 LTS
3. Instance type: t3.micro (free tier) or t3.small
4. Key pair: Create new or use existing
5. Security Group:
   - SSH (22) from My IP
   - HTTP (80) from Anywhere
   - HTTPS (443) from Anywhere
6. Storage: 20GB+ gp3 SSD
7. Launch
```

**DigitalOcean:**
```
1. Create Droplet
2. Ubuntu 22.04 LTS
3. Plan: Basic, Regular, $6/mo (1GB) or $12/mo (2GB)
4. Region: Closest to users
5. Auth: SSH Key (add your public key)
6. Create
```

### Step 2: Initial Server Security
```bash
# Connect
ssh root@your-server-ip           # DO (root access by default)
ssh ubuntu@your-server-ip -i key.pem  # AWS (ubuntu user + key file)

# Update system
sudo apt update && sudo apt upgrade -y

# Create deploy user
adduser deploy                     # Set a strong password
usermod -aG sudo deploy

# Copy SSH key to deploy user
# Method 1: From root
rsync --archive --chown=deploy:deploy ~/.ssh /home/deploy

# Method 2: From your local machine
ssh-copy-id deploy@your-server-ip

# Test login as deploy user (new terminal)
ssh deploy@your-server-ip

# Disable root login and password auth
sudo nano /etc/ssh/sshd_config
```
```
# Change these lines:
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
```
```bash
sudo systemctl restart sshd

# Setup firewall
sudo ufw allow OpenSSH
sudo ufw enable
sudo ufw status

# Install fail2ban
sudo apt install fail2ban -y
sudo systemctl enable fail2ban

# Setup swap (for 1-2GB servers)
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab

# Set timezone
sudo timedatectl set-timezone UTC
```

### Step 3: Install Runtime

**Node.js (via nvm):**
```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.0/install.sh | bash
source ~/.bashrc
nvm install 20
nvm alias default 20
node -v && npm -v
```

**PHP:**
```bash
sudo apt install php8.2-fpm php8.2-mysql php8.2-xml php8.2-curl \
  php8.2-mbstring php8.2-zip php8.2-gd php8.2-intl -y
php -v
```

**Both (if running WordPress + Node on same server):**
```bash
# Install both - they don't conflict
```

### Step 4: Install & Configure Nginx
```bash
sudo apt install nginx -y
sudo ufw allow 'Nginx Full'
sudo systemctl enable nginx

# Verify
sudo systemctl status nginx
curl localhost   # Should show Nginx welcome page

# Remove default config
sudo rm /etc/nginx/sites-enabled/default
```

### Step 5: Configure Nginx for Your App

See `04-STACK-SPECIFIC-GUIDES.md` for specific Nginx configs.

### Step 6: Setup SSL
```bash
sudo apt install certbot python3-certbot-nginx -y
sudo certbot --nginx -d example.com -d www.example.com

# Test auto-renewal
sudo certbot renew --dry-run
```

---

## Phase 2: Deploying Code (Every Time)

### Method 1: Git Pull (Recommended)
```bash
# First time: Clone
cd /var/www
sudo mkdir myapp
sudo chown deploy:deploy myapp
git clone https://github.com/you/repo.git myapp
cd myapp

# Every deployment after:
ssh deploy@server
cd /var/www/myapp
git pull origin main

# Install dependencies + build + restart
npm install --production
npm run build
pm2 restart myapp
```

### Method 2: Rsync (Good for quick deploys)
```bash
# From your LOCAL machine
rsync -avz --exclude='node_modules' --exclude='.git' --exclude='.env' \
  ./ deploy@server:/var/www/myapp/

# Then SSH in and restart
ssh deploy@server
cd /var/www/myapp
npm install --production
npm run build
pm2 restart myapp
```

### Method 3: SCP (Simple file transfer)
```bash
# Upload a specific file or directory
scp -r ./build deploy@server:/var/www/myapp/build
```

### Git Pull vs Rsync Decision
```
Git Pull:
  + Version tracked
  + Can rollback easily (git checkout)
  + Team friendly
  - Needs git installed on server
  - Repo access needed on server

Rsync:
  + Fast partial transfers (only changed files)
  + No git needed on server
  + Works with build artifacts
  - No version history on server
  - Harder to rollback

RECOMMENDATION: Use Git Pull for backend/full-stack apps
               Use Rsync for static frontend builds
```

---

## Phase 3: Post-Deploy Verification

### Quick Verification Script
```bash
# Run these after every deploy:

# 1. Is the app process running?
pm2 list                        # Node.js
sudo systemctl status myapp     # systemd

# 2. Is it responding?
curl -s -o /dev/null -w "%{http_code}" http://localhost:3000
# Should return 200

# 3. Is Nginx forwarding correctly?
curl -s -o /dev/null -w "%{http_code}" http://example.com
# Should return 200

# 4. Any errors in logs?
pm2 logs myapp --lines 20      # Node.js
sudo tail -20 /var/log/nginx/error.log  # Nginx

# 5. Resource usage OK?
free -h                         # RAM not maxed out?
df -h                           # Disk not full?
```

---

## Folder Structure on Server (Best Practice)

```
/var/www/
  ├── myapp/                    # Your application
  │   ├── .env                  # Environment variables (created manually)
  │   ├── node_modules/         # Dependencies
  │   ├── src/                  # Source code
  │   ├── build/ or dist/       # Built files
  │   └── package.json
  │
  ├── wordpress/                # WordPress site
  │   ├── wp-config.php
  │   ├── wp-content/
  │   └── ...
  │
  └── static-site/              # Static React build
      ├── index.html
      ├── static/
      └── assets/

/etc/nginx/
  ├── nginx.conf                # Main config (rarely edit)
  ├── sites-available/          # Your site configs
  │   ├── myapp
  │   ├── wordpress
  │   └── static-site
  └── sites-enabled/            # Symlinks to active configs
      ├── myapp -> ../sites-available/myapp
      └── ...

/var/log/
  ├── nginx/
  │   ├── access.log
  │   └── error.log
  └── myapp/                    # If using custom log paths
```

---

## File Permissions (A Major Source of Bugs)

### Understanding Linux Permissions
```
drwxr-xr-x  deploy deploy  myapp/
 |||
 ||| Owner: deploy (rwx = read+write+execute)
 ||| Group: deploy (r-x = read+execute)
 ||| Others:       (r-x = read+execute)
```

### Correct Ownership
```bash
# Your app files: owned by deploy user
sudo chown -R deploy:deploy /var/www/myapp

# WordPress files: owned by www-data (so PHP-FPM can write)
sudo chown -R www-data:www-data /var/www/wordpress

# Nginx needs to READ your files but NOT own them
# deploy:deploy ownership is fine - Nginx reads as "others"
```

### Common Permission Fixes
```bash
# Can't write to directory (e.g., uploads)
sudo chown -R deploy:www-data /var/www/myapp/uploads
sudo chmod -R 775 /var/www/myapp/uploads

# WordPress wp-content not writable
sudo chown -R www-data:www-data /var/www/wordpress/wp-content
sudo chmod -R 755 /var/www/wordpress/wp-content

# .env file should be private
chmod 600 /var/www/myapp/.env
```

---

## Rollback Strategy (When Things Go Wrong)

### Using Git (Best)
```bash
# See recent deployments
git log --oneline -10

# Rollback to previous commit
git checkout HEAD~1

# Rollback to specific commit
git checkout abc123

# After rollback, restart app
npm install --production
npm run build
pm2 restart myapp
```

### Using Timestamps (Without Git on Server)
```bash
# Before deploying, backup current version
cp -r /var/www/myapp /var/www/myapp-backup-$(date +%Y%m%d-%H%M%S)

# If deploy fails, swap back
mv /var/www/myapp /var/www/myapp-failed
mv /var/www/myapp-backup-20240115-143022 /var/www/myapp
pm2 restart myapp
```

---

## The Manual Deploy Mindset

### Before You Deploy
1. What changed? (read the diff)
2. Any new env vars needed?
3. Any database migrations?
4. Can I rollback easily?

### During Deploy
1. Follow the same steps every time (muscle memory)
2. Check each step before moving to next
3. If something breaks, STOP and investigate

### After Deploy
1. Verify in browser
2. Check logs
3. Monitor for 10 minutes
4. Tell your team

### When Manual Becomes Painful
You should move to automation when:
- You deploy more than 2x per week
- Multiple people deploy
- Deploys take more than 15 minutes
- You forget steps and break things
- You have 5+ servers to update
