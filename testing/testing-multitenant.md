# Multi-Tenant Testing Manual

> **Audience:** QA Tester
> **Scope:** Multi-tenant isolation and subscription behavior only
> **Last updated:** February 2026

---

## Test Accounts

### Organizations

| Organization | Slug | Plan | Status |
|---|---|---|---|
| Default Organization | `default` | PROFESSIONAL | Active (1 year) |
| Asha Boutique | `test-org` | STARTER | Trial (14 days) |

### Users

| Organization | Username | Password | Role |
|---|---|---|---|
| Default Org | *(from .env)* | *(from .env)* | Super Admin |
| Asha Boutique | `testadmin` | `test123` | Super Admin |
| Asha Boutique | `testuser` | `test123` | User |
| Cross-tenant | `platform` | `platform123` | Platform Admin |

### What Each Org Has (for reference)

| | Default Organization | Asha Boutique |
|---|---|---|
| Categories | Electronics, Furniture, Clothing, Sports, Books | Women's Wear, Accessories, Men's Wear |
| Products | 18 (headphones, chairs, t-shirts, etc.) | 7 (pashmina shawls, sarees, kurtas, jewelry) |
| Vendors | 5 (Global Electronics, Comfort Furniture, etc.) | 2 (Silk Road Fabrics, Mountain Craft) |
| Locations | 3 (Warehouse + 2 Showrooms) | 1 (Main Store) |
| Members | 5 (Rajesh, Sunita, Amit, etc.) | 3 (Sita Devi, Hari Bahadur, Kamala) |
| Sales | 10 sample sales | None |

---

## Test Scenarios

### 1. Login & Tenant Selection

| # | Test | Steps | Expected |
|---|---|---|---|
| 1.1 | Login to Default Org | Select "Default Organization" in dropdown, enter your admin credentials | Dashboard loads, sidebar shows "Default Organization" + "PROFESSIONAL" badge |
| 1.2 | Login to Asha Boutique | Select "Asha Boutique (Test)", enter `testadmin` / `test123` | Dashboard loads, sidebar shows "Asha Boutique" + "STARTER" badge |
| 1.3 | Wrong org credentials | Select "Asha Boutique", enter Default Org credentials | Error: "Invalid username or password" |
| 1.4 | Tenant in user dropdown | Click avatar (top-right) | Shows username AND organization name |

### 2. Data Isolation (CRITICAL)

For each item below: log into Default Org, note the data. Log out. Log into Asha Boutique, verify the data is **completely different**.

| # | Test | Where to check | Default Org should show | Asha Boutique should show |
|---|---|---|---|---|
| 2.1 | Products | Products page | Electronics, furniture, clothing items | Pashmina shawls, sarees, kurtas, jewelry |
| 2.2 | Categories | Categories page | Electronics, Furniture, Clothing, Sports, Books | Women's Wear, Accessories, Men's Wear |
| 2.3 | Vendors | Vendors page | Global Electronics, Comfort Furniture, etc. | Silk Road Fabrics, Mountain Craft Supplies |
| 2.4 | Locations | Locations page | Main Warehouse, Showroom A, Showroom B | Main Store |
| 2.5 | Members | Members page | Rajesh, Sunita, Amit, Priya, Bikash | Sita Devi, Hari Bahadur, Kamala |
| 2.6 | Sales | Sales page | 10 sample sales | Empty |
| 2.7 | Users | Users page (superAdmin) | Default Org users only | testadmin, testuser only |

**If any data from one org appears in the other, that is a CRITICAL bug.**

### 3. Cross-Tenant Write Isolation

| # | Test | Steps | Expected |
|---|---|---|---|
| 3.1 | Create product in Asha | Login to Asha Boutique, create a new product | Product appears in Asha Boutique |
| 3.2 | Verify not in Default | Login to Default Org, check products | The product from 3.1 must NOT appear |
| 3.3 | Create member in Asha | Login to Asha Boutique, create a new member | Member appears in Asha Boutique |
| 3.4 | Verify not in Default | Login to Default Org, check members | The member from 3.3 must NOT appear |
| 3.5 | Create sale in Asha | Login to Asha Boutique, create a sale | Only Asha products/locations/members available in dropdowns |
| 3.6 | Verify not in Default | Login to Default Org, check sales | The sale from 3.5 must NOT appear |

### 4. Dropdown/Filter Isolation

| # | Test | Steps | Expected |
|---|---|---|---|
| 4.1 | Category filter | When creating a product, open category dropdown | Only current org's categories listed |
| 4.2 | Vendor filter | When creating a product, open vendor dropdown | Only current org's vendors listed |
| 4.3 | Location filter | When creating a sale, open location dropdown | Only current org's locations listed |
| 4.4 | Member search | During sale, search for a member by phone | Only current org's members returned |

### 5. Subscription & Plan Display

| # | Test | Steps | Expected |
|---|---|---|---|
| 5.1 | Plan badge (Default) | Login to Default Org, check sidebar header | Shows "PROFESSIONAL" badge |
| 5.2 | Plan badge (Asha) | Login to Asha Boutique, check sidebar header | Shows "STARTER" badge |
| 5.3 | Different plan context | Compare the two orgs | Plans are visually different, reflecting each org's subscription |

### 6. Search Isolation

| # | Test | Steps | Expected |
|---|---|---|---|
| 6.1 | Product search | In Asha Boutique, search "Pashmina" | Found |
| 6.2 | Cross-org search | In Default Org, search "Pashmina" | NOT found |
| 6.3 | Member search | In Default Org, search "Rajesh" | Found |
| 6.4 | Cross-org member search | In Asha Boutique, search "Rajesh" | NOT found |

### 7. Same-Name Across Tenants

| # | Test | Steps | Expected |
|---|---|---|---|
| 7.1 | Same category name | Create category "Test" in Default Org AND in Asha Boutique | Both succeed (names are unique per org, not globally) |
| 7.2 | Same member phone | Create member with phone "9800000000" in both orgs | Both succeed |
| 7.3 | Same username | Note: `testadmin` exists in Asha. A user named `testadmin` could also exist in Default Org | Both can coexist (usernames are unique per org) |

---

## Bug Reporting

When you find an isolation issue:

```
TITLE: [Short description]
SEVERITY: Critical / High / Medium / Low
ORG LOGGED INTO: [Default / Asha Boutique]
USER: [username]
STEPS:
1. 
2. 
3. 
EXPECTED: [What should happen]
ACTUAL: [What actually happened]
SCREENSHOT: [Attach if possible]
```

**Severity guide:**
- **Critical**: Data from one org visible in another org
- **High**: Can create/modify data that appears in wrong org
- **Medium**: Dropdown shows items from wrong org
- **Low**: Plan badge or tenant name display issue
