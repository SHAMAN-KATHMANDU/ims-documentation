# Remaining Tasks & Polish Roadmap

> **Current state:** Multi-Tenant Phase 1 (Foundation) complete
> **Last updated:** February 2026

---

## Table of Contents

1. [What's Done](#1-whats-done)
2. [Critical Remaining Work](#2-critical-remaining-work)
3. [High Priority Polish](#3-high-priority-polish)
4. [Medium Priority Enhancements](#4-medium-priority-enhancements)
5. [Low Priority / Nice-to-Have](#5-low-priority--nice-to-have)
6. [Technical Debt](#6-technical-debt)
7. [Suggested Implementation Order](#7-suggested-implementation-order)

---

## 1. What's Done

### Multi-Tenant Foundation (Phase 1) -- COMPLETE
- [x] Prisma schema updated with `tenantId` on all business models
- [x] Compound unique constraints (`[tenantId, field]`)
- [x] `AsyncLocalStorage`-based tenant context
- [x] Prisma client extensions for automatic query scoping
- [x] Tenant resolution middleware (from JWT)
- [x] Subscription status enforcement middleware (TRIAL/ACTIVE/SUSPENDED/LOCKED/CANCELLED)
- [x] Auth controller updated for tenant-aware login
- [x] JWT now contains `tenantId` and `tenantSlug`
- [x] Platform admin role and routes (`/platform/*`)
- [x] Shared feature registry (`packages/shared/src/config/features.ts`)
- [x] Plan limits, pricing plans, subscription data model
- [x] Frontend: auth store persists tenant info
- [x] Frontend: axios auto-sends `X-Tenant-Slug` header
- [x] Frontend: login form has organization selector
- [x] Frontend: sidebar shows tenant name + plan badge
- [x] Frontend: top bar shows tenant in user dropdown
- [x] Comprehensive seed with two distinct tenants
- [x] Automated isolation test script

---

## 2. Critical Remaining Work

These items are **required before going to production** with multi-tenancy.

### 2.1 Backend Plan Limit Enforcement Middleware
**Status:** Feature registry exists, but NO backend middleware enforces limits.
**Impact:** Tenants on STARTER plan can create unlimited products/users/locations.

**What to build:**
- `enforcePlanLimits` middleware that checks current counts against `PlanLimit` table
- Apply to creation endpoints: `POST /products`, `POST /users`, `POST /locations`, `POST /members`
- Return `403` with upgrade message when limit is reached

```
Example:
  STARTER plan: max 100 products
  Tenant has 100 products → POST /products returns 403:
  { "message": "Product limit reached", "limit": 100, "plan": "STARTER", "upgradeRequired": true }
```

**Estimated effort:** 1-2 days

---

### 2.2 Backend Feature Gating Middleware
**Status:** Feature registry exists in shared package, NOT enforced on backend.
**Impact:** STARTER tenants can access bulk upload, analytics, promo management.

**What to build:**
- `requireFeature(Feature.BULK_UPLOAD_PRODUCTS)` middleware
- Apply to specific routes that require higher-tier plans
- Return `403` with feature name and required plan

**Routes to protect:**
| Route | Feature | Min Plan |
|---|---|---|
| `POST /bulk/upload/products` | BULK_UPLOAD_PRODUCTS | PROFESSIONAL |
| `POST /bulk/upload/sales` | BULK_UPLOAD_SALES | PROFESSIONAL |
| `GET /analytics/*` | ANALYTICS_BASIC | PROFESSIONAL |
| `POST /promos` | PROMO_MANAGEMENT | PROFESSIONAL |
| `GET /audit-logs` | AUDIT_LOGS | ENTERPRISE |

**Estimated effort:** 1 day

---

### 2.3 Frontend Feature Gating UI
**Status:** Feature registry is in shared package but frontend doesn't use it.
**Impact:** Users see features they can't use; no upgrade prompts.

**What to build:**
- `useFeatureGate(feature)` hook that checks current plan against feature registry
- `<FeatureGate feature={Feature.BULK_UPLOAD}>` wrapper component
- Upgrade prompt component: "Upgrade to Professional to unlock this feature"
- Disable/hide buttons, show lock icons on gated features
- Sidebar items with lock badges for unavailable features

**Estimated effort:** 2-3 days

---

### 2.4 Tenant Onboarding Flow
**Status:** Tenants can only be created by platform admin via API.
**Impact:** No self-service registration.

**What to build:**
- Public registration page: org name, slug, admin email, password
- Automatic STARTER + TRIAL setup
- Welcome email (or welcome screen)
- Initial setup wizard (company info, first location, first category)

**Estimated effort:** 3-5 days

---

### 2.5 Payment Integration (eSewa / Khalti)
**Status:** Schema supports `TenantPayment` and `Subscription` models, but no integration.
**Impact:** Cannot collect payments; all upgrades must be manual.

**What to build:**
- Payment initiation endpoint (generates eSewa/Khalti redirect URL)
- Payment callback/webhook handler (verifies payment, activates plan)
- Manual payment recording (for bank transfers)
- Subscription renewal reminder system
- Payment history page for tenant admins
- Platform admin: view all payments, mark manual payments

**Estimated effort:** 5-7 days

---

### 2.6 Subscription Expiry Handling
**Status:** `planExpiresAt` and `trialEndsAt` exist in schema, but no cron/scheduled job.
**Impact:** Expired subscriptions stay "ACTIVE" forever.

**What to build:**
- Scheduled job (cron) that runs daily:
  - Check `trialEndsAt` for TRIAL tenants → move to SUSPENDED
  - Check `planExpiresAt` for ACTIVE tenants → move to PAST_DUE
  - Check PAST_DUE grace period (e.g., 7 days) → move to SUSPENDED
  - Check SUSPENDED grace period (e.g., 30 days) → move to LOCKED
- Email/notification for upcoming expirations
- Frontend banner: "Your trial expires in X days"

**Estimated effort:** 2-3 days

---

## 3. High Priority Polish

These significantly improve the user experience and system reliability.

### 3.1 Frontend Subscription Status Banners
**What:** Show contextual banners based on subscription status headers already sent by backend.

- **TRIAL**: "You're on a 14-day free trial. X days remaining. [Upgrade now]"
- **PAST_DUE**: "Your payment is overdue. Please pay to avoid service interruption. [Pay now]"
- **SUSPENDED**: "Your account is suspended. Data is read-only. [Pay to reactivate]"
- **Near expiry**: "Your plan expires in X days. [Renew now]"

**Location:** Global banner below top bar, reads from `X-Subscription-Status` and `X-Trial-Days-Left` response headers.

**Estimated effort:** 1 day

---

### 3.2 Platform Admin Dashboard
**What:** Build a proper UI for the platform admin to manage tenants.

Currently, platform admin routes exist (`/platform/tenants`) but there's no frontend for them. The platform admin logs in but sees a regular dashboard.

**What to build:**
- Tenant list page (with search, filter by plan/status)
- Tenant detail page (users, stats, subscription info)
- Create tenant form
- Change plan / activate / deactivate actions
- Platform-wide analytics (total tenants, revenue, MRR, churn)

**Estimated effort:** 3-5 days

---

### 3.3 Login Page Organization Selector -- Dynamic
**What:** Currently the login form has hardcoded org options. Should fetch available orgs or accept a slug input.

**Options:**
1. **Subdomain-based**: `asha-boutique.yourims.com` → auto-resolves tenant
2. **Slug input field**: User types their org slug
3. **API-based dropdown**: `GET /public/tenants` returns list of active tenant slugs

For production, subdomain-based is cleanest. For now, a text input or API-based dropdown is practical.

**Estimated effort:** 0.5-1 day

---

### 3.4 Proper Error Handling for Tenant/Subscription Errors
**What:** When subscription middleware blocks a request (SUSPENDED/LOCKED), the frontend should show a meaningful page, not a generic error.

**What to build:**
- Axios response interceptor that checks for 403 + subscription headers
- Dedicated "Account Suspended" page with payment CTA
- Dedicated "Account Locked" page with support contact
- Toast notifications for plan limit errors

**Estimated effort:** 1 day

---

### 3.5 Data Export with Tenant Context
**What:** Verify all bulk export operations (products, members, sales) respect tenant scoping.

**Check:**
- Excel exports contain only the current tenant's data
- Downloaded templates are tenant-neutral (no IDs leaking)
- Bulk upload creates records within the correct tenant

**Estimated effort:** 0.5 day (audit + fix if needed)

---

## 4. Medium Priority Enhancements

### 4.1 Tenant Settings Page
**What:** Allow tenant admins to manage their organization settings.

- Organization name, logo, contact info
- Business settings (currency, tax rate, receipt format)
- Notification preferences
- View current plan & usage (X/100 products used)

**Estimated effort:** 2-3 days

---

### 4.2 Usage Dashboard for Tenant Admins
**What:** Show plan usage metrics so admins know when they're approaching limits.

```
Products:    45 / 100  ████████░░ 45%
Users:        2 / 3    █████████░ 67%
Locations:    1 / 2    █████░░░░░ 50%
Members:     12 / 500  █░░░░░░░░░  2%
```

**Estimated effort:** 1-2 days

---

### 4.3 Audit Log Enrichment
**What:** Audit logs exist but may not capture all multi-tenant actions.

**Ensure logging for:**
- Tenant creation / plan change (platform admin)
- Subscription status changes
- Payment events
- Plan limit reached events
- Cross-tenant access attempts (should be zero, but log for security)

**Estimated effort:** 1 day

---

### 4.4 Rate Limiting
**What:** Per-tenant API rate limiting to prevent one tenant from consuming all resources.

**Implementation:**
- Rate limit by `tenantId` (not just IP)
- Different limits per plan tier
- STARTER: 100 req/min, PROFESSIONAL: 500 req/min, ENTERPRISE: 2000 req/min
- Return `429 Too Many Requests` with retry-after header

**Estimated effort:** 1 day

---

### 4.5 Multi-Tenant Backup Strategy
**What:** Ability to export/backup a single tenant's data.

**Use cases:**
- Tenant wants their data export (GDPR-like)
- Platform admin needs to backup a tenant before plan changes
- Disaster recovery per tenant

**Estimated effort:** 2-3 days

---

### 4.6 Invitation System
**What:** Allow tenant admins to invite users via email/link instead of creating passwords manually.

**Flow:**
1. Admin enters email and role
2. System sends invite link
3. New user clicks link, sets password
4. User is added to the tenant

**Estimated effort:** 2-3 days

---

## 5. Low Priority / Nice-to-Have

### 5.1 Subdomain-Based Tenant Routing
Replace header-based tenant identification with subdomains:
- `default.ims.com` → Default Organization
- `asha-boutique.ims.com` → Asha Boutique
- Requires DNS wildcard + reverse proxy config

### 5.2 Tenant Branding / White-Label
Allow each tenant to customize:
- Logo, primary color, accent color
- Custom domain mapping
- Receipt/invoice branding

### 5.3 Inter-Tenant Marketplace
Allow ENTERPRISE tenants to list products visible to other tenants (supplier network).

### 5.4 Mobile App / PWA
Progressive Web App support for mobile POS usage.

### 5.5 Real-time Notifications
WebSocket-based notifications for:
- Low stock alerts
- Transfer status changes
- Payment reminders

### 5.6 Two-Factor Authentication
Add 2FA support for admin and super admin roles.

### 5.7 API Key Management (ENTERPRISE)
Allow ENTERPRISE tenants to generate API keys for external integrations.

---

## 6. Technical Debt

### 6.1 Test Coverage
**Current:** Zero automated tests.
**Need:**
- Unit tests for Prisma auto-scoping logic
- Integration tests for tenant isolation (expand on `test-tenants.sh`)
- E2E tests for critical flows (login → create sale → verify isolation)
- API endpoint tests for all routes

**Estimated effort:** 5-10 days (ongoing)

---

### 6.2 findUnique vs findFirst Audit
**Current:** Many `findUnique` calls were changed to `findFirst` for auto-scoping. This works but `findFirst` doesn't guarantee uniqueness.
**Improvement:** For critical lookups, use compound unique `findUnique` with explicit `tenantId`:

```typescript
// Current (works via auto-scoping but no uniqueness guarantee)
const user = await prisma.user.findFirst({ where: { username } });

// Better (explicit compound unique, guaranteed single result)
const user = await prisma.user.findUnique({
  where: { tenantId_username: { tenantId: req.user!.tenantId, username } }
});
```

**Estimated effort:** 1-2 days

---

### 6.3 Error Response Standardization
**Current:** Some controllers return `{ message }`, others return `{ message, data }`, analytics return different formats.
**Need:** Consistent response envelope:

```json
{
  "success": true,
  "message": "Products fetched",
  "data": [...],
  "pagination": { "page": 1, "limit": 10, "total": 45 }
}
```

**Estimated effort:** 2-3 days

---

### 6.4 Database Indexing
**Current:** Prisma auto-creates indexes for unique constraints and relations.
**Need:** Performance indexes for common queries:

```sql
-- Frequently filtered by tenant + date
CREATE INDEX idx_sales_tenant_date ON sales(tenant_id, created_at);
CREATE INDEX idx_products_tenant_category ON products(tenant_id, category_id);
CREATE INDEX idx_audit_tenant_date ON audit_logs(tenant_id, created_at);
```

**Estimated effort:** 0.5 day

---

### 6.5 Migration File Cleanup
**Current:** Migrations may be out of sync due to `db push` usage in development.
**Need:** Generate a clean, linear migration history before production:

```bash
npx prisma migrate diff --from-empty --to-schema-datamodel prisma/schema.prisma --script > migration.sql
```

**Estimated effort:** 0.5 day

---

### 6.6 Environment Variable Validation
**Current:** Partial validation via `config/env.ts`.
**Need:** Validate ALL required env vars at startup with clear error messages. Use `zod` for env schema validation.

**Estimated effort:** 0.5 day

---

### 6.7 Logging Improvements
**Current:** Basic structured logging.
**Need:**
- Add `tenantId` to all log entries automatically
- Log tenant context for debugging
- Structured log format suitable for log aggregation (ELK, Datadog)

**Estimated effort:** 1 day

---

## 7. Suggested Implementation Order

### Phase 2: Plan Enforcement (1-2 weeks)
Priority: **Must-have before production multi-tenancy**

| # | Task | Effort | Section |
|---|---|---|---|
| 1 | Backend plan limit enforcement middleware | 1-2 days | 2.1 |
| 2 | Backend feature gating middleware | 1 day | 2.2 |
| 3 | Frontend feature gating (hook + components) | 2-3 days | 2.3 |
| 4 | Subscription status banners in frontend | 1 day | 3.1 |
| 5 | Subscription expiry cron job | 2-3 days | 2.6 |
| 6 | Error handling for 403 subscription/limit errors | 1 day | 3.4 |

### Phase 3: Onboarding & Payments (2-3 weeks)

| # | Task | Effort | Section |
|---|---|---|---|
| 7 | Tenant self-registration page | 3-5 days | 2.4 |
| 8 | eSewa / Khalti payment integration | 5-7 days | 2.5 |
| 9 | Platform admin dashboard UI | 3-5 days | 3.2 |
| 10 | Dynamic login org selector | 0.5-1 day | 3.3 |

### Phase 4: Polish & Hardening (2-3 weeks)

| # | Task | Effort | Section |
|---|---|---|---|
| 11 | Tenant settings page | 2-3 days | 4.1 |
| 12 | Usage dashboard for admins | 1-2 days | 4.2 |
| 13 | findUnique audit + fix | 1-2 days | 6.2 |
| 14 | Database indexing | 0.5 day | 6.4 |
| 15 | Migration file cleanup | 0.5 day | 6.5 |
| 16 | Rate limiting | 1 day | 4.4 |
| 17 | Audit log enrichment | 1 day | 4.3 |

### Phase 5: Testing & Reliability (ongoing)

| # | Task | Effort | Section |
|---|---|---|---|
| 18 | Integration tests for isolation | 3-5 days | 6.1 |
| 19 | E2E tests for critical flows | 3-5 days | 6.1 |
| 20 | Error response standardization | 2-3 days | 6.3 |
| 21 | Logging improvements | 1 day | 6.7 |

---

### Total Estimated Effort

| Phase | Duration | Priority |
|---|---|---|
| Phase 2: Plan Enforcement | 1-2 weeks | CRITICAL |
| Phase 3: Onboarding & Payments | 2-3 weeks | HIGH |
| Phase 4: Polish & Hardening | 2-3 weeks | MEDIUM |
| Phase 5: Testing & Reliability | Ongoing | HIGH |
| **Total to production-ready** | **~6-8 weeks** | |
