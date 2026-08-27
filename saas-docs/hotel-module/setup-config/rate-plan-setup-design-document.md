# Rate Plan Setup — Design Document
> Hotel Module | Setup & Configuration | Areas 3 + 4 + 5 + 8 (Unified)
> Version: 1.0 | For Team Review
> Validated against: Apaleo, Oracle OPERA, Mews, Cloudbeds, Little Hotelier, Hotelogix
> HTNG Standard | Booking.com API | Expedia API | Agoda API

---

## 1. The Problem We Are Solving

After Room Setup defines what rooms exist, the system still cannot take a single booking.

```
ROOM SETUP answered:  "We have Standard Double rooms."

System still cannot answer:
  → How much does it cost?
  → Can the guest cancel for free?
  → How much deposit is needed now?
  → Which OTA websites can sell it?
  → What happens if they don't show up?

RATE PLAN answers ALL of these. Together. In one place.
```

Without Rate Plan Setup:

```
Guest arrives → "I want a Standard Double"
System: ✅ Room exists
System: ❓ What is the price?           → No answer
System: ❓ What are the rules?          → No answer
System: ❓ How do I collect money?      → No answer
System: ❓ Can I show this on OTA?      → No answer
→ CANNOT CREATE A SINGLE BOOKING
```

---

## 2. What is a Rate Plan?

A Rate Plan is NOT just a price. It is a **complete sellable package**.

```
"Bed & Breakfast — Flexible"

  PRICE      → LKR 12,000/night + breakfast included
  RULES      → Free cancel until 24 hrs, check-in 2 PM, no-show = 1 night charge
  PAYMENT    → 20% deposit at booking, balance at check-in
  CHANNELS   → Sold on Direct + Booking.com + Agoda

One Rate Plan = everything a guest needs to make a booking decision.
```

This is why Rate Setup + Policy Setup + Payment Setup + Channel Setup are ONE concept — not four separate screens. They cannot exist without each other.

---

## 3. Our Design Philosophy

```
"Simple by Default. Powerful when needed."
```

**Small Guesthouse (Simple):**
```
Select template: "Flexible"
Fill: room type, price
Done in 3 minutes.
System pre-fills: free cancel 24hrs, pay at hotel, direct booking only.
```

**Luxury Resort (Powerful):**
```
8 rate plans with derived rates cascading from BAR
Seasonal date overrides for 12 holiday periods
Weekend premium multipliers
Min-stay restrictions with CTA/CTD for peak dates
Corporate accounts with auto-invoicing
Packages: room + breakfast + spa + airport transfer with revenue split
All OTA channels with individual commission tracking and inventory allocation
Rate parity monitoring with auto-alerts
Full audit trail of every price change
```

Same 14 entities. Same system. Different depth of usage.

---

## 4. Competitive Analysis — What Every System Gets Wrong

| System | Best Feature | Critical Gap |
|--------|-------------|--------------|
| **Apaleo** | API-first, Time Slice (day-use rates) | OTA mapping is external — painful |
| **Mews** | Policy reusability, clean rate/policy separation | Non-refundable = fully rigid. Cannot allow date change. OTAs support this. |
| **Oracle OPERA** | Derived rates (best in industry), 3-tier hierarchy, package revenue splitting | Enterprise-only complexity. Too complex for small hotels. |
| **Cloudbeds** | Channel revenue allocation, AI forecasting (Signals AI) | **MAX 4 rate plans per room type on OTA** — arbitrary hard limit |
| **Little Hotelier** | Simple for tiny hotels | NO derived rates. Each room type manually updated. Does not scale. |
| **Hotelogix** | Dynamic pricing engine | Poor API documentation. Basic package support. |

**What ALL systems get wrong:**

```
1. No hybrid cancellation policy
   → Non-refundable but date-changeable (OTAs support this, no PMS does it well)

2. No rate plan audit trail
   → "Who changed the corporate rate last Tuesday?" — no system answers this cleanly

3. No derived rate cascade in small hotel systems
   → Corporate rate = BAR × 85%. Change BAR → corporate should auto-update.
   → Every small hotel does this manually. Every hotel makes errors.

4. No package revenue splitting (except OPERA)
   → "Executive Package LKR 25,000" split into rooms + F&B + spa revenue
   → Critical for department-level P&L reporting

5. CTA/CTD not supported in simpler systems
   → "No arrivals on New Year's Eve" — basic hotel operations need this

6. No booking impact warning before rate changes
   → "Changing this rate affects 12 future bookings" — no system warns you
```

**Our system fixes all 6.**

---

## 5. The Rate Plan — Bridge Between Rooms and Bookings

```
ROOM SETUP                RATE PLAN                 BOOKING
──────────                ─────────                 ───────

"Standard                 "Standard                 "Booking #2025-001
 Double exists"  ──────→   Double is            ──→  Standard Double
                           LKR 12,000,               Flexible
 "60 rooms                 free cancel,              Dec 20-22
  total"                   20% deposit,              LKR 28,800 total
                           Booking.com"              Deposit: LKR 5,760"

Rate Plan is the bridge.
Without it: rooms exist but cannot be sold.
With it: complete bookable product.
```

---

## 6. All 14 Entities — Purpose and Fields

---

### Entity 1: `RatePlan`
**The master record. Every other entity hangs off this.**

