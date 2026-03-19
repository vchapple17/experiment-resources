# Backend Service & Data Model Design
## Multi-Tenant SaaS — Capacity-Constrained Ordering Platform

**Status:** Draft — v3
**Date:** 2026-03-19
**Stack:** Python (FastAPI), PostgreSQL, Service + Repository pattern

---

## 1. Overview

A multi-tenant SaaS backend where each **company** (tenant) runs its own ordering
storefront. Companies sell products in two modes:

- **Ready** — items always available; no production constraint (e.g. packaged goods,
  pre-made items stocked on a shelf)
- **Made-to-order** — capacity-constrained; quantity per time window is limited by
  production capacity (e.g. fresh-baked bread, custom cakes)

An order can mix both product types. Ready items are added freely; made-to-order
items require the customer to select an available pickup window.

### Goals

- Isolate each company's data, staff, products, schedules, and orders completely
- Support a single user belonging to multiple companies (e.g. an owner with two shops)
- Provide a platform-level admin role for the SaaS operator
- Handle both ready and made-to-order products in the same order flow
- Email confirmation on order placement

---

## 2. Multi-Tenancy Design

### Strategy: Row-Level Tenancy + Scoped JWT

All tenant data lives in a single shared database. Every tenant-owned table has a
`company_id` column. The authenticated user's JWT carries a `company_id` claim
indicating which company context is active for that session.

**Why not schema-per-tenant?** At this scale, row-level is simpler to operate and
query. Migrating to schema-per-tenant later is possible but not needed now.

### Company Context Flow

```
1. POST /auth/login
   → Returns: user info + list of companies user belongs to

2a. Single company → auto-select, return scoped JWT
2b. Multiple companies → client calls POST /auth/select-company/{company_id}
    → Returns: scoped JWT with { user_id, company_id, role } claims

3. All subsequent API calls use the scoped JWT
   → Middleware extracts company_id and injects into request context
   → Repository layer always filters by company_id
```

### Roles

| Role | Scope | Can do |
|---|---|---|
| `platform_admin` | Global | Manage all companies, users, billing; support access |
| `admin` | Company | Full company control — products, staff, schedules, orders |
| `staff` | Company | Manage products, schedules, view/update orders |
| `customer` | Company | Browse products, place and view own orders |

A user's role is per-company (stored in `company_memberships`). The same person can be
`admin` at one company and `customer` at another.

---

## 3. Data Model

### 3.1 Core Entities

#### `companies`
Each tenant on the platform.

| Column | Type | Notes |
|---|---|---|
| `id` | UUID PK | |
| `name` | VARCHAR(255) | e.g. "The Sourdough Co." |
| `slug` | VARCHAR(100) UNIQUE | URL-safe identifier, e.g. `sourdough-co` |
| `is_active` | BOOLEAN | Platform admin can suspend a company |
| `created_at` | TIMESTAMPTZ | |

#### `users`
Platform-level accounts. Role and company association live in `company_memberships`.

| Column | Type | Notes |
|---|---|---|
| `id` | UUID PK | |
| `email` | VARCHAR(255) UNIQUE | |
| `hashed_password` | TEXT | |
| `full_name` | VARCHAR(255) | |
| `phone` | VARCHAR(50) NULLABLE | |
| `is_platform_admin` | BOOLEAN | SaaS operator superuser |
| `is_active` | BOOLEAN | |
| `created_at` | TIMESTAMPTZ | |

#### `company_memberships`
Links users to companies with a role. A user can have one membership per company.

| Column | Type | Notes |
|---|---|---|
| `id` | UUID PK | |
| `user_id` | UUID FK → `users` | |
| `company_id` | UUID FK → `companies` | |
| `role` | ENUM(`admin`, `staff`, `customer`) | Role within this company |
| `created_at` | TIMESTAMPTZ | |
| — | UNIQUE(`user_id`, `company_id`) | One membership per company per user |

#### `products`
Tenant-scoped. Two types: ready-to-sell or made-to-order.

| Column | Type | Notes |
|---|---|---|
| `id` | UUID PK | |
| `company_id` | UUID FK → `companies` | Tenant scope |
| `category_id` | UUID FK → `categories` NULLABLE | |
| `name` | VARCHAR(255) | |
| `description` | TEXT | |
| `price_cents` | INTEGER | Stored in cents |
| `product_type` | ENUM(`ready`, `made_to_order`) | Controls ordering flow |
| `is_active` | BOOLEAN | Permanently removed when false |
| `is_on_hold` | BOOLEAN | Temporarily hidden; staff can resume |
| `image_url` | TEXT NULLABLE | |
| `created_at` | TIMESTAMPTZ | |
| `updated_at` | TIMESTAMPTZ | |

