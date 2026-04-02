---
layout: default
title: Runtime Layer
---

# 15 - Runtime Layer (Layer 4) - Deep Dive

> The language platform that actually executes your code on the server.

---

## What is a Runtime?

```
Your code is just text files. It can't do anything on its own.
The RUNTIME is the engine that reads and executes your code.

Code (app.js) --> Runtime (Node.js) --> Running Process (PID 1234, port 3000)

Different stacks need different runtimes:
- Node.js apps     --> Node.js runtime
- PHP/WordPress    --> PHP-FPM runtime
- Python apps      --> Python runtime
- React (static)   --> No runtime needed (just HTML/CSS/JS files served by Nginx)
- Next.js          --> Node.js runtime (it's a Node app)
```

---

## Node.js Runtime

### Why Use nvm (Never apt install node)

```
apt install nodejs gives you:
- Old version (often Node 12 or 14 on Ubuntu)
- System-wide install (permission headaches)
- Hard to switch versions between projects
- Needs sudo for global npm packages

nvm gives you:
- Any version you want
- Per-user install (no sudo needed)
- Switch versions instantly
- Multiple versions side by side
```

### Complete nvm + Node Setup

```bash
# Step 1: Install nvm
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash

# Step 2: Load nvm (or open new terminal)
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"

# Step 3: Install Node LTS
nvm install --lts          # Latest LTS (recommended)
nvm install 20             # Specific major version
nvm install 20.11.0        # Exact version

# Step 4: Set default
nvm alias default 20       # Use Node 20 by default
nvm use default

# Step 5: Verify
node -v      # v20.x.x
npm -v       # 10.x.x
which node   # /home/deploy/.nvm/versions/node/v20.x.x/bin/node
```

### nvm Commands You Need
```bash
nvm ls                    # List installed versions
nvm ls-remote             # List all available versions
nvm install 22            # Install Node 22
nvm use 22                # Switch to Node 22
nvm alias default 22      # Make Node 22 the default
nvm current               # Show current version
nvm uninstall 18          # Remove Node 18
```

### Node.js Version Strategy
```
RULE: Always use the EVEN-numbered LTS versions on servers.

Node 18 LTS  -- Maintenance (end April 2025)
Node 20 LTS  -- Active LTS (recommended as of 2024)
Node 22 LTS  -- Current LTS (2025+)

NEVER use odd versions (19, 21, 23) on servers -- they're not LTS.
Check: https://nodejs.org/en/about/releases/
```

### npm on Servers
```bash
# Install production dependencies only (skip devDependencies)
npm ci                        # Clean install from lock file (preferred)
npm install --omit=dev        # Alternative

# Why npm ci over npm install?
# npm ci:
#   - Deletes node_modules first (clean slate)
#   - Installs EXACTLY what's in package-lock.json
#   - Faster and reproducible
#   - Fails if package-lock.json is out of sync

# Global packages (installed for your user via nvm, no sudo needed)
npm install -g pm2            # Process manager
npm install -g npm            # Update npm itself
```

### Node.js Memory Configuration
```bash
# Default: Node uses ~1.5GB max heap
# On small servers (1-2GB RAM), limit it:

# Method 1: Environment variable
export NODE_OPTIONS="--max-old-space-size=512"

# Method 2: In PM2 ecosystem file
module.exports = {
  apps: [{
    name: 'myapp',
    script: 'app.js',
    node_args: '--max-old-space-size=512',
  }]
};

# Sizing guide:
# 1GB server  -> 512MB for Node
# 2GB server  -> 1024MB for Node
# 4GB server  -> 2048MB for Node (or use cluster mode)

# Check current memory usage
node -e "console.log(process.memoryUsage())"
```

---

## PHP-FPM Runtime

### What is PHP-FPM?
```
FPM = FastCGI Process Manager

Old way:  Apache loads PHP for EVERY request (slow, wasteful)
New way:  PHP-FPM keeps PHP processes ready and waiting (fast)

Nginx --> (FastCGI) --> PHP-FPM --> Processes PHP --> Returns HTML
```

### Complete PHP Setup
```bash
# Add PHP repository (for latest versions)
sudo add-apt-repository ppa:ondrej/php -y
sudo apt update

# Install PHP-FPM with common extensions
sudo apt install php8.3-fpm \
    php8.3-mysql \       # MySQL/MariaDB support
    php8.3-xml \         # XML parsing
    php8.3-curl \        # HTTP requests
    php8.3-mbstring \    # Multi-byte strings
    php8.3-zip \         # ZIP handling
    php8.3-gd \          # Image processing
    php8.3-intl \        # Internationalization
    php8.3-bcmath \      # Math functions
    php8.3-opcache \     # Performance (bytecode cache)
    php8.3-redis \       # Redis support (optional)
    -y

# Verify
php -v
sudo systemctl status php8.3-fpm
```