```
id
hotel_id
  → Multi-tenant isolation. Every entity needs this.

─── IDENTITY ───────────────────────────────────────────────

name
  → "Bed & Breakfast — Flexible"
  → Guest-facing. Shown at: booking widget, OTA listing,
    confirmation email, folio, invoice.
  → Most visible field in the rate plan system.
  → Must be: clear, descriptive, benefit-focused.

code
  → "BBF"
  → Internal identifier. Reports, OTA API, audit logs.
  → Why needed: "Bed & Breakfast — Flexible" too long for
    reports and floor map. "BBF" fits anywhere.
  → Unique per hotel.

description
  → "Enjoy a comfortable stay with breakfast included.
     Free cancellation up to 24 hours before arrival."
  → Guest-facing. Booking widget + OTA description.
  → Sells the rate plan. Uses benefit language, not just features.

status
  → DRAFT / ACTIVE / ARCHIVED
  → DRAFT: being configured. Not visible anywhere.
    Allows full setup before going live.
  → ACTIVE: live. Bookable on all configured channels.
  → ARCHIVED: was active, now retired. History preserved.
    Cannot book. Reports still show historical data.
  → OUR INNOVATION: DRAFT state. No competitor has this.
    All systems: only active/inactive. Hotels go live with
    half-configured rate plans by accident. DRAFT prevents this.

─── MEAL PLAN ──────────────────────────────────────────────

template_source
  → FLEXIBLE / NON_REFUNDABLE / ADVANCE_PURCHASE /
    LONG_STAY / BREAKFAST / CORPORATE / CUSTOM
  → Which template was used to create this rate plan.
  → Template pre-fills 80% of fields.
  → CUSTOM = built from scratch.
  → Tracked for: analytics (which templates are popular),
    future reset to template defaults.

meal_plan
  → ROOM_ONLY / BREAKFAST / HALF_BOARD / FULL_BOARD / ALL_INCLUSIVE
  → What food is included in the rate.
  → Drives auto-posting to folio:
      BREAKFAST   → charge posts at 10:30 AM daily (after service ends)
      HALF_BOARD  → breakfast 10:30 AM + dinner 10:30 PM daily
      FULL_BOARD  → breakfast + lunch 2:00 PM + dinner 10:30 PM daily
  → OTA filter: "Breakfast Included" checkbox uses this field.
  → Routes to Restaurant Module as revenue.

meal_charge_per_night
  → decimal. What the meal costs within the total rate.
  → ROOM_ONLY: 0
  → BREAKFAST: LKR 800 (2 breakfasts included)
  → Why needed: Revenue split.
    "Deluxe B&B LKR 18,000" =
      Room revenue:  LKR 17,200
      F&B revenue:   LKR    800
    Critical for: department P&L, correct tax (room gets TDL, food doesn't).

─── PRICING MODEL ──────────────────────────────────────────

pricing_model
  → PER_ROOM: LKR 12,000 regardless of 1 or 2 guests (most common)
  → PER_PERSON: LKR 6,000 × number of adults (hostels, some resorts)
  → Changes how booking total is calculated at every step.

─── VISIBILITY ─────────────────────────────────────────────

visibility
  → PUBLIC:    Anyone sees this. OTA + direct + walk-in.
  → PRIVATE:   Promo code or direct link only. Hidden by default.
  → CORPORATE: Only guests with linked corporate account.
  → INTERNAL:  Staff only. Walk-in negotiated rates.
               Never shown to guests.
  → Why: Corporate rate LKR 9,500 must NEVER appear on OTA
    next to public rate LKR 12,000. Guest confusion + rate issues.

─── AVAILABILITY ────────────────────────────────────────────

valid_from / valid_until
  → nullable dates. Rate plan only bookable in this window.
  → "Summer Special": valid Jun 1 – Aug 31 only.
  → null = always valid (no restriction).
  → OTA: automatically de-listed after valid_until date.

is_active
  → bool. Quick on/off switch within active status.
  → Seasonal: active in summer, inactive in off-season.
  → Different from status: status = lifecycle. is_active = operational switch.

display_order
  → int. Booking widget order.
  → Hotel decides: show cheapest first (price anchor) or best first.

─── REVENUE MANAGEMENT ─────────────────────────────────────

is_yieldable
  → bool. Can yield management engine auto-adjust this rate?
  → true: Revenue manager or Phase 3 AI can raise/lower automatically.
  → false: Fixed rate. Never auto-adjusted.
           Use for: contracted corporate rates, long-term agreements.
  → From Oracle OPERA — industry standard field.

min_adr / max_adr
  → decimal nullable. Rate floor and ceiling.
  → min_adr: LKR 8,000 — yield engine CANNOT go below this.
  → max_adr: LKR 30,000 — yield engine CANNOT go above this.
  → Protects: contracted rates from being undercut by algorithm.
  → Phase 2 feature. Fields exist in V1 data model (future-ready).

revenue_category
  → ROOMS / PACKAGES / CORPORATE / PROMOTIONAL / LONG_STAY
  → Groups rate plans for revenue reporting.
  → "How much revenue came from corporate vs promotional rates?"
  → From Oracle OPERA's rate category concept.

─── DERIVED RATE ────────────────────────────────────────────

parent_rate_plan_id
  → nullable FK → RatePlan
  → If this is a derived rate plan: points to parent.
  → "Corporate rate" derived from "BAR".
  → null = base rate (not derived from anything).
  → See Entity 12 (DerivedRatePlan) for full cascade logic.

─── AUDIT ───────────────────────────────────────────────────

created_at / updated_at / created_by / updated_by
```

---

### Entity 2: `RatePlanRoom`
**Price per room type under this rate plan. Derived rate support built in.**

```
rate_plan_id      FK → RatePlan   ↘
room_type_id      FK → RoomType   → Composite primary key

base_rate
  → For non-derived: actual price per night.
  → For derived: auto-calculated (read-only field).
  → Night audit posts this to folio.
  → OTA receives this as the room rate.

is_derived
  → bool. Is this rate auto-calculated from parent?

derived_from_rate_plan_id
  → nullable FK → RatePlan (parent plan)
  → "Corporate Deluxe" derived from "BAR Deluxe"

adjustment_type
  → PERCENTAGE / FLAT nullable

adjustment_value
  → decimal nullable
  → PERCENTAGE: -15 means 15% discount from parent
  → FLAT: -2000 means LKR 2,000 less than parent

adjustment_direction
  → DISCOUNT / PREMIUM

rounding_rule
  → NONE / NEAREST_100 / ROUND_UP / ROUND_DOWN
  → BAR Deluxe: LKR 13,500 × 0.85 = LKR 11,475
  → NEAREST_100: rounds to LKR 11,500 (cleaner number to show guest)
```

**Why one rate plan for multiple room types:**
```
"Flexible" rate plan:
  Standard Double  → LKR 12,000/night  (4 records in RatePlanRoom)
  Deluxe King      → LKR 18,000/night
  Junior Suite     → LKR 28,000/night
  Suite            → LKR 45,000/night

Without this entity: need 4 separate rate plans named
"Flexible Standard", "Flexible Deluxe", "Flexible Suite"...
→ 6 rate plan types × 4 room types = 24 rate plans to manage.
With this entity: 6 rate plans. Much cleaner.
```

