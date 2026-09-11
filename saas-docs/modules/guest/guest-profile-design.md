# Guest Profile — Design Document
> Module: Guest (Business Level — Cross-Operation)
> Status: Final | Version: 1.0
> Used by: Stays, Dining, Wellness, Events, Bar — all operations under a Business

---

## 1. What Problem Does This Solve

Every business needs to answer one question at every customer interaction:

```
"Have you been here before?"
```

Without guest profiles:

```
Customer walks in for the 5th time.
Staff asks: "Name please?"
Customer: "Kasun. I've stayed here 4 times."
Staff has no record. Starts from scratch. Every. Single. Time.

No VIP awareness → regular treatment for a high-value customer
No blacklist     → problem guest walks in, no warning
No preferences   → same questions asked every visit
No history       → "Is this your first time?" asked to a 10-visit guest
```

Guest Profile solves this. One record per customer per business. Built once, used everywhere.

---

## 2. Scope — What This Module Is and Is Not

```
IS: Core guest identity at Business level
  → Who is this customer?
  → Have they been here before?
  → Are they VIP? Blacklisted?
  → What's their visit history?
  → How do we link them to a login account later?

IS NOT: Operation-specific customer data
  → Room preferences (Stays module owns this)
  → Dietary restrictions (Dining module owns this)
  → Treatment preferences (Wellness module owns this)
  → Loyalty points calculation (Loyalty module owns this)
  → Document capture / passport scan (Stays module — legal requirement)
```

---

## 3. Where Guest Profile Lives

```
Owner
  └── Business (Grand Hotel Colombo)
        ├── guest_profiles ← lives here. Business level.
        │     Mr. Kasun Perera — VIP, 12 visits
        │     Ms. Priya Fernando — 3 visits
        │     Mr. James Silva — Blacklisted
        │
        ├── Stays Operation    → uses guest_profile_id FK
        ├── Dining Operation   → uses guest_profile_id FK
        └── Wellness Operation → uses guest_profile_id FK
```

One customer → one guest_profile per business → used by all operations.

Not one profile per operation. Not global across all businesses. **Per business.**

---

## 4. The 4 Entities

### Entity 1: `GuestProfile` — Core Record

```
id                    UUID PK
business_id           FK → Business. Profile belongs to this business only.

-- Identity
first_name            varchar(100)
last_name             varchar(100)
phone                 varchar(20). Primary lookup key. Staff always searches by phone.
email                 varchar(200) nullable.
nationality           varchar(100) nullable. "Sri Lankan" / "British" / "Indian"
date_of_birth         date nullable.
gender                enum nullable. MALE / FEMALE / OTHER / PREFER_NOT_TO_SAY

-- Account Link (Phase 2 — Guest Portal)
customer_account_id   UUID nullable FK → customer_accounts
                      null  = walk-in or phone booking guest. No login. V1.
                      set   = guest has created a OneNex customer account. Phase 2.
                      When set: guest can log in and see their own history.

-- Status Flags
is_vip                bool default false.
vip_notes             text nullable.
                      "CEO of MAS Holdings. Always upgrade if suite available."
                      "Top corporate client — LKR 2M+ annual spend."

is_blacklisted        bool default false.
blacklist_reason      text nullable. Required when is_blacklisted = true.
                      "Property damage Oct 2025 — Room 204 TV broken."
                      "Payment fraud — bounced cheque Nov 2024."
blacklisted_by        UUID nullable FK → Staff. Who flagged this.
blacklisted_at        timestamp nullable.

-- Aggregated Stats (auto-updated by system on each visit/booking)
total_visits          int default 0. Incremented per stay/visit/reservation.
total_spend           decimal default 0. Lifetime revenue from this guest.
last_visited_at       timestamp nullable. Last interaction with any operation.

-- Data Consent & GDPR
consent_source        enum.
                      STAFF_CREATED    → staff entered details. Guest present.
                      BOOKING_IMPORT   → created via online booking (no direct consent).
                      SELF_REGISTERED  → guest created their own account.

gdpr_anonymized_at    timestamp nullable.
                      Set when guest requests data erasure.
                      PII fields cleared. Record kept for audit + financial history.

created_at / updated_at
```

