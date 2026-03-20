# 17 - Process Management Layer (Layer 6) - Deep Dive

> Keeping your application alive, restarting on crash, and surviving reboots.

---

## Why You Need Process Management

```
Without process management:
  ssh deploy@server
  node app.js              # App runs...
  # You close terminal    --> App dies
  # App crashes at 3 AM   --> App stays dead until you wake up
  # Server reboots         --> App doesn't start

With process management (PM2 / systemd):
  - App auto-restarts on crash
  - App starts on server boot
  - App runs in background (no terminal needed)
  - Logs are captured and rotated
  - Zero-downtime restarts
  - Cluster mode (use all CPU cores)
```

---

## The Two Main Tools

```
PM2       --> Best for Node.js apps. Feature-rich. Easy.
systemd   --> Built into Linux. Works for everything. More manual.

Decision:
  Node.js / Next.js  --> Use PM2
  PHP                 --> PHP-FPM is already process-managed (see Runtime Layer)
  Python              --> Use systemd (or Gunicorn + systemd)
  Anything else       --> Use systemd
  WordPress           --> PHP-FPM + MySQL (both already managed by systemd)
```

---

## PM2 - Complete Guide

### Installation
```bash
# Install globally (nvm = no sudo needed)
npm install -g pm2

# Verify
pm2 --version
```

### Starting Apps
```bash
# Basic start
pm2 start app.js                         # Start a script
pm2 start app.js --name "my-api"         # With a custom name

# Start npm scripts
pm2 start npm --name "my-api" -- start   # Runs: npm start
pm2 start npm --name "my-api" -- run dev # Runs: npm run dev (not for prod)

# Start Next.js
pm2 start npm --name "nextjs" -- start   # After: npm run build

# With options
pm2 start app.js --name "api" \
    --max-memory-restart 300M \           # Restart if exceeds 300MB
    --log /var/log/pm2/api.log \          # Custom log path
    --time                                # Add timestamps to logs
```

### Essential PM2 Commands
```bash
# --- PROCESS CONTROL ---
pm2 list                                 # List all processes
pm2 show my-api                          # Detailed info on one process
pm2 restart my-api                       # Restart
pm2 reload my-api                        # Zero-downtime restart (cluster mode)
pm2 stop my-api                          # Stop
pm2 delete my-api                        # Remove from PM2

pm2 restart all                          # Restart everything
pm2 stop all                             # Stop everything

# --- LOGS ---
pm2 logs                                 # All logs (live stream)
pm2 logs my-api                          # Logs for one app
pm2 logs my-api --lines 200             # Last 200 lines
pm2 flush                                # Clear all log files

# --- MONITORING ---
pm2 monit                                # Real-time dashboard (CPU, memory, logs)
pm2 status                               # Same as pm2 list

# --- INFO ---
pm2 describe my-api                      # Full details
pm2 env my-api                           # Environment variables
```

### PM2 Ecosystem File (Production Standard)

```javascript
// ecosystem.config.js -- put this in your project root

module.exports = {
  apps: [{
    name: 'my-api',
    script: 'app.js',                    // Entry point
    cwd: '/var/www/myapp',               // Working directory

    // --- CLUSTER MODE ---
    instances: 'max',                    // Use all CPU cores
    exec_mode: 'cluster',               // Enable clustering
    // instances: 2,                     // Or specify exact number

    // --- ENVIRONMENT ---
    env: {
      NODE_ENV: 'production',
      PORT: 3000
    },

    // --- MEMORY & RESTART ---
    max_memory_restart: '500M',          // Restart if exceeds 500MB
    max_restarts: 10,                    // Max 10 restarts in restart_delay window
    min_uptime: '10s',                   // Must be up 10s to count as "started"
    restart_delay: 4000,                 // Wait 4s between restarts

    // --- LOGS ---
    log_file: '/var/log/pm2/combined.log',
    out_file: '/var/log/pm2/out.log',
    error_file: '/var/log/pm2/error.log',
    log_date_format: 'YYYY-MM-DD HH:mm:ss Z',
    merge_logs: true,                    // Merge cluster logs into one file

    // --- WATCH (not for production usually) ---
    watch: false,
    ignore_watch: ['node_modules', 'logs', '.git'],
  }]
};
```

