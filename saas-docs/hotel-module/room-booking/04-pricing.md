# Room Booking — 04: Pricing at Booking
> Hotel Module → Room Booking → Topic 4 of 7

---

## Overview

Guest booking பண்றப்போ — which rate apply ஆகும், how is the price calculated என்று இது cover பண்றது. Same room-ku same dates-la different guests different prices pay பண்ணலாம் — இது intentional, system-driven pricing strategy.

---

## Rate Types — All Possible Rates

ஒரு room-ku multiple rates இருக்கலாம் same time-la:

```
Room 102 — Double Room (Base: ₹4,000/night)
  ├── Rack Rate          ₹4,000   (default, no discount, no conditions)
  ├── Weekend Rate       ₹5,000   (Friday, Saturday nights)
  ├── Peak Season Rate   ₹7,000   (Dec 20 - Jan 5, summer holidays)
  ├── Early Bird Rate    ₹3,200   (book 30+ days in advance)
  ├── Last Minute Rate   ₹3,500   (book within 48 hours of check-in)
  ├── Corporate Rate     ₹3,000   (Company ABC contract — not public)
  ├── OTA Rate           ₹4,400   (Booking.com — includes commission)
  └── Promo Rate         ₹3,600   (limited time offer, promo code)
```

---

## Rate Decision Logic — Priority Order

Booking request வரும்போது system இந்த order-la check பண்ணும்:

```
Step 1: Corporate account?
  → Guest's company has a contract with this hotel?
  → YES: Apply Corporate Rate (pre-negotiated, non-public)
  → NO: Continue

Step 2: Valid promo code entered?
  → YES: Apply Promo Rate (validate: active, not expired, usage limit not reached)
  → NO: Continue

Step 3: What is the booking source?
  → OTA (Booking.com, Expedia...): Apply OTA Rate (includes commission markup)
  → Direct (website/app/phone): Continue

Step 4: How many days before check-in?
  → 30+ days before: Early Bird Rate applies (if hotel has enabled this)
  → Within 48 hours: Last Minute Rate applies (if enabled)
  → Otherwise: Continue

Step 5: What dates are selected? (Check each night)
  → Peak season dates: Peak Season Rate
  → Weekend nights (Fri/Sat): Weekend Rate
  → Normal weekday: Rack Rate
```

---

## Rate Plan — More Than Just a Price

Rate plan = complete package of rules, not just a number.

**Example: "Standard Flexible" Rate Plan**
```
Price:              ₹4,000/night
Meal inclusion:     Breakfast included
Cancellation:       Free cancel up to 48 hours before check-in
Deposit required:   No deposit (pay at hotel)
Minimum stay:       1 night
Refundable:         Yes
Modify allowed:     Yes
```

**Example: "Non-Refundable Saver" Rate Plan**
```
Price:              ₹3,200/night (20% cheaper)
Meal inclusion:     Room only (no breakfast)
Cancellation:       No refund at any time
Deposit required:   Full payment at booking
Minimum stay:       1 night
Refundable:         No
Modify allowed:     No
```

**Guest trade-off:** Cheap price ↔ No flexibility. Guest chooses based on their certainty of travel.

---

## Night-by-Night Calculation

System calculates rate EACH night separately — not one flat rate for all nights.

**Example:**
```
Guest books: Dec 15 (Fri) → Dec 18 (Mon) — 3 nights

Night 1: Dec 15 (Friday)   → Weekend Rate    ₹5,000
Night 2: Dec 16 (Saturday) → Weekend Rate    ₹5,000
Night 3: Dec 17 (Sunday)   → Rack Rate       ₹4,000
                                             ────────
                              Room Subtotal  ₹14,000

Meal inclusion (Breakfast, 2 adults × 3 nights × ₹300):  ₹1,800
GST (12% on room + breakfast):                            ₹1,896
                                             ────────────────────
                              Grand Total    ₹17,696
```

---

## Meal Inclusions (Rate Plan Packages)

Rate plan can include meals — system tracks these:

| Code | Name | Includes |
|------|------|---------|
| RO | Room Only | Nothing included |
| BB | Bed & Breakfast | Breakfast included |
| HB | Half Board | Breakfast + Dinner |
| FB | Full Board | All 3 meals |
| AI | All Inclusive | Meals + selected beverages |

**How inclusions work with Restaurant Module:**
```
Guest on BB (Bed & Breakfast) rate goes to hotel restaurant
  → Breakfast order placed
  → Restaurant system checks: Guest on BB? YES
  → Breakfast marked as ₹0 (already included in room rate)
  → Charged to room folio as ₹0 (tracked but not billed again)
  → Lunch ordered: Charged normally to folio
```

---

## Rate Visibility by Booking Source

Same room, different prices for different channels:

```
Direct website / app   → ₹4,000  (best rate — no commission)
Booking.com            → ₹4,400  (includes 10% OTA commission)
Expedia                → ₹4,480  (includes 12% OTA commission)
Corporate (Company A)  → ₹3,000  (contract rate — not visible publicly)
Group                  → ₹3,500  (negotiated bulk rate)
```

