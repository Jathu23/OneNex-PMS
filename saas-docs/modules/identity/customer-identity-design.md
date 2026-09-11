# Customer Identity — Design Document

> Status: DRAFT — Decisions finalized, implementation pending.

---

## Overview

OneNex has 3 surfaces. Customer identity applies to Surface 3.

```
Surface 1: Owner Portal      → ApplicationUser (owner/staff identity)
Surface 2: Operations Portal → ApplicationUser (owner/staff identity)
Surface 3: Customer Web/App  → Customer Identity (this document)
```

---

## Customer Web Model

ONE shared platform — not white-labeled per business.

```
customer.onenex.com
  /grand-hotel     ← QR or URL → this business's section
  /city-bistro
  /beach-resort

Customer signs up ONCE → interacts with any business on the platform.
Dashboard: visited businesses, upcoming bookings, deals, loyalty.
```

---

## 3 Identity Levels

```
Level 0: Anonymous
  → No identity captured
  → Actions: QR menu view, browse events
  → No account needed

Level 1: Guest (Lightweight)
  → Name + Phone + Email captured by staff or during booking
  → No password, no login
  → guest_profiles row created (per business)
  → Self-service via OTP link in SMS/email (cancel, modify)
  → Actions: room booking, table reservation, takeaway, event ticket

Level 2: Customer Account (Full)
  → OneNex account with login
  → customer_accounts row + linked guest_profiles
  → Actions: portal access, booking history, loyalty, delivery, folio view
```

---

## Two Data Layers

```
Layer 1: customer_accounts (OneNex Global — Optional)
  → Customer creates themselves on customer.onenex.com
  → OR accepts invite from booking confirmation email
  → One account works across ALL businesses on OneNex
  → Customer owns and controls this

Layer 2: guest_profiles (Per Business — Always)
  → Created by staff when customer interacts
  → That business's CRM data
  → Only that business can see it
  → Linked to customer_accounts via customer_account_id (nullable)
```

### Tables

```sql
customer_accounts
  id                    UUID        PK
  user_id               UUID        → AspNetUsers.Id (ApplicationUser)
  name                  VARCHAR(100)
  phone                 VARCHAR(20)  UNIQUE
  email                 VARCHAR(200) UNIQUE
  marketing_consent     BOOLEAN      DEFAULT FALSE
  marketing_consent_at  TIMESTAMP    NULLABLE
  deletion_requested_at TIMESTAMP    NULLABLE
  gdpr_anonymized_at    TIMESTAMP    NULLABLE
  status                ENUM         (active | deleted)
  created_at            TIMESTAMP

guest_profiles
  id                   UUID        PK
  business_id          UUID        → businesses.id
  customer_account_id  UUID        NULLABLE → customer_accounts.id
  name                 VARCHAR(100)
  phone                VARCHAR(20)
  email                VARCHAR(200)
  vip_flag             BOOLEAN      DEFAULT FALSE
  blacklist_flag       BOOLEAN      DEFAULT FALSE
  blacklist_reason     TEXT         NULLABLE
  preferences          JSONB        NULLABLE
  total_visits         INT          DEFAULT 0
  total_spend          DECIMAL
  last_visited_at      TIMESTAMP    NULLABLE
  consent_source       ENUM         (staff_created | self_registered | booking_import)
  gdpr_anonymized_at   TIMESTAMP    NULLABLE
  created_at           TIMESTAMP

guest_action_tokens
  id               UUID       PK
  guest_profile_id UUID       → guest_profiles.id
  token_hash       VARCHAR
  action_type      ENUM       (cancel_booking | modify_booking | confirm_booking)
  expires_at       TIMESTAMP
  used_at          TIMESTAMP  NULLABLE
  created_at       TIMESTAMP
```

---

## Staff Lookup Flow

```
Staff enters phone number in operations portal

→ Search guest_profiles WHERE business_id = THIS business AND phone = input

Found (returning customer):
  → Show name, history, preferences for THIS business only

Not found (new customer):
  → "New customer" → staff enters details manually
  → guest_profiles row created
  → customer_account_id = NULL

System NEVER searches global customer_accounts directly from staff portal.
Cross-business data is NEVER visible to staff.
```

---

## Account Linking Flow

```
Ravi visited Grand Hotel (Level 1 guest, no account)
  → guest_profiles row exists: business=Grand Hotel, customer_account_id=NULL

Ravi creates account on customer.onenex.com
  → customer_accounts row created
  → System detects: phone +94771234567 exists in guest_profiles at 2 businesses
  → "Link your existing bookings? Found records at: Grand Hotel, City Bistro"
  → Ravi confirms → guest_profiles.customer_account_id updated

Ravi's portal now shows:
  → Visits: Grand Hotel (3 stays), City Bistro (1 visit)
  → Upcoming bookings
  → Preferences (aggregated)
```

---

## Privacy Model

