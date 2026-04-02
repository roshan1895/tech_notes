---
layout: default
title: Stack-Specific Guides
---

# 04 - Stack-Specific Deployment Guides

## Stack Comparison Overview

| Stack | Web Server | Runtime | Process Mgr | Build Step | Database |
|-------|-----------|---------|-------------|------------|----------|
| Node.js API | Nginx (proxy) | Node.js | PM2 | Optional | MongoDB/PostgreSQL |
| React (CRA) | Nginx (static) | None | None | `npm run build` | None |
| Next.js | Nginx (proxy) | Node.js | PM2 | `npm run build` | Optional |
| WordPress | Nginx or Apache | PHP-FPM | systemd | None | MySQL/MariaDB |
| PHP (custom) | Nginx | PHP-FPM | systemd | Optional | MySQL/PostgreSQL |

---

## 1. Node.js Backend API

### Architecture
```
Client -> Nginx:443 -> PM2 -> Node.js:3000 -> Database
```

### Full Setup
```bash
# Runtime
nvm install 20 && nvm alias default 20

# Deploy code
cd /var/www
git clone https://github.com/you/node-api.git myapi
cd myapi
npm install --production

# Environment variables
nano .env
# DATABASE_URL=...
# JWT_SECRET=...
# NODE_ENV=production
# PORT=3000

# Start with PM2
pm2 start app.js --name "myapi"    # or: pm2 start npm --name "myapi" -- start
pm2 startup
pm2 save
```

### Nginx Config
```nginx
# /etc/nginx/sites-available/myapi
server {
    listen 80;
    server_name api.example.com;

    # Max upload size
    client_max_body_size 10M;

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

        # Timeouts for long requests
        proxy_read_timeout 60s;
        proxy_send_timeout 60s;
    }
}
```

### Redeploy Steps
```bash
ssh deploy@server
cd /var/www/myapi
git pull origin main
npm install --production
pm2 restart myapi
pm2 logs myapi --lines 20
```

### Common Issues
| Problem | Cause | Fix |
|---------|-------|-----|
| EADDRINUSE | Port 3000 already in use | `pm2 delete all` then restart |
| Cannot find module | Missing dependency | `rm -rf node_modules && npm install` |
| ECONNREFUSED to DB | Database not running | `sudo systemctl start mysql` |
| Memory spikes | No memory limit | PM2 ecosystem: `max_memory_restart: '500M'` |

---

## 2. React Frontend (Static Build)

### Architecture
```
Client -> Nginx:443 -> Static files (HTML/CSS/JS)
```

### Why No Node.js Needed on Server
React builds into static files. No server-side runtime needed.
Nginx serves them directly - extremely fast and simple.

### Build Locally, Deploy Files
```bash
# On YOUR machine (not the server)
npm run build
# Creates: build/ folder with static files

# Upload to server
rsync -avz ./build/ deploy@server:/var/www/myreactapp/
```

### OR Build on Server
```bash
ssh deploy@server
cd /var/www
git clone https://github.com/you/react-app.git myreactapp
cd myreactapp
npm install
npm run build
# Built files are in build/ or dist/
```

### Nginx Config
```nginx
# /etc/nginx/sites-available/myreactapp
server {
    listen 80;
    server_name app.example.com;

    root /var/www/myreactapp/build;  # or dist/ for Vite
    index index.html;

    # SPA routing - ALL routes serve index.html
    location / {
        try_files $uri $uri/ /index.html;
    }

    # Cache static assets aggressively
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2|ttf|eot)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }

    # Don't cache index.html (so new deploys are picked up)
    location = /index.html {
        expires -1;
        add_header Cache-Control "no-store, no-cache, must-revalidate";
    }

    # Gzip compression
    gzip on;
    gzip_types text/plain text/css application/json application/javascript text/xml;
    gzip_min_length 1000;
}
```

### Key Concept: SPA Routing
```
WHY try_files $uri $uri/ /index.html; ?

React Router handles routes like /dashboard, /users, /settings
But these don't exist as actual files on the server.
Without try_files, Nginx returns 404 for /dashboard.
With try_files, Nginx serves index.html, then React Router handles the route.
```

### API Connection
```
React app calls: https://api.example.com/users
                     ^
                     |
           This is your Node.js backend on another Nginx server block
           (could be same server or different server)

.env (in React project, before building):
REACT_APP_API_URL=https://api.example.com
```

