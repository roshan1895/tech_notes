# 14 - Security Layer (Layer 3) - Deep Dive

> Protecting your server from unauthorized access, attacks, and data breaches.

---

## Why Security Matters

```
Every server connected to the internet gets attacked.
Within minutes of creation, bots scan for:
- Open SSH ports with default passwords
- Known vulnerabilities in outdated software
- Exposed databases and admin panels
- Default WordPress/phpMyAdmin URLs

Security isn't optional. It's Layer 3 for a reason -- set it up BEFORE deploying your app.
```

---

## The Security Onion (Layers of Defense)

```
            [Internet / Attackers]
                    |
        +-----------+-----------+
        |   CLOUD FIREWALL      |   AWS Security Groups / DO Firewall
        |   (Provider Level)    |   Block ports before they hit your server
        +-----------+-----------+
                    |
        +-----------+-----------+
        |   OS FIREWALL (ufw)   |   Controls which ports are open on the server
        +-----------+-----------+
                    |
        +-----------+-----------+
        |   FAIL2BAN            |   Blocks IPs after repeated failed attempts
        +-----------+-----------+
                    |
        +-----------+-----------+
        |   SSH HARDENING       |   Key-only auth, no root login
        +-----------+-----------+
                    |
        +-----------+-----------+
        |   SSL/TLS (HTTPS)     |   Encrypts data in transit
        +-----------+-----------+
                    |
        +-----------+-----------+
        |   APP SECURITY        |   Input validation, CORS, rate limiting
        +-----------+-----------+
```

---

## 1. Cloud Firewall (First Line of Defense)

### AWS Security Groups
```
Think of it as: a bouncer at the building entrance.
Traffic that doesn't match rules never reaches your server.
```

```
# AWS Security Group Rules (set via AWS Console or CLI)

Inbound Rules:
+-----------+----------+----------------+---------------------------+
| Type      | Port     | Source         | Why                       |
+-----------+----------+----------------+---------------------------+
| SSH       | 22       | Your IP only   | Only YOU can SSH in       |
| HTTP      | 80       | 0.0.0.0/0     | Public web traffic        |
| HTTPS     | 443      | 0.0.0.0/0     | Public web traffic (SSL)  |
| Custom    | 3000     | NOWHERE        | App port stays internal   |
+-----------+----------+----------------+---------------------------+

Outbound Rules:
+-----------+----------+----------------+---------------------------+
| Type      | Port     | Destination    | Why                       |
+-----------+----------+----------------+---------------------------+
| All       | All      | 0.0.0.0/0     | Server can reach internet |
+-----------+----------+----------------+---------------------------+
```

### DigitalOcean Firewall
```
# DO Console > Networking > Firewalls > Create Firewall

Same logic as AWS:
- Allow SSH (22) from your IP only
- Allow HTTP (80) and HTTPS (443) from all
- Block everything else inbound
```

### Key Rule
```
NEVER expose your app port (3000, 5000, 8080) to the internet directly.
Nginx handles public traffic on 80/443 and proxies to your app internally.
```

---

## 2. OS Firewall (ufw) - Server Level

### What is ufw?
```
ufw = Uncomplicated Firewall
It controls iptables (the real Linux firewall) with simple commands.
It's the second bouncer -- at your server's door.
```

### Complete ufw Setup
```bash
# Check current status
sudo ufw status verbose

# Reset to clean state (if needed)
sudo ufw reset

# Default policies: deny incoming, allow outgoing
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Allow SSH (ALWAYS DO THIS FIRST or you'll lock yourself out!)
sudo ufw allow OpenSSH
# OR: sudo ufw allow 22/tcp

# Allow web traffic
sudo ufw allow 80/tcp      # HTTP
sudo ufw allow 443/tcp     # HTTPS
# OR shortcut: sudo ufw allow 'Nginx Full'

# Enable the firewall
sudo ufw enable

# Verify
sudo ufw status numbered
```

