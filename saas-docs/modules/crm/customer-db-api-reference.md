# Customer Module (CRM) — DB Tables & API Reference

> Status: DERIVED — extracted from `customer-module-design.md`. Not a new
> design; this document adds no new decisions of its own. If the source
> document changes, this one is stale until re-synced.
>
> Purpose: a quick-reference list of every table the Customer/CRM module
> owns and every endpoint it exposes, without paging through the full
> ~47KB design doc. For the Identity-owned side of the FK these tables
> point to (`AspNetUsers.Id`), see
> `../identity/identity-db-api-reference.md` — that table lives on their
> side, not this one.

---

## 1. Tables Owned by the Customer Module

Source: `customer-module-design.md` §4–5.

### 1.1 `business_customers`

The tenant-scoped customer record — one row per (business, person).

| Field | Type | Null | Key / Rule | Description |
|---|---|---|---|---|
| `id` | uuid | NO | PK | |
| `business_id` | uuid | NO | FK `businesses.id` (Business module) | Tenant |
| `user_id` | uuid | YES | FK `AspNetUsers.Id` (Identity module); `UNIQUE(business_id, user_id)` partial | NULL = guest, set = linked |
| `full_name` | varchar(150) | NO | — | |
| `phone` | varchar(20) | YES | `UNIQUE(business_id, phone)` partial | |
| `email` | varchar(254) | YES | indexed, not unique | |
| `source` | varchar(20) | NO | CHECK `walk_in/self_registered/imported` | |
| `status` | varchar(10) | NO | generated column: `guest`/`linked` from `user_id` | |
| `first_branch_id` | uuid | YES | FK `branches.id` (Business module) | |
| `notes` | varchar(1000) | YES | | |
| `marketing_opt_in` | boolean | NO | DEFAULT false | |
| `linked_at` | timestamptz | YES | | When `user_id` was attached |
| `linked_via` | varchar(30) | YES | `self_claim`/`auto_on_interaction` | |
| `created_by_user_id` | uuid | YES | FK `AspNetUsers.Id` (Identity module) | Staff who captured this row |
| `created_at` / `updated_at` | timestamptz | NO | | |

Constraint: `chk_has_contact` — phone or email required.

### 1.2 `business_customer_merge_log`

Audit trail of every guest→linked claim.

| Field | Type | Null | Description |
|---|---|---|---|
| `id` | uuid | NO | PK |
| `business_customer_id` | uuid | NO | FK `business_customers.id` |
| `user_id` | uuid | NO | FK `AspNetUsers.Id` (Identity module) — the account the profile was linked to |
| `matched_on` | varchar(20) | NO | `phone` / `email` |
| `linked_via` | varchar(30) | NO | `self_claim` / `auto_on_interaction` |
| `created_at` | timestamptz | NO | |

### 1.3 `customer_tags`

| Field | Type | Null | Description |
|---|---|---|---|
| `id` | uuid | NO | PK |
| `business_customer_id` | uuid | NO | FK `business_customers.id` |
| `label` | varchar(50) | NO | Free-text (`VIP`, `Allergy: peanuts`) |
| `created_by_user_id` | uuid | NO | FK `AspNetUsers.Id` (Identity module) |
| `created_at` | timestamptz | NO | |

---

## 2. Linking Rules Against Identity (Summary)

Full detail in `customer-module-design.md` §7, "Linking Rules — Security".

- Writing `business_customers.user_id` may only match against **verified**
  Identity fields (`EmailConfirmed = true` / `PhoneNumberConfirmed =
  true`), read through `IIdentityService` (Identity module) — never by
  querying `AspNetUsers` directly.
- `AspNetUsers` soft-delete (Identity's own GDPR erasure) does **not**
  cascade-delete `business_customers` rows — the link is severed but the
  business's own record stands on its own (`customer-module-design.md`
  §8.6).

---

## 3. What the Customer Module Exposes

```text
ICustomerService
  (full signature in customer-module-design.md §9)
  - LookupOrCreateAsync(...)     staff/POS find-or-create
  - AttachToBusinessAsync(...)   platform booking flow
  - GetClaimableProfilesAsync(userId)
  - ClaimAsync(businessCustomerId, userId)
```

Other modules never query `business_customers` directly — they hold a
`business_customer_id` FK and go through this interface.

---

## 4. API Endpoints — Customer Module (CRM)

Source: `customer-module-design.md` §11.

| Method | Endpoint | Auth | Purpose |
|---|---|---|---|
| POST | `/api/businesses/{businessId}/customers/lookup` | JWT_2, `crm:customers:create` | Staff/POS find-or-create by phone/email |
| GET | `/api/businesses/{businessId}/customers` | JWT_2, `crm:customers:view` | Staff-facing list/search |
| GET | `/api/businesses/{businessId}/customers/{id}` | JWT_2, `crm:customers:view` | Profile detail |
| PUT | `/api/businesses/{businessId}/customers/{id}` | JWT_2, `crm:customers:update` | Edit name/notes/tags/opt-in |
| DELETE | `/api/businesses/{businessId}/customers/{id}` | JWT_2, `crm:customers:delete` | Soft-delete |
| POST | `/api/businesses/{businessId}/customers/attach` | JWT_1 | Internal — platform booking attaches logged-in user |
| GET | `/api/customers/me/claimable` | JWT_1 | List unlinked profiles matching my **verified** contact info (reads Identity via `IIdentityService`) |
| POST | `/api/customers/me/claim/{businessCustomerId}` | JWT_1 | Link a guest profile to myself (writes `business_customers.user_id` + `business_customer_merge_log`) |
| GET | `/api/customers/me/businesses` | JWT_1 | "Where have I got history" |

`JWT_1` = Identity's plain login token (no `business_id` claim). `JWT_2`
= the same token reissued with `business_id` via Identity's
`/auth/select-business` (see `../identity/identity-db-api-reference.md`
§3). Staff-facing endpoints above require `JWT_2`; customer-self-service
endpoints (`/api/customers/me/*`) require only `JWT_1`.

---

## 5. Events Published by the Customer Module

```text
CustomerCapturedEvent
CustomerAccountLinkedEvent
CustomerProfileUpdatedEvent
    → consumed by Notification/analytics; Identity does not consume these
      (it has no concept of a business-scoped customer profile)
```

---

## 6. Excluded From This Document

- Membership module tables (`staff_memberships`, `authorization_audits`,
  etc.) — structurally parallel to `business_customers` but carry RBAC,
  out of scope here. See `customer-module-design.md` §14.
- Business module tables (`businesses`, `branches`, `api_keys`) — only
  referenced here as FK targets, not detailed.