**Why phone is the primary lookup key:**
```
Email → guests give wrong email. Type errors common.
Name  → "Kasun" could be 5 different people.
Phone → unique per person in practice. Staff dials to confirm.
        +94771234567 → one person. Fast lookup at front desk.
```

---

### Entity 2: `GuestOccasion` — Special Dates

Used across all operations. Hotel arranges flowers. Restaurant prepares cake. Spa does complimentary upgrade.

```
id
guest_profile_id      FK → GuestProfile

occasion_type         enum.
                      BIRTHDAY     → auto-flag on the date each year
                      ANNIVERSARY  → auto-flag on the date each year
                      HONEYMOON    → flag on check-in (one-time)
                      OTHER        → custom occasion

occasion_date         date. For BIRTHDAY / ANNIVERSARY: date in any year.
                      System checks: month + day match today = flag.

occasion_year         int nullable. For HONEYMOON / OTHER: exact year matters.
notes                 text nullable. "Bring roses — allergic to lilies."

created_at
```

**How front desk sees this:**
```
Check-in screen for Mr. Kasun (Sep 10):
  ⚠️  ANNIVERSARY TODAY (Sep 10)
      Notes: "12th anniversary. Arrange cake."

Front desk arranges. Guest surprised. Loyalty built.
```

---

### Entity 3: `GuestNote` — Staff Notes

Freeform notes staff add over time. Not structured. Not searchable by field. Just memory.

```
id
guest_profile_id      FK → GuestProfile
added_by              FK → Staff
added_at              timestamp

note_type             enum.
                      PREFERENCE    → "Likes extra towels. Dislikes strong AC."
                      COMPLAINT     → "Complained about noise in room 302. Sep 2025."
                      POSITIVE      → "Left glowing review. Mentioned Meena at front desk."
                      GENERAL       → catch-all

content               text. Free text. No length limit.
is_internal           bool. true = staff-only. false = shown to guest on portal (Phase 2).
```

**Why GuestNote is separate from GuestProfile.vip_notes:**
```
vip_notes    = permanent, important, always shown. "CEO of MAS Holdings."
GuestNote    = timestamped log. Multiple notes over time. Full history.
               "Oct 2025: Complained about AC. Resolved."
               "Dec 2025: Left positive review."
               "Jan 2026: Requested early check-in, was accommodated."
```

---

### Entity 4: `GuestMergeLog` — Duplicate Tracking

When staff accidentally creates two profiles for same person, merge is needed.

```
id
business_id
merged_from_id        FK → GuestProfile. The duplicate (now inactive).
merged_into_id        FK → GuestProfile. The surviving record.
merged_by             FK → Staff. Who performed the merge.
merged_at             timestamp.
reason                text nullable. "Same phone, different name spelling."
```

**Merge logic:**
```
Staff finds: "Kasun Perera" (profile A, 3 visits) and "K. Perera" (profile B, 2 visits)
Same phone number. Same person. Duplicate.

Merge action:
  1. All bookings/orders from profile B → reassigned to profile A
  2. profile A.total_visits = 3 + 2 = 5
  3. profile A.total_spend  = combined
  4. profile B.is_active = false (soft delete)
  5. GuestMergeLog created
  6. customer_account_id from B linked to A (if exists)

Result: One clean profile. Full history preserved.
```

---

## 5. Duplicate Detection — Finding Returning Customers

Staff types phone number at check-in / booking creation.

```
STEP 1: Exact phone match
  Search: guest_profiles WHERE business_id = X AND phone = input AND is_active = true

  Found 1 result → show profile immediately. Returning guest.
  Found 0 results → go to Step 2.

STEP 2: Fuzzy name match (when no phone given)
  Search: guest_profiles WHERE business_id = X
          AND (first_name ILIKE input OR last_name ILIKE input)

  Found results → show list. Staff selects correct person.
  Found 0 results → new customer.

STEP 3: Possible duplicate warning (before creating new)
  Before creating new profile, system checks:
  → Same phone already exists? → "Possible match: Kasun Perera. Same profile?"
  → Staff confirms same person → link / merge
  → Staff confirms different person → create new (rare: different person, same phone)
```