### PHP-FPM Configuration
```bash
# Main pool config
sudo nano /etc/php/8.3/fpm/pool.d/www.conf
```

```ini
; Key settings to tune:

; Process manager type
pm = dynamic                  ; dynamic | static | ondemand

; For DYNAMIC (recommended for most servers):
pm.max_children = 10          ; Max PHP processes
pm.start_servers = 3          ; Start with 3 processes
pm.min_spare_servers = 2      ; Keep at least 2 idle
pm.max_spare_servers = 5      ; Keep at most 5 idle
pm.max_requests = 500         ; Restart process after 500 requests (prevents memory leaks)

; Memory guide for pm.max_children:
; Each PHP process uses ~30-50MB
; Formula: (Available RAM - Other Services) / 40MB
; 1GB server: max_children = 5-8
; 2GB server: max_children = 10-15
; 4GB server: max_children = 25-40

; Socket (Nginx connects to this)
listen = /run/php/php8.3-fpm.sock
listen.owner = www-data
listen.group = www-data
```

### PHP Settings for Production
```bash
sudo nano /etc/php/8.3/fpm/php.ini
```

```ini
; Production settings
display_errors = Off              ; Never show errors to users
log_errors = On                   ; Log errors instead
error_log = /var/log/php/error.log
error_reporting = E_ALL & ~E_DEPRECATED & ~E_STRICT

; Upload limits (important for WordPress)
upload_max_filesize = 64M
post_max_size = 64M
max_execution_time = 300          ; 5 minutes max
memory_limit = 256M               ; Per-process memory limit

; OPcache (big performance boost)
opcache.enable = 1
opcache.memory_consumption = 128
opcache.max_accelerated_files = 10000
opcache.validate_timestamps = 0   ; Don't check for file changes (faster)
                                   ; Set to 1 during development
```

```bash
# Create log directory
sudo mkdir -p /var/log/php
sudo chown www-data:www-data /var/log/php

# Restart PHP-FPM after config changes
sudo systemctl restart php8.3-fpm

# Check for config errors
sudo php-fpm8.3 -t
```

### Nginx + PHP-FPM Connection
```nginx
# In your Nginx server block:
server {
    listen 80;
    server_name example.com;
    root /var/www/mysite;
    index index.php index.html;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    # Pass PHP files to PHP-FPM
    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php8.3-fpm.sock;

        # Performance tuning
        fastcgi_buffer_size 128k;
        fastcgi_buffers 256 16k;
        fastcgi_busy_buffers_size 256k;
        fastcgi_temp_file_write_size 256k;
    }

    # Block access to .ht files
    location ~ /\.ht {
        deny all;
    }
}
```

---

## MySQL / MariaDB (Database Runtime)

### When You Need It
```
WordPress  --> ALWAYS needs MySQL/MariaDB
PHP apps   --> Usually needs a database
Node.js    --> Might use MySQL, PostgreSQL, or MongoDB
```

### Installation
```bash
# Option 1: MySQL
sudo apt install mysql-server -y

# Option 2: MariaDB (MySQL-compatible, community-driven)
sudo apt install mariadb-server -y

# Secure installation (ALWAYS run this)
sudo mysql_secure_installation
# Answer: Set root password, Remove anonymous users, Disallow root remote login,
#         Remove test database, Reload privileges --> YES to all
```

### Create Database and User
```bash
sudo mysql

# In MySQL prompt:
CREATE DATABASE myapp_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'myapp_user'@'localhost' IDENTIFIED BY 'strong-password-here';
GRANT ALL PRIVILEGES ON myapp_db.* TO 'myapp_user'@'localhost';
FLUSH PRIVILEGES;
EXIT;

# Test connection
mysql -u myapp_user -p myapp_db
```

### MySQL Performance Basics
```bash
sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf
```

```ini
# Key settings:
[mysqld]
# InnoDB buffer pool (most important setting)
# Set to 50-70% of available RAM for database servers
# For shared servers: 256M-512M
innodb_buffer_pool_size = 512M

# Query cache (disable on MySQL 8+, use app-level caching)
# On MariaDB, it can still help for read-heavy WordPress:
query_cache_type = 1
query_cache_size = 64M

# Max connections
max_connections = 50        # 1GB server
# max_connections = 100     # 2GB+ server

# Logging slow queries
slow_query_log = 1
long_query_time = 2         # Log queries taking more than 2 seconds
slow_query_log_file = /var/log/mysql/slow.log
```

