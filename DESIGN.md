# Backend Service & Data Model Design
## Multi-Tenant SaaS — Capacity-Constrained Ordering Platform

**Status:** Draft — v4
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
- Support a single user belonging to multiple companies with per-company roles and a company switcher
- Support multiple **locations** per company, each inheriting company-level templates with the ability to override
- Share products (menu) across all locations within a company; locations can hide products they don't carry
- Provide a platform-level admin role for the SaaS operator; platform admins can promote other platform admins
- Handle both ready and made-to-order products in the same order flow
- Per-company and per-location kill switch to stop accepting orders instantly
- Email notifications across the full order lifecycle

---

## 2. Multi-Tenancy Design

### Strategy: Row-Level Tenancy + Scoped JWT

All tenant data lives in a single shared database. Every tenant-owned table has a
`company_id` column. The authenticated user's JWT carries a `company_id` claim
indicating which company context is active for that session.

**Why not schema-per-tenant?** At this scale, row-level is simpler to operate and
query. Migrating to schema-per-tenant later is possible but not needed now.

### Authentication Provider: Clerk (recommended)

[Clerk](https://clerk.com) is a managed auth service whose **Organizations** feature
maps directly to this design:

| This design | Clerk concept |
|---|---|
| `companies` | Organizations |
| `company_memberships` | Organization Memberships |
| `company_id` in JWT | `o.id` claim (built into JWT v2) |
| Role in JWT | `o.rol` claim (built into JWT v2) |
| Company switcher UI | `<OrganizationSwitcher />` component |

**FastAPI integration:** via `fastapi-clerk-auth` (PyPI, community-maintained) or
manual JWKS verification with PyJWT. Clerk's own FastAPI example repo is available at
`github.com/clerk/fastapi-example`.

**Roles strategy — cost trade-off:**

Clerk's built-in RBAC (custom roles beyond `admin`/`member`) requires the
**Enhanced B2B SaaS add-on** at +$100/month on top of Pro ($25/month).

**Recommended approach to avoid this cost:** Store role in
`organizationMembership.publicMetadata` (`{ "role": "staff" }`) and surface it as a
JWT claim via a Clerk JWT Template. Role enforcement lives in FastAPI middleware.
This preserves full control at no add-on cost.

**Pricing reality for this project:**

| Tier | Cost | Limits |
|---|---|---|
| Free | $0 | 100 orgs, 50k users, **5 members/org max** |
| Pro | $25/month | Removes member cap, 100 orgs included |
| Pro + B2B add-on | $125/month | Adds Clerk-managed custom roles/permissions |
| **Recommended** | **Pro ($25/month)** | Use metadata for roles, avoid add-on |

**Other Clerk notes:**
- JWT custom claims have ~1.2KB budget — be selective, don't embed whole metadata objects
- Metadata role changes take up to ~60s to appear in existing JWTs (next refresh)
- Webhooks available for all org/user/membership events via Svix (included free)
- Sync Clerk org events to your DB via webhooks to keep `companies` + `company_memberships` tables as a local mirror

### Company Context Flow

```
1. User visits jds-bakery.yourapp.com → slug identifies the company
2. POST /auth/login (Clerk handles credential verification)
   → Returns user + list of orgs (companies) they belong to
3a. Single company → auto-select active org in Clerk session
3b. Multiple companies → user picks via <OrganizationSwitcher />
4. All JWTs carry: { sub: user_id, o.id: company_id, o.rol: role }
5. FastAPI middleware extracts company_id + role, injects into request context
6. Repository layer always filters by company_id
```

### Customer Registration

Customers register **per company** at that company's storefront URL
(`jds-bakery.yourapp.com/register`). The slug is known at registration time and
automatically assigns the `customer` role in `company_memberships`.

The same email address can exist as a customer at multiple companies — they are
independent accounts per storefront. Staff/admin accounts are invited by the company
admin, not self-registered.

### Roles

| Role | Scope | Can do |
|---|---|---|
| `platform_admin` | Global | Manage all companies, users, billing; support access |
| `admin` | Company | Full company control — products, staff, schedules, orders |
| `staff` | Company | Manage products, schedules, view/update orders |
| `customer` | Company | Browse products, place and view own orders |

A user's role is per-company (stored in `company_memberships` and mirrored in Clerk
org membership metadata). The same person can be `admin` at one company and
`customer` at another.

