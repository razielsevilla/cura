# Cura — Frontend Component Guide

**Version:** 1.0.0  
**Last Updated:** 2026-05-06  
**Framework:** Next.js 15 (App Router) + TypeScript + Tailwind CSS 4 + shadcn/ui

---

## 1. Project Structure

```
src/
├── app/                        # Next.js App Router
│   ├── (auth)/                 # Auth group layout
│   │   ├── sign-in/
│   │   └── sign-up/
│   ├── (shop)/                 # Main shop layout
│   │   ├── page.tsx            # Home / featured
│   │   ├── products/
│   │   │   ├── page.tsx        # Catalog
│   │   │   └── [slug]/page.tsx # Product detail
│   │   ├── collections/
│   │   │   ├── page.tsx
│   │   │   └── [slug]/page.tsx
│   │   ├── curators/
│   │   │   ├── page.tsx
│   │   │   └── [id]/page.tsx
│   │   ├── shelves/[id]/page.tsx
│   │   └── account/
│   ├── (vendor)/               # Vendor portal layout
│   ├── (admin)/                # Admin layout
│   └── api/                    # API routes (webhooks, etc.)
│
├── components/
│   ├── ui/                     # shadcn/ui primitives (do not edit)
│   ├── layout/                 # Header, Footer, Sidebar, Navigation
│   ├── product/                # ProductCard, ProductDetail, ProductGrid
│   ├── curator/                # CuratorCard, CuratorProfile, CuratorBadge
│   ├── collection/             # CollectionCard, DropTimer, CollectionGrid
│   ├── shelf/                  # ShelfCard, ShelfDrawer, ShelfItem
│   ├── order/                  # OrderSummary, OrderItem, CheckoutFlow
│   ├── taste/                  # TasteQuiz, TasteTag
│   └── shared/                 # Avatar, Badge, Rating, Price, EmptyState
│
├── lib/
│   ├── api/                    # API client, hooks
│   ├── store/                  # Zustand stores
│   ├── utils/                  # Formatting, validation helpers
│   └── constants/              # App-wide constants
│
├── hooks/                      # Custom React hooks
├── types/                      # Global TypeScript types
└── styles/                     # Global styles, design tokens
```

---

## 2. Design Tokens

```css
/* styles/tokens.css */
:root {
  /* Colors */
  --color-sand:      #F5F0EA;
  --color-clay:      #C4A882;
  --color-bark:      #5C3D2E;
  --color-ink:       #1A1A1A;
  --color-mist:      #E8E4DF;
  --color-surface:   #FFFFFF;
  --color-accent:    #3D6B4F;  /* Curator green */
  --color-drop:      #8B3A3A;  /* Drop red */

  /* Typography */
  --font-display:    'Playfair Display', Georgia, serif;
  --font-body:       'Inter', system-ui, sans-serif;
  --font-mono:       'JetBrains Mono', monospace;

  /* Spacing (8px base) */
  --space-1:  8px;
  --space-2:  16px;
  --space-3:  24px;
  --space-4:  32px;
  --space-6:  48px;
  --space-8:  64px;

  /* Border radius */
  --radius-sm:  4px;
  --radius-md:  8px;
  --radius-lg:  16px;

  /* Shadows */
  --shadow-card:  0 1px 3px rgba(0,0,0,0.08), 0 4px 12px rgba(0,0,0,0.04);
  --shadow-modal: 0 8px 32px rgba(0,0,0,0.16);
}
```

---

## 3. Core Components

### 3.1 ProductCard

Displays a product in a grid. Used on catalog, collection, and shelf pages.

```tsx
// components/product/ProductCard.tsx
interface ProductCardProps {
  product: {
    id: string;
    name: string;
    slug: string;
    priceCents: number;
    images: string[];
    editorial: string;
    curator: { name: string; avatarUrl: string };
    inStock: boolean;
    collection?: { name: string };
  };
  size?: 'sm' | 'md' | 'lg';
  onShelf?: (productId: string) => void;
}
```

**Variants:**
- `sm` — Compact grid card (4 per row)
- `md` — Standard grid card (3 per row) — default
- `lg` — Feature card (1–2 per row, used in drops)

**States:** Default, Hover (image zoom + action reveal), Out of Stock (muted overlay), Loading (skeleton).

---

### 3.2 CuratorCard

Displays curator summary. Used in curator directory and on product pages.

```tsx
interface CuratorCardProps {
  curator: {
    id: string;
    name: string;
    avatarUrl: string;
    bio: string;
    specializations: string[];
    followerCount: number;
    reputationScore: number;
  };
  isFollowing?: boolean;
  onFollow?: (curatorId: string) => void;
}
```

---

### 3.3 CollectionCard

Displays a collection/drop with countdown timer for drop type.

