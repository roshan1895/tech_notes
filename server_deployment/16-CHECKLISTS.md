---
layout: default
title: Checklists
---

# 10 - Deployment Checklists

## Checklist A: New Server Setup (One-Time)

### Infrastructure
- [ ] Server provisioned (correct region, size, OS)
- [ ] SSH key added (can login without password)
- [ ] System updated: `sudo apt update && sudo apt upgrade -y`
- [ ] Swap enabled (for servers with < 4GB RAM)
- [ ] Timezone set: `sudo timedatectl set-timezone UTC`

### Security
- [ ] Non-root user created: `adduser deploy`
- [ ] User added to sudo: `usermod -aG sudo deploy`
- [ ] SSH key copied to deploy user
- [ ] Root login disabled in sshd_config
- [ ] Password auth disabled in sshd_config
- [ ] ufw enabled with only 22, 80, 443 open
- [ ] Cloud firewall configured (AWS SG / DO Firewall)
- [ ] Fail2ban installed and running
- [ ] Auto security updates enabled

### Runtime
- [ ] Node.js installed via nvm (NOT apt)
- [ ] PHP-FPM installed (if needed)
- [ ] Database installed and secured (if needed)
- [ ] Database user created (not root)

### Web Server
- [ ] Nginx installed and running
- [ ] Default site removed
- [ ] Site config created in sites-available
- [ ] Site enabled (symlinked to sites-enabled)
- [ ] `nginx -t` passes
- [ ] Nginx reloaded

### SSL
- [ ] Certbot installed
- [ ] Certificate obtained for domain(s)
- [ ] Auto-renewal tested: `certbot renew --dry-run`
- [ ] HTTP redirects to HTTPS

### Process Management
- [ ] PM2 installed (for Node.js)
- [ ] App started with PM2
- [ ] PM2 startup configured (survives reboot)
- [ ] PM2 save executed

---

## Checklist B: Code Deployment (Every Deploy)

### Pre-Deploy
- [ ] Code committed and pushed to repository
- [ ] Tests passing (if any)
- [ ] Environment variables updated (if changed)
- [ ] Database migration prepared (if needed)
- [ ] Know how to rollback if needed

### Deploy
- [ ] SSH into server
- [ ] Navigate to app directory
- [ ] Pull latest code: `git pull origin main`
- [ ] Install dependencies: `npm install --production`
- [ ] Build application: `npm run build` (if needed)
- [ ] Run database migrations (if any)
- [ ] Restart application: `pm2 restart myapp`

### Post-Deploy Verify
- [ ] App process is running: `pm2 list`
- [ ] App responds locally: `curl localhost:3000`
- [ ] Site loads in browser (HTTP)
- [ ] Site loads in browser (HTTPS)
- [ ] Check logs for errors: `pm2 logs myapp --lines 30`
- [ ] Key features working (quick manual test)
- [ ] Monitor for 5-10 minutes

---

## Checklist C: WordPress Deployment

### New WordPress Setup
- [ ] PHP-FPM installed with required extensions
- [ ] MySQL/MariaDB installed and secured
- [ ] Database and user created
- [ ] WordPress downloaded and extracted
- [ ] wp-config.php configured (DB creds + salts)
- [ ] File permissions set (www-data ownership)
- [ ] Nginx configured for PHP processing
- [ ] SSL certificate obtained
- [ ] WordPress install wizard completed in browser
- [ ] Admin username is NOT "admin"
- [ ] File editing disabled in wp-config.php
- [ ] Auto-updates enabled for minor versions

### WordPress Update/Plugin Deploy
- [ ] Backup database: `mysqldump -u user -p db > backup.sql`
- [ ] Backup files: `tar -czf backup.tar.gz /var/www/wordpress`
- [ ] Update WordPress core (if needed)
- [ ] Update plugins
- [ ] Update theme
- [ ] Check site functionality
- [ ] Clear any caches

