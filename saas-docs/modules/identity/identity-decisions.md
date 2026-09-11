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

**Decision:** Option A (OneNex-global) for the *account*; explicitly **not** Option C for the *data*.

**Reason:** The account itself is the same `users` (`ApplicationUser`) row Identity already uses for owners and staff — "no separate 'customer account' table" (`customer-module-design.md` §2, Tier 1). One email = one account everywhere on OneNex (`identity-module-design.md` § Registration: "One account per email — period").

What actually varies per scope is the *customer relationship data*, and it lands tighter than the "owner-level" this doc originally guessed: `business_customers` is scoped per `business_id`, not per owner. §8.4 states this directly with the exact Hotel+Restaurant/same-owner example this question raises — "A customer who is a guest of Grand Hotel is **not** automatically visible to City Apartments just because they share an owner." So: one global login, but zero automatic data-sharing even within one owner's businesses (a future opt-in "franchise/loyalty network" is flagged, not built — §8.4/§18).

---

## D2 — Anonymous vs Login (Which operations need account?)

**Question:** Which customer operations require a login vs can be done anonymously?

| Operation | Anonymous | Login Required | Notes |
|-----------|-----------|----------------|-------|
| View menu via QR | ✅ | — | Pure read, never touches `business_customers` |
| Place order via QR | ✅ | — | Guest capture only (name + phone) — same shape as the walk-in path, `customer-module-design.md` §6.1. Order FKs to a guest `business_customer_id`; no account needed |
| Online room booking | ✅ (login optional) | — | Guest checkout works the same way (name + phone/email is enough — yes, "just name+email" is sufficient). If already logged in, resolves via `AttachToBusinessAsync` instead (§6.2) — login is an enhancement, never a requirement |
| View booking history | — | ✅ | Served by `GET /api/customers/me/businesses` (JWT_1). A guest `business_customers` row has no login of its own to authenticate as until it's linked |
| Cancel/modify booking | — | ✅ (for now) | Neither canonical doc defines a guest-safe cancel/modify path — the OTP-link idea (`guest_action_tokens`) exists only in the superseded `customer-identity-design.md`. Until revisited, this needs a linked account, or a separate confirmation-code mechanism the *booking* module (Dining/Stays) would have to design on its own |
| Loyalty / points | — | ✅ | Needs a persistent identity to accrue against; also explicitly out of scope for V1 regardless (`customer-module-design.md` §16 — only `marketing_opt_in` exists, no points ledger) |

**Decision:** Guest-capable for anything one-shot (view/order/book); login required for anything that reads back persisted state (history/cancel/loyalty).

**Reason:** `customer-module-design.md` §1 states the governing rule directly — a walk-in "must not be forced through email verification/password creation just to get a receipt." `business_customers.user_id` being nullable (§2) is what makes this possible: a guest profile can capture and serve in the moment, but has no login path back to itself later, so anything requiring later retrieval needs the profile linked to a real account first (the claim flow, §6.1).

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

**V1 scope:** Option A (Email + Password), inherited from Identity — not a customer-specific choice.
**Phase 2:** Not specified in either canonical doc — phone+OTP or social login would need their own design pass if pursued.
**Decision:** Option A for V1.

**Reason:** This is derived, not directly stated — neither `customer-module-design.md` nor `identity-module-design.md` defines a customer-specific login mechanism, because there isn't one to define: "a customer registering is just a normal Identity registration with no business role attached yet" (`customer-module-design.md` §2). Identity's only implemented auth flow is `POST /auth/register` / `POST /auth/login` with email + password and mandatory email verification (`identity-module-design.md` § Registration, § APIs) — no OTP or social login endpoint exists anywhere in that doc. So customers get exactly the same mechanism as owners/staff, by the same "no separate account type" logic as D5 below — not a deliberate customer-specific choice, just the natural consequence of D5's answer.

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

**Decision:** Option A — same `ApplicationUser`, one account, two independent roles.

**Reason:** Stated explicitly in `customer-module-design.md` §14: "A person can simultaneously be `staff_memberships` (owner of Business A) and `business_customers` (a guest customer of Business B) — these are two independent rows in two independent tables, tied together only by the same `users.id`." No JWT-level separation of "staff context" vs "customer context" is introduced for this — the JWT's `business_id` claim already disambiguates which business a request is scoped to (`identity-module-design.md` § Business Context & Portal Access); whether that business relationship is a `staff_membership` or a `business_customers` row is resolved by the module handling that request, not by the identity layer.

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

**Decision:** Option A — same `ApplicationUser`, no separate `customer_accounts` table.

**Reason:** Unambiguous in `customer-module-design.md` §2, Tier 1: "`users` (`ApplicationUser`) ... Same table Identity already uses for owners and staff — no separate 'customer account' table. A customer registering is just a normal Identity registration with no business role attached yet." Reinforced by §3, "Explicitly NOT a Third Account Type": "There is still only **one** identity system ... This is deliberate: it keeps the 'one account per email' rule from the Identity module intact ... and avoids a customer ever needing multiple passwords for multiple businesses." (This also settles D4 above by the same reasoning — one account, any combination of roles.)

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

**Decision (guest without account):** Yes — a booking/order can exist with no customer account. `business_customers.user_id` is nullable by design (`customer-module-design.md` §2, §4.1) — `NULL` is a first-class "guest" state, not a workaround. (Note: the row is `business_customers`, not `guest_profiles` as this question assumed — that table name belongs to the superseded `customer-identity-design.md` model.)

**Decision (claim flow):** V1, not deferred. Fully specified end-to-end: `GET /api/customers/me/claimable` finds unlinked profiles matching the customer's *verified* phone/email, `POST /api/customers/me/claim/{businessCustomerId}` links it, and every write is audited in `business_customer_merge_log` (§6.1, §7, §9, §11). The security rules are explicit — match only verified Identity fields, re-check `status = 'guest'` inside the linking transaction, never silently auto-merge on an email match — so this isn't a stub deferred to Phase 2, it's a built flow with its own test cases (§17).

---

## Decision Log

| # | Decision | Status | Decided |
|---|----------|--------|---------|
| D1 | Customer account scope | RESOLVED — Option A (global account), data still business-scoped | `customer-module-design.md` §2, §8.4 |
| D2 | Anonymous vs login | RESOLVED — guest-capable for one-shot actions, login for persisted-state actions | `customer-module-design.md` §1, §6 |
| D3 | Customer login method | DERIVED — Option A (email + password), by consequence of D5 | `identity-module-design.md` § Registration/APIs |
| D4 | Same person, two roles | RESOLVED — Option A (one account, two independent rows) | `customer-module-design.md` §14 |
| D5 | Customer identity system | RESOLVED — Option A (same `ApplicationUser`, no separate table) | `customer-module-design.md` §2–3 |
| D6 | Guest vs customer account | RESOLVED — guest allowed, claim flow built in V1 | `customer-module-design.md` §2, §6.1, §11 |
