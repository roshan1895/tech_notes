---
layout: default
title: Web Server Layer
---

# 16 - Web Server Layer (Layer 5) - Deep Dive

> Nginx/Apache: the gatekeeper between the internet and your application.

---

## What a Web Server Does

```
Without Nginx:
  Internet --> Your app on port 3000 (no SSL, no static file optimization, exposed)

With Nginx:
  Internet --> Nginx (port 80/443)
                |-- Terminates SSL (HTTPS)
                |-- Serves static files directly (fast, no app overhead)
                |-- Reverse proxies to your app (localhost:3000)
                |-- Rate limits, caches, load balances
                |-- Adds security headers
                V
              Your app (port 3000, only accessible from localhost)
```

---

## Nginx vs Apache: When to Use What

```
+------------------+---------------------------+---------------------------+
| Feature          | Nginx                     | Apache                    |
+------------------+---------------------------+---------------------------+
| Architecture     | Event-driven (async)      | Process/thread per request|
| Concurrency      | Handles 10K+ connections  | Struggles at high load    |
| Static files     | Excellent                 | Good                      |
| Config reload    | Zero downtime             | May need restart          |
| .htaccess        | Not supported             | Supported                 |
| Reverse proxy    | Built-in, fast            | Needs mod_proxy           |
| PHP              | Via PHP-FPM (FastCGI)     | Via mod_php (built-in)    |
| Memory           | Low (fixed workers)       | High (per-connection)     |
+------------------+---------------------------+---------------------------+

RECOMMENDATION:
  Nginx = Default choice for everything
  Apache = Only if you NEED .htaccess (some WordPress plugins require it)
  Both = Nginx as reverse proxy in front of Apache (advanced)
```

---

## Nginx Installation and Structure

### Install
```bash
sudo apt install nginx -y
sudo systemctl enable nginx
sudo systemctl start nginx

# Verify
sudo systemctl status nginx
curl http://localhost           # Should show Nginx welcome page
nginx -v                       # Version
```

### Nginx File Structure
```
/etc/nginx/
├── nginx.conf                 # Main config (global settings)
├── sites-available/           # All site configs (inactive)
│   ├── default                # Default welcome page
│   ├── myapp                  # Your app config
│   └── wordpress              # Another site config
├── sites-enabled/             # Active sites (symlinks to sites-available)
│   ├── default -> ../sites-available/default
│   └── myapp -> ../sites-available/myapp
├── snippets/                  # Reusable config fragments
│   ├── security-headers.conf
│   └── fastcgi-php.conf
├── conf.d/                    # Additional configs (auto-loaded)
└── mime.types                 # File type definitions

/var/log/nginx/
├── access.log                 # All requests
└── error.log                  # Errors
```

### Workflow: Create a New Site
```bash
# 1. Create config in sites-available
sudo nano /etc/nginx/sites-available/myapp

# 2. Enable it (create symlink)
sudo ln -s /etc/nginx/sites-available/myapp /etc/nginx/sites-enabled/

# 3. Remove default site (optional, do after your site works)
sudo rm /etc/nginx/sites-enabled/default

# 4. Test config syntax
sudo nginx -t

# 5. Reload (zero downtime)
sudo systemctl reload nginx
```

---

## Nginx Main Config (nginx.conf)

```nginx
# /etc/nginx/nginx.conf

user www-data;                           # Run as www-data user
worker_processes auto;                   # One worker per CPU core
pid /run/nginx.pid;

events {
    worker_connections 1024;             # Max connections per worker
    # Total max connections = worker_processes x worker_connections
    # 4 cores x 1024 = 4096 concurrent connections
    multi_accept on;                     # Accept multiple connections at once
}

http {
    # --- BASIC SETTINGS ---
    sendfile on;                         # Efficient file sending
    tcp_nopush on;                       # Optimize packet sending
    tcp_nodelay on;                      # Don't delay small packets
    keepalive_timeout 65;                # Keep connections alive for 65s
    types_hash_max_size 2048;
    server_tokens off;                   # Hide Nginx version (security)

    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    # --- LOGGING ---
    access_log /var/log/nginx/access.log;
    error_log /var/log/nginx/error.log;

    # --- GZIP COMPRESSION ---
    gzip on;
    gzip_vary on;
    gzip_proxied any;
    gzip_comp_level 6;                   # 1-9 (6 = good balance)
    gzip_types
        text/plain
        text/css
        text/xml
        text/javascript
        application/json
        application/javascript
        application/xml
        application/rss+xml
        image/svg+xml;

    # --- RATE LIMITING (define zones) ---
    limit_req_zone $binary_remote_addr zone=general:10m rate=10r/s;
    limit_req_zone $binary_remote_addr zone=login:10m rate=1r/s;

    # --- INCLUDE SITE CONFIGS ---
    include /etc/nginx/conf.d/*.conf;
    include /etc/nginx/sites-enabled/*;
}
```

