# Business Module — Product Requirements Document
> Status: Awaiting Product Team Input
> Purpose: Product team fills this → Backend team builds from this → No mid-development changes
> Rule: Every question must have a final answer before development starts

---

## How to Use This Document

- Read each section
- Answer every question (no skipping)
- Mark each section: **DECIDED** or **PENDING**
- If unsure → default recommendation is given — accept or change it
- When all sections = DECIDED → this doc is locked → backend builds

---

## Section 1 — What Is a "Business" in OneNex?

### 1.1 Core Definition

> In OneNex, a "Business" is the entity an owner creates to operate from.

**Decision needed:**

Does one Business = one brand/company? Or one Business = one physical location?

```
Example A (one brand):
  Owner creates "Burger Palace"
  Burger Palace has 3 locations (Colombo, Kandy, Jaffna)
  = 1 Business with 3 branches

Example B (one location):
  Owner creates "Burger Palace Colombo"
  Owner creates "Burger Palace Kandy"
  = 2 separate Businesses
```

- [ ] **Option A** — One Business = one brand, can have multiple locations (branches)
- [ ] **Option B** — One Business = one location only

**Default recommendation:** Option A — one brand, multiple branches under it.

**Product Team Answer:**
> *(Write answer here)*

---

### 1.2 Business Identity

What information defines a Business?

| Field | Required? | Notes |
|---|---|---|
| Legal / Registered name | Yes / No | The official registered name |
| Trading name (display name) | Yes / No | What customers see |
| Business registration number | Yes / No | Government registration |
| Logo | Yes / No | Shown on receipts, invoices |
| Cover photo | Yes / No | Shown on booking pages |
| Description | Yes / No | About the business |
| Website URL | Yes / No | |
| Contact email | Yes / No | |
| Contact phone | Yes / No | |
| Physical address | Yes / No | |
| Google Maps / coordinates | Yes / No | For delivery, maps |
| Country | Yes / No | Affects tax, currency |
| Currency | Yes / No | Default billing currency |
| Timezone | Yes / No | Affects hours, reports |
| Language | Yes / No | Default display language |

Mark Yes/No for each.

**Product Team Answer:**
> *(Write answer here)*

---

## Section 2 — Business URL & Identity

### 2.1 Business Slug (URL identity)

Every business needs a unique URL identifier.

```
Example:
  grandhotel.onenex.com
  "grandhotel" = slug
```

**Questions:**

**Q1:** Who sets the slug — owner types it, or system auto-generates from business name?
- [ ] Owner types it manually
- [ ] System auto-generates, owner can edit
- [ ] System auto-generates, owner cannot change

**Q2:** Can the slug be changed later?
- [ ] Yes — owner can change anytime
- [ ] Yes — but old slug must still work (redirect)
- [ ] No — slug is permanent once set

**Q3:** What happens to old bookmarks and QR codes if slug changes?
- [ ] Old slug stops working
- [ ] Old slug redirects to new slug automatically

**Product Team Answer:**
> *(Write answer here)*

---

## Section 3 — Branches / Locations

### 3.1 Does V1 Support Multiple Branches?

```
Scenario: "Burger Palace" has branches in Colombo, Kandy, Jaffna.
One owner logs in and manages all three.
```

- [ ] **Yes** — V1 supports multi-branch
- [ ] **No** — V1 is single location only, multi-branch is Phase 2

**Product Team Answer:**
> *(Write answer here)*

---

### 3.2 If Multi-Branch — What Is Different Per Branch?

If owner has 3 branches, what can be different per branch vs shared?

| Setting | Same for all branches | Different per branch | Not applicable |
|---|---|---|---|
| Brand name | | | |
| Logo | | | |
| Address | | | |
| Phone number | | | |
| Operating hours | | | |
| Staff | | | |
| Tax rates | | | |
| Menu / Room types | | | |
| Currency | | | |

Mark each column.

**Product Team Answer:**
> *(Write answer here)*

---

### 3.3 Staff Access Across Branches

```
Scenario: Kasun works at Colombo branch.
Can he log in and see Kandy branch data?
```

- [ ] Staff sees only their assigned branch
- [ ] Staff can be assigned to multiple branches (sees all assigned)
- [ ] Owner decides per staff member which branches they can access

**Product Team Answer:**
> *(Write answer here)*

---

### 3.4 Headquarters Concept

```
When a business is created → one main/head branch is auto-created.
Additional branches can be added later.
```

- [ ] Yes — first branch = HQ, cannot be deleted
- [ ] No — no HQ concept, all branches equal

**Product Team Answer:**
> *(Write answer here)*

---

### 3.5 Branch Limits

Is there a limit on how many branches one Business can have?

- [ ] No limit
- [ ] Limit based on subscription plan
- [ ] Fixed limit (specify: _____)

**Product Team Answer:**
> *(Write answer here)*

---

## Section 4 — Operations (What the Business Does)

### 4.1 Available Operations

OneNex operations a business can enable:

