Multi-Tenant IMS - Architecture Plan & PRD



1. Architecture Decision: Why Shared Database, Not Instance-Per-Tenant

The two options analyzed

Option A: Instance-per-tenant (separate Docker stack per tenant)





Zero code changes to existing app



Complete data isolation



BUT: Each tenant needs ~512MB+ RAM (Postgres + API + Web), so 100 tenants = 50GB+ RAM



Rolling updates are a nightmare (update N instances)



No central management dashboard without building a separate system



Need Kubernetes or similar orchestration



Backups, monitoring, logging all multiplied by N



Verdict: Simple to start, impossible to scale. Avoid.

Option B: Shared database with tenantId column (RECOMMENDED)





Single deployment serves all tenants



10 or 10,000 tenants = same infrastructure, same update cycle



Industry standard: Slack, Notion, Shopify, Figma all use this



Central management dashboard is trivial



Prisma middleware can auto-inject tenant filtering (safety net)



Postgres Row-Level Security (RLS) as second safety layer



Verdict: More upfront work, but the only viable SaaS architecture.

Decision: Shared Database with tenantId



2. Tenant Routing: Subdomain-Based

**Recommendation: {tenant}.yourdomain.com**





Example: acme.ims.app, globex.ims.app



Professional SaaS standard



Doesn't conflict with the existing [workspace] URL pattern (which handles role-based nav like /admin/sales)



Tenant resolved from subdomain in middleware, not URL path



Platform admin portal lives at app.ims.app or admin.ims.app

How it works with current architecture:

Current:  ims.app/admin/sales
                  ↑ workspace (role-based)

Multi-tenant:  acme.ims.app/admin/sales
               ↑ tenant      ↑ workspace (role-based)

The [workspace] dynamic route stays as-is. Tenant is extracted from the subdomain in Next.js middleware and the API auth layer.



3. New Database Schema

New Models to Add

Tenant (Organization)

model Tenant {
  id            String       @id @default(uuid())
  name          String       // Display name: "Acme Corp"
  slug          String       @unique // URL slug: "acme" -> acme.ims.app
  plan          PlanTier     @default(STARTER)
  planExpiresAt DateTime?
  isActive      Boolean      @default(true)
  settings      Json?        // Tenant-specific config
  createdAt     DateTime     @default(now())
  updatedAt     DateTime     @updatedAt

  // Relations
  users         User[]
  products      Product[]
  categories    Category[]
  vendors       Vendor[]
  locations     Location[]
  members       Member[]
  sales         Sale[]
  transfers     Transfer[]
  promoCodes    PromoCode[]
  discountTypes DiscountType[]
  auditLogs     AuditLog[]
  errorReports  ErrorReport[]
}

PlanTier Enum

enum PlanTier {
  STARTER
  PROFESSIONAL
  ENTERPRISE
}

PlanLimit (configurable limits per tier)

model PlanLimit {
  id              String   @id @default(uuid())
  tier            PlanTier @unique
  maxUsers        Int      // STARTER: 3, PRO: 10, ENTERPRISE: unlimited (-1)
  maxProducts     Int      // STARTER: 100, PRO: 1000, ENTERPRISE: -1
  maxLocations    Int      // STARTER: 2, PRO: 10, ENTERPRISE: -1
  maxMembers      Int      // STARTER: 500, PRO: 5000, ENTERPRISE: -1
  bulkUpload      Boolean  // STARTER: false, PRO: true, ENTERPRISE: true
  analytics       Boolean  // STARTER: false (basic only), PRO: true, ENTERPRISE: true
  promoManagement Boolean  // STARTER: false, PRO: true, ENTERPRISE: true
  auditLogs       Boolean  // STARTER: false, PRO: false, ENTERPRISE: true
  apiAccess       Boolean  // STARTER: false, PRO: false, ENTERPRISE: true
  createdAt       DateTime @default(now())
  updatedAt       DateTime @updatedAt
}

Platform Admin (super-level above tenants)

enum UserRole {
  platformAdmin  // NEW: manages the entire platform across tenants
  superAdmin     // Existing: tenant-level super admin
  admin          // Existing: tenant-level admin
  user           // Existing: tenant-level user
}

Modified Existing Models

Every existing model gets a tenantId foreign key:

model User {
  id          String   @id @default(uuid())
  tenantId    String                          // <-- NEW
  tenant      Tenant   @relation(...)         // <-- NEW
  username    String
  password    String
  role        UserRole
  // ... rest stays the same

  @@unique([tenantId, username])              // <-- username unique per tenant, not globally
}

model Product {
  id          String   @id @default(uuid())
  tenantId    String                          // <-- NEW
  tenant      Tenant   @relation(...)         // <-- NEW
  imsCode     String
  // ... rest stays the same

  @@unique([tenantId, imsCode])              // <-- imsCode unique per tenant
}

Pattern applies to ALL models: Category, SubCategory, Vendor, Location, LocationInventory, Product, ProductVariation, ProductSubVariation, Sale, SaleItem, SalePayment, Member, Transfer, TransferItem, TransferLog, PromoCode, DiscountType, ProductDiscount, AuditLog, ErrorReport.

Unique constraints change:





Category.name -> @@unique([tenantId, name])



Vendor.name -> @@unique([tenantId, name])



Location.name -> @@unique([tenantId, name])



Member.phone -> @@unique([tenantId, phone])



PromoCode.code -> @@unique([tenantId, code])



Sale.saleCode -> @@unique([tenantId, saleCode])



Transfer.transferCode -> @@unique([tenantId, transferCode])



DiscountType.name -> @@unique([tenantId, name])



4. Backend Changes

4.1 Tenant Resolution Middleware (NEW)

A new middleware that runs on every request (after auth) to resolve and inject the tenant:

// apps/api/src/middlewares/tenantMiddleware.ts

const resolveTenant = async (req, res, next) => {
  // Platform admins bypass tenant resolution
  if (req.user?.role === "platformAdmin") return next();

  const tenantId = req.user?.tenantId;
  if (!tenantId) return res.status(403).json({ error: "No tenant" });

  const tenant = await prisma.tenant.findUnique({ where: { id: tenantId } });
  if (!tenant || !tenant.isActive)
    return res.status(403).json({ error: "Tenant inactive" });

  req.tenant = tenant;
  next();
};

4.2 Prisma Middleware for Auto-Filtering (Safety Net)

Use Prisma client extensions to automatically inject tenantId into every query:

// apps/api/src/config/prisma.ts

const prisma = new PrismaClient().$extends({
  query: {
    $allModels: {
      async findMany({ args, query }) {
        // Auto-inject tenantId filter from async local storage context
        const tenantId = getTenantContext();
        if (tenantId) {
          args.where = { ...args.where, tenantId };
        }
        return query(args);
      },
      async create({ args, query }) {
        const tenantId = getTenantContext();
        if (tenantId) {
          args.data = { ...args.data, tenantId };
        }
        return query(args);
      },
      // Same for update, delete, findFirst, findUnique, count, aggregate
    },
  },
});

This uses Node.js AsyncLocalStorage to pass the tenant context through the request lifecycle without modifying every service function signature.

4.3 Plan Enforcement Middleware (NEW)

// apps/api/src/middlewares/planMiddleware.ts

const enforcePlanLimits = (resource: string) => async (req, res, next) => {
  const tenant = req.tenant;
  const limits = await getPlanLimits(tenant.plan);

  // Check resource-specific limits
  if (resource === "users") {
    const count = await prisma.user.count({ where: { tenantId: tenant.id } });
    if (limits.maxUsers !== -1 && count >= limits.maxUsers) {
      return res
        .status(403)
        .json({ error: "Plan limit reached", upgrade: true });
    }
  }
  // Similar for products, locations, members...
  next();
};

4.4 JWT Token Changes

Current JWT payload: { id, username, role }
New JWT payload: { id, username, role, tenantId, tenantSlug }

This is extracted in apps/api/src/modules/auth/auth.controller.ts:

const token = jwt.sign(
  {
    id: user.id,
    username: user.username,
    role: user.role,
    tenantId: user.tenantId,
    tenantSlug: user.tenant.slug,
  },
  env.jwtSecret,
  { expiresIn: "24h" },
);

4.5 Service Layer Changes

Every service file needs tenant-scoping. With the Prisma middleware approach above, most queries are auto-scoped. But explicit tenant-awareness is needed for:





Cross-tenant operations (platform admin)



Seed data per tenant



Tenant onboarding (creating initial data)