---

## Complete Configs for Every Stack

### 1. Node.js / Express API
```nginx
# /etc/nginx/sites-available/node-api

upstream node_backend {
    server 127.0.0.1:3000;
    # Add more for load balancing:
    # server 127.0.0.1:3001;
    # server 127.0.0.1:3002;
    keepalive 64;                        # Keep connections to Node alive
}

server {
    listen 80;
    server_name api.example.com;

    # Redirect to HTTPS (Certbot adds this automatically)
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name api.example.com;

    # SSL (Certbot manages these)
    ssl_certificate /etc/letsencrypt/live/api.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/api.example.com/privkey.pem;

    # Security headers
    include /etc/nginx/snippets/security-headers.conf;

    # Logging
    access_log /var/log/nginx/api.access.log;
    error_log /var/log/nginx/api.error.log;

    # Max upload size
    client_max_body_size 10M;

    location / {
        proxy_pass http://node_backend;
        proxy_http_version 1.1;

        # WebSocket support
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';

        # Forward client info
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_cache_bypass $http_upgrade;

        # Timeouts
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }
}
```

### 2. React / Static SPA
```nginx
# /etc/nginx/sites-available/react-app

server {
    listen 80;
    server_name app.example.com;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name app.example.com;

    ssl_certificate /etc/letsencrypt/live/app.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/app.example.com/privkey.pem;

    root /var/www/react-app/build;       # React build output
    index index.html;

    include /etc/nginx/snippets/security-headers.conf;

    # SPA routing: send all routes to index.html
    location / {
        try_files $uri $uri/ /index.html;
    }

    # Cache static assets aggressively
    location /static/ {
        expires 1y;
        add_header Cache-Control "public, immutable";
        access_log off;                  # Don't log static file requests
    }

    # Cache other assets
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2|ttf|eot)$ {
        expires 30d;
        add_header Cache-Control "public";
        access_log off;
    }

    # Don't serve dotfiles
    location ~ /\. {
        deny all;
    }
}
```

### 3. Next.js (SSR)
```nginx
# /etc/nginx/sites-available/nextjs-app

upstream nextjs_backend {
    server 127.0.0.1:3000;
    keepalive 64;
}

server {
    listen 80;
    server_name example.com www.example.com;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name example.com www.example.com;

    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    include /etc/nginx/snippets/security-headers.conf;

    client_max_body_size 10M;

    # Next.js static files (served directly by Nginx, bypass Node)
    location /_next/static/ {
        alias /var/www/nextjs-app/.next/static/;
        expires 1y;
        add_header Cache-Control "public, immutable";
        access_log off;
    }

    # Public folder
    location /public/ {
        alias /var/www/nextjs-app/public/;
        expires 30d;
        access_log off;
    }

    # Everything else goes to Next.js
    location / {
        proxy_pass http://nextjs_backend;
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

### 4. WordPress (Nginx)
```nginx
# /etc/nginx/sites-available/wordpress

server {
    listen 80;
    server_name blog.example.com;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name blog.example.com;

    ssl_certificate /etc/letsencrypt/live/blog.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/blog.example.com/privkey.pem;

    root /var/www/wordpress;
    index index.php index.html;

    include /etc/nginx/snippets/security-headers.conf;

    client_max_body_size 64M;           # WordPress media uploads

    # WordPress permalinks
    location / {
        try_files $uri $uri/ /index.php?$args;
    }

    # PHP processing
    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php8.3-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;

        fastcgi_buffer_size 128k;
        fastcgi_buffers 256 16k;
        fastcgi_busy_buffers_size 256k;
        fastcgi_temp_file_write_size 256k;
    }

    # Cache static assets
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2)$ {
        expires 30d;
        add_header Cache-Control "public";
        access_log off;
    }

    # Security: block access to sensitive files
    location = /xmlrpc.php { deny all; }
    location ~* wp-config.php { deny all; }
    location ~ /\.ht { deny all; }
    location ~ /wp-includes/.*\.php$ { deny all; }
    location ~ /wp-content/uploads/.*\.php$ { deny all; }

    # Block directory listing
    autoindex off;
}
```

### 5. PHP Application (Generic)
```nginx
# /etc/nginx/sites-available/php-app