---

### Entity 3: `RatePlanDateOverride`
**Seasonal and holiday pricing. CTA/CTD for precise demand control.**

```
id
rate_plan_id      FK → RatePlan
room_type_id      FK → RoomType

date_from / date_to
  → The date range this override applies.

rate
  → Override price for this date range.
  → "Flexible Standard" normally LKR 12,000.
    Dec 20 – Jan 5 override: LKR 22,000.

override_reason
  → text nullable. Internal reference.
  → "Christmas & New Year peak"
  → "Galle Literary Festival weekend"
  → Staff understands WHY 6 months later.

closed_to_arrival (CTA)
  → bool. Guests CANNOT check-in on dates in this range.
  → "Dec 31 — no check-ins on New Year's Eve"
  → All existing bookings with earlier arrival: unaffected.
  → New bookings with Dec 31 arrival date: blocked.
  → OTA compliance: Booking.com will not show this rate plan
    for Dec 31 arrivals. Sync is automatic.
  → Research gap: missing from simpler PMS systems.

closed_to_departure (CTD)
  → bool. Guests CANNOT check-out on dates in this range.
  → "Jan 1 — no checkouts on New Year's Day"
  → Forces guests to stay through the date.
  → Combined with min_nights: powerful yield tool.
  → Prevents 1-night stays that fragment peak inventory.
```

**Override priority logic:**
```
Multiple overrides can exist. Specificity wins:
  Specific room type override > All room types override > Base rate
  Date override > Day rule > Base rate
  More specific date range > Broader date range
```

---

### Entity 4: `RatePlanDayRule`
**Automatic recurring price multiplier. Weekend premiums without manual entries.**

```
id
rate_plan_id      FK → RatePlan
room_type_id      FK → RoomType

days_of_week
  → JSON: ["FRI", "SAT"]
  → Which days this rule applies every week.

multiplier
  → decimal.
  → 1.10 = 10% weekend premium
  → 1.25 = 25% premium (peak demand day)
  → 0.85 = 15% discount (off-peak day)
```

**Why separate from DateOverride:**
```
DateOverride = specific date ranges (Christmas, Galle Lit Fest)
DayRule      = recurring every week automatically (every Fri + Sat)

Without DayRule:
  Must create 52 weekend date overrides per year.
  Miss one: that weekend priced wrong.

With DayRule:
  One record. Runs every week. Forever.
  Zero maintenance.

Priority: If DateOverride AND DayRule both apply → DateOverride wins.
```

---

### Entity 5: `RatePlanLOSRule`
**Length of Stay restrictions. Peak season minimum stay. CTA/CTD control.**

> LOS = Length of Stay

```
id
rate_plan_id      FK → RatePlan
room_type_id      FK → RoomType

date_from / date_to
  → Peak period this restriction covers.

min_nights
  → int. Minimum stay required during this period.
  → Peak: 3 nights minimum on Dec 20 – Jan 5.
  → Without: guests book 1 night on Dec 24, block a room
    that could earn 3× the revenue with a min-stay booking.

max_nights
  → int nullable. Maximum stay on this rate plan.
  → Advance Purchase: max 7 nights (beyond = negotiate separately).
  → null = no maximum.

arrival_days
  → JSON nullable: ["FRI", "SAT"]
  → Guest can ONLY check-in on these days during the period.
  → Event weekend: must arrive Friday or Saturday.
  → null = any arrival day allowed.
  → Why: prevents mid-week arrivals that create unsellable gaps
    around weekend peak demand.

closed_to_arrival
  → bool. No arrivals allowed during this LOS period.

closed_to_departure
  → bool. No departures allowed during this LOS period.
```

**Real example:**
```
Galle Literary Festival: Jan 15-20

LOS Rule:
  date_from: Jan 15, date_to: Jan 20
  min_nights: 3
  arrival_days: ["THU", "FRI"]
  closed_to_arrival: false

Guest tries: Jan 17 (Saturday) check-in, 2 nights
  → BLOCKED: "Minimum 3 nights for Jan 15-20 festival period"

Guest tries: Jan 16 (Friday) check-in, 3 nights
  → ✅ Allowed. 3 nights, Friday arrival. Correct.
```

---

### Entity 6: `OccupancyPricing`
**Additional charges for extra guests beyond base occupancy.**

```
rate_plan_id      FK → RatePlan   ↘
room_type_id      FK → RoomType   → Composite primary key

extra_adult_charge
  → decimal per night. Charge for each adult beyond base occupancy.
  → Standard Double base = 2 adults.
  → 3rd adult (rollaway bed): LKR 1,500/night.
  → Auto-posts to folio each night by night audit.

child_charge
  → decimal per night. Charge per child.
  → LKR 500/night per child (meal plan contribution).
  → 0 = children free (hotel's policy).

infant_charge
  → decimal per night. Usually 0.
  → Separate from child: different age bracket.
  → infant_free_until age in RatePlanPolicy handles exemption.
```

**Why on Rate Plan (not Room Type):**
```
Non-Refundable rate: extra adult LKR 1,500/night
Corporate rate:      extra adult LKR 0 (company pays flat)
Long Stay rate:      extra adult LKR 800/night (discounted)

Same room type. Different extra charge per rate plan.
Rate Plan controls the rule. Room Type defines the space.
```

---

### Entity 7: `CancellationPolicy`
**Reusable cancellation rules. Change once — all linked rate plans update.**