4.6 New API Routes

POST   /api/v1/platform/tenants          -- Create tenant (platformAdmin)
GET    /api/v1/platform/tenants          -- List tenants (platformAdmin)
GET    /api/v1/platform/tenants/:id      -- Get tenant details (platformAdmin)
PUT    /api/v1/platform/tenants/:id      -- Update tenant (platformAdmin)
PATCH  /api/v1/platform/tenants/:id/plan -- Change plan (platformAdmin)
DELETE /api/v1/platform/tenants/:id      -- Deactivate tenant (platformAdmin)
GET    /api/v1/platform/stats            -- Platform-wide stats (platformAdmin)
GET    /api/v1/tenant/info               -- Current tenant info (any authenticated)
GET    /api/v1/tenant/plan               -- Current plan & usage (superAdmin)



5. Frontend Changes

5.1 Subdomain Resolution

Update apps/web/middleware.ts to extract tenant from subdomain:

export function middleware(request: NextRequest) {
  const hostname = request.headers.get("host") || "";
  const tenantSlug = hostname.split(".")[0]; // "acme" from "acme.ims.app"

  // Validate tenant exists (cache this)
  // Pass tenant slug to the app via headers or cookies
  const response = NextResponse.next();
  response.headers.set("x-tenant-slug", tenantSlug);
  return response;
}

5.2 Auth Store Changes

Update apps/web/stores/auth-store.ts to store tenant info:

interface AuthState {
  user: AuthUser | null;
  token: string | null;
  tenant: { id: string; slug: string; name: string; plan: PlanTier } | null; // NEW
  // ...
}

5.3 Plan-Aware UI Components

// apps/web/hooks/usePlan.ts
export function usePlan() {
  const tenant = useAuthStore((s) => s.tenant);
  return {
    plan: tenant?.plan,
    canUseBulkUpload: tenant?.plan !== "STARTER",
    canUseAnalytics: tenant?.plan !== "STARTER",
    canUsePromos: tenant?.plan !== "STARTER",
    canUseAuditLogs: tenant?.plan === "ENTERPRISE",
  };
}

Sidebar items in apps/web/components/layout/sidebar.tsx become plan-aware: gated features show a lock icon with "Upgrade" tooltip.

5.4 Platform Admin Dashboard (NEW)

A separate route group or even a separate Next.js app for the platform admin to:





View all tenants



Create/manage tenants



Change plans



View platform-wide analytics



Manage platform admins



6. Three-Tier Plan Structure







Capability



Starter



Professional



Enterprise





Users



3



10



Unlimited





Products



100



1,000



Unlimited





Locations



2



10



Unlimited





Members



500



5,000



Unlimited





Sales



Unlimited



Unlimited



Unlimited





Transfers



Yes



Yes



Yes





Bulk Upload



No



Yes



Yes





Advanced Analytics



No (basic dashboard only)



Yes



Yes





Promo Code Management



No



Yes



Yes





Audit Logs



No



No



Yes





API Access



No



No



Yes





Custom Branding



No



No



Yes





Priority Support



No



No



Yes

These are stored in the PlanLimit database table so they can be adjusted without code deployments.



7. Subscription & Billing System

7.1 Context: Nepal Payment Ecosystem

Nepal does not have auto-recurring payment infrastructure. Payment gateways (eSewa, Khalti, FonePay, ConnectIPS) only support one-time payments with server-to-server verification callbacks. This means:





No auto-debit or recurring charge capability



Tenant must manually initiate payment each billing cycle



System must track subscription expiry and prompt renewal



Platform admin needs ability to manually extend subscriptions (for bank transfers, cheques, etc.)

7.2 Subscription Model: Pre-Paid with Expiry

Tenant signs up (or is created by platform admin)
  -> Starts with a FREE TRIAL (14 days, STARTER features)
  -> Before trial ends, system shows renewal banners
  -> Tenant chooses plan + billing cycle (monthly/annual)
  -> Tenant pays via eSewa/Khalti (one-time payment)
  -> Payment verified via gateway callback
  -> Subscription activated for the paid period
  -> Near expiry, system shows renewal warnings
  -> Tenant pays again to renew
  -> If not renewed, enters GRACE PERIOD (7 days)
  -> After grace period, account is SUSPENDED (read-only)
  -> After 30 more days, account is LOCKED (no access)

7.3 New Database Models

Subscription

model Subscription {
  id              String             @id @default(uuid())
  tenantId        String
  tenant          Tenant             @relation(fields: [tenantId], references: [id])
  plan            PlanTier
  billingCycle    BillingCycle
  status          SubscriptionStatus @default(TRIAL)
  currentPeriodStart DateTime
  currentPeriodEnd   DateTime
  trialEndsAt     DateTime?
  gracePeriodEnd  DateTime?          // 7 days after expiry
  cancelledAt     DateTime?
  createdAt       DateTime           @default(now())
  updatedAt       DateTime           @updatedAt

  payments        Payment[]
}

enum BillingCycle {
  MONTHLY
  ANNUAL
}

enum SubscriptionStatus {
  TRIAL          // Free trial period (14 days)
  ACTIVE         // Paid and current
  PAST_DUE       // Payment period ended, within grace period (7 days)
  SUSPENDED      // Grace period ended, read-only access (30 days)
  LOCKED         // 30 days after suspension, no access at all
  CANCELLED      // Manually cancelled by admin
}

Payment

model Payment {
  id              String        @id @default(uuid())
  tenantId        String
  tenant          Tenant        @relation(fields: [tenantId], references: [id])
  subscriptionId  String
  subscription    Subscription  @relation(fields: [subscriptionId], references: [id])
  amount          Float
  currency        String        @default("NPR")
  gateway         PaymentGateway
  gatewayTxnId    String?       // Transaction ID from eSewa/Khalti
  gatewayResponse Json?         // Full response from payment gateway
  status          PaymentStatus @default(PENDING)
  paidFor         PlanTier      // Which plan this payment is for
  billingCycle    BillingCycle  // Monthly or annual
  periodStart     DateTime      // What period this payment covers
  periodEnd       DateTime
  verifiedAt      DateTime?
  verifiedBy      String?       // Platform admin ID (for manual verification)
  notes           String?       // For manual payments: "Bank transfer ref #123"
  createdAt       DateTime      @default(now())
  updatedAt       DateTime      @updatedAt
}

enum PaymentGateway {
  ESEWA
  KHALTI
  FONEPAY
  CONNECTIPS
  BANK_TRANSFER   // Manual/offline payment
  MANUAL          // Platform admin manually credits
}

enum PaymentStatus {
  PENDING         // Payment initiated but not verified
  COMPLETED       // Payment verified successfully
  FAILED          // Payment verification failed
  REFUNDED        // Payment was refunded
}

PricingPlan (platform-configurable pricing)

model PricingPlan {
  id            String       @id @default(uuid())
  tier          PlanTier
  billingCycle  BillingCycle
  price         Float        // Price in NPR
  originalPrice Float?       // Strike-through price (for annual discount display)
  isActive      Boolean      @default(true)
  createdAt     DateTime     @default(now())
  updatedAt     DateTime     @updatedAt

  @@unique([tier, billingCycle])
}

7.4 Updated Tenant Model

model Tenant {
  id            String             @id @default(uuid())
  name          String
  slug          String             @unique
  plan          PlanTier           @default(STARTER)
  planExpiresAt DateTime?
  isActive      Boolean            @default(true)
  isTrial       Boolean            @default(true)
  trialEndsAt   DateTime?
  subscriptionStatus SubscriptionStatus @default(TRIAL)
  settings      Json?
  createdAt     DateTime           @default(now())
  updatedAt     DateTime           @updatedAt

  // Relations
  subscriptions Subscription[]
  payments      Payment[]
  // ... all existing relations
}

7.5 Subscription Lifecycle & State Machine

                        ┌──────────────────────────────────────┐
                        │                                      │
  ┌───────┐   pay    ┌─┴────┐   period    ┌──────────┐       │
  │ TRIAL ├────────► │ACTIVE├───expires──► │ PAST_DUE │       │
  └───┬───┘          └─┬────┘             └────┬──────┘       │
      │                │  ▲                    │              │
      │ expires        │  │ pay                │ 7 days       │
      │ (no pay)       │  │                    ▼              │
      │                │  │              ┌───────────┐        │
      └────────────────┘  └──────────────┤ SUSPENDED │        │
                          (renew)        └─────┬─────┘        │
                                               │              │
                                               │ 30 days      │
                                               ▼              │
                                          ┌────────┐          │
                                          │ LOCKED │──pay───►─┘
                                          └────────┘  (reactivate)