**Why this matters:**
```
Without duplicate detection:
  Same guest → 3 profiles after 3 staff interactions
  total_visits = 1 each. VIP flag missing on 2.
  No consolidated history.

With duplicate detection:
  New profile creation always checks first.
  Staff prompted before creating duplicate.
  Clean data from day 1.
```

---

## 6. VIP & Blacklist — Rules

### VIP

```
Who can set:   Any staff with guest_modify permission
How to set:    Toggle on guest profile + add vip_notes
What happens:  VIP badge shown at all touchpoints
               Check-in screen: prominent "VIP" banner
               Booking list: VIP icon
               Night briefing: "3 VIP arrivals today"

No auto-VIP in V1. Manual flag only.
Phase 2: auto-VIP scoring based on total_spend / visit_frequency.
```

### Blacklist

```
Who can set:   Manager permission only (not regular staff)
               requires blacklist_reason to be filled (cannot save empty)

What happens at booking attempt:
  Staff creates booking for blacklisted guest →
  ⛔ "This guest is blacklisted."
      Reason: Property damage Oct 2025 — Room 204 TV broken.
      Blacklisted by: Meena (Manager) on Oct 22, 2025.
      [Proceed with Manager Override PIN] [Cancel Booking]

  Manager override: requires 6-digit PIN. Logged in audit.

Cross-operation:
  Blacklisted guest tries to reserve restaurant table →
  Same warning shown. Blacklist is business-wide, not operation-specific.

Remove blacklist:
  Manager only. Reason required. Logged.
```

---

## 7. Visit History — How It Works

```
NOT a separate table. Queried from operation modules.

GET /guests/{id}/history
  → Stays:    SELECT * FROM stays.bookings WHERE guest_profile_id = X
  → Dining:   SELECT * FROM dining.reservations WHERE guest_profile_id = X
  → Wellness: SELECT * FROM wellness.appointments WHERE guest_profile_id = X
  → Combined, sorted by date DESC

Staff sees:
  "Mr. Kasun Perera — Visit History"

  Sep 10, 2026 → Hotel Stay (Deluxe King, 3 nights, LKR 54,000)  [CURRENT]
  Aug 22, 2026 → Restaurant dinner (Table 5, LKR 8,400)
  Jul 15, 2026 → Hotel Stay (Suite, 2 nights, LKR 70,000)
  Jul 16, 2026 → Spa treatment (Deep Tissue, LKR 12,000)
  ...

  TOTAL: 12 visits | LKR 456,000 lifetime spend | Last: Sep 10, 2026
```

**total_visits and total_spend on GuestProfile:**
```
These are CACHED aggregates. Not calculated on every view.
Updated by system after each booking/order is completed.

Why cache: Profile page loads fast. No heavy join query on every open.
Trade-off: May be 1 visit behind in rare cases. Acceptable for display.
```

---

## 8. GDPR & Privacy

```
ROLE:
  guest_profiles = Business's CRM data.
  Business = Data Controller. OneNex = Data Processor.
  DPA (Data Processing Agreement) required with every business.

LAWFUL BASIS:
  Staff-created profile at booking → "Contract" (guest booked a service)
  Marketing preferences → "Consent" (explicit checkbox)

ANONYMIZATION (not hard delete):
  Guest requests data erasure →
    first_name → "Deleted"
    last_name  → "Guest"
    phone      → null
    email      → null
    nationality, date_of_birth, gender → null
    vip_notes, blacklist_reason → cleared
    gdpr_anonymized_at → NOW()

    KEEP: total_visits, total_spend, last_visited_at (non-PII, for analytics)
    KEEP: bookings / orders / invoices (financial records — 7 year legal minimum)
          but guest_name on those records → "Deleted Guest"

RETENTION:
  Active guest → keep indefinitely
  No visit > 3 years → auto-archive (notify business)
  No visit > 7 years → auto-anonymize PII
```

---

## 9. Phase 2 — Customer Account Link

