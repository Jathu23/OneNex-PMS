# Rate & Revenue Management — Overview & Smart Features
> Hotel Module → Rate & Revenue Management
> Status: V1 required (simplified) | Advanced features Phase 2+

---

## Existing Systems — Problems

| System | Weakness |
|--------|---------|
| Oracle OPERA | Complex setup, requires dedicated revenue manager |
| IDeaS / Duetto | Excellent but ₹5-15L/year — only big hotels |
| Mews | Basic rate plans, limited intelligence |
| Cloudbeds | Basic rate setup only — no yield, no intelligence |

**Gap we fill:** Revenue management tools are either too expensive (enterprise) or too simple (basic).
Middle market hotels have nothing powerful yet affordable.

---

## 1. Rate Plan Setup

### V1 Approach — Simple Builder
```
Step 1: Base rate → ₹4,000/night
Step 2: Variations → Weekend ₹5,000 | Peak season dates + price
Step 3: Inclusions → Room only / Breakfast / All meals
Step 4: Cancellation → Free cancel / Non-refundable / Custom
Step 5: Visibility → Website / Booking.com / Expedia / Corporate only
[Save]
```

Simple hotels: 5 minutes to set up.
Complex hotels: Full control available by going deeper.

---

## 2. Visual Rate Calendar *(Phase 2)*

```
RATE CALENDAR — December 2026 — Double Room

     Mon    Tue    Wed    Thu    Fri    Sat    Sun
W1  [4000] [4000] [4000] [4000] [5000] [5000] [4000]
W2  [4000] [4000] [4000] [4000] [5000] [5000] [4000]
W3  [4000] [4000] [4000] [4000] [5000] [5000] [4000]
W4  [7000] [7000] [7000] [7000] [8000] [8000] [7000] ← Peak season
W5  [7000] [7000]

Color coded:
  White  = Base rate
  Yellow = Weekend rate
  Red    = Peak season
  Green  = Discount / promo

Click any date → Edit rate directly on calendar.
Errors visible at a glance.
```

---

## 3. Yield Management *(Phase 2)*

### Simple Yield Visualizer
```
YIELD MANAGEMENT — Dec 15

Current occupancy: 73% (146 of 200 rooms)

Room Type    Current    Suggested    Action
Double       ₹4,000     ₹4,600      [Apply +15%]
Single       ₹3,000     ₹3,450      [Apply +15%]
Suite        ₹12,000    ₹12,000     [No change]

Why: 73% occupancy = high demand zone
     Historical: Same date last year → 89% final
     
[Apply All]  [Customize]  [Dismiss]
```

### Yield Rules (Set Once, Runs Forever)
```
0%  - 40% occupied  → Base rate
41% - 60% occupied  → Base rate + 10%
61% - 75% occupied  → Base rate + 20%
76% - 90% occupied  → Base rate + 35%
91% - 100% occupied → Base rate + 50%

System checks occupancy hourly → Auto-adjusts rates across all channels
Zero staff action needed after setup.
```

---

## 4. Revenue KPI Dashboard *(Phase 2)*

```
TODAY              THIS MONTH        VS LAST YEAR
Occupancy: 73%     Occupancy: 68%    +5%  ▲
ADR:       ₹4,850  ADR:       ₹4,620 +8%  ▲
RevPAR:    ₹3,540  RevPAR:    ₹3,141 +14% ▲
Revenue:   ₹7.08L  Revenue:   ₹1.8Cr +18% ▲

Tooltips for first-time users:
  ADR    = Average price per room sold
  RevPAR = Revenue per every available room (occupancy × ADR)

TOP CHANNEL:
  Direct: 40% bookings, 0% commission ← Best margin
  Booking.com: 35%, 10% commission
  Walk-in: 25%, 0% commission

TONIGHT:
  Forecast: 89% occupancy | 22 rooms unsold
  Suggestion: Raise rates +20% → Capture ₹44,000 more
  [Apply Now]
```

---