### Common ufw Operations
```bash
# Allow specific port
sudo ufw allow 8080/tcp

# Allow from specific IP only (great for SSH)
sudo ufw allow from 203.0.113.50 to any port 22

# Deny a specific IP (block an attacker)
sudo ufw deny from 203.0.113.100

# Delete a rule by number
sudo ufw status numbered
sudo ufw delete 3

# Rate limiting on SSH (auto-blocks after 6 attempts in 30 seconds)
sudo ufw limit ssh

# Allow port range
sudo ufw allow 6000:6007/tcp

# Check firewall logs
sudo tail -f /var/log/ufw.log
```

### ufw Mistakes That Lock You Out
```
MISTAKE 1: Enabling ufw without allowing SSH first
RESULT:    You can't SSH back in. Need provider console access.
FIX:       ALWAYS run "sudo ufw allow OpenSSH" BEFORE "sudo ufw enable"

MISTAKE 2: Denying all then forgetting to allow needed ports
RESULT:    Site goes down
FIX:       Test after each rule change

MISTAKE 3: Allowing your app port publicly
RESULT:    Users can bypass Nginx (no SSL, no security headers)
FIX:       App port should ONLY be accessible via localhost
```

---

## 3. SSH Hardening

### Why Default SSH is Dangerous
```
Default SSH:
- Listens on port 22 (every bot knows this)
- Allows password auth (can be brute-forced)
- Allows root login (attacker gets full access)

You need to fix ALL THREE.
```

### Step 1: Set Up SSH Keys (On Your Local Machine)
```bash
# Generate key pair (if you don't have one)
ssh-keygen -t ed25519 -C "your-email@example.com"
# Saves to: ~/.ssh/id_ed25519 (private) and ~/.ssh/id_ed25519.pub (public)

# Copy public key to server
ssh-copy-id -i ~/.ssh/id_ed25519.pub deploy@your-server-ip

# Test key login (should not ask for password)
ssh deploy@your-server-ip
```

### Step 2: Harden SSH Config
```bash
sudo nano /etc/ssh/sshd_config
```

```
# Change these settings:

Port 2222                          # Change from default 22 (optional but recommended)
PermitRootLogin no                 # No root SSH login
PasswordAuthentication no          # Key-only (MAKE SURE your key works first!)
PubkeyAuthentication yes           # Enable key auth
MaxAuthTries 3                     # Max 3 login attempts per connection
LoginGraceTime 60                  # 60 seconds to authenticate
AllowUsers deploy                  # Only this user can SSH in
ClientAliveInterval 300            # Disconnect idle sessions after 5 min
ClientAliveCountMax 2              # After 2 missed keepalives
```

```bash
# Test config before restarting (IMPORTANT!)
sudo sshd -t

# If no errors, restart
sudo systemctl restart sshd

# If you changed the port, update firewall:
sudo ufw allow 2222/tcp
sudo ufw delete allow OpenSSH     # Remove old port 22 rule
```

### Step 3: SSH Config on Your Local Machine
```bash
# ~/.ssh/config -- makes SSH easier
Host myserver
    HostName 143.198.50.100
    User deploy
    Port 2222
    IdentityFile ~/.ssh/id_ed25519

# Now you just type:
ssh myserver
# Instead of:
ssh -p 2222 -i ~/.ssh/id_ed25519 deploy@143.198.50.100
```

### Emergency: Locked Out of SSH?
```
1. Use provider console (AWS EC2 Connect / DO Recovery Console)
2. Fix the sshd_config
3. Restart sshd
4. Test from local machine

Prevention: ALWAYS test SSH in a new terminal before closing your current session
```

---

## 4. Fail2ban - Automated Brute Force Protection

### What It Does
```
Fail2ban watches log files for suspicious activity.
After X failed attempts, it bans the IP using iptables.

Example: Someone tries 5 wrong SSH passwords -> Banned for 10 minutes
```