| Operation | Include in V1? | Notes |
|---|---|---|
| Stays (Hotel / Accommodation) | Yes / No | |
| Dining (Restaurant / F&B) | Yes / No | |
| Bar (Beverage service) | Yes / No | |
| Wellness (Spa / Treatments) | Yes / No | |
| Events (Conference / Ticketing) | Yes / No | |
| Retail (Products / POS) | Yes / No | |

Mark Yes/No for V1 inclusion.

**Product Team Answer:**
> *(Write answer here)*

---

### 4.2 Who Can Enable/Disable Operations?

- [ ] Only the Business Owner
- [ ] Owner + Admin role
- [ ] Any staff with permission

**Product Team Answer:**
> *(Write answer here)*

---

### 4.3 Operation Enable Flow

When owner enables an operation (e.g., Stays):

**Q1:** Does it activate immediately or go through a setup wizard first?
- [ ] Activates immediately → owner can set up later
- [ ] Setup wizard must be completed before operation is "live"
- [ ] Activates immediately but shows "incomplete setup" warning

**Q2:** Can an operation be disabled after it has real data (bookings, orders)?
- [ ] Yes — disable blocks new activity, old data kept
- [ ] No — once enabled with data, cannot be disabled
- [ ] Yes — but requires manager confirmation warning

**Q3:** If operation is disabled and re-enabled, does old data come back?
- [ ] Yes — all old data restored
- [ ] Yes — data kept but needs reconfiguration

**Product Team Answer:**
> *(Write answer here)*

---

### 4.4 Operation per Branch

```
Scenario: "Grand Hotel Colombo" 
  Colombo branch → Stays + Dining enabled
  Kandy branch   → Dining only
```

- [ ] Operations are per Business (all branches get same operations)
- [ ] Operations can be different per branch
- [ ] Operations are per Business in V1, per Branch in Phase 2

**Product Team Answer:**
> *(Write answer here)*

---

## Section 5 — Business Hours

### 5.1 What Are Business Hours Used For?

Which of these does business hours affect?

| Use case | Yes / No |
|---|---|
| Shown to customers (public info) | |
| Controls when staff can log in | |
| Controls when orders/bookings can be accepted | |
| Used in reports | |
| Controls kitchen/bar operational availability | |

**Product Team Answer:**
> *(Write answer here)*

---

### 5.2 Split Hours (Multiple Time Slots Per Day)

```
Scenario: Restaurant is open:
  Lunch:  11:00 AM – 3:00 PM
  Dinner: 6:00 PM – 11:00 PM
```

- [ ] V1 supports multiple time slots per day
- [ ] V1 = one time slot per day only (open time → close time)

**Product Team Answer:**
> *(Write answer here)*

---

### 5.3 Overnight Hours

```
Scenario: Bar is open 6 PM to 3 AM (closes next day)
```

- [ ] V1 supports overnight hours (cross-midnight)
- [ ] Not needed in V1

**Product Team Answer:**
> *(Write answer here)*

---

### 5.4 Hour Exceptions (Holidays, Special Days)

```
Scenario: Christmas Day — business is closed.
  Ramadan — different hours for the month.
```

- [ ] V1 supports date-specific exceptions
- [ ] V1 supports date-range exceptions (e.g., whole of Ramadan)
- [ ] Not in V1 — Phase 2

**Product Team Answer:**
> *(Write answer here)*

---

### 5.5 Operation-Specific Hours

```
Scenario: Hotel is 24/7 but:
  Restaurant: 7 AM – 10 PM
  Spa:        9 AM – 8 PM
```

- [ ] Each operation has its own hours (separate from business hours)
- [ ] All operations follow business hours
- [ ] Business hours = default, operations can override

**Product Team Answer:**
> *(Write answer here)*

---

## Section 6 — Tax Configuration

### 6.1 Tax Types in V1

Which tax types does OneNex need to support?

| Tax Type | Include? |
|---|---|
| VAT | Yes / No |
| GST | Yes / No |
| Service Charge | Yes / No |
| Tourism Levy | Yes / No |
| Government Fee | Yes / No |
| Custom (owner-defined) | Yes / No |

**Product Team Answer:**
> *(Write answer here)*

---

### 6.2 Tax Application Level

How are taxes applied?

- [ ] Same tax applies to all operations (VAT = 18% on everything)
- [ ] Different tax per operation (Dining gets VAT+SC, Stays gets VAT+SC+Tourism)
- [ ] Different tax per item/product category (Phase 2 complexity)

**V1 recommendation:** Per operation level only. Item-level in Phase 2.

**Product Team Answer:**
> *(Write answer here)*

---

### 6.3 Tax Rate Changes Over Time

```
Scenario: VAT was 15% in 2025, changed to 18% in 2026.
Old invoices must still show 15%.
New invoices show 18%.
```

- [ ] Yes — tax rate history must be kept, old transactions unaffected
- [ ] No — tax change updates everything (not recommended)

**Product Team Answer:**
> *(Write answer here)*

---

### 6.4 Tax Inclusive vs Exclusive

