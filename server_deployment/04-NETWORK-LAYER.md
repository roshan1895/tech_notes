---
layout: default
title: Network Layer
---

# 02 - Network Layer Deep Dive

## The Full Network Path

```
[User's Browser]
      |
      | 1. DNS Lookup
      v
[DNS Server] (Cloudflare / Route53 / Namecheap DNS)
      |
      | 2. Returns IP address
      v
[User's Browser] now knows IP
      |
      | 3. TCP Connection (3-way handshake)
      v
[Cloud Provider Network]
      |
      | 4. Cloud firewall check
      v
[Your Server's Public IP]
      |
      | 5. OS firewall (ufw/iptables) check
      v
[Nginx listening on port 80/443]
      |
      | 6. Reverse proxy
      v
[Your App on localhost:3000]
```

---

## DNS - Domain Name System

### Record Types You Need to Know
| Record | Purpose | Example |
|--------|---------|---------|
| **A** | Maps domain to IPv4 | `example.com -> 143.198.50.100` |
| **AAAA** | Maps domain to IPv6 | `example.com -> 2001:db8::1` |
| **CNAME** | Alias to another domain | `www.example.com -> example.com` |
| **MX** | Mail server | `example.com -> mail.example.com` |
| **TXT** | Verification/SPF/DKIM | Used for email, SSL verification |
| **NS** | Nameserver delegation | Points to DNS provider |

### DNS Setup Pattern
```
# Minimum DNS setup for a web app:

example.com         A       143.198.50.100      # Root domain
www.example.com     CNAME   example.com         # www subdomain
api.example.com     A       143.198.50.100      # API subdomain (same or different server)
```

### DNS Propagation
- Changes take 5 min to 48 hours to spread globally
- TTL (Time To Live) controls caching duration
- Before migration: lower TTL to 300 (5 min) 24h ahead
- After migration confirmed: raise TTL to 3600+ (1 hour)

### DNS Debugging
```bash
# Check what IP a domain resolves to
dig example.com
dig example.com +short

# Check specific record types
dig example.com A
dig example.com CNAME
dig example.com MX

# Check from specific DNS server
dig @8.8.8.8 example.com    # Google DNS
dig @1.1.1.1 example.com    # Cloudflare DNS

# Quick check
nslookup example.com

# Check propagation globally
# Use: https://dnschecker.org (web tool)
```

---

## Ports - Understanding What Listens Where

### Standard Ports
| Port | Service | Notes |
|------|---------|-------|
| 22 | SSH | Remote login. NEVER expose without key auth |
| 80 | HTTP | Unencrypted web traffic |
| 443 | HTTPS | Encrypted web traffic |
| 3000 | Node.js (common) | Not exposed directly, behind Nginx |
| 3306 | MySQL | NEVER expose to internet |
| 5432 | PostgreSQL | NEVER expose to internet |
| 27017 | MongoDB | NEVER expose to internet |
| 6379 | Redis | NEVER expose to internet |
| 8080 | Alt HTTP | Sometimes used for staging |

### Key Rules
1. **Only expose ports 22, 80, 443 to the internet**
2. **Database ports should ONLY be accessible from localhost**
3. **App ports (3000, 5000) should be behind Nginx, not directly exposed**

### Check What's Listening
```bash
sudo ss -tlnp              # Show all listening TCP ports
sudo netstat -tlnp          # Alternative
sudo lsof -i :3000          # What's using port 3000?
curl localhost:3000          # Test if app responds locally
```

---

## Firewall (ufw)

### Basic Setup
```bash
# Reset (careful - can lock you out of SSH!)
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Allow essential services
sudo ufw allow 22/tcp       # SSH (ALWAYS DO THIS FIRST before enabling)
sudo ufw allow 80/tcp       # HTTP
sudo ufw allow 443/tcp      # HTTPS

# Enable firewall
sudo ufw enable

# Check status
sudo ufw status verbose
sudo ufw status numbered     # Numbered list (useful for deleting rules)

# Remove a rule
sudo ufw delete 3            # Delete rule number 3

# Allow from specific IP only (good for database access between servers)
sudo ufw allow from 10.0.0.5 to any port 3306
```

### Cloud Firewalls vs ufw
```
[Cloud Firewall]  - Managed in AWS/DO dashboard
                  - Filters traffic BEFORE it reaches your server
                  - Recommended as first line of defense
                  - AWS: Security Groups
                  - DO: Cloud Firewalls

[ufw]             - Runs ON your server
                  - Second line of defense
                  - Always configure even if using cloud firewall

USE BOTH. Defense in depth.
```

### AWS Security Groups
```
Inbound Rules:
  - SSH (22)    from YOUR IP only (not 0.0.0.0/0)
  - HTTP (80)   from 0.0.0.0/0 (anywhere)
  - HTTPS (443) from 0.0.0.0/0 (anywhere)

Outbound Rules:
  - All traffic to 0.0.0.0/0 (default, usually fine)
```

---

## SSL/TLS (HTTPS)

### Why
- Encrypts traffic between user and server
- Required for modern web (browsers warn without it)
- SEO ranking factor
- Required for HTTP/2