```
id
hotel_id

name
  → "Flexible-24" / "Non-Refundable" / "Tiered-7day"
  → Staff selects from list when creating rate plan.

type
  → FREE_UNTIL     Cancellation free until X hours before check-in
  → NON_REFUNDABLE No refund under any circumstances
  → PARTIAL        Percentage charged if cancelled within window
  → TIERED         Different % at different time windows

rules
  → JSON. The actual cancellation calculation logic.

  FREE_UNTIL:
    { "free_until_hours": 24, "after_charge_type": "FIRST_NIGHT" }

  NON_REFUNDABLE:
    { "charge_type": "FULL_STAY" }

  PARTIAL:
    { "within_hours": 48, "charge_percentage": 50 }

  TIERED:
    [
      { "days_before": 7,  "charge_pct": 0   },
      { "days_before": 3,  "charge_pct": 50  },
      { "days_before": 0,  "charge_pct": 100 }
    ]
    > 7 days: free
    3-7 days: 50% charge
    < 3 days: 100% charge

no_show_charge
  → NONE / FIRST_NIGHT / FULL_STAY / CUSTOM_PCT
  → Guest never arrives (no cancellation made).
  → Night audit auto-processes this at 11:59 PM.

date_change_allowed
  → bool. Can guest change dates on non-refundable bookings?
  → MEWS GAP WE FILL: "Non-refundable rates are fully rigid —
    cannot allow date changes. OTAs support this hybrid policy."
  → date_change_allowed = true enables:
      Guest cannot cancel (rate is non-refundable)
      But CAN change check-in/check-out dates (date is flexible)
      This is a common OTA feature. Now available in our PMS.

date_change_fee
  → decimal nullable. Fee charged for date change.
  → LKR 1,500 = one-time fee to change dates.
  → 0 = date change is free (but still non-refundable).

max_date_changes
  → int nullable. How many date changes allowed?
  → 1 = can change once only.
  → null = unlimited.
```

**Why reusable:**
```
Hotel has 6 rate plans.
4 of them use "Flexible-24" cancellation.

Without reusable policy:
  "Flexible-24" rules copied into 4 rate plans.
  Hotel changes: "free until 48hrs" (was 24hrs).
  Must update 4 rate plans. One gets missed. Inconsistency.
  Guest disputes: policy says 24hrs in one place, 48hrs in another.

With reusable policy:
  Change "Flexible-24" once.
  All 4 rate plans update instantly. Always consistent.
```

---

### Entity 8: `RatePlanPolicy`
**Operational rules for this specific rate plan.**

```
rate_plan_id                → FK → RatePlan (one-to-one)
cancellation_policy_id      → FK → CancellationPolicy (reusable)

─── CHECK-IN / CHECK-OUT ───────────────────────────────────

check_in_time
  → "15:00". Check-in time for this rate plan.
  → Corporate rate: "12:00" (early check-in included in rate).
  → Standard rate: "15:00".
  → Why per rate plan: premium rates can include early check-in
    as a benefit. Not just a property-wide rule.

check_out_time
  → "11:00". Standard check-out time.
  → Late checkout (12 PM / 13 PM) can be built into premium rates.

early_checkin_fee
  → nullable decimal. Charge for early check-in (when not included).
  → LKR 1,000 flat or LKR 500/hour before check_in_time.
  → null = early check-in not offered on this rate plan.
  → Posts to folio when approved by front desk.

late_checkout_fee
  → nullable decimal. Charge for late checkout.
  → LKR 500/hour or LKR 1,500 flat (up to 2 PM).
  → null = late checkout not offered or included free.

─── CHILD POLICY ────────────────────────────────────────────

child_allowed             → bool
child_age_limit           → int. Up to age 12 = child rate applies.
infant_free_until         → int. Under age 5 = free (LKR 0).

─── PET POLICY ──────────────────────────────────────────────

pet_allowed               → bool
pet_fee                   → nullable decimal. LKR 1,500/night.
                            Auto-posts to folio each night.
pet_types_allowed         → JSON: ["DOG", "CAT", "ANY"]
pet_size_limit            → SMALL / MEDIUM / LARGE / ANY
max_pets                  → int. Max 2 pets per booking.

─── NO-SHOW ─────────────────────────────────────────────────

no_show_charge
  → FIRST_NIGHT / FULL_STAY / NONE
  → Guest never arrived (no cancellation made).
  → Different from cancellation: this triggers when status is still
    CONFIRMED at night audit time with no check-in recorded.
  → Night audit auto-processes. No manual action needed.
```

---

### Entity 9: `RatePlanPayment`
**How and when money is collected.**

```
rate_plan_id                  → FK → RatePlan (one-to-one)

collection_type
  → PAY_NOW         Full amount at time of booking.
                    Required for: Non-Refundable rates.
  → DEPOSIT         Percentage now, rest later.
                    Most common for: Flexible rates.
  → PAY_AT_HOTEL    Nothing collected now. Pay at check-in or checkout.
                    Used for: Walk-in, phone bookings, trusted guests.
  → CC_GUARANTEE    Card saved, not charged now.
                    Charged only if: no-show occurs.
                    Used for: last-minute bookings, flexible corporate.

deposit_percentage
  → nullable decimal. 20 = 20% deposit at booking.

deposit_due
  → AT_BOOKING: deposit taken immediately.
  → DAYS_BEFORE_ARRIVAL: system auto-charges X days before.

deposit_due_days
  → nullable int. If DAYS_BEFORE_ARRIVAL: how many days?
  → 14 = deposit auto-charged 14 days before check-in.
  → System sends reminder email 3 days before auto-charge.

balance_collection
  → AT_CHECKIN / AT_CHECKOUT
  → When is the remaining balance due?
  → Most hotels: AT_CHECKIN (less risk).
  → Corporate / trusted guests: AT_CHECKOUT (city ledger).

auto_charge_no_show
  → bool. If no-show: auto-charge the saved card?
  → Must be true for CC_GUARANTEE bookings to make sense.
  → Night audit triggers charge + sends receipt to guest.

accepted_methods
  → JSON: ["CARD", "UPI", "CASH", "BANK_TRANSFER", "CORPORATE_ACCOUNT"]
  → Which payment methods are valid for this rate plan.
  → OTA rate plans: CARD only (OTA collects, remits to hotel).
  → Corporate rate plans: CORPORATE_ACCOUNT (city ledger) allowed.
  → Non-refundable: CARD only (need card to charge if cancellation).

refund_timeline_days
  → int. Days to process refund on valid cancellation.
  → 7 = refund within 7 business days.
  → Shown to guest on cancellation confirmation screen.
  → Sets legal expectation.
```

---

### Entity 10: `RatePlanChannel`
**OTA distribution — where this rate plan is sold.**

