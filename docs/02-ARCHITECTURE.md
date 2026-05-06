# Cura — System Architecture & Tech Stack

**Version:** 1.0.0  
**Last Updated:** 2026-05-06  
**Status:** Active

---

## 1. Architecture Overview

Cura is built as a **modular monolith** with clean domain boundaries — enabling rapid development in early phases while supporting decomposition into microservices when scale demands it. All services communicate internally via well-defined interfaces, making extraction straightforward.

```
┌─────────────────────────────────────────────────────────┐
│                      CDN / Edge                         │
│                  (Cloudflare / AWS CF)                  │
└────────────────────────┬────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────┐
│                   Next.js Frontend                      │
│            (App Router, SSR/SSG/ISR, PWA)               │
└────────────────────────┬────────────────────────────────┘
                         │ REST / GraphQL
┌────────────────────────▼────────────────────────────────┐
│                  NestJS API Gateway                     │
│         (Auth Middleware, Rate Limiting, Logging)       │
└──┬──────────────┬──────────────┬───────────────┬────────┘
   │              │              │               │
┌──▼────┐   ┌─────▼──┐   ┌──────▼──┐   ┌────────▼──┐
│Catalog│   │ Orders │   │Curation │   │  Users /  │
│Module │   │ Module │   │ Module  │   │ Profiles  │
└──┬────┘   └─────┬──┘   └──────┬──┘   └────────┬──┘
   │              │              │               │
┌──▼──────────────▼──────────────▼───────────────▼───────┐
│                   PostgreSQL (Primary DB)              │
│                   Redis (Cache / Sessions / Queues)    │
└────────────────────────────────────────────────────────┘
         │                  │                  │
    ┌────▼───┐         ┌────▼────┐       ┌─────▼────┐
    │Algolia │         │Stripe   │       │Cloudinary│
    │(Search)│         │(Payments│       │(Media)   │
    └────────┘         └─────────┘       └──────────┘
```

---

## 2. Tech Stack

### 2.1 Frontend

| Technology | Version | Purpose |
|---|---|---|
| Next.js | 15.x | App Router, SSR/SSG, API routes |
| TypeScript | 5.x | Type safety across the stack |
| Tailwind CSS | 4.x | Utility-first styling |
| shadcn/ui | latest | Accessible component primitives |
| Zustand | 5.x | Client-side state management |
| React Query (TanStack) | 5.x | Server state, caching, prefetching |
| Framer Motion | 11.x | Animations and transitions |

### 2.2 Backend

| Technology | Version | Purpose |
|---|---|---|
| Node.js | 22.x LTS | Runtime |
| NestJS | 11.x | Modular framework, DI, decorators |
| TypeScript | 5.x | Type safety |
| Prisma | 5.x | ORM, schema management, migrations |
| BullMQ | 5.x | Job queues (drops, notifications, emails) |
| Passport.js | 0.7.x | Auth strategy integration |
| class-validator | 0.14.x | DTO validation |
| Zod | 3.x | Runtime schema validation |

### 2.3 Database & Storage

| Technology | Purpose |
|---|---|
| PostgreSQL 16 | Primary relational database |
| Redis 7 | Session store, caching, BullMQ broker |
| Cloudinary | Image/video upload, transformation, CDN |
| AWS S3 | Document and raw asset storage |

### 2.4 Search

| Technology | Purpose |
|---|---|
| Algolia | Product search with curator-aware faceting |
| Typesense (fallback) | Self-hosted alternative for cost control |

### 2.5 Payments

| Technology | Purpose |
|---|---|
| Stripe | Checkout, subscriptions, Connect (vendor payouts) |
| Stripe Webhooks | Order lifecycle events |

### 2.6 Auth

| Technology | Purpose |
|---|---|
| Clerk | Multi-role auth (buyer, curator, vendor, admin) |
| JWT | API token validation |
| RBAC middleware | Route-level role enforcement |

### 2.7 Email & Notifications

| Technology | Purpose |
|---|---|
| Resend | Transactional email |
| React Email | Email templates |
| OneSignal | Push notifications (web) |

### 2.8 Infrastructure

| Technology | Purpose |
|---|---|
| AWS (primary) | EC2, RDS, ElastiCache, S3, CloudFront |
| Terraform | Infrastructure as Code |
| Docker | Containerization |
| GitHub Actions | CI/CD pipelines |
| Datadog | APM, logs, metrics, alerting |
| Sentry | Error tracking |

---

## 3. Environments

| Environment | Purpose | Branch |
|---|---|---|
| `local` | Developer machines | `feature/*` |
| `dev` | Shared development, integration testing | `develop` |
| `staging` | Pre-production, QA, stakeholder review | `staging` |
| `production` | Live platform | `main` |

---

## 4. Key Architectural Decisions

### 4.1 Modular Monolith First
We start with a modular monolith to reduce operational complexity in early phases. Each module (catalog, orders, curation, users) has strict boundaries: no cross-module database queries, communication via service interfaces only. This allows future extraction to microservices without rewriting business logic.

### 4.2 SSR + ISR for SEO
Product pages and curator profiles are rendered server-side or via Incremental Static Regeneration for maximum SEO performance. Editorial content is a core growth driver; every product page functions as a landing page.

### 4.3 Event-Driven Drop System
Collection drops use BullMQ scheduled jobs. When a drop is scheduled, a job is created to activate listings and notify waitlisted buyers at the exact scheduled time, independent of user request cycles.

### 4.4 Curator-Aware Search
Algolia indices include curator metadata as facets. Buyers can filter by curator, specialty, or collection. Search ranking is weighted by curator reputation score, not just sales volume.

---

## 5. Non-Functional Requirements

| Requirement | Target |
|---|---|
| API response time (p95) | < 200ms |
| Page load (LCP) | < 2.5s |
| Uptime SLA | 99.9% |
| Checkout flow availability | 99.99% |
| Data retention (orders) | 7 years |
| GDPR / data deletion SLA | 30 days |