### Let's Encrypt with Certbot (Free SSL)
```bash
# Install Certbot
sudo apt install certbot python3-certbot-nginx

# Get certificate (Nginx must be configured and domain must point to server)
sudo certbot --nginx -d example.com -d www.example.com

# What this does:
# 1. Verifies you own the domain (HTTP challenge)
# 2. Downloads SSL certificate
# 3. Automatically modifies Nginx config to use HTTPS
# 4. Sets up HTTP -> HTTPS redirect

# Auto-renewal (certbot sets up a timer, but verify)
sudo certbot renew --dry-run

# Check certificate expiry
sudo certbot certificates

# Manual renewal
sudo certbot renew
```

### SSL Troubleshooting
```bash
# Check if SSL is working
curl -I https://example.com

# Check certificate details
openssl s_client -connect example.com:443 -servername example.com

# Common issues:
# 1. DNS not pointing to server yet -> certbot fails
# 2. Port 80 blocked -> certbot can't verify
# 3. Nginx config error -> certbot can't modify
# 4. Certificate expired -> forgot renewal
```

---

## Reverse Proxy (Nginx)

### What It Does
```
WITHOUT reverse proxy:
  User -> server:3000   (must expose app port directly, no SSL, no static file serving)

WITH reverse proxy (Nginx):
  User -> Nginx:443 (HTTPS) -> localhost:3000 (your app)
                |
                +--> Serves static files directly (fast)
                +--> Handles SSL termination
                +--> Can add headers, caching, rate limiting
                +--> Can load balance to multiple app instances
```

### Why Not Just Expose Port 3000?
1. No SSL/HTTPS
2. Users would type `example.com:3000` (ugly)
3. Can't serve static files efficiently
4. No request buffering
5. No connection limiting
6. Can't run multiple apps on same server

### Multiple Apps on One Server
```nginx
# App 1: example.com -> Node.js on port 3000
server {
    listen 80;
    server_name example.com;
    location / {
        proxy_pass http://localhost:3000;
    }
}

# App 2: api.example.com -> Node.js on port 4000
server {
    listen 80;
    server_name api.example.com;
    location / {
        proxy_pass http://localhost:4000;
    }
}

# App 3: blog.example.com -> WordPress on PHP-FPM
server {
    listen 80;
    server_name blog.example.com;
    root /var/www/wordpress;
    index index.php;
    location ~ \.php$ {
        fastcgi_pass unix:/run/php/php8.2-fpm.sock;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    }
}
```

---

## CDN (Content Delivery Network)

### What It Does
```
WITHOUT CDN:
  User in India -> Your server in US -> Response travels across ocean (slow)

WITH CDN (Cloudflare):
  User in India -> Cloudflare edge in Mumbai (cached) -> Fast response
  If not cached -> Cloudflare -> Your server -> Cache + respond
```

### When to Use CDN
- Static assets (images, CSS, JS) - ALWAYS benefit from CDN
- Full site (WordPress) - Yes, if global audience
- API-only backend - Usually not needed (dynamic, uncacheable)

### Cloudflare (Most Popular Free CDN)
```
Setup:
1. Add site to Cloudflare
2. Change nameservers at your registrar to Cloudflare's
3. Configure DNS records in Cloudflare dashboard
4. Enable proxy (orange cloud icon) for records you want CDN for

Benefits:
- Free SSL
- DDoS protection
- Caching
- Analytics
- Page rules (redirects, cache settings)
```

---

## Common Network Debugging Scenarios

### "Site is not loading"
```bash
# Step 1: Is the server reachable?
ping your-server-ip

# Step 2: Is DNS pointing correctly?
dig example.com +short
# Should show your server IP

# Step 3: Are ports open?
sudo ufw status
# Should show 80, 443 ALLOW

# Step 4: Is Nginx running?
sudo systemctl status nginx

# Step 5: Is Nginx config correct?
sudo nginx -t

# Step 6: Is your app running?
curl localhost:3000
# Or: pm2 list

# Step 7: Check Nginx error logs
sudo tail -50 /var/log/nginx/error.log
```

### "Works on HTTP but not HTTPS"
```bash
# Check if certbot ran successfully
sudo certbot certificates

# Check Nginx SSL config
sudo nginx -t

# Check if port 443 is open
sudo ufw status | grep 443
```

### "502 Bad Gateway"
```
This means Nginx is working, but your app is NOT.

Fix:
1. Check if app is running: pm2 list / systemctl status myapp
2. Check if app port matches Nginx proxy_pass port
3. Check app logs: pm2 logs / journalctl -u myapp
```

### "Connection Refused"
```
This means nothing is listening on that port.

Fix:
1. Check if Nginx is running: systemctl status nginx
2. Check if correct port is in Nginx config
3. Check firewall: ufw status
```

---

## Network Architecture Decision Tree

```
Q: Do I need a domain?
   Yes (always for production)
   |
Q: Where to manage DNS?
   ├── Cloudflare (free, CDN, DDoS protection) --> RECOMMENDED
   ├── Route53 (if all-in on AWS)
   └── Registrar DNS (Namecheap, GoDaddy) --> least features
   |
Q: Do I need CDN?
   ├── Global users? --> Yes (Cloudflare)
   ├── Static-heavy site? --> Yes
   └── Small/local API? --> No, skip it
   |
Q: Do I need multiple servers?
   ├── < 1000 users/day? --> No, single server is fine
   ├── Need high availability? --> Yes, load balancer
   └── Separate frontend/backend? --> Can be same or different server
```
