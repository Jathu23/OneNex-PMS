# Guest Module — Master Plan
> Status: Planning | Last Updated: 2026-09-10
> Module Type: Shared Always-On (not an operation — always active for every business)
> Philosophy: One guest identity per business. Used by every operation.

---

## What Is the Guest Module?

Every business on OneNex — hotel, restaurant, spa — needs to answer one question at every interaction:

```
"Have you been here before?"
```

Guest Module is the answer. It owns the identity of every customer who has ever interacted with a Business.

```
Owner
  └── Business (Grand Hotel Colombo)
        ├── Guest Module    ← always on. owns all guest records for this business.
        │     Mr. Kasun Perera  — VIP, 12 visits, LKR 456,000 lifetime
        │     Ms. Priya Fernando — 3 visits
        │     Mr. James Silva   — Blacklisted (property damage)
        │
        ├── Stays Operation    → uses guest_profile_id FK
        ├── Dining Operation   → uses guest_profile_id FK
        └── Wellness Operation → uses guest_profile_id FK
```

One customer → one `guest_profile` per business → used by ALL operations under that business.

---

## Why Shared Always-On (Not Operation-Specific)

| If Guest Profile were inside Stays... | Problem |
|---|---|
| Restaurant looks up returning guest | No access — profile belongs to Stays |
| VIP flag at check-in | Doesn't show at restaurant reservation |
| Blacklist warning | Only fires at hotel front desk — not at spa, dining |
| Guest's total lifetime spend | Fragmented — hotel sees hotel spend, dining sees dining spend |

**The bounded context test:**
```
"Guest" in Stays = a person who stayed in a room
"Guest" in Dining = a person who dined at the restaurant
"Guest" in Wellness = a person who had a treatment

Same person. Same business. Same identity record.
→ One module owns it. All operations reference it.
```

Guest Module is **Shared Always-On** — it activates automatically when a Business is created.
No owner needs to "enable" it. A business without guest identity is not a business.

---

## What This Module Is and Is Not

```
IS:
  → Who is this customer? (name, phone, email, nationality)
  → Have they been here before? (total_visits, last_visited_at)
  → Are they VIP? Blacklisted?
  → What occasions are coming up? (birthday, anniversary)
  → What notes has staff added over time?
  → How do we link them to a self-service login later? (Phase 2)

IS NOT:
  → Room preferences (Stays module owns this — queryable for room assignment)
  → Dietary restrictions (Dining module owns this)
  → Treatment preferences (Wellness module owns this)
  → Passport/NIC capture (Stays module — Sri Lanka legal requirement)
  → Loyalty points calculation (Loyalty module — Phase 2)
  → Campaign targeting (CRM module — Phase 3)
```

---

## Core Features (Always-On, Always Built)

> Guest Module has no "Add-ons" split. It is itself a shared module.
> Everything below ships as one always-on capability.

### G1 — Guest Profile (Core Identity)

The foundation record. Every other feature builds on top of this.

- Create guest profile with name, phone, email, nationality, date of birth, gender
- Phone = primary lookup key (unique per person in practice; staff searches by phone first)
- VIP flag + VIP notes (any staff with guest_modify permission)
- Blacklist flag + reason (manager only; reason mandatory; PIN override at booking attempt)
- Aggregated stats auto-updated: `total_visits`, `total_spend`, `last_visited_at`
- Consent tracking: `STAFF_CREATED` / `BOOKING_IMPORT` / `SELF_REGISTERED`
- GDPR soft anonymization: clear PII, keep aggregate stats and financial references
- `customer_account_id` nullable FK — null in V1, set in Phase 2 when guest creates portal account

### G2 — Occasions & Special Dates

Used by all operations to deliver personalised moments.

- Occasion types: `BIRTHDAY` / `ANNIVERSARY` / `HONEYMOON` / `OTHER`
- Annual occasions (birthday, anniversary): system checks month + day match today → flag at any interaction
- One-time occasions (honeymoon): year matters
- Free-text notes per occasion: "Bring roses — allergic to lilies."
- Visible on:
  - Hotel check-in screen
  - Restaurant reservation screen
  - Wellness appointment screen
  - Any operation that loads the guest profile

```
Check-in — Mr. Kasun (Sep 10):
  ⚠️  ANNIVERSARY TODAY (Sep 10)
      Notes: "12th anniversary. Arrange cake and flowers."
→ Front desk arranges. Guest surprised. Loyalty built.
```

### G3 — Staff Notes

Freeform timestamped log. Multiple notes over time. Full history.