State transitions:





TRIAL -> ACTIVE: Tenant makes first payment



TRIAL -> PAST_DUE: Trial expires without payment (immediate, no grace for trial)



ACTIVE -> PAST_DUE: Paid period ends without renewal



PAST_DUE -> ACTIVE: Tenant renews payment (within 7-day grace)



PAST_DUE -> SUSPENDED: 7 days pass without payment (read-only mode)



SUSPENDED -> ACTIVE: Tenant pays (reactivation)



SUSPENDED -> LOCKED: 30 more days pass (no access, data retained)



LOCKED -> ACTIVE: Tenant pays (platform admin may need to assist)



Any -> CANCELLED: Platform admin manually cancels

7.6 Access Restrictions by Subscription Status







Status



Can Login



Can Read Data



Can Create/Edit



Sees Banner





TRIAL



Yes



Yes



Yes (within plan limits)



"X days left in trial"





ACTIVE



Yes



Yes



Yes (within plan limits)



None (or "Renews in X days" near expiry)





PAST_DUE



Yes



Yes



Yes (within plan limits)



"Payment overdue - renew now" (urgent)





SUSPENDED



Yes



Yes (read-only)



No - all writes blocked



"Account suspended - pay to reactivate" (blocking)





LOCKED



No



No



No



Redirected to payment page





CANCELLED



No



No



No



Contact support

7.7 Payment Integration (eSewa / Khalti)

eSewa Integration Flow:

1. Tenant clicks "Subscribe" or "Renew"
2. Frontend shows plan selection + billing cycle
3. Frontend calculates amount and redirects to eSewa payment page
4. Tenant completes payment on eSewa
5. eSewa redirects back to our success URL with transaction details
6. Backend verifies payment via eSewa's verification API:
   POST https://uat.esewa.com.np/epay/transrec (test)
   POST https://esewa.com.np/epay/transrec (production)
7. If verified: Payment marked COMPLETED, subscription activated/renewed
8. If failed: Payment marked FAILED, tenant notified

Khalti Integration Flow:

1. Tenant clicks "Subscribe" or "Renew"
2. Frontend initiates Khalti payment (web SDK or redirect)
3. Tenant completes payment on Khalti
4. Khalti sends callback to our webhook URL
5. Backend verifies via Khalti's lookup API:
   POST https://khalti.com/api/v2/payment/verify/
6. Same completion logic as eSewa

Manual Payment Flow (Bank Transfer):

1. Tenant contacts platform admin
2. Tenant makes bank transfer / cheque payment
3. Platform admin opens admin portal -> tenant -> payments
4. Platform admin clicks "Record Manual Payment"
5. Enters amount, reference number, notes
6. System creates Payment record (COMPLETED, gateway: BANK_TRANSFER)
7. Subscription activated/renewed

7.8 New API Routes for Subscription

-- Tenant-facing (authenticated tenant users)
GET    /api/v1/subscription                    -- Current subscription status
GET    /api/v1/subscription/plans              -- Available plans and pricing
POST   /api/v1/subscription/initiate           -- Initiate payment (returns gateway redirect URL)
POST   /api/v1/subscription/verify             -- Verify payment after gateway redirect
GET    /api/v1/subscription/payments            -- Payment history
POST   /api/v1/subscription/change-plan        -- Request plan change (upgrade/downgrade)

-- Payment gateway callbacks (public, verified by signature)
POST   /api/v1/webhooks/esewa/callback         -- eSewa payment callback
POST   /api/v1/webhooks/khalti/callback        -- Khalti payment callback

-- Platform admin
GET    /api/v1/platform/subscriptions          -- All subscriptions across tenants
PATCH  /api/v1/platform/tenants/:id/subscription -- Manually adjust subscription
POST   /api/v1/platform/tenants/:id/payments   -- Record manual payment
GET    /api/v1/platform/revenue                -- Revenue dashboard data
GET    /api/v1/platform/revenue/forecast       -- MRR, churn, forecast

7.9 Subscription Middleware (Access Control)

Added to the middleware chain, after tenant resolution:

// apps/api/src/middlewares/subscriptionMiddleware.ts

const checkSubscription = async (req, res, next) => {
  const tenant = req.tenant;
  if (!tenant) return next(); // Platform admin

  const status = tenant.subscriptionStatus;

  // LOCKED or CANCELLED: block everything
  if (status === "LOCKED" || status === "CANCELLED") {
    return res.status(403).json({
      error: "subscription_expired",
      message: "Your subscription has expired. Please renew to continue.",
      renewUrl: `/subscription/renew`,
    });
  }

  // SUSPENDED: allow reads, block writes
  if (status === "SUSPENDED") {
    const isWriteMethod = ["POST", "PUT", "PATCH", "DELETE"].includes(
      req.method,
    );
    const isSubscriptionRoute = req.path.startsWith("/api/v1/subscription");

    if (isWriteMethod && !isSubscriptionRoute) {
      return res.status(403).json({
        error: "subscription_suspended",
        message: "Your account is suspended. Renew to restore full access.",
        renewUrl: `/subscription/renew`,
      });
    }
  }

  // Attach subscription status to request for frontend banner logic
  res.setHeader("X-Subscription-Status", status);

  // For TRIAL and PAST_DUE, add days remaining info
  if (status === "TRIAL" && tenant.trialEndsAt) {
    const daysLeft = Math.ceil((tenant.trialEndsAt - Date.now()) / 86400000);
    res.setHeader("X-Trial-Days-Left", Math.max(0, daysLeft));
  }

  next();
};

7.10 Cron Job: Subscription Status Updater

A scheduled job runs every hour to transition subscription states:

// apps/api/src/jobs/subscriptionCron.ts

// Runs every hour
async function updateSubscriptionStatuses() {
  const now = new Date();

  // TRIAL -> PAST_DUE (trial expired, no payment)
  await prisma.tenant.updateMany({
    where: { subscriptionStatus: "TRIAL", trialEndsAt: { lt: now } },
    data: { subscriptionStatus: "PAST_DUE" },
  });

  // ACTIVE -> PAST_DUE (period ended, no renewal)
  await prisma.tenant.updateMany({
    where: { subscriptionStatus: "ACTIVE", planExpiresAt: { lt: now } },
    data: {
      subscriptionStatus: "PAST_DUE" /* set gracePeriodEnd = now + 7 days */,
    },
  });

  // PAST_DUE -> SUSPENDED (grace period ended)
  const gracePeriodAgo = new Date(now.getTime() - 7 * 86400000);
  await prisma.tenant.updateMany({
    where: {
      subscriptionStatus: "PAST_DUE",
      planExpiresAt: { lt: gracePeriodAgo },
    },
    data: { subscriptionStatus: "SUSPENDED" },
  });

  // SUSPENDED -> LOCKED (30 days after suspension)
  const lockThreshold = new Date(now.getTime() - 30 * 86400000);
  await prisma.tenant.updateMany({
    where: {
      subscriptionStatus: "SUSPENDED",
      planExpiresAt: { lt: lockThreshold },
    },
    data: { subscriptionStatus: "LOCKED" },
  });
}

7.11 Frontend Subscription UI

Renewal/Subscription Page (/subscription):





Shows current plan, status, and expiry date



Plan comparison table with pricing (monthly/annual)



"Upgrade", "Downgrade", "Renew" buttons



Payment gateway selection (eSewa, Khalti)



Payment history table

Persistent Banners (shown across all pages based on status):





Trial: Soft blue banner — "You have X days left in your free trial. [Choose a plan]"



Active (near expiry, <7 days): Yellow banner — "Your subscription renews in X days. [Renew now]"



Past Due: Orange banner — "Your payment is overdue. Renew within X days to avoid suspension. [Pay now]"



Suspended: Red banner (blocking, top of page) — "Your account is suspended. You can view data but cannot make changes. [Reactivate now]"

Plan Selection Component:





Toggle between Monthly / Annual (annual shows discount)



Three cards: Starter, Professional, Enterprise



Current plan highlighted



"Pay with eSewa" / "Pay with Khalti" buttons



Shows price in NPR

7.12 Pricing Structure (Example, Platform Admin Configurable)







Plan



Monthly (NPR)



Annual (NPR)



Annual Savings





Starter



2,000



20,000



17% off (save 4,000)





Professional



5,000



50,000



17% off (save 10,000)