### Installation and Setup
```bash
# Install
sudo apt install fail2ban -y

# Create local config (never edit jail.conf directly)
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
sudo nano /etc/fail2ban/jail.local
```

### Essential Jail Configuration
```ini
# /etc/fail2ban/jail.local

[DEFAULT]
bantime  = 3600          # Ban for 1 hour (in seconds)
findtime = 600           # Look at last 10 minutes
maxretry = 5             # Ban after 5 failures
banaction = ufw          # Use ufw to ban (important!)

# Email notifications (optional)
# destemail = you@example.com
# action = %(action_mwl)s

[sshd]
enabled = true
port    = 2222           # Match your SSH port
logpath = /var/log/auth.log
maxretry = 3             # Stricter for SSH

[nginx-http-auth]
enabled = true
logpath = /var/log/nginx/error.log
maxretry = 5

[nginx-botsearch]
enabled = true
logpath = /var/log/nginx/access.log
maxretry = 2

[nginx-limit-req]
enabled = true
logpath = /var/log/nginx/error.log
maxretry = 5
```

### Fail2ban Commands
```bash
# Start and enable
sudo systemctl start fail2ban
sudo systemctl enable fail2ban

# Check status
sudo fail2ban-client status
sudo fail2ban-client status sshd     # Check specific jail

# See banned IPs
sudo fail2ban-client status sshd

# Manually unban an IP (if you accidentally banned yourself)
sudo fail2ban-client set sshd unbanip 203.0.113.50

# Check fail2ban logs
sudo tail -f /var/log/fail2ban.log

# Test regex against log (debugging)
sudo fail2ban-regex /var/log/auth.log /etc/fail2ban/filter.d/sshd.conf
```

---

## 5. SSL/TLS (HTTPS) with Certbot

### Why HTTPS is Mandatory
```
Without HTTPS:
- Passwords sent in plain text (anyone on the network can read them)
- Google penalizes HTTP sites in search rankings
- Browsers show "Not Secure" warning
- No HTTP/2 (slower)

With HTTPS:
- All data encrypted between user and server
- Green padlock / no warnings
- Better SEO
- Required for many modern APIs
```

### Certbot Setup (Let's Encrypt - Free SSL)
```bash
# Install Certbot
sudo apt install certbot python3-certbot-nginx -y

# Get certificate (Nginx plugin - easiest)
sudo certbot --nginx -d example.com -d www.example.com

# What this does:
# 1. Verifies you own the domain (via HTTP challenge)
# 2. Gets a free certificate from Let's Encrypt
# 3. Auto-configures Nginx for HTTPS
# 4. Sets up HTTP -> HTTPS redirect

# Test auto-renewal
sudo certbot renew --dry-run

# Certificates auto-renew via systemd timer
sudo systemctl status certbot.timer
```

### What Certbot Adds to Your Nginx Config
```nginx
server {
    listen 443 ssl;
    server_name example.com www.example.com;

    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

    # ... your existing config
}

# HTTP -> HTTPS redirect (auto-added)
server {
    listen 80;
    server_name example.com www.example.com;
    return 301 https://$server_name$request_uri;
}
```

### Certificate Troubleshooting
```bash
# Check certificate expiry
sudo certbot certificates

# Force renewal
sudo certbot renew --force-renewal

# Common errors:
# "Challenge failed" -> DNS not pointing to this server yet
# "Too many requests" -> Rate limited, wait 1 hour
# "Port 80 not open" -> Firewall blocking HTTP (need it for verification)
```

---

## 6. Security Headers (Nginx)

### Add to Your Nginx Server Block
```nginx
# /etc/nginx/snippets/security-headers.conf

# Prevent clickjacking
add_header X-Frame-Options "SAMEORIGIN" always;

# Prevent MIME type sniffing
add_header X-Content-Type-Options "nosniff" always;

# XSS protection
add_header X-XSS-Protection "1; mode=block" always;

# Referrer policy
add_header Referrer-Policy "strict-origin-when-cross-origin" always;

# Content Security Policy (customize per app)
# add_header Content-Security-Policy "default-src 'self';" always;

# HSTS (only enable after confirming HTTPS works!)
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
```

