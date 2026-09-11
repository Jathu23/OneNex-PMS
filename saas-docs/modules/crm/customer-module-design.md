# OneNex CRM / Customer Module — Design Document

> Status: DRAFT — Needs team review before implementation.
> Architecture: Modular Monolith + Clean Architecture + CQRS + DDD (consistent with Membership and Business modules)
> Primary stack: ASP.NET Core + EF Core + PostgreSQL + Redis + MediatR

---

## 1. Problem Statement

OneNex end-customers (guests, diners, hotel guests, retail buyers — not staff) reach a business through two different paths, and the data model must support both without forcing one into the other:

```text
Scenario 1 — Walk-in
  A customer walks into a business that is already registered on OneNex.
  Staff captures their details (name, phone, maybe email) directly at the
  counter/front-desk — e.g. for a receipt, a loyalty card, a table booking.
  The customer never touches onenex.ai and has no global OneNex login.

Scenario 2 — Platform-first
  A customer registers on OneNex directly (same global registration as
  Identity already defines), browses/discovers businesses through the
  platform, and interacts with a business (books, orders) through it.
  A global identity exists before any single business ever sees them.
```

Both must converge on the same underlying customer, without either path being blocked by the other:

- Scenario 1 must not force a walk-in through email verification/password creation just to get a receipt.
- Scenario 2 must not create a second, disconnected identity every time the same person interacts with a new business.
- If the *same person* does both — walk in today, register on OneNex next month — their history should be linkable, not duplicated forever.

This module owns that model. It sits alongside Membership as the second consumer of Identity's global user, but for customers instead of staff.

---

## 2. Final Model — Two Tiers

```text
Tier 1 — Global Identity (owned by Identity module)
  users (ApplicationUser)
  → "Who is this person, globally, if they have ever created a OneNex account?"
  → Same table Identity already uses for owners and staff — no separate
    "customer account" table. A customer registering is just a normal
    Identity registration with no business role attached yet.

Tier 2 — Business-Scoped Customer Profile (owned by THIS module)
  business_customers
  → "What does Business A know about this person, and is it linked to
     a global account?"
  → One row per (business, person) — mirrors staff_memberships' shape
    (UNIQUE per business) but is NOT staff_memberships and carries no
    RBAC. A customer profile has no business_role, no operation access,
    no permissions.
```

```text
User
  ↓ (optional — may not exist yet)
Global Identity (users.id)
  ↓ (0..N — one per business they've interacted with)
Business Customer Profile (business_customers)
  ↓
Business-local data: name, phone, email, notes, tags, loyalty
  ↓
Order / Booking / Folio history (owned by Dining/Stays/etc., FK'd to
business_customers.id — never to users.id directly)
```

`business_customers.user_id` is **nullable**. That nullability is the entire design:

```text
user_id = NULL   → "guest" profile. Captured by a business (walk-in),
                    no global OneNex account attached (yet, or ever).

user_id = <uuid> → "linked" profile. Tied to a real Identity account,
                    either because the customer registered through the
                    platform (Scenario 2, linked at creation) or because
                    a guest profile was later claimed (Scenario 1 → linked).
```

---

## 3. Module Responsibility

### Owns

- Business-scoped customer profiles (`business_customers`)
- Walk-in capture / upsert-by-contact-info flow
- Linking a guest profile to a global Identity account ("claim")
- Customer search/lookup within a business (for POS/front-desk)
- Customer tags, notes, marketing opt-in (V1 minimal fields)
- Loyalty program data (V1: opt-in flag only; point balances are a future phase)

### Does NOT own

- User authentication, password, email/phone verification → **Identity module**
- Business identity, branches → **Business module**
- Staff, roles, permissions, RBAC → **Membership module** (a customer is never a `staff_membership` row)
- Orders, bookings, folios themselves → **respective operation modules** (they hold a `business_customer_id` FK, this module does not know order/booking details)
- Sending the actual verification/claim emails/SMS → **Notification module**

### Explicitly NOT a Third Account Type

