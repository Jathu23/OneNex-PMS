# Identity Module — Open Decisions

> Status: IN DISCUSSION — Resolve before implementation.
> These decisions affect all 3 surfaces and the overall identity architecture.

---

## Context: 3 Surfaces

```
Surface 1: Owner Portal      → owner creates & manages business
Surface 2: Operations Portal → staff day-to-day work
Surface 3: Customer Web/App  → end customer of a business (white-labeled)
```

Surface 1 + 2 identity → `ApplicationUser` (confirmed)
Surface 3 identity → decisions below

---

## D1 — Customer Account Scope

**Question:** When a customer creates an account, what scope does it belong to?

```
Option A: OneNex-global
  → Same account works across ALL businesses on OneNex (any owner)
  → Customer has a "OneNex account" (they know it's OneNex)

Option B: Business-level
  → Account tied to one specific business
  → Hotel + Restaurant (same owner) → 2 separate accounts

Option C: Owner-level  ← likely correct
  → Account works across all businesses of the same owner
  → Hotel + Restaurant (same owner) → 1 account, full history
  → Customer never knows about "OneNex" — they see the brand
```

**Decision:** ___
**Reason:** ___

---

## D2 — Anonymous vs Login (Which operations need account?)

**Question:** Which customer operations require a login vs can be done anonymously?

| Operation | Anonymous | Login Required | Notes |
|-----------|-----------|----------------|-------|
| View menu via QR | ? | ? | |
| Place order via QR | ? | ? | |
| Online room booking | ? | ? | just name+email enough? |
| View booking history | ? | ? | |
| Cancel/modify booking | ? | ? | |
| Loyalty / points | ? | ? | |

**Decision:** ___
**Reason:** ___

---

## D3 — Customer Login Method

**Question:** How does a customer authenticate?

```
Option A: Email + Password only
Option B: Phone + OTP only
Option C: Email + Password AND Phone + OTP
Option D: Google / Apple login
Option E: All of the above
```

**V1 scope:** ___
**Phase 2:** ___
**Decision:** ___

---

## D4 — Same Person, Two Roles

**Question:** Can the same person be both staff and customer?

```
Example:
  Arun works as staff at Business A (has ApplicationUser)
  Arun also books a room at Business B (same owner) as a customer

Options:
  A: Same ApplicationUser — one account, two roles
     → Clean, no duplicate accounts
     → Need to separate staff context vs customer context in JWT

  B: Separate accounts — staff account vs customer account
     → Simpler per-system
     → Same person = 2 accounts (bad UX)
```

**Decision:** ___
**Reason:** ___

---

## D5 — Customer Identity System

**Question:** Do customers use the same ASP.NET Identity (ApplicationUser) or a separate system?

```
Option A: Same ApplicationUser
  → One identity system for everyone
  → customer_profiles table linked to ApplicationUser
  → Simpler infrastructure

Option B: Separate customer_accounts table
  → Lightweight auth for customers
  → Completely decoupled from staff/owner identity
  → More complex to maintain two auth systems
```

**Decision:** ___
**Reason:** ___

---

## D6 — Guest (No Account) vs Customer (Has Account)

**Question:** Can a booking exist without a customer account?

```
Scenario: Staff creates walk-in booking → just name + phone
  → No login, no account
  → guest_profiles record only

Scenario: Customer books online → creates account
  → ApplicationUser (or customer_accounts) + guest_profiles

Question: Can a guest later "claim" their booking?
  → Walk-in guest later registers online
  → System links their new account to old bookings
  → V1 or Phase 2?
```

**Decision (guest without account):** ___
**Decision (claim flow):** ___

---

## Decision Log

| # | Decision | Status | Decided |
|---|----------|--------|---------|
| D1 | Customer account scope | PENDING | — |
| D2 | Anonymous vs login | PENDING | — |
| D3 | Customer login method | PENDING | — |
| D4 | Same person, two roles | PENDING | — |
| D5 | Customer identity system | PENDING | — |
| D6 | Guest vs customer account | PENDING | — |
