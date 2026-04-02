---
layout: default
title: Overview
---

# Server Deployment Study Guide - Master Index

## Your Background
- Deployed 10-20 servers manually (AWS, DigitalOcean)
- Tech stack: WordPress, Node.js, React, Next.js, PHP
- Goal: Master architecture -> Move manual to auto -> DevOps roadmap

---

## Study Order (Follow This Sequence)

| # | File | What You Learn |
|---|------|---------------|
| 1 | `01-LAYERED-ARCHITECTURE.md` | The full stack of layers from hardware to app |
| 2 | `02-NETWORK-LAYER.md` | DNS, ports, firewalls, SSL, CDN, reverse proxy |
| 3 | `03-MANUAL-DEPLOYMENT-MASTERY.md` | Proper manual deployment (your current stage) |
| 4 | `04-STACK-SPECIFIC-GUIDES.md` | WordPress, Node, React, Next.js, PHP specifics |
| 5 | `05-DEBUGGING-AND-SECURITY.md` | Systematic debugging + security hardening |
| 6 | `06-SEMI-AUTO-DEPLOYMENT.md` | Shell scripts, basic automation |
| 7 | `07-FULL-AUTO-DEPLOYMENT.md` | CI/CD pipelines, GitHub Actions |
| 8 | `08-CONTAINERS-AND-DEVOPS.md` | Docker, Kubernetes, full DevOps roadmap |
| 9 | `09-LOAD-BALANCERS-AND-SCALING.md` | When to scale, LB setup, cost management |
| 10 | `10-CHECKLISTS.md` | Pre/post deployment checklists for every stage |
| 11 | `11-SMART-DEPLOYMENT-TEMPLATES.md` | Reusable scripts to deploy in minutes |
| 12 | `12-DECISION-MAKING-FRAMEWORK.md` | How to choose: which server, which approach |
| 13 | `13-OS-LAYER.md` | Linux deep dive: filesystem, users, permissions, processes, systemd, memory, disk, cron |

---

## The Big Picture (Read This First)

### What Actually Happens When You Deploy a Server?

```
YOUR CODE
   |
   v
[1. Cloud Provider] --> You rent a machine (AWS EC2 / DO Droplet)
   |
   v
[2. OS Layer]       --> Ubuntu/Debian/Amazon Linux installed
   |
   v
[3. Network Layer]  --> IP assigned, DNS pointed, ports opened
   |
   v
[4. Security Layer] --> SSH keys, firewall (ufw/iptables), fail2ban
   |
   v
[5. Runtime Layer]  --> Node.js/PHP/Python installed
   |
   v
[6. Web Server]     --> Nginx/Apache configured (reverse proxy)
   |
   v
[7. App Layer]      --> Your code deployed, dependencies installed
   |
   v
[8. Process Mgmt]   --> PM2/systemd keeps your app alive
   |
   v
[9. SSL Layer]      --> Let's Encrypt / Certbot for HTTPS
   |
   v
[10. DNS Layer]     --> Domain -> IP mapping (A record / CNAME)
   |
   v
[11. Monitoring]    --> Logs, uptime checks, alerts
```

### Why Things Break (Common Pattern)
Most deployment issues fall into 3 categories:
1. **Network/Port issues** - App runs but can't be reached (firewall, wrong port, DNS)
2. **Permission issues** - Files owned by wrong user, can't write to directory
3. **Process issues** - App crashes and nobody restarts it (no PM2/systemd)

Understanding these 3 categories solves 80% of deployment debugging.
