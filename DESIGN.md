# Backend Service & Data Model Design
## Bakery Ordering System — Capacity-Constrained Product Management

**Status:** Draft — v2
**Date:** 2026-03-19
**Stack:** Python (FastAPI), PostgreSQL, Service + Repository pattern

---

## 1. Overview

A backend REST API for a bakery that sells products online (web + iOS). Unlike a
standard e-commerce system, product availability is **capacity-constrained** —
only a finite number of items can be produced per time window based on available
oven time, counter space, and staffing. Orders and listings must respect these
capacity limits.

### Goals

- Manage a catalog of bakery products with rich metadata (price, description, images)
- Define **capacity settings** that cap how many of each product can be offered per time window
- Allow customers (web + mobile) to browse available inventory and place orders
- Expose internal APIs for backend service integrations (inventory sync, reporting)
- Enforce authentication and role-based access (customer vs. staff vs. admin)

---

## 2. Architecture

```
 iOS App  ──┐
            │  HTTPS
 Web App  ──┼──────────▶  FastAPI Service  ──────▶  PostgreSQL
            │                   │
 Services ──┘                   └──────────────────▶  (future) Redis cache
```

### Layer Breakdown

| Layer | Responsibility |
|---|---|
| **Router (Controller)** | HTTP routing, request validation (Pydantic), response serialization |
| **Service** | Business logic, capacity enforcement, transaction orchestration |
| **Repository** | All database access; no SQL in service/router layers |
| **Models** | SQLAlchemy ORM models mapping to PostgreSQL tables |
| **Schemas** | Pydantic models for request/response contracts |

---

## 3. Data Model

### 3.1 Core Entities

#### `products`
Represents a bakery item available for sale.

| Column | Type | Notes |
|---|---|---|
| `id` | UUID PK | |
| `name` | VARCHAR(255) | e.g. "Sourdough Loaf" |
| `description` | TEXT | |
| `price_cents` | INTEGER | Stored in cents to avoid float issues |
| `category_id` | UUID FK → `categories` | |
| `is_active` | BOOLEAN | Permanently removes product from menu when false |
| `is_on_hold` | BOOLEAN | Temporarily hides from customers without deleting; staff can resume |
| `image_url` | TEXT | |
| `created_at` | TIMESTAMPTZ | |
| `updated_at` | TIMESTAMPTZ | |

#### `categories`
Groups products (e.g. Breads, Pastries, Cakes).

| Column | Type | Notes |
|---|---|---|
| `id` | UUID PK | |
| `name` | VARCHAR(100) | |
| `sort_order` | INTEGER | Display ordering |

#### `schedule_templates`
Weekly repeating schedule that defines the bakery's default production windows.
Staff configure this once; daily windows are generated from it automatically.

| Column | Type | Notes |
|---|---|---|
| `id` | UUID PK | |
| `name` | VARCHAR(100) | e.g. "Standard Week", "Holiday Schedule" |
| `is_active` | BOOLEAN | Only one template active at a time |
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
| `lead_time_hours` | INTEGER | Minimum hours before this slot opens for ordering |

#### `product_slot_capacity`
How many of each product can be produced for a given template slot.
Decouples product capacity from the time slot definition.

| Column | Type | Notes |
|---|---|---|
| `id` | UUID PK | |
| `template_slot_id` | UUID FK → `schedule_template_slots` | |
| `product_id` | UUID FK → `products` | |
| `max_quantity` | INTEGER | |

#### `availability_windows`
Concrete per-day instances generated from the active template.
Staff can override `max_quantity` or mark a window `is_blocked` for holidays/closures.

| Column | Type | Notes |
|---|---|---|
| `id` | UUID PK | |
| `template_slot_id` | UUID FK → `schedule_template_slots` NULLABLE | NULL if manually created |
| `date` | DATE | The specific calendar date |
| `pickup_start` | TIMESTAMPTZ | |
| `pickup_end` | TIMESTAMPTZ | |
| `is_blocked` | BOOLEAN | Staff can close a window (holiday, sold out early) |
| `created_at` | TIMESTAMPTZ | |

#### `window_product_capacity`
Per-product capacity for a specific availability window.
Copied from `product_slot_capacity` at generation time; staff can override per day.

| Column | Type | Notes |
|---|---|---|
| `id` | UUID PK | |
| `window_id` | UUID FK → `availability_windows` | |
| `product_id` | UUID FK → `products` | |
| `max_quantity` | INTEGER | Staff-overridable |
| `reserved_quantity` | INTEGER | Atomically incremented on order placement |

#### `orders`
A customer's purchase of one or more products. Payment is collected in person.