#### `categories`
Tenant-scoped groupings (e.g. Breads, Pastries, Beverages).

| Column | Type | Notes |
|---|---|---|
| `id` | UUID PK | |
| `company_id` | UUID FK → `companies` | |
| `name` | VARCHAR(100) | |
| `sort_order` | INTEGER | |

#### `schedule_templates`
Tenant-scoped weekly repeating production schedule. Applies only to `made_to_order`
products.

| Column | Type | Notes |
|---|---|---|
| `id` | UUID PK | |
| `company_id` | UUID FK → `companies` | |
| `name` | VARCHAR(100) | e.g. "Standard Week", "Holiday Schedule" |
| `is_active` | BOOLEAN | Only one active per company at a time |
| `created_at` | TIMESTAMPTZ | |

#### `schedule_template_slots`
One row per day-of-week + time block within a template.

| Column | Type | Notes |
|---|---|---|
| `id` | UUID PK | |
| `template_id` | UUID FK → `schedule_templates` | |
| `day_of_week` | SMALLINT | 0=Mon … 6=Sun |
| `pickup_start` | TIME | e.g. `08:00` |
| `pickup_end` | TIME | e.g. `12:00` |
| `lead_time_hours` | INTEGER | Min hours ahead a customer must order |

#### `product_slot_capacity`
How many of each made-to-order product can be produced per template slot.

| Column | Type | Notes |
|---|---|---|
| `id` | UUID PK | |
| `template_slot_id` | UUID FK → `schedule_template_slots` | |
| `product_id` | UUID FK → `products` | Must be `made_to_order` type |
| `max_quantity` | INTEGER | |

#### `availability_windows`
Concrete calendar-date instances generated from the active template.
Staff can block a window (holiday) or adjust per day.

| Column | Type | Notes |
|---|---|---|
| `id` | UUID PK | |
| `company_id` | UUID FK → `companies` | |
| `template_slot_id` | UUID FK → `schedule_template_slots` NULLABLE | NULL if manually created |
| `date` | DATE | |
| `pickup_start` | TIMESTAMPTZ | |
| `pickup_end` | TIMESTAMPTZ | |
| `is_blocked` | BOOLEAN | Staff can close (holiday, early sellout) |
| `created_at` | TIMESTAMPTZ | |

#### `window_product_capacity`
Per-product, per-window capacity. Copied from `product_slot_capacity` at generation;
staff can override per day.

| Column | Type | Notes |
|---|---|---|
| `id` | UUID PK | |
| `window_id` | UUID FK → `availability_windows` | |
| `product_id` | UUID FK → `products` | |
| `max_quantity` | INTEGER | Staff-overridable |
| `reserved_quantity` | INTEGER | Atomically incremented on order placement |

#### `orders`
A customer's purchase. May contain both ready and made-to-order items.
Payment collected in person.

| Column | Type | Notes |
|---|---|---|
| `id` | UUID PK | |
| `company_id` | UUID FK → `companies` | |
| `customer_id` | UUID FK → `users` | |
| `status` | ENUM(`pending`, `confirmed`, `ready`, `completed`, `cancelled`) | |
| `total_cents` | INTEGER | Reference only until in-person payment |
| `notes` | TEXT NULLABLE | Customer special instructions |
| `pickup_window_id` | UUID FK → `availability_windows` NULLABLE | Required when order has made-to-order items |
| `created_at` | TIMESTAMPTZ | |
| `updated_at` | TIMESTAMPTZ | |

#### `order_items`

| Column | Type | Notes |
|---|---|---|
| `id` | UUID PK | |
| `order_id` | UUID FK → `orders` | |
| `product_id` | UUID FK → `products` | |
| `quantity` | INTEGER | |
| `unit_price_cents` | INTEGER | Price locked at order time |

---

### 3.2 Entity Relationship Summary

```
companies ──< company_memberships >── users
    │
    ├──< categories
    │
    ├──< products (type: ready | made_to_order)
    │         │
    ├──< schedule_templates
    │         └──< schedule_template_slots >──< product_slot_capacity >── products
    │                        │
    │                        ▼ (generated)
    └──< availability_windows >──< window_product_capacity >── products
                   │
              orders (pickup_window_id nullable)
                │  └──< order_items >── products
               users
```

