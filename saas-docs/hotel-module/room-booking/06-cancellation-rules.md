# Room Booking — 06: Cancellation Rules
> Hotel Module → Room Booking → Topic 6 of 7

---

## Overview

Guest booking cancel பண்றான் — என்ன நடக்கும், எவ்வளவு charge பண்ணலாம், எப்படி refund பண்றது என்று இது cover பண்றது. Cancellation policy = hotel revenue protection + guest fairness balance.

---

## Why Cancellation Policies Exist

```
Hotel perspective:
  Last-minute cancel → Room empty → Revenue lost
  Hotel paid for: staff, housekeeping, food prep
  Policy protects: guaranteed revenue even on cancellation

Guest perspective:
  Plans change (flight cancelled, illness, emergency)
  Need flexibility → Willing to pay premium for it
  
Balance: Strict enough to protect hotel, fair enough to not scare guests away
```

---

## 4 Main Cancellation Policy Types

### Type 1 — Free Cancellation

```
"Cancel before X hours/days → Full refund, no questions"

Example:
  Policy: Free cancel up to 48 hours before check-in

  Check-in: Dec 15, 2:00 PM
  Free cancel deadline: Dec 13, 2:00 PM

  Cancel Dec 12 (3 days before) → 100% refund ✓
  Cancel Dec 14 (1 day before)  → Penalty applies ✗
```

Best for: Flexible rate plans (higher price, maximum flexibility)

---

### Type 2 — Partial Penalty (Night-based)

```
"Cancel within X hours → Charged for Y nights"

Example policy:
  Cancel 24-48 hrs before check-in → 1 night charge
  Cancel within 24 hrs             → 2 nights charge

Booking: 3 nights × ₹5,000 = ₹15,000 total

Cancel Dec 14, 3 PM (27 hours before Dec 15, 2 PM check-in):
  Penalty: 1 night = ₹5,000
  Refund:  ₹10,000

Cancel Dec 15, 10 AM (4 hours before check-in):
  Penalty: 2 nights = ₹10,000
  Refund:  ₹5,000
```

Best for: Standard flexible rates — some protection, some flexibility

---

### Type 3 — Non-Refundable

```
"No refund at any time after booking is confirmed"

Guest cancels → No refund, regardless of when they cancel
Hotel keeps full payment

When to use:
  ├── Saver/discounted rates (20-30% cheaper — guest trades flexibility)
  ├── Flash sale rates
  ├── Peak season bookings (Dec 20 - Jan 5, summer)
  └── High-demand event periods

Important: Room gets released back to inventory even with no refund
  → Hotel can resell the room AND keep the payment (best outcome for hotel)
```

Best for: Discount rates — maximum hotel revenue protection

---

### Type 4 — Percentage-Based Penalty

```
"Cancel X days before → Pay Y% of total booking as penalty"

Tiered example:
  30+ days before    → 0% penalty   (100% refund)
  15-29 days before  → 25% penalty  (75% refund)
  7-14 days before   → 50% penalty  (50% refund)
  3-6 days before    → 75% penalty  (25% refund)
  0-2 days before    → 100% penalty (no refund)

Booking total: ₹20,000

Cancel 20 days before check-in:
  Penalty: 25% = ₹5,000
  Refund:  ₹15,000

Cancel 10 days before check-in:
  Penalty: 50% = ₹10,000
  Refund:  ₹10,000
```

Best for: Long stays, luxury properties, high-value bookings

---

## Special Cancellation Cases

### Group Booking Cancellation

Groups have stricter, separate policies:

```
Group: 20 rooms, Dec 15-18

Group cancellation policy:
  60+ days before  → Full refund
  30-59 days       → 50% refund (rooms hard to resell with short notice)
  15-29 days       → 25% refund
  Within 15 days   → No refund

Partial cancellation rules:
  "Can reduce group size by up to 10% without penalty"
  "Reduce by more than 10% → penalty on cancelled rooms"
  
Example: Group booked 20 rooms
  Reduce to 18 rooms (10% cut) → No penalty on 2 cancelled rooms
  Reduce to 15 rooms (25% cut) → Penalty applies on 3 of the 5 cancelled rooms
```

---

### Early Departure (Guest Leaves Before Check-out Date)

```
Guest booked: Dec 15 → Dec 20 (5 nights, ₹25,000)
Guest wants to leave early: Dec 18 (after 3 nights)

Early departure policy options (hotel configures):
  Option A (Strict):   Charge full 5 nights (₹25,000) — hotel's right
  Option B (Moderate): Charge 3 nights stayed + 1 night penalty
                        = ₹15,000 + ₹5,000 = ₹20,000
  Option C (Flexible): Charge only nights stayed (₹15,000) — no penalty
  
Policy set per rate plan. "Non-refundable" rate → Option A always.
```

