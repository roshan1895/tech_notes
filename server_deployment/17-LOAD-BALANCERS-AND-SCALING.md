---
layout: default
title: Load Balancers & Scaling
---

# 09 - Load Balancers, Scaling & Cost Management

## When Do You Need Scaling?

### You DON'T Need a Load Balancer If:
- Under 50,000 daily visitors
- Single app on single server
- Server CPU < 70% consistently
- Response times are acceptable (< 500ms)
- No high-availability requirement

### You NEED Scaling When:
- Server CPU consistently > 80%
- Response times degrading under load
- Need zero downtime (can't afford being offline)
- Single server RAM maxed out
- Compliance requires redundancy

### Decision Tree
```
Q: Is your single server struggling?
  ├── NO -> Don't add complexity. Stop here.
  └── YES
      |
Q: What's the bottleneck?
  ├── CPU -> Vertical scale first (bigger server)
  ├── RAM -> Vertical scale first (bigger server)
  ├── Disk I/O -> Switch to SSD or add caching
  └── Network -> CDN for static, LB for dynamic
      |
Q: Still not enough after vertical scaling?
  └── YES -> Horizontal scale (multiple servers + load balancer)
```

---

## Vertical Scaling (Scale Up)

### What It Is
Make your single server bigger (more CPU, more RAM).

```
BEFORE:  1 vCPU, 2GB RAM  ($12/mo)
AFTER:   4 vCPU, 8GB RAM  ($48/mo)

Pros:
  - Simplest approach
  - No code changes
  - No architecture changes
  - Resize in cloud dashboard

Cons:
  - Has a ceiling (can't go infinitely big)
  - Still single point of failure
  - Brief downtime during resize (usually)
```

### How to Resize
```
AWS EC2:
  1. Stop instance
  2. Change instance type (e.g., t3.micro -> t3.large)
  3. Start instance
  4. Elastic IP remains same

DigitalOcean:
  1. Power off droplet
  2. Resize (choose new plan)
  3. Power on
  4. IP remains same
```

### Optimization Before Scaling (Do This First!)
```
Before spending money on bigger servers, optimize:

1. Database queries
   - Add indexes to slow queries
   - Use EXPLAIN on slow queries
   - Cache frequent queries

2. Caching
   - Redis for session/query caching
   - Nginx caching for static assets
   - CDN for global static delivery

3. Code optimization
   - Reduce unnecessary API calls
   - Optimize image sizes
   - Lazy load where possible

4. PM2 cluster mode (use all CPU cores)
   pm2 start app.js -i max
```

---

## Horizontal Scaling (Scale Out)

### What It Is
Add more servers and distribute traffic between them.

```
BEFORE:  1 server handling everything

AFTER:
  [Load Balancer]
    ├── [Server 1: App]
    ├── [Server 2: App]
    └── [Server 3: App]
  [Shared Database]
  [Shared File Storage]
```

### Requirements for Horizontal Scaling
```
Your app MUST be:
1. STATELESS - No data stored in memory between requests
   - Sessions: Use Redis (not in-memory)
   - Uploads: Use S3 or shared storage (not local disk)
   - Cache: Use Redis (not local variables)

2. SHARED DATABASE - All servers connect to same DB
   - Run database on separate server
   - Or use managed DB (RDS, DO Managed DB)

3. SHARED FILES - Uploads accessible from all servers
   - Use S3 / DigitalOcean Spaces for file uploads
```

---

## Load Balancers

### What a Load Balancer Does
```
[Internet Traffic]
       |
       v
[Load Balancer]        -- Distributes requests evenly
  ├── [Server 1] ---- 33% of traffic
  ├── [Server 2] ---- 33% of traffic
  └── [Server 3] ---- 33% of traffic

If Server 2 dies:
  ├── [Server 1] ---- 50% of traffic
  └── [Server 3] ---- 50% of traffic

  Load Balancer detects failure and stops sending traffic to dead server.
```

### Load Balancer Options

| Option | Type | Cost | Complexity |
|--------|------|------|------------|
| **Nginx LB** | Self-managed | Free (your server) | Medium |
| **AWS ALB** | Managed | ~$22/mo + traffic | Low |
| **DO Load Balancer** | Managed | $12/mo | Low |
| **Cloudflare LB** | Managed | $5/mo | Low |
| **HAProxy** | Self-managed | Free | Medium-High |

### Nginx as Load Balancer
```nginx
# /etc/nginx/sites-available/loadbalancer
upstream myapp_servers {
    # Load balancing methods:
    # (default) round-robin: each server gets requests in turn
    # least_conn: sends to server with fewest active connections
    # ip_hash: same client always goes to same server

    least_conn;

    server 10.0.0.1:3000;    # Server 1 (private IP)
    server 10.0.0.2:3000;    # Server 2
    server 10.0.0.3:3000;    # Server 3

    # Health check: if server fails 3 times in 30s, mark as down
    # Retry after 30 seconds
}

server {
    listen 80;
    server_name example.com;

    location / {
        proxy_pass http://myapp_servers;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### AWS Application Load Balancer (ALB) Setup
```
1. Create Target Group
   - Target type: Instances
   - Protocol: HTTP, Port: 3000
   - Health check path: /health (create this endpoint in your app)
   - Register your EC2 instances

2. Create ALB
   - Internet-facing
   - Select subnets (at least 2 AZs)
   - Security group: allow 80, 443
   - Listener: HTTP:80 -> Target Group
   - Add HTTPS:443 listener with ACM certificate

3. Point DNS to ALB
   - Create CNAME or Alias record pointing to ALB DNS name
```

---

## Architecture Patterns

### Pattern 1: Single Server (Most Common, Start Here)
```
[Nginx + App + DB]    # Everything on one server
Cost: $6-24/mo
Good for: < 50k daily visitors
```

### Pattern 2: App + Separate DB
```
[Server: Nginx + App] <-> [Managed DB]
Cost: $20-60/mo
Good for: Better DB performance, easier backups
When: DB is bottleneck, need managed backups
```

### Pattern 3: Load Balanced
```
[Load Balancer]
  ├── [App Server 1]
  └── [App Server 2]
       └── [Managed DB]
           [Redis]
Cost: $60-200/mo
Good for: High availability, handling traffic spikes
```

### Pattern 4: Full Production
```
[CDN (Cloudflare)]
       |
[Load Balancer]
  ├── [App Server 1]
  ├── [App Server 2]
  └── [App Server 3]
       |
  [Managed DB (primary + replica)]
  [Redis Cluster]
  [S3 for file storage]
  [Monitoring]
Cost: $200-1000/mo
Good for: Production SaaS, high traffic
```

---

## Cost Management

### Cost Per Provider (Approximate Monthly)

| Resource | AWS | DigitalOcean | Estimate |
|----------|-----|-------------|----------|
| Small VPS (1 CPU, 1GB) | t3.micro: $8 | Basic: $6 | $6-8 |
| Medium VPS (2 CPU, 4GB) | t3.medium: $30 | Basic: $24 | $24-30 |
| Large VPS (4 CPU, 8GB) | t3.large: $60 | Basic: $48 | $48-60 |
| Managed DB (small) | RDS: $15+ | DO: $15 | $15-25 |
| Load Balancer | ALB: $22+ | DO LB: $12 | $12-22 |
| Storage (50GB) | S3: $1 | Spaces: $5 | $1-5 |
| SSL | Free (ACM/LE) | Free (LE) | $0 |
| Domain | Route53: $12/yr | Elsewhere | $10-15/yr |

### Cost Saving Tips
```
1. START SMALL
   - Begin with 1GB, upgrade when needed
   - Don't pre-optimize for scale you don't have

2. USE RESERVED INSTANCES (AWS)
   - Commit to 1 year: save 30-40%
   - Commit to 3 years: save 50-60%

3. SPOT/PREEMPTIBLE INSTANCES
   - 60-90% cheaper
   - Can be terminated anytime
   - Good for: CI/CD runners, batch processing
   - NOT for: Production servers

4. RIGHT-SIZE
   - Monitor actual usage
   - Downsize over-provisioned servers
   - Check: free -h, top, df -h

5. CLEAN UP
   - Delete unused snapshots
   - Remove old backups
   - Stop unused instances
   - Check billing dashboard monthly

6. DO > AWS for simple setups
   - DigitalOcean is 20-40% cheaper for equivalent specs
   - Simpler pricing, no surprise bills
   - AWS makes sense for: complex setups, enterprise features
```

### AWS Cost Gotchas
```
WATCH OUT FOR:
  - Data transfer OUT charges (first 1GB free, then $0.09/GB)
  - Elastic IP charges when NOT attached to running instance
  - EBS volumes still charged when instance is stopped
  - NAT Gateway: $32/mo + data charges
  - Load Balancer: $22/mo minimum even with no traffic
  - CloudWatch logs: can accumulate cost

SET UP BILLING ALERTS:
  AWS Console -> Billing -> Budgets -> Create Budget
  Set alert at $50, $100 or your max acceptable spend
```

---

## Practical Scaling Playbook

### Step 1: Optimize First (Free)
```bash
# Enable PM2 cluster mode (use all CPUs)
pm2 start app.js -i max

# Enable Nginx gzip
gzip on;
gzip_types text/plain text/css application/json application/javascript;

# Enable Nginx caching
location ~* \.(jpg|jpeg|png|gif|ico|css|js)$ {
    expires 30d;
}

# Add Redis caching to your app
# Install: sudo apt install redis-server
# Use in app: cache frequent DB queries
```

### Step 2: Vertical Scale ($)
```
Resize to a bigger server.
Often takes you from 1k to 50k users.
```

### Step 3: Separate Database ($$)
```
Move DB to managed service.
Reduces load on app server.
Enables independent scaling.
```

### Step 4: Add CDN ($ or Free)
```
Cloudflare free tier for static assets.
Reduces server load by 50-80% for static-heavy sites.
```

### Step 5: Horizontal Scale ($$$)
```
Only if steps 1-4 aren't enough.
Add load balancer + multiple app servers.
Requires stateless app design.
```