---

## 4. Order Flow by Product Type

### Ready products only
1. Customer adds ready items to cart, submits order
2. No pickup window required
3. Order created immediately, confirmation email sent
4. Staff marks order ready/completed at counter

### Made-to-order products (or mixed cart)
1. Customer adds items (ready + made-to-order)
2. Frontend shows available `availability_windows` for the made-to-order items
3. Customer selects a window; order submitted with `pickup_window_id`
4. Capacity check + atomic reservation (see Section 6)
5. Confirmation email with pickup window details sent

**Validation rule:** `pickup_window_id` is required if any `order_item.product.product_type == 'made_to_order'`.

---

## 5. API Design

### Tenant Scoping
All endpoints (except auth and platform admin) operate within the company context
extracted from the JWT's `company_id` claim. No `company_id` parameter is needed
in request bodies — it is injected server-side.

### Auth
```
POST   /auth/register                    # Create account (no company yet)
POST   /auth/login                       # Returns user + company list
POST   /auth/select-company/{company_id} # Exchange for scoped JWT
POST   /auth/refresh
```

### Companies (platform admin only)
```
GET    /platform/companies               # List all tenants
POST   /platform/companies               # Create new company
GET    /platform/companies/{id}
PATCH  /platform/companies/{id}          # Suspend, rename
GET    /platform/users                   # All platform users
```

### Company Onboarding / Members [admin]
```
GET    /company/profile                  # Current company details
PATCH  /company/profile                  # Update name, slug
GET    /company/members                  # List staff + customers
POST   /company/members                  # Invite user by email
PATCH  /company/members/{user_id}        # Change role
DELETE /company/members/{user_id}        # Remove from company
```

### Products
```
GET    /products                          # Active, non-held (customers)
GET    /products?include_held=true        # Include on-hold [staff+]
GET    /products?type=ready               # Filter by product_type
GET    /products?type=made_to_order
GET    /products/{id}
POST   /products                          # Create [staff+]
PUT    /products/{id}                     # Update [staff+]
PATCH  /products/{id}/hold                # Toggle is_on_hold [staff+]
DELETE /products/{id}                     # Soft-deactivate [admin]
```

### Categories
```
GET    /categories
POST   /categories                        # [staff+]
PUT    /categories/{id}                   # [staff+]
```

### Schedule Templates [staff+]
```
GET    /schedules
POST   /schedules
PUT    /schedules/{id}
POST   /schedules/{id}/activate
GET    /schedules/{id}/slots
POST   /schedules/{id}/slots
PUT    /schedules/{id}/slots/{slot_id}
POST   /schedules/{id}/slots/{slot_id}/capacity
```

### Availability Windows
```
GET    /availability                            # By date range (public)
POST   /availability/generate                  # Generate from template [staff+]
PATCH  /availability/{id}                      # Block/unblock day [staff+]
GET    /availability/{id}/capacity
PATCH  /availability/{id}/capacity/{product_id} # Override qty [staff+]
```

### Orders
```
GET    /orders                            # Customer: own. Staff: all company orders
POST   /orders                            # Place order
GET    /orders/{id}
PATCH  /orders/{id}/status               # [staff+]
DELETE /orders/{id}                      # Cancel
```

---

## 6. Availability Window Generation

Staff trigger generation for a date range. The service materializes
`availability_windows` + `window_product_capacity` rows from the company's
active `schedule_template`.

```
POST /availability/generate
{ "from_date": "2026-03-24", "to_date": "2026-03-30" }
```

Logic:
1. Fetch the company's active `schedule_template` and its slots.
2. For each date in range, match by `day_of_week`.
3. Skip dates already having a window for that slot (idempotent).
4. Insert `availability_windows` + copy `product_slot_capacity` → `window_product_capacity`.

---

## 7. Capacity Enforcement Logic

Applies only to `made_to_order` items when an order is placed.

1. Look up `window_product_capacity` for each made-to-order item in the order.
2. Validate lead time: reject if `window.pickup_start < now() + lead_time_hours`.
3. Within a single DB transaction:
   - For each made-to-order item:
     ```sql
     UPDATE window_product_capacity
     SET reserved_quantity = reserved_quantity + :qty
     WHERE id = :id
       AND company_id = :company_id
       AND (max_quantity - reserved_quantity) >= :qty
     ```
   - If any UPDATE affects 0 rows → rollback → return `409 Conflict`.
   - Insert `order` + `order_items` rows.

