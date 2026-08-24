# Guest Folio — V1 Scope Decision
> What to build first vs what to skip for later

---

## Decision Framework

```
Question for every feature:
"Hotel operate பண்ண இது இல்லாம முடியுமா?"

  NO  → V1 MUST HAVE
  YES → Later phase
```

---

## V1 — MUST HAVE

```
✅ Auto folio creation at check-in
   (Cannot track charges without this)

✅ Room charge auto-posting (Night Audit)
   (Core hotel billing — rooms must be charged nightly)

✅ Manual charge entry by staff
   (Staff needs to add any charge: parking, laundry, misc)

✅ Cross-module charge routing (basic)
   Restaurant "charge to room" → appears on folio
   (This is the core inter-module feature — must work)

✅ Deposit tracking
   (Guest paid deposit at booking → must reflect in folio)

✅ Basic checkout settlement
   Total → Payment collected → Folio closed

✅ Invoice / Receipt generation (basic)
   Guest needs proof of payment

✅ Tax calculation (GST)
   Legal requirement — cannot skip
```

---

## V1 — SKIP (Add Later)

```
❌ Real-time folio on guest app
   → Staff can show folio at front desk
   → App view is nice-to-have, not must-have

❌ Folio split (corporate / shared room)
   → V1: One folio per booking, one payment
   → Phase 2: Add split functionality

❌ Group master folio
   → Phase 2: Add when group bookings mature

❌ In-app dispute flow
   → V1: Guest disputes verbally at counter
   → Phase 2: In-app dispute with tracking

❌ Comp approval workflow with budget limits
   → V1: Staff manually marks comp, note in system
   → Phase 2: Formal workflow + reports

❌ Dispute pattern detection
   → Phase 3: Analytics feature

❌ Mini bar QR scan auto-posting
   → V1: Staff manually enters mini bar charges
   → Phase 2: Scan-based auto-posting

❌ Night audit detailed reports (flash report, forecasts)
   → V1: Basic room charge posting only
   → Phase 2: Full night audit reports
```

---

## V1 Folio — What It Looks Like

Simple. Clean. Functional.

```
FOLIO — Booking #1234 | Room 412 | Rajesh Kumar

DATE        DESCRIPTION              AMOUNT
─────────────────────────────────────────────
Dec 15      Room charge              ₹5,000
Dec 15      Restaurant (staff added) ₹1,800
Dec 16      Room charge              ₹5,000
Dec 16      Bar (staff added)        ₹1,200
Dec 16      Spa (staff added)        ₹2,500
Dec 17      Room charge              ₹4,000
Dec 17      Parking (staff added)      ₹300
─────────────────────────────────────────────
Subtotal                            ₹19,800
GST (12%)                            ₹1,788 (rooms)
GST (5%)                               ₹280 (F&B/Spa)
─────────────────────────────────────────────
Total                               ₹21,868
Deposit paid                        (₹4,500)
─────────────────────────────────────────────
Balance due                         ₹17,368

Payment: [Collect Payment]
```

No app. No split. No disputes UI.
Staff adds charges manually. Night audit auto-posts room charges.
That's it. Hotel can operate fully.

---

## V1 Cross-Module — Simplified

```
V1 cross-module (Restaurant → Folio):

  Waiter closes bill
  → "Charge to room?" → Staff enters room number
  → System verifies room exists and is checked-in
  → Charge added to folio (staff-initiated)

NOT in V1:
  → Guest self-selects "charge to room" on customer app
  → Real-time instant sync without staff action
  
V1 still works — just requires one staff action vs zero.
Simpler to build, same result for hotel.
```

---

## Phase Roadmap for Folio

```
V1 (Now):
  Auto folio creation
  Night audit (room charge posting)
  Manual charge entry by staff
  Basic cross-module (staff-initiated)
  GST calculation
  Basic invoice / receipt
  Checkout settlement

Phase 2:
  Real-time guest app folio view
  Folio split (corporate, shared room)
  In-app dispute flow
  Comp workflow with approval
  Group master folio
  Night audit full reports

Phase 3:
  Mini bar QR scan auto-posting
  Dispute pattern detection
  Advanced GST reports
  Multi-currency folio
  Accounting software integration (Tally, QuickBooks)
```

---

## Summary

```
V1 Folio goal:
  Guest checks in → Charges accumulate → Guest checks out → Payment collected

Everything else → Later.

Don't build Phase 2 features in V1.
Don't design V1 in a way that breaks Phase 2 later.

Key: Data model must support future features —
     even if UI doesn't expose them in V1.
```

---

## Important: Data Model Must Be Future-Ready

Even though V1 UI is simple — DB design must plan for future:

```
folio_charges table:
  id, folio_id, charge_date, description,
  amount, tax_amount, tax_rate,
  charge_type (ROOM/FOOD/SPA/BAR/MISC/COMP/ADJUSTMENT),
  source_module (HOTEL/RESTAURANT/BAR/SPA/MANUAL),
  source_reference_id (order_id, booking_id etc.),
  posted_by (staff_id or SYSTEM),
  is_void, void_reason, void_by,
  created_at

This structure supports:
  V1:      Basic charge entry ✓
  Phase 2: Split folio, disputes, comps ✓
  Phase 3: Advanced analytics, pattern detection ✓
```

Build simple UI. But build right data model from day 1.
