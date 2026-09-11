# Identity Module — DB Tables & API Reference

> Status: DERIVED — extracted from `identity-module-design.md`. Not a new
> design; this document adds no new decisions of its own. If the source
> document changes, this one is stale until re-synced.
>
> Purpose: a quick-reference list of every table Identity owns and every
> endpoint it exposes, without paging through the full ~60KB design doc.
> For how other modules (e.g. Customer/CRM) hang off `AspNetUsers.Id`,
> see `../crm/customer-db-api-reference.md` — that FK lives on their
> side, not this one.

---

## 1. Tables Owned by Identity

Source: `identity-module-design.md` § Entities.

### 1.1 `AspNetUsers` (extends ASP.NET Core `IdentityUser`)

| Field | Type | Null | Key / Rule | Description |
|---|---|---|---|---|
| `Id` | uuid | NO | PK | Global account id — every other module's `user_id` FK points here |
| `Email` | varchar | NO | UNIQUE (Identity-enforced) | |
| `NormalizedEmail` | varchar | NO | — | Identity default |
| `EmailConfirmed` | boolean | NO | — | Other modules may only trust this contact field once true |
| `PhoneNumber` | varchar | YES | UNIQUE (custom validation) | |
| `PhoneNumberConfirmed` | boolean | NO | — | Other modules may only trust this contact field once true |
| `PasswordHash` | varchar | NO | — | PBKDF2, Identity-managed |
| `SecurityStamp` | varchar | NO | — | Regenerates on password change; drives session invalidation (§ Fast Session Invalidation) |
| `LockoutEnd` / `LockoutEnabled` / `AccessFailedCount` | — | — | — | Identity lockout mechanics |
| `Name` | varchar(100) | NO | custom addition | Display name |
| `Status` | enum | NO | `active` / `suspended` / `deleted` | Custom addition |
| `CreatedAt` / `UpdatedAt` | timestamptz | NO | — | Custom addition |

### 1.2 `refresh_tokens`

| Field | Type | Null | Description |
|---|---|---|---|
| `id` | uuid | NO | PK |
| `user_id` | uuid | NO | FK `AspNetUsers.Id` |
| `family_id` | uuid | NO | Constant across a rotation chain (one login session) |
| `token_hash` | varchar(500) | NO | Hashed, never plain |
| `expires_at` | timestamptz | NO | 30 days from issue |
| `revoked_at` | timestamptz | YES | NULL = still valid |
| `revoked_reason` | varchar(100) | YES | `logout` / `rotation` / `suspicious` / `reuse_detected` |
| `device_info` | varchar(200) | YES | Descriptive only |
| `client_id` | varchar(50) | YES | e.g. `guest-web`, `staff-ios` |
| `ip_address` | varchar(45) | YES | |
| `created_at` | timestamptz | NO | |

### 1.3 `identity_security_audits`

| Field | Type | Null | Description |
|---|---|---|---|
| `id` | uuid | NO | PK |
| `user_id` | uuid | NO | FK `AspNetUsers.Id` |
| `event_type` | varchar(50) | NO | `LoginSucceeded`, `EmailVerified`, `AccountSuspended`, etc. |
| `ip_address` | varchar(45) | YES | |
| `user_agent` | varchar(500) | YES | |
| `metadata` | jsonb | YES | |
| `created_at` | timestamptz | NO | |

### 1.4 `api_keys` (forward reference only — owned by Business module)

Identity documents the shape but does not own or validate this table (see
`identity-module-design.md` § Machine-to-Machine — API Keys). Listed here
only so it isn't mistaken for an Identity- or Customer-owned table.

---

## 2. What Identity Exposes to Other Modules

```text
IIdentityService
  - IsEmailVerified(userId) / IsPhoneVerified(userId)
  - GetVerifiedContactInfo(userId)   (verified phone/email only)
```

Consumers (e.g. the Customer module's guest→linked claim flow) only ever
read through this interface — nothing queries `AspNetUsers` directly, and
Identity never writes to a consumer's tables.

---

## 3. API Endpoints — Identity

Source: `identity-module-design.md` § APIs.

| Method | Endpoint | Auth | Purpose |
|---|---|---|---|
| POST | `/auth/register` | none | Create global account (pending verification) |
| POST | `/auth/verify-email` | none | Confirm email, unlocks login |
| POST | `/auth/resend-verification` | none | Rate-limited (3/hour) |
| POST | `/auth/login` | none | Issue JWT + refresh token |
| POST | `/auth/refresh-token` | refresh token | Rotate JWT/refresh token |
| POST | `/auth/logout` | JWT | Revoke one session |
| POST | `/auth/logout-all` | JWT | Revoke all sessions (V1) |
| GET | `/auth/sessions` | JWT | List active sessions (Phase 2) |
| DELETE | `/auth/sessions/{id}` | JWT | Revoke one device (Phase 2) |
| POST | `/auth/forgot-password` | none | Always same response (no enumeration) |
| POST | `/auth/reset-password` | reset token | Also invalidates all sessions via SecurityStamp |
| GET | `/auth/me` | JWT | Own profile — `emailVerified`/phone-verified state consumed by other modules' claim/link flows |
| GET | `/auth/me/memberships` | JWT | Business/role list (Membership-backed) |
| POST | `/auth/select-business` | JWT | Reissue JWT with `business_id` (Membership context) |

There is exactly **one** JWT structure (see `identity-module-design.md` →
"JWT Design") — it is never a family of separate token types. Other docs'
tables mark which *state* of that one token an endpoint needs:
`JWT (no business_id)` = the token as issued by `/auth/login`, before any
business is selected; `JWT (business_id)` = the same token reissued with
`business_id` populated via `/auth/select-business`, required by any
business-scoped endpoint in another module.

---

## 4. Events Published by Identity

```text
UserPhoneVerifiedEvent
UserEmailVerifiedEvent
    → consumed optionally by other modules (e.g. Customer/CRM) to
      proactively recompute claim/link candidates
```

---

## 5. Note — the Older `customer-identity-design.md`

This folder also contains `customer-identity-design.md`, which defines a
different, earlier model (`guest_profiles`, `customer_accounts`,
`guest_sessions`, `guest_action_tokens`). That document predates and is
superseded by `../crm/customer-module-design.md`'s `business_customers`
model — confirmed by `identity-decisions.md`, whose D1/D4/D5/D6 are still
marked `PENDING` and are answered differently by the later doc (one
`ApplicationUser`, one business-scoped `business_customers` table, not a
second parallel identity system). Not reconciled here — flagged so the
two aren't accidentally read as consistent with each other.
