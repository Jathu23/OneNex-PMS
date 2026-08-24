# Setup & Configuration — 03: Rate Plan Setup
> Hotel Module → Setup & Configuration → Areas 3 + 4 + 5 + 8 (Unified)
> Covers: Rate Setup + Policy Setup + Payment Setup + Channel Setup
> Foundation for: Booking Flow, Availability Logic, OTA Sync, Folio & Billing

---

## Why These 4 Areas Are One Concept

```
Rate Setup       → WHAT IS THE PRICE?
Policy Setup     → WHAT ARE THE RULES?
Payment Setup    → HOW TO COLLECT MONEY?
Channel Setup    → WHERE TO SELL IT?

These 4 = ONE Rate Plan.

A Rate Plan is a complete bookable package:
"Bed & Breakfast — Flexible"
  → ₹5,000/night + breakfast included
  → Free cancellation up to 24 hrs
  → 20% deposit at booking, balance at check-in
  → Sold on Direct + Booking.com + MakeMyTrip

Separating these 4 into different screens = the mistake every existing system makes.
We design them as one unified builder.
```

---

## Existing Systems — Problems

| System | Problem |
|--------|---------|
| Oracle OPERA | Rate plans in one screen, policies in another, channels in a third. Staff jumps between 3 screens to understand one Rate Plan. No single view of what a guest is actually buying. |
| Mews | Clear rate plans, but payment rules are completely separate. No template to start from — must build from scratch every time. |
| Cloudbeds | OTA rates manually entered per channel. Change base rate → manually update each OTA separately. No automatic sync of rate changes. |
| All systems | Creating a rate plan = 30+ fields across multiple screens. Small hotel owner gives up, leaves defaults, then billing errors occur. |

---

## Our Design Principles

### 1. Rate Plan Templates (Start Smart)
```
"What kind of rate plan do you want to create?"

  ○ Flexible          → Free cancellation, standard rate, all channels
  ○ Non-Refundable    → 10-15% discount, full charge on cancel, all channels
  ○ Advance Purchase  → Discount for booking 30+ days ahead, pay now
  ○ Long Stay         → 7+ nights discount, relaxed cancellation
  ○ Bed & Breakfast   → Breakfast included, moderate cancellation
  ○ Corporate         → Negotiated rate, invoice billing, private visibility
  ○ Custom            → Start from scratch

Template selected → 80% pre-filled → Hotel adjusts 20%
```

### 2. One-Screen Rate Plan Builder
```
┌─────────────────────────────────────────────────────────────┐
│ RATE PLAN: Bed & Breakfast — Flexible                       │
├──────────────┬──────────────┬──────────────┬────────────────┤
│  💰 RATE     │  📋 POLICY   │  💳 PAYMENT  │  📡 CHANNELS  │
│              │              │              │                │
│ Base: ₹5,000 │ Check-in:3PM │ 20% deposit  │ ✅ Direct      │
│ + Breakfast  │ Check-out:11A│ at booking   │ ✅ Booking.com │
│              │ Cancel: Free │ Balance at   │ ✅ MakeMyTrip  │
│ Weekend: +10%│ until 24 hrs │ check-in     │ ☐ Expedia      │
│              │ No-show: 1nt │              │ ☐ Agoda        │
└──────────────┴──────────────┴──────────────┴────────────────┘

One Rate Plan. Full picture. No jumping between screens.
```

### 3. Visual Rate Calendar
```
         JAN 2025
MON  TUE  WED  THU  FRI  SAT  SUN
                 1    2    3    4    5
               4000 4000 4500 4500 4000   ← Weekend premium auto-applied

 6    7    8    9   10   11   12
4000 4000 4000 4000 4500 4500 4000

[DIWALI BLOCK: OCT 20-25 → ₹8,000 — Override]  ← Holiday override visible

Color: Green=Available | Yellow=Low inventory | Red=Sold out | Blue=Override
```
> Phase 2 feature — V1 uses manual date override table

