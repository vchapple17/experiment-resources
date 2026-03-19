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

### Authentication Provider: Supabase Auth

[Supabase Auth](https://supabase.com/docs/guides/auth) handles user registration,
login, JWT issuance, and session management. It is open source (GoTrue), self-hostable,
and has no per-user cost at this scale. Auth is kept intentionally simple — roles are
global, not per-company, for now.

**Stack benefit:** If using Supabase for the PostgreSQL database as well, auth and
data live in the same platform — one dashboard, one billing account, no separate
Railway/Render setup needed.

### JWT Structure

Supabase issues standard JWTs (RS256). A `custom_access_token_hook` (a Postgres
function) is used to inject role and company context at token issuance time:

```json
{
  "sub": "user-uuid",
  "role": "staff",
  "company_id": "company-uuid",
  "iat": 1234567890,
  "exp": 1234567890
}
```

The hook queries `company_memberships` at login to determine the user's role and
default company. When multi-tenancy is added later, this hook can be extended to
support company selection and scoped tokens with minimal change.

### FastAPI Integration

```python
from jose import jwt, JWTError

class JWTBearer(HTTPBearer):
    async def __call__(self, request: Request):
        credentials = await super().__call__(request)
        return jwt.decode(
            credentials.credentials,
            SUPABASE_JWT_SECRET,
            algorithms=["HS256"],
            audience="authenticated"
        )

# Protect a route
@router.get("/orders")
async def list_orders(claims: dict = Depends(JWTBearer())):
    role = claims["role"]
    company_id = claims["company_id"]
```

**Important:** Never use the Supabase `service_role` key in FastAPI routes — it
bypasses all security. Only verify user JWTs.

### Auth Flow

```
1. Customer visits jds-bakery.yourapp.com/register
   → Supabase creates user, assigns customer role via hook
   → company_memberships row inserted

2. Staff/admin are invited by company admin
   → Supabase sends magic link / email invite
   → Role assigned in company_memberships on acceptance

3. Login → Supabase issues JWT with role + company_id claims
4. FastAPI middleware verifies JWT on every request
5. Repository layer filters all queries by company_id from token
```

### Roles

| Role | Scope | Can do |
|---|---|---|
| `platform_admin` | Global | Manage all companies, users, billing; full access |
| `admin` | Company | Full company control — products, staff, schedules, orders |
| `staff` | Company | Manage products, schedules, view/update orders |
| `customer` | Company | Browse products, place and view own orders |

Roles are stored in `company_memberships.role` and surfaced in the JWT via the
custom hook. Role checks happen in FastAPI middleware — not in Supabase RLS for now,
keeping complexity low.

### Future: Adding Multi-Tenancy to Auth

When a second company is onboarded:
1. Extend the custom JWT hook to accept a `company_id` parameter at login
2. Add a company-selection step after login (if user has multiple memberships)
3. Re-issue a token scoped to the selected company
4. No schema changes required — `company_id` is already on every table

### Pricing

| Plan | Cost | Limits |
|---|---|---|
| Free | $0 | 50,000 MAU — pauses after 7 days inactivity (dev only) |
| **Pro** | **$25/month** | 100,000 MAU, no pausing, daily backups |

**Recommendation: Pro ($25/month)** from day one in production. The free tier pauses
inactive projects and is not suitable for a live app.

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
| 16 | Auth provider | **Supabase Auth** — simple global roles via custom JWT hook; open source; no per-user cost; upgrade path to multi-tenant auth when needed |
| 17 | Customer registration | Self-register at company storefront URL; assigned `customer` role via JWT hook |
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
│   ├── auth_service.py       # JWT verification middleware, role extraction
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

### Recommended Stack

| Service | Role | Cost |
|---|---|---|
| **Supabase** | PostgreSQL database + Auth | $25/month (Pro) |
| **Railway** or **Render** | FastAPI application server | ~$5–10/month |
| **Resend** | Transactional email | Free tier (3,000 emails/month) |

Using Supabase for both database and auth consolidates two infrastructure concerns into
one platform and one bill. Supabase Pro includes daily backups, no project pausing,
and 8GB database storage.

### FastAPI on Railway

- Connect GitHub repo → Railway auto-detects FastAPI
- Set environment variables: `SUPABASE_URL`, `SUPABASE_JWT_SECRET`, `RESEND_API_KEY`
- Alembic migrations run on deploy via a release command
- Auto-deploys on push to `main`

### What You'll Need

- `Dockerfile` or `railway.toml` (minimal for FastAPI)
- Supabase project configured with custom JWT hook (Postgres function)
- Alembic configured to point at Supabase's connection string
- Resend API key for order notification emails

### Total Monthly Cost at Launch

| | |
|---|---|
| Supabase Pro | $25 |
| Railway (Hobby) | ~$5–10 |
| Resend | $0 |
| **Total** | **~$30–35/month** |

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
- [x] Select auth provider: Supabase Auth
- [ ] Create Supabase project (Pro plan); configure custom JWT hook for role + company_id claims
- [ ] Scaffold FastAPI project (SQLAlchemy + Alembic pointed at Supabase Postgres)
- [ ] Implement JWT verification middleware using Supabase JWT secret
- [ ] Implement `schedule_service` — window generation (per-location) + capacity enforcement
- [ ] Build platform admin dashboard (company onboarding, billing management, promote/demote admins)
- [ ] Set up Railway for FastAPI app server; wire Supabase env vars
- [ ] Integrate Resend for order lifecycle notification emails
- [ ] Set up Sentry for error tracking + UptimeRobot for health monitoring
- [ ] Build OpenAPI schema, share with web + iOS teams
- [ ] Future: online payment (Stripe), multi-tenant auth extension, per-company email branding