On cancellation: decrement `reserved_quantity` in the same transaction as the status update.

Ready items have no capacity check — they are added to the order freely.

---

## 8. Decisions Log

| # | Question | Decision |
|---|---|---|
| 1 | Payment | In-person; online = future epic |
| 2 | Availability window generation | Staff-triggered via API from weekly template |
| 3 | Locations | Single location per company |
| 4 | Menu changes | `is_on_hold` for temporary holds; `is_active=false` for permanent removal |
| 5 | Notifications | Order confirmation email only (pickup instructions included) |
| 6 | Deployment | Railway recommended (see Section 11) |
| 7 | Admin UI | Single frontend, role-based routing |
| 8 | Multi-tenancy | Row-level; `company_id` on all tenant tables |
| 9 | User–company relationship | Many-to-many via `company_memberships`; role is per-company |
| 10 | Tenant identification | Scoped JWT with `company_id` + `role` claims |
| 11 | Platform admin | `is_platform_admin` flag on `users`; separate `/platform/*` endpoints |
| 12 | Product types | `ready` (no capacity) and `made_to_order` (capacity-constrained) |

---

## 9. Project Structure (FastAPI)

```
app/
├── main.py
├── core/
│   ├── config.py           # Settings (env vars)
│   ├── security.py         # JWT, password hashing, company context
│   └── database.py         # SQLAlchemy engine & session
├── models/
│   ├── company.py          # companies, company_memberships
│   ├── user.py
│   ├── product.py          # products, categories
│   ├── order.py            # orders, order_items
│   └── schedule.py         # templates, slots, windows, capacity
├── schemas/
│   ├── company.py
│   ├── user.py
│   ├── product.py
│   ├── order.py
│   └── schedule.py
├── repositories/
│   ├── company_repo.py
│   ├── user_repo.py
│   ├── product_repo.py
│   ├── order_repo.py
│   └── schedule_repo.py
├── services/
│   ├── auth_service.py       # Login, company selection, JWT issuance
│   ├── company_service.py
│   ├── product_service.py
│   ├── order_service.py      # Handles mixed ready + made-to-order carts
│   └── schedule_service.py   # Window generation + capacity enforcement
├── routers/
│   ├── auth.py
│   ├── platform.py           # Platform admin routes
│   ├── company.py            # Company profile + member management
│   ├── products.py
│   ├── categories.py
│   ├── schedules.py
│   ├── availability.py
│   └── orders.py
└── migrations/               # Alembic
```

---

## 10. Email Notifications

A single transactional email on order confirmation.

**Contents:**
- Company name + branding (company `name` in subject line)
- Order summary (items, quantities, total)
- Pickup window if made-to-order items are included
- Next steps (pay at pickup, bring confirmation number)
- Company contact info for changes/cancellations

**Implementation:**
- [Resend](https://resend.com) or [SendGrid](https://sendgrid.com) — both have free tiers
- Send synchronously on order creation; log failure but don't fail the order
- Each company can eventually have its own reply-to address (future)

---

## 11. Deployment Recommendation

**[Railway](https://railway.app)** — best fit for a small-business SaaS:

| | |
|---|---|
| FastAPI | Auto-detected from repo, builds with no config |
| PostgreSQL | One-click plugin, connection string injected automatically |
| Deploys | Auto-deploy on push to main |
| Env vars | Managed in Railway dashboard |
| Cost | ~$5–20/month for this scale |

**What you'll need:**
- `Dockerfile` or `railway.toml` (minimal for FastAPI)
- Alembic migration step on deploy
- Resend/SendGrid API key as env var

---

## 12. Next Steps

- [x] Resolve all design questions
- [ ] Scaffold FastAPI project (SQLAlchemy models, Alembic, JWT auth with company context)
- [ ] Implement `auth_service` — login → company list → scoped JWT
- [ ] Implement `schedule_service` — window generation + mixed-cart capacity enforcement
- [ ] Set up Railway + PostgreSQL
- [ ] Integrate Resend for order confirmation email
- [ ] Build OpenAPI schema, share with web + iOS teams
- [ ] Future: online payment (Stripe), per-company email branding, multi-location