server {
    listen 80;
    server_name phpapp.example.com;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name phpapp.example.com;

    ssl_certificate /etc/letsencrypt/live/phpapp.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/phpapp.example.com/privkey.pem;

    root /var/www/phpapp/public;         # Usually public/ is the web root
    index index.php index.html;

    include /etc/nginx/snippets/security-headers.conf;

    client_max_body_size 20M;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php8.3-fpm.sock;
    }

    location ~ /\. {
        deny all;
    }
}
```

---

## Nginx Reverse Proxy - Deep Understanding

### What Happens Step by Step
```
1. Client sends HTTPS request to example.com:443
2. Nginx receives it, decrypts SSL
3. Nginx checks server_name (which site?)
4. Nginx matches location block (which path?)
5. Nginx forwards to upstream (your app on localhost:3000)
6. Your app processes and returns response
7. Nginx sends response back to client (re-encrypts for HTTPS)

The app NEVER talks to the internet directly.
The app only sees requests from 127.0.0.1 (Nginx).
```

### Proxy Headers Explained
```nginx
# Why each header matters:

proxy_set_header Host $host;
# Without: Your app sees "Host: localhost:3000"
# With:    Your app sees "Host: example.com"

proxy_set_header X-Real-IP $remote_addr;
# Without: Your app sees "req.ip = 127.0.0.1" (Nginx's IP)
# With:    Your app sees "req.ip = 203.0.113.50" (actual client)

proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
# Chain of IPs if multiple proxies: "client, proxy1, proxy2"

proxy_set_header X-Forwarded-Proto $scheme;
# Without: Your app thinks it's HTTP (because Nginx->App is HTTP)
# With:    Your app knows the original request was HTTPS
# Important for: redirect URLs, secure cookies, CORS
```

### WebSocket Proxy
```nginx
# Required for Socket.io, WebSocket, real-time apps

proxy_http_version 1.1;                  # HTTP/1.1 required for WebSocket
proxy_set_header Upgrade $http_upgrade;  # Tell backend about upgrade
proxy_set_header Connection "upgrade";   # Switch to WebSocket protocol

# Without these: WebSocket connections fail silently
# Symptom: Socket.io falls back to polling, real-time features break
```

---

## Nginx Performance Tuning

### Gzip Compression
```nginx
# Already in nginx.conf, but here's the detail:

gzip on;
gzip_vary on;                           # Tell proxies about gzip
gzip_proxied any;                       # Compress proxied responses too
gzip_comp_level 6;                      # 1=fast/less, 9=slow/more, 6=sweet spot
gzip_min_length 1000;                   # Don't compress tiny files
gzip_types
    text/plain text/css text/xml text/javascript
    application/json application/javascript application/xml
    application/rss+xml image/svg+xml;

# DON'T gzip: images (jpg, png, gif), videos, already compressed files
# They're already compressed -- gzipping them wastes CPU
```

### Client-Side Caching
```nginx
# Tell browsers to cache static files

# Aggressive caching for fingerprinted files (React/Next.js build output)
location /static/ {
    expires 1y;                          # Cache for 1 year
    add_header Cache-Control "public, immutable";
    # "immutable" = don't even check for updates
    # Safe because build tools add hash to filename: main.abc123.js
}

# Moderate caching for regular assets
location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg)$ {
    expires 30d;
    add_header Cache-Control "public";
}

# No caching for HTML (so users get latest version)
location ~* \.html$ {
    expires -1;
    add_header Cache-Control "no-store, no-cache, must-revalidate";
}
```

### Buffering and Timeouts
```nginx
# Proxy buffering (Nginx reads full response from app, then sends to client)
proxy_buffering on;                      # Default: on
proxy_buffer_size 4k;                    # Response header buffer
proxy_buffers 8 4k;                      # Response body buffers

# For large responses (file downloads, big API responses):
proxy_buffer_size 128k;
proxy_buffers 4 256k;
proxy_busy_buffers_size 256k;

# Timeouts
proxy_connect_timeout 60s;              # Time to establish connection to app
proxy_send_timeout 60s;                 # Time to send request to app
proxy_read_timeout 60s;                 # Time to read response from app
# Increase for slow operations (file uploads, report generation):
# proxy_read_timeout 300s;
```

---

## Multiple Sites on One Server

```
Server (1 IP: 143.198.50.100)
    |
    Nginx (port 80/443)
    |
    |-- api.example.com     --> localhost:3000 (Node.js API)
    |-- app.example.com     --> /var/www/react-app/build (React)
    |-- blog.example.com    --> PHP-FPM (WordPress)
    |-- example.com         --> localhost:3001 (Next.js)