---

## 3. Data Model

### 3.1 Core Entities

#### `companies`
Each tenant on the platform.

| Column | Type | Notes |
|---|---|---|
| `id` | UUID PK | |
| `legal_name` | VARCHAR(255) | Registered legal entity, e.g. "John Doe, LLC" |
| `dba_name` | VARCHAR(255) NULLABLE | Customer-facing trade name, e.g. "JD's Bakery" |
| `internal_name` | VARCHAR(100) UNIQUE | Short identifier for platform admin ops, e.g. "JDBAKERY" |
| `slug` | VARCHAR(100) UNIQUE | URL/subdomain identifier, e.g. `jds-bakery` → `jds-bakery.yourapp.com` |
| `is_active` | BOOLEAN | Platform admin can suspend a company |
| `is_accepting_orders` | BOOLEAN | **Global kill switch** — when false, all locations stop accepting orders |
| `created_at` | TIMESTAMPTZ | |

**Display name rule:** use `dba_name` if set, otherwise fall back to `legal_name`.
`internal_name` is never shown to customers or company staff — platform admin only.
`slug` is URL-safe, lowercase, hyphenated; used for subdomain or path routing.

#### `locations`
A physical storefront or location belonging to a company. Inherits company-level
templates but can override schedule and product availability independently.

| Column | Type | Notes |
|---|---|---|
| `id` | UUID PK | |
| `company_id` | UUID FK → `companies` | |
| `name` | VARCHAR(255) | e.g. "Downtown", "Northside" |
| `address` | TEXT | Full street address |
| `phone` | VARCHAR(50) NULLABLE | Location-specific contact |
| `timezone` | VARCHAR(100) | e.g. `America/Chicago` — critical for schedule windows |
| `schedule_template_id` | UUID FK → `schedule_templates` NULLABLE | NULL = use company's active template |
| `is_active` | BOOLEAN | |
| `is_accepting_orders` | BOOLEAN | **Per-location kill switch** — overrides company setting when false |
| `created_at` | TIMESTAMPTZ | |

**Kill switch hierarchy:**
- `company.is_accepting_orders = false` → all locations closed, regardless of location setting
- `location.is_accepting_orders = false` → only that location closed
- Both must be `true` for a location to accept orders

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
Defined at the **company level** and shared across all locations. A location cannot
create its own products — it can only show/hide company products via `location_product_overrides`.

| Column | Type | Notes |
|---|---|---|
| `id` | UUID PK | |
| `company_id` | UUID FK → `companies` | Tenant scope |
| `category_id` | UUID FK → `categories` NULLABLE | |
| `name` | VARCHAR(255) | |
| `description` | TEXT | |
| `price_cents` | INTEGER | Stored in cents |
| `product_type` | ENUM(`ready`, `made_to_order`) | Controls ordering flow |
| `is_active` | BOOLEAN | Permanently removed when false — affects all locations |
| `is_on_hold` | BOOLEAN | Temporarily hidden company-wide; staff can resume |
| `image_url` | TEXT NULLABLE | |
| `created_at` | TIMESTAMPTZ | |
| `updated_at` | TIMESTAMPTZ | |

#### `location_product_overrides`
Allows a location to hide a product it doesn't carry, without affecting other locations.
By default, all active company products are available at all locations.

| Column | Type | Notes |
|---|---|---|
| `id` | UUID PK | |
| `location_id` | UUID FK → `locations` | |
| `product_id` | UUID FK → `products` | |
| `is_available` | BOOLEAN | false = hidden at this location only |
| — | UNIQUE(`location_id`, `product_id`) | One override per product per location |

**Visibility rule for a product at a location:**
`product.is_active AND NOT product.is_on_hold AND (no override OR override.is_available = true)`

#### `categories`
Tenant-scoped groupings (e.g. Breads, Pastries, Beverages).

