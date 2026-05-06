# Cura — DevOps & Deployment Guide

**Version:** 1.0.0  
**Last Updated:** 2026-05-06  
**Cloud Provider:** AWS (primary)  
**IaC:** Terraform  
**CI/CD:** GitHub Actions

---

## 1. Infrastructure Overview

```
                        ┌──────────────────────────┐
                        │     Route 53 (DNS)       │
                        └────────────┬─────────────┘
                                     │
                        ┌────────────▼─────────────┐
                        │  CloudFront (CDN + WAF)  │
                        └────────────┬─────────────┘
                              ┌──────┴──────┐
                              │             │
                   ┌──────────▼──┐   ┌──────▼──────────┐
                   │   Next.js   │   │   NestJS API    │
                   │ (App Runner)│   │  (ECS Fargate)  │
                   └─────────────┘   └──────┬──────────┘
                                            │
                              ┌─────────────┼──────────────┐
                              │             │              │
                   ┌──────────▼──┐  ┌───────▼────┐  ┌──────▼──────┐
                   │  RDS Aurora │  │ ElastiCache│  │  S3 Bucket  │
                   │  PostgreSQL │  │  (Redis)   │  │  (Assets)   │
                   └─────────────┘  └────────────┘  └─────────────┘
```

---

## 2. AWS Services

| Service | Usage |
|---|---|
| Route 53 | DNS, health checks |
| CloudFront | CDN, WAF, SSL termination |
| App Runner | Next.js frontend (auto-scaling) |
| ECS Fargate | NestJS API containers |
| RDS Aurora PostgreSQL | Managed primary DB with read replicas |
| ElastiCache (Redis) | Session store, BullMQ, caching |
| S3 | Static assets, uploaded documents |
| CloudWatch | Logs, metrics, alarms |
| Secrets Manager | ENV secrets, API keys |
| ACM | SSL certificates |
| ECR | Docker image registry |

---

## 3. Docker

### Frontend (`Dockerfile.frontend`)

```dockerfile
FROM node:22-alpine AS base

FROM base AS deps
WORKDIR /app
COPY package*.json ./
RUN npm ci

FROM base AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npm run build

FROM base AS runner
WORKDIR /app
ENV NODE_ENV=production
RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 nextjs
COPY --from=builder /app/public ./public
COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static
USER nextjs
EXPOSE 3000
CMD ["node", "server.js"]
```

### API (`Dockerfile.api`)

```dockerfile
FROM node:22-alpine AS base

FROM base AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM base AS runner
WORKDIR /app
ENV NODE_ENV=production
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
EXPOSE 4000
CMD ["node", "dist/main.js"]
```

---

## 4. Environment Variables

### Frontend (`.env.local` / App Runner)

```env
NEXT_PUBLIC_API_URL=https://api.cura.com/v1
NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=pk_...
CLERK_SECRET_KEY=sk_...
NEXT_PUBLIC_ALGOLIA_APP_ID=...
NEXT_PUBLIC_ALGOLIA_SEARCH_KEY=...
NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY=pk_live_...
NEXT_PUBLIC_CLOUDINARY_CLOUD_NAME=cura
```

### API (`.env` / Secrets Manager)

```env
NODE_ENV=production
PORT=4000
DATABASE_URL=postgresql://...
REDIS_URL=redis://...
CLERK_SECRET_KEY=sk_...
STRIPE_SECRET_KEY=sk_live_...
STRIPE_WEBHOOK_SECRET=whsec_...
CLOUDINARY_API_KEY=...
CLOUDINARY_API_SECRET=...
ALGOLIA_APP_ID=...
ALGOLIA_ADMIN_KEY=...
RESEND_API_KEY=re_...
SENTRY_DSN=https://...
```

> **Rule:** No secrets in code or `.env` files committed to git. All production secrets live in AWS Secrets Manager.

---

## 5. CI/CD Pipeline

### GitHub Actions Workflow

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main, staging]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 22 }
      - run: npm ci
      - run: npm run lint
      - run: npm run type-check
      - run: npm run test

  build-api:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1
      - name: Login to ECR
        uses: aws-actions/amazon-ecr-login@v2
      - name: Build & Push API image
        run: |
          docker build -f Dockerfile.api -t $ECR_REPO:${{ github.sha }} .
          docker push $ECR_REPO:${{ github.sha }}

  deploy:
    needs: [test, build-api]
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to ECS
        uses: aws-actions/amazon-ecs-deploy-task-definition@v1
        with:
          task-definition: ecs-task-definition.json
          service: cura-api
          cluster: cura-${{ github.ref_name }}
          wait-for-service-stability: true
```

### Pipeline Stages

```
Push to branch
     │
     ▼
Lint + Type Check + Unit Tests   ← Fail fast
     │
     ▼
Build Docker images
     │
     ▼
Push to ECR
     │
     ▼
Run DB migrations (staging only)
     │
     ▼
Deploy to ECS / App Runner
     │
     ▼
Smoke tests (health check endpoints)
     │
     ▼
Notify Slack (#deployments)
```

---

## 6. Database Migrations

```bash
# Generate migration from schema changes
npx prisma migrate dev --name describe_your_change

# Apply migrations in production (automated in CI)
npx prisma migrate deploy

# Rollback strategy: migrations are forward-only
# Use a new migration to revert changes
```

**Rules:**
- Never edit existing migrations
- Always test migrations on a production data clone before `main`
- Additive changes (new columns with defaults) are zero-downtime
- Destructive changes (drops, renames) require a multi-phase migration

---

## 7. Monitoring & Alerting

### Datadog Dashboards

| Dashboard | Key Metrics |
|---|---|
| API Health | p95 latency, error rate, request volume |
| Database | Query time, connection pool, slow queries |
| Business | Orders/hr, GMV, failed payments |
| Infrastructure | CPU, memory, ECS task count |

### Alerts (PagerDuty via Datadog)

| Alert | Threshold | Severity |
|---|---|---|
| API error rate | > 1% for 5 min | P2 |
| p95 latency | > 500ms for 5 min | P2 |
| Checkout failures | > 0.5% for 2 min | P1 |
| DB connection pool | > 80% for 5 min | P2 |
| Pod crash loop | Any | P1 |

---

## 8. Disaster Recovery

| Scenario | RTO | RPO | Recovery Procedure |
|---|---|---|---|
| API pod crash | < 2 min | 0 | ECS auto-restarts |
| DB primary failure | < 5 min | < 1 min | Aurora auto-failover to replica |
| Full region failure | < 30 min | < 5 min | Route 53 failover to DR region |
| Data corruption | < 4 hr | < 24 hr | Restore from RDS automated snapshot |

**Backup Schedule:**
- RDS: Automated daily snapshots, 30-day retention
- Redis: Hourly snapshots to S3
- S3 assets: Cross-region replication enabled

---

## 9. Runbook: Common Operations

### Scale API pods

```bash
aws ecs update-service \
  --cluster cura-production \
  --service cura-api \
  --desired-count 6
```

### Force deployment (no code change)

```bash
aws ecs update-service \
  --cluster cura-production \
  --service cura-api \
  --force-new-deployment
```

### Access production logs

```bash
# Via AWS CLI
aws logs tail /ecs/cura-api --follow

# Via Datadog — filter: service:cura-api env:production status:error
```

### Emergency: Disable a vendor's products

```bash
# Via admin API
curl -X PATCH https://api.cura.com/v1/admin/vendors/{id}/status \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -d '{"status": "suspended"}'
```
