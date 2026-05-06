# Cura — Product Requirements Document (PRD)

**Version:** 1.0.0  
**Last Updated:** 2026-05-06  
**Status:** Active  
**Owner:** Engineering & Product

---

## 1. Executive Summary

Cura is a hyper-niche, curated physical goods ecommerce platform that differentiates itself through expert curation, editorial storytelling, and community-driven taste profiles. Unlike traditional ecommerce marketplaces, Cura does not allow open product listings. Every product earns its place through a structured curator review process, ensuring that every item on the platform is intentional, meaningful, and high-quality.

**Tagline:** *Shop with intention. Buy what matters.*

---

## 2. Problem Statement

Modern ecommerce is overwhelmed with noise — infinite product listings, fake reviews, and algorithmically-driven discovery that optimizes for clicks over quality. Consumers are fatigued by choice and distrust generic recommendations. There is no trusted destination for curated, expert-backed physical goods discovery.

---

## 3. Goals & Objectives

### Business Goals
- Build the most trusted curated physical goods platform globally
- Achieve $1M GMV within 12 months of launch
- Onboard 50+ verified curators across 10+ product categories in Year 1

### Product Goals
- Reduce time-to-purchase decision by 40% vs. traditional ecommerce through curation
- Achieve >4.5 average curator trust score across all categories
- Maintain <2% return rate through quality-gated product approval

### Non-Goals (v1)
- Marketplace model (open seller listings) — not supported in v1
- Digital products — physical goods only
- Auction/bidding model — fixed pricing only

---

## 4. Users & Roles

### 4.1 Buyer
A registered user who browses, saves, and purchases products on Cura. Buyers have taste profiles and shelves.

### 4.2 Curator
A vetted expert who reviews, approves, and contextualizes products with editorial content. Curators have public profiles, specializations, and reputation scores.

### 4.3 Vendor
A brand or seller who submits products for curatorial review. Vendors do not have a public-facing storefront — their products appear only within Cura's curated collections.

### 4.4 Admin
Internal Cura staff who manage curators, vendors, platform health, and compliance.

---

## 5. User Stories

### Buyer
- As a buyer, I want to browse curated collections so that I can discover products I can trust.
- As a buyer, I want to follow curators whose taste I align with so that I get personalized discovery.
- As a buyer, I want to build a public or private shelf of saved items so that I can plan purchases and share recommendations.
- As a buyer, I want to see the editorial context behind every product so that I understand *why* it was chosen.
- As a buyer, I want to be notified of upcoming collection drops so that I can access limited items early.

### Curator
- As a curator, I want to submit product reviews and "Why We Chose This" write-ups so that buyers get context.
- As a curator, I want to see my follower count and reputation score so that I can track my influence.
- As a curator, I want to manage my active collections and upcoming drops so that I can plan editorial calendars.

### Vendor
- As a vendor, I want to submit products for curatorial review so that they can be featured on Cura.
- As a vendor, I want to receive structured feedback when a product is rejected so that I can improve and resubmit.
- As a vendor, I want to track orders, inventory, and payouts through a vendor portal.

### Admin
- As an admin, I want to approve or reject curator applications so that only trusted experts join the platform.
- As an admin, I want to view platform-level analytics so that I can track GMV, conversion, and retention.

---

## 6. Features

### 6.1 Core Commerce
| Feature | Priority | Notes |
|---|---|---|
| Product catalog with curator metadata | P0 | Core differentiator |
| Cart and checkout (Stripe) | P0 | Single-step checkout preferred |
| Order management | P0 | Status tracking, email notifications |
| Inventory management | P0 | Per-SKU, multi-variant |
| Vendor portal | P1 | Submission, payout, analytics |

### 6.2 Curation Engine
| Feature | Priority | Notes |
|---|---|---|
| Product submission & review flow | P0 | Multi-stage: submit → review → approve/reject |
| "Why We Chose This" editorial content | P0 | Required for all live products |
| Curator profiles | P0 | Public, with reputation score |
| Collection management | P1 | Themed groupings, drops, evergreen |
| Drop scheduling & waitlists | P1 | Time-gated access |

### 6.3 Personalization
| Feature | Priority | Notes |
|---|---|---|
| Taste profile onboarding quiz | P1 | Aesthetic, values, lifestyle axes |
| Curator following | P1 | Activity feed driven by followed curators |
| Shelf (wishlist 2.0) | P1 | Public/private, shareable |

### 6.4 Reviews & Trust
| Feature | Priority | Notes |
|---|---|---|
| Verified purchase reviews | P1 | Photo/video supported |
| No anonymous or unverified reviews | P0 | Platform policy |
| Curator trust score | P1 | Computed from buyer ratings + sales data |

---

## 7. Out of Scope (v1)

- Mobile native apps (web-first, PWA in v2)
- Subscription boxes
- International shipping (US-only launch)
- Crypto payments
- Live shopping / video commerce

---

## 8. Success Metrics

| Metric | Target (6 months) |
|---|---|
| Monthly Active Buyers | 10,000 |
| GMV | $250,000 |
| Curator count | 25+ |
| Approved products | 500+ |
| Shelf saves per session | >2 |
| NPS | >50 |

---

## 9. Risks & Mitigations

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Curator quality inconsistency | Medium | High | Structured onboarding + reputation system |
| Vendor supply chain issues | Medium | Medium | Require verified inventory before approval |
| Low organic discovery | High | High | SEO-first editorial content strategy |
| Fake verified reviews | Low | High | Purchase verification + photo requirement |
