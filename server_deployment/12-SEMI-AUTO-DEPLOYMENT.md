---
layout: default
title: Semi-Auto Deployment
---

# 06 - Semi-Auto Deployment (Shell Scripts)

## What is Semi-Auto?
You still trigger the deploy manually, but the STEPS are automated in a script.
Instead of typing 10 commands, you type ONE command.

```
MANUAL:     SSH -> git pull -> npm install -> build -> restart -> verify (10 mins)
SEMI-AUTO:  Run one script that does all of the above (2 mins)
FULL-AUTO:  Push to git -> deployment happens automatically (0 mins)
```

---

## Level 1: Deploy Script on Server

### Basic Deploy Script
```bash
#!/bin/bash
# /home/deploy/deploy.sh
# Usage: ./deploy.sh

set -e  # Exit on any error

APP_DIR="/var/www/myapp"
APP_NAME="myapp"
BRANCH="main"

echo "=== Starting deployment ==="

# Step 1: Pull latest code
echo "Pulling latest code..."
cd $APP_DIR
git pull origin $BRANCH

# Step 2: Install dependencies
echo "Installing dependencies..."
npm install --production

# Step 3: Build
echo "Building..."
npm run build

# Step 4: Restart app
echo "Restarting application..."
pm2 restart $APP_NAME

# Step 5: Verify
echo "Waiting 3 seconds..."
sleep 3
if curl -s -o /dev/null -w "%{http_code}" http://localhost:3000 | grep -q "200"; then
    echo "=== Deployment successful! ==="
else
    echo "=== WARNING: App might not be responding correctly ==="
    pm2 logs $APP_NAME --lines 20
fi
```
```bash
chmod +x /home/deploy/deploy.sh
# Now deploy with: ./deploy.sh
```

### Deploy Script with Rollback
```bash
#!/bin/bash
# /home/deploy/deploy-safe.sh
set -e

APP_DIR="/var/www/myapp"
APP_NAME="myapp"
BRANCH="main"

cd $APP_DIR

# Save current commit for rollback
PREVIOUS_COMMIT=$(git rev-parse HEAD)
echo "Current commit: $PREVIOUS_COMMIT"

# Deploy
echo "Pulling latest..."
git pull origin $BRANCH
NEW_COMMIT=$(git rev-parse HEAD)
echo "New commit: $NEW_COMMIT"

if [ "$PREVIOUS_COMMIT" = "$NEW_COMMIT" ]; then
    echo "No new changes. Skipping deploy."
    exit 0
fi

echo "Installing dependencies..."
npm install --production

echo "Building..."
npm run build

echo "Restarting..."
pm2 restart $APP_NAME

# Health check
sleep 5
HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:3000)
if [ "$HTTP_CODE" = "200" ]; then
    echo "Deploy successful! ($PREVIOUS_COMMIT -> $NEW_COMMIT)"
else
    echo "DEPLOY FAILED! Rolling back to $PREVIOUS_COMMIT"
    git checkout $PREVIOUS_COMMIT
    npm install --production
    npm run build
    pm2 restart $APP_NAME
    echo "Rolled back to $PREVIOUS_COMMIT"
    exit 1
fi
```

---

## Level 2: Deploy from Local Machine

### Local Deploy Script (runs on YOUR machine)
```bash
#!/bin/bash
# ./deploy-to-server.sh (on your local machine)
# Usage: ./deploy-to-server.sh production

set -e

if [ -z "$1" ]; then
    echo "Usage: ./deploy-to-server.sh <environment>"
    echo "  Environments: production, staging"
    exit 1
fi

ENV=$1

if [ "$ENV" = "production" ]; then
    SERVER="deploy@your-production-ip"
    APP_DIR="/var/www/myapp"
elif [ "$ENV" = "staging" ]; then
    SERVER="deploy@your-staging-ip"
    APP_DIR="/var/www/myapp-staging"
else
    echo "Unknown environment: $ENV"
    exit 1
fi

echo "Deploying to $ENV ($SERVER)..."

# Option A: Run remote deploy script
ssh $SERVER "cd $APP_DIR && git pull origin main && npm install --production && npm run build && pm2 restart myapp"

# Option B: If you have a deploy script on server
# ssh $SERVER "/home/deploy/deploy.sh"

echo "Deployment to $ENV complete!"
```

### Rsync Deploy Script (For Static Sites / React)
```bash
#!/bin/bash
# ./deploy-react.sh
set -e

SERVER="deploy@your-server-ip"
REMOTE_DIR="/var/www/myreactapp/build"

echo "Building locally..."
npm run build

echo "Uploading to server..."
rsync -avz --delete ./build/ $SERVER:$REMOTE_DIR/

echo "Deployment complete!"
echo "Site: https://app.example.com"
```

---

## Level 3: Multi-Server Deploy

