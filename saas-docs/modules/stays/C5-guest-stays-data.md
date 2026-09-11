# C5 — Guest Data for Stays
> Stays Module | Core Feature 5 of 9
> Status: Final | Version: 1.0
> Depends on: Guest Module (guest-profile-design.md) — must be read first

---

## 1. What This Is (and Is Not)

```
Guest Module owns:   who the guest is (name, phone, VIP, blacklist, history)
This document owns:  what Stays specifically needs to know about a guest
```

When a guest checks into a hotel, the front desk needs more than just their name.
They need to know: preferred bed type, floor, smoking preference — to assign the right room.
They need legal documents: passport or NIC (required by Sri Lanka law for all hotel guests).
They need to know upcoming occasions: anniversary today → arrange flowers.

None of this is needed by the restaurant or spa. **Stays-specific. Stays module owns it.**

---

## 2. The 3 Stays-Specific Entities

```
GuestProfile (Guest Module — foundation)
  ├── StaysGuestPreference    ← room assignment preferences
  ├── StaysGuestDocument      ← passport / NIC (legal)
  └── (GuestOccasion is in Guest Module — shared across all operations)
```

Stay history = queried from `stays.bookings` — no separate entity needed.

---

## 3. Entity 1: `StaysGuestPreference` — Room Assignment Preferences

Used by front desk when assigning a room at check-in. Structured fields — not JSONB.
Must be queryable: "show available King bed rooms on high floor, non-smoking."

```
id
guest_profile_id      FK → guest_profiles (Guest Module)
hotel_id              FK → Hotel (one preference set per hotel — guest may prefer
                      different things at a business hotel vs beach resort)

-- Room preferences
bed_type_preference
  → KING / QUEEN / TWIN / SINGLE / ANY
  → Used to pre-filter room options at check-in

floor_preference
  → HIGH / MIDDLE / LOW / ANY
  → HIGH = top floors (view, quieter from street)
  → LOW  = near ground (accessibility, elevator avoidance)

smoking_preference
  → NON_SMOKING / SMOKING / ANY
  → Default: NON_SMOKING (most hotels are fully non-smoking)

room_location_preference
  → QUIET          far from elevator, ice machine, street
  → NEAR_ELEVATOR  accessibility or convenience
  → POOL_VIEW      overlooks pool
  → CITY_VIEW      overlooks city
  → GARDEN_VIEW    overlooks garden
  → ANY            no preference

extra_pillows         bool default false.
extra_towels          bool default false.
quiet_hours_sensitive bool default false.
                      "Do not disturb after 10 PM — light sleeper."

specific_notes        text nullable.
                      "Prefers rooms not above floor 5 — claustrophobic in high floors."
                      "Always request room away from kitchen — food smell sensitivity."

updated_at            timestamp. When preferences were last updated/confirmed.
```

**How front desk uses this at check-in:**
```
Mr. Kasun checks in.
System shows: "Preferences on file: King bed | High floor | Non-smoking | Quiet"

Available rooms matching:
  Room 801 — Deluxe King, Floor 8, Non-smoking, Quiet wing   ← best match
  Room 602 — Deluxe King, Floor 6, Non-smoking, Near elevator
  Room 405 — Deluxe King, Floor 4, Non-smoking, Pool view

Staff picks Room 801. One click. No guessing.
Guest preference honoured without asking every visit.
```

**Why structured fields (not JSONB):**
```
JSONB approach:
  preferences: { "bed": "KING", "floor": "HIGH" }
  → Cannot do: WHERE preference->>'bed' = 'KING' efficiently
  → Cannot index individual fields
  → Type safety lost (any string can be stored)

Structured fields:
  bed_type_preference = 'KING'
  → Full index support
  → Enum validation at DB level
  → Front desk filter query runs in milliseconds
```

---

## 4. Entity 2: `StaysGuestDocument` — Legal Document Capture

**Sri Lanka legal requirement:** All hotels must record guest identity documents.
- Foreign guests: Passport (number + expiry + nationality)
- Local guests: NIC (National Identity Card)

Required by: Sri Lanka Tourism Development Authority + Police registration (for foreign guests).
Also needed for: TDL (Tourism Development Levy) — applies only to foreign guests.

```
id
guest_profile_id      FK → guest_profiles (Guest Module)
hotel_id              FK → Hotel

document_type
  → PASSPORT          Foreign guests. Number + country + expiry required.
  → NIC               Sri Lankan guests. 9-digit old or 12-digit new format.
  → DRIVING_LICENSE   Accepted when NIC not available (local guests).

document_number       varchar. Stored as-is (no format enforcement — international varies).

issuing_country       varchar nullable.
                      Required for PASSPORT.
                      Not needed for NIC / DRIVING_LICENSE.

expiry_date           date nullable.
                      Required for PASSPORT.
                      NIC: no expiry. DRIVING_LICENSE: expiry optional.

scan_url              varchar nullable.
                      URL to stored document scan (file storage — S3/Blob).
                      Phase 2: capture via mobile camera at check-in.
                      V1: optional (manual number entry sufficient for legal compliance).

captured_at           timestamp. When document was verified.
captured_by_staff_id  FK → Staff. Who verified the document.

is_verified           bool default false.
                      Staff confirms document was physically seen (not just number entered).
                      Affects compliance audit.
```