```tsx
interface CollectionCardProps {
  collection: {
    id: string;
    name: string;
    slug: string;
    type: 'drop' | 'evergreen' | 'seasonal';
    dropAt?: string;
    curator: { name: string; avatarUrl: string };
    previewImages: string[];
    productCount: number;
  };
}
```

Drop collections show a live countdown timer and "Join Waitlist" CTA before the drop time.

---

### 3.4 DropTimer

```tsx
// components/collection/DropTimer.tsx
interface DropTimerProps {
  dropAt: string;  // ISO date string
  onExpire?: () => void;
}
```

Renders `DD:HH:MM:SS` countdown. On expiry, calls `onExpire` (triggers collection refresh).

---

### 3.5 TasteQuiz

Multi-step onboarding quiz for building a buyer's taste profile. Uses a wizard pattern with step indicators.

```tsx
interface TasteQuizProps {
  onComplete: (profile: TasteProfileInput) => void;
  initialValues?: Partial<TasteProfileInput>;
}

interface TasteProfileInput {
  aesthetic: string[];
  values: string[];
  lifestyle: string[];
}
```

**Steps:**
1. Aesthetic (visual style preference — image-based selection)
2. Values (sustainability, craftsmanship, local, etc.)
3. Lifestyle (home, cooking, travel, fashion, etc.)

---

### 3.6 CheckoutFlow

Multi-step checkout: Cart Review → Shipping → Payment → Confirmation.

```tsx
interface CheckoutFlowProps {
  items: CartItem[];
  onSuccess: (orderId: string) => void;
}
```

Payment step embeds Stripe Elements. Never stores card data client-side.

---

### 3.7 ShelfDrawer

Slide-in drawer for saving a product to a shelf. Shows all user shelves with option to create new.

```tsx
interface ShelfDrawerProps {
  productId: string;
  isOpen: boolean;
  onClose: () => void;
}
```

---

## 4. Page Patterns

### 4.1 Product Detail Page (`/products/[slug]`)

Structure:
```
<ProductHero>           ← Full-width image gallery + key info
<EditorialBlock>        ← "Why We Chose This" — serif, long-form
<CuratorStrip>          ← Curator card + follow button
<ProductSpecs>          ← Materials, dimensions, origin
<PairsWellWith>         ← Curated companion products
<VerifiedReviews>       ← Photo reviews only
<CollectionContext>     ← Which collection this belongs to
```

### 4.2 Curator Profile (`/curators/[id]`)

Structure:
```
<CuratorHero>           ← Large avatar + bio + stats + follow
<CollectionList>        ← Curated collections, filterable
<FeaturedProducts>      ← Top curator products
<ReputationBadges>      ← Specialty badges
```

### 4.3 Home Page

Structure:
```
<HeroFeature>           ← Latest drop or editorial feature
<ActiveDropBanner>      ← Countdown + waitlist CTA
<FeaturedCurators>      ← 3–4 curator spotlights
<NewArrivals>           ← Recent live products
<EditorialSection>      ← Static/CMS-driven article
```

---

## 5. State Management

### Zustand Stores

```ts
// lib/store/cart.store.ts
interface CartStore {
  items: CartItem[];
  addItem: (item: CartItem) => void;
  removeItem: (productId: string, variantId?: string) => void;
  clearCart: () => void;
  total: number;
}

// lib/store/ui.store.ts
interface UIStore {
  shelfDrawer: { open: boolean; productId: string | null };
  openShelfDrawer: (productId: string) => void;
  closeShelfDrawer: () => void;
}
```

---

## 6. Data Fetching Conventions

- **Server Components** — use for static or low-churn data (product pages, curator profiles)
- **React Query** — use for user-specific or interactive data (cart, shelves, following state)
- **Optimistic updates** — required for follow/unfollow and shelf save actions
- **Error boundaries** — wrap all async page sections

```tsx
// Preferred server component fetch pattern
async function ProductPage({ params }: { params: { slug: string } }) {
  const product = await getProduct(params.slug); // Direct DB or API call
  if (!product) notFound();
  return <ProductDetail product={product} />;
}
```

---

## 7. Accessibility Standards

- All interactive elements must have ARIA labels
- Color contrast: minimum AA (4.5:1 for body text)
- Keyboard navigation: full tab order support, focus rings visible
- Images: `alt` text required; decorative images use `alt=""`
- Forms: every input has an associated `<label>`
- Error messages are associated with their field via `aria-describedby`

---

## 8. Performance Guidelines

- Images: always use `next/image` with explicit `width` and `height`
- Fonts: use `next/font` with `display: swap`
- Large components: lazy load with `dynamic()` and Suspense boundaries
- Route prefetching: enabled by default via `next/link`
- Bundle analysis: run `ANALYZE=true next build` before every release
