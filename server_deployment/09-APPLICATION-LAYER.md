---
layout: default
title: Application Layer
---

# 18 - Application Layer (Layer 7) - Deep Dive

> Your actual code: getting it to the server, configuring it, and keeping it running in production.

---

## The Application Layer in Context

```
All other layers exist to SERVE this layer:

  Infrastructure  --> Provides the machine
  Network         --> Routes traffic to it
  Security        --> Protects it
  Runtime         --> Executes the code
  Web Server      --> Exposes it to the internet
  Process Mgmt    --> Keeps it alive

This layer is about:
  1. Getting your code onto the server
  2. Installing dependencies
  3. Building for production
  4. Configuring environment variables
  5. Database setup and migrations
  6. Verifying everything works
```

---

## 1. Getting Code to the Server

### Method 1: Git (Recommended)

```bash
# --- FIRST TIME SETUP ---

# On server: create app directory
sudo mkdir -p /var/www/myapp
sudo chown deploy:deploy /var/www/myapp

# On server: clone repository
cd /var/www/myapp
git clone https://github.com/you/myapp.git .
# Note the dot (.) at the end -- clones INTO current directory

# For private repos, use SSH:
git clone git@github.com:you/myapp.git .

# --- SUBSEQUENT DEPLOYMENTS ---
cd /var/www/myapp
git pull origin main
```

### Setting Up SSH Key for Private Repos
```bash
# On server: generate deploy key
ssh-keygen -t ed25519 -C "deploy@server" -f ~/.ssh/github_deploy

# View public key
cat ~/.ssh/github_deploy.pub

# Add as Deploy Key on GitHub:
# Repo > Settings > Deploy Keys > Add Deploy Key
# Paste the public key, check "Allow write access" only if needed

# Configure SSH to use this key for GitHub
nano ~/.ssh/config
```
```
Host github.com
    HostName github.com
    User git
    IdentityFile ~/.ssh/github_deploy
```
```bash
# Test connection
ssh -T git@github.com
# Should say: "Hi you/myapp! You've successfully authenticated..."
```

### Method 2: rsync (For When Git Isn't Ideal)
```bash
# From your LOCAL machine, sync files to server
# Skips node_modules, .git, and other unnecessary files

rsync -avz --delete \
    --exclude 'node_modules' \
    --exclude '.git' \
    --exclude '.env' \
    --exclude 'dist' \
    ./ deploy@server:/var/www/myapp/

# Flags:
# -a  = archive (preserves permissions, timestamps)
# -v  = verbose
# -z  = compress during transfer
# --delete = remove files on server that don't exist locally
```

### Method 3: scp (Simple One-Time Transfers)
```bash
# Copy a single file
scp ./config.json deploy@server:/var/www/myapp/

# Copy entire directory
scp -r ./dist/ deploy@server:/var/www/myapp/dist/

# scp is simple but slow for repeated deployments -- prefer rsync or git
```

### Which Method When?
```
git pull        --> Standard. Code in a repo. Most deployments.
rsync           --> Large files, binary assets, or no git on server.
scp             --> Quick one-off file copy.
CI/CD pipeline  --> Automated. See 07-FULL-AUTO-DEPLOYMENT.md.
```

---

## 2. Installing Dependencies

### Node.js
```bash
cd /var/www/myapp

# PRODUCTION install (skip devDependencies)
npm ci --omit=dev

# Why npm ci (not npm install)?
# - npm ci: Deletes node_modules, installs from package-lock.json exactly
# - npm install: May update packages, modify lock file
# - npm ci is faster, deterministic, and won't surprise you

# If you get permission errors:
# You're probably using system Node, not nvm Node
# Fix: nvm use 20 (or whatever version)
```

### PHP / WordPress
```bash
cd /var/www/myapp

# Composer (PHP dependency manager)
composer install --no-dev --optimize-autoloader

# WordPress doesn't use Composer by default
# Dependencies = plugins and themes installed via wp-admin or WP-CLI

# WP-CLI (WordPress command line)
curl -O https://raw.githubusercontent.com/wp-cli/builds/gh-pages/phar/wp-cli.phar
chmod +x wp-cli.phar
sudo mv wp-cli.phar /usr/local/bin/wp

# Install WordPress via CLI
wp core download --path=/var/www/wordpress
wp config create --dbname=wordpress --dbuser=wp_user --dbpass=password --dbhost=localhost
wp core install --url=example.com --title="My Site" --admin_user=admin --admin_password=strongpass --admin_email=you@example.com
```