- Note types: `PREFERENCE` / `COMPLAINT` / `POSITIVE` / `GENERAL`
- Any authorised staff can add notes
- `is_internal` flag: true = staff-only (never shown to guest even in Phase 2 portal)
- Staff can delete only their own notes
- Managers can delete any note

```
Why GuestNote is separate from vip_notes on GuestProfile:
  vip_notes   = permanent, always shown. "CEO of MAS Holdings."
  GuestNote   = timestamped log. History.
                Oct 2025: "Complained about AC noise, Room 302. Resolved."
                Dec 2025: "Left 5-star review. Mentioned Meena by name."
                Jan 2026: "Requested early check-in — accommodated."
```

### G4 — Duplicate Detection & Merge

Prevents fragmented guest history across multiple profiles for the same person.

**Detection (at creation time):**
```
Step 1: Exact phone match
  → Found 1 result → returning guest. Show profile.
  → Found 0 results → go to Step 2.

Step 2: Fuzzy name match (when no phone given)
  → Show list. Staff selects correct person.
  → Found 0 → new customer.

Step 3: Pre-creation duplicate warning
  → Before creating, system checks: same phone already exists?
  → "Possible match: Kasun Perera. Same profile?"
  → Staff confirms → link / merge.
  → Staff confirms different person → create new (rare).
```

**Merge (when duplicates are found):**
- Manager-only action
- All bookings/orders from duplicate profile → reassigned to surviving profile
- `total_visits` and `total_spend` → combined
- Duplicate profile → soft-deleted (`is_active = false`)
- `GuestMergeLog` created: who merged, when, reason
- `customer_account_id` links preserved on surviving profile

### G5 — VIP & Blacklist

Cross-operation guest status flags. Set once, fires everywhere.

**VIP:**
```
Who can set:   Any staff with guest_modify permission
How to set:    Toggle + required vip_notes ("CEO of MAS Holdings. Upgrade if suite available.")
What happens:
  → ⭐ VIP badge on check-in screen
  → VIP icon on booking lists
  → Night briefing: "3 VIP arrivals today"
  → Night audit: run VIP room checks before posting
```

**Blacklist:**
```
Who can set:   Manager only. Reason mandatory. Cannot save empty.
What happens at booking attempt:
  ⛔ "This guest is blacklisted."
      Reason: Property damage Oct 2025 — Room 204 TV broken.
      Blacklisted by: Meena (Manager) on Oct 22, 2025.
      [Proceed with Manager Override PIN] [Cancel Booking]

Cross-operation:
  Hotel booking → warning fires.
  Restaurant reservation → same warning fires.
  Spa appointment → same warning fires.
  Blacklist is BUSINESS-WIDE. Not operation-specific.

Remove blacklist: Manager only. Reason required. Logged.
```

### G6 — Cross-Operation Visit History

Full timeline of every interaction across all operations. Not a separate table — queried live.

```
GET /guests/{id}/history
  → Stays:    SELECT from stays.bookings WHERE guest_profile_id = X
  → Dining:   SELECT from dining.reservations WHERE guest_profile_id = X
  → Wellness: SELECT from wellness.appointments WHERE guest_profile_id = X
  → Sorted by date DESC. Unified view.

Staff sees:
  Sep 10, 2026 → Hotel Stay (Deluxe King, 3 nights, LKR 54,000)   [CURRENT]
  Aug 22, 2026 → Restaurant dinner (Table 5, LKR 8,400)
  Jul 15, 2026 → Hotel Stay (Suite, 2 nights, LKR 70,000)
  Jul 16, 2026 → Spa treatment (Deep Tissue, LKR 12,000)

  TOTAL: 12 visits | LKR 456,000 lifetime spend | Last: Sep 10, 2026
```

`total_visits` and `total_spend` on GuestProfile = cached aggregates.
Updated after each completed booking/order. Not recalculated on every view.

---

## GDPR & Data Privacy

```
Role split:
  guest_profiles = Business's CRM data
  Business       = Data Controller
  OneNex         = Data Processor
  Requirement:   DPA (Data Processing Agreement) with every business.

Lawful basis:
  Staff-created profile (booking) → "Contract"
  Marketing opt-in               → "Consent" (separate explicit checkbox)

Anonymization on erasure request:
  first_name, last_name → "Deleted" / "Guest"
  phone, email          → null
  nationality, dob, gender → null
  vip_notes, blacklist_reason → cleared
  KEEP: total_visits, total_spend, last_visited_at (non-PII analytics)
  KEEP: booking/order/invoice records (7-year financial retention minimum)
        → guest_name on those records → "Deleted Guest"

Retention:
  Active guest           → keep indefinitely
  No visit > 3 years     → auto-archive, notify business (Phase 2 automated job)
  No visit > 7 years     → auto-anonymize PII (Phase 2 automated job)
```