---

## Checklist D: React/Static Site Deployment

### Deploy
- [ ] Build locally or on server: `npm run build`
- [ ] Upload build files: `rsync -avz --delete ./build/ server:/var/www/app/build/`
- [ ] Nginx config has `try_files $uri $uri/ /index.html`
- [ ] Static asset caching headers set
- [ ] index.html NOT cached (no-cache header)
- [ ] Gzip enabled in Nginx

### Verify
- [ ] Site loads in browser
- [ ] All routes work (try direct URL access to a sub-route)
- [ ] API connections working (check browser console)
- [ ] No mixed content warnings (HTTP assets on HTTPS page)

---

## Checklist E: Next.js Deployment

### Deploy
- [ ] Pull latest code
- [ ] Install dependencies: `npm ci`
- [ ] Build: `npm run build` (check for build errors)
- [ ] Environment variables set (.env.local)
- [ ] Restart with PM2: `pm2 restart mynextapp`

### Verify
- [ ] Server-rendered pages load (view page source to confirm)
- [ ] API routes working: `curl localhost:3000/api/health`
- [ ] Static assets loading (check browser network tab)
- [ ] Image optimization working
- [ ] No hydration errors (check browser console)

---

## Checklist F: Security Audit (Monthly)

- [ ] System packages updated: `sudo apt update && sudo apt upgrade`
- [ ] Check for failed SSH attempts: `grep "Failed" /var/log/auth.log | wc -l`
- [ ] Review fail2ban bans: `sudo fail2ban-client status sshd`
- [ ] Check open ports: `sudo ss -tlnp` (only expected ports)
- [ ] Review running processes: `ps aux` (nothing unexpected)
- [ ] Check disk usage: `df -h` (not > 80%)
- [ ] Check RAM usage: `free -h`
- [ ] Verify SSL certificate expiry: `sudo certbot certificates`
- [ ] Review Nginx access logs for anomalies
- [ ] Verify backups are running and recent
- [ ] Check .env file permissions: `ls -la .env` (should be 600)
- [ ] No world-writable files: `find /var/www -perm -o+w -type f`

---

## Checklist G: Before Going Live (Production Launch)

### Performance
- [ ] Gzip compression enabled
- [ ] Static asset caching configured
- [ ] CDN configured (Cloudflare recommended)
- [ ] Images optimized (WebP where possible)
- [ ] Database queries optimized (indexes on common queries)
- [ ] PM2 in cluster mode: `pm2 start app.js -i max`

### Security
- [ ] All security items from Checklist A
- [ ] CORS configured correctly (not wildcard in production)
- [ ] Rate limiting enabled (Nginx or app-level)
- [ ] Input validation on all user inputs
- [ ] SQL injection protection (parameterized queries)
- [ ] XSS protection (escape user content)
- [ ] CSRF protection (if using forms)
- [ ] Security headers in Nginx
- [ ] No sensitive data in client-side code
- [ ] API keys and secrets in .env only

### Reliability
- [ ] Automated backups configured and tested
- [ ] Health check endpoint: GET /health returns 200
- [ ] Error handling in application (no crashes on bad input)
- [ ] PM2/systemd configured to auto-restart on crash
- [ ] Logging configured and accessible
- [ ] Uptime monitoring set up (UptimeRobot - free)

### DNS & Domain
- [ ] Domain pointing to correct server
- [ ] www and non-www both work
- [ ] HTTP redirects to HTTPS
- [ ] Email records configured (if needed)

---

## Quick Reference: Which Checklist When

| Situation | Use Checklist |
|-----------|---------------|
| Setting up brand new server | A (Server Setup) |
| Deploying code update | B (Code Deployment) |
| New WordPress site | C (WordPress) |
| Deploying React app | D (Static Site) |
| Deploying Next.js | E (Next.js) |
| Monthly maintenance | F (Security Audit) |
| Launching new product | G (Production Launch) |
