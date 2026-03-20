# 12 - Decision Making Framework

## How to Choose: The Right Setup for Your Project

---

## Decision 1: Which Cloud Provider?

```
                        AWS                     DigitalOcean
                        ---                     ------------
Cost                    Higher                  Lower (20-40% less)
Complexity              Complex                 Simple
Services                200+                    ~20
Learning curve          Steep                   Easy
Free tier               12 months               None (but $200 credit)
Support                 Paid plans              Included
Billing surprises       Common                  Rare (flat pricing)
Enterprise features     Yes                     Limited

CHOOSE AWS IF:
  - Need specific AWS services (Lambda, SQS, SNS, etc.)
  - Enterprise client requires AWS
  - Need advanced networking (VPC peering, Transit Gateway)
  - Auto-scaling groups needed
  - Multi-region deployment

CHOOSE DIGITALOCEAN IF:
  - Simple VPS deployment
  - Predictable pricing matters
  - Fewer than 10 servers
  - Don't need AWS-specific services
  - Want faster setup

CHOOSE BOTH:
  - Many companies use DO for simple apps and AWS for complex ones
  - Totally fine to mix
```

---

## Decision 2: Server Size

```
PROJECT TYPE                     MINIMUM SIZE           RECOMMENDED
-----------                      ------------           -----------
Static site (React build)        1 CPU / 512MB ($4)     1 CPU / 1GB ($6)
Node.js API (light traffic)      1 CPU / 1GB ($6)       1 CPU / 2GB ($12)
Node.js API (medium traffic)     2 CPU / 4GB ($24)      2 CPU / 4GB ($24)
Next.js (SSR)                    1 CPU / 2GB ($12)      2 CPU / 4GB ($24)
WordPress (small)                1 CPU / 2GB ($12)      2 CPU / 4GB ($24)
WordPress (WooCommerce)          2 CPU / 4GB ($24)      4 CPU / 8GB ($48)
Multiple apps on one server      2 CPU / 4GB ($24)      4 CPU / 8GB ($48)

RULE OF THUMB:
  - npm run build needs 1-2GB RAM (build on local if server is small)
  - WordPress + MySQL needs minimum 2GB
  - Always add swap on servers < 4GB
  - Start small, scale up when needed
```

---

## Decision 3: One Server or Multiple?

```
ONE SERVER (Start Here):
  When:
    - < 50k daily visitors total
    - 1-3 apps
    - Same team manages everything
    - Budget conscious
    - Learning/early stage

  Example:
    1 server ($24/mo) running:
      - React frontend (Nginx static)
      - Node.js API (PM2, port 3000)
      - PostgreSQL database
      - WordPress blog (PHP-FPM, separate Nginx block)

SEPARATE SERVERS:
  When:
    - One app consuming too many resources
    - Different security requirements
    - Need to scale independently
    - Different teams
    - Database needs dedicated resources

  Example:
    Server 1 ($12/mo): React frontend + Nginx
    Server 2 ($24/mo): Node.js API + PM2
    Server 3 ($15/mo): Managed PostgreSQL database
```

---

## Decision 4: Deployment Method

```
YOUR SITUATION                           USE THIS METHOD
----------------                         ---------------
Solo dev, < 5 servers, learning          Manual + deploy scripts (semi-auto)
Solo dev, want faster deploys            GitHub Actions (auto)
Small team, shared servers               GitHub Actions + staging env
Frontend only (React/Next static)        Vercel or Netlify (easiest)
WordPress                                Manual or WP-specific hosting
Multiple microservices                   Docker Compose
Enterprise / large team                  Docker + Kubernetes
```

---

## Decision 5: Database Choices

```
DATABASE         BEST FOR                           COST (MANAGED)
--------         --------                           ----
PostgreSQL       Most apps, relational data         $15/mo (DO) / $15/mo (RDS)
MySQL/MariaDB    WordPress, PHP apps, legacy        $15/mo (DO) / $15/mo (RDS)
MongoDB          Document data, flexible schema     $0 (Atlas free) / $57/mo (Atlas)
Redis            Caching, sessions, queues          $0 (self-hosted) / $15/mo (managed)
SQLite           Tiny apps, prototypes              Free (file-based)

SELF-HOSTED vs MANAGED:
  Self-hosted (on same server):
    + Free (no extra cost)
    + Lower latency (localhost)
    - You manage backups, updates, security
    - Dies if server dies

  Managed (RDS, DO Managed DB):
    + Auto backups
    + Auto updates
    + Replication available
    + Survives if app server dies
    - $15-50/mo extra
    - Slightly higher latency (network hop)

RECOMMENDATION:
  - Start self-hosted on same server
  - Move to managed when you need reliability or the DB is a bottleneck
```

---

## Decision 6: Domain & DNS Provider

```
REGISTRAR (Where you buy domains):
  - Namecheap: Cheap, good UI
  - Cloudflare Registrar: At-cost pricing (cheapest)
  - Google Domains: Simple (transitioning to Squarespace)
  - GoDaddy: Avoid (expensive renewals, upselling)

DNS PROVIDER (Where you manage records):
  - Cloudflare: FREE, CDN included, DDoS protection -> RECOMMENDED
  - Route53 (AWS): $0.50/zone/mo, integrates with AWS
  - Registrar DNS: Basic, works but fewer features

RECOMMENDATION:
  Buy domain anywhere cheap.
  Use Cloudflare for DNS (free + CDN + DDoS protection).
```

