---
layout: default
title: Debugging & Security
---

# 05 - Debugging & Security

## Part 1: Systematic Debugging

### The Debugging Mindset
```
Don't Google randomly. Follow the layers.
Traffic flows: DNS -> Network -> Firewall -> Nginx -> App -> Database
Debug in this order.
```

---

### Master Debugging Flowchart

```
PROBLEM: Site not loading / error
          |
          v
Step 1: Can you reach the server?
  ping server-ip
  ├── NO  -> Server down? Check cloud provider dashboard
  │         Wrong IP? Check DNS: dig example.com +short
  └── YES -> Continue
          |
Step 2: Is the port open?
  curl -I http://server-ip
  ├── Connection refused -> Firewall issue
  │   sudo ufw status
  │   Check cloud security group
  └── Response received -> Continue
          |
Step 3: Is Nginx running?
  sudo systemctl status nginx
  ├── NOT running -> sudo systemctl start nginx
  │   Still fails? -> sudo nginx -t (check config)
  │   Config error? -> Fix config, then start
  └── Running -> Continue
          |
Step 4: Is your app running?
  curl localhost:3000 (or your app port)
  ├── Connection refused -> App not running
  │   pm2 list / systemctl status myapp
  │   Check app logs: pm2 logs / journalctl -u myapp
  └── Response received -> Continue
          |
Step 5: Is Nginx forwarding correctly?
  curl -I http://example.com
  ├── 502 Bad Gateway -> Nginx can't reach app
  │   Check proxy_pass port matches app port
  │   Check app is bound to 0.0.0.0 or localhost
  ├── 404 -> Wrong Nginx root or location
  ├── 403 -> Permission issue on files
  └── 200 -> Working! Issue might be elsewhere
```

---

### Log Locations (Know These by Heart)

| Service | Log Location | Command |
|---------|-------------|---------|
| Nginx access | `/var/log/nginx/access.log` | `sudo tail -f /var/log/nginx/access.log` |
| Nginx errors | `/var/log/nginx/error.log` | `sudo tail -f /var/log/nginx/error.log` |
| PM2 logs | `~/.pm2/logs/` | `pm2 logs appname` |
| systemd service | journald | `journalctl -u myapp -f` |
| PHP-FPM | `/var/log/php8.2-fpm.log` | `sudo tail -f /var/log/php8.2-fpm.log` |
| MySQL | `/var/log/mysql/error.log` | `sudo tail -f /var/log/mysql/error.log` |
| System auth | `/var/log/auth.log` | `sudo tail -f /var/log/auth.log` |
| System general | `/var/log/syslog` | `sudo tail -f /var/log/syslog` |

### Log Commands
```bash
# Follow log in real-time (-f = follow)
sudo tail -f /var/log/nginx/error.log

# Show last 100 lines
sudo tail -100 /var/log/nginx/error.log

# Search for errors
sudo grep -i error /var/log/nginx/error.log | tail -20

# Filter PM2 logs by app
pm2 logs myapp --lines 50

# See systemd logs for last hour
journalctl -u myapp --since "1 hour ago"

# See all logs with errors from today
journalctl --since today | grep -i error
```

---

### Common Error Scenarios & Solutions

#### 502 Bad Gateway
```
MEANING: Nginx received request but your app didn't respond.

CAUSES:
1. App crashed / not running
2. App on wrong port
3. App taking too long to respond

DEBUG:
  pm2 list                           # Is app running?
  pm2 logs myapp                     # Any crash errors?
  curl localhost:3000                 # Does app respond locally?
  grep proxy_pass /etc/nginx/sites-enabled/myapp  # Correct port?

FIX:
  pm2 restart myapp                  # If crashed
  Fix port in Nginx config           # If port mismatch
  Increase proxy_read_timeout        # If timeout issue
```

#### 504 Gateway Timeout
```
MEANING: App is running but took too long to respond.

CAUSES:
1. Heavy database query
2. External API call timing out
3. Proxy timeout too low

FIX:
  # Increase Nginx timeout
  proxy_read_timeout 120s;
  proxy_send_timeout 120s;

  # Fix the actual slow query/call in your app code
```

#### 403 Forbidden
```
MEANING: Nginx found the file but can't read it.

CAUSES:
1. Wrong file permissions
2. Wrong file ownership
3. Nginx user can't access directory

DEBUG:
  ls -la /var/www/myapp/             # Check ownership
  namei -l /var/www/myapp/index.html # Check full path permissions

FIX:
  sudo chown -R deploy:deploy /var/www/myapp
  sudo chmod -R 755 /var/www/myapp
```