### 4. Cancellation Policy as Reusable Building Block
```
Cancellation policies are separate entities — reused across multiple rate plans.

POLICY: "Flexible-24"
  → Free cancellation up to 24 hours before check-in
  → After 24 hrs: First night charge
  → No-show: Full stay charge

POLICY: "Non-Refundable"
  → No refund on cancellation (any time)
  → No-show: Full stay charge

POLICY: "Tiered-7day"
  → More than 7 days before: Free
  → 3-7 days before: 50% charge
  → Less than 3 days: Full charge

One policy → linked to many rate plans.
Change policy once → all linked rate plans update automatically.
```

### 5. Channel Rate Logic
```
Base Rate: ₹5,000

Rate Parity ON (same rate everywhere — OTA contract requirement):
  Direct Booking:  ₹5,000  (no commission — hotel keeps full ₹5,000)
  Booking.com:     ₹5,000  (hotel pays 15% commission = ₹750 to OTA)
  MakeMyTrip:      ₹5,000  (hotel pays 12% commission = ₹600 to OTA)

Rate Parity OFF (different rates per channel — discourage):
  Direct Booking:  ₹4,750  (cheaper to encourage direct bookings)
  Booking.com:     ₹5,000
  MakeMyTrip:      ₹5,000

Rate Parity Enforcement:
  System warns if direct rate > OTA rate.
  OTA contracts usually require parity.
  System protects hotel from accidental violation.
```

### 6. Rate Plan Visibility Levels
```
PUBLIC      → Everyone sees it (guests, OTAs, walk-ins)
PRIVATE     → Requires promo code or direct link
CORPORATE   → Only guests with corporate account linked
INTERNAL    → Staff only (walk-in negotiated rates, not shown to guests)
```
> AGENT visibility (travel agents) → Phase 2

### 7. LOS Restrictions (Length of Stay)
```
Peak Season Rules:
  Minimum Stay: 3 nights (Dec 20 – Jan 5)
  Maximum Stay: 14 nights (any time)
  Must arrive on: Friday or Saturday (event weekends only)

Guest tries to book 2 nights in Dec → System blocks, shows clear message:
"Minimum 3-night stay required for this period.
 Earliest available: check-in Dec 20, check-out Dec 23."
```
> Phase 2 feature

### 8. Meal Plan Pricing
```
ROOM_ONLY       ₹4,000  (no meals)
WITH_BREAKFAST  ₹4,500  (+ ₹500/room/night)
HALF_BOARD      ₹5,500  (+ ₹1,500 — breakfast + dinner)
FULL_BOARD      ₹6,500  (+ ₹2,500 — all 3 meals)
ALL_INCLUSIVE   ₹9,000  (meals + drinks + activities)

Meal charge auto-routes to Restaurant Module as a folio entry.
No manual entry needed. Setup once → works automatically every booking.
```

---

## Data Model

### Rate Plan (Core Entity)
```
id, hotel_id
name                    "Bed & Breakfast — Flexible"
code                    "BBF"
description             (shown to guests at booking)
template_source         FLEXIBLE / NON_REFUNDABLE / ADVANCE / LONG_STAY /
                        BREAKFAST / CORPORATE / CUSTOM
meal_plan               ROOM_ONLY / BREAKFAST / HALF_BOARD / FULL_BOARD / ALL_INCLUSIVE
meal_charge_per_night   decimal (0 if ROOM_ONLY)
pricing_model           PER_ROOM / PER_PERSON
visibility              PUBLIC / PRIVATE / CORPORATE / INTERNAL
is_active               bool
valid_from              date nullable
valid_until             date nullable
display_order           int
```

### Rate Rules
```
RatePlanRoom
  rate_plan_id, room_type_id, base_rate

RatePlanDateOverride
  rate_plan_id, date_from, date_to, rate, override_reason

RatePlanDayRule
  rate_plan_id, day_of_week[], multiplier    ← e.g. FRI+SAT = 1.10 (10% premium)

RatePlanLOSRule
  rate_plan_id, date_from, date_to, min_nights, max_nights, arrival_days[]

OccupancyPricing
  rate_plan_id, extra_adult_charge, child_charge, infant_charge
```