| Column | Type | Notes |
|---|---|---|
| `id` | UUID PK | |
| `company_id` | UUID FK → `companies` | |
| `name` | VARCHAR(100) | |
| `sort_order` | INTEGER | |

#### `schedule_templates`
Company-level weekly repeating production schedule. Applies only to `made_to_order`
products. A company can have multiple templates (e.g. "Standard Week", "Holiday Schedule").
Locations reference a template; if no template is assigned to a location, the company's
active default template is used.

| Column | Type | Notes |
|---|---|---|
| `id` | UUID PK | |
| `company_id` | UUID FK → `companies` | All templates belong to a company |
| `name` | VARCHAR(100) | e.g. "Standard Week", "Holiday Schedule" |
| `is_default` | BOOLEAN | Company's fallback template when a location has none assigned |
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
Concrete calendar-date instances generated per **location** from the template assigned
to that location (or the company default). Staff can block a window or adjust per day
at the location level without affecting other locations.

| Column | Type | Notes |
|---|---|---|
| `id` | UUID PK | |
| `company_id` | UUID FK → `companies` | For fast filtering |
| `location_id` | UUID FK → `locations` | Windows are per-location |
| `template_slot_id` | UUID FK → `schedule_template_slots` NULLABLE | NULL if manually created |
| `date` | DATE | |
| `pickup_start` | TIMESTAMPTZ | |
| `pickup_end` | TIMESTAMPTZ | |
| `is_blocked` | BOOLEAN | Staff can close (holiday, early sellout) at this location |
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
A customer's purchase at a specific location. May contain both ready and made-to-order
items. Payment collected in person.

| Column | Type | Notes |
|---|---|---|
| `id` | UUID PK | |
| `company_id` | UUID FK → `companies` | |
| `location_id` | UUID FK → `locations` | Which location the order is for |
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
    │               (is_accepting_orders = company kill switch)
    │
    ├──< categories
    │
    ├──< products (company-level, shared across locations)
    │         │
    ├──< schedule_templates (company-level, locations reference one)
    │         └──< schedule_template_slots >──< product_slot_capacity >── products
    │
    └──< locations (is_accepting_orders = location kill switch)
              │   └──< location_product_overrides >── products
              │         (hide products not carried at this location)
              │
              │   schedule_template_id → schedule_templates
              │         (inherit company default or assign own)
              │
              └──< availability_windows >──< window_product_capacity >── products
                             │
                        orders (location_id + pickup_window_id)
                          └──< order_items >── products
                         customer (users)
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

### Platform Admin [is_platform_admin only]
```
GET    /platform/companies               # List all tenants
POST   /platform/companies               # Create new company (triggers onboarding)
GET    /platform/companies/{id}
PATCH  /platform/companies/{id}          # Update names, slug, status, kill switch
GET    /platform/companies/{id}/billing
PUT    /platform/companies/{id}/billing

GET    /platform/users                   # All platform users
PATCH  /platform/users/{id}/promote      # Grant is_platform_admin
PATCH  /platform/users/{id}/demote       # Revoke is_platform_admin
```

### Company Management [admin]
```
GET    /company/profile
PATCH  /company/profile                            # Update name, slug
PATCH  /company/profile/orders/toggle              # Toggle is_accepting_orders (kill switch)

GET    /company/members                            # List staff + customers
POST   /company/members                            # Invite staff by email
PATCH  /company/members/{user_id}                  # Change role
DELETE /company/members/{user_id}                  # Remove from company

GET    /company/customers                          # List customer accounts
PATCH  /company/customers/{user_id}                # Edit customer (flag, notes)
DELETE /company/customers/{user_id}                # Remove customer account
```

