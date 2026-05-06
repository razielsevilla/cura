# Cura — API Design & Endpoints

**Version:** 1.0.0  
**Last Updated:** 2026-05-06  
**Base URL:** `https://api.cura.com/v1`  
**Auth:** Bearer JWT (Clerk-issued)

---

## 1. Conventions

- All requests and responses use `application/json`
- Dates are ISO 8601: `2026-05-06T12:00:00Z`
- Pagination uses cursor-based: `?cursor=<id>&limit=20`
- Errors follow RFC 7807 Problem Details format
- All list endpoints return `{ data: [], meta: { cursor, total, hasMore } }`

### Standard Error Response

```json
{
  "type": "https://api.cura.com/errors/not-found",
  "title": "Resource Not Found",
  "status": 404,
  "detail": "Product with id 'prod_abc123' does not exist.",
  "instance": "/v1/products/prod_abc123"
}
```

### Role Guards

| Role | Header Requirement |
|---|---|
| Public | None |
| Buyer | `Authorization: Bearer <buyer_token>` |
| Curator | `Authorization: Bearer <curator_token>` |
| Vendor | `Authorization: Bearer <vendor_token>` |
| Admin | `Authorization: Bearer <admin_token>` |

---

## 2. Auth

### POST `/auth/session`
Exchange Clerk session token for API JWT.

**Request:**
```json
{ "clerk_token": "sess_abc..." }
```
**Response:** `200`
```json
{
  "access_token": "eyJ...",
  "expires_at": "2026-05-07T12:00:00Z",
  "user": { "id": "usr_123", "role": "buyer", "email": "user@example.com" }
}
```

---

## 3. Products

### GET `/products`
Browse live products. Public.

**Query Params:**
| Param | Type | Description |
|---|---|---|
| `cursor` | string | Pagination cursor |
| `limit` | int | Max 50, default 20 |
| `category` | string | Slug filter |
| `curator_id` | string | Filter by curator |
| `collection_id` | string | Filter by collection |
| `q` | string | Full-text search (Algolia) |

**Response:** `200`
```json
{
  "data": [
    {
      "id": "prod_abc123",
      "name": "Kinfolk Ceramic Mug",
      "slug": "kinfolk-ceramic-mug",
      "price_cents": 4800,
      "currency": "USD",
      "images": ["https://res.cloudinary.com/cura/..."],
      "curator": { "id": "cur_xyz", "name": "Mara Lund", "avatar": "..." },
      "editorial": "Thrown by hand in Oaxaca. Every imperfection is the point.",
      "category": "kitchen",
      "in_stock": true,
      "collection": { "id": "col_001", "name": "Slow Living Drop 03" }
    }
  ],
  "meta": { "cursor": "prod_def456", "total": 312, "hasMore": true }
}
```

### GET `/products/:id`
Get single product detail. Public.

**Response:** `200` — Full product object including full editorial, curator write-up, variants, specs, reviews summary.

### POST `/products` *(Admin, Curator)*
Create a new product (from approved vendor submission).

**Request:**
```json
{
  "vendor_id": "ven_001",
  "name": "Kinfolk Ceramic Mug",
  "description": "...",
  "price_cents": 4800,
  "sku": "KCM-001",
  "category_id": "cat_kitchen",
  "variants": [{ "label": "Stone Grey", "sku": "KCM-001-GRY", "stock": 40 }],
  "images": ["cloudinary_id_1", "cloudinary_id_2"],
  "editorial": "Thrown by hand in Oaxaca...",
  "why_we_chose": "We were looking for mugs that feel like objects...",
  "collection_id": "col_001"
}
```
**Response:** `201` — Created product object.

### PATCH `/products/:id` *(Admin, Curator)*
Update product fields.

### DELETE `/products/:id` *(Admin)*
Soft-delete (archive) a product.

---

## 4. Collections

### GET `/collections`
List all active collections. Public.

### GET `/collections/:id`
Get collection with all products. Public.

### POST `/collections` *(Admin, Curator)*
Create a collection.

**Request:**
```json
{
  "name": "Slow Living Drop 03",
  "slug": "slow-living-drop-03",
  "type": "drop",
  "drop_at": "2026-06-01T12:00:00Z",
  "description": "...",
  "curator_id": "cur_xyz",
  "product_ids": ["prod_abc", "prod_def"]
}
```