### Python
```bash
cd /var/www/myapp
source venv/bin/activate
pip install -r requirements.txt
```

---

## 3. Building for Production

### Why Build?
```
Development:
  - Source maps, hot reload, verbose errors
  - Unminified code, dev dependencies

Production build:
  - Minified and optimized code
  - Tree-shaking (removes unused code)
  - Compressed assets
  - No source maps (hide your code)
  - Much smaller bundle size
```

### Build Commands Per Stack

```bash
# --- REACT (Create React App / Vite) ---
npm run build
# Output: ./build/ or ./dist/ (static files)
# Serve with: Nginx pointing to build directory

# --- NEXT.JS ---
npm run build
# Output: ./.next/ directory
# Run with: npm start (or pm2 start npm -- start)

# --- NODE.JS API ---
# Usually no build step needed
# If using TypeScript:
npm run build
# Output: ./dist/ directory
# Run with: node dist/index.js

# --- WORDPRESS ---
# No build step. PHP is interpreted.
# But themes might need: npm run build (for assets)

# --- PHP (Laravel) ---
composer install --no-dev --optimize-autoloader
php artisan config:cache
php artisan route:cache
php artisan view:cache
```

### Build on Server vs Build Locally?
```
BUILD ON SERVER:
  + Simpler workflow (git pull, npm build, restart)
  + No need to transfer build artifacts
  - Uses server CPU/RAM during build
  - Server might not have enough RAM (Next.js build needs 2GB+)

BUILD LOCALLY / IN CI:
  + Server only runs the app (doesn't waste resources building)
  + Build artifacts are tested before deployment
  - Need to transfer built files (rsync/scp/artifact upload)

RECOMMENDATION:
  Small projects: Build on server (simpler)
  Large projects: Build in CI/CD pipeline (see 07-FULL-AUTO-DEPLOYMENT.md)
  Low RAM server: Always build locally/CI (build WILL crash the server)
```

### Dealing with Build Memory Issues
```bash
# Next.js / React build running out of memory?

# Increase Node memory for build
export NODE_OPTIONS="--max-old-space-size=2048"
npm run build

# Or add to package.json scripts:
"build": "NODE_OPTIONS='--max-old-space-size=2048' next build"

# If server has only 1GB RAM:
# Option 1: Add swap (see 01-LAYERED-ARCHITECTURE.md)
# Option 2: Build locally and rsync the output
```

---

## 4. Environment Variables

### Why .env Files?
```
Your code needs configuration that CHANGES between environments:
  - Database passwords (different in dev vs prod)
  - API keys (different keys for test vs live)
  - URLs (localhost:3000 vs api.example.com)
  - Feature flags

These should NEVER be hardcoded in your source code.
They should NEVER be committed to git.
```

### Setting Up .env on Server
```bash
# Create .env file on the server
nano /var/www/myapp/.env
```

```bash
# Example .env for Node.js
NODE_ENV=production
PORT=3000
DATABASE_URL=mysql://myapp_user:strongpassword@localhost:3306/myapp_db
REDIS_URL=redis://localhost:6379
JWT_SECRET=your-64-character-random-string-here
API_KEY=sk-live-xxxxxxxxxxxx
FRONTEND_URL=https://app.example.com

# Example .env for WordPress (wp-config.php handles this differently)
# WordPress uses wp-config.php instead of .env
# But the principle is the same: keep secrets out of git
```

### Generating Secure Secrets
```bash
# Generate a random secret
openssl rand -hex 32
# Output: a3f4b5c6d7e8f9... (64 character hex string)

# Generate a base64 secret
openssl rand -base64 32
# Output: Kx7mN2pQ... (44 character base64 string)

# For JWT_SECRET, SESSION_SECRET, encryption keys:
# Always use: openssl rand -hex 32
```

### .env Security
```bash
# Set strict permissions (only owner can read)
chmod 600 /var/www/myapp/.env

# Verify
ls -la /var/www/myapp/.env
# Should show: -rw------- deploy deploy

# Make sure .env is in .gitignore
cat .gitignore | grep .env
# If not: echo ".env" >> .gitignore

# Check if .env was ever committed to git
git log --all --full-history -- .env
# If YES: ALL secrets in that file are compromised
# You MUST rotate (change) every password/key in that file
```