```
rate_plan_id      FK → RatePlan  ↘
channel           DIRECT / BOOKING_COM / AGODA /
                  EXPEDIA / AIRBNB / MAKEMYTRIP
                  Together → Composite primary key

channel_rate_override
  → nullable decimal. Different rate for this channel.
  → null = same as base rate (rate parity on).
  → LKR 13,500 = this channel sells at LKR 13,500 specifically.

commission_percentage
  → nullable decimal.
  → Booking.com: 15%. Agoda: 12%. Expedia: 18%. Direct: null.

commission_borne_by
  → HOTEL: hotel pays commission. Guest sees base rate.
  → GUEST: commission added to guest rate. Hotel nets base.
  → Most OTA contracts: HOTEL bears commission.

channel_rate_plan_code
  → text nullable. OTA's internal code for our rate plan.
  → Booking.com assigns their own code: "FLEX-STD-BB-2025"
  → Expedia: "RatePlan-789456"
  → WHY CRITICAL: OTA API requires their code when pushing
    rate/availability updates. Without this: sync fails silently.
  → Research finding: "Room types and rate plans don't map
    1:1 across systems — causes booking miscoding and price errors."
  → We store and sync this precisely.

inventory_allocation
  → int nullable. Max rooms to sell through this channel.
  → null = no limit (sell full available inventory).
  → 10 = max 10 rooms through Booking.com per date.
        Rest available for Direct and other OTAs.
  → CLOUDBEDS BEST FEATURE: we adopt this.
  → Why: prevents one OTA from absorbing all inventory,
    leaving nothing for direct bookings (zero commission).

auto_close_at_allocation
  → bool. When allocation reached, auto-stop-sell this channel?
  → true: Booking.com hits 10 rooms sold → auto-closed.
         Other channels continue selling.
  → false: alert only — staff decides whether to close.

is_active
  → bool. Is this channel currently selling this rate plan?
  → false = stop-sell on this channel only.
           Rate plan + other channels continue.
```

---

### Entity 11: `RateParity`
**Rate parity monitoring. OTA contract compliance protection.**

```
rate_plan_id
  → FK → RatePlan (one per rate plan)

enforce_parity
  → bool. Block if direct rate lower than OTA rate?
  → true: system prevents saving rate plan if parity violated.
  → false: warn only. Staff can override.

alert_on_violation
  → bool. Notify GM if violation detected?

parity_check_channels
  → JSON: ["BOOKING_COM", "AGODA"]
  → Which channels must match direct rate.

WHY THIS MATTERS FOR SRI LANKA:
  Booking.com and Agoda contracts include rate parity clauses.
  Violation = de-ranking on OTA listing (less visibility).
  Repeat violations = removal from OTA listing.
  System catches accidental violations before they happen.
```

---

### Entity 12: `DerivedRatePlan` (NEW — from OPERA research)
**Auto-cascading rate relationships. Change parent → all children update.**

**This solves the #1 manual error in hotel rate management.**

```
id
hotel_id

parent_rate_plan_id
  → FK → RatePlan. The base rate (usually BAR — Best Available Rate).

child_rate_plan_id
  → FK → RatePlan. The derived rate plan.

adjustment_type
  → PERCENTAGE / FLAT

adjustment_value
  → decimal. -15 = 15% below parent. +10 = 10% above parent.

adjustment_direction
  → DISCOUNT (child < parent) / PREMIUM (child > parent)

applies_to
  → ALL_ROOM_TYPES / SPECIFIC_ROOM_TYPES

applies_to_room_type_ids
  → JSON nullable. If SPECIFIC: which room types get this derivation.

rounding_rule
  → NONE / NEAREST_100 / ROUND_UP / ROUND_DOWN
  → BAR Deluxe: LKR 13,500 × 0.85 = LKR 11,475
  → NEAREST_100 → LKR 11,500 (cleaner price to show guests)

is_active
  → bool. Temporarily break the link without deleting.
  → "Corporate contract expired — pause derivation until renewed."
```

**How it works:**
```
SETUP (once):
  Parent: BAR (Best Available Rate)
    Standard Double: LKR 12,000

  Children derived from BAR:
    Corporate: BAR × 0.85 (-15%)     → LKR 10,200
    Staff Rate: BAR × 0.50 (-50%)    → LKR  6,000
    Long Stay:  BAR × 0.80 (-20%)    → LKR  9,600
    Last Minute: BAR × 1.10 (+10%)   → LKR 13,200

WHEN BAR CHANGES (e.g. peak season):
  Staff updates BAR Standard Double: LKR 12,000 → LKR 18,000

  System auto-cascades:
    Corporate:   LKR 18,000 × 0.85 = LKR 15,300 ✅ (auto)
    Staff Rate:  LKR 18,000 × 0.50 = LKR  9,000 ✅ (auto)
    Long Stay:   LKR 18,000 × 0.80 = LKR 14,400 ✅ (auto)
    Last Minute: LKR 18,000 × 1.10 = LKR 19,800 ✅ (auto)

  Zero manual work. Zero human error. Instant.

WITHOUT THIS (what hotels do today):
  Staff updates BAR. Forgets to update Corporate.
  Corporate guest checks in at old rate LKR 10,200.
  Hotel loses LKR 5,100 per night. Undetected for weeks.
```

---

### Entity 13: `PackageInclusion` (NEW — from OPERA research)
**Bundle services into a rate plan. Revenue split by department.**

