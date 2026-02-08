# IMS System Overview

> **Version:** Multi-Tenant Phase 1
> **Last updated:** February 2026

---

## Table of Contents

1. [Architecture](#1-architecture)
2. [Tech Stack](#2-tech-stack)
3. [Monorepo Structure](#3-monorepo-structure)
4. [Backend Deep Dive](#4-backend-deep-dive)
5. [Frontend Deep Dive](#5-frontend-deep-dive)
6. [Multi-Tenancy Architecture](#6-multi-tenancy-architecture)
7. [Subscription & Plan System](#7-subscription--plan-system)
8. [Data Model](#8-data-model)
9. [Authentication & Authorization](#9-authentication--authorization)
10. [Infrastructure](#10-infrastructure)

---

## 1. Architecture

```
                    ┌─────────────────────────────────────────────┐
                    │              Client (Browser)                │
                    └────────────────────┬────────────────────────┘
                                         │
                           ┌─────────────┴─────────────┐
                           │                           │
                    ┌──────┴──────┐            ┌───────┴───────┐
                    │  Next.js    │            │  Express API   │
                    │  Frontend   │───────────>│  Backend       │
                    │  (port 3000)│  REST API  │  (port 4000)   │
                    └─────────────┘            └───────┬───────┘
                                                       │
                                               ┌───────┴───────┐
                                               │  PostgreSQL    │
                                               │  Database      │
                                               └───────────────┘
```

**Request flow for tenant-scoped routes:**

```
Browser Request
  → Express receives request
    → requestId middleware (assigns unique ID)
    → requestLogging middleware (logs request)
    → verifyToken middleware (validates JWT, extracts user + tenantId)
    → resolveTenant middleware (loads tenant, sets AsyncLocalStorage context)
    → checkSubscription middleware (enforces plan status)
    → Route Handler / Controller
      → Prisma Client (auto-scoped queries via tenantId from AsyncLocalStorage)
        → PostgreSQL
```

---

## 2. Tech Stack

### Backend
| Technology | Purpose |
|---|---|
| **Node.js + Express** | HTTP server and REST API |
| **TypeScript** | Type safety |
| **Prisma** (v5.22) | ORM and database migrations |
| **PostgreSQL** | Primary database |
| **JWT** | Stateless authentication |
| **bcryptjs** | Password hashing |
| **AsyncLocalStorage** | Request-scoped tenant context |
| **Winston** (custom logger) | Structured logging |

### Frontend
| Technology | Purpose |
|---|---|
| **Next.js 14+** (App Router) | React framework with SSR/SSG |
| **TypeScript** | Type safety |
| **TanStack Query** (React Query) | Server state management, caching |
| **Zustand** | Client state management (auth, tenant) |
| **Tailwind CSS** | Utility-first styling |
| **shadcn/ui** | Component library (Radix primitives) |
| **React Hook Form + Zod** | Form management and validation |
| **Recharts** | Charts and analytics visualization |
| **Lucide React** | Icon library |
| **Axios** | HTTP client with interceptors |

### Tooling
| Technology | Purpose |
|---|---|
| **Turborepo** | Monorepo build orchestration |
| **pnpm** | Fast, disk-efficient package manager |
| **Husky** | Git hooks (pre-commit) |
| **lint-staged** | Run linters on staged files |
| **ESLint** | Code linting |
| **Prettier** | Code formatting |
| **Docker** | Containerization |

---

## 3. Monorepo Structure

```
ims/
├── apps/
│   ├── api/                        # Express backend
│   │   ├── prisma/
│   │   │   ├── schema.prisma       # Database schema
│   │   │   └── seed.ts             # Database seeding
│   │   └── src/
│   │       ├── config/             # App config, Prisma client, router, logger
│   │       ├── middlewares/        # Auth, tenant, subscription, error handling
│   │       └── modules/           # Feature modules (auth, platform, etc.)
│   │           ├── auth/
│   │           ├── platform/
│   │           ├── users/
│   │           ├── products/
│   │           ├── categories/
│   │           ├── sales/
│   │           ├── members/
│   │           ├── vendors/
│   │           ├── locations/
│   │           ├── inventory/
│   │           ├── transfers/
│   │           ├── promos/
│   │           ├── analytics/
│   │           ├── dashboard/
│   │           ├── audit-logs/
│   │           ├── error-reports/
│   │           └── bulk/
│   │
│   └── web/                        # Next.js frontend
│       ├── app/                    # Next.js App Router pages
│       │   ├── [workspace]/        # Dynamic workspace routes
│       │   │   ├── (admin)/        # Admin-accessible routes
│       │   │   └── (superadmin)/   # SuperAdmin-only routes
│       │   └── login/              # Login page
│       ├── components/             # Reusable UI components
│       │   ├── auth/               # Auth guard, login form
│       │   ├── layout/             # Sidebar, top bar, dashboard layout
│       │   └── ui/                 # shadcn/ui components
│       ├── hooks/                  # Custom React hooks (data fetching)
│       ├── services/               # API service layer
│       ├── stores/                 # Zustand stores
│       ├── views/                  # Page-level view components
│       ├── config/                 # Routes, constants
│       ├── lib/                    # Utilities (axios, cn, apiError)
│       └── utils/                  # Type definitions
│
├── packages/
│   ├── shared/                     # Shared types, utils, feature registry
│   │   └── src/
│   │       ├── config/features.ts  # Plan tiers, feature definitions, limits
│   │       ├── types/user.ts       # UserRole enum
│   │       └── utils/date.ts       # Date utilities
│   ├── eslint-config/              # Shared ESLint configurations
│   └── typescript-config/          # Shared TypeScript configurations
│
├── scripts/
│   ├── test-tenants.sh             # Multi-tenant isolation test
│   └── check-user.ts              # User inspection utility
│
├── docs/                           # Documentation
├── docker-compose.yml              # Base Docker Compose
├── docker-compose.dev.yml          # Development Docker Compose
├── docker-compose.prod.yml         # Production Docker Compose
├── turbo.json                      # Turborepo config
└── pnpm-workspace.yaml             # pnpm workspace config
```

---

## 4. Backend Deep Dive

### API Endpoints (17 Route Groups)

| Route Group | Base Path | Auth Required | Tenant Scoped | Description |
|---|---|---|---|---|
| Auth | `/auth` | No | No | Login, logout, current user |
| Platform | `/platform` | Yes (platformAdmin) | No | Tenant CRUD, plan management |
| Users | `/users` | Yes (superAdmin) | Yes | User CRUD |
| Products | `/products` | Yes | Yes | Product CRUD, discounts |
| Categories | `/categories` | Yes | Yes | Category/subcategory CRUD |
| Vendors | `/vendors` | Yes | Yes | Vendor CRUD |
| Locations | `/locations` | Yes | Yes | Location CRUD |
| Inventory | `/inventory` | Yes | Yes | Stock management |
| Transfers | `/transfers` | Yes | Yes | Transfer workflow |
| Members | `/members` | Yes | Yes | Member CRM |
| Sales | `/sales` | Yes | Yes | Sales + analytics |
| Promos | `/promos` | Yes | Yes | Promo code management |
| Analytics | `/analytics` | Yes | Yes | Business intelligence |
| Dashboard | `/dashboard` | Yes | Yes | Role-based summaries |
| Audit Logs | `/audit-logs` | Yes (superAdmin) | Yes | Activity tracking |
| Error Reports | `/error-reports` | Yes | Yes | Bug reporting |
| Bulk | `/bulk` | Yes | Yes | Upload/download operations |

### Middleware Chain

```
┌──────────────────────────────────────────────────────────────────────┐
│  Public Routes: /auth/*                                              │
│  → No middleware (login, logout)                                     │
├──────────────────────────────────────────────────────────────────────┤
│  Platform Routes: /platform/*                                        │
│  → verifyToken → authorizeRoles("platformAdmin") → handler           │
├──────────────────────────────────────────────────────────────────────┤
│  All Other Routes:                                                   │
│  → verifyToken → resolveTenant → checkSubscription → handler         │
└──────────────────────────────────────────────────────────────────────┘
```

### Prisma Auto-Scoping

The Prisma client is extended to automatically inject `tenantId` into queries:

- **`findMany`, `findFirst`, `count`, `aggregate`, `groupBy`**: `tenantId` added to `where` clause
- **`create`**: `tenantId` added to `data`
- **`createMany`**: `tenantId` added to each item in `data` array
- **`update`, `updateMany`, `delete`, `deleteMany`**: `tenantId` added to `where` clause
- **`findUnique`**: NOT modified (uses compound unique constraints or ID)

Models scoped: `User, Category, Product, DiscountType, Vendor, Location, Transfer, Member, Sale, PromoCode, AuditLog, ErrorReport`

---

## 5. Frontend Deep Dive

### State Management

```
┌─────────────────────────────────────────────┐
│  Zustand Store (Client State)                │
│  ├── auth-store: user, token, tenant         │
│  └── Persisted in cookies (7-day expiry)     │
├─────────────────────────────────────────────┤
│  TanStack Query (Server State)               │
│  ├── Automatic caching & revalidation        │
│  ├── Optimistic updates                      │
│  └── Query key invalidation on mutations     │
└─────────────────────────────────────────────┘
```

### Data Flow Pattern

```
View Component
  → Custom Hook (e.g., useProduct)
    → Service Function (e.g., productService.getProducts)
      → Axios Instance (auto-injects token + X-Tenant-Slug header)
        → Backend API
```

### Page Organization

Pages follow the **View + Hook + Service** pattern:

```
app/[workspace]/(admin)/product/page.tsx    → Thin page wrapper
  └── views/products/ProductIndex.tsx        → View component with UI
        └── hooks/useProduct.ts              → Data fetching + mutations
              └── services/productService.ts → API calls
```

### Feature Inventory

| Feature | Pages | CRUD | Bulk Ops | Analytics |
|---|---|---|---|---|
| Products | List, New, Edit, Catalog | Full | Upload + Export | - |
| Categories | List (with inline edit) | Full | - | - |
| Sales | List, New, Detail | Create + Read | Upload + Export | Summary, Daily, By Location |
| Members | List, New, Edit | Full | Upload + Export | - |
| Locations | List, New, Edit | Full | - | - |
| Vendors | List, New, Edit | Full | - | - |
| Transfers | List, New, Detail | Full workflow | - | - |
| Promos | List, New, Edit | Full | - | - |
| Analytics | 4 pages | Read | - | Sales, Inventory, Customers, Pivot |
| Dashboard | 1 page (role-based) | Read | - | KPIs, Charts, Alerts |
| Users | List, New, Edit | Full | - | - |
| Audit Logs | List | Read | - | - |
| Error Reports | List + Dialog | Create + Status | - | - |
| Settings | 1 page | Update password | - | - |

---

## 6. Multi-Tenancy Architecture

### Approach: Shared Database with Row-Level Isolation

All tenants share a single PostgreSQL database. Data is isolated by a `tenantId` column on every business model.

```
┌─────────────────────────────────────────────────┐
│                Single PostgreSQL DB               │
│                                                   │
│  ┌─────────────┐  ┌─────────────┐  ┌──────────┐ │
│  │  Tenant A   │  │  Tenant B   │  │ Platform │ │
│  │  tenantId=1 │  │  tenantId=2 │  │  (admin) │ │
│  │  ─────────  │  │  ─────────  │  │          │ │
│  │  Products   │  │  Products   │  │ Tenants  │ │
│  │  Sales      │  │  Sales      │  │ Plans    │ │
│  │  Members    │  │  Members    │  │ Payments │ │
│  │  ...        │  │  ...        │  │          │ │
│  └─────────────┘  └─────────────┘  └──────────┘ │
└─────────────────────────────────────────────────┘
```

### How Isolation Works

1. **Login**: User authenticates with `X-Tenant-Slug` header → JWT contains `tenantId`
2. **Middleware**: `resolveTenant` reads `tenantId` from JWT, stores it in `AsyncLocalStorage`
3. **Prisma Extension**: All queries automatically add `WHERE tenantId = ?` or `SET tenantId = ?`
4. **Platform Admin**: Bypasses scoping with `bypassScoping: true` flag

### Tenant Identification

```
Browser → POST /auth/login (X-Tenant-Slug: "test-org")
  → Backend resolves tenant by slug
  → JWT issued with { tenantId, tenantSlug, userId, role }
  → Frontend stores tenant info in Zustand
  → All subsequent requests include:
      - Authorization: Bearer <jwt> (contains tenantId)
      - X-Tenant-Slug: test-org (from Zustand store, via axios interceptor)
```

### Compound Unique Constraints

Fields that were globally unique are now unique per tenant:

```sql
-- Before multi-tenancy
UNIQUE (username)
UNIQUE (imsCode)
UNIQUE (phone)

-- After multi-tenancy
UNIQUE (tenantId, username)
UNIQUE (tenantId, imsCode)
UNIQUE (tenantId, phone)
```

---

## 7. Subscription & Plan System

### Plan Tiers

| Feature | STARTER | PROFESSIONAL | ENTERPRISE |
|---|---|---|---|
| Max Users | 3 | 10 | Unlimited |
| Max Products | 100 | 1,000 | Unlimited |
| Max Locations | 2 | 10 | Unlimited |
| Max Members | 500 | 5,000 | Unlimited |
| Bulk Upload | No | Yes | Yes |
| Analytics | No | Yes | Yes |
| Promo Management | No | Yes | Yes |
| Audit Logs | No | No | Yes |
| API Access | No | No | Yes |

### Pricing (NPR)

| Plan | Monthly | Annual (save ~17%) |
|---|---|---|
| STARTER | Rs. 2,000 | Rs. 20,000 |
| PROFESSIONAL | Rs. 5,000 | Rs. 50,000 |
| ENTERPRISE | Rs. 12,000 | Rs. 120,000 |

### Subscription Status Lifecycle

```
  New Tenant
      │
      ▼
   ┌──────┐    Payment     ┌────────┐
   │ TRIAL │──────────────>│ ACTIVE │
   │(14 d) │               │        │
   └──┬───┘               └───┬────┘
      │ Trial expires          │ Plan expires
      ▼                        ▼
   ┌──────────┐          ┌──────────┐
   │ SUSPENDED │<────────│ PAST_DUE │
   │(read-only)│         │ (warning)│
   └────┬─────┘          └──────────┘
        │ Grace period expires
        ▼
   ┌────────┐    Admin     ┌───────────┐
   │ LOCKED │──────────────>│ CANCELLED │
   │(blocked)│   action     │ (final)   │
   └────────┘              └───────────┘
```

### Access Control by Status

| Status | Read | Write | Billing | Description |
|---|---|---|---|---|
| TRIAL | Yes | Yes | No | Free trial period (14 days) |
| ACTIVE | Yes | Yes | No | Paid and current |
| PAST_DUE | Yes | Yes | Warning | Payment overdue, grace period |
| SUSPENDED | Yes | No | Blocked | Reads only, writes blocked |
| LOCKED | No | No | Blocked | Full lockout |
| CANCELLED | No | No | Blocked | Tenant deactivated |

### Feature Registry

Defined in `packages/shared/src/config/features.ts`:

| Feature | Minimum Plan | Description |
|---|---|---|
| BULK_UPLOAD_PRODUCTS | PROFESSIONAL | Upload products via Excel |
| BULK_UPLOAD_SALES | PROFESSIONAL | Upload sales via Excel |
| ANALYTICS_BASIC | PROFESSIONAL | Basic business analytics |
| ANALYTICS_ADVANCED | ENTERPRISE | Advanced analytics & reporting |
| PROMO_MANAGEMENT | PROFESSIONAL | Create and manage promo codes |
| AUDIT_LOGS | ENTERPRISE | Full audit trail |
| API_ACCESS | ENTERPRISE | External API access |
| MULTIPLE_LOCATIONS | PROFESSIONAL | More than 2 locations |

---

## 8. Data Model

### Entity Relationship Overview

```
Tenant (1) ─────────┬──> (N) User
                     ├──> (N) Category ───> (N) SubCategory
                     ├──> (N) Product ──┬──> (N) ProductVariation ──> (N) ProductSubVariation
                     │                  ├──> (N) ProductDiscount
                     │                  └──> (N) VariationPhoto
                     ├──> (N) DiscountType
                     ├──> (N) Vendor
                     ├──> (N) Location ───> (N) LocationInventory
                     ├──> (N) Transfer ──┬──> (N) TransferItem
                     │                   └──> (N) TransferLog
                     ├──> (N) Member
                     ├──> (N) Sale ──────┬──> (N) SaleItem
                     │                   └──> (N) SalePayment
                     ├──> (N) PromoCode ──> (N) PromoCodeProduct
                     ├──> (N) AuditLog
                     └──> (N) ErrorReport

PlanLimit (per tier)
PricingPlan (per tier + billing cycle)
Subscription (per tenant)
TenantPayment (payment records)
```

### Key Enums

| Enum | Values |
|---|---|
| Role | platformAdmin, superAdmin, admin, user |
| PlanTier | STARTER, PROFESSIONAL, ENTERPRISE |
| SubscriptionStatus | TRIAL, ACTIVE, PAST_DUE, SUSPENDED, LOCKED, CANCELLED |
| LocationType | WAREHOUSE, SHOWROOM |
| TransferStatus | PENDING, APPROVED, IN_TRANSIT, COMPLETED, CANCELLED |
| SaleType | GENERAL, MEMBER |
| PaymentMethod | CASH, CARD, CHEQUE, FONEPAY, QR |
| MemberStatus | ACTIVE, INACTIVE, PROSPECT, VIP |
| PaymentGateway | ESEWA, KHALTI, FONEPAY, CONNECTIPS, BANK_TRANSFER, MANUAL |

---

## 9. Authentication & Authorization

### JWT Token Structure

```json
{
  "id": "user-uuid",
  "username": "admin",
  "role": "superAdmin",
  "tenantId": "tenant-uuid",
  "tenantSlug": "default",
  "iat": 1707350400,
  "exp": 1707436800
}
```

### Auth Flow

```
1. POST /auth/login { username, password } + X-Tenant-Slug header
2. Server resolves tenant, finds user within tenant, verifies password
3. Server returns: { token, user, tenant }
4. Frontend stores token + user + tenant in Zustand (persisted in cookies)
5. Axios interceptor adds Authorization + X-Tenant-Slug to all requests
6. On 401: Auto-logout and redirect to /login
```

### Role Hierarchy

```
platformAdmin  ──  Cross-tenant access, manages all tenants
     │
superAdmin     ──  Full access within their tenant
     │
admin          ──  Manages products, inventory, sales, locations, vendors, promos
     │
user           ──  Creates sales, views catalog, limited access
```

---

## 10. Infrastructure

### Docker Configuration

**Development** (`docker-compose.dev.yml`):
- PostgreSQL with volume persistence
- Hot reloading for API and Web
- Port mapping: API on 4000, Web on 3000

**Production** (`docker-compose.prod.yml`):
- Multi-stage builds for smaller images
- PostgreSQL with persistent volume
- Nginx reverse proxy (if configured)
- Health checks

### Environment Variables

| Variable | Description | Example |
|---|---|---|
| `DATABASE_URL` | PostgreSQL connection string | `postgresql://user:pass@host:5432/ims` |
| `JWT_SECRET` | Secret key for JWT signing | Random secure string |
| `PORT` | API server port | `4000` |
| `HOST` | API server host | `0.0.0.0` |
| `SUPERADMIN_USERNAME` | Default super admin username | `admin` |
| `SUPERADMIN_PASSWORD` | Default super admin password | Secure password |
| `NEXT_PUBLIC_API_URL` | Frontend API URL | `http://localhost:4000/api/v1` |

### CI/CD

- GitHub Actions workflow for Docker build and push (`.github/workflows/docker-build-push.yml`)
- Pre-commit hooks: formatting, linting, type checking (via Husky + lint-staged)

### Dev Commands

```bash
# Start development
pnpm dev                          # Start all apps
pnpm --filter api dev             # Start API only
pnpm --filter web dev             # Start frontend only

# Database
npx prisma db push                # Sync schema to DB
npx prisma db seed                # Run seed script
npx prisma db push --force-reset  # Reset DB + sync schema
npx prisma studio                 # Open Prisma Studio GUI

# Quality
pnpm lint                         # Run linters
pnpm check-types                  # Run TypeScript checks

# Testing
source apps/api/.env && bash scripts/test-tenants.sh  # Multi-tenant test

# Docker
docker compose -f docker-compose.dev.yml up   # Dev environment
docker compose -f docker-compose.prod.yml up   # Production
```