### Loading .env in Your App

```javascript
// Node.js -- using dotenv package
// npm install dotenv

// At the VERY TOP of your entry file (app.js/index.js)
require('dotenv').config();

// Now use:
const port = process.env.PORT || 3000;
const dbUrl = process.env.DATABASE_URL;
```

```php
// PHP -- manual or using vlucas/phpdotenv
// composer require vlucas/phpdotenv

$dotenv = Dotenv\Dotenv::createImmutable(__DIR__);
$dotenv->load();

// Now use:
$dbHost = $_ENV['DB_HOST'];
```

### WordPress wp-config.php
```php
// /var/www/wordpress/wp-config.php

// Database settings
define('DB_NAME', 'wordpress');
define('DB_USER', 'wp_user');
define('DB_PASSWORD', 'strong-password-here');
define('DB_HOST', 'localhost');

// Security keys (generate at: https://api.wordpress.org/secret-key/1.1/salt/)
define('AUTH_KEY',         'unique-phrase-here');
define('SECURE_AUTH_KEY',  'unique-phrase-here');
define('LOGGED_IN_KEY',    'unique-phrase-here');
define('NONCE_KEY',        'unique-phrase-here');
define('AUTH_SALT',        'unique-phrase-here');
define('SECURE_AUTH_SALT', 'unique-phrase-here');
define('LOGGED_IN_SALT',   'unique-phrase-here');
define('NONCE_SALT',       'unique-phrase-here');

// Force HTTPS (if behind Nginx SSL)
define('FORCE_SSL_ADMIN', true);
if ($_SERVER['HTTP_X_FORWARDED_PROTO'] === 'https') {
    $_SERVER['HTTPS'] = 'on';
}

// File permissions
define('FS_METHOD', 'direct');
```

---

## 5. Database Setup

### MySQL/MariaDB for Applications
```bash
# Create database and user (run as root MySQL user)
sudo mysql
```

```sql
-- Create database
CREATE DATABASE myapp_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

-- Create application user (NEVER use root for your app)
CREATE USER 'myapp_user'@'localhost' IDENTIFIED BY 'strong-password-here';

-- Grant permissions (only on this database)
GRANT ALL PRIVILEGES ON myapp_db.* TO 'myapp_user'@'localhost';
FLUSH PRIVILEGES;

-- Verify
SHOW GRANTS FOR 'myapp_user'@'localhost';
EXIT;
```

```bash
# Test connection
mysql -u myapp_user -p myapp_db -e "SELECT 1;"
```

### Database Migrations
```bash
# --- NODE.JS (with common ORMs) ---

# Prisma
npx prisma migrate deploy         # Apply pending migrations in production
npx prisma generate                # Generate client

# Sequelize
npx sequelize db:migrate           # Run migrations
npx sequelize db:seed:all          # Run seeders (if needed)

# Knex
npx knex migrate:latest            # Run migrations

# --- PHP (Laravel) ---
php artisan migrate --force         # --force required in production

# --- WORDPRESS ---
# No migration step -- WordPress handles schema internally
# Plugins may create tables on activation
```

### Database Backups
```bash
# Manual backup
mysqldump -u myapp_user -p myapp_db > /backups/myapp_$(date +%Y%m%d_%H%M%S).sql

# Automated daily backup (add to crontab)
crontab -e
# Add:
0 3 * * * mysqldump -u myapp_user -pYOUR_PASSWORD myapp_db | gzip > /backups/myapp_$(date +\%Y\%m\%d).sql.gz

# Restore from backup
mysql -u myapp_user -p myapp_db < /backups/myapp_20240115.sql

# Compressed restore
gunzip < /backups/myapp_20240115.sql.gz | mysql -u myapp_user -p myapp_db
```

---

## 6. File Permissions and Ownership

### The Rules
```
WHO should own application files?

Option A (Recommended for Node.js/Next.js):
  Owner: deploy (the user who deploys)
  Nginx connects via proxy -- doesn't need file access

Option B (Required for PHP/WordPress):
  Owner: www-data (the web server user)
  PHP-FPM runs as www-data, needs to read/write files
```

