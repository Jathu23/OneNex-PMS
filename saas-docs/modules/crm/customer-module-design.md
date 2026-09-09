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
| `user_id` | uuid | YES | FK `users.id` | Linked global identity; NULL = guest |
| `full_name` | varchar(150) | NO | — | Captured/display name |
| `phone` | varchar(20) | YES | E.164 | Contact phone, if captured |
| `email` | varchar(254) | YES | — | Contact email, if captured |
| `source` | varchar(20) | NO | CHECK `walk_in/self_registered/imported` | How this profile originated |
| `status` | varchar(20) | NO | CHECK `guest/linked` | Link state |
| `first_branch_id` | uuid | YES | FK `branches.id` | Branch where first captured (analytics only) |
| `notes` | varchar(1000) | YES | — | Staff-visible free-text notes |
| `marketing_opt_in` | boolean | NO | DEFAULT `false` | Consent to marketing contact |
| `linked_at` | timestamptz | YES | — | When `user_id` was attached |
| `linked_via` | varchar(30) | YES | `self_claim/auto_on_interaction` | How the link happened |
| `created_by_user_id` | uuid | YES | FK `users.id` | Staff who captured this (NULL if self-created via platform) |
| `created_at` | timestamptz | NO | — | Created timestamp |
| `updated_at` | timestamptz | NO | — | Last change |

At least one of `phone` / `email` must be present — a profile with neither is not contactable and not useful (enforced at application layer, since a `CHECK` across nullable OR is awkward to keep readable in raw SQL but is straightforward in EF Core / a domain invariant).

### Constraints

```sql
UNIQUE (business_id, user_id)                         -- one profile per business per linked account
                                                        -- (partial: WHERE user_id IS NOT NULL)

UNIQUE (business_id, phone)                            -- one profile per business per phone
                                                        -- (partial: WHERE phone IS NOT NULL)

UNIQUE (business_id, email)                            -- one profile per business per email
                                                        -- (partial: WHERE email IS NOT NULL)

INDEX (user_id)                                        -- "which businesses know me" lookups

INDEX (business_id, status)                            -- staff-facing customer list, filter by linked/guest
```

### Why Business-Scoped Uniqueness, Not Global

The same phone number legitimately appears in `business_customers` once per business — a person can be a guest of Grand Hotel and, separately, a guest of Bella Salon, and those are two independent rows. This mirrors `staff_memberships`' `UNIQUE(user_id, business_id)` — the relationship is always scoped to one business, never global.

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

    status              varchar(20) NOT NULL
        CHECK (status IN ('guest', 'linked')),

    first_branch_id     uuid REFERENCES branches(id),

    notes               varchar(1000),
    marketing_opt_in    boolean NOT NULL DEFAULT false,

    linked_at           timestamptz,
    linked_via          varchar(30)
        CHECK (linked_via IN ('self_claim', 'auto_on_interaction')),

    created_by_user_id  uuid REFERENCES users(id),

    created_at          timestamptz NOT NULL DEFAULT now(),
    updated_at          timestamptz NOT NULL DEFAULT now(),

    CONSTRAINT chk_status_user_id CHECK (
        (status = 'linked' AND user_id IS NOT NULL) OR
        (status = 'guest'  AND user_id IS NULL)
    ),

    CONSTRAINT chk_has_contact CHECK (
        phone IS NOT NULL OR email IS NOT NULL
    )
);

CREATE UNIQUE INDEX uq_business_customer_user
    ON business_customers(business_id, user_id)
    WHERE user_id IS NOT NULL;

CREATE UNIQUE INDEX uq_business_customer_phone
    ON business_customers(business_id, phone)
    WHERE phone IS NOT NULL;

CREATE UNIQUE INDEX uq_business_customer_email
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
  2a. Found → return existing profile (guest or linked — staff doesn't
      need to know or care which; the order attaches to this profile
      either way)
  2b. Not found → INSERT business_customers
        (business_id, phone, full_name, source='walk_in',
         status='guest', user_id=NULL,
         created_by_user_id=<staff user>, first_branch_id=<branch>)
      → CustomerCapturedEvent