**Sri Lanka NIC format note:**
```
Old format: 9 digits + V or X  (e.g. 199023401234V)
New format: 12 digits           (e.g. 199023401234)

System accepts both. No format enforcement — just store as entered.
Validation: must be non-empty if document_type = NIC.
```

**TDL (Tourism Development Levy) connection:**
```
TDL applies to: foreign guests only.
How system knows: StaysGuestDocument.issuing_country ≠ "Sri Lanka"
                  OR GuestProfile.nationality ≠ "Sri Lankan"

Billing module reads this at folio creation:
  Foreign guest → TDL line item added to folio automatically.
  Local guest   → TDL not applied.

This is why document capture happens at check-in, before folio is finalized.
```

---

## 5. Stay History — No Separate Entity

```
Front desk views stay history:

GET /guests/{id}/stay-history (Stays module endpoint)

SELECT
  b.id, b.check_in_date, b.check_out_date,
  rt.name as room_type, r.room_number,
  rp.name as rate_plan,
  b.total_amount, b.status
FROM stays.bookings b
JOIN stays.rooms r ON b.room_id = r.id
JOIN stays.room_types rt ON r.room_type_id = rt.id
JOIN stays.rate_plans rp ON b.rate_plan_id = rp.id
WHERE b.guest_profile_id = :guest_id
  AND b.hotel_id = :hotel_id
ORDER BY b.check_in_date DESC

Staff sees:
  Sep 10, 2026 → Deluxe King (Room 801), 3 nights, Executive Package, LKR 54,000 [IN-HOUSE]
  Jul 15, 2026 → Suite (Room 1201), 2 nights, BAR Standard, LKR 70,000        [CHECKED-OUT]
  Mar 02, 2026 → Standard Double (Room 302), 1 night, Flexible, LKR 12,000    [CHECKED-OUT]

Summary: 3 stays | 6 nights | LKR 136,000 total (stays only)
```

---

## 6. Check-In Screen — Full Picture

When front desk opens check-in for a guest, all data comes together:

```
┌────────────────────────────────────────────────────────────────┐
│  MR. KASUN PERERA                                    ⭐ VIP   │
│  +94 77 123 4567 | kasun@masholdings.com                       │
│  Sri Lankan | NIC: 199023401234                                │
│                                                                │
│  ⚠️  ANNIVERSARY TODAY (Sep 10)                               │
│      Notes: "12th anniversary. Arrange cake and flowers."      │
│                                                                │
│  STAY HISTORY: 3 stays | 6 nights | LKR 136,000               │
│  Last stay: Jul 2026                                           │
│                                                                │
│  VIP NOTES: CEO of MAS Holdings. Upgrade to suite if avail.   │
│                                                                │
│  PREFERENCES:                                                  │
│    Bed: King  |  Floor: High  |  Smoking: No  |  Quiet wing   │
│                                                                │
│  STAFF NOTES:                                                  │
│    Oct 2025: Complained about AC noise Room 302. Resolved.     │
│    Dec 2025: Left 5-star review. Mentioned Meena by name.      │
│                                                                │
│  BOOKING: Sep 10-13, 2026 | Deluxe King | Executive Package   │
│  Assign room: [Room 801 — Best match ▼]   [ Check In → ]      │
└────────────────────────────────────────────────────────────────┘
```

Every field on this screen comes from:
- Name, VIP, vip_notes → GuestProfile (Guest Module)
- Anniversary → GuestOccasion (Guest Module)
- Preferences → StaysGuestPreference (Stays Module — this doc)
- NIC → StaysGuestDocument (Stays Module — this doc)
- Stay history → queried from stays.bookings
- Staff notes → GuestNote (Guest Module)

---

## 7. API Surface (Stays-Specific)

```
STAYS GUEST PREFERENCES
──────────────────────────────────────────────────────────────────
GET    /guests/{id}/stays-preferences           Get preferences
POST   /guests/{id}/stays-preferences           Create preferences
PATCH  /guests/{id}/stays-preferences           Update preferences

STAYS GUEST DOCUMENTS
──────────────────────────────────────────────────────────────────
GET    /guests/{id}/documents                   Get all documents
POST   /guests/{id}/documents                   Add document (at check-in)
PATCH  /guests/{id}/documents/{did}             Update document
GET    /guests/{id}/documents/{did}/verify      Mark as verified

STAY HISTORY
──────────────────────────────────────────────────────────────────
GET    /guests/{id}/stay-history                All stays at this hotel
GET    /guests/{id}/stay-history/{booking_id}   Single stay detail
```

---

## 8. V1 Scope

```
✅ MUST BUILD (V1):
  StaysGuestPreference (all fields)
  StaysGuestDocument (NIC + Passport — scan_url optional)
  Stay history endpoint (queried from bookings — no separate table)
  Check-in screen integration (preferences + documents + occasions)
  TDL flag derivation from document.issuing_country

PHASE 2:
  → Document scan via mobile camera (scan_url populated)
  → OCR auto-fill from passport scan (Phase 3)
  → Preference learning (system suggests based on past room assignments)
  → Preference sync across operations (cross-op preference sharing)
```