#### EACCES / Permission Denied (Node.js)
```
MEANING: Node.js can't access a file or port.

COMMON CASES:
1. Trying to listen on port 80 (need root for ports < 1024)
   FIX: Use port 3000+ and Nginx reverse proxy

2. Can't write to log/upload directory
   FIX: sudo chown -R deploy:deploy /var/www/myapp/uploads

3. node_modules permission issues
   FIX: rm -rf node_modules && npm install (as deploy user, NOT sudo)
```

#### MySQL "Access denied" / "Can't connect"
```
DEBUG:
  sudo systemctl status mysql        # Is MySQL running?
  mysql -u youruser -p               # Can you login?

FIX:
  # Reset password
  sudo mysql
  ALTER USER 'youruser'@'localhost' IDENTIFIED BY 'newpassword';
  FLUSH PRIVILEGES;

  # Check app .env has correct credentials
```

#### "npm ERR! ENOMEM" / Server Freezes During Build
```
MEANING: Not enough RAM for npm install or npm run build.

FIX:
  # Add swap (if not already)
  sudo fallocate -l 2G /swapfile
  sudo chmod 600 /swapfile
  sudo mkswap /swapfile
  sudo swapon /swapfile

  # Or build locally and upload:
  # Local: npm run build
  # Then: rsync -avz ./build/ deploy@server:/var/www/myapp/build/
```

---

### Resource Debugging
```bash
# RAM usage
free -h
# If "available" is very low, you need more RAM or swap

# Disk usage
df -h
# If / is > 90%, clean up:
sudo apt autoremove
sudo journalctl --vacuum-time=3d
pm2 flush   # Clear PM2 logs
sudo truncate -s 0 /var/log/nginx/access.log

# CPU usage
top           # Press 'q' to quit
htop          # Better UI (install: apt install htop)
# Look for processes using > 50% CPU consistently

# What's using the most RAM?
ps aux --sort=-%mem | head -10

# What's using a specific port?
sudo lsof -i :3000
sudo ss -tlnp | grep 3000

# Check disk I/O
iostat -x 1   # Install: apt install sysstat
```

---

## Part 2: Security Hardening

### Security Layers (Defense in Depth)

```
Layer 1: CLOUD FIREWALL
  - AWS Security Groups / DO Cloud Firewall
  - Only allow 22, 80, 443
  - SSH only from your IP

Layer 2: OS FIREWALL (ufw)
  - Second line of defense
  - Same rules as cloud firewall

Layer 3: SSH HARDENING
  - Key-only authentication
  - No root login
  - Non-standard port (optional)
  - Fail2ban

Layer 4: SYSTEM HARDENING
  - Auto security updates
  - Non-root user for apps
  - Minimal installed software

Layer 5: WEB SERVER HARDENING
  - Hide version info
  - Rate limiting
  - Security headers

Layer 6: APPLICATION SECURITY
  - Input validation
  - Parameterized queries (prevent SQL injection)
  - CORS configuration
  - Rate limiting
  - Environment variables (no hardcoded secrets)

Layer 7: MONITORING
  - Log monitoring
  - Intrusion detection
  - Uptime monitoring
```

### SSH Hardening
```bash
sudo nano /etc/ssh/sshd_config
```
```
# Essential settings
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
MaxAuthTries 3
LoginGraceTime 30
AllowUsers deploy            # Only allow specific users

# Optional: Change SSH port (adds obscurity, not security)
# Port 2222
# If you change port: sudo ufw allow 2222/tcp BEFORE restarting sshd
```
```bash
sudo systemctl restart sshd

# Test in new terminal BEFORE closing current session!
ssh deploy@server
```

### Nginx Security Headers
```nginx
# Add to server block or http block in nginx.conf
add_header X-Frame-Options "SAMEORIGIN" always;
add_header X-Content-Type-Options "nosniff" always;
add_header X-XSS-Protection "1; mode=block" always;
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
add_header Permissions-Policy "camera=(), microphone=(), geolocation=()" always;

# Hide Nginx version
# In /etc/nginx/nginx.conf, inside http block:
server_tokens off;
```