Order/booking flow then references business_customer_id, not user_id.
Kamal never sees onenex.ai. No account was created. No email/password
was required.
```

Six months later, Kamal registers on OneNex directly (unrelated reason — maybe a friend told him about it) and verifies his phone number through Identity's normal flow. He now has a global `users` row with `PhoneNumberConfirmed = true` for `+94771234567`.

```text
GET /api/customers/me/claimable   (JWT_1)

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
       SET user_id = Kamal, status = 'linked',
           linked_at = now(), linked_via = 'self_claim'
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

POST /api/businesses/{bellaSalonId}/customers/attach   (JWT_1)
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
        → found a guest row (maybe a friend gave the salon her number
          once)?
            → link it in place (same rules as §6.1's claim step),
              linked_via = 'auto_on_interaction'
        → not found?
            → INSERT business_customers
                (business_id, user_id=Priya.userId,
                 full_name=Priya.name, phone=Priya.phone,
                 email=Priya.email, source='self_registered',
                 status='linked', linked_at=now())

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
- Write a `business_customer_merge_log` row for every link, regardless of path.
- Re-check `status = 'guest'` immediately before linking, inside the same transaction — two concurrent claims (or a claim racing a staff edit) must not both succeed.
- Publish `CustomerAccountLinkedEvent` so operation modules that cache "is this a guest or linked customer" (if any ever do) get invalidated.

### Never

- Never link two `business_customers` rows to the same `user_id` within one business (enforced by `uq_business_customer_user`) — if a duplicate is discovered (e.g. two guest rows with different phone numbers that turn out to be the same person), that is a manual merge/support operation, not an automatic one. V1 does not attempt automatic duplicate-guest detection beyond the phone/email uniqueness constraint.
- Never treat a `business_customers.phone`/`email` as verified just because it is stored — it is only ever "what the business was told."
- Never let the Dining/Stays/etc. modules query `business_customers` directly — they hold a `business_customer_id` FK and go through `ICustomerService` for anything beyond that ID.
- Never expose another business's customer list through `/api/customers/me/*` — those endpoints are scoped to "businesses that have a profile linked to me," never a directory of other people.

---

## 8. Module Boundary — `ICustomerService`

```csharp
public interface ICustomerService
{
    // Staff/POS-facing — capture or find a customer within one business
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

Operation modules (Dining, Stays, ...) call `LookupOrCreateAsync` / `GetAsync` only — they never see `user_id`, linking state, or claimable logic. Whether a customer is a guest or linked is a CRM concern, invisible to an order.

---

## 9. Domain Events

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
```

---

## 10. API Shape

| Method | Endpoint | Auth | Purpose |
|---|---|---|---|
| POST | `/api/businesses/{businessId}/customers/lookup` | JWT_2, `crm:customers:create` | Staff/POS: find-or-create by phone/email (Scenario 1 capture) |
| GET | `/api/businesses/{businessId}/customers` | JWT_2, `crm:customers:view` | Staff-facing customer list/search |
| GET | `/api/businesses/{businessId}/customers/{id}` | JWT_2, `crm:customers:view` | Profile detail |
| PUT | `/api/businesses/{businessId}/customers/{id}` | JWT_2, `crm:customers:update` | Edit name/notes/tags/opt-in |
| DELETE | `/api/businesses/{businessId}/customers/{id}` | JWT_2, `crm:customers:delete` | Soft-delete (GDPR-style; never hard-delete a row referenced by order history) |
| POST | `/api/businesses/{businessId}/customers/attach` | JWT_1 (platform flow) | Internal call from booking/order flow (Scenario 2) — not staff-facing |
| GET | `/api/customers/me/claimable` | JWT_1 | Customer: list unlinked profiles matching my verified contact info |
| POST | `/api/customers/me/claim/{businessCustomerId}` | JWT_1 | Customer: link a guest profile to myself |
| GET | `/api/customers/me/businesses` | JWT_1 | Customer: "where have I got history" — powers a OneNex-wide order/booking history screen |

`crm:customers:*` permission codes already exist in the Membership module's seeded permission catalog (`Custom_RBAC.md` §17) — this module consumes them, it does not define its own permission scheme.

---

## 11. Caching

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

## 12. Entity Relationships

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
  └── business_customers            0..N per business, UNIQUE per phone/email

Dining/Stays/etc. orders, bookings, folios
  └── business_customer_id FK       (never user_id directly)
```

---

## 13. Relationship to Other Modules

| Module | Relationship |
|---|---|
| **Identity** | Source of the global `user_id` and the *verified* phone/email that linking depends on. This module never writes to Identity tables and never verifies contact info itself — it only reads verification state through `IIdentityService`. |
| **Business** | Source of `business_id` / `branch_id`. `first_branch_id` is informational only, resolved through `IBusinessService`, never joined directly. |
| **Membership** | Structurally parallel (`business_customers` mirrors `staff_memberships`) but functionally unrelated — a customer profile carries no `business_role`, no operation access, no permissions. A person can simultaneously be `staff_memberships` (owner of Business A) and `business_customers` (a guest customer of Business B) — these are two independent rows in two independent tables, tied together only by the same `users.id`. |
| **Dining / Stays / other operation modules** | Consume `ICustomerService.LookupOrCreateAsync` / `GetAsync` when creating an order/booking. They store `business_customer_id`, never `user_id`, so that guest→linked transitions never require rewriting order history. |
| **Notification** | Consumes `CustomerCapturedEvent` / `CustomerAccountLinkedEvent` if/when marketing or transactional messaging is layered on top (opt-in gated by `marketing_opt_in`). |

---

## 14. Project Structure

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
        └── MyCustomerProfilesController.cs (JWT_1, customer-facing)
```

---

## 15. V1 Scope

| Table | V1 |
|---|---|
| `business_customers` | Build |
| `business_customer_merge_log` | Build |
| `customer_tags` | Build (simple free-text label only) |
| Loyalty points / tiers | Out of scope — V1 has `marketing_opt_in` only, no points ledger |
| Automatic duplicate-guest detection (fuzzy name match, phone typos) | Out of scope — exact phone/email match only |

---

## 16. Mandatory Test Cases

| Test | Expected |
|---|---|
| Walk-in captured with phone only | `business_customers` row created, `status=guest`, `user_id=NULL` |
| Same phone captured twice at same business | Second call returns existing row, no duplicate |
| Same phone captured at two different businesses | Two independent rows, no conflict |
| Customer registers on OneNex, phone matches an existing guest row at Business A | Row appears in `GET /customers/me/claimable` |
| Customer claims a guest profile | `status` → `linked`, `user_id` set, merge log written, order history unchanged (same `business_customer_id`) |
| Customer attempts to claim a profile with an unverified phone | Rejected — matching requires `PhoneNumberConfirmed = true` |
| Two concurrent claim requests for the same guest profile | Only one succeeds; the other gets a conflict/already-linked response |
| Platform booking by an already-registered customer, no prior guest row | New `business_customers` row created directly as `linked`, `source=self_registered` |
| Platform booking by an already-registered customer, matching guest row exists | Existing row is linked in place (`linked_via=auto_on_interaction`), not duplicated |
| Staff searches customers by partial name/phone | Returns matches scoped to that business only |
| Attempt to create a profile with neither phone nor email | Rejected (`chk_has_contact`) |
| Delete (soft) a customer profile referenced by existing orders | Profile hidden from staff list; orders retain the FK and still resolve |

---

## 17. Open Questions

- Should `GetClaimableProfiles` be surfaced proactively (a notification/banner: "you have visit history to claim") or only on-demand when the customer opens a "link my history" screen? V1 leans on-demand to avoid a background matching job.
- Loyalty points/tiers: separate module (`Loyalty`) once needed, or absorbed into this one? Leaning separate module, consuming `business_customer_id` as its key, same pattern as operation modules.
- Should a business be able to *merge two guest profiles it owns* (e.g. staff realizes "John S." and "J. Silva" are the same person, both unlinked)? Manual merge tooling is not in V1 — flag as a support/ops task for now.
- Phone number recycling (a number is reassigned to a new person years later) is not handled — a stale guest row could theoretically be claimed by the wrong new owner of an old number. Low-probability, not addressed in V1; revisit if it becomes a real complaint.
- Does `marketing_opt_in` need to be per-channel (email vs SMS) from V1, or is a single flag sufficient until a real campaign feature exists?
- Should `business_customers.full_name` sync from Identity's `users.Name` after linking (keeping the two in sync), or stay independently editable per business (a business might want to record "Mr. Silva" while the platform-wide name is "K. Silva")? Leaning independently editable — same reasoning as staff display names vs account names elsewhere in the product.