### Setting Permissions
```bash
# --- NODE.JS APP ---
sudo chown -R deploy:deploy /var/www/myapp
chmod -R 755 /var/www/myapp

# --- WORDPRESS / PHP ---
sudo chown -R www-data:www-data /var/www/wordpress

# Directories: 755 (rwxr-xr-x)
find /var/www/wordpress -type d -exec chmod 755 {} \;

# Files: 644 (rw-r--r--)
find /var/www/wordpress -type f -exec chmod 644 {} \;

# wp-config.php: Extra restrictive
chmod 400 /var/www/wordpress/wp-config.php

# Uploads directory: writable by web server
chmod 775 /var/www/wordpress/wp-content/uploads

# --- .env file (any stack) ---
chmod 600 /var/www/myapp/.env
```

### Permission Numbers Explained
```
7 = rwx (read + write + execute)
6 = rw- (read + write)
5 = r-x (read + execute)
4 = r-- (read only)
0 = --- (no access)

755 = owner:rwx, group:r-x, others:r-x  (directories)
644 = owner:rw-, group:r--, others:r--   (files)
600 = owner:rw-, group:---, others:---   (.env, secrets)
400 = owner:r--, group:---, others:---   (wp-config.php)
```

---

## 7. Health Checks and Verification

### After Every Deployment, Verify

```bash
# Step 1: Is the app process running?
pm2 list                               # PM2
sudo systemctl status myapp            # systemd

# Step 2: Is the app responding locally?
curl http://localhost:3000             # Direct app check
curl http://localhost:3000/api/health  # Health endpoint

# Step 3: Is Nginx forwarding correctly?
curl -I http://localhost               # Through Nginx (port 80)
curl -I https://example.com            # Full external check

# Step 4: Check for errors
pm2 logs my-api --lines 20            # Recent app logs
sudo tail -20 /var/log/nginx/error.log # Nginx errors

# Step 5: Database connection working?
curl http://localhost:3000/api/health  # If health check queries DB
```

### Build a Health Check Endpoint
```javascript
// Node.js / Express
app.get('/api/health', async (req, res) => {
    try {
        // Check database
        await db.query('SELECT 1');

        res.json({
            status: 'ok',
            timestamp: new Date().toISOString(),
            uptime: process.uptime(),
            memory: process.memoryUsage().rss
        });
    } catch (error) {
        res.status(503).json({
            status: 'error',
            message: 'Database connection failed'
        });
    }
});
```

```php
// PHP
// /api/health.php
<?php
try {
    $pdo = new PDO($dsn, $user, $pass);
    $pdo->query('SELECT 1');
    echo json_encode(['status' => 'ok', 'timestamp' => date('c')]);
} catch (Exception $e) {
    http_response_code(503);
    echo json_encode(['status' => 'error']);
}
```

---

## 8. Complete Deployment Flows

### Node.js API Deployment
```bash
# SSH into server
ssh deploy@server

# Navigate to app
cd /var/www/myapp

# Pull latest code
git pull origin main

# Install dependencies (production only)
npm ci --omit=dev

# Run database migrations
npx prisma migrate deploy       # Or your ORM's command

# Restart app (zero downtime with PM2 cluster)
pm2 reload my-api

# Verify
curl http://localhost:3000/api/health
pm2 logs my-api --lines 10
```

### Next.js Deployment
```bash
ssh deploy@server
cd /var/www/nextjs-app

git pull origin main
npm ci --omit=dev

# Build (this takes a while and uses RAM)
npm run build

# Restart
pm2 reload nextjs-app

# Verify
curl http://localhost:3000
```

### React (Static) Deployment
```bash
ssh deploy@server
cd /var/www/react-app

git pull origin main
npm ci
npm run build                    # Output goes to build/

# No restart needed -- Nginx serves static files directly
# But reload Nginx if you changed the config:
sudo nginx -t && sudo systemctl reload nginx

# Verify
curl -I https://app.example.com
```

### WordPress Deployment
```bash
ssh deploy@server
cd /var/www/wordpress

# Pull theme/plugin updates from git (if version controlled)
git pull origin main

# Update WordPress core via WP-CLI
wp core update
wp plugin update --all
wp theme update --all

# Clear caches
wp cache flush

# Verify
curl -I https://blog.example.com
```