```
Customer sees:     ALL businesses they've visited (their dashboard)
Business A sees:   ONLY their own guest_profiles records
Business A CANNOT: see customer's history at Business B
Staff cannot:      browse global customer_accounts
```

---

## GDPR Compliance

### OneNex's Two Roles

```
Data Processor → for guest_profiles
  Business collects customer data → Business is Data Controller
  OneNex stores/processes on business's behalf
  Requirement: DPA (Data Processing Agreement) with every business

Data Controller → for customer_accounts
  Customer registers on OneNex platform directly
  OneNex is responsible for full GDPR compliance
  Requirement: Privacy policy, consent, data subject request handling
```

### Lawful Basis

```
guest_profile created by staff    → "Contract" (customer booked a service)
customer_account signup           → "Consent" (checkbox at registration)
Booking confirmation email        → "Contract" (no separate consent needed)
Marketing / promotional emails    → "Consent" (separate explicit checkbox)
```

### 6 Data Subject Rights

```
Right to Access:
  customer_accounts data → OneNex provides directly
  guest_profiles data    → customer contacts each business
                           OR OneNex aggregates (requires DPA clause)

Right to Erasure:
  customer_accounts → anonymize immediately
  guest_profiles    → anonymize PII (name, phone, email → null/"Deleted Guest")
  bookings/invoices → KEEP (financial records, 7 year legal minimum)
                      anonymize PII fields only

Right to Portability:
  Export: profile info, linked businesses, booking history
  Format: JSON or CSV download from customer portal

Right to Rectification:
  customer_accounts → customer updates directly on portal
  guest_profiles    → customer requests correction → business approves (Phase 2)

Right to Object (Marketing):
  marketing_consent flag → customer toggles off → no more promotional emails
  Transactional emails (booking confirm, OTP) → always sent regardless

Data Breach:
  Notify supervisory authority within 72 hours
  Notify affected customers if high risk
```

### Anonymization (Not Deletion)

```
On erasure request:

customer_accounts:
  name   → "Deleted User"
  email  → deleted_{uuid}@deleted.onenex.com
  phone  → null
  status → deleted

guest_profiles:
  name   → "Deleted Guest"
  email  → null
  phone  → null
  vip_flag, blacklist_reason, preferences → cleared
  customer_account_id → null
  KEEP: total_visits, total_spend, last_visited_at (non-PII analytics)

bookings / orders / invoices:
  guest_name    → "Deleted Guest"
  guest_contact → null
  KEEP: dates, amounts, room numbers, tax records (legal requirement)
```

### Data Retention Policy

```
guest_profiles (active)            → keep indefinitely
guest_profiles (no visit > 3 yrs)  → auto-archive, notify business
guest_profiles (no visit > 7 yrs)  → auto-anonymize PII

customer_accounts (active)         → keep indefinitely
customer_accounts (deleted)        → purge after 7 years

Booking / Invoice records          → keep 7 years minimum
Audit logs                         → keep 1 year minimum
Refresh tokens                     → 30 days (standard)
guest_action_tokens                → expires_at (short-lived, single-use)
```

### GDPR Request Tracking Table

```sql
data_subject_requests
  id            UUID
  customer_id   UUID       → customer_accounts.id
  request_type  ENUM       (access | erasure | portability | rectification)
  status        ENUM       (pending | in_progress | completed | rejected)
  requested_at  TIMESTAMP
  completed_at  TIMESTAMP  NULLABLE
  notes         TEXT       NULLABLE
```

---

## V1 Scope

```
Must Build:
  ✓ guest_profiles table (per business)
  ✓ customer_accounts table (global)
  ✓ guest_action_tokens (OTP-based self-service)
  ✓ Consent checkbox at customer_account signup
  ✓ Marketing consent separate field
  ✓ Soft delete / anonymize function
  ✓ Basic data export (customer_accounts + linked bookings)
  ✓ Privacy policy page on customer web
  ✓ DPA agreement template for businesses (legal, not code)

Phase 2:
  → Customer portal "My Data" section (view, export, delete request)
  → Business dashboard: GDPR requests received
  → Automated retention job (archive/anonymize inactive profiles)
  → Right to rectification request flow (customer → business approval)
  → Breach notification system
```

---

## Guest Session Implementation

### Approach: Guest Token = Signed JWT

> Reference: Proven in production (Booking-Backend / New Receipt Team project).

Guest token is a **lightweight signed JWT** — not a random string with hash in DB.

```
Why JWT (not random token + hash):
  ✓ Self-validating — signature check, no DB lookup needed
  ✓ Expiry built-in to token itself
  ✓ guest_id directly readable from claims
  ✓ Same middleware pattern as regular user JWT
  ✓ No hashing complexity on every request
```

### Token Structure

