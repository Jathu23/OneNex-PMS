# Room Booking — 05: Deposit & Payment Policies
> Hotel Module → Room Booking → Topic 5 of 7

---

## Overview

Guest booking confirm ஆனா — payment எப்போது collect பண்றது, எவ்வளவு collect பண்றது என்று இது cover பண்றது. Payment policy = hotel's financial protection strategy.

---

## 3 Main Payment Timing Options

```
Option 1: PAY NOW (Full)    → Book பண்ணும் போதே full amount collect
Option 2: PAY DEPOSIT       → Partial now, balance at check-in/check-out
Option 3: PAY AT HOTEL      → No payment now, full payment at property
```

---

## Option 1 — Pay Now (Full Prepayment)

Guest booking confirm பண்ணும் போதே total amount collect பண்றோம்.

**When to use:**
```
├── Non-refundable / saver rates (guest gets discount in exchange)
├── OTA bookings (Booking.com often collects full and remits to hotel)
├── Peak season / high demand — hotel wants guaranteed revenue
├── Short stays (1 night) — not worth the risk of non-payment
└── New guests (no history with hotel)
```

**Flow:**
```
Guest selects Non-refundable Saver rate
        ↓
Total: ₹9,600 (3 nights × ₹3,200)
        ↓
Payment gateway: Card / UPI / Net Banking
        ↓
Payment success → Booking CONFIRMED
Payment failure → Retry (3 attempts) → Auto-cancel if all fail
```

**Hotel benefit:** Full revenue secured before guest arrives. Zero risk.
**Guest benefit:** 20-30% cheaper rate.

---

## Option 2 — Pay Deposit

Partial amount now, balance when they arrive or check out.

**Deposit percentage — hotel configures:**
```
Common structures:
  ├── 25% deposit  (low barrier — encourage bookings)
  ├── 30% deposit  (standard)
  ├── 50% deposit  (balanced commitment)
  └── Up to 100%   (= Pay Now)
```

**Example:**
```
Booking total:    ₹15,000
Deposit (30%):    ₹4,500  ← Collected at booking
Balance:          ₹10,500 ← Collected at check-out

Confirmation:
  → Booking CONFIRMED after ₹4,500 received
  → Confirmation email shows: "Balance ₹10,500 due at check-out"
```

**Deposit deadline:**
```
Booking made: Nov 1
Check-in:     Dec 15

Policy: "Pay deposit within 48 hours of booking"
Deadline: Nov 3, same time as booking

If deposit not received by Nov 3:
  Option A: Auto-cancel booking (strict policy)
  Option B: Send reminder email → 24 more hours → then cancel
  Option C: Keep booking, try collecting at check-in (lenient policy)
  
  → Hotel configures which option
```

---

## Option 3 — Pay at Hotel

No payment at booking time. Full amount paid during stay or at check-out.

**When to use:**
```
├── Flexible / cancellable rates (guest pays premium for this flexibility)
├── Corporate accounts (company pays monthly — not per-stay)
├── VIP / loyalty members (trusted, history exists)
├── Walk-in guests (immediate payment at desk)
└── Phone bookings (card not collected during call)
```

**Risk: No-show protection needed**

```
Without protection:
  Guest books → Doesn't show → Hotel gets nothing

With Credit Card Guarantee:
  Guest provides card details at booking (not charged)
  Card saved securely (tokenized — PCI DSS compliant)
  Guest shows up → Pays at checkout → Card never charged
  Guest doesn't show → System auto-charges no-show fee
```

---

## Credit Card Guarantee — How It Works

```
Step 1: Guest books with Pay at Hotel option
Step 2: Card details collected (card number, expiry, CVV)
Step 3: Card TOKENIZED (actual number not stored — only secure token)
Step 4: System does a pre-authorization (small hold, eg ₹1)
        → Confirms card is valid and active
        → Hold released immediately
Step 5: Booking confirmed — card NOT charged

On arrival:
  Guest checks in → Stays → Checks out → Pays by card or cash
  Saved card NOT used (unless guest requests it)

On no-show (Topic 7):
  System charges no-show fee to saved card automatically
  Receipt emailed to guest
```

---

## Payment Methods

**At booking (online):**
```
├── Credit Card (Visa, Mastercard, Amex)
├── Debit Card
├── UPI (Google Pay, PhonePe, Paytm)
├── Net Banking
└── Payment Link (staff sends link to guest via SMS/email)
```

**At hotel (check-in / check-out):**
```
├── Card (tap / swipe / chip)
├── Cash
├── UPI (QR code scan)
├── Room charge (if guest has folio — inter-module)
└── Corporate credit (charge to company master account)
```