**Rate Parity Rule:**
```
Direct rate must always be ≤ OTA rate.
Hotel's own website always offers the best deal.

Why: Encourage guests to book direct → save commission → better margin
     OTA contracts sometimes require parity — monitor this
```

---

## Yield Management (Layer 3+)

System auto-adjusts price based on current occupancy:

```
Hotel sets up yield rules:
  0%  - 40% occupied  → Base rate           (attract bookings)
  41% - 70% occupied  → Base rate + 15%     (growing demand)
  71% - 90% occupied  → Base rate + 30%     (high demand)
  91% - 100% occupied → Base rate + 50%     (peak, maximize revenue)

Real example (Dec 15):
  Current occupancy: 85% (170 of 200 rooms sold)
  Double room base:  ₹4,000
  Yield multiplier:  +30%
  
  Auto-adjusted price: ₹4,000 × 1.30 = ₹5,200
  No staff action needed — system auto-raises price
```

---

## Length of Stay (LOS) Restrictions

Hotel can set minimum stay rules:

```
Rule: "Minimum 2 nights on weekends (Fri/Sat)"
Rule: "Minimum 3 nights during Dec 20 - Jan 2"

Guest tries: Dec 15 (Sat) → Dec 16 (Sun) — 1 night only
System: ❌ "Minimum 2 nights required for this period"
Guest must: Book Dec 15 → Dec 17 (2 nights) to proceed

Why hotels do this:
  Single-night stays during peak period = lost opportunity
  (That room could have been part of a 3-night booking)
  LOS restrictions maximize revenue during high-demand periods
```

---

## Dynamic / AI Pricing (Layer 4)

Advanced hotels use AI for automated pricing:

```
Inputs the AI considers:
  ├── Current occupancy
  ├── Historical demand for same dates last year
  ├── Local events (concerts, conferences, holidays)
  ├── Competitor pricing (rate shopping)
  ├── Booking pace (how fast rooms are selling)
  ├── Weather forecast (for leisure destinations)
  └── Days until check-in

Output:
  → Optimal price per room type per night
  → Updated automatically (hourly or daily)
  → No manual intervention needed
```

---

## Price Display to Guest

How guest sees pricing during booking:

```
Double Room — Dec 15 to Dec 18 (3 nights)
2 Adults

┌─────────────────────────────────────────────────────┐
│  Standard Flexible                    ₹14,000       │
│  ✓ Breakfast included  ✓ Free cancellation          │
│  Pay at hotel                                        │
│                                                      │
│  Non-refundable Saver           ₹11,200  SAVE 20%   │
│  ✗ Room only  ✗ No cancellation                     │
│  Full payment now                                    │
└─────────────────────────────────────────────────────┘

Before taxes:    ₹14,000  (Standard Flexible)
Breakfast incl:  ₹1,800   (already in rate)
GST (12%):       ₹1,896
─────────────────────────────────────────────────────
Total:           ₹15,896  per stay
```

---

## Pricing Configuration (Hotel Setup)

What hotel admin sets up:

```
Room Type: Double Room
  Base (Rack) Rate:     ₹4,000/night

  Rate Plans:
    ├── Standard Flexible:    ₹4,000 | BB | Free cancel 48hr | No deposit
    ├── Non-refundable Saver: ₹3,200 | RO | No cancel | Full pay now
    └── Corporate Plan:       ₹3,000 | BB | Free cancel 72hr | No deposit

  Date-based overrides:
    ├── Dec 20 - Jan 5:  ₹7,000 (peak season)
    ├── Every Fri/Sat:   ₹5,000 (weekend rate)
    └── Jun 1 - Aug 31: ₹6,000 (summer holidays)

  Early Bird:
    ├── Enabled: Yes
    ├── Days in advance: 30+
    └── Discount: 20% off base rate

  Last Minute:
    ├── Enabled: Yes
    ├── Window: within 48 hours
    └── Rate: ₹3,500 (flat)

  Yield Management:
    ├── Enabled: Yes (Layer 3)
    └── Rules: 0-40% → base, 41-70% → +15%, 71-90% → +30%, 91-100% → +50%

  OTA Rates (Channel Manager syncs these):
    ├── Booking.com: Base rate + 10%
    └── Expedia:     Base rate + 12%
```

---

## Pricing Summary Flow

```
Booking request arrives
        ↓
Identify guest type:
  Corporate account? → Corporate Rate
  Promo code?        → Promo Rate
  OTA source?        → OTA Rate
        ↓
No special type → Check dates night by night:
  Peak season?  → Peak Rate
  Weekend?      → Weekend Rate
  Normal?       → Rack Rate
        ↓
Check advance booking:
  30+ days?      → Early Bird Rate (if cheaper, apply)
  Within 48 hrs? → Last Minute Rate
        ↓
Apply Yield Management adjustment (if enabled)
        ↓
Check LOS restrictions (minimum stay met?)
        ↓
Calculate: Per-night × nights
Add: Meal inclusions
Add: Taxes (GST / applicable taxes)
        ↓
Show final price to guest with breakdown
```
