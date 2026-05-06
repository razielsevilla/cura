# Cura — Database Schema & ERD

**Version:** 1.0.0  
**Last Updated:** 2026-05-06  
**Database:** PostgreSQL 16  
**ORM:** Prisma 5.x

---

## 1. Schema Overview

```
users ──────────────── taste_profiles
  │                          
  ├── shelves ─────── shelf_items ──── products
  │                                       │
  ├── orders ──────── order_items ────────┤
  │                                       │
  ├── reviews ────────────────────────────┤
  │                                       │
curator_profiles ──── curated_products ───┤
  │                                       │
  └── collections ── collection_products ─┘
                                           │
vendors ──── vendor_products ──────────────┤
                                           │
                                    product_variants
                                    categories
```

---

## 2. Prisma Schema

```prisma
// schema.prisma
// datasource and generator blocks omitted for brevity

// ─────────────────────────────────────────
// USERS & AUTH
// ─────────────────────────────────────────

model User {
  id            String    @id @default(cuid())
  clerkId       String    @unique
  email         String    @unique
  name          String
  avatarUrl     String?
  role          UserRole  @default(BUYER)
  createdAt     DateTime  @default(now())
  updatedAt     DateTime  @updatedAt

  tasteProfile  TasteProfile?
  shelves       Shelf[]
  orders        Order[]
  reviews       Review[]
  following     CuratorFollow[]
  curatorProfile CuratorProfile?
  vendorProfile  VendorProfile?

  @@index([email])
  @@index([clerkId])
}

enum UserRole {
  BUYER
  CURATOR
  VENDOR
  ADMIN
}

// ─────────────────────────────────────────
// TASTE PROFILES
// ─────────────────────────────────────────

model TasteProfile {
  id          String   @id @default(cuid())
  userId      String   @unique
  aesthetic   String[] // e.g. ["minimalist", "wabi-sabi"]
  values      String[] // e.g. ["sustainability", "craftsmanship"]
  lifestyle   String[] // e.g. ["home", "cooking"]
  isPublic    Boolean  @default(false)
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt

  user        User     @relation(fields: [userId], references: [id], onDelete: Cascade)
}

// ─────────────────────────────────────────
// CURATORS
// ─────────────────────────────────────────

model CuratorProfile {
  id                String    @id @default(cuid())
  userId            String    @unique
  bio               String
  specializations   String[]  // e.g. ["kitchen", "textiles"]
  reputationScore   Float     @default(0)
  followerCount     Int       @default(0)
  isVerified        Boolean   @default(false)
  createdAt         DateTime  @default(now())
  updatedAt         DateTime  @updatedAt

  user              User            @relation(fields: [userId], references: [id])
  products          CuratedProduct[]
  collections       Collection[]
  followers         CuratorFollow[]

  @@index([reputationScore])
}

model CuratorFollow {
  id          String   @id @default(cuid())
  userId      String
  curatorId   String
  createdAt   DateTime @default(now())

  user        User            @relation(fields: [userId], references: [id], onDelete: Cascade)
  curator     CuratorProfile  @relation(fields: [curatorId], references: [id], onDelete: Cascade)

  @@unique([userId, curatorId])
}

// ─────────────────────────────────────────
// VENDORS
// ─────────────────────────────────────────

model VendorProfile {
  id                  String    @id @default(cuid())
  userId              String    @unique
  brandName           String
  website             String?
  stripeAccountId     String?   @unique
  payoutEnabled       Boolean   @default(false)
  createdAt           DateTime  @default(now())
  updatedAt           DateTime  @updatedAt

  user                User      @relation(fields: [userId], references: [id])
  submissions         ProductSubmission[]
  products            Product[]
}

// ─────────────────────────────────────────
// CATEGORIES
// ─────────────────────────────────────────

model Category {
  id          String     @id @default(cuid())
  name        String
  slug        String     @unique
  parentId    String?
  parent      Category?  @relation("CategoryTree", fields: [parentId], references: [id])
  children    Category[] @relation("CategoryTree")
  products    Product[]
}

// ─────────────────────────────────────────
// PRODUCTS
// ─────────────────────────────────────────

model Product {
  id              String        @id @default(cuid())
  name            String
  slug            String        @unique
  description     String
  editorial       String        // Short editorial blurb on product card
  whyWeChoseThis  String        // Long-form curator write-up
  priceCents      Int
  currency        String        @default("USD")
  status          ProductStatus @default(PENDING)
  categoryId      String
  vendorId        String
  createdAt       DateTime      @default(now())
  updatedAt       DateTime      @updatedAt

  category          Category          @relation(fields: [categoryId], references: [id])
  vendor            VendorProfile     @relation(fields: [vendorId], references: [id])
  variants          ProductVariant[]
  images            ProductImage[]
  curatedProducts   CuratedProduct[]
  collectionItems   CollectionProduct[]
  shelfItems        ShelfItem[]
  orderItems        OrderItem[]
  reviews           Review[]

  @@index([status])
  @@index([categoryId])
  @@index([vendorId])
}

enum ProductStatus {
  PENDING
  IN_REVIEW
  APPROVED
  LIVE
  ARCHIVED
  REJECTED
}

model ProductVariant {
  id          String  @id @default(cuid())
  productId   String
  label       String  // e.g. "Stone Grey - M"
  sku         String  @unique
  stock       Int     @default(0)
  priceDelta  Int     @default(0) // Adjustment from base price in cents

  product     Product     @relation(fields: [productId], references: [id], onDelete: Cascade)
  orderItems  OrderItem[]

  @@index([productId])
}

model ProductImage {
  id            String  @id @default(cuid())
  productId     String
  cloudinaryId  String
  altText       String?
  position      Int     @default(0)

  product       Product @relation(fields: [productId], references: [id], onDelete: Cascade)
}

model CuratedProduct {
  id          String   @id @default(cuid())
  productId   String
  curatorId   String
  note        String?  // Additional curator note
  curatedAt   DateTime @default(now())

  product     Product        @relation(fields: [productId], references: [id])
  curator     CuratorProfile @relation(fields: [curatorId], references: [id])

  @@unique([productId, curatorId])
}

// ─────────────────────────────────────────
// PRODUCT SUBMISSIONS
// ─────────────────────────────────────────

model ProductSubmission {
  id          String           @id @default(cuid())
  vendorId    String
  data        Json             // Submitted product data snapshot
  status      SubmissionStatus @default(PENDING)
  feedback    String?
  reviewedAt  DateTime?
  reviewedBy  String?          // Admin user ID
  createdAt   DateTime         @default(now())

  vendor      VendorProfile @relation(fields: [vendorId], references: [id])
}

enum SubmissionStatus {
  PENDING
  IN_REVIEW
  APPROVED
  REJECTED
}

// ─────────────────────────────────────────
// COLLECTIONS & DROPS
// ─────────────────────────────────────────

model Collection {
  id          String         @id @default(cuid())
  name        String
  slug        String         @unique
  description String
  type        CollectionType @default(EVERGREEN)
  dropAt      DateTime?      // Only for DROP type
  curatorId   String
  createdAt   DateTime       @default(now())
  updatedAt   DateTime       @updatedAt

  curator     CuratorProfile      @relation(fields: [curatorId], references: [id])
  products    CollectionProduct[]
  waitlist    DropWaitlist[]
}

enum CollectionType {
  DROP
  EVERGREEN
  SEASONAL
}

model CollectionProduct {
  id           String   @id @default(cuid())
  collectionId String
  productId    String
  position     Int      @default(0)

  collection   Collection @relation(fields: [collectionId], references: [id], onDelete: Cascade)
  product      Product    @relation(fields: [productId], references: [id])

  @@unique([collectionId, productId])
}

model DropWaitlist {
  id           String   @id @default(cuid())
  collectionId String
  userId       String
  joinedAt     DateTime @default(now())

  collection   Collection @relation(fields: [collectionId], references: [id])

  @@unique([collectionId, userId])
}

// ─────────────────────────────────────────
// SHELVES
// ─────────────────────────────────────────

model Shelf {
  id          String          @id @default(cuid())
  userId      String
  name        String
  visibility  ShelfVisibility @default(PRIVATE)
  createdAt   DateTime        @default(now())
  updatedAt   DateTime        @updatedAt

  user        User        @relation(fields: [userId], references: [id], onDelete: Cascade)
  items       ShelfItem[]

  @@index([userId])
}

enum ShelfVisibility {
  PUBLIC
  PRIVATE
  FOLLOWERS_ONLY
}

model ShelfItem {
  id        String   @id @default(cuid())
  shelfId   String
  productId String
  addedAt   DateTime @default(now())

  shelf     Shelf   @relation(fields: [shelfId], references: [id], onDelete: Cascade)
  product   Product @relation(fields: [productId], references: [id])

  @@unique([shelfId, productId])
}

// ─────────────────────────────────────────
// ORDERS
// ─────────────────────────────────────────

model Order {
  id                    String      @id @default(cuid())
  userId                String
  status                OrderStatus @default(PENDING_PAYMENT)
  totalCents            Int
  currency              String      @default("USD")
  stripePaymentIntentId String?     @unique
  shippingAddress       Json
  createdAt             DateTime    @default(now())
  updatedAt             DateTime    @updatedAt

  user        User        @relation(fields: [userId], references: [id])
  items       OrderItem[]

  @@index([userId])
  @@index([status])
}

enum OrderStatus {
  PENDING_PAYMENT
  PAID
  PROCESSING
  SHIPPED
  DELIVERED
  REFUNDED
  CANCELLED
}

model OrderItem {
  id          String @id @default(cuid())
  orderId     String
  productId   String
  variantId   String?
  quantity    Int
  priceCents  Int    // Snapshot at time of purchase

  order       Order          @relation(fields: [orderId], references: [id], onDelete: Cascade)
  product     Product        @relation(fields: [productId], references: [id])
  variant     ProductVariant? @relation(fields: [variantId], references: [id])
}

// ─────────────────────────────────────────
// REVIEWS
// ─────────────────────────────────────────

model Review {
  id            String   @id @default(cuid())
  productId     String
  userId        String
  rating        Int      // 1–5
  body          String
  mediaIds      String[] // Cloudinary IDs
  isVerified    Boolean  @default(false) // Verified purchase
  createdAt     DateTime @default(now())

  product       Product @relation(fields: [productId], references: [id])
  user          User    @relation(fields: [userId], references: [id])

  @@unique([productId, userId])
  @@index([productId])
}
```

---

## 3. Indexing Strategy

| Table | Index | Reason |
|---|---|---|
| `users` | `email`, `clerkId` | Auth lookups |
| `products` | `status`, `categoryId`, `vendorId` | Catalog filtering |
| `orders` | `userId`, `status` | Buyer order history |
| `reviews` | `productId` | Product rating aggregation |
| `curator_profiles` | `reputationScore` | Leaderboard sorting |

---

## 4. Soft Deletes

Products, curators, and users use soft deletes via `status: ARCHIVED` or `deletedAt` timestamps. Hard deletes are admin-only and only permitted for GDPR data deletion requests.

---

## 5. Migrations Strategy

- All schema changes go through Prisma migrations
- Migrations are reviewed and approved before merging to `main`
- Non-breaking changes (additive) can deploy without downtime
- Breaking changes (column removal, type changes) require a multi-step migration with shadow deployment
