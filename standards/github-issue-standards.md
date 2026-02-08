# GitHub Issue Standards

> **Last updated:** February 2026

---

## Table of Contents

1. [General Guidelines](#1-general-guidelines)
2. [Labels](#2-labels)
3. [Bug Report Template](#3-bug-report-template)
4. [Feature Request Template](#4-feature-request-template)
5. [Improvement / Refactor Template](#5-improvement--refactor-template)
6. [Issue Lifecycle](#6-issue-lifecycle)
7. [Good Examples vs Bad Examples](#7-good-examples-vs-bad-examples)

---

## 1. General Guidelines

- **One issue = one concern.** Don't combine multiple bugs or features into a single ticket.
- **Use clear, searchable titles.** Start with a prefix like `[Bug]`, `[Feature]`, or `[Refactor]`.
- **Assign labels** immediately when creating the issue.
- **Link related issues** using `Related to #123` or `Depends on #456`.
- **Attach screenshots or recordings** whenever the issue involves UI.
- **Use markdown** for readability — code blocks, bullet lists, headings.

---

## 2. Labels

Use these labels consistently across the repo:

| Label | Color | Description |
|---|---|---|
| `bug` | `#d73a4a` | Something isn't working |
| `feature` | `#0075ca` | New functionality request |
| `enhancement` | `#a2eeef` | Improvement to existing functionality |
| `refactor` | `#e4e669` | Code cleanup with no behavior change |
| `documentation` | `#0075ca` | Documentation updates |
| `priority: critical` | `#b60205` | Must fix immediately — blocking production |
| `priority: high` | `#d93f0b` | Should fix in current sprint |
| `priority: medium` | `#fbca04` | Plan for next sprint |
| `priority: low` | `#0e8a16` | Nice to have |
| `frontend` | `#bfdadc` | Frontend related |
| `backend` | `#c5def5` | Backend related |
| `needs-triage` | `#ededed` | Needs review and prioritization |

---

## 3. Bug Report Template

Use this template when reporting a bug.

### Title Format

```
[Bug] Short description of the broken behavior
```

**Examples:**
- `[Bug] Inventory count goes negative when two users submit at the same time`
- `[Bug] Login page shows blank screen on Safari 17`

### Template

```markdown
## Bug Report

### Description
A clear and concise description of what the bug is.

### Steps to Reproduce
1. Go to '...'
2. Click on '...'
3. Scroll down to '...'
4. See error

### Expected Behavior
A clear description of what you expected to happen.

### Actual Behavior
A clear description of what actually happened.

### Screenshots / Recordings
If applicable, add screenshots or screen recordings to help explain the problem.

### Environment
- **Browser:** Chrome 120 / Safari 17 / Firefox 121
- **OS:** macOS 14.2 / Windows 11 / Ubuntu 22.04
- **Screen size:** Desktop / Tablet / Mobile
- **Tenant:** (if multi-tenant related)
- **User role:** Super Admin / Org Admin / Staff

### Logs / Error Messages
```
Paste any relevant console errors, network errors, or server logs here.
```

### Severity
- [ ] **Critical** — App is down or data loss is occurring
- [ ] **High** — Core feature is broken, no workaround
- [ ] **Medium** — Feature is broken but has a workaround
- [ ] **Low** — Minor visual issue or edge case

### Additional Context
Add any other context about the problem here (e.g., started after a recent deployment, only happens with specific data, etc.)
```

---

## 4. Feature Request Template

Use this template when requesting a new feature.

### Title Format

```
[Feature] Short description of the desired functionality
```

**Examples:**
- `[Feature] Allow bulk CSV export of inventory items with filters`
- `[Feature] Add role-based dashboard widgets`

### Template

```markdown
## Feature Request

### Description
A clear and concise description of the feature you want.

### Problem Statement
Describe the problem this feature solves. Why is it needed?

> As a [type of user], I want [goal] so that [benefit].

### Proposed Solution
Describe how you think this should work. Be as specific as possible.

### UI/UX (if applicable)
- Describe where this fits in the current UI
- Attach wireframes, mockups, or sketches if available
- Describe the user flow step by step:
  1. User navigates to '...'
  2. User clicks '...'
  3. System shows '...'

### Acceptance Criteria
Define what "done" looks like. Use checkboxes:
- [ ] User can perform [action]
- [ ] System validates [condition]
- [ ] Data is persisted in [location]
- [ ] Works for all tenant types
- [ ] Responsive on mobile

### API Changes (if applicable)
Describe any new or modified endpoints:

| Method | Endpoint | Description |
|---|---|---|
| `POST` | `/api/v1/resource` | Creates a new resource |
| `GET` | `/api/v1/resource/:id` | Fetches resource by ID |

### Database Changes (if applicable)
Describe any schema changes:
- New table: `table_name`
- New column: `column_name` on `existing_table`
- New index needed on `column_name`

### Dependencies
- Depends on #issue_number (if any)
- Requires third-party service/library: [name]

### Priority
- [ ] **Critical** — Blocking a launch or major workflow
- [ ] **High** — Important for upcoming sprint
- [ ] **Medium** — Should be planned soon
- [ ] **Low** — Nice to have

### Additional Context
Any other context, references, competitor examples, or design documents.
```

---

## 5. Improvement / Refactor Template

Use this template for code improvements that don't change behavior.

### Title Format

```
[Refactor] Short description of what is being improved
```

**Examples:**
- `[Refactor] Extract shared validation logic into reusable utility`
- `[Refactor] Migrate inventory queries to use Prisma raw SQL for performance`

### Template

```markdown
## Improvement / Refactor

### Description
What code or system is being improved and why.

### Current State
Describe the current implementation and its problems:
- Performance issues
- Code duplication
- Poor readability
- Tech debt

### Proposed Changes
Describe the changes at a high level:
- [ ] Move X from A to B
- [ ] Extract shared logic into utility
- [ ] Replace library X with Y

### Files Affected
List the files or modules that will change:
- `src/modules/inventory/...`
- `src/shared/utils/...`

### Risks
- [ ] Could break existing tests
- [ ] Requires migration
- [ ] Impacts other modules

### Testing Plan
How will you verify nothing is broken:
- [ ] Existing tests pass
- [ ] Manual smoke test of affected features
- [ ] New tests added for extracted logic
```

---

## 6. Issue Lifecycle

```
┌──────────┐     ┌──────────┐     ┌─────────────┐     ┌───────────┐     ┌────────┐
│  Open /  │────▶│  Triaged │────▶│ In Progress │────▶│ In Review │────▶│ Closed │
│  New     │     │          │     │             │     │           │     │        │
└──────────┘     └──────────┘     └─────────────┘     └───────────┘     └────────┘
                      │                                                      ▲
                      │              ┌──────────┐                            │
                      └─────────────▶│ Won't Do │────────────────────────────┘
                                     └──────────┘
```

| Stage | Who | Action |
|---|---|---|
| **Open / New** | Author | Issue is created with `needs-triage` label |
| **Triaged** | Tech Lead | Labels, priority, and assignee are set |
| **In Progress** | Developer | Work has started, linked to a branch or PR |
| **In Review** | Reviewer | PR is open and under review |
| **Closed** | Developer | PR is merged or issue is resolved |
| **Won't Do** | Tech Lead | Issue is rejected with a reason in comments |

---

## 7. Good Examples vs Bad Examples

### Bad Bug Title
> `Login broken`

### Good Bug Title
> `[Bug] Login returns 500 error when email contains a "+" character`

---

### Bad Feature Description
> `We need export functionality`

### Good Feature Description
> **As an** Org Admin, **I want** to export my inventory items as a CSV file with applied filters **so that** I can share reports with stakeholders who don't have system access.
>
> **Acceptance Criteria:**
> - Export respects current search/filter state
> - CSV includes columns: name, SKU, quantity, category, last updated
> - Download triggers immediately (no email required for <10k rows)
> - Works for all tenant plan types

---

### Bad Steps to Reproduce
> `Just go to the page and it breaks`

### Good Steps to Reproduce
> 1. Log in as Org Admin (tenant: `acme-corp`)
> 2. Navigate to **Inventory > All Items**
> 3. Set filter: Category = "Electronics"
> 4. Click **Export CSV**
> 5. Observe: browser shows a blank white page and console logs `TypeError: Cannot read property 'map' of undefined`