Enterprise



12,000



120,000



17% off (save 24,000)

These prices are stored in the PricingPlan table and editable by the platform admin without code changes.

7.13 Plan Upgrades & Downgrades

Upgrade (e.g., Starter -> Professional):





Immediate effect



Prorated: tenant pays the difference for remaining days at new price



Example: 15 days left on Starter monthly (1,000 NPR remaining value), upgrading to Pro monthly (5,000). Tenant pays 5,000 - 1,000 = 4,000 NPR. New period: 30 days from now.



Alternatively (simpler to start): full price, new period starts immediately

Downgrade (e.g., Professional -> Starter):





Takes effect at end of current billing period



System checks if current usage exceeds new plan limits



If over limit: warn but allow downgrade (data retained, just can't add more)



8. Data Migration Strategy

Phase 1: Schema Migration





Create Tenant and PlanLimit tables



Create a default tenant for existing data ("legacy" or the actual business name)



Add tenantId column (nullable initially) to all existing tables



Backfill tenantId with the default tenant's ID for all existing rows



Make tenantId non-nullable



Update unique constraints to include tenantId



Add database indexes on tenantId for every table

Phase 2: Code Migration

This is the bulk of the work. The Prisma middleware approach minimizes manual query changes, but every controller/service needs review.

Migration order (by dependency):

1. Tenant model + PlanLimit model
2. User model (add tenantId, update auth)
3. Category + Vendor (no deps)
4. Product + Variations (depends on Category, Vendor)
5. Location + Inventory (depends on Product)
6. Member
7. Sale + Payments (depends on Product, Location, Member)
8. Transfer (depends on Location, Product)
9. PromoCode (depends on Product)
10. AuditLog + ErrorReport
11. Dashboard + Analytics (read-only, update queries)



8. Tenant Onboarding Flow

Recommendation: Admin-managed first (simpler), self-service later.

Admin-Managed Flow (Phase 1)

Platform Admin creates tenant in admin portal
  -> System creates Tenant record
  -> System creates default DiscountTypes for tenant
  -> System creates initial superAdmin user for tenant
  -> Platform Admin shares credentials with tenant
  -> Tenant superAdmin logs in at {slug}.ims.app
  -> Tenant superAdmin creates their own users, locations, etc.

Self-Service Flow (Phase 2 - future)

User visits ims.app/signup
  -> Enters org name, email, password
  -> System generates slug, creates Tenant (STARTER plan)
  -> System creates superAdmin user
  -> Redirects to {slug}.ims.app
  -> Onboarding wizard guides setup



9. Infrastructure Changes

DNS & Routing





Wildcard DNS: *.ims.app -> your server IP



Reverse proxy (Nginx/Caddy) routes subdomains to the single Next.js + API deployment



Wildcard SSL certificate (Let's Encrypt supports this)

Docker Compose (single deployment for all tenants)

The existing docker-compose.prod.yml stays mostly the same - one Postgres, one API, one Web. Add:





Nginx/Caddy as reverse proxy (new service)



Redis for caching tenant resolution and plan limits (optional, recommended)

Database Indexes

Critical indexes for performance with tenantId:

CREATE INDEX idx_users_tenant ON "User"("tenantId");
CREATE INDEX idx_products_tenant ON "Product"("tenantId");
CREATE INDEX idx_sales_tenant ON "Sale"("tenantId");
CREATE INDEX idx_members_tenant ON "Member"("tenantId");
CREATE INDEX idx_inventory_tenant ON "LocationInventory"("tenantId");
-- ... for every table



10. Security Considerations





Prisma Middleware auto-injects tenantId into all queries (prevents accidental cross-tenant data access)



Postgres RLS (Row-Level Security) as a second safety layer at the DB level



JWT contains tenantId - validated on every request



Subdomain validation - tenant slug in subdomain must match JWT's tenantSlug



Platform admin routes completely separated from tenant routes



Audit logs scoped per tenant



Rate limiting per tenant (not just per IP)



11. Frontend Feature Locking & Subscription Tier Architecture

This section defines the production-grade architecture for how the frontend handles plan-based feature locking and subscription status. It follows and extends the existing patterns: Zustand stores with selectors, guard components, hooks, and service layers.

11.1 Design Principles





Defense in Depth -- Backend enforces limits via middleware (source of truth). Frontend prevents the attempt (UX optimization). Frontend is never the only line of defense.



Type-Safe Feature Registry -- Every gated feature is a typed enum key. No magic strings anywhere.



Single Source of Truth -- The feature-to-plan mapping is defined ONCE in a shared config, derived everywhere.



Three Layers of Gating -- (a) Route-level guards, (b) Component-level guards, (c) Hook-level programmatic checks.



Graceful Degradation -- Locked features show upgrade prompts, not errors or invisible elements.



Consistent Patterns -- Mirrors the existing AuthGuard / RoleGuard / useAuth patterns exactly.

11.2 Architecture Diagram

┌─────────────────────────────────────────────────────────────┐
│                    API Response Headers                       │
│  X-Subscription-Status: ACTIVE                               │
│  X-Plan-Tier: PROFESSIONAL                                   │
│  X-Trial-Days-Left: 5                                        │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│              Axios Response Interceptor                       │
│  Reads headers -> Updates TenantStore                        │
│  Handles 403 subscription_expired / plan_limit_reached       │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│              TenantStore (Zustand)                            │
│                                                              │
│  tenant: { id, slug, name, plan, subscriptionStatus,         │
│            planExpiresAt, trialEndsAt, usage }               │
│                                                              │
│  Selectors: selectPlan, selectSubscriptionStatus,            │
│             selectIsActive, selectIsSuspended                 │
└──────┬──────────┬──────────┬────────────────────────────────┘
       │          │          │
       ▼          ▼          ▼
┌──────────┐ ┌──────────┐ ┌────────────────────┐
│ usePlan  │ │useSub    │ │FeatureRegistry     │
│ Hook     │ │Hook      │ │(shared config)     │
│          │ │          │ │                    │
│ canUse() │ │status    │ │ Feature -> Plan    │
│ limits() │ │banners   │ │ mapping + limits   │
│ usage()  │ │renewal   │ │ + descriptions     │
└────┬─────┘ └────┬─────┘ └─────────┬──────────┘
     │            │                  │
     ▼            ▼                  ▼
┌─────────────────────────────────────────────────────────────┐
│                   Guard Components                           │
│                                                              │
│  ┌─────────────┐  ┌──────────────┐  ┌───────────────────┐  │
│  │FeatureGuard │  │SubscGuard    │  │ UsageLimitGuard   │  │
│  │             │  │              │  │                   │  │
│  │ Wraps UI    │  │ Blocks       │  │ Shows "X of Y     │  │
│  │ by feature  │  │ writes when  │  │ used" + blocks    │  │
│  │ Shows lock  │  │ suspended    │  │ when at limit     │  │
│  │ + upgrade   │  │ Shows banner │  │                   │  │
│  └─────────────┘  └──────────────┘  └───────────────────┘  │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ PlanGate (inline)                                    │    │
│  │ <PlanGate feature="BULK_UPLOAD">                     │    │
│  │   <BulkUploadButton />                               │    │
│  │ </PlanGate>                                          │    │
│  └─────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘

11.3 Shared Feature Registry (Source of Truth)

This is the SINGLE PLACE where feature-to-plan mapping is defined. It lives in the shared package so both frontend and backend can use it.

// packages/shared/src/config/features.ts

/**
 * Feature Registry — Single source of truth for plan-based feature access.
 * Add new features here. Both frontend guards and backend middleware read from this.
 */

export enum Feature {
  // Product features
  BULK_UPLOAD_PRODUCTS = "BULK_UPLOAD_PRODUCTS",
  BULK_UPLOAD_MEMBERS = "BULK_UPLOAD_MEMBERS",
  BULK_UPLOAD_SALES = "BULK_UPLOAD_SALES",
  DATA_EXPORT = "DATA_EXPORT",

  // Analytics features
  ANALYTICS_SALES = "ANALYTICS_SALES",
  ANALYTICS_INVENTORY = "ANALYTICS_INVENTORY",
  ANALYTICS_CUSTOMERS = "ANALYTICS_CUSTOMERS",
  ANALYTICS_ADVANCED = "ANALYTICS_ADVANCED",

  // Promo & discount
  PROMO_MANAGEMENT = "PROMO_MANAGEMENT",

  // Admin features
  AUDIT_LOGS = "AUDIT_LOGS",
  API_ACCESS = "API_ACCESS",
  CUSTOM_BRANDING = "CUSTOM_BRANDING",

  // Resource creation (for limit-gated features)
  CREATE_USER = "CREATE_USER",
  CREATE_PRODUCT = "CREATE_PRODUCT",
  CREATE_LOCATION = "CREATE_LOCATION",
  CREATE_MEMBER = "CREATE_MEMBER",
}

export enum PlanTier {
  STARTER = "STARTER",
  PROFESSIONAL = "PROFESSIONAL",
  ENTERPRISE = "ENTERPRISE",
}

export interface FeatureDefinition {
  key: Feature;
  label: string;                         // Human-readable name
  description: string;                   // Shown in upgrade prompts
  minimumPlan: PlanTier;                 // Lowest plan that gets this
  upgradeMessage: string;                // CTA text
}

/**
 * The master feature registry.
 * To add a new gated feature:
 *   1. Add enum value to Feature
 *   2. Add definition here
 *   3. Use <FeatureGuard feature={Feature.X}> in frontend
 *   4. Use enforcePlanFeature(Feature.X) in backend middleware
 * That's it.
 */
export const FEATURE_REGISTRY: Record<Feature, FeatureDefinition> = {
  [Feature.BULK_UPLOAD_PRODUCTS]: {
    key: Feature.BULK_UPLOAD_PRODUCTS,
    label: "Bulk Product Upload",
    description: "Import products via CSV/Excel",
    minimumPlan: PlanTier.PROFESSIONAL,
    upgradeMessage: "Upgrade to Professional to import products in bulk",
  },
  [Feature.BULK_UPLOAD_MEMBERS]: {
    key: Feature.BULK_UPLOAD_MEMBERS,
    label: "Bulk Member Upload",
    description: "Import members via CSV/Excel",
    minimumPlan: PlanTier.PROFESSIONAL,
    upgradeMessage: "Upgrade to Professional to import members in bulk",
  },
  [Feature.BULK_UPLOAD_SALES]: {
    key: Feature.BULK_UPLOAD_SALES,
    label: "Bulk Sales Upload",
    description: "Import sales via CSV/Excel",
    minimumPlan: PlanTier.PROFESSIONAL,
    upgradeMessage: "Upgrade to Professional to import sales in bulk",
  },
  [Feature.DATA_EXPORT]: {
    key: Feature.DATA_EXPORT,
    label: "Data Export",
    description: "Export data to CSV/Excel",
    minimumPlan: PlanTier.PROFESSIONAL,
    upgradeMessage: "Upgrade to Professional to export your data",
  },
  [Feature.ANALYTICS_SALES]: {
    key: Feature.ANALYTICS_SALES,
    label: "Sales Analytics",
    description: "Detailed sales & revenue reports",
    minimumPlan: PlanTier.PROFESSIONAL,
    upgradeMessage: "Upgrade to Professional for detailed sales analytics",
  },
  [Feature.ANALYTICS_INVENTORY]: {
    key: Feature.ANALYTICS_INVENTORY,
    label: "Inventory Analytics",
    description: "Inventory & operations reports",
    minimumPlan: PlanTier.PROFESSIONAL,
    upgradeMessage: "Upgrade to Professional for inventory analytics",
  },
  [Feature.ANALYTICS_CUSTOMERS]: {
    key: Feature.ANALYTICS_CUSTOMERS,
    label: "Customer Analytics",
    description: "Customer & promo analytics",
    minimumPlan: PlanTier.PROFESSIONAL,
    upgradeMessage: "Upgrade to Professional for customer analytics",
  },
  [Feature.ANALYTICS_ADVANCED]: {
    key: Feature.ANALYTICS_ADVANCED,
    label: "Advanced Analytics",
    description: "Cohort analysis, payment trends, location comparison",
    minimumPlan: PlanTier.ENTERPRISE,
    upgradeMessage: "Upgrade to Enterprise for advanced analytics",
  },
  [Feature.PROMO_MANAGEMENT]: {
    key: Feature.PROMO_MANAGEMENT,
    label: "Promo Code Management",
    description: "Create and manage promotional codes",
    minimumPlan: PlanTier.PROFESSIONAL,
    upgradeMessage: "Upgrade to Professional to create promo codes",
  },
  [Feature.AUDIT_LOGS]: {
    key: Feature.AUDIT_LOGS,
    label: "Audit Logs",
    description: "Track all user activity",
    minimumPlan: PlanTier.ENTERPRISE,
    upgradeMessage: "Upgrade to Enterprise for full audit trail",
  },
  [Feature.API_ACCESS]: {
    key: Feature.API_ACCESS,
    label: "API Access",
    description: "Programmatic API access",
    minimumPlan: PlanTier.ENTERPRISE,
    upgradeMessage: "Upgrade to Enterprise for API access",
  },
  [Feature.CUSTOM_BRANDING]: {
    key: Feature.CUSTOM_BRANDING,
    label: "Custom Branding",
    description: "Custom logo and branding",
    minimumPlan: PlanTier.ENTERPRISE,
    upgradeMessage: "Upgrade to Enterprise for custom branding",
  },
  [Feature.CREATE_USER]: {
    key: Feature.CREATE_USER,
    label: "Create User",
    description: "Add team members",
    minimumPlan: PlanTier.STARTER,
    upgradeMessage: "Upgrade your plan to add more users",
  },
  [Feature.CREATE_PRODUCT]: {
    key: Feature.CREATE_PRODUCT,
    label: "Create Product",
    description: "Add products to inventory",
    minimumPlan: PlanTier.STARTER,
    upgradeMessage: "Upgrade your plan to add more products",
  },
  [Feature.CREATE_LOCATION]: {
    key: Feature.CREATE_LOCATION,
    label: "Create Location",
    description: "Add warehouses and showrooms",
    minimumPlan: PlanTier.STARTER,
    upgradeMessage: "Upgrade your plan to add more locations",
  },
  [Feature.CREATE_MEMBER]: {
    key: Feature.CREATE_MEMBER,
    label: "Create Member",
    description: "Add customer members",
    minimumPlan: PlanTier.STARTER,
    upgradeMessage: "Upgrade your plan to add more members",
  },
};

/** Plan hierarchy for comparison: higher index = higher plan */
const PLAN_HIERARCHY: PlanTier[] = [
  PlanTier.STARTER,
  PlanTier.PROFESSIONAL,
  PlanTier.ENTERPRISE,
];

/** Check if a plan meets or exceeds the minimum required plan */
export function isPlanSufficient(
  currentPlan: PlanTier,
  minimumPlan: PlanTier
): boolean {
  return PLAN_HIERARCHY.indexOf(currentPlan) >= PLAN_HIERARCHY.indexOf(minimumPlan);
}

/** Check if a specific feature is available on a given plan */
export function isFeatureAvailable(
  feature: Feature,
  plan: PlanTier
): boolean {
  const def = FEATURE_REGISTRY[feature];
  return isPlanSufficient(plan, def.minimumPlan);
}

11.4 Tenant Store (Zustand) — Extends Existing Auth Pattern

Follows the exact same Zustand + selectors pattern as the existing auth-store.ts.

// apps/web/stores/tenant-store.ts

"use client";

import { create } from "zustand";
import { persist, createJSONStorage } from "zustand/middleware";
import type { PlanTier } from "@repo/shared/config/features";

// ============================================
// Types
// ============================================

type SubscriptionStatus =
  | "TRIAL"
  | "ACTIVE"
  | "PAST_DUE"
  | "SUSPENDED"
  | "LOCKED"
  | "CANCELLED";

interface TenantInfo {
  id: string;
  slug: string;
  name: string;
  plan: PlanTier;
  subscriptionStatus: SubscriptionStatus;
  planExpiresAt: string | null;
  trialEndsAt: string | null;
}

interface PlanUsage {
  users: { current: number; limit: number };
  products: { current: number; limit: number };
  locations: { current: number; limit: number };
  members: { current: number; limit: number };
}

interface TenantState {
  // State
  tenant: TenantInfo | null;
  usage: PlanUsage | null;
  isHydrated: boolean;

  // Actions
  setTenant: (tenant: TenantInfo) => void;
  setUsage: (usage: PlanUsage) => void;
  clearTenant: () => void;
  setHydrated: (value: boolean) => void;
}

// ============================================
// Store
// ============================================

export const useTenantStore = create<TenantState>()(
  persist(
    (set) => ({
      tenant: null,
      usage: null,
      isHydrated: false,

      setTenant: (tenant) => set({ tenant }),
      setUsage: (usage) => set({ usage }),
      clearTenant: () => set({ tenant: null, usage: null }),
      setHydrated: (value) => set({ isHydrated: value }),
    }),
    {
      name: "tenant-storage",
      storage: createJSONStorage(() => cookieStorage), // same cookie pattern as auth
      partialize: (state) => ({ tenant: state.tenant }),
      onRehydrateStorage: () => (state) => {
        if (state) state.setHydrated(true);
      },
    },
  ),
);

// ============================================
// Selectors (optimized re-renders)
// ============================================

export const selectTenant = (s: TenantState) => s.tenant;
export const selectPlan = (s: TenantState) => s.tenant?.plan ?? null;
export const selectSubscriptionStatus = (s: TenantState) =>
  s.tenant?.subscriptionStatus ?? null;
export const selectUsage = (s: TenantState) => s.usage;
export const selectIsActive = (s: TenantState) =>
  s.tenant?.subscriptionStatus === "ACTIVE" ||
  s.tenant?.subscriptionStatus === "TRIAL";
export const selectIsSuspended = (s: TenantState) =>
  s.tenant?.subscriptionStatus === "SUSPENDED";
export const selectIsTrial = (s: TenantState) =>
  s.tenant?.subscriptionStatus === "TRIAL";

11.5 Hooks — Following useAuth Pattern

usePlan: Feature access and usage limits

// apps/web/hooks/usePlan.ts

"use client";

import { useCallback } from "react";
import { useQuery } from "@tanstack/react-query";
import {
  Feature,
  FEATURE_REGISTRY,
  isFeatureAvailable,
} from "@repo/shared/config/features";
import {
  useTenantStore,
  selectPlan,
  selectUsage,
} from "@/stores/tenant-store";
import { fetchPlanUsage } from "@/services/subscriptionService";

export const planKeys = {
  all: ["plan"] as const,
  usage: () => [...planKeys.all, "usage"] as const,
};

export function usePlan() {
  const plan = useTenantStore(selectPlan);
  const usage = useTenantStore(selectUsage);
  const setUsage = useTenantStore((s) => s.setUsage);

  // Query: Fetch current usage (products, users, locations, members counts)
  const { data: currentUsage } = useQuery({
    queryKey: planKeys.usage(),
    queryFn: async () => {
      const data = await fetchPlanUsage();
      setUsage(data);
      return data;
    },
    staleTime: 2 * 60 * 1000, // 2 minutes
    enabled: !!plan,
  });

  /** Check if a feature is available on the current plan */
  const canUse = useCallback(
    (feature: Feature): boolean => {
      if (!plan) return false;
      return isFeatureAvailable(feature, plan);
    },
    [plan],
  );

  /** Get the feature definition (for upgrade messages, labels) */
  const getFeature = useCallback((feature: Feature) => {
    return FEATURE_REGISTRY[feature];
  }, []);

  /** Check if a resource limit has been reached */
  const isAtLimit = useCallback(
    (resource: "users" | "products" | "locations" | "members"): boolean => {
      const u = currentUsage ?? usage;
      if (!u) return false;
      const { current, limit } = u[resource];
      return limit !== -1 && current >= limit;
    },
    [currentUsage, usage],
  );

  /** Get usage info for a resource */
  const getUsage = useCallback(
    (resource: "users" | "products" | "locations" | "members") => {
      const u = currentUsage ?? usage;
      if (!u) return { current: 0, limit: 0, isUnlimited: false };
      const { current, limit } = u[resource];
      return { current, limit, isUnlimited: limit === -1 };
    },
    [currentUsage, usage],
  );

  return {
    plan,
    canUse,
    getFeature,
    isAtLimit,
    getUsage,
    usage: currentUsage ?? usage,
  };
}

useSubscription: Subscription status and renewal

// apps/web/hooks/useSubscription.ts

"use client";

import { useMemo } from "react";
import {
  useTenantStore,
  selectSubscriptionStatus,
  selectTenant,
} from "@/stores/tenant-store";

export function useSubscription() {
  const tenant = useTenantStore(selectTenant);
  const status = useTenantStore(selectSubscriptionStatus);

  const statusInfo = useMemo(() => {
    if (!tenant || !status) return null;

    const now = Date.now();
    const expiresAt = tenant.planExpiresAt
      ? new Date(tenant.planExpiresAt).getTime()
      : null;
    const trialEndsAt = tenant.trialEndsAt
      ? new Date(tenant.trialEndsAt).getTime()
      : null;

    const daysUntilExpiry = expiresAt
      ? Math.ceil((expiresAt - now) / 86400000)
      : null;
    const daysUntilTrialEnd = trialEndsAt
      ? Math.ceil((trialEndsAt - now) / 86400000)
      : null;

    return {
      status,
      isActive: status === "ACTIVE" || status === "TRIAL",
      isTrial: status === "TRIAL",
      isPastDue: status === "PAST_DUE",
      isSuspended: status === "SUSPENDED",
      isLocked: status === "LOCKED",
      canWrite: status !== "SUSPENDED" && status !== "LOCKED" && status !== "CANCELLED",
      daysUntilExpiry,
      daysUntilTrialEnd,
      isNearExpiry: daysUntilExpiry !== null && daysUntilExpiry <= 7 && daysUntilExpiry > 0,
      showBanner: status !== "ACTIVE" || (daysUntilExpiry !== null && daysUntilExpiry <= 7),
    };
  }, [tenant, status]);

  return statusInfo;
}

11.6 Guard Components — Following AuthGuard / RoleGuard Pattern

The codebase uses two guard patterns: (a) AuthGuard for route-level redirects, (b) RoleGuard for component-level conditional rendering. The new guards follow the same split.

FeatureGuard: Component-level feature locking (like RoleGuard)

// apps/web/components/guards/feature-guard.tsx

"use client";

import { type ReactNode } from "react";
import { Lock, ArrowUpRight } from "lucide-react";
import { Card, CardHeader, CardTitle, CardDescription } from "@/components/ui/card";
import { Button } from "@/components/ui/button";
import { usePlan } from "@/hooks/usePlan";
import { Feature, FEATURE_REGISTRY } from "@repo/shared/config/features";
import Link from "next/link";

interface FeatureGuardProps {
  /** The feature to check access for */
  feature: Feature;
  /** Content to render if feature is available */
  children: ReactNode;
  /** Optional custom fallback when feature is locked */
  fallback?: ReactNode;
  /**
   * Render mode when locked:
   * - "block": Show upgrade card (default, for page sections)
   * - "hide": Render nothing (for optional UI elements)
   * - "disable": Render children but visually disabled with tooltip
   */
  mode?: "block" | "hide" | "disable";
}

/**
 * FeatureGuard Component
 *
 * Plan-based feature access control. Mirrors RoleGuard pattern.
 * Wraps UI elements and shows upgrade prompts when the tenant's
 * plan doesn't include the feature.
 *
 * @example
 * // Block mode (default) — shows upgrade card
 * <FeatureGuard feature={Feature.BULK_UPLOAD_PRODUCTS}>
 *   <BulkUploadForm />
 * </FeatureGuard>
 *
 * @example
 * // Hide mode — silently removes from UI
 * <FeatureGuard feature={Feature.PROMO_MANAGEMENT} mode="hide">
 *   <PromoSidebarItem />
 * </FeatureGuard>
 *
 * @example
 * // Disable mode — shows but greyed out
 * <FeatureGuard feature={Feature.DATA_EXPORT} mode="disable">
 *   <ExportButton />
 * </FeatureGuard>
 */
export function FeatureGuard({
  feature,
  children,
  fallback,
  mode = "block",
}: FeatureGuardProps) {
  const { canUse } = usePlan();

  if (canUse(feature)) {
    return <>{children}</>;
  }

  // Feature is locked
  const def = FEATURE_REGISTRY[feature];

  if (fallback) return <>{fallback}</>;

  switch (mode) {
    case "hide":
      return null;

    case "disable":
      return (
        <div className="relative opacity-50 pointer-events-none select-none" title={def.upgradeMessage}>
          <div className="absolute inset-0 z-10 flex items-center justify-center">
            <Lock className="h-4 w-4 text-muted-foreground" />
          </div>
          {children}
        </div>
      );

    case "block":
    default:
      return (
        <Card className="border-dashed">
          <CardHeader className="text-center">
            <div className="mx-auto mb-2 flex h-10 w-10 items-center justify-center rounded-full bg-muted">
              <Lock className="h-5 w-5 text-muted-foreground" />
            </div>
            <CardTitle className="text-lg">{def.label}</CardTitle>
            <CardDescription>{def.upgradeMessage}</CardDescription>
            <Button asChild variant="outline" size="sm" className="mt-3 mx-auto">
              <Link href="/subscription">
                Upgrade Plan <ArrowUpRight className="ml-1 h-3 w-3" />
              </Link>
            </Button>
          </CardHeader>
        </Card>
      );
  }
}

UsageLimitGuard: Resource limit gating

// apps/web/components/guards/usage-limit-guard.tsx

"use client";

import { type ReactNode } from "react";
import { usePlan } from "@/hooks/usePlan";
import { AlertTriangle } from "lucide-react";
import { Alert, AlertDescription } from "@/components/ui/alert";
import Link from "next/link";

interface UsageLimitGuardProps {
  /** Which resource to check: "users", "products", "locations", "members" */
  resource: "users" | "products" | "locations" | "members";
  /** Content to render (e.g., a Create button) */
  children: ReactNode;
  /** Optional: show the current usage counter */
  showUsage?: boolean;
}

/**
 * UsageLimitGuard Component
 *
 * Wraps "create" actions and blocks them when the tenant
 * has hit the plan limit for that resource.
 *
 * @example
 * <UsageLimitGuard resource="products" showUsage>
 *   <Button onClick={...}>Add Product</Button>
 * </UsageLimitGuard>
 */
export function UsageLimitGuard({
  resource,
  children,
  showUsage = false,
}: UsageLimitGuardProps) {
  const { isAtLimit, getUsage } = usePlan();
  const { current, limit, isUnlimited } = getUsage(resource);
  const atLimit = isAtLimit(resource);

  return (
    <>
      {showUsage && !isUnlimited && (
        <p className="text-xs text-muted-foreground mb-2">
          {current} of {limit} {resource} used
        </p>
      )}
      {atLimit ? (
        <Alert variant="destructive" className="mb-4">
          <AlertTriangle className="h-4 w-4" />
          <AlertDescription>
            You have reached the {resource} limit for your plan ({current}/{limit}).{" "}
            <Link href="/subscription" className="underline font-medium">
              Upgrade
            </Link>{" "}
            to add more.
          </AlertDescription>
        </Alert>
      ) : (
        children
      )}
    </>
  );
}

SubscriptionGuard: Route-level subscription status gate (like AuthGuard)

// apps/web/components/guards/subscription-guard.tsx

"use client";

import type React from "react";
import { useEffect } from "react";
import { useRouter } from "next/navigation";
import { useSubscription } from "@/hooks/useSubscription";
import { SuspendedOverlay } from "@/components/subscription/suspended-overlay";

interface SubscriptionGuardProps {
  children: React.ReactNode;
  /** If true, blocks writes (shows overlay on suspended) */
  requireWrite?: boolean;
}

/**
 * SubscriptionGuard Component
 *
 * Route-level guard that enforces subscription status.
 * Mirrors AuthGuard pattern: redirects for LOCKED, shows
 * overlay for SUSPENDED.
 *
 * Applied in the workspace layout alongside AuthGuard.
 */
export function SubscriptionGuard({
  children,
  requireWrite = false,
}: SubscriptionGuardProps) {
  const router = useRouter();
  const sub = useSubscription();

  useEffect(() => {
    if (!sub) return;
    if (sub.isLocked) {
      router.push("/subscription/renew");
    }
  }, [sub, router]);

  if (!sub) return <>{children}</>;

  if (sub.isLocked) return null;

  if (sub.isSuspended && requireWrite) {
    return <SuspendedOverlay>{children}</SuspendedOverlay>;
  }

  return <>{children}</>;
}

11.7 Subscription Banner — Persistent Global UI

Lives in the DashboardLayout, always visible based on subscription status.

// apps/web/components/subscription/subscription-banner.tsx

"use client";

import { useSubscription } from "@/hooks/useSubscription";
import { AlertTriangle, Clock, XCircle, Info } from "lucide-react";
import Link from "next/link";
import { cn } from "@/lib/utils";

/**
 * SubscriptionBanner
 *
 * Persistent banner shown in DashboardLayout.
 * Changes color and urgency based on subscription status.
 * Only renders when there is something to communicate.
 */
export function SubscriptionBanner() {
  const sub = useSubscription();
  if (!sub || !sub.showBanner) return null;

  const config = getBannerConfig(sub);
  if (!config) return null;

  return (
    <div className={cn("px-4 py-2.5 text-sm font-medium flex items-center justify-between", config.className)}>
      <div className="flex items-center gap-2">
        {config.icon}
        <span>{config.message}</span>
      </div>
      <Link
        href="/subscription"
        className="underline font-semibold whitespace-nowrap ml-4"
      >
        {config.cta}
      </Link>
    </div>
  );
}

function getBannerConfig(sub: NonNullable<ReturnType<typeof useSubscription>>) {
  if (sub.isTrial) {
    return {
      className: "bg-blue-50 text-blue-800 dark:bg-blue-950 dark:text-blue-200",
      icon: <Info className="h-4 w-4" />,
      message: `You have ${sub.daysUntilTrialEnd ?? 0} days left in your free trial.`,
      cta: "Choose a plan",
    };
  }
  if (sub.isNearExpiry && sub.status === "ACTIVE") {
    return {
      className: "bg-yellow-50 text-yellow-800 dark:bg-yellow-950 dark:text-yellow-200",
      icon: <Clock className="h-4 w-4" />,
      message: `Your subscription expires in ${sub.daysUntilExpiry} days.`,
      cta: "Renew now",
    };
  }
  if (sub.isPastDue) {
    return {
      className: "bg-orange-50 text-orange-800 dark:bg-orange-950 dark:text-orange-200",
      icon: <AlertTriangle className="h-4 w-4" />,
      message: "Your payment is overdue. Renew to avoid service suspension.",
      cta: "Pay now",
    };
  }
  if (sub.isSuspended) {
    return {
      className: "bg-red-50 text-red-800 dark:bg-red-950 dark:text-red-200",
      icon: <XCircle className="h-4 w-4" />,
      message: "Your account is suspended. You can view data but cannot make changes.",
      cta: "Reactivate now",
    };
  }
  return null;
}

11.8 Sidebar Integration — Plan-Gated Nav Items

The existing sidebar already filters by role. We add a second layer for plan gating.

// In apps/web/components/layout/sidebar.tsx — NavItem gets a new optional field:

interface NavItem {
  path: string;
  label: string;
  icon: React.ComponentType<{ className?: string }>;
  roles: UserRole[];
  requiredFeature?: Feature;   // <-- NEW: plan-gated
  children?: NavItem[];
  href?: string;
}

// Example nav items with feature gating:
{
  path: "product/bulk-upload",
  label: "Bulk Upload",
  icon: Upload,
  roles: ["admin", "superAdmin"],
  requiredFeature: Feature.BULK_UPLOAD_PRODUCTS,  // locked on Starter
},
{
  path: "promos",
  label: "Promo Codes",
  icon: Percent,
  roles: ["admin", "superAdmin"],
  requiredFeature: Feature.PROMO_MANAGEMENT,      // locked on Starter
},
{
  path: "settings/logs",
  label: "Audit Logs",
  icon: BarChart3,
  roles: ["superAdmin"],
  requiredFeature: Feature.AUDIT_LOGS,            // Enterprise only
},

// In the render logic, items with locked features show a lock icon:
const isLocked = item.requiredFeature && !canUse(item.requiredFeature);

return (
  <Link
    href={isLocked ? "#" : href}
    onClick={isLocked ? (e) => { e.preventDefault(); showUpgradeDialog(); } : undefined}
    className={cn(
      "flex items-center gap-3 px-3 py-2 text-sm font-medium rounded-md",
      isLocked && "opacity-50 cursor-not-allowed",
    )}
  >
    <Icon className="h-4 w-4 shrink-0" />
    {isOpen && <span>{item.label}</span>}
    {isLocked && <Lock className="h-3 w-3 ml-auto text-muted-foreground" />}
  </Link>
);

11.9 API Error Handling for Plan Limits

Extend the existing handleApiError in apps/web/lib/apiError.ts and the Axios interceptor to handle plan-specific 403 responses:

// In apps/web/lib/axios.ts — add to response interceptor:

api.interceptors.response.use(
  (response) => {
    // Sync subscription status from response headers
    const status = response.headers["x-subscription-status"];
    if (status) {
      useTenantStore.getState().setTenant({
        ...useTenantStore.getState().tenant!,
        subscriptionStatus: status,
      });
    }
    return response;
  },
  (error) => {
    if (error.response?.status === 403) {
      const errorType = error.response?.data?.error;

      // Subscription expired — redirect to renewal
      if (errorType === "subscription_expired" || errorType === "subscription_suspended") {
        const renewUrl = error.response?.data?.renewUrl;
        if (renewUrl && typeof window !== "undefined") {
          window.location.href = renewUrl;
          return Promise.reject(error);
        }
      }

      // Plan limit reached — show upgrade toast
      if (errorType === "plan_limit_reached") {
        toast({
          title: "Plan limit reached",
          description: error.response?.data?.message || "Upgrade your plan to continue.",
          variant: "destructive",
        });
        return Promise.reject(error);
      }
    }

    // existing 401 handling...
    if (error.response?.status === 401) { /* ... existing logic ... */ }

    return Promise.reject(error);
  },
);

11.10 Workspace Layout Integration

The DashboardLayout wires everything together:

// Updated apps/web/components/layout/dashboard-layout.tsx

export function DashboardLayout({ children }: { children: React.ReactNode }) {
  return (
    <div className="flex h-screen overflow-hidden bg-background">
      <Sidebar isOpen={isOpen} onToggle={() => toggle(isMobile)} />
      <div className="flex min-w-0 flex-1 flex-col overflow-hidden">
        <SubscriptionBanner />           {/* <-- NEW: persistent status banner */}
        <TopBar onMenuClick={handleMenuClick} />
        <main className="min-w-0 flex-1 overflow-auto p-4 md:p-6">
          <SubscriptionGuard>           {/* <-- NEW: blocks when LOCKED */}
            {children}
          </SubscriptionGuard>
        </main>
      </div>
    </div>
  );
}

And the workspace layout adds TenantProvider alongside AuthGuard:

// Updated apps/web/app/[workspace]/layout.tsx

export default function WorkspaceLayout({ children }) {
  return (
    <AuthGuard>
      <TenantProvider>                  {/* <-- NEW: fetches & provides tenant context */}
        <DashboardLayout>{children}</DashboardLayout>
      </TenantProvider>
    </AuthGuard>
  );
}

11.11 Summary: How a Developer Adds a New Gated Feature

This is the developer experience for adding a new plan-gated feature. It should be this simple:

Step 1: Add enum value to Feature in packages/shared/src/config/features.ts
Step 2: Add definition to FEATURE_REGISTRY (label, minimumPlan, upgradeMessage)
Step 3: Wrap the frontend UI with <FeatureGuard feature={Feature.MY_FEATURE}>
Step 4: Add enforcePlanFeature(Feature.MY_FEATURE) to the backend route middleware

Done. The feature is now:
  - Hidden/locked in sidebar (if nav item has requiredFeature)
  - Blocked on the page (FeatureGuard shows upgrade card)
  - Blocked on the API (middleware returns 403)
  - Shown in plan comparison table (auto-derived from registry)

11.12 Complete Guard Hierarchy (Defense in Depth)

Request Flow:

1. Next.js Middleware       → Subdomain → Tenant slug resolution
2. AuthGuard (layout)       → Is user logged in? Has valid JWT?
3. TenantProvider (layout)  → Fetch tenant info, plan, usage
4. SubscriptionGuard        → Is subscription active? Block if LOCKED/SUSPENDED
5. SubscriptionBanner       → Show warnings for trial/expiry/past-due
6. RoleGuard (page)         → Does user's role allow this page?
7. FeatureGuard (section)   → Does the tenant's plan include this feature?
8. UsageLimitGuard (action) → Has the tenant hit the resource limit?

Backend mirrors this:
1. verifyToken              → Valid JWT?
2. resolveTenant            → Valid, active tenant?
3. checkSubscription        → Subscription allows this action?
4. authorizeRoles           → User role allowed?
5. enforcePlanFeature       → Plan includes this feature?
6. enforcePlanLimits        → Under resource limit?

Every layer is independent. Removing any single layer still leaves protection from the others.



12. Implementation Phases

Phase 1: Foundation (Weeks 1-2)





Tenant, PlanLimit, Subscription, Payment, PricingPlan models



tenantId on User model



Auth changes (JWT, middleware)



Tenant resolution middleware



Platform admin CRUD for tenants



Migrate existing data to default tenant

Phase 2: Core Tenant Scoping (Weeks 3-4)





Prisma auto-scoping middleware (AsyncLocalStorage)



Add tenantId to all remaining models



Update unique constraints



Update all services (with Prisma middleware doing heavy lifting)



Database migration with backfill

Phase 3: Plan & Subscription Enforcement (Week 5)





Plan enforcement middleware (resource limits)



Subscription status middleware (access control by status)



Subscription status cron job (TRIAL->PAST_DUE->SUSPENDED->LOCKED transitions)



Plan-aware frontend (feature gating, limit warnings)



Usage tracking and limit checks

Phase 4: Subscription Billing (Weeks 6-7)





eSewa payment integration (initiate + verify)



Khalti payment integration (initiate + verify)



Payment webhook endpoints



Manual payment recording (platform admin)



Subscription renewal flow



Plan upgrade/downgrade logic



Pricing plan management (platform admin configurable)



Frontend: plan selection page, payment flow, payment history



Frontend: subscription status banners (trial, expiring, past due, suspended)

Phase 5: Frontend Multi-Tenant (Week 8)





Subdomain resolution in Next.js middleware



Tenant context in auth store



Plan-gated sidebar items



Subscription status UI throughout app



Tenant branding basics (name, logo)

Phase 6: Platform Admin Portal (Week 9)





Tenant management UI



Subscription & payment management



Revenue dashboard (MRR, churn, active tenants, payment history)



Platform-wide analytics



Tenant onboarding flow

Phase 7: Hardening (Week 10)





Postgres RLS policies



Cross-tenant security audit



Performance testing with multiple tenants



Rate limiting per tenant



Cron job reliability (retry logic, error handling)



Documentation



13. Key Files That Will Be Modified

Backend (high-impact):





apps/api/prisma/schema.prisma - Add Tenant, Subscription, Payment, PricingPlan models, tenantId to all models



apps/api/src/config/prisma.ts - Prisma extensions for auto-scoping



apps/api/src/middlewares/authMiddleware.ts - Extract tenantId from JWT



apps/api/src/modules/auth/auth.controller.ts - Include tenantId in JWT, check subscription status



Every service file in apps/api/src/modules/*/ - Tenant-aware queries

Backend (new files):





apps/api/src/middlewares/tenantMiddleware.ts - Tenant resolution



apps/api/src/middlewares/planMiddleware.ts - Plan limit enforcement



apps/api/src/middlewares/subscriptionMiddleware.ts - Subscription status access control



apps/api/src/modules/platform/ (tenant CRUD, platform stats, revenue)



apps/api/src/modules/subscription/ (subscription management, plan changes)



apps/api/src/modules/payment/ (payment initiation, verification, webhooks)



apps/api/src/modules/payment/gateways/esewa.ts - eSewa integration



apps/api/src/modules/payment/gateways/khalti.ts - Khalti integration



apps/api/src/config/tenantContext.ts (AsyncLocalStorage)



apps/api/src/jobs/subscriptionCron.ts - Scheduled subscription status transitions

Frontend (high-impact):





apps/web/middleware.ts - Subdomain resolution



apps/web/stores/auth-store.ts - Tenant + subscription state



apps/web/components/layout/sidebar.tsx - Plan-gated items



apps/web/components/layout/dashboard-layout.tsx - Subscription status banners



apps/web/config/routes.ts - Tenant-aware routing

Frontend (new files):





apps/web/hooks/usePlan.ts - Plan feature flags



apps/web/hooks/useTenant.ts - Tenant context



apps/web/hooks/useSubscription.ts - Subscription status + renewal



apps/web/components/subscription/subscription-banner.tsx - Status banners



apps/web/components/subscription/plan-selector.tsx - Plan comparison + selection



apps/web/components/subscription/payment-gateway.tsx - eSewa/Khalti payment buttons



apps/web/app/[workspace]/subscription/ - Subscription management pages



Platform admin pages (new route group)

Infrastructure:





docker-compose.prod.yml - Add reverse proxy, add cron job service



Nginx/Caddy config for wildcard subdomain routing


