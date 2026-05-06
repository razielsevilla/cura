# Cura — Contributing & Git Workflow

**Version:** 1.0.0  
**Last Updated:** 2026-05-06

---

## 1. Branch Strategy

Cura uses a **trunk-based development** model with short-lived feature branches.

```
main          ← Production. Protected. Deploy via CI only.
staging       ← Pre-production. QA and stakeholder review.
develop       ← Integration. All feature branches merge here first.
feature/*     ← Short-lived. < 3 days lifespan target.
fix/*         ← Bug fixes.
hotfix/*      ← Critical production fixes. Branch from main.
chore/*       ← Maintenance (deps, config, docs, refactors).
```

### Branch Naming

```
feature/cura-123-product-curator-badge
fix/cura-456-checkout-quantity-overflow
hotfix/cura-789-stripe-webhook-verification
chore/upgrade-prisma-5
docs/add-api-auth-section
```

All branches reference a Jira/Linear ticket number where applicable.

---

## 2. Commit Message Convention

Cura follows the **Conventional Commits** specification.

```
<type>(<scope>): <short description>

[optional body]

[optional footer]
```

### Types

| Type | When to use |
|---|---|
| `feat` | New feature |
| `fix` | Bug fix |
| `docs` | Documentation only |
| `style` | Formatting, no logic change |
| `refactor` | Code change that is neither a fix nor a feature |
| `test` | Adding or updating tests |
| `chore` | Build, deps, config changes |
| `perf` | Performance improvement |
| `ci` | CI/CD pipeline changes |

### Examples

```
feat(products): add editorial write-up field to product detail page

fix(checkout): prevent double-submit on payment confirmation

feat(drops): implement BullMQ scheduled job for drop activation

Closes CURA-234
```

**Rules:**
- Subject line max 72 characters
- Use imperative mood: "add feature" not "added feature"
- No period at end of subject line
- Reference ticket in footer with `Closes CURA-XXX` or `Refs CURA-XXX`

---

## 3. Pull Request Process

### Before Opening a PR

- [ ] All tests pass locally (`npm run test`)
- [ ] Lint passes (`npm run lint`)
- [ ] Type check passes (`npm run type-check`)
- [ ] New code has test coverage
- [ ] No `console.log` left in code
- [ ] `.env.example` updated if new env vars added
- [ ] Database migrations tested against a local copy
- [ ] Self-review completed

### PR Template

```markdown
## Summary
Brief description of what this PR does and why.

## Changes
- Added X
- Changed Y
- Removed Z

## Type of Change
- [ ] New feature
- [ ] Bug fix
- [ ] Breaking change
- [ ] Documentation update
- [ ] Refactor

## Testing
Describe how you tested this change. Include test names if applicable.

## Screenshots (if UI change)
[Attach screenshots or Loom]

## Checklist
- [ ] Tests added/updated
- [ ] No console.logs
- [ ] Migrations reviewed
- [ ] .env.example updated (if applicable)

## Related Tickets
Closes CURA-XXX
```

### Review Requirements

| Branch Target | Required Approvals | CI Required |
|---|---|---|
| `develop` | 1 engineer | Yes |
| `staging` | 1 senior engineer | Yes |
| `main` | 2 engineers (1 senior) | Yes |

- PRs must be approved before merging — no exceptions
- Authors cannot approve their own PRs
- Stale PRs (no activity > 7 days) are auto-closed with a label

---

## 4. Code Review Standards

### For Reviewers

- Review within **24 hours** (working days)
- Leave actionable, specific feedback — not vague criticism
- Distinguish blocking (must fix) from non-blocking (suggestion): use `[blocking]` prefix
- Approve explicitly with ✅ comment when satisfied
- Be kind — code review is about the code, not the person

### For Authors

- Respond to all comments, even if just `done` or `agreed, not in scope here`
- Don't resolve threads you didn't open — let the reviewer resolve
- Re-request review after addressing feedback

### What Reviewers Check

- Correctness — does it work as described?
- Security — any new attack surfaces?
- Performance — any N+1 queries, missing indexes?
- Test coverage — is the behavior tested?
- API contract — does it break existing consumers?
- Naming — is the code self-documenting?

---

## 5. Development Environment Setup

```bash
# Prerequisites: Node 22+, Docker, pnpm

# Clone the repo
git clone https://github.com/cura-platform/cura.git
cd cura

# Install dependencies
pnpm install

# Copy env files
cp .env.example .env.local      # frontend
cp apps/api/.env.example apps/api/.env

# Start infrastructure (Postgres + Redis)
docker compose up -d

# Run DB migrations
pnpm db:migrate

# Seed development data
pnpm db:seed

# Start development servers
pnpm dev        # starts both frontend (3000) and API (4000)
```

### Repo Structure (Monorepo)

```
cura/
├── apps/
│   ├── web/        # Next.js frontend
│   └── api/        # NestJS backend
├── packages/
│   ├── types/      # Shared TypeScript types
│   ├── utils/      # Shared utility functions
│   └── config/     # Shared ESLint, TS, Tailwind config
├── docker-compose.yml
├── pnpm-workspace.yaml
└── turbo.json
```

---

## 6. Testing Standards

### Test Types

| Type | Location | Runner | Coverage Target |
|---|---|---|---|
| Unit | `*.spec.ts` alongside source | Jest | 80% line coverage |
| Integration | `test/integration/` | Jest + Supertest | Core flows |
| E2E | `test/e2e/` | Playwright | Critical user paths |

### What Must Be Tested

- All service methods (unit)
- All API endpoints (integration — happy path + error cases)
- Checkout flow, curator approval flow, drop activation (E2E)

### Running Tests

```bash
pnpm test              # All unit tests
pnpm test:integration  # Integration tests (requires DB)
pnpm test:e2e          # E2E tests (requires full stack)
pnpm test:coverage     # Coverage report
```

---

## 7. Versioning & Releases

Cura uses **semantic versioning** for the API: `MAJOR.MINOR.PATCH`

| Bump | When |
|---|---|
| PATCH | Bug fixes, internal refactors (no API changes) |
| MINOR | New endpoints, new optional fields, additive changes |
| MAJOR | Breaking changes to existing API contracts |

### Release Cadence

- **Hotfixes:** As needed. Branch from `main`, merge back to `main` + `develop`.
- **Regular releases:** Every 2 weeks from `staging` → `main`.
- **Major releases:** Planned quarterly with migration guides.

### Release Checklist

- [ ] All target tickets closed/verified in staging
- [ ] QA sign-off from staging
- [ ] DB migrations reviewed by senior engineer
- [ ] Rollback plan documented
- [ ] Status page updated
- [ ] Release notes drafted in GitHub Releases
- [ ] Post-deploy smoke test passed

---

## 8. Git Hooks (via Husky)

```bash
# Pre-commit
- lint-staged (ESLint + Prettier on staged files)
- git-secrets (no credentials in commits)

# Commit-msg
- commitlint (enforces Conventional Commits format)

# Pre-push
- npm run type-check
- npm run test (unit only, fast)
```

Setup is automatic after `pnpm install` (Husky is a devDependency).