### Rate Limiting with Nginx
```nginx
# In http block (/etc/nginx/nginx.conf):
limit_req_zone $binary_remote_addr zone=general:10m rate=10r/s;
limit_req_zone $binary_remote_addr zone=login:10m rate=1r/s;

# In server block:
location / {
    limit_req zone=general burst=20 nodelay;
    proxy_pass http://localhost:3000;
}

location /api/login {
    limit_req zone=login burst=3 nodelay;
    proxy_pass http://localhost:3000;
}
```

### Fail2ban Configuration
```bash
sudo apt install fail2ban
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
sudo nano /etc/fail2ban/jail.local
```
```ini
[DEFAULT]
bantime  = 3600        # Ban for 1 hour
findtime = 600         # Within 10 minutes
maxretry = 5           # After 5 attempts

[sshd]
enabled = true
port    = ssh
filter  = sshd
logpath = /var/log/auth.log
maxretry = 3

[nginx-http-auth]
enabled = true
port    = http,https
filter  = nginx-http-auth
logpath = /var/log/nginx/error.log
```
```bash
sudo systemctl restart fail2ban
sudo fail2ban-client status           # Check status
sudo fail2ban-client status sshd      # Check SSH bans
```

### Automatic Security Updates
```bash
sudo apt install unattended-upgrades
sudo dpkg-reconfigure -plow unattended-upgrades

# Verify it's configured:
cat /etc/apt/apt.conf.d/20auto-upgrades
# Should show:
# APT::Periodic::Update-Package-Lists "1";
# APT::Periodic::Unattended-Upgrade "1";
```

### Database Security
```bash
# MySQL / MariaDB
sudo mysql_secure_installation
# - Set root password
# - Remove anonymous users
# - Disallow root login remotely
# - Remove test database

# NEVER expose database port to internet
# Database should ONLY listen on localhost

# Check MySQL binding:
sudo grep bind-address /etc/mysql/mysql.conf.d/mysqld.cnf
# Should be: bind-address = 127.0.0.1
```

### Secrets Management
```
NEVER DO:
  - Hardcode passwords/keys in code
  - Commit .env files to git
  - Use same password everywhere
  - Store secrets in plain text logs

ALWAYS DO:
  - Use .env files on server (chmod 600)
  - Add .env to .gitignore
  - Use different passwords per environment
  - Rotate secrets periodically

.gitignore must include:
  .env
  .env.local
  .env.production
  *.pem
  *.key
```

### Security Audit Commands
```bash
# Check open ports
sudo ss -tlnp
# Only expected ports should be listening

# Check failed SSH attempts
sudo grep "Failed password" /var/log/auth.log | tail -20

# Check running processes
ps aux | grep -v "^root" | grep -v "^deploy"
# Look for anything unexpected

# Check for rootkits (install: apt install rkhunter)
sudo rkhunter --check

# Check file permissions
find /var/www -perm -o+w -type f   # World-writable files (bad!)
find /var/www -perm -o+w -type d   # World-writable directories (bad!)

# Last logins
last -10
lastb -10   # Failed logins

# Check sudo usage
sudo grep sudo /var/log/auth.log | tail -20
```

### Backup Security
```bash
# Automate daily backups
# Create: /home/deploy/backup.sh
#!/bin/bash
DATE=$(date +%Y%m%d)
BACKUP_DIR="/home/deploy/backups"
mkdir -p $BACKUP_DIR

# Database backup
mysqldump -u dbuser -p'password' dbname > $BACKUP_DIR/db-$DATE.sql

# Files backup
tar -czf $BACKUP_DIR/files-$DATE.tar.gz /var/www/myapp --exclude='node_modules'

# Keep only last 7 days
find $BACKUP_DIR -mtime +7 -delete

echo "Backup completed: $DATE"
```
```bash
chmod +x /home/deploy/backup.sh

# Add to crontab (runs daily at 2 AM)
crontab -e
0 2 * * * /home/deploy/backup.sh >> /home/deploy/backup.log 2>&1
```

---

## Quick Security Checklist

### Before Going Live
- [ ] SSH: Key auth only, no root login
- [ ] Firewall: ufw enabled, only 22/80/443 open
- [ ] Fail2ban: Installed and running
- [ ] Auto-updates: Enabled
- [ ] SSL: Certbot configured, auto-renewal tested
- [ ] Non-root user: App runs as `deploy`, not `root`
- [ ] Database: Not exposed to internet, strong password
- [ ] .env: Not in git, proper permissions (600)
- [ ] Nginx: server_tokens off, security headers added
- [ ] Backups: Automated and tested