### Redeploy Steps
```bash
# Option 1: Build locally, upload
npm run build
rsync -avz --delete ./build/ deploy@server:/var/www/myreactapp/build/

# Option 2: Build on server
ssh deploy@server
cd /var/www/myreactapp
git pull origin main
npm install
npm run build
# No restart needed - Nginx serves static files automatically
```

---

## 3. Next.js (SSR / Full Stack)

### Architecture
```
Client -> Nginx:443 -> PM2 -> Next.js:3000
                                  |
                         (Server-side rendering)
                                  |
                              Database (optional)
```

### Why Next.js is Different from React
```
React (CRA):  Build once -> Static files -> Nginx serves directly
Next.js:      Running Node.js server -> Renders pages on each request (SSR)
              Also has static pages (SSG) and API routes

This means Next.js NEEDS:
- Node.js running on server
- PM2 to keep it alive
- Nginx as reverse proxy (just like Node.js API)
```

### Full Setup
```bash
# Runtime
nvm install 20

# Deploy
cd /var/www
git clone https://github.com/you/nextjs-app.git mynextapp
cd mynextapp
npm install
npm run build    # Creates .next/ folder

# Environment
nano .env.local
# DATABASE_URL=...
# NEXT_PUBLIC_API_URL=...

# Start with PM2
pm2 start npm --name "mynextapp" -- start
# OR with custom port:
# PORT=3001 pm2 start npm --name "mynextapp" -- start
pm2 startup
pm2 save
```

### Nginx Config
```nginx
# /etc/nginx/sites-available/mynextapp
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

    # Next.js static assets
    location /_next/static {
        proxy_pass http://localhost:3000;
        expires 1y;
        add_header Cache-Control "public, immutable";
    }

    # Next.js image optimization
    location /_next/image {
        proxy_pass http://localhost:3000;
    }
}
```

### Redeploy Steps
```bash
ssh deploy@server
cd /var/www/mynextapp
git pull origin main
npm install
npm run build           # MUST rebuild for SSR changes
pm2 restart mynextapp
pm2 logs mynextapp --lines 20
```

### Next.js Specific Decisions
```
Static Export (next export):
  - Output: Pure static files (like React)
  - No SSR, no API routes
  - Deploy like React (Nginx static)
  - Good for: Marketing sites, blogs without server features

SSR (default Next.js):
  - Output: Node.js server + .next folder
  - Has SSR, API routes, middleware
  - Deploy like Node.js (PM2 + Nginx proxy)
  - Good for: Apps needing server-side features

Standalone Output (output: 'standalone' in next.config.js):
  - Smaller deployment footprint
  - Better for containers
  - Copies only needed node_modules
```

---

## 4. WordPress

### Architecture
```
Client -> Nginx:443 -> PHP-FPM -> WordPress -> MySQL/MariaDB
```

### Full Setup
```bash
# Install dependencies
sudo apt install nginx mysql-server php8.2-fpm php8.2-mysql \
  php8.2-xml php8.2-curl php8.2-mbstring php8.2-zip php8.2-gd \
  php8.2-intl php8.2-soap php8.2-imagick -y

# Secure MySQL
sudo mysql_secure_installation

# Create database
sudo mysql
CREATE DATABASE wordpress;
CREATE USER 'wpuser'@'localhost' IDENTIFIED BY 'strong-password-here';
GRANT ALL PRIVILEGES ON wordpress.* TO 'wpuser'@'localhost';
FLUSH PRIVILEGES;
EXIT;

# Download WordPress
cd /var/www
sudo wget https://wordpress.org/latest.tar.gz
sudo tar -xzf latest.tar.gz
sudo rm latest.tar.gz
sudo chown -R www-data:www-data wordpress

# Configure WordPress
cd wordpress
sudo cp wp-config-sample.php wp-config.php
sudo nano wp-config.php
# Set: DB_NAME, DB_USER, DB_PASSWORD, DB_HOST (localhost)
# Get fresh salts from: https://api.wordpress.org/secret-key/1.1/salt/
```

### Nginx Config for WordPress
```nginx
# /etc/nginx/sites-available/wordpress
server {
    listen 80;
    server_name blog.example.com;

    root /var/www/wordpress;
    index index.php index.html;

    # Max upload size (for media uploads)
    client_max_body_size 64M;

    location / {
        try_files $uri $uri/ /index.php?$args;
    }

    # PHP processing
    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php8.2-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }

    # Deny access to sensitive files
    location ~ /\.ht {
        deny all;
    }
    location = /wp-config.php {
        deny all;
    }
    location ~* /wp-includes/.*.php$ {
        deny all;
    }

    # Cache static assets
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2)$ {
        expires 30d;
        add_header Cache-Control "public";
    }
}
```