**Luxury / Enterprise (Layer 4):**
```
├── Wire transfer (large bookings, international)
└── Cryptocurrency (select luxury properties)
```

---

## Multi-Payment (Split Payment)

Guest wants to split the bill across multiple methods or people:

**Split by payment method:**
```
Total bill: ₹15,000

Split:
  ├── ₹5,000 by Visa Card (Guest's card)
  ├── ₹5,000 by Mastercard (Spouse's card)
  └── ₹5,000 by Cash
  
System: Process each separately, mark folio as settled when total reached
```

**Split by charge type (Corporate):**
```
Guest is on business trip, company pays room, guest pays personal:

Company Master Account:
  ├── Room charges (3 nights): ₹12,000
  └── Breakfast:               ₹1,800
  Subtotal company:            ₹13,800

Guest Personal:
  ├── Mini bar:                  ₹800
  ├── Spa:                     ₹2,500
  └── Personal restaurant:     ₹1,200
  Subtotal guest:               ₹4,500

Two separate invoices → Company gets one, guest pays other
```

---

## Folio Settlement at Check-out

Complete picture of what guest pays at the end:

```
Guest Folio — Room 104, Dec 15-18

Charges:
  Room (Dec 15 night) - Weekend Rate:         ₹5,000
  Room (Dec 16 night) - Weekend Rate:         ₹5,000
  Room (Dec 17 night) - Rack Rate:            ₹4,000
  Restaurant (Dec 15, dinner):                ₹1,800
  Bar (Dec 16, drinks):                         ₹900
  Spa (Dec 17, massage):                      ₹2,500
  Mini bar (Dec 17):                            ₹450
  Room service (Dec 16, breakfast):             ₹650
  ─────────────────────────────────────────────────
  Gross Total:                               ₹20,300
  
  GST (12%):                                  ₹2,436
  ─────────────────────────────────────────────────
  Total with tax:                            ₹22,736
  
  Less: Deposit paid at booking (30%):       (₹4,500)
  ─────────────────────────────────────────────────
  Balance Due at Check-out:                  ₹18,236
  
  Payment: Visa Card ✓
  ─────────────────────────────────────────────────
  SETTLED ✓
```

---

## Refund Flow (When Cancellation Happens)

```
Guest cancels booking
        ↓
Cancellation policy applied (see Topic 6)
        ↓
Refund amount calculated:
  Full refund:    Return entire deposit / payment
  Partial refund: Return deposit minus penalty
  No refund:      No refund (non-refundable rate)
        ↓
Refund method (same as payment method):
  Card payment → Reverse to same card
    → Credit cards: 3-7 business days
    → Debit cards: 2-5 business days
  UPI payment  → Reverse to UPI ID (instant or 1-2 days)
  Cash         → Staff manual cash refund at desk
        ↓
Refund confirmation:
  → Email to guest with refund amount and timeline
  → Booking status: CANCELLED
```

---

## Policy Configuration (Hotel Setup)

Hotel admin configures per rate plan and booking type:

```
Rate Plan: "Standard Flexible"
  Payment type:         Pay at Hotel
  Deposit required:     No
  Card guarantee:       Yes (card details mandatory)
  Deposit deadline:     N/A

Rate Plan: "Non-refundable Saver"
  Payment type:         Pay Now (Full)
  Deposit required:     100% (full payment)
  Deposit deadline:     Immediate (booking only confirms on payment)
  Card guarantee:       N/A (already charged)

Rate Plan: "Early Bird 30"
  Payment type:         Deposit
  Deposit %:            30%
  Deposit deadline:     48 hours from booking
  Auto-cancel if unpaid: Yes
  Balance due:          At check-out

Booking Type: Corporate
  Payment type:         Pay at Hotel (company credit)
  Deposit:              No (contract-based)
  Billing:              Monthly consolidated invoice to company

Booking Type: Group
  Payment type:         Deposit schedule
  Deposit 1:            25% at contract signing
  Deposit 2:            50% 30 days before check-in
  Balance:              At check-out or day before
```

---

## Summary

```
Booking created
        ↓
Rate plan determines payment type:
  Pay Now    → Collect full → Confirm booking
  Deposit    → Collect X%  → Confirm → Collect balance at checkout
  Pay Later  → Collect card guarantee → Confirm → Collect at checkout
        ↓
During stay:
  All charges accumulate in Guest Folio
        ↓
Check-out:
  Folio reviewed → Balance calculated → Payment collected → Settled
  
(If cancelled → Refund per policy)
(If no-show → Charge saved card per policy)
```