### Deploy to Multiple Servers
```bash
#!/bin/bash
# deploy-all.sh
set -e

SERVERS=(
    "deploy@server1-ip"
    "deploy@server2-ip"
    "deploy@server3-ip"
)

DEPLOY_CMD="cd /var/www/myapp && git pull origin main && npm install --production && npm run build && pm2 restart myapp"

for SERVER in "${SERVERS[@]}"; do
    echo "Deploying to $SERVER..."
    ssh $SERVER "$DEPLOY_CMD"
    echo "Done: $SERVER"
    echo "---"
done

echo "All servers deployed!"
```

---

## Level 4: Server Setup Script (New Server Bootstrap)

### One Command to Setup a Fresh Server
```bash
#!/bin/bash
# setup-server.sh - Run this on a fresh Ubuntu server
# Usage: curl -s https://yoururl/setup.sh | bash
# Or: scp setup.sh deploy@newserver: && ssh deploy@newserver './setup.sh'

set -e

echo "=== Server Setup Script ==="

# Update system
echo "Updating system..."
sudo apt update && sudo apt upgrade -y

# Install essentials
echo "Installing essentials..."
sudo apt install -y curl git nginx ufw fail2ban certbot python3-certbot-nginx htop

# Setup firewall
echo "Configuring firewall..."
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow OpenSSH
sudo ufw allow 'Nginx Full'
sudo ufw --force enable

# Setup swap (2GB)
if [ ! -f /swapfile ]; then
    echo "Setting up swap..."
    sudo fallocate -l 2G /swapfile
    sudo chmod 600 /swapfile
    sudo mkswap /swapfile
    sudo swapon /swapfile
    echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
fi

# Install Node.js via nvm
echo "Installing Node.js..."
if [ ! -d "$HOME/.nvm" ]; then
    curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.0/install.sh | bash
    export NVM_DIR="$HOME/.nvm"
    [ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"
    nvm install 20
    nvm alias default 20
fi

# Install PM2
echo "Installing PM2..."
npm install -g pm2

# Enable fail2ban
sudo systemctl enable fail2ban
sudo systemctl start fail2ban

# Enable auto security updates
sudo apt install -y unattended-upgrades
echo 'APT::Periodic::Update-Package-Lists "1";' | sudo tee /etc/apt/apt.conf.d/20auto-upgrades
echo 'APT::Periodic::Unattended-Upgrade "1";' | sudo tee -a /etc/apt/apt.conf.d/20auto-upgrades

# Set timezone
sudo timedatectl set-timezone UTC

echo ""
echo "=== Server setup complete! ==="
echo "Next steps:"
echo "  1. Configure Nginx: sudo nano /etc/nginx/sites-available/myapp"
echo "  2. Enable site: sudo ln -s /etc/nginx/sites-available/myapp /etc/nginx/sites-enabled/"
echo "  3. Clone your app: git clone <repo> /var/www/myapp"
echo "  4. Setup SSL: sudo certbot --nginx -d example.com"
```

---

## Level 5: Makefile (Organize Commands)

### Using Makefile for Common Operations
```makefile
# Makefile (in your project root)

SERVER=deploy@your-server-ip
APP_DIR=/var/www/myapp
APP_NAME=myapp

.PHONY: deploy deploy-staging setup logs restart status

deploy:
	@echo "Deploying to production..."
	ssh $(SERVER) "cd $(APP_DIR) && git pull origin main && npm install --production && npm run build && pm2 restart $(APP_NAME)"
	@echo "Done!"

deploy-staging:
	@echo "Deploying to staging..."
	ssh deploy@staging-ip "cd /var/www/myapp-staging && git pull origin develop && npm install && npm run build && pm2 restart myapp-staging"

logs:
	ssh $(SERVER) "pm2 logs $(APP_NAME) --lines 50"

restart:
	ssh $(SERVER) "pm2 restart $(APP_NAME)"

status:
	ssh $(SERVER) "pm2 list && echo '---' && free -h && echo '---' && df -h"

backup:
	ssh $(SERVER) "/home/deploy/backup.sh"
	@echo "Backup complete"

setup-nginx:
	scp ./nginx/myapp.conf $(SERVER):/tmp/myapp.conf
	ssh $(SERVER) "sudo mv /tmp/myapp.conf /etc/nginx/sites-available/myapp && sudo ln -sf /etc/nginx/sites-available/myapp /etc/nginx/sites-enabled/ && sudo nginx -t && sudo systemctl reload nginx"
```
```bash
# Now you can run:
make deploy           # Deploy to production
make deploy-staging   # Deploy to staging
make logs             # View logs
make status           # Check server status
make backup           # Run backup
```

---

## When to Move from Semi-Auto to Full Auto

### You Need Full Auto When:
- [ ] Deploying more than 3x per week
- [ ] Multiple developers deploying
- [ ] Need deployment history/audit trail
- [ ] Need automatic rollback
- [ ] Running tests before deploy
- [ ] Deploying to 3+ servers
- [ ] Need zero-downtime deployments

### Semi-Auto is Fine When:
- Solo developer or small team
- Deploy 1-2x per week
- 1-2 servers
- Simple apps without complex pipelines
