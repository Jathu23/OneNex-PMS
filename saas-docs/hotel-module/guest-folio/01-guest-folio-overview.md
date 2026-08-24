# Guest Folio & Billing — Overview & Smart Features
> Hotel Module → Guest Folio & Billing
> Status: Critical inter-module topic | V1 required (simplified)

---

## What is a Guest Folio?

Folio = Guest-oda running bill.
Check-in-la open ஆகும். Check-out-la close ஆகும்.
Everything the guest uses during their stay → added to folio → paid at checkout.

```
Think of it like a bar tab:
  Guest arrives → Tab opened
  Uses any service → Added to tab
  Leaves → Tab closed → Pay everything at once
```

---

## Existing Systems — Problems

| System | Weakness |
|--------|---------|
| Oracle OPERA | Comprehensive but requires specialist training. UI cluttered. |
| Mews | Good UI, limited cross-module (only their own integrations) |
| Cloudbeds | Basic folio only. No split folio, no complex billing. |

**Core problem:** Folio systems built for accountants, not hotel staff.
Complex UI, too many steps, cross-module charges are manual.

---

## 1. Folio Creation

### Our Approach — Auto Folio on Check-in
```
Guest checks in → Room assigned
        ↓
System AUTOMATICALLY creates:
  Folio #F-10234
  ├── Linked to: Booking #1234
  ├── Linked to: Guest — Rajesh Kumar
  ├── Linked to: Room 412
  ├── Currency: INR
  ├── Tax rules: GST 12% (rooms), 5% (F&B)
  └── Status: OPEN

Zero staff action needed.
```

---

## 2. Charge Types

```
ROOM CHARGES (Auto — Night Audit):
  ├── Room rate per night
  ├── Early check-in fee
  └── Late check-out fee

F&B CHARGES (from Restaurant / Bar module):
  ├── Dine-in restaurant
  ├── Bar charges
  ├── Room service
  ├── Mini bar
  └── In-room dining

SPA CHARGES:
  └── Treatments, products

EVENT CHARGES:
  └── Tickets, catering

MISCELLANEOUS:
  ├── Laundry, Parking, Telephone
  └── Any manual charge by staff

ADJUSTMENTS (negative):
  ├── Discounts
  ├── Complimentary (comps)
  ├── Voids / corrections
  └── Loyalty redemptions
```

---

## 3. Night Audit — Auto Room Charge Posting

Every night at configured time (eg: 11:59 PM) — runs automatically.

```
For EVERY open folio:
  → Post room charge for that night

Dec 15 audit:  Folio F-10234 + ₹5,000 (Weekend rate) → Running: ₹5,000
Dec 16 audit:  Folio F-10234 + ₹5,000 (Weekend rate) → Running: ₹10,000
Dec 17 audit:  Folio F-10234 + ₹4,000 (Rack rate)   → Running: ₹14,000
```

Night audit also:
- Marks no-shows
- Generates daily flash report
- Closes business day (date rollover)
- Checks deposit deadlines
- Sends scheduled reports to management

---

## 4. Cross-Module Charge Routing

### Our Approach — Instant Auto-routing

```
Rajesh (Room 412) finishes dinner at hotel restaurant

Waiter: "How would you like to pay?"
Guest:  "Charge to Room 412"
Waiter: [Charge to Room] → System verifies Room 412 = Rajesh ✅
        ↓
INSTANTLY posted to Folio F-10234:
  Dec 15, 8:30 PM — Restaurant Dinner: ₹1,800

No manual transfer. No paper slip. No error risk.
Guest sees it on app in real-time.
```

Same for all modules:
```
Bar      → "Charge to room" → Instant folio entry
Spa      → Treatment done  → "Add to room?" → Instant
Room Svc → QR order placed → Auto-charged
Mini bar → Housekeeper scans items → Auto-charged
Parking  → Duration calc'd at checkout → Auto-charged
```

---

## 5. Real-time Folio View (Guest App)

### Our Approach — Transparent, No Surprises

```
Guest opens app during stay:

MY BILL — Room 412 (Dec 15-18)

ROOM CHARGES
  Dec 15 (Fri):          ₹5,000
  Dec 16 (Sat):          ₹5,000
  Dec 17 (Sun):          ₹4,000

RESTAURANT
  Dec 15, 8:30 PM:       ₹1,800
  Dec 16, 8:00 AM:       ₹0 (Breakfast included ✓)
  Dec 16, 1:15 PM:       ₹950

BAR
  Dec 16, 9:45 PM:       ₹1,200

SPA
  Dec 16, 2:00 PM:       ₹2,500

ROOM SERVICE
  Dec 17, 7:30 AM:       ₹650

──────────────────────────
Subtotal:                ₹21,100
GST:                      ₹1,890
──────────────────────────
Total:                   ₹22,990
Deposit paid:           (₹4,500)
──────────────────────────
Balance due:             ₹18,490

[Dispute a charge]  [Pay Now]  [Chat with Front Desk]
```

---

## 6. Folio Split — Multiple Billing Scenarios

### Split by charge type (Corporate):
```
Company pays: Room + Breakfast → Folio A (invoice to company)
Guest pays:   Spa + Bar + Minibar → Folio B (guest pays personally)
```

### Split by guest (Shared room):
```
Two guests sharing: Each pays 50% room + their own personal charges
```

### Group master folio:
```
Master Folio: All room charges → Company invoice
Individual Folios: Personal charges → Each employee pays
```

---

## 7. Disputed Charges

```
Guest: [Dispute] on app → "₹1,200 bar — I didn't visit"
        ↓
Front desk notified → Staff investigates (POS records, key log)
        ↓
Valid:   Explain with evidence → Keep charge
Invalid: Void entry → Adjusted folio → Guest notified

All disputes logged. Pattern detection (repeated errors flagged).
```

---

## 8. Complimentary Charges (Comps)

```
Manager comps guest dinner (service recovery):
  + Restaurant Dinner:  ₹1,800
  - Comp (Mgr Sanjay): -₹1,800  ← reason logged
  Net:                      ₹0

Monthly comp report → Management reviews
Comp budget per dept → Excess flagged
```

---

## 9. Checkout Settlement

```
Final folio review
        ↓
System checks: Night audit done? Mini bar scanned? Late checkout fee?
        ↓
Payment: Card / Cash / UPI / Corporate / Split
        ↓
Invoice: PDF emailed + GST invoice for business guests
        ↓
Folio: CLOSED → Key deactivated → Room → Dirty
```

---

## 10. Our Improvements vs Existing Systems

| Feature | Existing Systems | Our System |
|---------|-----------------|------------|
| Cross-module charges | Manual / semi-manual | Instant, automatic |
| Guest bill visibility | Only at checkout | Real-time on app |
| Dispute handling | Ad-hoc, no tracking | In-app, tracked, pattern detection |
| Comp management | No audit trail | Approval workflow + monthly report |
| Split folio | Complex, error-prone | Rule-based, intuitive UI |
| Mini bar | Paper slip | Housekeeping QR scan → auto-posted |
| Night audit | Often manual steps | Fully automated |
| GST invoice | Manual or basic | Auto-generated with correct breakdowns |

---

## V1 vs Later — See: `02-folio-v1-scope.md`