```json
Guest JWT payload:
{
  "guest_id": "uuid",
  "business_id": "uuid",
  "token_type": "guest",
  "iat": 1234567890,
  "exp": 1234567890   ← 7 days
}

client_id ("guest-web" / "guest-ios" / "guest-android") deliberately not included yet —
see identity-module-design.md § Platform-Specific ClientId, which adds the equivalent
concept to the logged-in ApplicationUser session (refresh_tokens.client_id) but leaves
the guest JWT as an open decision (D-below) since it's a different, DB-row-less token
shape with its own owner.

Signed with: RS256 (same key as access tokens)
Stored by frontend: localStorage per business
  Key: "guest_token_{business-slug}"
```

### Middleware Resolution

```
Every request → middleware checks in order:

1. Authorization: Bearer <token>
   → Valid JWT, token_type = access → IsAuthenticated() = true

2. X-Guest-Token: <token>
   → Valid JWT, token_type = guest  → IsGuest() = true
   → guest_id extracted from claims

3. Nothing
   → Anonymous (Level 0)

LoggedUser helper:
  IsAuthenticated() → JWT access token present + valid
  IsGuest()         → guest JWT present + valid
  GetGuestId()      → extract from guest JWT claims
```

### Controller Authorization

```csharp
[AllowAnonymous]   → menu view, event browse (Level 0)
[AllowGuest]       → cart, order, reserve (Level 1 + Level 2)
[Authorize]        → booking history, loyalty, folio (Level 2 only)
```

### First Request Flow

```
Guest opens customer.onenex.com/grand-hotel (no token in localStorage)

Frontend → GET /api/businesses/grand-hotel/menu
           (no header)

Backend:
  1. No token detected → anonymous request
  2. Check if endpoint allows anonymous → YES (menu view)
  3. Return menu data + issue guest JWT:

Response:
{
  "menu": [...],
  "guestToken": "eyJhbGc..."   ← signed JWT with guest_id + business_id
}

Frontend:
  → Save to localStorage["guest_token_grand-hotel"]
  → All subsequent calls → X-Guest-Token: eyJhbGc...
```

### Guest Session Table

```sql
guest_sessions
  id                   UUID        PK  DEFAULT gen_random_uuid()
  business_id          UUID        NOT NULL → businesses.id
  guest_profile_id     UUID        NULLABLE → guest_profiles.id
  customer_account_id  UUID        NULLABLE → customer_accounts.id
  last_active_at       TIMESTAMP   NOT NULL
  expires_at           TIMESTAMP   NOT NULL
  converted_at         TIMESTAMP   NULLABLE  ← set when guest logs in
  created_at           TIMESTAMP   NOT NULL

-- guest_id from JWT = guest_sessions.id (UUID embedded in token)
-- No token_hash needed — JWT signature handles validation

INDEX: id (PK, used for all lookups via JWT claim)
INDEX: customer_account_id (for migration query)
```

### Login / Migration Flow

```
Guest (has X-Guest-Token header) decides to login:

POST /auth/login
Header: X-Guest-Token: eyJhbGc...
Body:   { "email": "...", "password": "..." }

Backend:
  1. Validate credentials → find customer_account
  2. Read X-Guest-Token from header (middleware already parsed it)
  3. Extract guest_id from guest JWT claims
  4. Find guest_session by guest_id
  5. Migrate all associated data:
       UPDATE orders       SET customer_account_id = X WHERE guest_session_id = Y
       UPDATE reservations SET customer_account_id = X WHERE guest_session_id = Y
       UPDATE cart_items   SET customer_account_id = X WHERE guest_session_id = Y
       IF guest_session.guest_profile_id NOT NULL:
         UPDATE guest_profiles SET customer_account_id = X
  6. guest_session.converted_at = NOW()
  7. Return normal access JWT (Level 2)

No guestToken in request body — read from header always.
History fully preserved. Zero data loss.
```

### Per-Business Token Isolation

```
Same guest visits two businesses:

localStorage:
  "guest_token_grand-hotel"  → JWT (guest_id_A, business_id_A)
  "guest_token_city-bistro"  → JWT (guest_id_B, business_id_B)

Each business = separate guest_session row.
On login → BOTH sessions migrate to same customer_account.
```

### Two Token Types — Different Purposes

```
guest_sessions (JWT)         → self-initiated anonymous sessions
                               Customer scans QR, browses, orders
                               Frontend saves in localStorage
                               7 day expiry, refreshed on activity

guest_action_tokens (hashed) → staff-initiated OTP links
                               "Click here to cancel your booking" (in SMS)
                               Single-use, 24hr expiry
                               Never stored in frontend
```

---

## Open Decisions

| # | Decision | Status |
|---|----------|--------|
| D2 | QR table ordering — anonymous or phone OTP? | PENDING |
| D3 | Customer login method — email+pass? OTP? Google? | PENDING |
| D6 | Guest → Account upgrade flow (claim bookings) — V1 or Phase 2? | PENDING |
| D7 | Add `client_id` claim to guest JWT (guest-web/guest-ios/guest-android), mirroring `identity-module-design.md` § Platform-Specific ClientId — V1 or later? | PENDING |
