# Cura — Security & Compliance

**Version:** 1.0.0  
**Last Updated:** 2026-05-06  
**Standards:** OWASP Top 10, PCI DSS (via Stripe), GDPR-aware

---

## 1. Authentication & Authorization

### 1.1 Authentication Stack

Cura uses **Clerk** for identity management. Clerk handles:
- Email/password and social login (Google, Apple)
- Multi-factor authentication (TOTP, SMS)
- Session management and token rotation
- Device tracking and anomaly detection

All API requests must include a valid JWT issued by Clerk. The NestJS API validates tokens on every request via Clerk's JWT verification SDK.

### 1.2 Role-Based Access Control (RBAC)

```ts
// Four roles, enforced at middleware layer
enum UserRole {
  BUYER    // Can browse, purchase, manage shelves
  CURATOR  // + manage collections, write editorials, review submissions
  VENDOR   // + submit products, view own orders/payouts
  ADMIN    // Full access to all resources and operations
}

// NestJS guard example
@UseGuards(ClerkAuthGuard, RolesGuard)
@Roles(UserRole.ADMIN, UserRole.CURATOR)
@Patch('/products/:id')
async updateProduct(...) {}
```

### 1.3 Token Policies

- Access tokens expire in **15 minutes**
- Refresh tokens expire in **7 days**
- Refresh tokens are rotated on every use
- Sessions invalidated on password change or logout
- Admin sessions require MFA — no exceptions

---

## 2. API Security

### 2.1 Rate Limiting

Implemented at the API Gateway level using Redis-backed token buckets.

| Endpoint Group | Rate Limit |
|---|---|
| `POST /auth/*` | 10 req/min per IP |
| `GET /products*` | 300 req/min per IP |
| `POST /orders` | 20 req/min per user |
| `POST /reviews` | 5 req/min per user |
| `POST /vendor/*` | 60 req/min per vendor |
| Admin endpoints | 120 req/min per admin |

Exceeding limits returns `429 Too Many Requests` with `Retry-After` header.

### 2.2 Input Validation

All incoming request bodies are validated with `class-validator` DTOs. Validation rules:
- Strings: max length, sanitized against XSS (`class-sanitizer`)
- Integers: range-checked (e.g., `rating` must be 1–5)
- Enums: whitelist-validated only
- UUIDs/cuids: format-validated before DB queries
- File uploads: MIME type validation + file size limits (10MB images, 50MB video)

### 2.3 SQL Injection Prevention

Prisma's query builder uses parameterized queries exclusively. Raw SQL is prohibited unless reviewed and approved by a senior engineer. All raw queries must use `Prisma.$queryRaw` with tagged template literals (automatically parameterized).

### 2.4 CORS Policy

```ts
// main.ts
app.enableCors({
  origin: [
    'https://cura.com',
    'https://www.cura.com',
    process.env.NODE_ENV !== 'production' && 'http://localhost:3000',
  ].filter(Boolean),
  methods: ['GET', 'POST', 'PATCH', 'DELETE', 'OPTIONS'],
  allowedHeaders: ['Content-Type', 'Authorization'],
  credentials: true,
});
```

### 2.5 Security Headers

Applied via Next.js middleware and CloudFront response headers policy:

```
Strict-Transport-Security: max-age=63072000; includeSubDomains; preload
Content-Security-Policy: default-src 'self'; img-src 'self' res.cloudinary.com; ...
X-Frame-Options: DENY
X-Content-Type-Options: nosniff
Referrer-Policy: strict-origin-when-cross-origin
Permissions-Policy: camera=(), microphone=(), geolocation=()
```

---

## 3. Payment Security (PCI DSS)

Cura **never handles raw card data**. All payment flows use Stripe Elements (client-side) and Stripe's server-side API.

Rules:
- Card numbers, CVVs, and expiry dates **never touch Cura servers**
- Payment intent is created server-side; client completes via Stripe.js
- All Stripe webhook payloads are verified using `stripe.webhooks.constructEvent()` with the webhook signing secret
- Stripe Connect is used for vendor payouts — Cura never holds vendor funds
- PCI SAQ A compliance maintained (no cardholder data environment)

---

## 4. Data Privacy & GDPR