### Locations [admin]
```
GET    /locations                                  # List company locations
POST   /locations                                  # Create location
GET    /locations/{id}
PUT    /locations/{id}                             # Update details
PATCH  /locations/{id}/orders/toggle               # Toggle location kill switch [staff+]
PATCH  /locations/{id}/schedule                    # Assign a schedule template
GET    /locations/{id}/products                    # Products with override status
PATCH  /locations/{id}/products/{product_id}       # Set is_available override [staff+]
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
GET    /locations/{id}/availability                        # Windows for a location by date range (public)
POST   /locations/{id}/availability/generate               # Generate from assigned template [staff+]
PATCH  /locations/{id}/availability/{window_id}            # Block/unblock a day [staff+]
GET    /locations/{id}/availability/{window_id}/capacity
PATCH  /locations/{id}/availability/{window_id}/capacity/{product_id}  # Override qty [staff+]
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
| 13 | Company naming | `legal_name`, `dba_name` (customer-facing), `internal_name` (platform admin only), `slug` (URL/subdomain) |
| 14 | Platform admin onboarding | Dedicated super admin dashboard + `company_billing` table for billing/plan tracking |
| 15 | Backup & recovery | Railway automated snapshots + transactional writes + idempotent generation (see Section 14) |
| 16 | Auth provider | Clerk — Organizations map to companies; roles stored in org membership metadata to avoid $100/month add-on; Pro plan ($25/month) |
| 17 | Customer registration | Per-company via storefront slug; same email can exist at multiple companies independently |
| 18 | Billing model | Flat monthly tiers (Trial free / Starter $29 / Pro $79); gated by staff seats, product count, template count |
| 19 | Locations | Companies support multiple locations; products are company-level (shared); schedules and availability windows are per-location |
| 20 | Template inheritance | Company defines default schedule template; locations can assign their own or inherit default |
| 21 | Product visibility | Locations can hide company products via `location_product_overrides`; cannot create location-only products |
| 22 | Kill switch | Two-level: company (global) and location (local); both must be true to accept orders |
| 23 | Platform admin management | Platform admins can promote/demote other platform admins via `/platform/users/{id}/promote` |
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
│   ├── company.py          # companies, company_memberships, company_billing
│   ├── location.py         # locations, location_product_overrides
│   ├── user.py
│   ├── product.py          # products, categories
│   ├── order.py            # orders, order_items
│   └── schedule.py         # templates, slots, windows, capacity
├── schemas/
│   ├── company.py
│   ├── location.py
│   ├── user.py
│   ├── product.py
│   ├── order.py
│   └── schedule.py
├── repositories/
│   ├── company_repo.py
│   ├── location_repo.py
│   ├── user_repo.py
│   ├── product_repo.py
│   ├── order_repo.py
│   └── schedule_repo.py
├── services/
│   ├── auth_service.py       # Login, company selection, JWT issuance
│   ├── company_service.py    # Company + platform admin management
│   ├── location_service.py   # Location CRUD, kill switch, product overrides
│   ├── product_service.py
│   ├── order_service.py      # Mixed ready + made-to-order carts, kill switch check
│   └── schedule_service.py   # Window generation (per-location) + capacity enforcement
├── routers/
│   ├── auth.py
│   ├── platform.py           # Platform admin: companies, promote/demote admins
│   ├── company.py            # Company profile, members, customers
│   ├── locations.py          # Location CRUD, kill switch, product overrides, availability
│   ├── products.py
│   ├── categories.py
│   ├── schedules.py
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

## 12. Billing Model

### Recommended: Flat Monthly Tiers

Small business owners prefer predictable bills. A tiered flat-fee model is easiest
to sell, explain, and enforce.

| Plan | Price | Limits |
|---|---|---|
| **Trial** | Free, 30 days | Full features, 1 staff seat |
| **Starter** | $29/month | Up to 3 staff, 50 active products, 1 schedule template |
| **Pro** | $79/month | Unlimited staff, unlimited products, unlimited templates |

### What Gets Gated by Plan

Enforced server-side in a `plan_enforcement` middleware layer that checks
`company_billing.plan` before allowing writes:

| Feature | Trial | Starter | Pro |
|---|---|---|---|
| Active products | 50 | 50 | Unlimited |
| Staff/admin seats | 1 | 3 | Unlimited |
| Schedule templates | 1 | 1 | Unlimited |
| Order history (months) | 3 | 12 | Unlimited |
| Email notification customization | — | — | ✓ |
| Analytics/reporting | — | — | ✓ |

### Future Billing Infrastructure

- **Stripe** for subscription management and invoicing (future epic)
- `company_billing.stripe_customer_id` is already in the schema for when this is built
- For now: manual invoicing tracked in `company_billing.notes`

---

## 13. Platform Admin — Company Onboarding


> **TODO:** Build a dedicated super admin dashboard for onboarding and managing companies.

This is separate from the company-facing UI. Only `is_platform_admin` users can access it.

### Onboarding Flow (manual for now)
1. Platform admin creates the company record via the dashboard
2. Fills in all name fields, slug, and billing details
3. Creates the first `admin` membership (invites the business owner by email)
4. Company admin takes over from there (products, schedules, staff)

### `company_billing`
Stores billing and contact information visible only to the platform admin.

| Column | Type | Notes |
|---|---|---|
| `id` | UUID PK | |
| `company_id` | UUID FK → `companies` UNIQUE | One billing record per company |
| `billing_email` | VARCHAR(255) | Invoice recipient |
| `billing_address` | TEXT NULLABLE | Full mailing address |
| `plan` | ENUM(`trial`, `starter`, `pro`) | Subscription tier (expand as needed) |
| `plan_started_at` | DATE NULLABLE | |
| `trial_ends_at` | DATE NULLABLE | |
| `stripe_customer_id` | VARCHAR(255) NULLABLE | For future online billing |
| `notes` | TEXT NULLABLE | Internal notes — payment history, special terms, support context |
| `created_at` | TIMESTAMPTZ | |
| `updated_at` | TIMESTAMPTZ | |

### Platform Admin API (future dashboard)
```
GET    /platform/companies                  # List + search all companies
POST   /platform/companies                  # Onboard new company
GET    /platform/companies/{id}             # Full detail incl. billing
PATCH  /platform/companies/{id}             # Update names, slug, status
GET    /platform/companies/{id}/billing     # Billing record
PUT    /platform/companies/{id}/billing     # Update billing info
PATCH  /platform/companies/{id}/suspend     # Suspend/unsuspend
GET    /platform/users                      # All platform users
GET    /platform/orders                     # Cross-company order visibility (support)
```

---

## 14. Backup & Failure Recovery

> **TODO:** Define and implement backup strategy before going to production.

### Database Backups
- **Railway/Render** provide automated daily PostgreSQL snapshots — verify retention period (aim for 7–30 days)
- Enable **point-in-time recovery (PITR)** if the hosting provider supports it
- Periodically test restores — a backup that has never been restored is unverified

### Application-Level Concerns
- **Capacity reservation failures:** If an order is inserted but the email fails, the order is still valid. A staff-facing order list is the fallback — never depend solely on email.
- **Partial order writes:** All order + capacity updates happen in a single DB transaction; a crash mid-request leaves no partial state.
- **Idempotent window generation:** `POST /availability/generate` skips existing windows, so re-running after a failure is safe.

### Future Hardening
- [ ] Database replication / read replica for reporting queries
- [ ] Automated backup restore test (monthly)
- [ ] Health check endpoint (`GET /health`) for uptime monitoring (e.g. UptimeRobot)
- [ ] Error tracking (e.g. Sentry) to catch and alert on production exceptions
- [ ] Audit log table for critical mutations (order status changes, capacity overrides)

---

## 15. Next Steps

- [x] Resolve all design questions
- [ ] Scaffold FastAPI project (SQLAlchemy models, Alembic, JWT auth with company context)
- [ ] Implement `auth_service` — login → company list → scoped JWT
- [ ] Implement `schedule_service` — window generation + mixed-cart capacity enforcement
- [ ] Build platform admin dashboard (company onboarding, billing management)
- [ ] Set up Railway + PostgreSQL with automated backups verified
- [ ] Integrate Resend for order status notification emails
- [ ] Set up Sentry for error tracking + UptimeRobot for health monitoring
- [ ] Build OpenAPI schema, share with web + iOS teams
- [ ] Future: online payment (Stripe), per-company email branding, multi-location