| Column | Type | Notes |
|---|---|---|
| `id` | UUID PK | |
| `customer_id` | UUID FK → `users` | |
| `status` | ENUM(`pending`, `confirmed`, `ready`, `completed`, `cancelled`) | |
| `total_cents` | INTEGER | Calculated at order time; reference only until in-person payment |
| `notes` | TEXT NULLABLE | Special instructions |
| `pickup_window_id` | UUID FK → `availability_windows` | The chosen pickup window |
| `created_at` | TIMESTAMPTZ | |
| `updated_at` | TIMESTAMPTZ | |

#### `order_items`
Line items within an order (many-to-many between orders and products).

| Column | Type | Notes |
|---|---|---|
| `id` | UUID PK | |
| `order_id` | UUID FK → `orders` | |
| `product_id` | UUID FK → `products` | |
| `quantity` | INTEGER | |
| `unit_price_cents` | INTEGER | Price locked at time of order |

#### `users`
Customers and staff accounts.

| Column | Type | Notes |
|---|---|---|
| `id` | UUID PK | |
| `email` | VARCHAR(255) UNIQUE | |
| `hashed_password` | TEXT | |
| `role` | ENUM(`customer`, `staff`, `admin`) | |
| `full_name` | VARCHAR(255) | |
| `phone` | VARCHAR(50) NULLABLE | |
| `is_active` | BOOLEAN | |
| `created_at` | TIMESTAMPTZ | |

### 3.2 Entity Relationship Summary

```
schedule_templates ──< schedule_template_slots >──< product_slot_capacity
                                │                            │
                                ▼                            │ (copied at generation)
                       availability_windows                  │
                                │                            ▼
                                └──────────< window_product_capacity >── products
                                                                              │
categories ──────────────────────────────────────────────────────────────────┤
                                                                              │
orders ──< order_items >──────────────────────────────────────────────────────┘
  │
users
```

- `schedule_templates` → `schedule_template_slots`: one-to-many
- `schedule_template_slots` → `product_slot_capacity`: one-to-many (one per product)
- `schedule_template_slots` → `availability_windows`: one-to-many (one per calendar date)
- `availability_windows` → `window_product_capacity`: one-to-many
- `window_product_capacity` → `products`: many-to-one
- `orders` → `availability_windows`: many-to-one
- `orders` → `order_items`: one-to-many
- `order_items` → `products`: many-to-one

---

## 4. API Design

### Authentication
- JWT Bearer tokens (issued on login)
- Role-based access: `customer` / `staff` / `admin`

### Endpoint Groups

#### Products (public + staff)
```
GET    /products                  # List active, non-held products (customers)
GET    /products?include_held=true # Include on-hold products [staff+]
GET    /products/{id}             # Product detail
POST   /products                  # Create product [staff+]
PUT    /products/{id}             # Update product [staff+]
PATCH  /products/{id}/hold        # Toggle is_on_hold [staff+]
DELETE /products/{id}             # Soft-deactivate (is_active=false) [admin]
```

#### Categories
```
GET    /categories                # List all
POST   /categories                # Create [staff+]
```

#### Schedule Templates [staff+]
```
GET    /schedules                           # List templates
POST   /schedules                           # Create a new template
PUT    /schedules/{id}                      # Update template
POST   /schedules/{id}/activate             # Set as active template
GET    /schedules/{id}/slots                # List slots in a template
POST   /schedules/{id}/slots                # Add a slot to template
PUT    /schedules/{id}/slots/{slot_id}      # Update slot (time, lead time)
POST   /schedules/{id}/slots/{slot_id}/capacity   # Set product capacity for slot
```

#### Availability Windows
```
GET    /availability                        # Query windows by date range (public)
POST   /availability/generate               # Generate windows from active template [staff+]
PATCH  /availability/{id}                   # Override a specific day (block, adjust qty) [staff+]
GET    /availability/{id}/capacity          # Per-product capacity for a window
PATCH  /availability/{id}/capacity/{product_id}  # Override product qty for one day [staff+]
```

#### Orders
```
GET    /orders                    # Customer: own orders. Staff: all orders
POST   /orders                    # Place an order
GET    /orders/{id}               # Order detail
PATCH  /orders/{id}/status        # Update status [staff+]
DELETE /orders/{id}               # Cancel order
```

#### Auth
```
POST   /auth/register
POST   /auth/login
POST   /auth/refresh
```

---

## 5. Availability Window Generation

Staff trigger window generation weekly (or ahead of a holiday schedule change).
The service materializes `availability_windows` + `window_product_capacity` rows
from the currently active `schedule_template`.

```
POST /availability/generate  { "from_date": "2026-03-24", "to_date": "2026-03-30" }
```

Logic:
1. Fetch the active `schedule_template` and its slots.
2. For each date in range, find the matching `day_of_week` slots.
3. Skip dates that already have a window for that slot (idempotent).
4. Insert `availability_windows` + copy `product_slot_capacity` → `window_product_capacity`.