### PHP-FPM Tuning (Important for Performance)
```bash
sudo nano /etc/php/8.2/fpm/pool.d/www.conf
```
```ini
# For 2GB RAM server:
pm = dynamic
pm.max_children = 10
pm.start_servers = 3
pm.min_spare_servers = 2
pm.max_spare_servers = 5
pm.max_requests = 500
```
```bash
sudo systemctl restart php8.2-fpm
```

### PHP.ini Tweaks
```bash
sudo nano /etc/php/8.2/fpm/php.ini
```
```ini
upload_max_filesize = 64M
post_max_size = 64M
memory_limit = 256M
max_execution_time = 300
```

### WordPress Security
```bash
# File permissions
sudo find /var/www/wordpress -type d -exec chmod 755 {} \;
sudo find /var/www/wordpress -type f -exec chmod 644 {} \;
sudo chmod 600 /var/www/wordpress/wp-config.php

# Disable file editing in dashboard
# Add to wp-config.php:
# define('DISALLOW_FILE_EDIT', true);

# Auto updates for security patches
# Add to wp-config.php:
# define('WP_AUTO_UPDATE_CORE', 'minor');
```

### WordPress Backup
```bash
# Database backup
mysqldump -u wpuser -p wordpress > wordpress-backup-$(date +%Y%m%d).sql

# Files backup
tar -czf wordpress-files-$(date +%Y%m%d).tar.gz /var/www/wordpress

# Restore database
mysql -u wpuser -p wordpress < wordpress-backup-20240115.sql
```

---

## 5. Custom PHP Application

### Architecture
```
Client -> Nginx:443 -> PHP-FPM -> Your PHP App -> MySQL/PostgreSQL
```

### Nginx Config
```nginx
server {
    listen 80;
    server_name phpapp.example.com;

    root /var/www/phpapp/public;    # Note: public/ directory (for Laravel, Symfony)
    index index.php index.html;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php8.2-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }

    location ~ /\. {
        deny all;
    }
}
```

### Laravel Specific
```bash
cd /var/www
git clone https://github.com/you/laravel-app.git mylaravelapp
cd mylaravelapp

# Install dependencies
composer install --no-dev --optimize-autoloader

# Environment
cp .env.example .env
nano .env
# Set: APP_ENV=production, APP_DEBUG=false, DB settings

# Generate key
php artisan key:generate

# Migrations
php artisan migrate --force

# Cache config for production
php artisan config:cache
php artisan route:cache
php artisan view:cache

# Permissions
sudo chown -R deploy:www-data storage bootstrap/cache
sudo chmod -R 775 storage bootstrap/cache
```

---

## Stack Decision Matrix

```
Q: What type of project?

Static website / Landing page
  --> React build + Nginx static
  --> Cheapest, simplest, fastest

Blog / Content site with CMS
  --> WordPress + Nginx + PHP-FPM + MySQL
  --> Non-technical users can edit content

API Backend only
  --> Node.js + PM2 + Nginx proxy
  --> or PHP/Laravel + Nginx + PHP-FPM

Full-stack app with SSR
  --> Next.js + PM2 + Nginx proxy
  --> Best SEO + dynamic features

Frontend + Backend (separate)
  --> React (static on Nginx) + Node.js API (PM2 + Nginx proxy)
  --> Can be same or different servers
  --> Same server: different Nginx server blocks, different ports

WordPress + Custom API
  --> WordPress on PHP-FPM + Node.js API on PM2
  --> Same server, different Nginx server blocks
```

---

## Running Multiple Stacks on One Server

### Port Allocation Convention
```
Port 3000: Primary Node.js app
Port 3001: Secondary Node.js app / Next.js
Port 4000: Another API
Port 8080: Staging/testing

PHP: Uses unix socket, no port needed
  /run/php/php8.2-fpm.sock
```

### Example: WordPress + Node API + React on One Server
```
Nginx:
  blog.example.com    -> PHP-FPM (WordPress)
  api.example.com     -> localhost:3000 (Node.js API)
  app.example.com     -> /var/www/react-app/build (static files)

All sharing:
  - Same Nginx instance
  - Same SSL (certbot handles each domain)
  - Same server resources (watch RAM usage!)
```

### When to Split to Separate Servers
- One app is consuming too many resources
- Different security requirements
- Need to scale independently
- Different teams managing different apps
- Total RAM usage > 80% consistently