```nginx
# Include in your server block:
server {
    listen 443 ssl;
    server_name example.com;

    include /etc/nginx/snippets/security-headers.conf;

    # ... rest of config
}
```

### Test Your Headers
```bash
# Check headers
curl -I https://example.com

# Online tools:
# securityheaders.com
# ssllabs.com/ssltest (for SSL grade)
```

---

## 7. Application-Level Security

### Node.js / Express
```javascript
// Essential security middleware
const helmet = require('helmet');     // Security headers
const cors = require('cors');         // CORS control
const rateLimit = require('express-rate-limit');

const app = express();

// Security headers
app.use(helmet());

// CORS - restrict to your frontend domain
app.use(cors({
    origin: 'https://yourfrontend.com',
    methods: ['GET', 'POST', 'PUT', 'DELETE'],
    credentials: true
}));

// Rate limiting
const limiter = rateLimit({
    windowMs: 15 * 60 * 1000,  // 15 minutes
    max: 100,                   // 100 requests per window
    message: 'Too many requests, try again later'
});
app.use('/api/', limiter);

// Stricter limit for auth routes
const authLimiter = rateLimit({
    windowMs: 15 * 60 * 1000,
    max: 5,                     // Only 5 login attempts per 15 min
    message: 'Too many login attempts'
});
app.use('/api/auth/', authLimiter);
```

### WordPress Security
```bash
# 1. File permissions
find /var/www/wordpress -type d -exec chmod 755 {} \;
find /var/www/wordpress -type f -exec chmod 644 {} \;
chmod 400 wp-config.php

# 2. Block xmlrpc (used in brute force attacks)
# In Nginx:
location = /xmlrpc.php {
    deny all;
}

# 3. Block wp-config access
location ~* wp-config.php {
    deny all;
}

# 4. Hide WordPress version
# In functions.php:
remove_action('wp_head', 'wp_generator');

# 5. Limit login attempts
# Install plugin: "Limit Login Attempts Reloaded"

# 6. Change default admin URL (optional)
# Install plugin: "WPS Hide Login"
```

### Environment Variables Security
```bash
# .env file permissions
chmod 600 .env

# NEVER commit .env to git
echo ".env" >> .gitignore

# Check if .env is in git history (if accidentally committed)
git log --all --full-history -- .env
# If found, you need to rotate ALL secrets in that file

# Use different secrets for dev/staging/production
# Production secrets should only exist on the production server
```

---

## 8. Nginx Rate Limiting

```nginx
# In http block (/etc/nginx/nginx.conf)
http {
    # Define rate limit zones
    limit_req_zone $binary_remote_addr zone=general:10m rate=10r/s;
    limit_req_zone $binary_remote_addr zone=login:10m rate=1r/s;
    limit_req_zone $binary_remote_addr zone=api:10m rate=30r/s;

    # ... server blocks
}

# In server block
server {
    # General pages: 10 req/sec with burst of 20
    location / {
        limit_req zone=general burst=20 nodelay;
        proxy_pass http://localhost:3000;
    }

    # Login: 1 req/sec (strict)
    location /api/auth/ {
        limit_req zone=login burst=5;
        proxy_pass http://localhost:3000;
    }

    # API: 30 req/sec
    location /api/ {
        limit_req zone=api burst=50 nodelay;
        proxy_pass http://localhost:3000;
    }
}
```

---

## 9. Automated Security Updates

```bash
# Install unattended-upgrades
sudo apt install unattended-upgrades -y

# Configure
sudo nano /etc/apt/apt.conf.d/50unattended-upgrades
```