```
id
rate_plan_id        FK → RatePlan

service_type
  → BREAKFAST / LUNCH / DINNER / ALL_MEALS /
    AIRPORT_TRANSFER / SPA_CREDIT / PARKING /
    MINIBAR / FRUIT_BASKET / BOTTLE_OF_WINE /
    WELCOME_DRINK / LAUNDRY_CREDIT / ACTIVITY / CUSTOM

name
  → "Complimentary Airport Transfer"
  → "LKR 2,000 Spa Credit"
  → "Daily Breakfast for 2"
  → Guest-facing. Shown in rate plan details.

quantity
  → int. How many per booking?

quantity_basis
  → PER_STAY    → 1 welcome basket (entire stay)
  → PER_NIGHT   → 1 bottle of water (each night)
  → PER_PERSON  → 1 breakfast per person per night
  → PER_ADULT   → specific to adults only

price
  → decimal. Cost of this inclusion to the hotel.
  → Used for revenue attribution — not charged to guest.
  → "Executive Package LKR 25,000" is one price.
    Internally split: LKR 18,000 rooms + LKR 3,200 F&B +
                      LKR 2,400 transport + LKR 1,400 spa.

revenue_center
  → ROOMS / FOOD_AND_BEVERAGE / SPA / TRANSPORT / OTHER
  → Which department does this revenue belong to?
  → Critical for: departmental P&L reporting.
  → Breakfast revenue → F&B center, NOT Rooms.
  → OPERA's revenue splitting — no other small system has this.

folio_post_trigger
  → AT_CHECKIN   → posts once at check-in (fruit basket, transfer)
  → DAILY        → posts every day (breakfast, parking, minibar)
  → AT_CHECKOUT  → posts once at checkout (total minibar credit used)
  → MANUAL       → staff posts when service is consumed (spa credit)
  → NEVER        → included in rate, no separate folio line

is_included_in_rate
  → bool. Is service price already inside the rate plan total?
  → true:  "Executive Package LKR 25,000 includes breakfast"
           No extra charge. Folio shows LKR 0 for breakfast.
  → false: "Standard + Breakfast Add-On"
           Breakfast posts as separate folio charge daily.

creates_task
  → bool. Does this inclusion require a staff task?
  → Fruit basket: true → HK task: "Place fruit basket in room 205"
  → Airport transfer: true → FD task: "Arrange pickup for Room 205,
                                       Dec 20 at 3 PM"
  → Breakfast: false → restaurant handles via meal plan system
  → Spa credit: false → guest redeems at spa desk

task_department
  → nullable: HOUSEKEEPING / FRONT_DESK / RESTAURANT / SPA / TRANSPORT
```

**Example — Executive Package:**
```
Rate Plan: "Executive Package" — LKR 25,000/night

PackageInclusions:
  ┌──────────────────┬────────────┬──────────┬──────────┬───────────┐
  │ Service          │ Qty/Basis  │ Price    │ Revenue  │ Trigger   │
  ├──────────────────┼────────────┼──────────┼──────────┼───────────┤
  │ Breakfast for 2  │ Per night  │ LKR 1,600│ F&B      │ DAILY     │
  │ Airport Transfer │ Per stay   │ LKR 2,400│ Transport│ AT_CHECKIN│
  │ Spa Credit 2,000 │ Per stay   │ LKR 2,000│ Spa      │ MANUAL    │
  │ Fruit Basket     │ Per stay   │ LKR  500 │ F&B      │ AT_CHECKIN│
  │ Parking          │ Per night  │ LKR  500 │ Other    │ DAILY     │
  └──────────────────┴────────────┴──────────┴──────────┴───────────┘

Revenue attribution (auto):
  Rooms:     LKR 18,000  (72%)  → Rooms P&L
  F&B:       LKR  2,100  (8.4%) → Restaurant P&L
  Transport: LKR  2,400  (9.6%) → Transport P&L
  Spa:       LKR  2,000  (8%)   → Spa P&L
  Other:     LKR    500  (2%)   → Other

No manual revenue allocation. Setup once → auto-splits forever.
```

---

### Entity 14: `RatePlanAuditLog` (NEW — industry gap we fill)
**Complete history of every rate plan change. Who, what, when, why.**

**Research finding:** "Most systems don't track who changed what rate plan and when. No easy way to revert."

```
id
hotel_id
rate_plan_id        FK → RatePlan

changed_by_staff_id FK → Staff
changed_at          timestamp

change_type
  → RATE_CHANGED / POLICY_CHANGED / CHANNEL_ADDED /
    CHANNEL_REMOVED / STATUS_CHANGED / DATE_OVERRIDE_ADDED /
    LOS_RULE_CHANGED / PACKAGE_ADDED / COMMISSION_CHANGED

entity_changed
  → "RatePlanRoom" / "CancellationPolicy" / "RatePlanChannel" ...

field_changed
  → "base_rate" / "min_nights" / "commission_percentage" ...

old_value
  → text. What it was before. "12000"

new_value
  → text. What it became. "14000"

change_reason
  → text nullable. Why was this changed?
  → "Peak season rate increase — approved by GM"
  → "Corporate contract renewed — MAS Holdings"
  → "Competitor dropped rate — adjusting"

booking_impact_count
  → int. Future bookings using this rate plan.
  → AUTO-CALCULATED before change is confirmed.
  → Warning shown: "This change affects 12 future bookings.
                    Proceed?"
  → Staff must acknowledge impact before saving.
  → No system today warns about this. Our innovation.
```

**Real use cases:**
```
SCENARIO 1: Owner asks accounting question
  "Who raised the corporate rate in October?"
  AuditLog → changed_by: Meena, Oct 14, 10:23 AM
              LKR 14,000 → LKR 16,000
              reason: "Q4 rate revision"

SCENARIO 2: Guest dispute
  "Guest says they were quoted LKR 12,000 but billed LKR 14,000"
  AuditLog → rate changed Oct 15, booking made Oct 12.
              Guest booked at old rate = LKR 12,000 was correct.
              Billing error confirmed. Refund LKR 2,000.

SCENARIO 3: OTA dispute
  "Booking.com says our rate was wrong on Nov 20"
  AuditLog → commission changed Nov 18 from 12% → 15%.
              Channel sync fired Nov 18. Booking.com received update.
              Evidence: hotel made the change. Not OTA error.
```

---

## 7. Complete Relationship Map

```
RatePlan (1)
  ├── RatePlanRoom (many)              price per room type + derived calc
  │     └── DerivedRatePlan link ──→  parent rate plan cascade
  │
  ├── RatePlanDateOverride (many)      seasonal pricing + CTA/CTD
  ├── RatePlanDayRule (many)           weekend/weekday multipliers
  ├── RatePlanLOSRule (many)           min/max stay + CTA/CTD
  ├── OccupancyPricing (many)          extra adult/child charges
  ├── PackageInclusion (many)          bundled services + revenue split
  │
  ├── RatePlanPolicy (1)               operational rules
  │     └── CancellationPolicy ─────→ reusable + hybrid support
  │           (shared across many rate plans)
  │
  ├── RatePlanPayment (1)              collection method
  ├── RatePlanChannel (many)           OTA distribution + allocation
  ├── RateParity (1)                   parity monitoring
  └── RatePlanAuditLog (many)          full change history

DerivedRatePlan
  parent_rate_plan ──→ child_rate_plan (cascade on parent change)
  Can have multiple children from one parent.
  Can chain: BAR → Corporate → VIP Corporate (derived of derived).
```