### PHP Application Deployment
```bash
ssh deploy@server
cd /var/www/phpapp

git pull origin main
composer install --no-dev --optimize-autoloader

# Laravel specific
php artisan migrate --force
php artisan config:cache
php artisan route:cache
php artisan view:cache

# Restart PHP-FPM (picks up new code)
sudo systemctl restart php8.3-fpm

# Verify
curl http://localhost
```

---

## 9. Common Application Issues

### App Starts But Can't Connect to Database
```bash
# Check database is running
sudo systemctl status mysql

# Check credentials
mysql -u myapp_user -p myapp_db

# Check .env has correct DATABASE_URL
cat .env | grep DATABASE

# Common mistake: localhost vs 127.0.0.1
# MySQL socket vs TCP -- try both in your connection string
```

### App Works Locally But Not Through Nginx
```bash
# Check Nginx error log
sudo tail -20 /var/log/nginx/error.log

# Check proxy_pass port matches app port
grep proxy_pass /etc/nginx/sites-enabled/myapp

# Check app is listening on the right port
ss -tlnp | grep 3000

# Common mistake: app listening on 127.0.0.1:3000 vs 0.0.0.0:3000
# For PM2/Nginx setup, 127.0.0.1 is fine (Nginx connects locally)
```

### "Module not found" After Deploy
```bash
# Dependencies not installed
npm ci --omit=dev

# Wrong Node version
node -v                           # Check version
nvm use 20                        # Switch if needed

# Missing native modules (need rebuild)
npm rebuild
```

### WordPress White Screen of Death
```bash
# Enable debug mode temporarily
wp config set WP_DEBUG true --raw
wp config set WP_DEBUG_LOG true --raw

# Check the debug log
tail -50 /var/www/wordpress/wp-content/debug.log

# Common causes:
# - PHP memory limit too low
# - Plugin conflict
# - Corrupt theme

# Disable all plugins via CLI
wp plugin deactivate --all

# Switch to default theme
wp theme activate twentytwentyfour
```

---

## 10. Application Monitoring Basics

### What to Monitor
```
+-----------------------+-----------------------------------+
| What                  | How to Check                      |
+-----------------------+-----------------------------------+
| App is running        | pm2 list / systemctl status       |
| App is responding     | curl health endpoint              |
| Error rate            | Check error logs                  |
| Memory usage          | pm2 monit / free -h               |
| Disk space            | df -h                             |
| CPU usage             | htop                              |
| Response time         | Nginx access log analysis         |
| SSL certificate       | certbot certificates              |
+-----------------------+-----------------------------------+
```

### Simple Uptime Check Script
```bash
#!/bin/bash
# /home/deploy/check-health.sh

URL="http://localhost:3000/api/health"
RESPONSE=$(curl -s -o /dev/null -w "%{http_code}" $URL)

if [ "$RESPONSE" != "200" ]; then
    echo "[$(date)] ALERT: Health check failed! Status: $RESPONSE" >> /var/log/health-check.log
    pm2 restart my-api
    # Optional: send notification
    # curl -X POST https://hooks.slack.com/... -d '{"text":"App restarted!"}'
fi
```

```bash
# Run every 5 minutes via cron
crontab -e
# Add:
*/5 * * * * /home/deploy/check-health.sh
```

---

## Quick Reference: Deployment Commands

```bash
# --- CODE ---
git pull origin main                    # Get latest code
rsync -avz ./ deploy@server:/var/www/   # Sync files

# --- DEPENDENCIES ---
npm ci --omit=dev                       # Node.js
composer install --no-dev               # PHP

# --- BUILD ---
npm run build                           # React/Next.js/TypeScript
php artisan config:cache                # Laravel

# --- RESTART ---
pm2 reload my-api                       # Node.js (zero downtime)
sudo systemctl restart myapp            # systemd
sudo systemctl restart php8.3-fpm      # PHP

# --- VERIFY ---
curl http://localhost:3000/api/health   # Health check
pm2 logs my-api --lines 20             # Check logs
sudo tail -20 /var/log/nginx/error.log  # Nginx errors

# --- ROLLBACK ---
git log --oneline -5                    # Find previous commit
git checkout abc1234                    # Go back to that commit
npm ci --omit=dev && pm2 reload my-api  # Reinstall + restart
```