```bash
# Start with ecosystem file
pm2 start ecosystem.config.js

# Restart with ecosystem file
pm2 restart ecosystem.config.js

# Delete and restart fresh
pm2 delete my-api && pm2 start ecosystem.config.js
```

### PM2 Cluster Mode Explained

```
Without cluster mode (fork mode):
  1 Node.js process = 1 CPU core used
  4-core server = 75% CPU wasted

With cluster mode:
  PM2 runs multiple instances of your app
  All share port 3000 (PM2 load balances between them)
  4-core server = 4 instances = all cores used

         [PM2 - Master Process]
            /    |    |    \
          [App] [App] [App] [App]
          :3000 :3000 :3000 :3000
          Core1 Core2 Core3 Core4
            \    |    |    /
             [All share port 3000]
                    |
               [Nginx :443]
                    |
               [Internet]
```

```bash
# Start in cluster mode
pm2 start app.js -i max                 # All cores
pm2 start app.js -i 2                   # 2 instances

# Zero-downtime reload (cluster mode only)
pm2 reload my-api
# This restarts instances one by one
# At least one instance is always running = zero downtime

# Scale up/down
pm2 scale my-api +2                     # Add 2 more instances
pm2 scale my-api 4                      # Set to exactly 4 instances
```

### PM2 Startup and Save (Critical!)

```bash
# STEP 1: Generate startup script
pm2 startup
# This outputs a command like:
# sudo env PATH=$PATH:/home/deploy/.nvm/... pm2 startup systemd -u deploy --hp /home/deploy
# COPY AND RUN that exact command!

# STEP 2: Save current process list
pm2 save

# Now: when server reboots, PM2 auto-starts with your saved processes

# If you change your processes (add/remove apps):
pm2 save                                # Save again!

# Remove startup script
pm2 unstartup systemd
```

### PM2 Log Management

```bash
# PM2 logs can grow huge. Install log rotation:
pm2 install pm2-logrotate

# Configure rotation
pm2 set pm2-logrotate:max_size 10M      # Rotate at 10MB
pm2 set pm2-logrotate:retain 7          # Keep 7 rotated files
pm2 set pm2-logrotate:compress true     # Compress old logs
pm2 set pm2-logrotate:dateFormat YYYY-MM-DD_HH-mm-ss
pm2 set pm2-logrotate:rotateInterval '0 0 * * *'  # Rotate daily at midnight

# View log rotation settings
pm2 get pm2-logrotate
```

---

## systemd - Complete Guide

### When to Use systemd Over PM2
```
Use systemd when:
- Running non-Node apps (Python, Go, Java, etc.)
- You want system-level integration
- You don't want to install PM2
- Running background services (workers, queue processors)
- You need dependency ordering (start after database)
```

### Creating a Service

```bash
# Step 1: Create the service file
sudo nano /etc/systemd/system/myapp.service
```

```ini
[Unit]
Description=My Node.js Application
Documentation=https://github.com/yourrepo
After=network.target                     # Start after network is up
# After=network.target mysql.service     # Start after network AND MySQL
# Wants=mysql.service                    # Soft dependency on MySQL

[Service]
Type=simple                              # Process runs in foreground
User=deploy                              # Run as this user (NOT root!)
Group=deploy
WorkingDirectory=/var/www/myapp          # cd to this directory first

# The command to start your app
ExecStart=/home/deploy/.nvm/versions/node/v20.11.0/bin/node app.js
# For npm: ExecStart=/home/deploy/.nvm/versions/node/v20.11.0/bin/npm start
# For Python: ExecStart=/var/www/myapp/venv/bin/gunicorn app:app

# Pre/Post commands
# ExecStartPre=/usr/bin/npm run migrate  # Run before start
# ExecStartPost=/usr/bin/curl -s http://localhost:3000/health  # Verify after start

# Restart policy
Restart=on-failure                       # Restart only on crash (not clean exit)
# Restart=always                         # Restart no matter what
RestartSec=10                            # Wait 10 seconds before restart
StartLimitBurst=5                        # Max 5 restarts...
StartLimitIntervalSec=60                 # ...in 60 seconds

# Environment variables
Environment=NODE_ENV=production
Environment=PORT=3000
# Or load from file:
EnvironmentFile=/var/www/myapp/.env

# Security hardening
NoNewPrivileges=true                     # Can't gain extra privileges
PrivateTmp=true                          # Isolated /tmp directory

# Logging (goes to journald by default)
StandardOutput=journal
StandardError=journal
SyslogIdentifier=myapp

[Install]
WantedBy=multi-user.target              # Start at normal boot
```