### POST `/collections/:id/waitlist` *(Buyer)*
Join a drop waitlist.

---

## 5. Curators

### GET `/curators`
List all active curators. Public.

**Query Params:** `?category=kitchen&sort=reputation`

### GET `/curators/:id`
Get curator profile with collections and top products. Public.

### POST `/curators/:id/follow` *(Buyer)*
Follow a curator.

### DELETE `/curators/:id/follow` *(Buyer)*
Unfollow a curator.

### GET `/curators/:id/stats` *(Admin, Curator - own)*
Get curator reputation stats.

**Response:** `200`
```json
{
  "followers": 1240,
  "products_curated": 87,
  "avg_product_rating": 4.7,
  "reputation_score": 92.4,
  "total_sales_facilitated": 3420
}
```

---

## 6. Users & Taste Profiles

### GET `/users/me` *(Buyer)*
Get current user profile.

### PATCH `/users/me` *(Buyer)*
Update profile fields.

### GET `/users/me/taste-profile` *(Buyer)*
Get taste profile.

### POST `/users/me/taste-profile` *(Buyer)*
Create/update taste profile from quiz.

**Request:**
```json
{
  "aesthetic": ["minimalist", "wabi-sabi"],
  "values": ["sustainability", "craftsmanship"],
  "lifestyle": ["home", "cooking", "travel"]
}
```

### GET `/users/me/feed` *(Buyer)*
Personalized feed based on followed curators and taste profile.

---

## 7. Shelves

### GET `/users/me/shelves` *(Buyer)*
List own shelves.

### POST `/users/me/shelves` *(Buyer)*
Create a shelf.

```json
{ "name": "Dream Kitchen", "visibility": "public" }
```

### POST `/users/me/shelves/:id/items` *(Buyer)*
Add product to shelf.

```json
{ "product_id": "prod_abc123" }
```

### DELETE `/users/me/shelves/:id/items/:product_id` *(Buyer)*
Remove from shelf.

### GET `/shelves/:id` *(Public if visibility=public)*
View a public shelf.

---

## 8. Orders

### POST `/orders` *(Buyer)*
Create order / initiate checkout.

**Request:**
```json
{
  "items": [{ "product_id": "prod_abc", "variant_id": "var_001", "quantity": 1 }],
  "shipping_address": { "line1": "...", "city": "...", "state": "CA", "zip": "90210", "country": "US" }
}
```
**Response:** `201`
```json
{
  "order_id": "ord_001",
  "stripe_payment_intent_id": "pi_xxx",
  "client_secret": "pi_xxx_secret_xxx",
  "total_cents": 5800,
  "status": "pending_payment"
}
```

### GET `/orders` *(Buyer)*
List own orders.

### GET `/orders/:id` *(Buyer, Admin)*
Get order detail with line items and status.

### PATCH `/orders/:id/status` *(Admin)*
Update order status (e.g., fulfilled, shipped).

---

## 9. Reviews

### POST `/products/:id/reviews` *(Buyer - verified purchase only)*
Submit a review.

```json
{
  "rating": 5,
  "body": "Feels incredible in hand...",
  "media": ["cloudinary_id_1"]
}
```

### GET `/products/:id/reviews`
List reviews for a product. Public.

---

## 10. Vendor Portal

### POST `/vendor/products/submit`
Submit a product for curatorial review.

### GET `/vendor/products`
List submitted products with review status.

### GET `/vendor/orders`
List orders for vendor's products.

### GET `/vendor/payouts`
List payout history.

---

## 11. Admin

### GET `/admin/dashboard`
Platform metrics summary.

### GET `/admin/submissions`
List pending product submissions.

### PATCH `/admin/submissions/:id`
Approve or reject a submission.

```json
{ "status": "rejected", "feedback": "Image quality below standard. Please resubmit with studio photos." }
```

### GET `/admin/curators/applications`
List curator applications.

### PATCH `/admin/curators/applications/:id`
Approve or reject a curator application.

---

## 12. Webhooks (Stripe → Cura)

| Event | Handler |
|---|---|
| `payment_intent.succeeded` | Confirm order, trigger fulfillment |
| `payment_intent.payment_failed` | Mark order failed, notify buyer |
| `charge.refunded` | Process refund, update inventory |
| `account.updated` (Connect) | Sync vendor payout account |