---

## V1 Scope — What Ships First

```
✅ MUST BUILD (V1):
  GuestProfile entity (all fields, including customer_account_id = nullable)
  GuestOccasion entity
  GuestNote entity
  GuestMergeLog entity
  Phone-based duplicate detection at creation
  VIP flag + notes (any staff)
  Blacklist flag + reason (manager only)
  Manager PIN override at booking attempt for blacklisted guest
  Cross-operation blacklist warning (fires in Stays, Dining, Wellness, etc.)
  Soft GDPR anonymization (manual, on request)
  total_visits / total_spend auto-update on booking completion
  Cross-operation visit history endpoint (queries live from operation modules)
  Full API surface (see guest-profile-design.md)
```

---

## Phase 2 — Customer Account Link (Guest Portal)

```
V1 (now):
  guest_profiles.customer_account_id = null
  Guest has no login. Staff manages profile entirely.
  Self-service only via OTP action tokens in SMS (cancel/modify booking link).

Phase 2:
  Guest creates account on customer.onenex.com
  System detects: "Phone +94771234567 found at Grand Hotel, City Bistro"
  Guest confirms → customer_account_id linked on their guest_profiles (both businesses)
  Guest can now:
    → View own visit history across all businesses
    → Update personal preferences
    → Make bookings online
    → View folio / invoices

  nullable FK already in schema from V1. Zero migration needed.

Phase 2 also adds:
  → Auto-VIP scoring (based on spend thresholds + visit frequency — no manual tagging)
  → Auto-archive job (no visit > 3 years → notify business)
  → Auto-anonymize job (no visit > 7 years → clear PII)
  → Right to rectification request flow (customer requests → business approves)
  → Breach notification system
  → Advanced guest search (by nationality, date range, spend)
```

---

## Module Connections

| Module | How It Uses Guest | What It Adds |
|---|---|---|
| **Stays** | `bookings.guest_profile_id` FK | StaysGuestPreference (room prefs), StaysGuestDocument (NIC/Passport) |
| **Dining** | `reservations.guest_profile_id` FK | DiningGuestPreference (dietary, seating, allergies) |
| **Wellness** | `appointments.guest_profile_id` FK | WellnessGuestPreference (therapist pref, health notes) |
| **Loyalty** | reads `guest_profiles.total_spend` | adds points balance, tier level (Phase 2) |
| **CRM** | reads `guest_profiles` + all operation data | adds segments, campaign targeting (Phase 3) |
| **Identity** | `guest_profiles.customer_account_id` FK | links to login account (Phase 2) |
| **Notification** | receives guest contact details | sends booking confirmations, OTP links |

---

## Design Decisions Locked In

1. **One profile per business, not global** — Grand Hotel's guest data belongs to Grand Hotel only. Cross-business data visible only via customer portal (Phase 2) with guest consent.
2. **Phone = primary lookup key** — not email, not name. Unique in practice. Staff dials to confirm.
3. **VIP is manual in V1** — no auto-scoring. Phase 2 adds spend-based auto-VIP.
4. **Blacklist is business-wide** — one blacklist flag fires across ALL operations for that business.
5. **Visit history is not a separate table** — queried live from each operation's tables. No sync needed.
6. **Operation-specific preferences belong to that operation** — Guest Module does NOT own StaysGuestPreference or DiningGuestPreference. Those modules own their own extension data.
7. **total_visits / total_spend = cached aggregates** — updated post-completion, not recalculated on load.
8. **customer_account_id nullable from V1** — schema ready for Phase 2 without migration.
9. **GDPR = anonymization, not deletion** — financial records (7-year) kept; PII cleared.

---

## Documents

| Doc | Status | Description |
|---|---|---|
| `00-guest-module-plan.md` | ✅ This file | Module overview, features, V1 scope |
| `guest-profile-design.md` | ✅ Complete | Entity designs, API surface, all rules |

### What Comes Next

Guest Module design is complete for V1. The two documents above cover everything needed to build it.

**Remaining work before implementation:**
- No further design docs needed for V1 Guest Module.
- Operation-specific preference entities (StaysGuestPreference, DiningGuestPreference) are designed within their respective module docs.
- Phase 2 design (customer portal, account linking, retention jobs) → when Phase 2 planning begins.
