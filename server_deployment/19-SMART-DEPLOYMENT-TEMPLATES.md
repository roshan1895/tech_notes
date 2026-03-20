# 11 - Smart Deployment Templates (Reusable Scripts)

## Goal: Deploy Any Project in Under 10 Minutes

These templates are copy-paste ready. Keep them on your local machine
and reuse for every new deployment.

---

## Template 1: Server Bootstrap (Use on Every New Server)

Save as: `~/deploy-templates/server-bootstrap.sh`

```bash
#!/bin/bash
# Server Bootstrap Script
# Usage: scp server-bootstrap.sh deploy@newserver: && ssh deploy@newserver 'bash server-bootstrap.sh'

set -e
echo "========================================="
echo "  SERVER BOOTSTRAP"
echo "========================================="

# System update
echo "[1/8] Updating system..."
sudo apt update && sudo apt upgrade -y

# Install essentials
echo "[2/8] Installing essentials..."
sudo apt install -y curl git nginx ufw fail2ban certbot python3-certbot-nginx \
  htop unzip wget software-properties-common

# Firewall
echo "[3/8] Configuring firewall..."
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow OpenSSH
sudo ufw allow 'Nginx Full'
sudo ufw --force enable

# Swap
echo "[4/8] Setting up swap..."
if [ ! -f /swapfile ]; then
    sudo fallocate -l 2G /swapfile
    sudo chmod 600 /swapfile
    sudo mkswap /swapfile
    sudo swapon /swapfile
    echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
fi

# Node.js (via nvm)
echo "[5/8] Installing Node.js..."
if [ ! -d "$HOME/.nvm" ]; then
    curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.0/install.sh | bash
    export NVM_DIR="$HOME/.nvm"
    [ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"
    nvm install 20
    nvm alias default 20
    npm install -g pm2
fi

# Fail2ban
echo "[6/8] Configuring fail2ban..."
sudo systemctl enable fail2ban
sudo systemctl start fail2ban

# Auto-updates
echo "[7/8] Enabling auto security updates..."
sudo apt install -y unattended-upgrades
echo 'APT::Periodic::Update-Package-Lists "1";' | sudo tee /etc/apt/apt.conf.d/20auto-upgrades
echo 'APT::Periodic::Unattended-Upgrade "1";' | sudo tee -a /etc/apt/apt.conf.d/20auto-upgrades

# Nginx defaults
echo "[8/8] Configuring Nginx defaults..."
sudo rm -f /etc/nginx/sites-enabled/default
sudo tee /etc/nginx/conf.d/security.conf > /dev/null <<'CONF'
server_tokens off;
add_header X-Frame-Options "SAMEORIGIN" always;
add_header X-Content-Type-Options "nosniff" always;
add_header X-XSS-Protection "1; mode=block" always;
CONF
sudo nginx -t && sudo systemctl reload nginx

# Timezone
sudo timedatectl set-timezone UTC

echo ""
echo "========================================="
echo "  BOOTSTRAP COMPLETE!"
echo "========================================="
echo ""
echo "Installed: Nginx, Node.js 20, PM2, ufw, fail2ban, certbot"
echo ""
echo "Next steps:"
echo "  1. Create app directory: sudo mkdir -p /var/www/myapp && sudo chown deploy:deploy /var/www/myapp"
echo "  2. Clone your repo: git clone <url> /var/www/myapp"
echo "  3. Create Nginx config: sudo nano /etc/nginx/sites-available/myapp"
echo "  4. Enable site: sudo ln -s /etc/nginx/sites-available/myapp /etc/nginx/sites-enabled/"
echo "  5. Setup SSL: sudo certbot --nginx -d example.com"
```

---

## Template 2: Nginx Configs (Copy & Customize)

Save as: `~/deploy-templates/nginx/`

### Node.js / Next.js API (Reverse Proxy)
```nginx
# node-proxy.conf
# Replace: DOMAIN, PORT

server {
    listen 80;
    server_name DOMAIN;

    client_max_body_size 10M;

    location / {
        proxy_pass http://localhost:PORT;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
        proxy_read_timeout 60s;
    }
}
```

### React / Static Site
```nginx
# static-site.conf
# Replace: DOMAIN, ROOT_PATH

server {
    listen 80;
    server_name DOMAIN;

    root ROOT_PATH;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }

    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }

    location = /index.html {
        expires -1;
        add_header Cache-Control "no-store, no-cache, must-revalidate";
    }

    gzip on;
    gzip_types text/plain text/css application/json application/javascript text/xml;
    gzip_min_length 1000;
}
```

### WordPress / PHP
```nginx
# wordpress.conf
# Replace: DOMAIN, ROOT_PATH

server {
    listen 80;
    server_name DOMAIN;

    root ROOT_PATH;
    index index.php index.html;
    client_max_body_size 64M;

    location / {
        try_files $uri $uri/ /index.php?$args;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php8.2-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }

    location ~ /\.ht { deny all; }
    location = /wp-config.php { deny all; }

    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2)$ {
        expires 30d;
        add_header Cache-Control "public";
    }
}
```

---

## Template 3: Quick Deploy Script

Save as: `~/deploy-templates/deploy.sh`