---

### Force Majeure (Extraordinary Circumstances)

Situations completely outside guest's control:

```
Qualifying situations:
  ├── Natural disaster (flood, earthquake, cyclone)
  ├── Government travel ban / lockdown
  ├── Medical emergency (with documentation)
  ├── Airline / transport cancellation (beyond guest's control)
  └── Death in family

Hotel options (configurable policy):
  Option A: Full refund (guest-friendly, builds trust)
  Option B: Credit for future stay (hotel keeps money, guest gets value)
  Option C: Case-by-case review (staff decides with documentation)

System support (Layer 3+):
  Staff can mark booking as "Force Majeure"
  → Overrides normal policy
  → Full refund or credit issued
  → Reason + documentation logged
  → Audit trail maintained
```

---

## Who Can Cancel

```
Guest (self-service via app/website):
  ├── Only within allowed window (per rate plan)
  ├── Shows: penalty amount and refund amount before confirming
  └── Auto-refund initiated after confirmation

Staff (system):
  ├── Can cancel any booking at any time
  ├── Can apply standard penalty OR override it (with reason)
  ├── Manager permission required for penalty waiver
  └── Reason/note mandatory for any cancellation

System (automated):
  ├── Deposit not received by deadline → Auto-cancel
  ├── Payment failure after X retries → Auto-cancel
  └── Guest notified with reason
```

---

## Cancellation Flow

```
Cancellation request received
        ↓
System checks: Which rate plan was booked?
        ↓
System calculates: Days/hours until check-in
        ↓
Find applicable policy tier
        ↓
Calculate:
  Penalty amount = ?
  Refund amount = Total paid - Penalty
        ↓
Show to guest / staff:
  "Cancellation penalty: ₹5,000
   Refund to your card:  ₹10,000
   [Confirm Cancellation]"
        ↓
Confirmed:
        ↓
┌───────────────────────────────────────┐
│  Refund > 0?                          │
│  → Initiate refund to original method │
│  → Refund confirmation email to guest │
│                                       │
│  Penalty > 0?                         │
│  → Charge saved card if not yet paid  │
│  → Receipt emailed to guest           │
└───────────────────────────────────────┘
        ↓
Booking status: CANCELLED
        ↓
Room released → Available for new bookings
        ↓
Cancellation confirmation email/SMS → Guest
        ↓
Reports updated: Cancellation revenue tracked
```

---

## Refund Processing Times

```
Refund method:
  Credit card  → 3-7 business days (bank processing time)
  Debit card   → 2-5 business days
  UPI          → Instant to 2 business days
  Net banking  → 3-5 business days
  Cash         → Immediate (at front desk)

System shows: "Refund of ₹10,000 initiated. 
               Expected to reflect in 3-5 business days."
```

---

## Cancellation Analytics (Reports for Hotel Management)

```
Reports the hotel needs:
  ├── Cancellation rate % by month / season
  ├── Cancellation rate % by booking source (OTA cancels more than direct?)
  ├── Revenue lost to cancellations (missed revenue)
  ├── Penalty revenue collected (recovered revenue)
  ├── Last-minute cancellation trend (< 24 hours)
  ├── Which rate plan has highest cancellation rate?
  ├── Average days before check-in when cancellation happens
  └── Cancellation vs No-show ratio
```

---

## Policy Configuration (Hotel Admin Setup)

```
Per Rate Plan:
  ├── Policy type: Free / Night-based / Non-refundable / Percentage
  ├── Free cancel window: 0 to 365 days (or hours)
  ├── Penalty tiers: Up to 5 tiers configurable
  ├── Early departure policy: Strict / Moderate / Flexible
  └── Force majeure handling: Refund / Credit / Case-by-case

Per Booking Type:
  ├── Individual → Rate plan policy applies
  ├── Group      → Separate stricter group cancellation policy
  └── Corporate  → Contract-defined (often more flexible)

Staff Permissions:
  ├── Staff:   Can cancel (normal policy applies)
  ├── Manager: Can waive penalty (with reason)
  └── All cancellations: Audit log maintained
```

---

## Summary

| Policy Type | When to Use | Guest Gets | Hotel Gets |
|-------------|-------------|-----------|-----------|
| Free Cancel | Flexible rates | Full refund | Nothing (but trust) |
| Partial Penalty | Standard rates | Partial refund | Penalty amount |
| Non-refundable | Saver rates | Nothing | Full amount + can resell |
| % Based | Long stays / luxury | Sliding refund | Sliding penalty |

**Key Rule:** Regardless of cancellation policy, room is ALWAYS released back to inventory. Hotel can resell it.
