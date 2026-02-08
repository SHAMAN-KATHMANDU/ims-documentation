# IMS - QA Testing Strategy

> **Audience:** Non-technical QA Tester
> **Application:** Inventory Management System (IMS)
> **Version:** 1.0
> **Last Updated:** February 8, 2026

---

## Table of Contents

1. [Testing Philosophy](#1-testing-philosophy)
2. [Understanding the System](#2-understanding-the-system)
3. [Testing Approach](#3-testing-approach)
4. [Test Priorities (Risk-Based)](#4-test-priorities-risk-based)
5. [Module-by-Module Test Areas](#5-module-by-module-test-areas)
6. [Role-Based Testing Matrix](#6-role-based-testing-matrix)
7. [Test Types You Will Perform](#7-test-types-you-will-perform)
8. [Bug Reporting Standards](#8-bug-reporting-standards)
9. [Test Execution Workflow](#9-test-execution-workflow)
10. [Environment & Data Guidelines](#10-environment--data-guidelines)
11. [Glossary](#11-glossary)

---

## 1. Testing Philosophy

### Think Like a User, Break Like a Tester

Your job is NOT just to verify that things work. Your job is to **find problems before real users do**. Always ask yourself:

- "What would happen if I did something unexpected here?"
- "What if the data is wrong, missing, or too long?"
- "What if I do things out of order?"
- "What would a confused or rushed user do?"

### The Golden Rules

1. **Never assume it works** — verify everything yourself
2. **Test one thing at a time** — so you know exactly what caused a problem
3. **Document everything** — if you didn't write it down, it didn't happen
4. **Re-test after fixes** — a fix in one place can break another
5. **Fresh eyes find bugs** — your non-technical perspective is a strength, not a weakness

---

## 2. Understanding the System

### What Is IMS?

IMS is an Inventory Management System for retail/showroom operations. It tracks products, manages inventory across multiple locations (warehouses and showrooms), processes sales, manages customer memberships, and generates analytics reports.

### Three User Roles

| Role | What They Can Do |
|------|-----------------|
| **Super Admin** | Everything — manage users, view audit logs, error reports, system controls |
| **Admin** | Manage products, inventory, sales, transfers, members, promos, locations, vendors, analytics |
| **User (Staff)** | View products/catalog, create sales, view own reports — mostly read-only |

### Core Modules

| Module | Purpose |
|--------|---------|
| **Authentication** | Login, logout, session management |
| **Dashboard** | Role-specific overview with KPIs and quick actions |
| **Products** | Create/edit products with variations (colors), photos, dimensions, pricing |
| **Categories** | Organize products into categories and subcategories |
| **Inventory** | Track stock quantities across locations, adjust/set stock levels |
| **Sales** | Process sales with discounts, promo codes, multiple payment methods |
| **Members** | Customer membership management with phone-based lookup |
| **Transfers** | Move inventory between locations with approval workflow |
| **Locations** | Manage warehouses and showrooms |
| **Vendors** | Manage product suppliers |
| **Promo Codes** | Create and manage promotional codes with various rules |
| **Analytics/Reports** | Sales trends, inventory metrics, customer analytics, charts |
| **Bulk Operations** | Import/export data via CSV/Excel files |
| **Audit Logs** | Track who did what and when (Super Admin only) |
| **Error Reports** | In-app bug reporting system |

---

## 3. Testing Approach

### Phase 1: Smoke Testing (Week 1)

> "Does the application basically work?"

Go through every page and every major feature once. Don't go deep — just verify that:
- Pages load without errors
- Buttons are clickable
- Forms can be submitted
- Data appears where expected
- Navigation works

### Phase 2: Functional Testing (Weeks 2–4)

> "Does every feature work correctly in all scenarios?"

Test each module thoroughly using the test areas in [Section 5](#5-module-by-module-test-areas). Cover:
- Happy path (everything correct)
- Negative cases (wrong inputs, missing data)
- Edge cases (boundary values, unusual combinations)
- All three user roles

### Phase 3: Integration Testing (Weeks 4–5)

> "Do the modules work correctly together?"

Test cross-module workflows:
- Create a product → Add inventory → Sell it → Check stock decreased
- Create a transfer → Approve → Complete → Verify stock moved
- Create a member → Make a member sale → Check member total updated
- Apply a promo code → Verify discount calculated → Check analytics reflect it
- Bulk upload products → Verify they appear in catalog with correct data

### Phase 4: Regression Testing (Ongoing)

> "Did fixing something break something else?"

After every bug fix or new feature, re-run the smoke tests and any tests related to the changed area.

---

## 4. Test Priorities (Risk-Based)

Not all features are equally important. Prioritize testing based on **business impact** and **likelihood of failure**.

### Priority 1 — CRITICAL (Test First, Test Most)

These directly affect money, data accuracy, or system access:

| Area | Why It's Critical |
|------|------------------|
| **Sales processing** | Revenue — wrong calculations = lost money |
| **Inventory accuracy** | Stock levels must be exact across all locations |
| **Transfer workflow** | Stock movement between locations must be bulletproof |
| **Authentication & authorization** | Wrong access = security breach |
| **Payment calculations** | Discounts, promos, totals must be mathematically correct |

### Priority 2 — HIGH (Test Thoroughly)

| Area | Why It's Important |
|------|-------------------|
| **Product management** | Foundation of the entire system |
| **Member management** | Customer data integrity |
| **Promo codes** | Discount rules can be complex and error-prone |
| **Bulk upload/import** | Mass data entry — one bug affects hundreds of records |
| **Dashboard data accuracy** | Decisions are made based on these numbers |

### Priority 3 — MEDIUM (Test Adequately)

| Area | Reason |
|------|--------|
| **Analytics & reports** | Important but read-only — no data modification risk |
| **Categories & vendors** | Simpler CRUD operations |
| **Locations management** | Less frequently changed |
| **Data export** | Read-only operation |

### Priority 4 — LOW (Test Basically)

| Area | Reason |
|------|--------|
| **Audit logs** | Read-only, super admin only |
| **Error reports** | Internal tool |
| **Settings page** | Rarely used |
| **UI polish/cosmetic** | Important but not system-breaking |

---

## 5. Module-by-Module Test Areas

### 5.1 Authentication

**Happy Path:**
- [ ] Login with valid credentials for each role (Super Admin, Admin, User)
- [ ] After login, user lands on the correct dashboard
- [ ] Logout works and redirects to login page
- [ ] After logout, cannot access protected pages by typing URL directly

**Negative / Edge Cases:**
- [ ] Login with wrong password — should show error, not crash
- [ ] Login with non-existent username
- [ ] Login with empty username/password fields
- [ ] Login with spaces-only in fields
- [ ] Paste very long text (1000+ characters) into username/password
- [ ] Submit login form multiple times rapidly (double-click)
- [ ] After session expires, user should be redirected to login
- [ ] Check that password is hidden (dots/asterisks) in the field
- [ ] Browser back button after logout should NOT show dashboard

**Security Checks:**
- [ ] URL of a Super Admin page typed by a regular User — should show 401/unauthorized
- [ ] URL of an Admin page typed by a regular User — should show 401/unauthorized

---

### 5.2 Dashboard

**For Each Role (test as Super Admin, Admin, and User separately):**
- [ ] Dashboard loads without errors
- [ ] KPI numbers match actual data (cross-verify with other pages)
- [ ] Quick action buttons navigate to correct pages
- [ ] Widgets display relevant data for the role
- [ ] Date ranges (if any) filter data correctly

**Admin Dashboard Specific:**
- [ ] Inventory signals show correct low-stock alerts
- [ ] Location snapshot matches actual inventory data
- [ ] Transfer status counts match transfers page

**Super Admin Dashboard Specific:**
- [ ] Audit insights show recent activity
- [ ] Data integrity checks are accurate
- [ ] Risk indicators are meaningful

---

### 5.3 Products

**Creating a Product:**
- [ ] Fill all required fields — product is created successfully
- [ ] IMS Code must be unique — try creating with a duplicate code
- [ ] Try creating without required fields (name, IMS code, category, prices)
- [ ] Cost Price must be > 0 — try 0, negative, and text values
- [ ] MRP must be >= Cost Price — try MRP less than cost price
- [ ] At least one variation is required — try submitting with no variations
- [ ] Add multiple variations with different colors
- [ ] Add sub-variations to a variation
- [ ] Upload photos for variations — try valid images, oversized files, non-image files
- [ ] Set a primary photo — verify it shows as the main image
- [ ] Add discounts (percentage and flat) — verify calculations
- [ ] Dimensions (length, breadth, height, weight) — try negative and zero values

**Editing a Product:**
- [ ] All existing data loads correctly in the edit form
- [ ] Change each field one at a time and save — verify changes persist
- [ ] Try changing IMS Code to a code that already exists
- [ ] Add/remove variations during edit
- [ ] Add/remove photos during edit

**Deleting a Product:**
- [ ] Delete a product that has NO sales — should succeed
- [ ] Delete a product that HAS sales — what happens? (should it be blocked?)
- [ ] Delete a product that has inventory in locations — what happens?
- [ ] Confirm deletion dialog appears before actual deletion

**Product Catalog (Read-Only View):**
- [ ] All products appear in the catalog
- [ ] Search/filter works correctly
- [ ] Product details show correct information
- [ ] Regular Users can view but NOT edit

**Product Discounts:**
- [ ] Create a percentage discount — verify calculation on product
- [ ] Create a flat discount — verify calculation
- [ ] Set start/end dates — verify discount only applies within the date range
- [ ] Deactivate a discount — verify it no longer applies
- [ ] Multiple discounts on the same product — how do they interact?

---

### 5.4 Categories

- [ ] Create a category with name and description
- [ ] Create a category with a duplicate name — should fail
- [ ] Create subcategories within a category
- [ ] Delete a subcategory
- [ ] Delete a category that has products — what happens?
- [ ] Delete an empty category
- [ ] Verify products show correct category/subcategory relationships

---

### 5.5 Inventory

**Viewing Inventory:**
- [ ] Inventory summary shows correct totals
- [ ] View inventory by location — numbers match
- [ ] View inventory by product — shows stock at each location
- [ ] Zero-stock items display correctly

**Adjusting Inventory (Admin/Super Admin):**
- [ ] Increase stock — verify new quantity is correct
- [ ] Decrease stock — verify new quantity is correct
- [ ] Try to decrease stock below zero — should be blocked or show warning
- [ ] Adjust stock for a product variation at a specific location

**Setting Inventory (Admin/Super Admin):**
- [ ] Set stock to a specific number — verify it changes exactly
- [ ] Set stock to zero
- [ ] Try setting stock to a negative number — should fail

**Cross-Module Verification:**
- [ ] After a sale, verify inventory decreased by the sold quantity at that location
- [ ] After a transfer completes, verify source decreased and destination increased
- [ ] After bulk upload, verify inventory numbers are correct

---

### 5.6 Sales (CRITICAL — Test Extensively)

**Creating a General Sale:**
- [ ] Select a location
- [ ] Add one item — verify unit price and line total
- [ ] Add multiple items — verify subtotal
- [ ] Change quantity — verify totals recalculate
- [ ] Remove an item — verify totals recalculate
- [ ] Try to sell more than available stock — should warn/block
- [ ] Try to sell with quantity of 0 or negative — should fail
- [ ] Select payment method (CASH, CARD, CHEQUE, FONEPAY, QR)
- [ ] Split payment across multiple methods — verify total equals sale total
- [ ] Try to submit with payment less than total (non-credit sale) — should fail
- [ ] Submit sale — verify sale code is generated

**Creating a Member Sale:**
- [ ] Enter member phone number — member info should auto-populate
- [ ] Enter a non-member phone number — should indicate not a member
- [ ] After member sale, verify member's total sales updated
- [ ] Verify member-specific discounts/promos apply correctly

**Credit Sales:**
- [ ] Mark a sale as credit sale
- [ ] Verify partial payments can be added later
- [ ] Add a payment to a credit sale — verify remaining balance updates
- [ ] Check credit sale appears in reports correctly

**Discount & Promo Calculations (VERY IMPORTANT):**
- [ ] Product with percentage discount — verify math: price × (discount% / 100)
- [ ] Product with flat discount — verify math: price - discount amount
- [ ] Apply a promo code — verify additional discount applies
- [ ] Apply a promo code to a member sale — verify eligibility rules
- [ ] Apply a promo code to a non-member sale — verify eligibility rules
- [ ] Try an expired promo code — should fail
- [ ] Try a promo code at usage limit — should fail
- [ ] Promo code with override discounts — should replace product discounts
- [ ] Promo code with stacking allowed — should add on top of product discounts
- [ ] Use the preview endpoint — verify preview matches actual sale total
- [ ] **Manual calculation check:** For at least 10 sales, calculate expected total by hand and compare

**After Sale Verification:**
- [ ] Sale appears in the sales list
- [ ] Sale details page shows all correct information
- [ ] Inventory at the sale location decreased by the correct amount
- [ ] Payment record is correct
- [ ] Sale code format is correct
- [ ] "Since last login" sales report shows the new sale

---

### 5.7 Members

**Creating a Member:**
- [ ] Create with phone number only (minimum required)
- [ ] Create with all fields filled
- [ ] Try duplicate phone number — should fail
- [ ] Try invalid phone number format
- [ ] Try invalid email format
- [ ] Leave optional fields empty — should succeed

**Editing a Member:**
- [ ] Edit each field and save — verify changes persist
- [ ] Try changing phone to a number that already exists

**Member Search:**
- [ ] Search/check member by phone number — should find existing member
- [ ] Search for non-existent phone number — should indicate not found
- [ ] Phone lookup during sale creation — should work correctly

**Member Status:**
- [ ] Verify member status updates correctly (ACTIVE, INACTIVE, PROSPECT, VIP)
- [ ] Verify totalSales amount updates after member sales
- [ ] Verify memberSince and firstPurchase dates are set correctly

---

### 5.8 Transfers (CRITICAL — Multi-Step Workflow)

**Creating a Transfer:**
- [ ] Select source location (From) and destination location (To)
- [ ] Try same location as source and destination — should fail
- [ ] Add items with quantities
- [ ] Try to transfer more than available stock at source — should fail
- [ ] Add notes
- [ ] Submit — status should be PENDING

**Transfer Workflow (Test the Full Lifecycle):**
- [ ] PENDING → APPROVED: Admin approves the transfer
- [ ] APPROVED → IN_TRANSIT: Marks transfer as in transit — **verify source inventory decreases**
- [ ] IN_TRANSIT → COMPLETED: Complete the transfer — **verify destination inventory increases**
- [ ] Verify the stock numbers match exactly (source lost X, destination gained X)

**Transfer Cancellation:**
- [ ] Cancel a PENDING transfer — no inventory changes should occur
- [ ] Cancel an APPROVED transfer — no inventory changes should occur
- [ ] Can a transfer be cancelled after IN_TRANSIT? What happens to inventory?

**Transfer Logs:**
- [ ] Each status change creates a log entry
- [ ] Logs show correct user, action, and timestamp

**Edge Cases:**
- [ ] What happens if source location runs out of stock between creation and transit?
- [ ] What happens if a location is deactivated during a transfer?

---

### 5.9 Locations

- [ ] Create a WAREHOUSE location
- [ ] Create a SHOWROOM location
- [ ] Try duplicate location name — should fail
- [ ] Edit location details
- [ ] Deactivate a location — what happens to inventory and active transfers?
- [ ] Delete a location — what happens if it has inventory?
- [ ] Verify default warehouse is set correctly
- [ ] Verify location appears in sales and transfer dropdowns

---

### 5.10 Vendors

- [ ] Create a vendor with all fields
- [ ] Try duplicate vendor name — should fail
- [ ] Edit vendor details
- [ ] Delete a vendor — what happens to associated products?
- [ ] Verify vendor appears in product creation dropdown

---

### 5.11 Promo Codes

**Creating a Promo:**
- [ ] Create with all fields filled
- [ ] Try duplicate promo code — should fail
- [ ] Set eligibility to ALL, MEMBER, NON_MEMBER, WHOLESALE — verify each
- [ ] Set usage limit — verify promo stops working after limit reached
- [ ] Set valid date range — verify promo only works within range
- [ ] Apply to specific products only — verify it doesn't apply to other products
- [ ] Percentage type — verify calculation
- [ ] Flat type — verify calculation
- [ ] Override discounts setting — verify behavior
- [ ] Allow stacking setting — verify behavior

**Editing a Promo:**
- [ ] Change value — verify new value applies to new sales
- [ ] Deactivate — verify it can no longer be used
- [ ] Extend date range — verify promo works again

---

### 5.12 Bulk Upload/Import

**Products Bulk Upload:**
- [ ] Download the template — verify format is correct
- [ ] Fill template with valid data — upload succeeds
- [ ] Upload with missing required columns — should fail with clear error
- [ ] Upload with invalid data (wrong types, negative prices) — should fail with clear error
- [ ] Upload with duplicate IMS codes — should fail with clear error
- [ ] Upload an empty file — should fail gracefully
- [ ] Upload a non-CSV/Excel file (e.g., PDF, image) — should fail gracefully
- [ ] Upload a very large file (1000+ rows) — should complete or show progress
- [ ] After successful upload, verify all products appear correctly

**Members Bulk Upload:**
- [ ] Same checks as above but with member data
- [ ] Duplicate phone numbers in the file — how are they handled?

**Sales Bulk Upload:**
- [ ] Same checks as above but with sale data
- [ ] Verify inventory adjustments after bulk sale upload

**Data Export:**
- [ ] Export data — verify file downloads
- [ ] Open exported file — verify data is correct and complete
- [ ] Export with filters applied — verify only filtered data exports

---

### 5.13 Analytics & Reports

- [ ] Sales & Revenue analytics load correctly
- [ ] Date range filters work
- [ ] Charts display and are readable
- [ ] Numbers on analytics pages match numbers on list pages (cross-verify)
- [ ] Inventory & Operations analytics (Admin/Super Admin) show correct data
- [ ] Customer & Promo analytics show correct data
- [ ] Discount analytics — verify accuracy
- [ ] Payment trends — verify accuracy
- [ ] Location comparison — verify data matches individual location views
- [ ] Member cohort analysis — verify new vs repeat numbers
- [ ] User-level reports show only that user's data
- [ ] Admin-level reports show all data

---

### 5.14 Audit Logs (Super Admin)

- [ ] Perform various actions and verify they appear in audit logs
- [ ] Log entries show correct user, action, resource, and timestamp
- [ ] Logs are not editable or deletable
- [ ] Filtering/searching logs works correctly

---

### 5.15 Error Reports

- [ ] Submit an error report from any role
- [ ] Super Admin can view all error reports
- [ ] Super Admin can change status (OPEN → REVIEWED → RESOLVED)
- [ ] Non-super-admin users cannot access error reports management

---

## 6. Role-Based Testing Matrix

**You must test every feature from each role's perspective.** Use this matrix as your guide:

| Feature | Super Admin | Admin | User |
|---------|:-----------:|:-----:|:----:|
| Login/Logout | Test | Test | Test |
| Dashboard | Test (SA view) | Test (Admin view) | Test (User view) |
| Create Product | Test | Test | Should be blocked |
| Edit Product | Test | Test | Should be blocked |
| Delete Product | Test | Test | Should be blocked |
| View Catalog | Test | Test | Test |
| Adjust Inventory | Test | Test | Should be blocked |
| Create Sale | Test | Test | Test |
| View Sales | Test (all) | Test (all) | Test (own only) |
| Create Member | Test | Test | Should be blocked |
| Edit Member | Test | Test | Should be blocked |
| View Members | Test | Test | Test |
| Create Transfer | Test | Test | Should be blocked |
| Approve/Complete Transfer | Test | Test | Should be blocked |
| View Transfers | Test | Test | Test (read-only) |
| Manage Locations | Test (full) | Test (limited) | Should be blocked |
| Manage Vendors | Test | Test | Should be blocked |
| Manage Promos | Test | Test | Should be blocked |
| View Promos | Test | Test | Test |
| Bulk Upload | Test | Test | Should be blocked |
| Analytics (full) | Test | Test | Test (limited) |
| Manage Users | Test | Should be blocked | Should be blocked |
| Audit Logs | Test | Should be blocked | Should be blocked |
| Error Reports (manage) | Test | Should be blocked | Should be blocked |
| Error Reports (submit) | Test | Test | Test |

> **"Should be blocked"** means: try to access it anyway — verify the system prevents access with a proper error message (401/unauthorized page), NOT a crash or blank screen.

---

## 7. Test Types You Will Perform

### 7.1 Functional Testing
"Does the feature do what it's supposed to do?"
- Verify each feature works correctly with valid inputs
- Verify each feature handles invalid inputs gracefully

### 7.2 Boundary Value Testing
"What happens at the edges?"
- Minimum values: 0, 1, empty string
- Maximum values: very long text, very large numbers
- Just below/above limits: if max is 100, try 99, 100, 101

### 7.3 Negative Testing
"What happens when things go wrong?"
- Wrong data types (text in number fields)
- Missing required fields
- Invalid combinations
- Duplicate data where uniqueness is required

### 7.4 UI/UX Testing
"Is it usable and clear?"
- Error messages are helpful and specific (not just "Error occurred")
- Success messages appear after actions
- Loading states show while data is being fetched
- Buttons are disabled during submission (prevent double-submit)
- Forms clear/reset after successful submission
- Navigation is intuitive
- Responsive design (if applicable) — test different screen sizes
- No broken layouts, overlapping text, or cut-off elements

### 7.5 Data Integrity Testing
"Is the data accurate and consistent?"
- Numbers add up correctly (sales totals, inventory counts)
- Data on one page matches the same data on another page
- After create/edit/delete, lists update immediately
- Date/time displays are correct
- Sorting and filtering produce correct results

### 7.6 Workflow Testing
"Do multi-step processes work end-to-end?"
- Complete the full transfer lifecycle
- Complete the full sale process including payment
- Complete the bulk upload → verify → use workflow
- Product creation → inventory setup → sale → inventory check

### 7.7 Regression Testing
"Did a fix break something else?"
- After any bug fix, re-test the fixed area
- Re-test related areas that might be affected
- Run smoke tests on all modules

---

## 8. Bug Reporting Standards

Every bug report must be **reproducible** by someone else. Use this template:

### Bug Report Template

```
BUG ID:        [IMS-XXX] (sequential number)
TITLE:         [Short, descriptive title]
SEVERITY:      [Critical / High / Medium / Low]
MODULE:        [Which module: Sales, Inventory, etc.]
ROLE TESTED:   [Super Admin / Admin / User]
ENVIRONMENT:   [URL, Browser, OS]

STEPS TO REPRODUCE:
1. [Step 1 — be specific]
2. [Step 2 — include exact data you entered]
3. [Step 3 — what you clicked]

EXPECTED RESULT:
[What should have happened]

ACTUAL RESULT:
[What actually happened]

SCREENSHOT/VIDEO:
[Attach if applicable]

ADDITIONAL NOTES:
[Any other context, frequency, workarounds found]
```

### Severity Definitions

| Severity | Definition | Example |
|----------|-----------|---------|
| **Critical** | System unusable, data loss, security breach | Cannot login, sales data lost, wrong user sees admin pages |
| **High** | Major feature broken, no workaround | Cannot create sales, inventory not updating, calculations wrong |
| **Medium** | Feature broken but has workaround, or minor feature completely broken | Filter doesn't work but search does, export missing some columns |
| **Low** | Cosmetic, minor UI issue, typo | Button alignment off, text truncated, minor spelling error |

### Tips for Better Bug Reports
- **Be specific:** "Clicked the Save button" not "submitted the form"
- **Include data:** "Entered phone: 9841234567" not "entered a phone number"
- **One bug per report:** Don't combine multiple issues
- **Check if it's already reported:** Avoid duplicates
- **Always include screenshots** for UI issues

---

## 9. Test Execution Workflow

### Daily Routine

```
Morning:
├── Check for new builds/deployments
├── Run smoke tests on critical modules (15 min)
├── Review any bug fixes from yesterday
└── Re-test fixed bugs (regression)

Core Testing (main block):
├── Pick a module from the priority list
├── Execute test cases systematically
├── Log all bugs found
└── Take screenshots of issues

End of Day:
├── Update test progress tracker
├── Summarize findings (bugs found, areas tested)
└── Flag any blockers for tomorrow
```

### Test Tracking

Maintain a simple spreadsheet with these columns:

| Test Case ID | Module | Test Description | Role | Status | Bug ID (if failed) | Date Tested |
|-------------|--------|-----------------|------|--------|-------------------|-------------|
| TC-001 | Auth | Login with valid credentials (Admin) | Admin | Pass | — | 2026-02-10 |
| TC-002 | Auth | Login with wrong password | All | Fail | IMS-001 | 2026-02-10 |

### Status Values
- **Pass** — Works as expected
- **Fail** — Found a bug (link to bug report)
- **Blocked** — Cannot test (dependency or environment issue)
- **Not Tested** — Haven't gotten to it yet

---

## 10. Environment & Data Guidelines

### Test Accounts

You will need three test accounts (one per role):

| Role | Purpose |
|------|---------|
| Super Admin | Test all features including user management and audit logs |
| Admin | Test product, inventory, sales, member, and transfer management |
| User | Test limited access — sales creation, catalog view, own reports |

### Test Data Principles

1. **Never test on production data** — always use a test/staging environment
2. **Create your own test data** — so you know exactly what to expect
3. **Use recognizable test data** — prefix with "TEST-" so it's easy to identify and clean up
4. **Test with realistic data** — use plausible product names, prices, phone numbers
5. **Clean up after testing** — or coordinate with the team on data resets

### Recommended Test Data

Create at least:
- 5-10 products across different categories (with variations, photos, discounts)
- 3-5 categories with subcategories
- 3-5 vendors
- 2-3 locations (at least 1 warehouse, 1 showroom)
- 10-15 members with different statuses
- 3-5 promo codes with different rules (dates, eligibility, types)
- 15-20 sales of various types (general, member, credit, with promos)
- 3-5 transfers in various states

---

## 11. Glossary

| Term | Meaning |
|------|---------|
| **IMS Code** | Unique identifier for each product in the system |
| **Variation** | A version of a product (e.g., same shirt in different colors) |
| **Sub-variation** | A further detail of a variation (e.g., size within a color) |
| **MRP** | Maximum Retail Price — the listed selling price |
| **Cost Price** | What the business paid for the product |
| **Final SP** | Final Selling Price after discounts |
| **Transfer** | Moving stock from one location to another |
| **Credit Sale** | A sale where the customer hasn't paid in full yet |
| **Promo Code** | A code customers can use for additional discounts |
| **Stacking** | Allowing promo discounts to add on top of existing product discounts |
| **Override** | A promo replaces (overrides) existing product discounts instead of adding |
| **Audit Log** | A record of who did what action and when |
| **Smoke Test** | A quick, surface-level test to check if things are basically working |
| **Regression** | When a fix or change accidentally breaks something that was working before |
| **Edge Case** | An unusual situation that might not be obvious to test |
| **Happy Path** | The normal, expected way a user uses a feature (everything goes right) |
| **Negative Test** | Testing what happens when things go wrong (bad input, wrong actions) |
| **CRUD** | Create, Read, Update, Delete — the four basic data operations |
| **KPI** | Key Performance Indicator — important business metrics shown on dashboards |

---

## Quick Start Checklist

If this is your first day, start here:

- [ ] Get access to the test environment URL
- [ ] Get login credentials for all three roles
- [ ] Open the application and explore every page (just click around)
- [ ] Read through this document fully
- [ ] Set up your bug tracking spreadsheet
- [ ] Start with [Phase 1: Smoke Testing](#phase-1-smoke-testing-week-1)
- [ ] Ask the development team any questions — there are no dumb questions

---

*Remember: The best QA testers are curious, detail-oriented, and persistent. Your fresh perspective as a non-technical tester is invaluable — you see the application the way real users will.*

