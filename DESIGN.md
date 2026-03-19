# Backend Service & Data Model Design
## Bakery Ordering System — Capacity-Constrained Product Management

**Status:** Draft
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

#### `capacity_settings`
Defines how many of a product can be produced per time window.
This is the core constraint that differentiates this system from standard e-commerce.

| Column | Type | Notes |
|---|---|---|
| `id` | UUID PK | |
| `product_id` | UUID FK → `products` | |
| `window_type` | ENUM(`daily`, `weekly`, `slot`) | Granularity of the cap |
| `max_quantity` | INTEGER | Max units producible in the window |
| `lead_time_hours` | INTEGER | Minimum hours before pickup/delivery |
| `active_from` | DATE | When this setting takes effect |
| `active_until` | DATE NULLABLE | NULL = indefinite |

#### `availability_windows`
Concrete time slots when orders can be picked up or delivered.
Generated from capacity settings, or manually managed by staff.

| Column | Type | Notes |
|---|---|---|
| `id` | UUID PK | |
| `product_id` | UUID FK → `products` | |
| `window_start` | TIMESTAMPTZ | |
| `window_end` | TIMESTAMPTZ | |
| `max_quantity` | INTEGER | Copied from capacity setting at generation time |
| `reserved_quantity` | INTEGER | Incremented as orders are placed |

#### `orders`
A customer's purchase of one or more products.

| Column | Type | Notes |
|---|---|---|
| `id` | UUID PK | |
| `customer_id` | UUID FK → `users` | |
| `status` | ENUM(`pending`, `confirmed`, `ready`, `completed`, `cancelled`) | |
| `total_cents` | INTEGER | |
| `notes` | TEXT NULLABLE | Special instructions |
| `pickup_window_id` | UUID FK → `availability_windows` NULLABLE | |
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
categories ──< products >──< order_items >── orders ──< users
                  │
          capacity_settings
                  │
          availability_windows
```

- `categories` → `products`: one-to-many
- `products` → `capacity_settings`: one-to-many (settings can change over time)
- `products` → `availability_windows`: one-to-many
- `orders` → `order_items`: one-to-many
- `order_items` → `products`: many-to-one
- `orders` → `availability_windows`: many-to-one (multiple orders can share a window up to capacity)

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

#### Capacity Settings
```
GET    /products/{id}/capacity    # Get capacity settings for a product [staff+]
POST   /products/{id}/capacity    # Add/update capacity setting [staff+]
```

#### Availability Windows
```
GET    /availability              # Query available slots (by date range, product)
POST   /availability/generate     # Generate windows from capacity settings [staff+]
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

## 5. Capacity Enforcement Logic

This is the most critical business rule in the system.

When a customer adds items to an order:

1. Look up the `availability_window` for the requested pickup time and product.
2. Check: `max_quantity - reserved_quantity >= requested_quantity`
3. If capacity is available, **atomically increment** `reserved_quantity` and create the order
   (use a `SELECT ... FOR UPDATE` row lock or a database transaction with optimistic locking).
4. If capacity is exceeded, return `409 Conflict` with a clear message.

On order cancellation: decrement `reserved_quantity` accordingly.

**Lead time validation:** Reject orders where `window_start < now() + lead_time_hours`.

---

## 6. Open Questions / Decisions Needed

| # | Question | Impact |
|---|---|---|
| 1 | Are orders paid online or in-person? | Determines if a payment provider (Stripe) is needed |
| 2 | How are availability windows generated? Automated nightly job or staff-triggered? | Affects background task design |
| 3 | Will there be a concept of "menu" that changes weekly/seasonally? | May require date-scoped product visibility |
| 4 | Should customers receive email/SMS notifications on order status changes? | Requires async notification service |
| 5 | Do we need multi-location support (multiple bakery locations)? | Adds `location` entity and complicates capacity model |
| 6 | What is the deployment target (AWS, GCP, fly.io, etc.)? | Affects infra choices and CI/CD |
| 7 | Do staff need a separate admin UI or will they use the same frontend? | May affect API design for staff endpoints |

---

## 7. Project Structure (FastAPI)

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
│   └── capacity.py
├── schemas/               # Pydantic request/response schemas
│   ├── product.py
│   ├── order.py
│   └── user.py
├── repositories/          # All DB queries live here
│   ├── product_repo.py
│   ├── order_repo.py
│   └── capacity_repo.py
├── services/              # Business logic
│   ├── product_service.py
│   ├── order_service.py
│   └── capacity_service.py
├── routers/               # FastAPI route definitions
│   ├── products.py
│   ├── orders.py
│   ├── categories.py
│   ├── availability.py
│   └── auth.py
└── migrations/            # Alembic migrations
```

---

## 8. Next Steps

- [ ] Answer open questions (Section 6)
- [ ] Finalize entity list and column details
- [ ] Prototype the capacity enforcement service method
- [ ] Set up FastAPI + SQLAlchemy + Alembic scaffold
- [ ] Define authentication strategy (JWT library, token expiry)
- [ ] Create OpenAPI schema and share with frontend teams