### systemd Commands

```bash
# --- LIFECYCLE ---
sudo systemctl daemon-reload             # ALWAYS run after editing service files
sudo systemctl start myapp              # Start the service
sudo systemctl stop myapp               # Stop
sudo systemctl restart myapp            # Restart (brief downtime)
sudo systemctl enable myapp             # Start on boot
sudo systemctl disable myapp            # Don't start on boot
sudo systemctl enable --now myapp       # Enable AND start

# --- STATUS ---
sudo systemctl status myapp             # Current status + recent logs
sudo systemctl is-active myapp          # Just "active" or "inactive"
sudo systemctl is-enabled myapp         # Will it start on boot?

# --- LOGS (journald) ---
sudo journalctl -u myapp                # All logs for this service
sudo journalctl -u myapp -f             # Follow logs (live)
sudo journalctl -u myapp --since today  # Today's logs
sudo journalctl -u myapp --since "1 hour ago"
sudo journalctl -u myapp -n 100        # Last 100 lines
sudo journalctl -u myapp --no-pager    # Don't page, just dump

# --- LIST SERVICES ---
sudo systemctl list-units --type=service --state=running  # Running services
sudo systemctl list-unit-files --type=service             # All services
```

### systemd Service Examples

#### Python Flask/Django with Gunicorn
```ini
[Unit]
Description=Python Web Application
After=network.target

[Service]
User=deploy
Group=deploy
WorkingDirectory=/var/www/pyapp
ExecStart=/var/www/pyapp/venv/bin/gunicorn \
    --workers 3 \
    --bind 0.0.0.0:8000 \
    --access-logfile /var/log/pyapp/access.log \
    --error-logfile /var/log/pyapp/error.log \
    app:app
Restart=on-failure
RestartSec=5
EnvironmentFile=/var/www/pyapp/.env

[Install]
WantedBy=multi-user.target
```

#### Background Worker (Queue Processor)
```ini
[Unit]
Description=Background Job Worker
After=network.target redis.service
Wants=redis.service

[Service]
User=deploy
WorkingDirectory=/var/www/myapp
ExecStart=/home/deploy/.nvm/versions/node/v20.11.0/bin/node worker.js
Restart=always
RestartSec=5
Environment=NODE_ENV=production

[Install]
WantedBy=multi-user.target
```

---

## PM2 vs systemd Comparison

```
+-------------------------+------------------+------------------+
| Feature                 | PM2              | systemd          |
+-------------------------+------------------+------------------+
| Install                 | npm install      | Built into Linux |
| Cluster mode            | Built-in         | Manual (run N)   |
| Zero-downtime reload    | pm2 reload       | Not built-in     |
| Log management          | Built-in         | journald         |
| Monitor dashboard       | pm2 monit        | systemctl status |
| Environment file        | ecosystem.config | EnvironmentFile  |
| Start on boot           | pm2 startup      | systemctl enable |
| Multiple apps           | One ecosystem    | One file per app |
| Non-Node apps           | Limited support  | Any app          |
| Memory limit restart    | Built-in         | Not built-in     |
| Learning curve          | Easy             | Moderate         |
+-------------------------+------------------+------------------+

TLDR:
  Node.js? --> PM2 (easier, more features for Node)
  Anything else? --> systemd (universal, already there)
```

---

## Common Patterns

### Pattern 1: Deploy and Restart
```bash
# With PM2
cd /var/www/myapp
git pull origin main
npm ci --omit=dev
npm run build
pm2 reload my-api              # Zero-downtime restart

# With systemd
cd /var/www/myapp
git pull origin main
npm ci --omit=dev
npm run build
sudo systemctl restart myapp   # Brief downtime (acceptable for most)
```