```
Tax Exclusive: Price = LKR 1000, then +18% VAT = LKR 1180 total
Tax Inclusive: Price shown = LKR 1180 (VAT already inside)
```

- [ ] V1 supports both modes, owner chooses per business
- [ ] V1 = exclusive only
- [ ] V1 = inclusive only

**Product Team Answer:**
> *(Write answer here)*

---

## Section 7 — Business Settings

### 7.1 Global Settings in V1

What global settings can the owner configure?

| Setting | V1 / Phase 2 / Not needed |
|---|---|
| Date format (DD/MM/YYYY vs MM/DD/YYYY) | |
| Time format (12h vs 24h) | |
| Week start day (Monday vs Sunday) | |
| Show logo on receipts | |
| Default language | |
| Receipt footer text | |
| Invoice prefix (INV-001 format) | |

**Product Team Answer:**
> *(Write answer here)*

---

## Section 8 — Business Lifecycle

### 8.1 Business Status States

What states can a business be in?

| Status | Include in V1? | Who can trigger? |
|---|---|---|
| Active (normal operation) | Yes | Auto on creation |
| Suspended (blocked — payment issue etc.) | Yes / No | Platform admin only |
| Temporarily Closed (owner-initiated) | Yes / No | Owner |
| Permanently Closed | Yes / No | Owner |

**Product Team Answer:**
> *(Write answer here)*

---

### 8.2 What Happens When Business Is Suspended?

```
Platform suspends a business (e.g., payment failure):
```

- [ ] Staff cannot log in at all
- [ ] Staff can log in but cannot take new orders/bookings
- [ ] Owner can still log in to resolve issue, but no operations work

**Product Team Answer:**
> *(Write answer here)*

---

### 8.3 Business Deletion

Can a business ever be deleted?

- [ ] No — businesses can only be closed, never deleted (data kept)
- [ ] Yes — owner can delete after X days closed
- [ ] Only platform admin can delete

**Product Team Answer:**
> *(Write answer here)*

---

## Section 9 — Out of Scope (Confirm These Are NOT V1)

Product team must confirm these are Phase 2+ only.

| Feature | Confirm Out of V1 |
|---|---|
| Franchise model (different owner per location, royalties) | Yes out / No bring in |
| White-label (client's own domain/branding) | Yes out / No bring in |
| Group/Chain management portal (manage 5 brands from one dashboard) | Yes out / No bring in |
| Business transfer (sell business to another owner) | Yes out / No bring in |
| Multi-currency per branch | Yes out / No bring in |
| API access for third-party integrations | Yes out / No bring in |
| Public business listing/directory on OneNex | Yes out / No bring in |

**Product Team Answer:**
> *(Confirm each row)*

---

## Section 10 — Special Scenarios (Must Answer)

These are edge cases that backend must handle. Product team decides the behavior.

**Scenario 1:**
> Owner creates business → immediately wants to add another branch. No setup done yet.
> **What happens?** Can they add branch before completing setup?

**Product Team Answer:**
> *(Write answer here)*

---

**Scenario 2:**
> Owner has Dining enabled. They enable Stays too.
> **What happens automatically?** Does anything auto-setup? Does "room charge" feature appear in Dining automatically?

**Product Team Answer:**
> *(Write answer here)*

---

**Scenario 3:**
> Owner wants to temporarily close for Ramadan (1 month).
> **How do they do this?** Through business hours exceptions? Through business status? Both?

**Product Team Answer:**
> *(Write answer here)*

---

**Scenario 4:**
> Owner changes their tax rate mid-year (VAT goes up).
> **What happens to invoices from before the change?** Must they still show old rate?

**Product Team Answer:**
> *(Write answer here)*

---

**Scenario 5:**
> Owner has 3 branches. One branch is permanently closing.
> **What happens to that branch's data** (bookings, orders, staff)? Is it deleted or archived?

**Product Team Answer:**
> *(Write answer here)*

---

**Scenario 6:**
> Two staff members are working at the same time, both editing business hours.
> **Who wins?** Last save wins, or should system warn about conflict?

**Product Team Answer:**
> *(Write answer here)*

---

## Section 11 — Sign-off

Once all sections are answered, product team signs off here.

| Section | Status | Signed by | Date |
|---|---|---|---|
| Section 1 — Business Definition | PENDING / DECIDED | | |
| Section 2 — URL & Slug | PENDING / DECIDED | | |
| Section 3 — Branches | PENDING / DECIDED | | |
| Section 4 — Operations | PENDING / DECIDED | | |
| Section 5 — Business Hours | PENDING / DECIDED | | |
| Section 6 — Tax | PENDING / DECIDED | | |
| Section 7 — Settings | PENDING / DECIDED | | |
| Section 8 — Lifecycle | PENDING / DECIDED | | |
| Section 9 — Out of Scope | PENDING / DECIDED | | |
| Section 10 — Special Scenarios | PENDING / DECIDED | | |

**All DECIDED → Backend team starts building. Zero changes after this.**