---

## Multiple PHP Versions (Side by Side)

```bash
# Scenario: One server runs WordPress (PHP 8.1) and a new app (PHP 8.3)

# Install both versions
sudo apt install php8.1-fpm php8.3-fpm -y

# Each has its own socket:
# /run/php/php8.1-fpm.sock
# /run/php/php8.3-fpm.sock

# Nginx: Point each site to its PHP version
# Site 1 (WordPress):
location ~ \.php$ {
    fastcgi_pass unix:/run/php/php8.1-fpm.sock;
}

# Site 2 (New app):
location ~ \.php$ {
    fastcgi_pass unix:/run/php/php8.3-fpm.sock;
}

# Switch CLI default:
sudo update-alternatives --set php /usr/bin/php8.3
```

---

## Python Runtime (Bonus)

```bash
# Install pyenv (like nvm for Python)
curl https://pyenv.run | bash

# Add to ~/.bashrc:
export PYENV_ROOT="$HOME/.pyenv"
export PATH="$PYENV_ROOT/bin:$PATH"
eval "$(pyenv init -)"

# Install Python
pyenv install 3.12
pyenv global 3.12

# Virtual environment (always use one)
python -m venv /var/www/myapp/venv
source /var/www/myapp/venv/bin/activate
pip install -r requirements.txt

# Gunicorn (Python's process manager, like PM2 for Node)
pip install gunicorn
gunicorn --workers 3 --bind 0.0.0.0:8000 app:app
```

---

## Runtime Troubleshooting

### Node.js Issues
```bash
# "command not found: node" after installing nvm
source ~/.bashrc                   # Reload shell
# Or: export NVM_DIR="$HOME/.nvm" && [ -s "$NVM_DIR/nvm.sh" ] && . "$NVM_DIR/nvm.sh"

# "permission denied" on npm install -g
# You're probably using system Node, not nvm Node
which node                         # Should be in ~/.nvm, NOT /usr/bin

# "ENOSPC: no space left on device" (not always about disk)
echo fs.inotify.max_user_watches=524288 | sudo tee -a /etc/sysctl.conf
sudo sysctl -p

# "JavaScript heap out of memory"
export NODE_OPTIONS="--max-old-space-size=2048"
npm run build                      # Try again
```

### PHP Issues
```bash
# "502 Bad Gateway" with Nginx + PHP
# PHP-FPM is not running or socket mismatch
sudo systemctl status php8.3-fpm   # Check if running
ls -la /run/php/                    # Check socket exists
# Make sure Nginx fastcgi_pass matches the actual socket

# "File not found" on PHP pages
# Wrong root directory in Nginx config
# Check: root /var/www/mysite; matches actual file location

# "Allowed memory size exhausted"
# Increase in php.ini:
memory_limit = 512M
sudo systemctl restart php8.3-fpm

# Check which php.ini is loaded
php --ini
php -i | grep "Loaded Configuration File"
```

### MySQL Issues
```bash
# "Can't connect to MySQL server"
sudo systemctl status mysql         # Is it running?
sudo systemctl start mysql          # Start it

# "Access denied for user"
sudo mysql                          # Login as root
ALTER USER 'myapp_user'@'localhost' IDENTIFIED BY 'new-password';
FLUSH PRIVILEGES;

# "Too many connections"
# Increase in my.cnf or check for connection leaks in your app
SHOW PROCESSLIST;                   # See active connections
```

---

## Quick Reference: Runtime Commands

```bash
# --- NODE.JS ---
node -v                            # Node version
npm -v                             # npm version
nvm ls                             # Installed Node versions
nvm use 20                         # Switch to Node 20
npm ci                             # Clean install from lock file
pm2 list                           # Running Node processes

# --- PHP ---
php -v                             # PHP version
php -m                             # Installed PHP modules
sudo systemctl status php8.3-fpm   # PHP-FPM status
sudo php-fpm8.3 -t                 # Test PHP-FPM config
php --ini                          # Config file locations

# --- MYSQL ---
sudo systemctl status mysql         # MySQL status
mysql -u user -p database           # Connect to database
mysqldump -u user -p db > backup.sql  # Backup database
mysql -u user -p db < backup.sql      # Restore database
```