---

## 8. Price Calculation — Complete Step-by-Step

```
BOOKING: Deluxe King, Executive Package, Dec 20 (Saturday), 3 nights,
         2 adults + 1 child (age 8), booked via Direct channel

STEP 1: Rate Plan found
  "Executive Package" — PER_ROOM, FULL_BOARD

STEP 2: Base rate (RatePlanRoom)
  Deluxe King, Executive Package → LKR 25,000/night

STEP 3: Date Override check
  Dec 20-22 override exists: LKR 30,000 (Christmas peak)
  Override applies → base rate replaced.

STEP 4: CTA check (RatePlanDateOverride)
  Dec 20 closed_to_arrival = false → check-in allowed ✅

STEP 5: Day Rule check
  Dec 20 is Saturday → DayRule ×1.10
  BUT DateOverride exists → DateOverride WINS.
  Day rule ignored for this date.

STEP 6: LOS Rule check
  Peak rule: min_nights = 3, arrival_days = ["FRI", "SAT"]
  Dec 20 = Saturday ✅, 3 nights ✅ → allowed

STEP 7: Occupancy (OccupancyPricing)
  2 adults = base (PER_ROOM — no extra adult charge)
  1 child (age 8, within child_age_limit 12) → LKR 500/night
  3 nights → child charge = LKR 1,500

STEP 8: Package inclusions (PackageInclusion)
  Breakfast/Lunch/Dinner: DAILY, included in rate (no extra post)
  Airport transfer: AT_CHECKIN, LKR 2,400 (revenue: Transport)
  Spa credit LKR 2,000: MANUAL (guest redeems at spa)
  Fruit basket: AT_CHECKIN, HK task created
  Parking: DAILY LKR 500 × 3 nights = LKR 1,500

STEP 9: Channel check (RatePlanChannel)
  Direct booking → commission = 0%, no inventory limit concern

STEP 10: Tax calculation (Tax Configuration — Area 14)

  ROOM CHARGES (3 nights × LKR 30,000):     LKR 90,000
  Child charge (3 nights × LKR 500):         LKR  1,500
  Parking (3 nights × LKR 500):              LKR  1,500
  Airport Transfer (1):                       LKR  2,400
  ───────────────────────────────────────────────────────
  BASE TOTAL:                                LKR 95,400

  Service Charge 10%:                        LKR  9,540
  ───────────────────────────────────────────────────────
  VAT BASE:                                  LKR 104,940
  VAT 18%:                                   LKR  18,889
  TDL 1% (on room only LKR 91,500):          LKR     915
  ───────────────────────────────────────────────────────
  GRAND TOTAL:                               LKR 124,744

  Deposit 20% (collected now):               LKR  24,949
  Balance at check-in:                       LKR  99,795

REVENUE ATTRIBUTION (internal):
  Rooms:     LKR 90,000 + 1,500 (child) = LKR 91,500 → Rooms P&L
  F&B:       3 nights full board + fruit basket        → Restaurant P&L
  Transport: LKR 2,400 (airport transfer)              → Transport P&L
  Parking:   LKR 1,500                                 → Other P&L
```

---

## 9. The Rate Plan Builder — What Staff Sees

```
STEP 1: Choose Template
  ○ Flexible         → 80% pre-filled (free cancel, standard deposit)
  ○ Non-Refundable   → Full charge, lower price
  ○ Advance Purchase → Early bird discount, pay now
  ○ Long Stay        → 7+ nights discount
  ○ Bed & Breakfast  → Meal plan included
  ○ Corporate        → Private, invoice billing
  ○ Custom           → Start from scratch

STEP 2: Basic Details
  Name:       [                    ]
  Code:       [    ] (auto-suggest)
  Visibility: [ Public ▼]
  Status:     [ DRAFT  ] ← starts as draft

STEP 3: Room Prices
  ┌────────────────────────────┬──────────────┐
  │ Room Type                  │ Base Rate    │
  ├────────────────────────────┼──────────────┤
  │ Standard Double            │ LKR [12,000] │
  │ Deluxe King                │ LKR [18,000] │
  │ Suite                      │ LKR [35,000] │
  └────────────────────────────┴──────────────┘
  Derive from existing rate plan? [ BAR ▼] [-15%]

STEP 4: Cancellation Policy
  [ Flexible-24 ▼ ] (select from saved policies)
  [ + Create new policy ]

STEP 5: Payment
  Collection: [ 20% Deposit ▼]
  Deposit due: [ At Booking ▼]
  Balance: [ At Check-in ▼]

STEP 6: Channels
  [✅] Direct    [✅] Booking.com  [✅] Agoda
  [  ] Expedia   [  ] Airbnb

STEP 7: Review → [ Activate Rate Plan ]
```

---

## 10. Basic → Advanced — Same Entities, Different Depth

```
                GUESTHOUSE    BUSINESS     RESORT       LUXURY
                (2 types)     (4 types)    (6 types)    (8 types)
                ──────────    ─────────    ──────────   ─────────
Rate Plans      2             5            8            12+
Date Overrides  2-3           8-10         15+          25+
Day Rules       0-1           1            2            3
LOS Rules       0             2            5            8
OTA Channels    1 (direct)    3            5            7
Packages        0             1            3            6+
Derived Plans   0             1-2          4            8
Audit log       auto          auto         auto         auto

Entities used:  All 14        All 14       All 14       All 14
Fields used:    25%           55%          75%          100%
```

Same 14 entities. Every hotel size. No schema changes between phases.

---

## 11. Competitive Advantage Summary