Staff can then override any individual day via `PATCH /availability/{id}`.

---

## 6. Capacity Enforcement Logic

This is the most critical business rule in the system.

When a customer places an order:

1. Look up `window_product_capacity` rows for the chosen `pickup_window_id` + each product.
2. Check: `max_quantity - reserved_quantity >= requested_quantity` for each item.
3. Validate lead time: reject if `window.pickup_start < now() + lead_time_hours`.
4. If all checks pass, within a single DB transaction:
   - `UPDATE window_product_capacity SET reserved_quantity = reserved_quantity + ? WHERE id = ? AND (max_quantity - reserved_quantity) >= ?`
   - Insert `order` + `order_items` rows.
5. If the UPDATE affects 0 rows (race condition), return `409 Conflict`.

On order cancellation: decrement `reserved_quantity` in the same transaction as the status update.

---

## 7. Decisions Log

All questions resolved.

| # | Question | Decision |
|---|---|---|
| 1 | Payment | In-person; online = future epic |
| 2 | Availability window generation | Staff-triggered via API from weekly template |
| 3 | Locations | Single location |
| 4 | Menu changes | Staff can add products or toggle `is_on_hold`; no date-scoped visibility needed |
| 5 | Notifications | Email confirmation only — sent when order is placed, includes pickup instructions |
| 6 | Deployment | TBD; prefer low-ops, small-business-friendly (see Section 9) |
| 7 | Admin UI | Single frontend, role-based routing (customer vs. staff/admin views) |

---

## 8. Project Structure (FastAPI)

```
app/
├── main.py
├── core/
│   ├── config.py          # Settings (env vars)
│   ├── security.py        # JWT, password hashing
│   └── database.py        # SQLAlchemy engine & session
├── models/                # SQLAlchemy ORM models
│   ├── product.py
│   ├── order.py
│   ├── user.py
│   └── schedule.py        # schedule_templates, slots, availability_windows, capacity
├── schemas/               # Pydantic request/response schemas
│   ├── product.py
│   ├── order.py
│   ├── schedule.py
│   └── user.py
├── repositories/          # All DB queries live here
│   ├── product_repo.py
│   ├── order_repo.py
│   └── schedule_repo.py
├── services/              # Business logic
│   ├── product_service.py
│   ├── order_service.py
│   └── schedule_service.py  # window generation + capacity enforcement
├── routers/               # FastAPI route definitions
│   ├── products.py
│   ├── orders.py
│   ├── categories.py
│   ├── schedules.py
│   ├── availability.py
│   └── auth.py
└── migrations/            # Alembic migrations
```

---

## 9. Email Notifications

A single transactional email is sent when a customer's order is confirmed.

**Trigger:** `POST /orders` success → enqueue email task

**Contents:**
- Order summary (items, quantities, total)
- Pickup window (date + time range)
- What to expect next (pay at pickup, bring order confirmation number)
- Contact info for changes/cancellations

**Implementation (keep it simple):**
- Use [SendGrid](https://sendgrid.com) or [Resend](https://resend.com) — both have generous free tiers
- Send synchronously on order creation for now (no queue needed at this scale)
- If the email fails, log the error but don't fail the order — order is the source of truth

Future: status-change emails (e.g. "Your order is ready for pickup") can be added later.

---

## 10. Deployment Recommendation

For a small business, **[Railway](https://railway.app)** or **[Render](https://render.com)** are the best fit:

| Option | Why it's good for this project |
|---|---|
| **Railway** | One-click PostgreSQL + FastAPI deploy, auto-deploys from GitHub, simple pricing (~$5–20/mo) |
| **Render** | Similar to Railway, free tier available, managed Postgres, easy SSL |
| **fly.io** | Slightly more control, very cheap, good for containerized FastAPI |

**Recommendation: Railway**
- Connect GitHub repo → it detects FastAPI and builds automatically
- Add a Postgres plugin with one click
- Environment variables managed in their dashboard
- No DevOps knowledge required

**What you'll need:**
- `Dockerfile` or `railway.toml` config (straightforward for FastAPI)
- Alembic migrations run on deploy
- SendGrid/Resend API key as an environment variable

---

## 11. Next Steps

- [x] Resolve all open questions
- [ ] Scaffold FastAPI project (structure, SQLAlchemy, Alembic, JWT auth)
- [ ] Implement `schedule_service` — window generation + capacity enforcement transaction
- [ ] Set up Railway deployment + PostgreSQL
- [ ] Integrate transactional email (Resend or SendGrid) on order confirmation
- [ ] Create OpenAPI schema and share with web + iOS frontend teams
- [ ] Future epic: online payment (Stripe) integration