### Policy Rules
```
CancellationPolicy
  id, hotel_id
  name                  "Flexible-24"
  type                  FREE_UNTIL / PARTIAL / NON_REFUNDABLE / TIERED
  rules                 JSON (time windows + charge percentages)

RatePlanPolicy
  rate_plan_id
  cancellation_policy_id    FK → CancellationPolicy (reusable)
  check_in_time             "15:00"
  check_out_time            "11:00"
  early_checkin_fee         nullable (₹/hour or flat)
  late_checkout_fee         nullable
  no_show_charge            FIRST_NIGHT / FULL_STAY / NONE
  child_allowed             bool
  child_age_limit           int
  infant_free_until         int (age in years)
  pet_allowed               bool
  pet_fee                   nullable
```

### Payment Rules
```
RatePlanPayment
  rate_plan_id
  collection_type           PAY_NOW / DEPOSIT / PAY_AT_HOTEL / CC_GUARANTEE
  deposit_percentage        nullable (e.g. 20)
  deposit_due               AT_BOOKING / DAYS_BEFORE_ARRIVAL
  deposit_due_days          nullable int
  balance_collection        AT_CHECKIN / AT_CHECKOUT
  auto_charge_no_show       bool
  accepted_methods          JSON [CARD, UPI, CASH, BANK_TRANSFER]
  refund_timeline_days      int
```

### Channel Rules
```
RatePlanChannel
  rate_plan_id
  channel                   DIRECT / BOOKING_COM / MAKEMYTRIP / EXPEDIA /
                            AIRBNB / AGODA / GOIBIBO
  channel_rate_override     nullable decimal (if different from base)
  commission_percentage     nullable decimal
  commission_borne_by       HOTEL / GUEST
  is_active                 bool

RateParity
  rate_plan_id
  enforce_parity            bool
  alert_on_violation        bool
```

---

## Key Relationships

```
RatePlan → RoomTypes        (many-to-many — one plan can cover multiple room types)
RatePlan → CancellationPolicy (reusable — many plans share one policy)
RatePlan → Channels         (which OTAs sell this plan)
RatePlan → DateOverrides    (seasonal / holiday pricing)
RatePlan → LOSRules         (minimum stay restrictions)

Booking  → RatePlan         (which plan was active when booked)
Folio    → RatePlan         (audit trail — rate at time of booking, immutable)
OTA Sync → RatePlanChannel  (what to push to each OTA)
```

---

## V1 vs Phase Split

| Feature | V1 | Phase 2 | Phase 3 |
|---------|:--:|:-------:|:-------:|
| Rate plan with name + base rate + room type | ✅ | | |
| Rate plan templates (pre-filled) | ✅ | | |
| Meal plan (Room Only + Breakfast) | ✅ | | |
| Cancellation policy: 3 types (Free / Partial / Non-Refundable) | ✅ | | |
| Payment: Pay Now / Deposit / Pay at Hotel / CC Guarantee | ✅ | | |
| Weekend / holiday date override (manual) | ✅ | | |
| Channel: Direct booking | ✅ | | |
| Channel: Booking.com + MakeMyTrip (OTA) | ✅ | | |
| Rate plan visibility (Public / Private / Corporate / Internal) | ✅ | | |
| Visual rate calendar | | ✅ | |
| Tiered cancellation policy | | ✅ | |
| LOS restrictions (min/max stay, arrival day rules) | | ✅ | |
| Occupancy-based pricing (extra adult / child charge) | | ✅ | |
| Rate parity monitoring + alerts | | ✅ | |
| Yield management (auto price by occupancy %) | | ✅ | |
| Channel-specific rate override | | ✅ | |
| Full meal plan (Half Board / Full Board / All Inclusive) | | ✅ | |
| Agent visibility + travel agent portal | | ✅ | |
| AI dynamic pricing | | | ✅ |
| Competitor rate monitoring | | | ✅ |
| GDS connectivity (Sabre, Amadeus) | | | ✅ |
