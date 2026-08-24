# Room Booking — 02: Booking Flow
> Hotel Module → Room Booking → Topic 2 of 7

---

## Overview

Oru booking create ஆனது முதல் guest check-out ஆகும் வரை — என்னென்ன states-la இருக்கும், என்ன actions நடக்கும் என்று இது cover பண்றது. Itha "Booking Lifecycle" என்று சொல்லலாம்.

---

## Full Lifecycle

```
CREATE → CONFIRM → [MODIFY] → CHECK-IN → STAY → CHECK-OUT
                 ↘ [CANCEL]
                 ↘ [NO-SHOW]
```

---

## State 1 — CREATE

Booking system-la first time enter ஆகும் moment.

**Required information at creation:**
```
├── Guest name + contact (phone / email)
├── Check-in date & Check-out date
├── Room type (Double, Suite, etc. — not specific room number yet)
├── Number of guests (adults / children)
├── Rate plan selected
└── Source (Walk-in / Online / OTA / Group...)
```

**Optional at creation:**
```
├── Special requests (high floor, extra bed, early check-in, quiet room)
├── Deposit payment
└── Internal notes (staff use)
```

**Important:** Specific room number இந்த stage-la assign ஆகாது. Room type மட்டும் select பண்றோம். Actual room number — check-in time-la assign பண்ணுவோம்.

**Why not assign room at booking?**
- Room அப்போ dirty / occupied-ஆ இருக்கலாம்
- Housekeeping schedule தெரியாது
- Check-in day-la best available room offer பண்ணலாம் (upgrade opportunity)

---

## State 2 — CONFIRM

Booking confirm ஆனவுடன் — guest-ku automatic communication போகும்.

**Auto-triggers on confirmation:**
```
├── Confirmation Email:
│     ├── Booking ID / Reference number
│     ├── Hotel name & address
│     ├── Check-in & Check-out dates
│     ├── Room type booked
│     ├── Rate & total amount
│     ├── Cancellation policy
│     └── Special requests noted
│
├── Confirmation SMS:
│     └── Short summary + Booking ID
│
└── WhatsApp (optional, Layer 2+):
      └── Booking summary card
```

**Booking Status:** `CONFIRMED`

**Deposit logic at this stage:**
```
Policy: No deposit required    → Status: CONFIRMED immediately
Policy: Deposit required       → Status: PENDING_PAYMENT
                                 Guest pays deposit
                                 Status: CONFIRMED

If deposit not paid by deadline → Auto-cancel (configurable)
```

---

## State 3 — MODIFY

Guest plans change → Booking-la ஏதாவது மாற்றணும்.

**What can be modified:**
```
├── Dates (check-in/check-out extend or shorten)
├── Room type (upgrade or downgrade)
├── Number of guests
├── Special requests (add/change)
└── Rate plan (policy-based — some rates non-modifiable)
```

**Rules for modification:**
```
Date change:
  → System re-checks availability for new dates
  → If not available → Suggest alternatives
  → Rate recalculated for new dates

Room type change:
  → Availability check for new type
  → Rate difference calculated
  → If deposit paid → Adjust (collect more or refund difference)

Modification deadline:
  → Guest (online): Can modify up to X hours before check-in (configurable)
  → Staff: Can modify anytime
```

**After modification:**
```
→ Updated confirmation email sent to guest
→ Booking status stays: CONFIRMED
→ Modification history logged (audit trail)
```

---

## State 4 — CANCEL

Booking cancel ஆகுது — guest or staff request-la, or system auto-cancel.

**Who can cancel:**
```
Guest (online):   Up to allowed window (per rate plan policy)
Staff:            Anytime (manager can override penalty)
System (auto):    If deposit not received by deadline
                  If payment fails after X retries
```

**On cancellation:**
```
├── Cancellation policy check (see Topic 6 — Cancellation Rules)
├── Penalty calculated
├── Refund initiated (if applicable)
├── Room released → Back to inventory (can resell)
├── Booking status: CANCELLED
└── Cancellation confirmation → Guest email/SMS
```

---

## State 5 — NO-SHOW

Guest booking இருக்கு — but check-in date-la வரலை, cancel-உம் பண்ணலை.

**How it's detected — Night Audit (End of Day Process):**
```
End of check-in date (eg: Dec 15 midnight)
        ↓
Night Audit runs automatically
        ↓
System checks: Any CONFIRMED bookings with today's check-in date
               that are still not checked in?
        ↓
Mark as NO-SHOW
        ↓
Apply no-show charge (usually 1 night — policy based)
        ↓
Charge saved card on file
        ↓
Room released back to inventory
        ↓
Notification to front desk manager
```

**Booking Status:** `NO_SHOW`

---

## State 6 — CHECK-IN

Guest hotel-ku arrive ஆகிறான்.