```
FEATURE                          US   OPERA   MEWS   CLOUDBEDS   LITTLE H.
──────────────────────────────────────────────────────────────────────────
Derived rates (auto-cascade)     ✅    ✅       ❌      ❌          ❌
Policy reusability               ✅    ✅       ✅      ❌          ❌
Hybrid cancellation              ✅    ❌       ❌      ❌          ❌
  (non-refundable + date change)
CTA / CTD restrictions           ✅    ✅       ✅      ✅          ❌
Rate plan DRAFT status           ✅    ❌       ❌      ❌          ❌
Audit log + booking impact warn  ✅    ❌       ❌      ❌          ❌
Package revenue splitting        ✅    ✅       ❌      ❌          ❌
Channel inventory allocation     ✅    ❌       ❌      ✅          ❌
No OTA rate plan limit           ✅    ✅       ✅      ❌(max 4)   ❌
OTA rate plan code mapping       ✅    ✅       ✅      Partial     ❌
Simple by default                ✅    ❌       ✅      ✅          ✅
Powerful when needed             ✅    ✅       Partial  ✅         ❌
Sri Lanka tax (VAT+SC+TDL)       ✅    ❌       ❌      ❌          ❌
──────────────────────────────────────────────────────────────────────────
Score (out of 13)               13/13  6/13   6/13    5/13        2/13
```

---

## 12. V1 Scope — What to Build First

**V1 — Must have (hotel cannot operate without):**
```
✅ RatePlan             name, code, template, meal_plan, pricing_model,
                        visibility, status (DRAFT/ACTIVE/ARCHIVED),
                        is_active, display_order

✅ RatePlanRoom         base_rate per room type
                        is_derived + adjustment fields (even if UI is Phase 2)

✅ RatePlanDateOverride seasonal + holiday overrides + CTA/CTD fields

✅ CancellationPolicy   3 types: FREE_UNTIL / NON_REFUNDABLE / PARTIAL
                        date_change_allowed + date_change_fee

✅ RatePlanPolicy       check-in/out times, no-show, child policy, pet policy

✅ RatePlanPayment      all 4 collection types, deposit config

✅ RatePlanChannel      Direct + 2 OTA channels + channel_rate_plan_code

✅ RatePlanAuditLog     all changes logged automatically (no UI needed V1)

✅ PackageInclusion     basic: Room Only + Breakfast inclusion
```

**Phase 2:**
```
→ RatePlanDayRule       weekend/weekday auto-multiplier
→ RatePlanLOSRule       min/max stay + arrival days + CTA/CTD
→ OccupancyPricing      extra adult + child charges
→ DerivedRatePlan       corporate auto-cascade from BAR
→ PackageInclusion      full bundle + revenue split by department
→ RateParity            monitoring + GM alerts
→ AuditLog UI           staff can view change history
→ Booking impact warning before rate changes
→ Tiered cancellation policy (TIERED type)
→ Channel inventory allocation
```

**Phase 3:**
```
→ AI dynamic pricing (yield management)
→ Rate plan simulation (impact preview before change)
→ Day-use / time slice rates (Apaleo concept)
→ Rate plan A/B testing
→ Competitor rate monitoring + alerts
→ Natural language rate plan builder
```

---

## 13. How Rate Plan Connects to Every Other Module

```
ROOM SETUP (Area 2)
  → RatePlanRoom.room_type_id FK → RoomType
  → Rate plan cannot exist without room type existing first.
  → Availability logic uses room type to count inventory.

BOOKING ENGINE
  → Booking stores: rate_plan_id (which plan was used)
  → Booking stores: rate_snapshot (rate at time of booking — immutable)
  → Rate plan can change later. Booking always uses original rate.

TAX CONFIGURATION (Area 14)
  → meal_charge_per_night split: food portion taxed differently than room
  → PackageInclusion.revenue_center used for correct tax category
  → TDL only applies to room revenue, not F&B or transport

FOLIO & BILLING (Area 9)
  → Night audit uses RatePlanRoom.base_rate to post room charge
  → PackageInclusion.folio_post_trigger controls when services post
  → OccupancyPricing.extra_adult_charge posts nightly

CHANNEL MANAGEMENT (Area 8)
  → RatePlanChannel defines which OTAs sell which rate plans
  → channel_rate_plan_code used for OTA API sync
  → inventory_allocation controls channel inventory distribution

NOTIFICATION SETUP (Area 10)
  → Booking confirmation email includes: rate plan name, cancellation policy
  → Deposit reminder uses: deposit_due_days from RatePlanPayment
  → Cancellation notice uses: CancellationPolicy rules to calculate refund

GUEST PORTAL (Area 11)
  → visibility = PUBLIC: shown on booking widget
  → visibility = PRIVATE: requires promo code
  → visibility = CORPORATE: shown only to linked corporate accounts
  → Rate plan description + cancellation policy shown to guest before booking

GROUP & CORPORATE (Area 15)
  → CorporateRatePlan links corporate account to rate plan
  → visibility = CORPORATE on rate plan
  → DerivedRatePlan: corporate rate auto-derived from BAR

HOUSEKEEPING (Area 6)
  → PackageInclusion.creates_task = true → housekeeping task auto-created
  → "Fruit basket in room 205" task from Executive Package inclusion

STAFF & ROLES (Area 7)
  → rate_view / rate_modify permissions control who sees/edits rate plans
  → RatePlanAuditLog.changed_by_staff_id tracks every change
  → Manager PIN required to change rates above X% (configurable)
```

---

## 14. Entity Quick Reference

| # | Entity | Records (5 rate plans, 4 room types) | Purpose |
|---|--------|--------------------------------------|---------|
| 1 | RatePlan | 5 | Master rate plan record |
| 2 | RatePlanRoom | 20 (5×4) | Price per room type |
| 3 | RatePlanDateOverride | 15-40 | Seasonal/holiday pricing |
| 4 | RatePlanDayRule | 5-10 | Weekend/day multipliers |
| 5 | RatePlanLOSRule | 5-15 | Min/max stay restrictions |
| 6 | OccupancyPricing | 20 (5×4) | Extra guest charges |
| 7 | CancellationPolicy | 3-5 | Reusable cancel rules |
| 8 | RatePlanPolicy | 5 | Operational rules per plan |
| 9 | RatePlanPayment | 5 | Payment collection config |
| 10 | RatePlanChannel | 15-25 | OTA distribution |
| 11 | RateParity | 5 | Parity monitoring |
| 12 | DerivedRatePlan | 2-4 | Auto-cascade relationships |
| 13 | PackageInclusion | 0-20 | Bundle service inclusions |
| 14 | RatePlanAuditLog | grows over time | Full change history |
