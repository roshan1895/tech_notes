# 07 - Full Auto Deployment (CI/CD)

## What is CI/CD?

```
CI = Continuous Integration
  - Automatically run tests when code is pushed
  - Catch bugs before they reach production

CD = Continuous Deployment/Delivery
  - Automatically deploy code after tests pass
  - Push to main -> tests pass -> deployed to server

The Pipeline:
  git push -> Run Tests -> Build -> Deploy -> Verify
                 |
              If fail -> STOP, notify developer
```

---

## CI/CD Tools Comparison

| Tool | Where it Runs | Cost | Best For |
|------|--------------|------|----------|
| **GitHub Actions** | GitHub's servers | Free (2000 min/mo) | Most projects |
| GitLab CI | GitLab's servers | Free tier available | GitLab users |
| Jenkins | Your server | Free (self-hosted) | Enterprise/custom |
| CircleCI | Their servers | Free tier | Complex pipelines |
| Vercel | Their servers | Free tier | Next.js/React |
| Netlify | Their servers | Free tier | Static sites |

**Recommendation: Start with GitHub Actions** (most popular, great free tier)

---

## GitHub Actions - Complete Guide

### How It Works
```
1. You create a .yml file in .github/workflows/
2. When you push code, GitHub runs your workflow
3. Workflow: install -> test -> build -> deploy
4. If any step fails, deployment stops
```

### Basic Structure
```
.github/
  workflows/
    deploy.yml        # Your CI/CD pipeline
```

---

### Example 1: Node.js API - Auto Deploy to VPS
```yaml
# .github/workflows/deploy.yml
name: Deploy Node.js API

on:
  push:
    branches: [main]  # Only deploy when pushing to main

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run tests
        run: npm test

  deploy:
    needs: test  # Only deploy if tests pass
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to server
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.SERVER_IP }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            cd /var/www/myapp
            git pull origin main
            npm install --production
            npm run build
            pm2 restart myapp
```

### Setting Up GitHub Secrets
```
Go to: Repository -> Settings -> Secrets and variables -> Actions

Add these secrets:
  SERVER_IP:        your-server-ip (e.g., 143.198.50.100)
  SERVER_USER:      deploy
  SSH_PRIVATE_KEY:  (contents of your private key file)

To get your SSH key content:
  cat ~/.ssh/id_rsa        # On your local machine
  # Copy the ENTIRE output including BEGIN and END lines
```

---

### Example 2: React Frontend - Build & Deploy Static Files
```yaml
# .github/workflows/deploy-react.yml
name: Deploy React App

on:
  push:
    branches: [main]

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Build
        run: npm run build
        env:
          REACT_APP_API_URL: https://api.example.com

      - name: Deploy to server
        uses: burnett01/rsync-deployments@7.0.1
        with:
          switches: -avz --delete
          path: build/
          remote_path: /var/www/myreactapp/build/
          remote_host: ${{ secrets.SERVER_IP }}
          remote_user: ${{ secrets.SERVER_USER }}
          remote_key: ${{ secrets.SSH_PRIVATE_KEY }}
```

---

### Example 3: Next.js - Full Deploy
```yaml
name: Deploy Next.js

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install & Build
        run: |
          npm ci
          npm run build

      - name: Deploy to server
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.SERVER_IP }}
          username: deploy
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            cd /var/www/mynextapp
            git pull origin main
            npm ci --production
            npm run build
            pm2 restart mynextapp
```

---

### Example 4: WordPress - Theme/Plugin Deploy
```yaml
name: Deploy WordPress Theme

on:
  push:
    branches: [main]
    paths:
      - 'wp-content/themes/mytheme/**'  # Only deploy when theme changes

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Deploy theme to server
        uses: burnett01/rsync-deployments@7.0.1
        with:
          switches: -avz --delete
          path: wp-content/themes/mytheme/
          remote_path: /var/www/wordpress/wp-content/themes/mytheme/
          remote_host: ${{ secrets.SERVER_IP }}
          remote_user: ${{ secrets.SERVER_USER }}
          remote_key: ${{ secrets.SSH_PRIVATE_KEY }}
```

---

## Vercel / Netlify (Easiest CI/CD for Frontend)

### When to Use Platform CI/CD vs Self-Hosted
```
USE VERCEL/NETLIFY:
  - Next.js or React static sites
  - Don't want to manage servers
  - Need preview deployments per PR
  - Small to medium projects
  - Cost: Free tier is generous

USE GITHUB ACTIONS + VPS:
  - Backend APIs
  - WordPress
  - Need full server control
  - Running databases
  - Multiple apps on one server
  - Cost: Only pay for VPS
```

### Vercel Setup (Next.js)
```
1. Go to vercel.com
2. Import your GitHub repo
3. Vercel auto-detects Next.js
4. Set environment variables in dashboard
5. Every push to main = auto deploy
6. Every PR = preview deployment URL

That's it. No config files needed.
```

### Netlify Setup (React Static)
```
1. Go to netlify.com
2. Import GitHub repo
3. Build command: npm run build
4. Publish directory: build (or dist)
5. Auto deploys on push to main
```

---

## Advanced CI/CD Patterns

### Staging + Production Pipeline
```yaml
name: Deploy Pipeline

on:
  push:
    branches: [main, develop]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
      - run: npm ci
      - run: npm test

  deploy-staging:
    if: github.ref == 'refs/heads/develop'
    needs: test
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to staging
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.STAGING_IP }}
          username: deploy
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            cd /var/www/myapp-staging
            git pull origin develop
            npm ci
            npm run build
            pm2 restart myapp-staging

  deploy-production:
    if: github.ref == 'refs/heads/main'
    needs: test
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to production
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.PRODUCTION_IP }}
          username: deploy
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            cd /var/www/myapp
            git pull origin main
            npm ci --production
            npm run build
            pm2 restart myapp
```

### Zero-Downtime Deployment
```bash
# On server, use PM2 reload instead of restart
# reload = starts new instances before killing old ones

# In your deploy script:
pm2 reload myapp    # Instead of: pm2 restart myapp
```

### Deployment Notifications (Slack/Discord)
```yaml
  notify:
    needs: deploy-production
    runs-on: ubuntu-latest
    if: always()
    steps:
      - name: Notify on success
        if: needs.deploy-production.result == 'success'
        run: |
          curl -X POST "${{ secrets.SLACK_WEBHOOK }}" \
            -H "Content-Type: application/json" \
            -d '{"text":"Deployment successful! Commit: ${{ github.sha }}"}'

      - name: Notify on failure
        if: needs.deploy-production.result == 'failure'
        run: |
          curl -X POST "${{ secrets.SLACK_WEBHOOK }}" \
            -H "Content-Type: application/json" \
            -d '{"text":"DEPLOYMENT FAILED! Check: ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"}'
```

---

## CI/CD Progression Path

```
Stage 1: Manual deploy scripts (you are here)
    |
Stage 2: GitHub Actions - auto test on push
    |
Stage 3: GitHub Actions - auto deploy to staging on develop branch
    |
Stage 4: GitHub Actions - auto deploy to production on main branch
    |
Stage 5: Add staging/production environments
    |
Stage 6: Add deployment notifications
    |
Stage 7: Add database migration steps
    |
Stage 8: Add zero-downtime deploys
    |
Stage 9: Container-based deploys (Docker)
```