**Flow:**
```
Guest front desk-la வருவான்
        ↓
Staff finds booking:
  Search by: Name / Phone / Booking ID / Email
        ↓
Guest identity verify (ID proof check — passport, aadhaar, etc.)
        ↓
Specific room assign பண்றாங்க:
  Check: Room clean? Inspected? Ready?
  Yes → Assign that room
  No  → Wait (housekeeping notify) or offer alternative ready room
        ↓
Any upgrades to offer? (if better room available — upsell opportunity)
        ↓
Key card program பண்றாங்க / Digital key send பண்றாங்க
        ↓
Guest Folio opened (billing starts from this moment)
        ↓
Booking status: CHECKED_IN
        ↓
Housekeeping notified: "Room 104 — Now Occupied"
        ↓
Welcome message send பண்றாங்க (optional — app notification)
```

---

## State 7 — STAY (During Residence)

Guest hotel-la இருக்கும் period.

**What happens during stay:**
```
Every night (Night Audit):
  → Room charge auto-post to Guest Folio
  → eg: ₹5,000/night automatically added

Guest uses other services:
  → Restaurant charge → Added to Folio
  → Bar charge        → Added to Folio
  → Spa charge        → Added to Folio
  → Mini bar          → Added to Folio (housekeeping scans)
  → Room service      → Added to Folio

Guest can view folio anytime:
  → At front desk
  → Via app (Layer 2+)

Room change possible:
  → Guest request → Staff reassign → Folio continues on new room
  
Stay extension:
  → Guest wants to extend → Staff check availability for extra nights
  → Folio continues
```

---

## State 8 — CHECK-OUT

Guest stay complete — settle பண்றான்.

**Flow:**
```
Guest front desk-ku வருவான் (or express/mobile checkout)
        ↓
Staff pulls up Guest Folio
        ↓
Final folio review:
  ├── All room nights charged?
  ├── All restaurant / bar / spa charges added?
  ├── Mini bar checked?
  └── Any pending charges?
        ↓
Any last-minute charges? Add now
        ↓
Guest reviews bill:
  Any disputes? → Staff resolves (void / adjust)
  All ok? → Proceed
        ↓
Payment collected:
  ├── Balance due (Total − Deposit already paid)
  ├── Payment method: Card / Cash / UPI
  └── Corporate? → Post to company account
        ↓
Invoice generated:
  → Print (if requested)
  → Email (automatic)
        ↓
Key card deactivated / Digital key expired
        ↓
Room status → DIRTY (housekeeping notified immediately)
        ↓
Booking status: CHECKED_OUT
        ↓
Auto-triggers:
  ├── Thank you email
  ├── Review request (Google / TripAdvisor)
  └── Loyalty points updated (if loyalty member)
```

---

## Booking Status — All States

| Status | Meaning | Next possible state |
|--------|---------|-------------------|
| `CONFIRMED` | Booking active, guest not arrived | MODIFIED, CANCELLED, CHECKED_IN, NO_SHOW |
| `MODIFIED` | Change made, still active | CANCELLED, CHECKED_IN, NO_SHOW |
| `CHECKED_IN` | Guest currently in hotel | CHECKED_OUT |
| `CHECKED_OUT` | Stay complete, settled | — (final) |
| `CANCELLED` | Booking cancelled | — (final) |
| `NO_SHOW` | Guest didn't arrive | — (final) |

---

## Full State Diagram

```
                    ┌─────────────┐
                    │   CREATED   │
                    └──────┬──────┘
                           │ deposit / auto confirm
                    ┌──────▼──────┐
              ┌─────│  CONFIRMED  │──────┐
              │     └──────┬──────┘      │
              │            │             │
         ┌────▼────┐       │       ┌─────▼──────┐
         │CANCELLED│       │       │  MODIFIED  │
         └─────────┘       │       └─────┬──────┘
                           │             │ re-confirm
                    ┌──────▼─────────────▼──┐
                    │      CHECKED-IN        │
                    └──────────┬─────────────┘
                               │ stay period
                    ┌──────────▼─────────────┐
                    │      CHECKED-OUT        │
                    └────────────────────────┘

    (Guest didn't arrive on check-in date)
                    ┌─────────────┐
                    │  NO-SHOW    │
                    └─────────────┘
```

---

## Audit Trail

Every state change → Log பண்ணணும்:

```
Booking #1234 — History:
  2026-12-01 10:30  CREATED       by: Guest (Online)
  2026-12-01 10:31  CONFIRMED     by: System (auto — deposit received)
  2026-12-05 14:20  MODIFIED      by: Staff (Priya) — dates changed Dec 15→16
  2026-12-15 14:05  CHECKED_IN    by: Staff (Ravi) — Room 104 assigned
  2026-12-18 11:30  CHECKED_OUT   by: Staff (Ravi) — Settled ₹16,000
```

Every action — who did it, when, what changed — trackable.