---

## Decision 7: SSL Strategy

```
OPTION                   COST    SETUP            RENEWAL
------                   ----    -----            -------
Let's Encrypt            Free    Certbot (easy)   Auto (90 days)
Cloudflare (proxy)       Free    Dashboard        Auto
AWS ACM                  Free    AWS console      Auto (for ALB/CloudFront)
Paid SSL (DigiCert)      $100+   Complex          Manual yearly

RECOMMENDATION:
  Use Let's Encrypt with Certbot. Always.
  If using Cloudflare proxy, you get SSL for free automatically.
```

---

## Decision 8: Monitoring

```
TOOL             WHAT IT DOES                   COST
----             ------------                   ----
UptimeRobot      Is my site up?                 Free (50 monitors)
Sentry           App error tracking             Free (5k events/mo)
PM2 Plus         PM2 dashboard + alerts         Free (limited) / $14/mo
Grafana + Prometheus  Full metrics dashboard    Free (self-hosted)
Datadog          Full observability             $15/host/mo
Better Stack     Uptime + logs                  Free tier

START WITH:
  1. UptimeRobot (free) - alerts when site goes down
  2. PM2 logs - check manually
  3. Add Sentry when you want error tracking

ADD LATER:
  4. Grafana + Prometheus when managing 5+ servers
```

---

## Decision Matrix: Complete Project Setup

### Example 1: Personal Project / MVP
```
Provider:    DigitalOcean ($12/mo droplet)
Server:      1 server, everything on it
Stack:       Node.js API + React frontend
Database:    PostgreSQL (self-hosted)
Deploy:      Git pull + deploy script
SSL:         Let's Encrypt
DNS:         Cloudflare (free)
Monitoring:  UptimeRobot (free)
Total cost:  ~$12/mo + domain
```

### Example 2: Small Business Website
```
Provider:    DigitalOcean ($24/mo droplet)
Server:      1 server
Stack:       WordPress
Database:    MySQL (self-hosted)
Deploy:      Manual + backup before update
SSL:         Let's Encrypt
DNS:         Cloudflare (free CDN)
Monitoring:  UptimeRobot
Total cost:  ~$24/mo + domain
```

### Example 3: SaaS Product
```
Provider:    AWS or DigitalOcean
Servers:     2 app servers + managed DB
Stack:       Next.js frontend + Node.js API
Database:    Managed PostgreSQL ($15/mo)
Deploy:      GitHub Actions (auto on push to main)
SSL:         Let's Encrypt or Cloudflare
DNS:         Cloudflare
Monitoring:  UptimeRobot + Sentry
Cache:       Redis
Total cost:  ~$60-100/mo
```

### Example 4: Growing Product (50k+ users)
```
Provider:    AWS
Servers:     Load balancer + 3 app servers
Stack:       Next.js + Node.js microservices
Database:    RDS PostgreSQL (multi-AZ)
Deploy:      GitHub Actions + Docker
SSL:         ACM (for ALB)
DNS:         Route53 or Cloudflare
Monitoring:  Grafana + Prometheus + Sentry
Cache:       ElastiCache Redis
Storage:     S3 for uploads
Total cost:  ~$200-500/mo
```

---

## Quick Decision Flowchart

```
What are you building?
  |
  ├── Static site / Landing page
  |     -> Vercel or Netlify (free)
  |     -> Or: DO $6 droplet + Nginx static
  |
  ├── WordPress / Blog with CMS
  |     -> DO $12-24 droplet + LEMP stack
  |
  ├── API backend only
  |     -> DO $12 droplet + Node.js + PM2
  |
  ├── Full-stack web app
  |     -> DO $24 droplet: Next.js/React + Node.js + PostgreSQL
  |
  ├── Multiple client projects
  |     -> DO $48 droplet with multiple Nginx server blocks
  |     -> Or: Separate $12 droplets per client
  |
  └── High-traffic SaaS
        -> AWS: ALB + multiple EC2 + RDS + ElastiCache
        -> With CI/CD pipeline and monitoring

Budget?
  |
  ├── $0      -> Vercel/Netlify free tier (frontend only)
  ├── $6-12   -> Single small DO droplet
  ├── $24-48  -> Single medium DO droplet (most projects)
  ├── $50-100 -> Multiple servers or DO + managed DB
  └── $100+   -> AWS with proper architecture
```

---

## Common Mistakes to Avoid

```
1. OVER-ENGINEERING
   Don't use Kubernetes for a blog.
   Don't set up microservices for an MVP.
   Don't add a load balancer for 100 users/day.

2. UNDER-ENGINEERING
   Don't skip SSL.
   Don't skip backups.
   Don't skip firewall setup.
   Don't run as root.

3. WRONG TOOL CHOICES
   Don't use Apache when Nginx works better for your stack.
   Don't self-host a database you can't maintain.
   Don't use AWS when DigitalOcean would be simpler and cheaper.
   Don't use Docker when a simple PM2 setup works.

4. DEPLOYMENT ANTI-PATTERNS
   Don't deploy on Fridays.
   Don't deploy without testing.
   Don't deploy without a rollback plan.
   Don't make multiple changes in one deploy.
   Don't skip the health check after deploy.

5. COST MISTAKES
   Don't leave unused servers running.
   Don't forget to set AWS billing alerts.
   Don't start with the biggest server size.
   Don't pay for managed services you don't need yet.
```