```bash
#!/bin/bash
# Universal deploy script
# Copy to each project and customize the variables at top

set -e

# ============ CUSTOMIZE THESE ============
APP_NAME="myapp"
APP_DIR="/var/www/myapp"
APP_PORT=3000
BRANCH="main"
BUILD_CMD="npm run build"      # Leave empty if no build step
INSTALL_CMD="npm install --production"
# =========================================

cd $APP_DIR

# Save current commit for rollback
PREVIOUS=$(git rev-parse HEAD)

echo "[$APP_NAME] Pulling latest from $BRANCH..."
git pull origin $BRANCH

CURRENT=$(git rev-parse HEAD)
if [ "$PREVIOUS" = "$CURRENT" ]; then
    echo "No changes. Skipping."
    exit 0
fi

echo "[$APP_NAME] Installing dependencies..."
$INSTALL_CMD

if [ -n "$BUILD_CMD" ]; then
    echo "[$APP_NAME] Building..."
    $BUILD_CMD
fi

echo "[$APP_NAME] Restarting..."
pm2 restart $APP_NAME

# Health check
sleep 3
HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:$APP_PORT)

if [ "$HTTP_CODE" = "200" ]; then
    echo "[$APP_NAME] Deploy successful! ($PREVIOUS -> $CURRENT)"
else
    echo "[$APP_NAME] HEALTH CHECK FAILED (HTTP $HTTP_CODE)!"
    echo "Rolling back..."
    git checkout $PREVIOUS
    $INSTALL_CMD
    [ -n "$BUILD_CMD" ] && $BUILD_CMD
    pm2 restart $APP_NAME
    echo "[$APP_NAME] Rolled back to $PREVIOUS"
    exit 1
fi
```

---

## Template 4: GitHub Actions CI/CD

Save as: `~/deploy-templates/github-actions/`

### Node.js API
```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      - run: npm test

  deploy:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to server
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.SERVER_IP }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            cd /var/www/myapp
            git pull origin main
            npm ci --production
            npm run build
            pm2 restart myapp
```

### React Static Deploy
```yaml
# .github/workflows/deploy-static.yml
name: Deploy Static

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      - run: npm run build

      - name: Deploy to server
        uses: burnett01/rsync-deployments@7.0.1
        with:
          switches: -avz --delete
          path: build/
          remote_path: /var/www/myapp/build/
          remote_host: ${{ secrets.SERVER_IP }}
          remote_user: ${{ secrets.SERVER_USER }}
          remote_key: ${{ secrets.SSH_PRIVATE_KEY }}
```

---

## Template 5: Docker Compose Stacks

### Node.js + PostgreSQL
```yaml
# docker-compose.yml
version: '3.8'
services:
  app:
    build: .
    ports:
      - "3000:3000"
    env_file: .env
    depends_on:
      - db
    restart: unless-stopped
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASS}
      POSTGRES_DB: ${DB_NAME}
    volumes:
      - pgdata:/var/lib/postgresql/data
    restart: unless-stopped
volumes:
  pgdata:
```

### WordPress + MySQL
```yaml
version: '3.8'
services:
  wordpress:
    image: wordpress:latest
    ports:
      - "80:80"
    env_file: .env
    volumes:
      - wp_data:/var/www/html
    restart: unless-stopped
  db:
    image: mysql:8
    env_file: .env
    volumes:
      - db_data:/var/lib/mysql
    restart: unless-stopped
volumes:
  wp_data:
  db_data:
```

---

## Template 6: Quick Commands Reference Card

Save as: `~/deploy-templates/COMMANDS.md`

```
=== SSH ===
ssh deploy@SERVER_IP
ssh -i key.pem ubuntu@SERVER_IP

=== DEPLOY ===
cd /var/www/APP && git pull origin main && npm install --production && npm run build && pm2 restart APP

=== PM2 ===
pm2 list | pm2 logs APP | pm2 restart APP | pm2 stop APP
pm2 startup && pm2 save    (survive reboot)

=== NGINX ===
sudo nginx -t                               (test config)
sudo systemctl reload nginx                 (apply changes)
sudo nano /etc/nginx/sites-available/APP    (edit config)
sudo ln -s /etc/nginx/sites-available/APP /etc/nginx/sites-enabled/  (enable)

=== SSL ===
sudo certbot --nginx -d DOMAIN
sudo certbot certificates
sudo certbot renew --dry-run

=== LOGS ===
pm2 logs APP --lines 50
sudo tail -50 /var/log/nginx/error.log
journalctl -u APP -f

=== SYSTEM ===
free -h          (RAM)
df -h            (Disk)
htop             (CPU/processes)
sudo ss -tlnp    (open ports)
sudo ufw status  (firewall)

=== DATABASE ===
sudo mysql -u root -p
mysqldump -u USER -p DB > backup.sql
mysql -u USER -p DB < backup.sql
```

---

## Workflow: Deploy a New Project (10-Minute Plan)

```
Minute 0-2:   Bootstrap server (if new) - run Template 1
Minute 2-4:   Clone repo, install deps, set .env
Minute 4-6:   Copy matching Nginx template, customize, enable
Minute 6-7:   Setup SSL with certbot
Minute 7-8:   Start with PM2, configure startup
Minute 8-10:  Verify everything, check logs

Done. Project live.
```

### Step-by-Step for Existing Server (Adding New App)
```bash
# 1. Create directory
sudo mkdir -p /var/www/newapp
sudo chown deploy:deploy /var/www/newapp

# 2. Clone and setup
git clone https://github.com/you/repo.git /var/www/newapp
cd /var/www/newapp
npm install --production
cp .env.example .env
nano .env    # Fill in values
npm run build

# 3. Nginx (copy template, customize)
sudo cp ~/deploy-templates/nginx/node-proxy.conf /etc/nginx/sites-available/newapp
sudo sed -i 's/DOMAIN/newapp.example.com/g' /etc/nginx/sites-available/newapp
sudo sed -i 's/PORT/3001/g' /etc/nginx/sites-available/newapp
sudo ln -s /etc/nginx/sites-available/newapp /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx

# 4. SSL
sudo certbot --nginx -d newapp.example.com

# 5. PM2
pm2 start npm --name "newapp" -- start
pm2 save

# 6. Verify
curl https://newapp.example.com
pm2 logs newapp --lines 10
```