### Pattern 2: View Why App Crashed
```bash
# PM2
pm2 logs my-api --lines 50    # Check error logs
pm2 show my-api                # See restart count, uptime, memory

# systemd
sudo journalctl -u myapp -n 50 --no-pager
sudo systemctl status myapp    # Shows last few log lines
```

### Pattern 3: Running Multiple Apps
```bash
# PM2 ecosystem.config.js
module.exports = {
  apps: [
    {
      name: 'api',
      script: 'api/app.js',
      env: { PORT: 3000 }
    },
    {
      name: 'frontend',
      script: 'npm',
      args: 'start',
      cwd: '/var/www/frontend',
      env: { PORT: 3001 }
    },
    {
      name: 'worker',
      script: 'worker/index.js',
      instances: 2,
      exec_mode: 'cluster'
    }
  ]
};

# systemd: separate service files
# /etc/systemd/system/myapp-api.service
# /etc/systemd/system/myapp-frontend.service
# /etc/systemd/system/myapp-worker.service
```

### Pattern 4: Graceful Shutdown
```javascript
// In your Node.js app -- handle shutdown signals

process.on('SIGTERM', gracefulShutdown);  // PM2/systemd sends this
process.on('SIGINT', gracefulShutdown);   // Ctrl+C

async function gracefulShutdown(signal) {
    console.log(`Received ${signal}. Shutting down gracefully...`);

    // Stop accepting new connections
    server.close(() => {
        console.log('HTTP server closed.');
    });

    // Close database connections
    await db.close();

    // Close Redis connections
    await redis.quit();

    // Exit
    process.exit(0);
}
```

---

## Troubleshooting

### PM2 Issues
```bash
# "pm2: command not found"
# nvm Node not in PATH for this shell
which pm2                                # Should be in ~/.nvm/...
# Fix: source ~/.bashrc or ~/.nvm/nvm.sh

# App keeps restarting (restart loop)
pm2 logs my-api                          # Check the error
pm2 show my-api                          # Check restart count
# Common causes: port in use, missing env vars, syntax error

# PM2 using too much memory
pm2 monit                                # Check per-process memory
# Add to ecosystem: max_memory_restart: '300M'

# PM2 startup not working after reboot
pm2 startup                              # Re-run, copy the command
pm2 save                                 # Don't forget to save!

# Clear all and start fresh
pm2 delete all
pm2 start ecosystem.config.js
pm2 save
```

### systemd Issues
```bash
# "Failed to start myapp.service"
sudo systemctl status myapp              # See error message
sudo journalctl -u myapp -n 30          # More detail

# "Main process exited, code=exited, status=1/FAILURE"
# Your app crashed on startup
# Check: ExecStart path is correct and absolute
# Check: WorkingDirectory exists
# Check: User has permission to run the command

# "code=exited, status=203/EXEC"
# Can't execute the command
# Fix: Use absolute path to binary
which node                               # Find full path
# Use that in ExecStart: /home/deploy/.nvm/versions/node/v20.../bin/node

# "code=exited, status=217/USER"
# User doesn't exist
# Fix: Create the user or change User= in service file

# Service starts but dies immediately
# Check min uptime and if app is actually listening
curl http://localhost:3000               # Test directly
```

---

## Quick Reference

```bash
# --- PM2 ---
pm2 start ecosystem.config.js           # Start from config
pm2 list                                 # List processes
pm2 logs my-api                          # View logs
pm2 restart my-api                       # Restart
pm2 reload my-api                        # Zero-downtime restart
pm2 monit                                # Live dashboard
pm2 save                                 # Save process list
pm2 startup                              # Enable boot startup

# --- SYSTEMD ---
sudo systemctl start myapp              # Start
sudo systemctl status myapp             # Status
sudo systemctl restart myapp            # Restart
sudo systemctl enable myapp             # Enable boot startup
sudo journalctl -u myapp -f             # Follow logs
sudo systemctl daemon-reload            # After editing .service file
```