```
// Enable security updates only (safe)
Unattended-Upgrade::Allowed-Origins {
    "${distro_id}:${distro_codename}-security";
};

// Auto-remove unused dependencies
Unattended-Upgrade::Remove-Unused-Dependencies "true";

// Auto-reboot if needed (careful with this)
Unattended-Upgrade::Automatic-Reboot "true";
Unattended-Upgrade::Automatic-Reboot-Time "04:00";

// Email notifications
// Unattended-Upgrade::Mail "you@example.com";
```

```bash
# Enable
sudo dpkg-reconfigure -plow unattended-upgrades

# Test
sudo unattended-upgrades --dry-run --debug

# Check logs
cat /var/log/unattended-upgrades/unattended-upgrades.log
```

---

## 10. Security Audit Checklist

### Run After Every Server Setup
```bash
# 1. Check open ports (should only be 22/2222, 80, 443)
sudo ss -tlnp
sudo nmap -sT localhost

# 2. Check running services (disable what you don't need)
sudo systemctl list-units --type=service --state=running

# 3. Check for root SSH login attempts
sudo grep "Failed password" /var/log/auth.log | tail -20

# 4. Check listening processes
sudo lsof -i -P -n | grep LISTEN

# 5. Check user accounts (look for unexpected ones)
cat /etc/passwd | grep -v nologin | grep -v false

# 6. Check sudo access
sudo cat /etc/sudoers
getent group sudo

# 7. Check file permissions on sensitive files
ls -la /etc/ssh/sshd_config
ls -la /etc/shadow
ls -la ~/.ssh/

# 8. Check if any services run as root (they shouldn't)
ps aux | grep root

# 9. Check for outdated packages
sudo apt list --upgradable

# 10. Check SSL certificate
echo | openssl s_client -connect example.com:443 2>/dev/null | openssl x509 -noout -dates
```

### Security Score Card
```
+----------------------------------+--------+------------------+
| Item                             | Status | Priority         |
+----------------------------------+--------+------------------+
| SSH key-only auth                |  [ ]   | CRITICAL         |
| Root login disabled              |  [ ]   | CRITICAL         |
| Firewall enabled (ufw)          |  [ ]   | CRITICAL         |
| Cloud firewall configured       |  [ ]   | HIGH             |
| Fail2ban running                |  [ ]   | HIGH             |
| SSL/HTTPS enabled               |  [ ]   | CRITICAL         |
| Auto security updates           |  [ ]   | HIGH             |
| Non-root deploy user            |  [ ]   | HIGH             |
| .env not in git                 |  [ ]   | CRITICAL         |
| App port not publicly exposed   |  [ ]   | HIGH             |
| Security headers set            |  [ ]   | MEDIUM           |
| Rate limiting enabled           |  [ ]   | MEDIUM           |
| SSH port changed from 22        |  [ ]   | LOW              |
| WordPress xmlrpc blocked        |  [ ]   | HIGH (if WP)     |
+----------------------------------+--------+------------------+
```

---

## Quick Reference: Security Commands

```bash
# --- FIREWALL ---
sudo ufw status                    # Check firewall status
sudo ufw allow 80/tcp             # Open a port
sudo ufw deny from 1.2.3.4       # Block an IP

# --- SSH ---
sudo cat /var/log/auth.log | grep "Failed" | tail  # Failed login attempts
sudo ss -tlnp | grep ssh          # Check SSH is listening
who                                # Who is logged in now
last                               # Login history

# --- FAIL2BAN ---
sudo fail2ban-client status        # Overall status
sudo fail2ban-client status sshd   # SSH jail status
sudo fail2ban-client set sshd unbanip 1.2.3.4  # Unban IP

# --- SSL ---
sudo certbot certificates          # List certificates
sudo certbot renew --dry-run       # Test renewal
openssl s_client -connect example.com:443  # Test SSL connection

# --- GENERAL ---
sudo apt update && sudo apt upgrade  # Update everything
sudo lynis audit system              # Full security audit (install: apt install lynis)
```