There is still only **one** identity system (Identity module's `users` table). "Business-specific customer account" does not mean a second login — it means a second *record*, scoped to a business, that may or may not be linked to that one login. This is deliberate: it keeps the "one account per email" rule from the Identity module intact (see `identity-module-design.md` → Registration) and avoids a customer ever needing multiple passwords for multiple businesses.

---

## 4. Database Design

| Table | Purpose | V1 |
|---|---|---|
| `business_customers` | Business ↔ person relationship, guest or linked | Required |
| `business_customer_merge_log` | Audit trail of guest→linked claims | Required |
| `customer_tags` | Optional labels (VIP, allergy, blacklist) — simple V1 | Recommended |

Existing `users` (Identity) and `businesses` / `branches` (Business module) are assumed to belong to their respective modules.

---

### 4.1 `business_customers`

This is the **tenant-scoped customer record** — the CRM equivalent of `staff_memberships`, but with no role/permission semantics.

| Field | Type | Null | Key / Rule | Description |
|---|---|---|---|---|
| `id` | uuid | NO | PK | Profile identifier |
| `business_id` | uuid | NO | FK `businesses.id` | Owning business (tenant) |
| `user_id` | uuid | YES | FK `users.id` | Linked global identity; NULL = guest — the one true source of link state |
| `full_name` | varchar(150) | NO | — | Captured/display name |
| `phone` | varchar(20) | YES | E.164, UNIQUE per business | Contact number — one profile per phone within a business; see below for the trade-off this accepts |
| `email` | varchar(254) | YES | indexed (not unique) | Contact/matching signal, if captured — never a claim of identity, unlike phone (see below) |
| `source` | varchar(20) | NO | CHECK `walk_in/self_registered/imported` | How this profile originated |
| `status` | varchar(10) | NO | `GENERATED ALWAYS AS (...) STORED` | `guest`/`linked` — computed from `user_id`, never written directly (see below) |
| `first_branch_id` | uuid | YES | FK `branches.id` | Branch where first captured (analytics only) |
| `notes` | varchar(1000) | YES | — | Staff-visible free-text notes |
| `marketing_opt_in` | boolean | NO | DEFAULT `false` | Consent to marketing contact |
| `linked_at` | timestamptz | YES | — | When `user_id` was attached |
| `linked_via` | varchar(30) | YES | `self_claim/auto_on_interaction` | How the link happened |
| `created_by_user_id` | uuid | YES | FK `users.id` | Staff who captured this (NULL if self-created via platform) |
| `created_at` | timestamptz | NO | — | Created timestamp |
| `updated_at` | timestamptz | NO | — | Last change |

At least one of `phone` / `email` must be present — a profile with neither is not contactable and not useful (enforced at application layer, since a `CHECK` across nullable OR is awkward to keep readable in raw SQL but is straightforward in EF Core / a domain invariant).

### Identity Model — Two Separate Notions of "Unique"

`business_customers.id` is the stable identity of this customer *within this
business* — always present, never null, never derived. `user_id` is an
*optional* pointer to a global OneNex identity (Identity module's `users`
table). The same global user can hold several business-customer identities,
one per business they've interacted with:

```text
                 users
                  U123
                   │
          ┌────────┴─────────┐
          ▼                  ▼
   business_customers   business_customers
       BC001                BC002
   Grand Hotel          Beach Resort
```

`user_id` is a hard uniqueness key *per business* (`UNIQUE(business_id,
user_id)` below) because Identity has already verified it — two rows
can't both claim to be the same verified account at the same business.
`phone` now gets the same treatment: `UNIQUE(business_id, phone)`. This
is a deliberate simplification, not a claim that a phone number is a
verified identity the way `user_id` is — it is *what the business was
told*, same as before. The trade-off it accepts: two different real
people can legitimately share a phone (family members, a front-desk
typo, a number reassigned after the original owner released it), and
with a hard constraint they can no longer both hold independent profiles
at the same business — the second person's capture resolves to the
first person's existing row instead of creating a new one. This is
accepted for V1 in exchange for a deterministic find-or-create at
capture time, with no ambiguous-match flow to build or for staff to
navigate (see §6.1). Revisit if it causes real complaints — tracked as
an open question, §18.

`email` does **not** get the same treatment and stays a soft matching
signal only — indexed for fast lookup, never enforced as a
database-level identity guarantee, because nothing about email needs
the same walk-in determinism phone does (email is rarely what a
front-desk capture keys off).

### Constraints

```sql
UNIQUE (business_id, user_id)                         -- one profile per business per LINKED account —
                                                        -- safe because Identity already verified this value
                                                        -- (partial: WHERE user_id IS NOT NULL)

UNIQUE (business_id, phone)                            -- one profile per business per phone number —
                                                        -- see "Identity Model" above for the accepted trade-off
                                                        -- (partial: WHERE phone IS NOT NULL)

INDEX  (business_id, email)                            -- matching signal for lookup — NOT unique,
                                                        -- see "Identity Model" above for why

INDEX (user_id)                                        -- "which businesses know me" lookups

INDEX (business_id, status)                            -- staff-facing customer list, filter by linked/guest
                                                        -- (status is a generated column — see DDL — this index
                                                        -- works exactly like an index on any stored column)
```

### Why `user_id` Uniqueness Is Business-Scoped, Not Global

The same global user legitimately gets a `business_customers` row once per
business — a person can be a guest of Grand Hotel and, separately, a guest
of Bella Salon, and those are two independent rows, both `UNIQUE(business_id,
user_id)`-satisfying because `business_id` differs. This mirrors
`staff_memberships`' `UNIQUE(user_id, business_id)` — the relationship is
always scoped to one business, never global. `UNIQUE(business_id, phone)`
follows the identical shape — the same phone number is fine at two
different businesses, never enforced across them. (Email is the only
field left out of this uniqueness argument — it stays a matching signal,
scoped or not, per the section above.)

---

### 4.2 `business_customer_merge_log`

Records every guest→linked claim, for support/dispute resolution ("why does my order history suddenly include someone else's visit?" should never happen, but must be auditable if it's ever questioned).

| Field | Type | Null | Description |
|---|---|---|---|
| `id` | uuid | NO | PK |
| `business_customer_id` | uuid | NO | FK `business_customers.id` |
| `user_id` | uuid | NO | The account the profile was linked to |
| `matched_on` | varchar(20) | NO | `phone` / `email` |
| `linked_via` | varchar(30) | NO | `self_claim` / `auto_on_interaction` |
| `created_at` | timestamptz | NO | When the link happened |

```sql
INDEX (business_customer_id)
INDEX (user_id)
```

---

### 4.3 `customer_tags` (V1 — simple)

```sql
CREATE TABLE customer_tags (
    id                     uuid PRIMARY KEY,
    business_customer_id   uuid NOT NULL REFERENCES business_customers(id) ON DELETE CASCADE,
    label                  varchar(50) NOT NULL,
    created_by_user_id     uuid NOT NULL REFERENCES users(id),
    created_at             timestamptz NOT NULL,

    CONSTRAINT uq_customer_tag UNIQUE (business_customer_id, label)
);
```

Free-text labels (`VIP`, `Allergy: peanuts`, `Do not seat window`) rather than a controlled catalog — V1 does not need a taxonomy. Revisit if reporting/filtering by tag becomes a real requirement.

---

## 5. PostgreSQL DDL Baseline

```sql
CREATE TABLE business_customers (
    id                  uuid PRIMARY KEY DEFAULT gen_random_uuid(),

    business_id         uuid NOT NULL REFERENCES businesses(id),
    user_id             uuid REFERENCES users(id),

    full_name           varchar(150) NOT NULL,
    phone               varchar(20),
    email               varchar(254),

    source              varchar(20) NOT NULL
        CHECK (source IN ('walk_in', 'self_registered', 'imported')),

    status              varchar(10) GENERATED ALWAYS AS (
                            CASE WHEN user_id IS NULL THEN 'guest' ELSE 'linked' END
                        ) STORED,
    -- Computed, not written. There is exactly one source of truth for
    -- link state (user_id) — status is a read-only projection of it, so
    -- "user_id set but status says guest" is not a state the database can
    -- even represent, let alone one application code has to keep in sync.

    first_branch_id     uuid REFERENCES branches(id),

    notes               varchar(1000),
    marketing_opt_in    boolean NOT NULL DEFAULT false,

    linked_at           timestamptz,
    linked_via          varchar(30)
        CHECK (linked_via IN ('self_claim', 'auto_on_interaction')),

    created_by_user_id  uuid REFERENCES users(id),

    created_at          timestamptz NOT NULL DEFAULT now(),
    updated_at          timestamptz NOT NULL DEFAULT now(),

    CONSTRAINT chk_has_contact CHECK (
        phone IS NOT NULL OR email IS NOT NULL
    )
);

-- Hard identity guarantee: Identity already verified this user_id, so at
-- most one profile per business may claim it.
CREATE UNIQUE INDEX uq_business_customer_user
    ON business_customers(business_id, user_id)
    WHERE user_id IS NOT NULL;

-- Hard identity guarantee, same shape as uq_business_customer_user: one
-- profile per business per phone number. Accepted trade-off: two
-- different real people sharing a phone (family members, a recycled
-- number) can no longer hold independent profiles at the same business —
-- see "Identity Model" (§4.1) for why this was chosen over the ambiguous
-- multi-match flow it replaces.
CREATE UNIQUE INDEX uq_business_customer_phone
    ON business_customers(business_id, phone)
    WHERE phone IS NOT NULL;

-- Matching signal, NOT an identity guarantee — deliberately not UNIQUE.
-- Two different real people can share an email far less often than a
-- phone in practice, but the same risk applies in principle; kept soft
-- since nothing about email needs walk-in determinism. See "Identity
-- Model" (§4.1) and the Lookup flow (§6.2) for how a multi-match lookup
-- is resolved.
CREATE INDEX ix_business_customer_email
    ON business_customers(business_id, email)
    WHERE email IS NOT NULL;

CREATE INDEX ix_business_customers_user
    ON business_customers(user_id);

CREATE INDEX ix_business_customers_business_status
    ON business_customers(business_id, status);
```

```sql
CREATE TABLE business_customer_merge_log (
    id                    uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    business_customer_id  uuid NOT NULL REFERENCES business_customers(id),
    user_id               uuid NOT NULL REFERENCES users(id),
    matched_on            varchar(20) NOT NULL CHECK (matched_on IN ('phone', 'email')),
    linked_via            varchar(30) NOT NULL,
    created_at            timestamptz NOT NULL DEFAULT now()
);

CREATE INDEX ix_merge_log_business_customer
    ON business_customer_merge_log(business_customer_id);

CREATE INDEX ix_merge_log_user
    ON business_customer_merge_log(user_id);
```

---

## 6. The Two Scenarios, End to End

### 6.1 Scenario 1 — Walk-in

```text
Kamal walks into Grand Hotel's restaurant. Staff takes his phone number
for the receipt/table booking.

POST /api/businesses/{grandHotelId}/customers/lookup
  { phone: "+94771234567", fullName: "Kamal" }

Server:
  1. Search business_customers WHERE business_id = grandHotel
       AND phone = "+94771234567"
     (phone is now unique per business — see §4.1's Identity Model — so
     this returns at most one row; no ambiguous-match case to handle)
  2a. Match found → return it (guest or linked — staff doesn't need to
      know or care which; the order attaches to this profile either
      way). If the phone happens to be shared by a different real person
      (family members, a recycled number), this is where that trade-off
      lands: the existing profile is what's returned, not a new one —
      see the accepted trade-off in §4.1 and the open question in §18.
  2b. No match → INSERT business_customers
        (business_id, phone, full_name, source='walk_in',
         user_id=NULL,   -- status computes to 'guest' automatically
         created_by_user_id=<staff user>, first_branch_id=<branch>)
      → CustomerCapturedEvent

Order/booking flow then references business_customer_id, not user_id.
Kamal never sees onenex.ai. No account was created. No email/password
was required.
```

Six months later, Kamal registers on OneNex directly (unrelated reason — maybe a friend told him about it) and verifies his phone number through Identity's normal flow. He now has a global `users` row with `PhoneNumberConfirmed = true` for `+94771234567`.

```text
GET /api/customers/me/claimable   (JWT, no business_id)

Server:
  SELECT business_customers
  WHERE status = 'guest'
    AND phone = <Kamal's VERIFIED phone from Identity>
       OR email = <Kamal's VERIFIED email from Identity>

  → returns [ { businessId: grandHotel, businessName: "Grand Hotel",
                lastVisit-ish info } ]

Kamal sees "Looks like you've visited Grand Hotel before — is this you?"
→ confirms → POST /api/customers/me/claim/{businessCustomerId}

Server:
  1. Re-verify the profile is still status='guest' (no race with someone
     else claiming it, or staff editing the phone in the meantime)
  2. Re-verify the matched phone/email still belongs to this Identity
     user AND is verified (never link on an unverified contact field)
  3. UPDATE business_customers
       SET user_id = Kamal,   -- status flips to 'linked' automatically
           linked_at = now(), linked_via = 'self_claim'
       WHERE id = businessCustomerId
  4. INSERT business_customer_merge_log
  5. Publish CustomerAccountLinkedEvent
  → Kamal's order/booking history at Grand Hotel is now visible under
    his OneNex account, with no data migration needed (orders were
    always FK'd to business_customer_id, which didn't change).
```

### 6.2 Scenario 2 — Platform-first

```text
Priya registers on OneNex directly (POST /auth/register — Identity
module, unchanged). She verifies her email. She has a global account
and has never visited any business.

She browses onenex.ai, finds "Bella Salon", and books an appointment.

POST /api/businesses/{bellaSalonId}/customers/attach   (JWT, no business_id)
  (called internally by the booking flow, not by Priya directly)

Server:
  1. Search business_customers WHERE business_id = bellaSalon
       AND user_id = Priya.userId
  2a. Found → use it (she's booked here before)
  2b. Not found:
        Search WHERE business_id = bellaSalon
          AND (phone = Priya's verified phone OR email = Priya's
               verified email)
          AND status = 'guest'
        (phone alone can match at most one row now — §4.1 — so any
        ambiguity here can only come from email, or from phone and email
        independently matching two different existing rows)
        → found EXACTLY ONE guest row (maybe a friend gave the salon her
          number once)?
            → link it in place (same rules as §6.1's claim step),
              linked_via = 'auto_on_interaction'
        → found MORE THAN ONE guest row (e.g. her phone matches one
          guest row while her email separately matches a different one)?
            → do NOT guess which one is really her — fall through to
              "not found" below and create a fresh linked profile.
              The ambiguous guest row(s) stay unlinked, discoverable later
              through her own explicit /customers/me/claimable + claim
              flow (§6.1), which has a confirmation step this silent
              auto-link path deliberately doesn't.
        → not found (or ambiguous, per above)?
            → INSERT business_customers
                (business_id, user_id=Priya.userId,
                 full_name=Priya.name, phone=Priya.phone,
                 email=Priya.email, source='self_registered',
                 linked_at=now())   -- status computes to 'linked' automatically

  → CustomerCapturedEvent (or CustomerAccountLinkedEvent if case 2a-linked)

Bella Salon's booking now references business_customer_id. Bella Salon
staff see Priya as a normal linked customer in their CRM screen — they
don't need to know or care that she came from the platform rather than
walking in.
```

Note the `auto_on_interaction` linking path in 6.2 (2a-linked) is the one deliberate case where linking happens without an explicit "is this you?" confirmation click — it is safe specifically *because* it only matches on **already-verified** Identity contact fields, and the customer is the one actively initiating the interaction (placing their own booking), not a bystander. Compare this to the never-allowed case in §7.

---

## 7. Linking Rules — Security

### Always

- Match only against **verified** Identity fields (`EmailConfirmed = true` / `PhoneNumberConfirmed = true`). A staff-typed phone number at a POS terminal is never itself proof of ownership — the proof comes from Identity having already verified that the *claiming* account owns that phone/email.
- Auto-link (`linked_via = 'auto_on_interaction'`, §6.2) only when a phone/email match resolves to **exactly one** guest row. More than one candidate means "ambiguous," never "pick the newest" or any other silent heuristic — see §6.2.
- Write a `business_customer_merge_log` row for every link, regardless of path.
- Re-check `status = 'guest'` immediately before linking, inside the same transaction — two concurrent claims (or a claim racing a staff edit) must not both succeed.
- Publish `CustomerAccountLinkedEvent` so operation modules that cache "is this a guest or linked customer" (if any ever do) get invalidated.

### Never

- Never link two `business_customers` rows to the same `user_id` within one business (enforced by `uq_business_customer_user`, a hard constraint because Identity has already verified that value) — if a duplicate is discovered (e.g. two guest rows with different phone numbers that turn out to be the same person), that is a manual merge/support operation, not an automatic one.
- Never treat a matching `email` as proof two guest rows are the same person, and never force them into one row via a database uniqueness constraint — email stays an indexed matching signal *precisely because* it is not reliable identity (see §4.1's Identity Model). `phone` is the one exception, by deliberate choice: it is hard-unique per business, so a phone match *is* the same row by construction, accepting the family-sharing/recycling trade-off documented in §4.1 rather than resolving it via a candidate-picker flow. V1 does not attempt automatic fuzzy duplicate-guest detection or auto-merge beyond an exact email match surfaced for a human to resolve.
- Never treat a `business_customers.phone`/`email` as verified just because it is stored — it is only ever "what the business was told."
- Never let the Dining/Stays/etc. modules query `business_customers` directly — they hold a `business_customer_id` FK and go through `ICustomerService` for anything beyond that ID.
- Never expose another business's customer list through `/api/customers/me/*` — those endpoints are scoped to "businesses that have a profile linked to me," never a directory of other people.
- Never allow one business (or its staff) to query, export, or view another business's `business_customers` rows — there is no cross-business query path in `ICustomerService`, and none should exist without the explicit, separately-consented feature described in §8.

---

## 8. Data Privacy & Tenant Isolation

### 8.1 Core Principle

Each business is the **data controller** for the customer relationship it creates — the customer gave *that business* their name/phone/email, under *that business's* terms (in-store receipt, booking form, whatever the business's own privacy notice says). OneNex is the **data processor**: it hosts the infrastructure `business_customers` lives in, but it does not thereby become a second party with an independent right to use that data.

This is the same tenant-isolation principle already applied to authorization in the Membership module (`Custom_RBAC.md` §13) and to business identity in the Business module — extended here to the data itself, not just access to it.

```text
Business A's customer relationship  ≠  OneNex's customer relationship
Business B's customer relationship  ≠  Business A's customer relationship
```

### 8.2 No Cross-Business Sharing, By Default

**Businesses cannot see each other's customers.** There is no API, no report, and no admin screen in this design that lets Business A list, search, export, or match against Business B's `business_customers` rows. This is enforced structurally, not just by policy:

```text
uq_business_customer_phone   → UNIQUE (business_id, phone)
ix_business_customer_email   → INDEX  (business_id, email)
uq_business_customer_user    → UNIQUE (business_id, user_id)
```

`business_id` leads every one of these — indexed or unique, none of them span businesses. The same phone number produces two independent rows at two businesses — never one shared row, and no code path joins across `business_id`. `ICustomerService` (§9) takes a `businessId` on every call; there is no method that omits it. (The isolation here comes from `business_id` being part of the key, not from uniqueness — see §4.1's Identity Model for why phone/email specifically dropped the uniqueness property while keeping the business-scoping.)

If a genuine business need for cross-business matching ever arises (see §8.4), it must be a new, explicit, opt-in feature — never a side-effect of shared infrastructure.

### 8.3 OneNex Platform Access — Operational, Not a Product Feature

OneNex, as the company operating the platform, necessarily has *technical* database access (it runs the servers). That is different from OneNex *using* customer data as a product feature or business asset. This design treats them as separate:

| Access type | Allowed | Governance |
|---|---|---|
| Engineering/support accessing a specific record to fix a reported bug | Yes | Least-privilege, time-boxed, logged — same as any production incident access |
| A platform admin dashboard listing every business's customers in one place, for OneNex's own use | No | Not part of this design; would require a separate legal basis and disclosure to businesses/customers |
| OneNex using business_customers PII for its own marketing (e.g. emailing Grand Hotel's guests about a OneNex promotion) | No | Business's customer data is not OneNex's to market with — `marketing_opt_in` is consent to *that business*, not to OneNex (see §8.5) |
| OneNex using **anonymized/aggregated** data (e.g. "average customers per business by category") for its own analytics or investor reporting | Yes, with care | Must not be re-identifiable; not a feature of this module — a reporting/analytics concern layered on top, out of scope here |
| A customer's own global account data (name, email, phone, login history) in the Identity module | Yes — that's OneNex's own user | This module's `business_customers` is distinct from Identity's `users`; OneNex's relationship is with the *account*, not with what a business recorded about that person |

In short: OneNex can operate the system a business's customer data lives in; it does not get to treat that data as its own dataset.

### 8.4 Common Ownership Does Not Imply Sharing

Sample data elsewhere in these docs shows the same owner (Abi) running two businesses — Grand Hotel and City Apartments. Even here, `business_customers` stays isolated per `business_id`. A customer who is a guest of Grand Hotel is **not** automatically visible to City Apartments just because they share an owner — the customer's relationship was with Grand Hotel specifically, and the customer never consented to City Apartments having their details.

```text
Grand Hotel      business_customers row for Kamal   →   Grand Hotel only
City Apartments  (no row for Kamal, even though Abi owns both)
```

A future "franchise network" or "loyalty network" feature that intentionally shares customers across commonly-owned or partnered businesses is not ruled out — but it must be:

- **Opt-in by the customer** (a specific consent checkbox, not inherited from either business's general terms), and
- **Opt-in by both businesses** (a business shouldn't have its customer list exposed to a sibling business without agreeing to it either).

This is flagged as an open question (§18), not decided here.

### 8.5 Consent Is Scoped to the Business That Collected It

`business_customers.marketing_opt_in` (§4.1) is per-row — i.e. per business. A customer opting in to Bella Salon's marketing has not opted in to Grand Hotel's, even if both profiles are linked to the same OneNex account. There is no platform-wide opt-in flag in this design, deliberately — introducing one would blur exactly the boundary this section exists to keep clear.

### 8.6 Right to Access / Erasure

Two distinct erasure requests can arrive, and they resolve differently:

```text
"Delete my OneNex account" (Identity-level request)
    → Identity module's existing soft-delete/anonymization (see
      identity-module-design.md → Account Suspension / Deletion)
    → business_customers.user_id rows referencing this account are
      NOT cascade-deleted — they revert to being unlinked-in-effect
      (the business's own record of that visit/order history stands
      on its own, same as a guest row always did); the linkage itself
      is what's severed, not the business's data
    → business_customer_merge_log rows are retained (audit trail of a
      link that once existed), consistent with financial/audit records
      needing a user reference even after deletion (see Identity's own
      rationale for keeping anonymized records)

"Delete my data at Business X specifically" (business-level request)
    → A request to that one business — it is the controller for that
      relationship. Handled the same way Identity handles it for a
      full account: anonymize business_customers row (name → "Deleted
      Customer", phone/email → null, notes cleared), never hard-delete
      if orders/invoices reference business_customer_id for financial
      record-keeping. Does not touch that customer's other businesses
      or their global OneNex account.
```

### 8.7 Enforcement Recommendation

Same posture as Membership and Business modules: application-layer scoping (every query includes `business_id`, every service method requires it) is the primary control. PostgreSQL Row-Level Security can be added as defense in depth once the application-layer tenant context is stable and tested — not a substitute for it (see `Custom_RBAC.md` §13 for the equivalent decision already made for authorization data).

---

## 9. Module Boundary — `ICustomerService`

```csharp
public interface ICustomerService
{
    // Staff/POS-facing — capture or find a customer within one business.
    // `phone` is UNIQUE per business (§4.1's Identity Model), so this is
    // a deterministic find-or-create — never an ambiguous match:
    Task<BusinessCustomerDto> LookupOrCreateAsync(
        Guid businessId,
        string? phone,
        string? email,
        string fullName,
        Guid? branchId,
        Guid? capturedByUserId,
        CancellationToken cancellationToken = default);

    Task<BusinessCustomerDto?> GetAsync(
        Guid businessCustomerId,
        CancellationToken cancellationToken = default);

    Task<IReadOnlyList<BusinessCustomerDto>> SearchAsync(
        Guid businessId,
        string query,
        CancellationToken cancellationToken = default);

    // Platform-facing — attach the logged-in user to a business
    Task<BusinessCustomerDto> AttachToBusinessAsync(
        Guid businessId,
        Guid userId,
        CancellationToken cancellationToken = default);

    // Customer self-service — claim a guest profile as their own
    Task<IReadOnlyList<ClaimableProfileDto>> GetClaimableProfilesAsync(
        Guid userId,
        CancellationToken cancellationToken = default);

    Task ClaimAsync(
        Guid businessCustomerId,
        Guid userId,
        CancellationToken cancellationToken = default);

    Task<IReadOnlyList<BusinessCustomerSummaryDto>> GetMyBusinessProfilesAsync(
        Guid userId,
        CancellationToken cancellationToken = default);
}
```

The staff-facing `/customers/lookup` endpoint (§11) calls `LookupOrCreateAsync` and always gets back exactly one profile — found or freshly created, never a candidate list to disambiguate, since `phone` being unique per business (§4.1) makes the match deterministic. Order/booking creation in Dining, Stays, etc. then calls `GetAsync` with that known `business_customer_id` — they never call `LookupOrCreateAsync` themselves and never see `user_id` or linking state. Whether a customer was a guest or linked at capture time is a CRM/front-desk concern, invisible to an order.

---

## 10. Domain Events

**Consumed:**

```text
UserPhoneVerifiedEvent / UserEmailVerifiedEvent (Identity)
    → optional V1+1 enhancement: recompute this user's claimable-profile
      list proactively (e.g. to power a "you have unclaimed history"
      notification) instead of only computing it on-demand when
      /api/customers/me/claimable is called. Not required for V1 — the
      on-demand query is sufficient to start.

BusinessCreatedEvent (Business)
    → no action needed; a business starts with zero customers.
```

**Published:**

```text
CustomerCapturedEvent        → analytics; Notification may send a
                                "thanks for visiting" flow depending on
                                business settings (not V1 default)
CustomerAccountLinkedEvent   → invalidate any cached guest/linked state;
                                audit
CustomerProfileUpdatedEvent  → staff edited name/notes/tags
GuestDeviceClaimedEvent       → introduced by guest-ordering-flow.md §12
                                (anonymous QR ordering's device-based
                                account-linking path) — consumed by the
                                ordering/Dining module to link its own
                                `orders` rows; this module never writes
                                `orders` directly
```

---

## 11. API Shape

| Method | Endpoint | Auth | Purpose |
|---|---|---|---|
| POST | `/api/businesses/{businessId}/customers/lookup` | JWT (business_id), `crm:customers:create` | Staff/POS: find-or-create by phone/email (Scenario 1 capture). Phone is unique per business (§4.1), so this always returns a single deterministic profile — found or freshly created, never a candidate list (§9). |
| GET | `/api/businesses/{businessId}/customers` | JWT (business_id), `crm:customers:view` | Staff-facing customer list/search |
| GET | `/api/businesses/{businessId}/customers/{id}` | JWT (business_id), `crm:customers:view` | Profile detail |
| PUT | `/api/businesses/{businessId}/customers/{id}` | JWT (business_id), `crm:customers:update` | Edit name/notes/tags/opt-in |
| DELETE | `/api/businesses/{businessId}/customers/{id}` | JWT (business_id), `crm:customers:delete` | Soft-delete (GDPR-style; never hard-delete a row referenced by order history) |
| POST | `/api/businesses/{businessId}/customers/attach` | JWT (no business_id, platform flow) | Internal call from booking/order flow (Scenario 2) — not staff-facing |
| GET | `/api/customers/me/claimable` | JWT (no business_id) | Customer: list unlinked profiles matching my verified contact info |
| POST | `/api/customers/me/claim/{businessCustomerId}` | JWT (no business_id) | Customer: link a guest profile to myself |
| GET | `/api/customers/me/businesses` | JWT (no business_id) | Customer: "where have I got history" — powers a OneNex-wide order/booking history screen |

`crm:customers:*` permission codes already exist in the Membership module's seeded permission catalog (`Custom_RBAC.md` §17) — this module consumes them, it does not define its own permission scheme.

---

## 12. Caching

Lighter than Membership's authorization cache — a customer profile isn't a security decision, so staleness tolerance is higher.

```text
Cache key:    customer-lookup:{businessId}:{phone-or-email}
Cache value:  business_customer_id (or "not_found" negative cache, short TTL)
TTL:          2 minutes — POS/front-desk repeatedly looks up the same
              handful of numbers in a shift; DB round-trip on every
              keystroke is wasteful, but this is not an authorization
              path so a short TTL is enough safety margin.

Invalidate on: CustomerCapturedEvent, CustomerAccountLinkedEvent,
               CustomerProfileUpdatedEvent (for that business+contact)
```

No L1/Redis two-tier cache is needed here (unlike Membership) — this is a convenience cache for a search box, not a per-request authorization check.

---

## 13. Entity Relationships

```text
users (Identity)
  │
  └── business_customers            0..N per user, UNIQUE(business_id, user_id)
        ├── business_id  → businesses (Business module)
        ├── first_branch_id → branches (Business module, nullable)
        ├── created_by_user_id → users (nullable, staff who captured)
        ├── customer_tags        (1:many)
        └── business_customer_merge_log  (1:many, append-only audit)

businesses
  └── business_customers            0..N per business, UNIQUE per phone; email stays a non-unique matching signal

Dining/Stays/etc. orders, bookings, folios
  └── business_customer_id FK       (never user_id directly)
```

---

## 14. Relationship to Other Modules

| Module | Relationship |
|---|---|
| **Identity** | Source of the global `user_id` and the *verified* phone/email that linking depends on. This module never writes to Identity tables and never verifies contact info itself — it only reads verification state through `IIdentityService`. |
| **Business** | Source of `business_id` / `branch_id`. `first_branch_id` is informational only, resolved through `IBusinessService`, never joined directly. |
| **Membership** | Structurally parallel (`business_customers` mirrors `staff_memberships`) but functionally unrelated — a customer profile carries no `business_role`, no operation access, no permissions. A person can simultaneously be `staff_memberships` (owner of Business A) and `business_customers` (a guest customer of Business B) — these are two independent rows in two independent tables, tied together only by the same `users.id`. |
| **Dining / Stays / other operation modules** | Consume `ICustomerService.GetAsync` with a `business_customer_id` already resolved by the staff-facing lookup step (§9) — they don't call `LookupOrCreateAsync` themselves. They store `business_customer_id`, never `user_id`, so that guest→linked transitions never require rewriting order history. |
| **Notification** | Consumes `CustomerCapturedEvent` / `CustomerAccountLinkedEvent` if/when marketing or transactional messaging is layered on top (opt-in gated by `marketing_opt_in`). |

---

## 15. Project Structure

```text
Modules/Crm/
├── Domain/
│   ├── Entities/
│   │   ├── BusinessCustomer.cs
│   │   ├── BusinessCustomerMergeLog.cs
│   │   └── CustomerTag.cs
│   └── Events/
│       ├── CustomerCapturedEvent.cs
│       ├── CustomerAccountLinkedEvent.cs
│       └── CustomerProfileUpdatedEvent.cs
│
├── Application/
│   └── Features/
│       ├── Customers/
│       │   ├── Commands/ LookupOrCreateCustomer, UpdateCustomer,
│       │   │             DeleteCustomer, AttachToBusiness
│       │   └── Queries/  SearchCustomers, GetCustomer
│       ├── Claiming/
│       │   ├── Commands/ ClaimBusinessCustomer
│       │   └── Queries/  GetClaimableProfiles, GetMyBusinessProfiles
│       └── Tags/
│           └── Commands/ AddTag, RemoveTag
│
├── Infrastructure/
│   ├── Repositories/
│   │   └── BusinessCustomerRepository.cs
│   ├── Caching/
│   │   └── CustomerLookupCacheService.cs
│   └── CrmDbContext.cs
│
└── API/
    └── Controllers/
        ├── CustomersController.cs          (business-scoped, staff-facing)
        └── MyCustomerProfilesController.cs (JWT, no business_id — customer-facing)
```

---

## 16. V1 Scope

| Table | V1 |
|---|---|
| `business_customers` | Build |
| `business_customer_merge_log` | Build |
| `customer_tags` | Build (simple free-text label only) |
| Loyalty points / tiers | Out of scope — V1 has `marketing_opt_in` only, no points ledger |
| Automatic duplicate-guest detection / auto-merge | Out of scope — `phone` matches resolve deterministically (unique per business, §4.1); exact `email` matches are surfaced only through the claim/auto-link flows (§6.1, §6.2), never silently merged. Fuzzy matching (name/typo) is also out of scope. |

---

## 17. Mandatory Test Cases

| Test | Expected |
|---|---|
| Walk-in captured with phone only | `business_customers` row created, `status` reads `guest` (computed), `user_id=NULL` |
| Same phone captured twice at same business, one existing row matches | Second call returns the existing row, no duplicate created |
| A second, different real person is captured with a phone already on file at that business (e.g. family members sharing a line) | Rejected as a new row by `uq_business_customer_phone`; the lookup resolves to the existing profile instead of creating a second one — the accepted trade-off, see §4.1 |
| Linking `user_id` to a `business_customers` row that already has a different `user_id` linked at that business | Rejected by `uq_business_customer_user` |
| Same phone captured at two different businesses | Two independent rows, no conflict |
| Customer registers on OneNex, phone matches an existing guest row at Business A | Row appears in `GET /customers/me/claimable` |
| Customer claims a guest profile | `status` reads `linked` (computed from `user_id`), merge log written, order history unchanged (same `business_customer_id`) |
| Customer attempts to claim a profile with an unverified phone | Rejected — matching requires `PhoneNumberConfirmed = true` |
| Two concurrent claim requests for the same guest profile | Only one succeeds; the other gets a conflict/already-linked response |
| Platform booking by an already-registered customer, no prior guest row | New `business_customers` row created directly as `linked`, `source=self_registered` |
| Platform booking by an already-registered customer, exactly one matching guest row exists | Existing row is linked in place (`linked_via=auto_on_interaction`), not duplicated |
| Platform booking by an already-registered customer, phone matches one existing guest row while email independently matches a different guest row at that business | Neither is auto-linked (ambiguous); a fresh `linked` profile is created instead, both guest rows remain unlinked and claimable later |
| Staff searches customers by partial name/phone | Returns matches scoped to that business only |
| Attempt to create a profile with neither phone nor email | Rejected (`chk_has_contact`) |
| Attempt to directly `UPDATE ... SET status = 'linked'` without setting `user_id` | Rejected by Postgres — `status` is a generated column and cannot be written to |
| Delete (soft) a customer profile referenced by existing orders | Profile hidden from staff list; orders retain the FK and still resolve |

---

## 18. Open Questions

- Should `GetClaimableProfiles` be surfaced proactively (a notification/banner: "you have visit history to claim") or only on-demand when the customer opens a "link my history" screen? V1 leans on-demand to avoid a background matching job.
- Loyalty points/tiers: separate module (`Loyalty`) once needed, or absorbed into this one? Leaning separate module, consuming `business_customer_id` as its key, same pattern as operation modules.
- Should a business be able to *merge two guest profiles it owns* (e.g. staff realizes "John S." and "J. Silva" are the same person, both unlinked)? Manual merge tooling is not in V1 — flag as a support/ops task for now.
- `phone` is now `UNIQUE(business_id, phone)` (§4.1) — accepted trade-off: two different real people sharing one phone at the same business (family members, or a number recycled to a new owner years later) can no longer hold independent profiles; the second person's capture, or a stale claim attempt, silently resolves to the first person's existing row. Low-probability, accepted for V1 in exchange for a deterministic capture flow with no candidate-picker UI to build; revisit (e.g. a manual "this is actually a different person, split the profile" action) if it becomes a real complaint.
- Does `marketing_opt_in` need to be per-channel (email vs SMS) from V1, or is a single flag sufficient until a real campaign feature exists?
- Should `business_customers.full_name` sync from Identity's `users.Name` after linking (keeping the two in sync), or stay independently editable per business (a business might want to record "Mr. Silva" while the platform-wide name is "K. Silva")? Leaning independently editable — same reasoning as staff display names vs account names elsewhere in the product.
- Cross-business sharing (§8.4): should OneNex ever offer an opt-in "franchise/loyalty network" feature letting commonly-owned or partnered businesses share customer profiles? Needs explicit product + legal sign-off (consent model, ToS changes) before design — not assumed anywhere else in this document.
- Terms of Service language: does OneNex's business-facing ToS currently state the controller/processor split described in §8.1, or does it need updating to match? This document assumes the split as the target model, not as an already-published legal fact.
