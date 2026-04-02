---
layout: default
title: Containers & DevOps
---

# 08 - Containers, Docker & DevOps Roadmap

## Why Containers?

### The Problem Containers Solve
```
WITHOUT CONTAINERS:
  "It works on my machine but not on the server"

  Your machine:    Node 20, npm 10, Ubuntu 22.04, specific libs
  Server:          Node 18, npm 9, Ubuntu 20.04, different libs
  Other dev:       Node 21, npm 10, macOS, different libs

  Result: Inconsistency, bugs, "works for me"

WITH CONTAINERS:
  Package your app + ALL its dependencies into one container.
  Same container runs everywhere identically.

  Your machine:    [Container: Node 20 + your app + all deps]
  Server:          [Same container]
  Other dev:       [Same container]

  Result: 100% consistency
```

---

## Docker Basics

### Core Concepts
```
IMAGE:      A blueprint/recipe (like a class)
CONTAINER:  A running instance (like an object)
DOCKERFILE: Instructions to build an image
REGISTRY:   Storage for images (Docker Hub, AWS ECR)

Flow:
  Dockerfile -> docker build -> Image -> docker run -> Container
```

### Dockerfile for Node.js
```dockerfile
# Dockerfile
FROM node:20-alpine

WORKDIR /app

# Copy package files first (caching optimization)
COPY package*.json ./
RUN npm ci --production

# Copy app code
COPY . .

# Build if needed
RUN npm run build

# Expose port
EXPOSE 3000

# Start command
CMD ["node", "app.js"]
```

### Dockerfile for Next.js
```dockerfile
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:20-alpine AS runner
WORKDIR /app
COPY --from=builder /app/.next ./.next
COPY --from=builder /app/public ./public
COPY --from=builder /app/package*.json ./
COPY --from=builder /app/node_modules ./node_modules

EXPOSE 3000
CMD ["npm", "start"]
```

### Dockerfile for React (Static)
```dockerfile
# Build stage
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Serve stage
FROM nginx:alpine
COPY --from=builder /app/build /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

### Essential Docker Commands
```bash
# Build image
docker build -t myapp:latest .

# Run container
docker run -d -p 3000:3000 --name myapp myapp:latest

# List running containers
docker ps

# View logs
docker logs myapp
docker logs -f myapp      # Follow

# Stop/start/restart
docker stop myapp
docker start myapp
docker restart myapp

# Get inside a running container
docker exec -it myapp sh

# Remove container
docker stop myapp && docker rm myapp

# Remove image
docker rmi myapp:latest

# Clean up everything unused
docker system prune -a
```

---

## Docker Compose (Multi-Container Apps)

### What It Does
Run your app + database + redis + etc. with one command.

### Example: Node.js + PostgreSQL + Redis
```yaml
# docker-compose.yml
version: '3.8'

services:
  app:
    build: .
    ports:
      - "3000:3000"
    environment:
      - DATABASE_URL=postgresql://user:pass@db:5432/mydb
      - REDIS_URL=redis://redis:6379
      - NODE_ENV=production
    depends_on:
      - db
      - redis
    restart: unless-stopped

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
      POSTGRES_DB: mydb
    volumes:
      - postgres_data:/var/lib/postgresql/data
    restart: unless-stopped

  redis:
    image: redis:7-alpine
    restart: unless-stopped

volumes:
  postgres_data:
```

### Docker Compose Commands
```bash
# Start all services
docker compose up -d

# View logs
docker compose logs -f

# Stop all
docker compose down

# Rebuild and restart
docker compose up -d --build

# View running services
docker compose ps
```

### Example: WordPress + MySQL
```yaml
version: '3.8'

services:
  wordpress:
    image: wordpress:latest
    ports:
      - "80:80"
    environment:
      WORDPRESS_DB_HOST: db
      WORDPRESS_DB_USER: wpuser
      WORDPRESS_DB_PASSWORD: wppass
      WORDPRESS_DB_NAME: wordpress
    volumes:
      - wp_data:/var/www/html
    restart: unless-stopped

  db:
    image: mysql:8
    environment:
      MYSQL_DATABASE: wordpress
      MYSQL_USER: wpuser
      MYSQL_PASSWORD: wppass
      MYSQL_ROOT_PASSWORD: rootpass
    volumes:
      - db_data:/var/lib/mysql
    restart: unless-stopped

volumes:
  wp_data:
  db_data:
```

---

## Docker Deployment to VPS

### Method 1: Build on Server
```bash
# On server
git pull origin main
docker compose down
docker compose up -d --build
```

### Method 2: Build Locally, Push to Registry
```bash
# Local machine
docker build -t username/myapp:latest .
docker push username/myapp:latest

# On server
docker pull username/myapp:latest
docker compose up -d
```

### Method 3: GitHub Actions + Docker
```yaml
name: Docker Deploy

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build and push Docker image
        run: |
          echo "${{ secrets.DOCKER_PASSWORD }}" | docker login -u "${{ secrets.DOCKER_USERNAME }}" --password-stdin
          docker build -t ${{ secrets.DOCKER_USERNAME }}/myapp:latest .
          docker push ${{ secrets.DOCKER_USERNAME }}/myapp:latest

      - name: Deploy to server
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.SERVER_IP }}
          username: deploy
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            docker pull ${{ secrets.DOCKER_USERNAME }}/myapp:latest
            cd /var/www/myapp
            docker compose down
            docker compose up -d
```

---

## Kubernetes (K8s) - When You Outgrow Docker Compose

### When You Need Kubernetes
```
Docker Compose: 1-3 servers, simple setup
Kubernetes:     10+ servers, auto-scaling, self-healing, enterprise

YOU DON'T NEED K8s IF:
  - Running < 5 services
  - < 10 servers
  - Small team
  - Not requiring auto-scaling

YOU NEED K8s WHEN:
  - Running 10+ microservices
  - Need auto-scaling based on traffic
  - Need zero-downtime rolling updates
  - Multiple teams deploying independently
  - High availability requirement
```

### Kubernetes Managed Options (Don't Self-Host K8s)
| Service | Provider | Starting Cost |
|---------|----------|---------------|
| EKS | AWS | ~$75/mo + nodes |
| GKE | Google Cloud | Free control plane + nodes |
| AKS | Azure | Free control plane + nodes |
| DOKS | DigitalOcean | $12/mo + nodes |

---

## Complete DevOps Roadmap

### Phase 1: Foundation (Where You Are)
```
[x] Linux basics (SSH, file system, permissions)
[x] Manual server setup and deployment
[x] Nginx configuration
[x] SSL/HTTPS setup
[x] Basic security (firewall, SSH keys)
[ ] Shell scripting for automation
[ ] Git workflow (branching, PRs)
```

### Phase 2: Automation
```
[ ] Deploy scripts (bash)
[ ] GitHub Actions basics
[ ] Auto-deploy on git push
[ ] Environment management (staging/production)
[ ] Basic monitoring (uptime, logs)
```

### Phase 3: Containers
```
[ ] Docker fundamentals
[ ] Dockerfile writing
[ ] Docker Compose
[ ] Container registries (Docker Hub / ECR)
[ ] Container-based deployments
```

### Phase 4: Infrastructure as Code
```
[ ] Terraform basics (define servers in code)
[ ] Ansible basics (configure servers in code)
    - Terraform: "Create 3 servers on AWS"
    - Ansible: "Install Nginx on all 3 servers"
[ ] Version-controlled infrastructure
```

### Phase 5: Orchestration
```
[ ] Kubernetes basics
[ ] Managed K8s (EKS/GKE/DOKS)
[ ] Helm charts (K8s package manager)
[ ] Service mesh concepts
```

### Phase 6: Advanced DevOps
```
[ ] Monitoring & Observability (Prometheus, Grafana)
[ ] Log aggregation (ELK Stack, Loki)
[ ] GitOps (ArgoCD, Flux)
[ ] Security scanning in CI/CD
[ ] Cost optimization
[ ] Disaster recovery
```

### Realistic Timeline
```
Phase 1: 1-2 months  (you're mostly here)
Phase 2: 1-2 months
Phase 3: 1-2 months
Phase 4: 2-3 months
Phase 5: 3-6 months
Phase 6: Ongoing

Total to "DevOps Engineer" level: 12-18 months of focused learning
```

---

## What to Learn Next (Priority Order)

```
IMMEDIATE (This month):
  1. Shell scripting - automate your current manual deploys
  2. GitHub Actions - set up auto-deploy for one project

NEXT MONTH:
  3. Docker basics - containerize one of your Node.js apps
  4. Docker Compose - add database to the container setup

NEXT QUARTER:
  5. Terraform - provision servers with code
  6. Advanced CI/CD - staging environments, rollbacks

LATER:
  7. Kubernetes (only when you actually need it)
  8. Monitoring stack
```