```

```bash
# Each site has its own config file:
/etc/nginx/sites-available/api
/etc/nginx/sites-available/react
/etc/nginx/sites-available/wordpress
/etc/nginx/sites-available/nextjs

# Nginx uses server_name to route:
# Request for api.example.com -> matches server_name api.example.com
# Request for blog.example.com -> matches server_name blog.example.com

# All share the same port 80/443 -- Nginx handles routing
```

---

## Apache (When You Need It)

### Install
```bash
sudo apt install apache2 -y
sudo systemctl enable apache2

# Enable needed modules
sudo a2enmod rewrite        # URL rewriting (.htaccess)
sudo a2enmod proxy          # Reverse proxy
sudo a2enmod proxy_http     # HTTP proxy
sudo a2enmod ssl            # SSL
sudo a2enmod headers        # Custom headers

sudo systemctl restart apache2
```

### Apache WordPress Config
```apache
# /etc/apache2/sites-available/wordpress.conf

<VirtualHost *:80>
    ServerName blog.example.com
    DocumentRoot /var/www/wordpress

    <Directory /var/www/wordpress>
        AllowOverride All           # Enable .htaccess
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/wordpress_error.log
    CustomLog ${APACHE_LOG_DIR}/wordpress_access.log combined
</VirtualHost>
```

```bash
# Enable site
sudo a2ensite wordpress.conf
sudo a2dissite 000-default.conf    # Disable default
sudo apache2ctl configtest          # Test config
sudo systemctl reload apache2
```

---

## Nginx Troubleshooting

### Common Errors and Fixes

```bash
# --- ERROR: 502 Bad Gateway ---
# Nginx can't connect to your app
# Fixes:
pm2 list                              # Is your app running?
sudo systemctl status php8.3-fpm      # Is PHP-FPM running?
curl http://localhost:3000             # Can you reach app directly?
# Check: proxy_pass port matches your app port

# --- ERROR: 403 Forbidden ---
# Permission issue
ls -la /var/www/myapp/                 # Check file ownership
sudo chown -R www-data:www-data /var/www/myapp  # Fix ownership
# Check: Nginx user (www-data) can read the files

# --- ERROR: 404 Not Found ---
# Wrong root or try_files
# Check: root path in config matches actual directory
# Check: index directive includes your index file
# For SPA: make sure try_files has /index.html fallback

# --- ERROR: 413 Request Entity Too Large ---
# Upload too big
# Fix: Add to server block:
client_max_body_size 64M;

# --- ERROR: upstream timed out ---
# App took too long to respond
# Fix: Increase timeouts:
proxy_read_timeout 300s;

# --- CONFIG TEST ---
sudo nginx -t                          # Always test before reload!
# Shows: "syntax is ok" or exact error line
```

### Useful Nginx Commands
```bash
sudo systemctl start nginx            # Start
sudo systemctl stop nginx             # Stop
sudo systemctl restart nginx          # Full restart (brief downtime)
sudo systemctl reload nginx           # Graceful reload (zero downtime)
sudo nginx -t                         # Test configuration
sudo nginx -T                         # Show full compiled config

# Logs
sudo tail -f /var/log/nginx/access.log
sudo tail -f /var/log/nginx/error.log
sudo tail -f /var/log/nginx/myapp.access.log    # Site-specific

# Check what's listening
sudo ss -tlnp | grep nginx            # Nginx ports
sudo ss -tlnp | grep :80              # What's on port 80
```

---

## Quick Reference

```bash
# --- SITE MANAGEMENT ---
sudo ln -s /etc/nginx/sites-available/myapp /etc/nginx/sites-enabled/   # Enable site
sudo rm /etc/nginx/sites-enabled/myapp                                    # Disable site
sudo nginx -t && sudo systemctl reload nginx                              # Test + reload

# --- DEBUGGING ---
sudo tail -f /var/log/nginx/error.log     # Watch errors live
curl -I https://example.com               # Check response headers
curl -v https://example.com               # Verbose connection info

# --- PERFORMANCE ---
# Check active connections
sudo cat /var/log/nginx/access.log | wc -l    # Total requests in log
# Real-time connections (enable stub_status module):
# location /nginx_status { stub_status; allow 127.0.0.1; deny all; }
# curl http://localhost/nginx_status
```