### 4.1 Data Minimization

Cura only collects data necessary for platform operation:
- Purchase data: required for order fulfillment
- Taste profiles: opt-in only, can be deleted at any time
- Email: required for account, can opt out of marketing separately

### 4.2 User Rights (GDPR Article 17 & 20)

| Right | Mechanism |
|---|---|
| Access | `GET /users/me/data-export` — full JSON export |
| Deletion | `DELETE /users/me` — triggers 30-day deletion job |
| Rectification | `PATCH /users/me` — self-service |
| Portability | JSON export available on demand |
| Opt-out of marketing | Unsubscribe link in every email + account settings |

### 4.3 Data Retention

| Data Type | Retention Period |
|---|---|
| Orders | 7 years (tax compliance) |
| Payment records | 7 years (financial compliance) |
| User accounts | Until deletion request |
| Taste profiles | Until deletion or opt-out |
| Application logs | 90 days |
| Access logs | 1 year |

### 4.4 Deletion Flow

When a user requests account deletion:
1. Account marked `PENDING_DELETION`
2. 30-day grace period (user can cancel)
3. After 30 days: PII anonymized (name → `[deleted]`, email → `deleted_<hash>@cura.com`)
4. Orders retained with anonymized user reference (legal requirement)
5. Taste profile, shelves, reviews: hard deleted
6. Confirmation email sent

---

## 5. Infrastructure Security

### 5.1 Network Isolation

```
Internet → CloudFront (WAF enabled)
                │
          ┌─────▼──────┐
          │ Public Subnet│ (App Runner, ALB)
          └─────┬───────┘
                │
          ┌─────▼──────┐
          │Private Subnet│ (ECS, RDS, Redis)
          └─────────────┘
```

- RDS and Redis are in private subnets with no public access
- ECS tasks communicate with RDS via VPC security groups only
- All outbound traffic from API is via NAT Gateway

### 5.2 Secrets Management

- **No secrets in code** — enforced via `git-secrets` pre-commit hook
- All production secrets stored in AWS Secrets Manager
- Secrets injected as environment variables at container startup
- Secrets are rotated every 90 days for service accounts
- IAM roles follow least-privilege principle

### 5.3 Dependency Security

```bash
# Automated via Dependabot + GitHub Actions
npm audit --audit-level=high   # Fails CI on high/critical
snyk test                       # SAST scanning on every PR
```

### 5.4 WAF Rules (CloudFront + AWS WAF)

- AWS Managed Core Rule Set enabled
- SQL injection and XSS rule groups enabled
- Rate-based rule: block IPs sending > 2000 req/5min
- Geo-blocking: phase 1 US-only, configurable per market

---

## 6. Incident Response

### Severity Levels

| Level | Definition | Response Time |
|---|---|---|
| P1 — Critical | Data breach, checkout down, auth system down | < 15 min |
| P2 — High | API degraded, payment failures > 0.5% | < 1 hr |
| P3 — Medium | Non-critical feature broken, elevated errors | < 4 hr |
| P4 — Low | Minor bug, cosmetic issue | Next sprint |

### Response Procedure

1. **Detect** — Alert fired via Datadog / PagerDuty
2. **Acknowledge** — On-call engineer acknowledges within SLA
3. **Assess** — Determine severity, create incident channel in Slack
4. **Contain** — Isolate affected systems (block IP, roll back, disable feature flag)
5. **Communicate** — Update status page (status.cura.com) within 30 min for P1
6. **Resolve** — Deploy fix or rollback
7. **Post-mortem** — Written within 48 hours for P1/P2 incidents

### Security Breach Protocol

If a data breach is suspected:
1. Immediately notify CTO and legal counsel
2. Preserve logs (do not delete)
3. Isolate affected systems
4. If user PII is involved, GDPR notification required within **72 hours**
5. Engage external security firm if scope is unclear

---

## 7. Security Audit Schedule

| Activity | Frequency |
|---|---|
| Dependency vulnerability scan | Every PR (automated) |
| Internal penetration test | Quarterly |
| External security audit | Annually |
| Access control review (IAM, RBAC) | Quarterly |
| Secrets rotation | Every 90 days |
| Security training for engineers | Annually |
