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
| `is_active` | BOOLEAN | Controls visibility to customers |
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
GET    /products                  # List active products (with availability)
GET    /products/{id}             # Product detail
POST   /products                  # Create product [staff+]
PUT    /products/{id}             # Update product [staff+]
DELETE /products/{id}             # Soft-deactivate [admin]
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

## 7. Open Questions / Decisions Needed

| # | Question | Status | Impact |
|---|---|---|---|
| 1 | Are orders paid online or in-person? | **In-person** (online = future epic) | No payment provider needed now; `total_cents` is reference-only |
| 2 | How are availability windows generated? | **Staff-triggered** via API; weekly template as the source of truth | No background job needed initially |
| 3 | Single or multi-location? | **Single location** | No `location` entity needed |
| 4 | Will there be a concept of "menu" that changes weekly/seasonally? | Open | May require date-scoped `is_active` on products |
| 5 | Should customers receive email/SMS notifications on order status changes? | Open | Requires async notification service (e.g. SendGrid, Twilio) |
| 6 | What is the deployment target (AWS, GCP, fly.io, etc.)? | Open | Affects infra choices and CI/CD |
| 7 | Do staff need a separate admin UI or will they use the same frontend? | Open | May affect API design for staff endpoints |

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

## 9. Next Steps

- [ ] Resolve remaining open questions (Section 7, items 4–7)
- [ ] Prototype the `schedule_service` — window generation + capacity enforcement transaction
- [ ] Set up FastAPI + SQLAlchemy + Alembic scaffold
- [ ] Define authentication strategy (JWT library, token expiry)
- [ ] Create OpenAPI schema and share with web + iOS frontend teams
- [ ] Future epic: online payment (Stripe) integration