## 5. Length of Stay (LOS) Restrictions

### V1: Not needed
### Phase 2:
```
Rule: Dec 20 - Jan 5 → Minimum 3 nights
Rule: Every Fri/Sat → Minimum 2 nights

Why: Short stays during peak block longer bookings = lost revenue

Guest tries Dec 22 → Dec 23 (1 night) → ❌ Blocked
Guest tries Dec 22 → Dec 25 (3 nights) → ✅ Allowed
```

---

## 6. Corporate & Group Rate Management

### V1 Approach (Basic)
```
Corporate account:
  Company name, contracted rate, valid dates
  Staff manually applies during booking
```

### Phase 2 Approach (Smart)
```
Company: Infosys Ltd
  Contract rate:    ₹3,000/night (-25%)
  Valid:            Jan 1 - Dec 31, 2026
  Volume:           100 room nights committed | 67 used
  Status:           ✅ Active

Auto-apply: System detects corporate account → Rate applied automatically
Auto-alert: Contract expires in 30 days → Sales manager notified
Auto-revert: Expired contract → Rack rate applied (no manual override needed)
```

---

## 7. Competitor Rate Monitoring *(Phase 3)*

```
COMPETITOR RATES — Dec 15

Hotel           Stars  Price    vs Us
Our Hotel       ★★★★  ₹4,800   —
Hotel Leela     ★★★★  ₹5,200   +₹400  ← We're cheaper
Taj Gateway     ★★★★★ ₹8,500   +₹3,700
Marriott        ★★★★  ₹5,800   +₹1,000

Analysis:
  We are ₹400 cheaper than nearest competitor
  Room to raise rate without losing bookings
  Suggestion: Raise to ₹5,100 → Still competitive + ₹300/room
  22 unsold rooms × ₹300 = ₹6,600 extra tonight
  [Apply]

Auto-alert: "⚠ Hotel Leela dropped to ₹3,800 — now below us"
```

---

## 8. Demand Forecasting *(Phase 2+)*

```
FORECAST — Next 30 Days

Dec 15  89% forecast  73% current  → Raise rates ▲
Dec 20  97% forecast  45% current  → Raise significantly ▲▲
Dec 25  100% forecast 82% current  → Max rates
Dec 26  95% forecast  38% current  → Local event (concert) — raise ▲▲
Jan 5   45% forecast  31% current  → Slow — consider promo
```

---

## V1 vs Later

```
V1 MUST HAVE:
  ✅ Basic rate plan setup (base rate per room type)
  ✅ Date-based rate overrides (weekends, peak season)
  ✅ Multiple rate plans (flexible, non-refundable)
  ✅ OTA rate (with commission markup)
  ✅ Corporate account rates (manual apply)
  ✅ Basic revenue report (daily revenue, occupancy %)

PHASE 2:
  ➕ Visual rate calendar
  ➕ Yield management (auto price by occupancy %)
  ➕ Revenue KPI dashboard (ADR, RevPAR, GOPPAR)
  ➕ LOS restrictions
  ➕ Corporate rate auto-apply + expiry alerts
  ➕ Basic demand forecasting

PHASE 3:
  ➕ Competitor rate monitoring
  ➕ AI-based dynamic pricing
  ➕ Local event demand detection
  ➕ Revenue manager recommendation engine
```

---

## Comparison Summary

| Feature | Existing Systems | Our System |
|---------|-----------------|------------|
| Rate setup | Complex multi-screen | Simple step-by-step |
| Rate visibility | Table-based | Visual calendar (Phase 2) |
| Yield management | Expert-only / expensive | Simple rules, auto-apply (Phase 2) |
| KPI dashboard | Numbers only | Explained + actionable (Phase 2) |
| Corporate rates | Spreadsheet / manual | Auto-apply by account (Phase 2) |
| Competitor monitoring | Manual weekly | Auto alerts (Phase 3) |
| Forecasting | Separate expensive tools | Built-in (Phase 2+) |