```
V1 (now):
  guest_profiles.customer_account_id = null
  Guest has no login. Staff manages profile entirely.
  Self-service via OTP action tokens only (cancel/modify link in SMS).

Phase 2 (Guest Portal):
  Guest creates account on customer.onenex.com
  System detects: "Phone +94771234567 found at 2 businesses"
  Guest confirms → customer_account_id linked on their guest_profiles
  Guest can now:
    → View their own history
    → Update preferences
    → Make bookings online
    → View folio

  nullable FK already in schema from V1.
  No migration needed when Phase 2 is built.
```

---

## 10. How Guest Profile Connects to Every Module

```
STAYS MODULE (C5 extension)
  → stays_guest_preferences: bed type, floor, smoking (structured — queryable)
  → stays_guest_documents: passport / NIC (legal requirement for hotel stays)
  → Booking.guest_profile_id FK
  → Check-in screen shows: occasions, vip_notes, GuestNotes

DINING MODULE
  → dining_guest_preferences: dietary restrictions, seating preference, allergies
  → Reservation.guest_profile_id FK
  → Table assignment considers seating preference

WELLNESS MODULE
  → wellness_guest_preferences: therapist preference, pressure level, health notes
  → Appointment.guest_profile_id FK

ALL OPERATIONS
  → Any booking/order/reservation references guest_profile_id
  → VIP flag visible at all touchpoints
  → Blacklist warning fires across all operations
  → total_visits / total_spend updated after each completed interaction

IDENTITY MODULE
  → customer_accounts: Phase 2 login (guest_profiles.customer_account_id FK)
  → guest_action_tokens: OTP self-service (cancel/modify without login)
```

---

## 11. API Surface

```
CORE PROFILE
──────────────────────────────────────────────────────────────────
GET    /guests/search                    Search by phone / name
POST   /guests                           Create new guest profile
GET    /guests/{id}                      Get full profile
PATCH  /guests/{id}                      Update profile
GET    /guests/{id}/history              Cross-operation visit history

VIP & BLACKLIST
──────────────────────────────────────────────────────────────────
POST   /guests/{id}/vip                  Set VIP flag + notes
DELETE /guests/{id}/vip                  Remove VIP flag
POST   /guests/{id}/blacklist            Blacklist (manager only + reason)
DELETE /guests/{id}/blacklist            Remove blacklist (manager only + reason)

OCCASIONS & NOTES
──────────────────────────────────────────────────────────────────
GET    /guests/{id}/occasions            Get all occasions
POST   /guests/{id}/occasions            Add occasion
PATCH  /guests/{id}/occasions/{oid}      Edit occasion
DELETE /guests/{id}/occasions/{oid}      Remove occasion

GET    /guests/{id}/notes                Get all staff notes
POST   /guests/{id}/notes                Add note
DELETE /guests/{id}/notes/{nid}          Remove note (own notes only)

DUPLICATE MANAGEMENT
──────────────────────────────────────────────────────────────────
GET    /guests/duplicate-check           Check before creating (phone lookup)
POST   /guests/merge                     Merge two profiles (manager only)
GET    /guests/{id}/merge-history        See if profile was merged

GDPR
──────────────────────────────────────────────────────────────────
POST   /guests/{id}/anonymize            GDPR erasure (manager only)
GET    /guests/{id}/export               Data export for guest (JSON/CSV)
```

---

## 12. V1 Scope

```
✅ MUST BUILD (V1):
  GuestProfile entity (all fields including customer_account_id nullable)
  GuestOccasion entity
  GuestNote entity
  GuestMergeLog entity
  Phone-based duplicate detection at creation
  VIP flag + notes (any staff)
  Blacklist flag + reason (manager only + manager PIN override at booking)
  Cross-operation blacklist warning
  Soft GDPR anonymization
  total_visits / total_spend auto-update
  Full API surface above

PHASE 2:
  → customer_account_id linking (Guest Portal login)
  → Auto-VIP scoring based on spend thresholds
  → Auto-archive / auto-anonymize jobs (retention policy)
  → Right to rectification request flow
  → Breach notification system
  → Advanced search (by nationality, by date range, by spend)
```
