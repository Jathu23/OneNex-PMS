# C7 — Rate Plan Setup: Owner & Staff Journey
> Stays Module | Core Feature 7 of 9
> Purpose: Every scenario where a rate plan is created, modified, or managed.
> What they see → What they do → What the system does → What API is needed.

---

## Who Is Doing This

```
PERSON 1: Business Owner (first-time setup)
  → Just enabled Stays. Rooms are set up. Now setting up pricing.
  → May not know terms like "derived rate" or "CTA".
  → Wants to go live fast. System guides them.

PERSON 2: Revenue Manager / Senior Staff
  → Ongoing rate management. Seasonal pricing. New OTA.
  → Understands hotel pricing concepts.
  → Needs full control: derived rates, overrides, packages.

PERSON 3: Front Desk Manager
  → Making a small rate change during operations.
  → Should NOT have the power to change active rates without permission.
  → System blocks and routes through proper approval flow.
```

**Prerequisite:**
```
Room Setup (C1) must be completed first.
Rate plans cannot exist without room types.
System blocks access to Rate Plan setup if no room types exist.
Error shown: "You must set up at least one room type before creating rate plans."
```

---

## The Full Flow Map

```
ENTRY POINT: Stays → Rate Plans tab
                    ↓
           ┌────────────────────────────────────────┐
           │  First time?                           │
           │  → "You have no rate plans yet"        │
           │  → [ Create your first rate plan ]     │
           │                                        │
           │  Returning?                            │
           │  → List of existing plans + status     │
           │  → [ + New Rate Plan ] button          │
           └────────────────────────────────────────┘
                    ↓
      ┌─────────────┴──────────────┐
      │                            │
   FLOW A                       FLOWS B-H
   Create new rate plan          Manage existing

FLOW A: Create Rate Plan (template → prices → policy → payment → channels → activate)
FLOW B: Create Cancellation Policy (reusable — can trigger from Flow A Step 4)
FLOW C: Add Date Override (seasonal / holiday pricing on existing plan)
FLOW D: Setup Derived Rate Plan (BAR → Corporate auto-cascade)
FLOW E: Add Package Inclusions (bundled services on existing plan)
FLOW F: Edit Rate on Active Plan (impact warning → DRAFT → re-activate)
FLOW G: Archive a Rate Plan (lifecycle end)
FLOW H: Channel Setup & Commission (add new OTA to existing plan)
```

---

## FLOW A — Create a New Rate Plan

This is the primary flow. Owner or Revenue Manager builds a complete rate plan from scratch
using a template as a starting point.

### A-Step 1: Entry — Rate Plans Dashboard

**What owner sees (first time):**
```
┌─────────────────────────────────────────────────┐
│  Rate Plans                                     │
│                                                 │
│  ⚠️  You have no rate plans yet.               │
│  Guests cannot be charged until at least        │
│  one rate plan is active.                       │
│                                                 │
│  [ + Create Your First Rate Plan ]              │
└─────────────────────────────────────────────────┘
```

**What owner sees (returning — has existing plans):**
```
┌──────────────────────────────────────────────────────────────────┐
│  Rate Plans                              [ + New Rate Plan ]     │
│                                                                  │
│  NAME              TEMPLATE      STATUS    ROOMS    CHANNELS     │
│  BAR Standard      Flexible      ACTIVE    4/4      Direct, B.com│
│  Corporate Rate    Corporate     ACTIVE    4/4      Direct       │
│  Peak Season       Custom        DRAFT     3/4      —            │
│  Summer Special    Advance Purch ARCHIVED  —        —            │
│                                                                  │
│  Filter: [ All ▼ ] [ ACTIVE ▼ ] [ Search... ]                   │
└──────────────────────────────────────────────────────────────────┘
```

**What system does:**
```
GET /rate-plans
  → Returns all rate plans for this hotel
  → Includes: status, template_type, room_count, channel_count
  → Sorted by: ACTIVE first, then DRAFT, then ARCHIVED
  → ARCHIVED plans collapsed by default (show "View archived" toggle)
```

**APIs needed:**
```
GET /rate-plans
  → Query params: status, search, page
  → Returns: list with summary (no full detail)
```

---

### A-Step 2: Choose Template

**What owner sees:**
```
"What kind of rate plan are you creating?"

┌─────────────────────────────────────────────────────────┐
│  ○ Flexible                                             │
│    Free cancel up to 24h before. 20% deposit.           │
│    Best for: most bookings                              │
│                                                         │
│  ○ Non-Refundable                                       │
│    No refund if cancelled. Lower price.                 │
│    Best for: guests who commit early for a discount     │
│                                                         │
│  ○ Advance Purchase                                     │
│    Pay full amount now. Discounted rate.                │
│    Best for: early bird promotions                      │
│                                                         │
│  ○ Long Stay                                            │
│    7+ nights get a discount. Free cancel 48h.           │
│    Best for: extended stay guests                       │
│                                                         │
│  ○ Bed & Breakfast                                      │
│    Room + breakfast included. Free cancel 24h.          │
│    Best for: B&B style properties                       │
│                                                         │
│  ○ Corporate                                            │
│    Private rate. Invoice on checkout.                   │
│    Best for: company accounts, business travelers       │
│                                                         │
│  ○ Custom                                               │
│    Start from scratch. I'll configure everything.       │
│    Best for: experienced revenue managers               │
└─────────────────────────────────────────────────────────┘

[ Back ]                                     [ Continue → ]
```

**What owner does:** Clicks one. Hits Continue.

**What system does:**
```
Based on template, pre-fills:

FLEXIBLE:
  cancellation_type:   FREE_UNTIL (24h)
  collection_type:     DEPOSIT_AT_BOOKING (20%)
  balance_due:         AT_CHECKIN
  meal_plan:           ROOM_ONLY
  visibility:          PUBLIC
  status:              DRAFT        ← always starts DRAFT

NON_REFUNDABLE:
  cancellation_type:   NON_REFUNDABLE
  collection_type:     FULL_AT_BOOKING
  meal_plan:           ROOM_ONLY
  visibility:          PUBLIC
  status:              DRAFT

ADVANCE_PURCHASE:
  cancellation_type:   FREE_UNTIL (72h)
  collection_type:     FULL_AT_BOOKING
  advance_purchase_min_days: 7     ← must book 7+ days in advance
  meal_plan:           ROOM_ONLY
  visibility:          PUBLIC
  status:              DRAFT

LONG_STAY:
  cancellation_type:   FREE_UNTIL (48h)
  collection_type:     DEPOSIT_AT_BOOKING (30%)
  meal_plan:           ROOM_ONLY
  visibility:          PUBLIC
  status:              DRAFT
  → Also pre-sets: min_nights suggestion = 7 (shown in Step 4)

BED_BREAKFAST:
  cancellation_type:   FREE_UNTIL (24h)
  collection_type:     DEPOSIT_AT_BOOKING (20%)
  meal_plan:           BREAKFAST
  visibility:          PUBLIC
  status:              DRAFT

CORPORATE:
  cancellation_type:   FREE_UNTIL (48h)
  collection_type:     INVOICE_ON_CHECKOUT
  meal_plan:           ROOM_ONLY
  visibility:          CORPORATE   ← not shown publicly
  status:              DRAFT

CUSTOM:
  All fields empty.
  Owner fills everything from scratch.
```

**APIs needed:**
```
GET /rate-plans/templates
  → Returns 7 templates with their default pre-fill values
  → Used to preview what each template gives before selection

POST /rate-plans/draft
  body: { template_type: "FLEXIBLE" }
  → Creates a DRAFT RatePlan record with pre-filled defaults
  → Returns: draft rate_plan_id (all subsequent steps use this ID)
  → Owner can close browser. Draft is auto-saved. Resumes next login.
```

---

### A-Step 3: Basic Details

**What owner sees:**
```
┌──────────────────────────────────────────────────────┐
│  Rate Plan Details                                   │
│                                                      │
│  Name *                                              │
│  [                                    ]              │
│  e.g. "Standard Flexible" / "Corporate Rate"        │
│                                                      │
│  Internal Code *                                     │
│  [ BAR ] ← auto-suggested from name, editable       │
│  Used in reports and OTA sync. Not shown to guests.  │
│                                                      │
│  Pricing Model                                       │
│  [ Per Room ▼ ]                                      │
│  ○ Per Room    One price regardless of guests        │
│  ○ Per Person  Price × number of guests              │
│  ○ Per Adult   Price × adults only                   │
│                                                      │
│  Meal Plan                                           │
│  [ Room Only ▼ ]                                     │
│                                                      │
│  Currency                                            │
│  [ LKR — Sri Lankan Rupee ▼ ]                       │
│  (Inherits from hotel. Override per plan if needed.) │
│                                                      │
│  Who can see this rate?                              │
│  ○ Public      Visible to all guests                 │
│  ● Private     Guests need a promo code              │
│  ○ Corporate   Only linked company accounts          │
│                                                      │
│  Status: DRAFT  ← auto-set. Cannot change here.     │
│  Rate plan is in DRAFT until you activate it.        │
│  Drafts are not visible to guests. Not synced to     │
│  OTAs. Safe to edit without risk.                    │
└──────────────────────────────────────────────────────┘

[ ← Back ]                             [ Save & Continue → ]
```

**Validation rules:**
```
name:        required. min 3 chars. max 100 chars.
code:        required. UPPERCASE + underscore only. unique per hotel.
             Auto-generated: "BAR Standard" → "BAR_STANDARD"
             Owner can edit. System checks uniqueness on blur.
pricing_model: required.
              ⚠️  Must be selected BEFORE entering room prices in Step 4.
              pricing_model determines what base_rate means (per room / per person / per adult).
              Once plan is ACTIVE, pricing_model is locked and cannot be changed.
meal_plan:   required.
visibility:  required.
```

**What system does:**
```
PATCH /rate-plans/{draft_id}
  body: { name, code, pricing_model, meal_plan, visibility, currency_code }
  → Updates draft record
  → Returns validation errors immediately (not on submit)
  → Code uniqueness checked: if taken → "BAR_STANDARD_2" suggested
```

**APIs needed:**
```
PATCH /rate-plans/{id}
  → Partial update. Auto-saves on field blur.

GET /rate-plans/check-code?code=BAR_STANDARD&hotel_id=123
  → Returns: { available: true } or { available: false, suggestion: "BAR_STANDARD_2" }
```

---

### A-Step 4: Set Room Prices

**What owner sees:**
```
┌─────────────────────────────────────────────────────────────────┐
│  Room Prices                                                    │
│  Set the rate per adult / night for each room type.             │
│  (label changes based on pricing_model selected in Step 3)      │
│    PER_ROOM   → "Base Rate / Night"                             │
│    PER_PERSON → "Rate per Person / Night"                       │
│    PER_ADULT  → "Rate per Adult / Night"                        │
│                                                                 │
│  ┌──────────────────────────────┬──────────┬─────────────────┐  │
│  │ Room Type                    │ Include? │ Base Rate/Night │  │
│  ├──────────────────────────────┼──────────┼─────────────────┤  │
│  │ Standard Double              │   [✅]   │  LKR [12,000]   │  │
│  │ Deluxe King                  │   [✅]   │  LKR [18,000]   │  │
│  │ Superior Twin                │   [✅]   │  LKR [15,000]   │  │
│  │ Suite                        │   [✅]   │  LKR [35,000]   │  │
│  └──────────────────────────────┴──────────┴─────────────────┘  │
│                                                                 │
│  ─── OR ───────────────────────────────────────────────────    │
│                                                                 │
│  Derive rates from an existing plan?                            │
│  [ None ▼ ]  [ — ] [ % ]                                       │
│  e.g. Derive from "BAR Standard" at -15% → auto-calculated     │
│                                                                 │
│  ⚠️  If you derive rates, you cannot edit room prices manually. │
│  The parent plan controls all prices via the formula.           │
└─────────────────────────────────────────────────────────────────┘

[ ← Back ]                             [ Save & Continue → ]
```

**Two paths from this step:**

**PATH 1 — Manual prices (default):**
```
Owner enters each room type's base rate manually.
Each room type can be included (✅) or excluded (unchecked).
Excluded room types = this rate plan does not apply to that room type.

The value entered is the UNIT RATE — its meaning depends on pricing_model:
  PER_ROOM   → entered value = total room price (guest count has no effect)
  PER_PERSON → entered value = per-person rate (system multiplies by guest count at booking)
  PER_ADULT  → entered value = per-adult rate (system multiplies by adult count at booking)

Final price is calculated at booking time, not stored.

Validation:
  → If included: base_rate must be > 0.
  → At least 1 room type must be included.
```

**PATH 2 — Derived rates:**
```
Owner selects a parent plan from dropdown.
Enters adjustment: type (% or flat) + value (positive or negative).

Examples:
  "BAR Standard" → -15% → Corporate Rate auto-derived
  "BAR Standard" → +20% → Peak Season rate auto-derived
  "Corporate Rate" → -10% → VIP Corporate auto-derived (chain)

When derived plan selected:
  → Room price fields LOCKED (greyed out, not editable)
  → Preview shown: "Based on current BAR Standard rates:"
    Standard Double: LKR 12,000 → LKR 10,200 (-15%)
    Deluxe King:     LKR 18,000 → LKR 15,300 (-15%)

  → round_to shown: "Round to nearest [ 100 ]" (optional, prevents LKR 10,165)

  ⚠️ Circular chain prevention:
     System checks before saving:
     "Does this derivation create a loop?"
     If yes → error: "Cannot derive. This would create a circular chain: 
                      VIP Corporate → BAR Standard → VIP Corporate."
```

**What system does:**
```
PATH 1 (manual):
  POST /rate-plans/{id}/rooms
  body: [
    { room_type_id: 1, base_rate: 12000, is_active: true },
    { room_type_id: 2, base_rate: 18000, is_active: true },
    { room_type_id: 3, base_rate: 15000, is_active: true },
    { room_type_id: 4, base_rate: 35000, is_active: true }
  ]
  → Creates RatePlanRoom records

PATH 2 (derived):
  POST /rate-plans/derived
  body: {
    parent_rate_plan_id: 5,
    child_rate_plan_id: 8,     ← current draft
    adjustment_type: "PERCENTAGE",
    adjustment_value: -15,
    round_to: 100
  }
  → Circular chain check runs FIRST (application level)
  → If safe: Creates DerivedRatePlan record
  → Calculates child base_rate for each room type
  → Creates RatePlanRoom records with is_derived = true (locked)
  → child plan's RatePlanRoom.base_rate = read-only forever
```

**APIs needed:**
```
GET /rate-plans/{id}/available-parents
  → Returns ACTIVE rate plans this hotel has
  → Excludes: current plan, any plan that would create a circular chain
  → Used to populate the "Derive from" dropdown

POST /rate-plans/{id}/rooms
  → Bulk creates/updates RatePlanRoom

POST /rate-plans/derived
  → Creates DerivedRatePlan + auto-calculates child rates

GET /rate-plans/derived/preview
  query: parent_id=5, adjustment_type=PERCENTAGE, adjustment_value=-15, round_to=100
  → Returns: preview of calculated rates without saving
  → Used to show owner "Standard Double: LKR 12,000 → LKR 10,200" before confirm
```

---

### A-Step 5: Cancellation Policy

**What owner sees:**
```
┌────────────────────────────────────────────────────────────┐
│  Cancellation Policy                                       │
│  What happens if the guest cancels or doesn't show up?    │
│                                                            │
│  Select an existing policy:                                │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ ● Flexible 24H                                       │  │
│  │   Free cancel up to 24h before arrival               │  │
│  │   No-show: First night charged                       │  │
│  │   Used by: 3 rate plans                              │  │
│  │                                                      │  │
│  │ ○ Non-Refundable                                     │  │
│  │   No refund under any condition                      │  │
│  │   No-show: Full stay charged                         │  │
│  │   Used by: 1 rate plan                               │  │
│  │                                                      │  │
│  │ ○ Peak Season Strict                                 │  │
│  │   Free cancel 72h before. 50% penalty within 72h.   │  │
│  │   No-show: Full stay charged                         │  │
│  │   Used by: 0 rate plans                              │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                            │
│  [ + Create a new cancellation policy ]                    │
│  (Creates a reusable policy for this and future plans)     │
└────────────────────────────────────────────────────────────┘

[ ← Back ]                             [ Save & Continue → ]
```

**Key concept shown to owner:**
```
"Policies are reusable. If you update a policy, all rate plans
using it update automatically. You won't need to change each plan separately."
```

**If "Create a new policy" clicked → triggers FLOW B (inline modal)**
After FLOW B completes, new policy auto-selected here and owner continues.

**What system does:**
```
GET /cancellation-policies
  → Returns all policies for this hotel
  → Includes: how many rate plans currently use each policy

PATCH /rate-plans/{id}
  body: { cancellation_policy_id: 3 }
  → Links policy to rate plan
```

**APIs needed:**
```
GET /cancellation-policies
  → Returns list with usage count per policy

PATCH /rate-plans/{id}
  body: { cancellation_policy_id }
```

---

### A-Step 6: Payment Collection

**What owner sees:**
```
┌──────────────────────────────────────────────────────────┐
│  Payment Collection                                      │
│  When and how do you collect payment from guests?       │
│                                                          │
│  Collection Method                                       │
│  ○ Full payment at booking                               │
│    Guest pays 100% when they book. No balance later.    │
│                                                          │
│  ● Deposit at booking, balance later                     │
│    Collect partial now, rest before or at check-in.     │
│                                                          │
│  ○ Full payment at check-in                              │
│    No charge until guest arrives.                        │
│                                                          │
│  ○ Invoice on checkout (corporate)                       │
│    Bill sent to company after stay. No advance payment.  │
│                                                          │
│  ─── If "Deposit at booking" selected ───               │
│                                                          │
│  Deposit Amount                                          │
│  [ 20 ] % of total booking value                        │
│                                                          │
│  Deposit Due                                             │
│  ○ Immediately (when booking is made)                   │
│  ● Within [ 3 ] days of booking                         │
│                                                          │
│  Balance Due                                             │
│  ● At check-in                                           │
│  ○ At check-out                                          │
│  ○ [ 7 ] days before check-in                           │
└──────────────────────────────────────────────────────────┘

[ ← Back ]                             [ Save & Continue → ]
```

**Validation rules:**
```
FULL_AT_BOOKING:      no extra fields needed
DEPOSIT_AT_BOOKING:   deposit_percentage required (1-99%)
                      deposit_due_days required (0 = immediately)
                      balance_due required
FULL_AT_CHECKIN:      no extra fields needed
INVOICE_ON_CHECKOUT:  shown only for CORPORATE template (or CUSTOM)
```

**What system does:**
```
POST /rate-plans/{id}/payment
  body: {
    collection_type: "DEPOSIT_AT_BOOKING",
    deposit_percentage: 20,
    deposit_due_days: 3,
    balance_due: "AT_CHECKIN"
  }
  → Creates or updates RatePlanPayment record
```

**APIs needed:**
```
POST /rate-plans/{id}/payment
  → Creates RatePlanPayment

PATCH /rate-plans/{id}/payment
  → Updates existing RatePlanPayment
```

---

### A-Step 7: Channel Setup

**What owner sees:**
```
┌──────────────────────────────────────────────────────────────┐
│  Distribution Channels                                       │
│  Where do you want to sell this rate plan?                  │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐  │
│  │ [✅] Direct                                            │  │
│  │      Walk-in / Phone / Staff booking / Website         │  │
│  │      Commission: 0%                                    │  │
│  │                                                        │  │
│  │ [✅] Booking.com                                       │  │
│  │      Commission: [ 15 ]%  ← pre-filled, editable      │  │
│  │      OTA Rate Plan Code: [        ] ← optional        │  │
│  │      ⓘ This is the code Booking.com uses for this     │  │
│  │         plan in their extranet. Get it from B.com.    │  │
│  │                                                        │  │
│  │ [  ] Agoda                                             │  │
│  │      Commission: [ 12 ]%                               │  │
│  │      OTA Rate Plan Code: [        ]                    │  │
│  │                                                        │  │
│  │ [  ] Expedia                                           │  │
│  │      Commission: [ 18 ]%                               │  │
│  │      OTA Rate Plan Code: [        ]                    │  │
│  │                                                        │  │
│  │ [  ] Airbnb                                            │  │
│  │      Commission: [  3 ]%                               │  │
│  │      OTA Rate Plan Code: [        ]                    │  │
│  └────────────────────────────────────────────────────────┘  │
│                                                              │
│  ⓘ Channels marked here control WHERE this rate plan is    │
│    visible. OTA sync requires Channel Manager (A1) to be   │
│    enabled. Without A1, you'll sync manually.              │
└──────────────────────────────────────────────────────────────┘

[ ← Back ]                             [ Save & Continue → ]
```

**Key rules:**
```
Direct channel:
  → Always shown. Cannot be unchecked. Every rate plan must be bookable direct.
  → Commission = 0%. Fixed. Cannot change.

OTA channels:
  → Commission pre-filled from Channel.default_commission_pct.
  → Owner edits per rate plan (negotiated rates may differ from default).
  → OTA Rate Plan Code = what Booking.com / Agoda calls this plan in their system.
    Used for sync via Channel Manager (A1). Empty = manual sync only.

Channel Manager warning:
  → If Channel Manager (A1) not enabled:
    "OTA channels are saved but rates won't sync automatically.
     Enable Channel Manager add-on to activate real-time OTA sync."
```

**What system does:**
```
POST /rate-plans/{id}/channels
  body: [
    { channel_id: 1, commission_percentage: 0 },           ← Direct
    { channel_id: 2, commission_percentage: 15,            ← Booking.com
      channel_rate_plan_code: "FLEX-24H" }
  ]
  → Creates RatePlanChannel records for each checked channel

If Channel Manager (A1) is enabled:
  → Triggers OTA sync for each active OTA channel
  → Sends rate plan + commission to OTA API
```

**APIs needed:**
```
GET /channels
  → Returns all available channels (seeded system channels)
  → Includes: name, code, type, default_commission_pct

POST /rate-plans/{id}/channels
  → Bulk creates RatePlanChannel records

DELETE /rate-plans/{id}/channels/{channel_id}
  → Removes a channel from this rate plan
  → If A1 active: triggers OTA stop-sell for this plan
```

---

### A-Step 8: Review & Activate

**What owner sees:**
```
┌──────────────────────────────────────────────────────────────┐
│  Review Rate Plan                                            │
│                                                              │
│  NAME           BAR Standard                                 │
│  CODE           BAR_STANDARD                                 │
│  TEMPLATE       Flexible                                     │
│  PRICING        Per Room                                     │
│  MEAL PLAN      Room Only                                    │
│  VISIBILITY     Public                                       │
│  CURRENCY       LKR                                          │
│                                                              │
│  ROOM PRICES                                                 │
│    Standard Double      LKR 12,000/night                     │
│    Deluxe King          LKR 18,000/night                     │
│    Suite                LKR 35,000/night                     │
│                                                              │
│  CANCELLATION   Flexible 24H                                 │
│    Free cancel until 24h before arrival.                     │
│    No-show: First night charged.                             │
│    Date change: Allowed. Fee: LKR 1,000.                     │
│                                                              │
│  PAYMENT        20% deposit at booking. Balance at check-in. │
│                                                              │
│  CHANNELS       ✅ Direct (0%)  ✅ Booking.com (15%)         │
│                                                              │
│  STATUS:  DRAFT                                              │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  [ Save as Draft ]    [ Activate Rate Plan → ]       │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                              │
│  Activating will make this rate plan live and bookable.      │
│  Once active, changes will show a warning.                   │
└──────────────────────────────────────────────────────────────┘
```

**Two actions:**

**"Save as Draft":**
```
→ PATCH /rate-plans/{id} body: { status: "DRAFT" }
→ Stays in DRAFT. Not visible to guests. Not synced to OTAs.
→ Owner can return anytime to complete or review.
→ Shown in rate plans list with DRAFT badge.
```

**"Activate Rate Plan":**
```
Pre-activation validation:
  ✅ At least 1 room type with base_rate > 0
  ✅ Cancellation policy linked
  ✅ Payment config exists
  ✅ At least 1 channel selected (Direct minimum)
  ✅ Name + code set

If all pass:
  → PATCH /rate-plans/{id} body: { status: "ACTIVE" }
  → Rate plan is now live.
  → If Channel Manager (A1) active: OTA sync fires immediately.
  → Appears on booking widget (if visibility = PUBLIC).
  → Confirmation shown: "Rate plan is now active and bookable."

If validation fails:
  → Error list shown: "These items must be completed before activating:"
  → [ Room prices missing for 2 room types. Set prices. → ]
```

**APIs needed:**
```
GET /rate-plans/{id}/validation
  → Pre-activation check
  → Returns: { valid: true } or { valid: false, errors: [...] }

PATCH /rate-plans/{id}
  body: { status: "ACTIVE" }
  → Status change. Triggers OTA sync if A1 active.
```

---

## FLOW B — Create a Cancellation Policy

Triggered from: Flow A Step 5 ("+ Create new policy") OR standalone from Settings.
This is a reusable entity. Create once, use across many rate plans.

**What owner sees (inline modal or standalone page):**
```
┌────────────────────────────────────────────────────────────┐
│  Create Cancellation Policy                                │
│                                                            │
│  Policy Name *                                             │
│  [                                    ]                    │
│  e.g. "Flexible 24H" / "Peak Season Strict"               │
│                                                            │
│  Cancellation Type *                                       │
│  ○ Free Cancel until X hours before arrival               │
│  ○ Non-Refundable (no refund under any circumstance)      │
│  ○ Partial Penalty (charge if cancelled within window)    │
│                                                            │
│  ─── If "Free Cancel" selected ───                        │
│  Free cancel until: [ 24 ] hours before check-in time     │
│                                                            │
│  ─── If "Partial Penalty" selected ───                    │
│  Charge type:    ○ Percentage  ● Flat amount              │
│  Charge amount:  LKR [ 5,000 ]                            │
│  Within how many hours: [ 48 ] hours before check-in      │
│                                                            │
│  ─────────────────────────────────────────────────────    │
│                                                            │
│  No-Show Charge                                            │
│  What to charge if guest doesn't arrive?                  │
│  ○ Nothing                                                 │
│  ● First night's rate                                      │
│  ○ Full stay                                               │
│  ○ Flat amount: LKR [      ]                               │
│                                                            │
│  ─────────────────────────────────────────────────────    │
│                                                            │
│  Date Change                                               │
│  Can guest change their dates instead of cancelling?      │
│  [✅] Allow date change                                    │
│  Date change fee: LKR [ 1,000 ] (0 = free)               │
│  Free date change until: [ 48 ] hours before check-in     │
│                                                            │
│  [ Cancel ]                            [ Save Policy → ]  │
└────────────────────────────────────────────────────────────┘
```

**Key rule shown to owner:**
```
"This policy will be reusable across all your rate plans.
 If you update it later, all rate plans using it update automatically."
```

**What system does:**
```
POST /cancellation-policies
  body: {
    name: "Flexible 24H",
    policy_type: "FREE_UNTIL",
    free_cancel_until_hours: 24,
    no_show_charge: "FIRST_NIGHT",
    date_change_allowed: true,
    date_change_fee: 1000,
    date_change_window_hours: 48
  }
  → Creates CancellationPolicy record
  → Returns: new policy id
  → If triggered from Flow A Step 5: auto-selects this policy and returns to flow
```

**APIs needed:**
```
GET /cancellation-policies
  → All policies for this hotel

POST /cancellation-policies
  → Create new reusable policy

PATCH /cancellation-policies/{id}
  → Update existing policy
  → ⚠️  If used by active rate plans: warning shown:
    "This policy is used by 3 rate plans. Updating it will affect all of them. Continue?"

DELETE /cancellation-policies/{id}
  → Only allowed if policy is used by 0 rate plans
  → If used: error "Cannot delete. Remove from all rate plans first."
```

---

## FLOW C — Add Date Override (Seasonal / Holiday Pricing)

**Trigger:** Owner clicks existing ACTIVE or DRAFT rate plan → "Seasonal Pricing" tab.

**Purpose:** Override the base rate for specific date ranges. Christmas, Sinhala New Year,
Valentine's Day. Also set CTA/CTD restrictions for peak periods.

**What owner sees:**
```
┌─────────────────────────────────────────────────────────────┐
│  Seasonal Pricing                              [ + Add Override ] │
│                                                             │
│  OVERRIDE NAME       DATES           ROOMS       RATE      │
│  Christmas Peak      Dec 20 - Jan 3  All rooms   LKR 30,000│
│  Valentine's Day     Feb 13-15       All rooms   ×1.20     │
│  New Year's Eve      Dec 31          All rooms   LKR 45,000│
│                      ↳ CTD: guests must stay through Jan 1  │
└─────────────────────────────────────────────────────────────┘
```

**"+ Add Override" form:**
```
┌──────────────────────────────────────────────────────────────┐
│  Add Date Override                                           │
│                                                              │
│  Override Name *     [ Christmas Peak 2027    ]              │
│                                                              │
│  Applies to which rooms?                                     │
│  ● All room types under this plan                            │
│  ○ Specific room type: [ Standard Double ▼ ]                 │
│                                                              │
│  Date Range *                                                │
│  From: [ Dec 20, 2027 ]  To: [ Jan 3, 2028 ]                │
│                                                              │
│  Override Type *                                             │
│  ● Fixed Rate       Set exact rate for these dates           │
│  ○ Multiplier       Multiply base rate (e.g. 1.20 = +20%)   │
│  ○ Adjustment       Add or subtract from base rate           │
│                                                              │
│  ─── Fixed Rate selected ───                                 │
│  New Rate: LKR [ 30,000 ] per night                          │
│                                                              │
│  ─────────────────────────────────────────────────────      │
│                                                              │
│  Arrival / Departure Restrictions                            │
│                                                              │
│  [  ] Closed to Arrival (CTA)                                │
│       No new check-ins during this period.                   │
│       Use case: Christmas Day — guests must arrive Dec 24.   │
│                                                              │
│  [  ] Closed to Departure (CTD)                              │
│       Guests cannot check out during this period.            │
│       Use case: New Year's Eve — guests must stay through.   │
│                                                              │
│  Priority: [ 1 ] (higher number wins if overrides overlap)   │
│                                                              │
│                                        [ Save Override → ]   │
└──────────────────────────────────────────────────────────────┘
```

**Important rule explained to owner:**
```
"Date overrides always win over day-of-week rules.
 If Dec 25 (Saturday) has an override: LKR 30,000,
 and you have a 'Saturdays = ×1.10' rule,
 the override wins. Saturdays rule is ignored for Dec 25."
```

**Overlap handling:**
```
If two overrides cover the same date:
  → Higher priority number wins.
  → System warns: "Dec 25 is already covered by 'Christmas Peak' (priority 1).
     Your new override has priority 1. Set a higher priority to override it."
  → Owner sets priority: 2 → new override wins for Dec 25.
```

**What system does:**
```
POST /rate-plans/{id}/date-overrides
  body: {
    override_name: "Christmas Peak 2027",
    room_type_id: null,            ← null = all room types
    date_from: "2027-12-20",
    date_to: "2028-01-03",
    override_type: "FIXED_RATE",
    override_rate: 30000,
    closed_to_arrival: false,
    closed_to_departure: false,
    priority: 1
  }
  → Creates RatePlanDateOverride record
  → If plan is ACTIVE + Channel Manager enabled: rate sync fires to OTAs
```

**APIs needed:**
```
GET /rate-plans/{id}/date-overrides
  → Returns all overrides sorted by date

POST /rate-plans/{id}/date-overrides
  → Create new override

PATCH /rate-plans/{id}/date-overrides/{override_id}
  → Edit existing override

DELETE /rate-plans/{id}/date-overrides/{override_id}
  → Remove override. If A1 active: OTA rates revert to base rate for those dates.

GET /rate-plans/{id}/date-overrides/conflicts?from=2027-12-20&to=2028-01-03
  → Returns any existing overrides that overlap the requested date range
  → Used for pre-submit warning
```

---

## FLOW D — Setup a Derived Rate Plan

**Trigger:** Owner creating a Corporate, VIP, or promotional rate based on BAR.

This flow is an extension of Flow A Step 4 (derived option selected) but documented
separately for clarity. This is the most powerful rate plan feature.

**Scenario: Creating "Corporate - MAS Holdings" derived from BAR Standard at -15%**

**What owner sees (Step 4 of Flow A, derived path):**
```
┌─────────────────────────────────────────────────────────────┐
│  Derive rates from an existing plan?                        │
│                                                             │
│  Parent Plan:     [ BAR Standard ▼ ]                        │
│  Adjustment:      [ - ] [ 15 ] [ % ▼ ]                      │
│  Round to:        Nearest [ 100 ] LKR                       │
│                                                             │
│  Preview (based on current BAR Standard rates):             │
│  ┌──────────────────────┬────────────┬────────────────┐     │
│  │ Room Type            │ BAR Rate   │ Corporate Rate │     │
│  ├──────────────────────┼────────────┼────────────────┤     │
│  │ Standard Double      │ LKR 12,000 │ LKR 10,200     │     │
│  │ Deluxe King          │ LKR 18,000 │ LKR 15,300     │     │
│  │ Suite                │ LKR 35,000 │ LKR 29,800     │     │
│  └──────────────────────┴────────────┴────────────────┘     │
│                                                             │
│  ⚠️  Room prices above are calculated automatically.        │
│  You cannot edit them manually.                             │
│  If BAR Standard changes, Corporate Rate updates instantly. │
└─────────────────────────────────────────────────────────────┘
```

**Cascade behavior — what happens when parent changes:**
```
Revenue Manager raises BAR Standard:
  Standard Double: LKR 12,000 → LKR 14,000

System immediately:
  1. Detects BAR Standard has derived children (Corporate Rate, VIP Corporate)
  2. Re-calculates Corporate Rate: LKR 14,000 × 0.85 = LKR 11,900 (rounded to LKR 11,900)
  3. Re-calculates VIP Corporate: LKR 11,900 × 0.90 = LKR 10,700 (rounded to LKR 10,700)
  4. Updates all DerivedRatePlan children's RatePlanRoom.base_rate
  5. If Chain Manager (A1) active: fires OTA sync for all affected plans
  6. Logs all changes to RatePlanAuditLog (change_type: RATE_CHANGED, reason: "Parent plan cascade")

Staff sees nothing manually. Auto-cascade completes in < 2 seconds.
```

**Circular chain prevention:**
```
SCENARIO: Staff tries to make BAR Standard derive from Corporate Rate.
  Corporate Rate already derives from BAR Standard.
  This would create: BAR → Corporate → BAR → Corporate → infinite loop.

System check (before save):
  Walk the parent chain starting from "Corporate Rate":
    Corporate Rate → parent: BAR Standard
    BAR Standard → no parent
  Now check: if we make BAR Standard's parent = Corporate Rate:
    BAR Standard → parent: Corporate Rate
    Corporate Rate → parent: BAR Standard ← LOOP FOUND

System blocks: "Cannot set Corporate Rate as parent of BAR Standard.
               This would create a circular chain:
               BAR Standard → Corporate Rate → BAR Standard."
```

**APIs needed:**
```
GET /rate-plans/derived/preview
  query: parent_id, adjustment_type, adjustment_value, round_to
  → Returns calculated rates per room type (no save)

POST /rate-plans/derived
  body: { parent_rate_plan_id, child_rate_plan_id, adjustment_type, adjustment_value, round_to }
  → Circular check first (application logic)
  → Creates DerivedRatePlan
  → Updates child RatePlanRoom records (base_rate auto-calculated, is_derived = true)

GET /rate-plans/derived/chain/{rate_plan_id}
  → Returns the full derivation chain from this plan
  → Used to display: "BAR Standard → Corporate (-15%) → VIP Corporate (-10%)"

POST /rate-plans/derived/cascade/{parent_rate_plan_id}
  → Internal endpoint. Called when parent plan rates change.
  → Re-calculates all children. Recursively handles grandchildren.
  → Not exposed to staff. Triggered internally by rate update flow.
```

---

## FLOW E — Add Package Inclusions

**Trigger:** Owner adds bundled services (meals, transport, spa) to a rate plan.

**What owner sees:**
```
┌───────────────────────────────────────────────────────────────┐
│  Package Inclusions                           [ + Add Item ]  │
│  What's included in this rate plan beyond the room?          │
│                                                               │
│  INCLUDED ITEM     WHEN           CHARGE     REVENUE CENTER   │
│  Breakfast         Daily (×stay)  Included   F&B             │
│  Airport Transfer  At check-in    LKR 2,400  Transport        │
│  Spa Credit        Manual         LKR 2,000  Spa              │
│  Parking           Daily (×stay)  LKR 500    Other            │
└───────────────────────────────────────────────────────────────┘
```

**"+ Add Item" form:**
```
┌──────────────────────────────────────────────────────────────┐
│  Add Package Inclusion                                       │
│                                                              │
│  Service Name *     [ Breakfast             ]                │
│  Description        [ Full Sri Lankan breakfast buffet ]     │
│                                                              │
│  Quantity Basis *                                            │
│  ● Per Room     One per room regardless of guests            │
│  ○ Per Person   One per guest                                │
│  ○ Per Adult    One per adult only (exclude children)        │
│  ○ Per Couple   One for up to 2 adults (romantic packages)   │
│                                                              │
│  Quantity: [ 1 ]                                             │
│                                                              │
│  How Often *                                                 │
│  ● Daily        Posted every night of stay                   │
│  ○ At Check-in  Posted once when guest arrives               │
│  ○ At Check-out Posted once when guest departs               │
│  ○ Manual       Guest redeems at department (spa credit)     │
│                                                              │
│  Is this charged extra?                                      │
│  ○ Included in rate (no extra charge)                        │
│  ● Extra charge: LKR [ 500 ] / unit                          │
│                                                              │
│  Revenue Department *                                        │
│  [ Food & Beverage ▼ ]                                       │
│  (Used for department P&L reporting)                         │
│                                                              │
│  Create a housekeeping / department task?                    │
│  [  ] Yes — auto-create task when this guest checks in       │
│       Task for: [ Housekeeping ▼ ]                           │
│       Instructions: [                                   ]    │
│                                                              │
│                                         [ Save Inclusion ]   │
└──────────────────────────────────────────────────────────────┘
```

**PER_COUPLE explained in UI:**
```
ⓘ "Per Couple" is for items like welcome champagne or romantic dinners
   that apply to a couple (2 adults), not per individual.
   Example: Honeymoon package — 1 bottle champagne per room (not 2).
   Even if 2 adults book: 1 unit. Not 2.
```

**Revenue center impact:**
```
When night audit runs:
  Breakfast (F&B) → posts to F&B department ledger
  Parking (Other) → posts to Other ledger
  Transport (Transport) → posts to Transport ledger

Not all to Rooms. Correct department P&L. Automatic.
```

**APIs needed:**
```
GET /rate-plans/{id}/inclusions
  → Returns all inclusions for this rate plan

POST /rate-plans/{id}/inclusions
  body: {
    service_name: "Breakfast",
    quantity_basis: "PER_ROOM",
    quantity: 1,
    frequency: "DAILY",
    unit_price: 0,
    revenue_center: "FB",
    creates_task: false
  }
  → Creates PackageInclusion record

PATCH /rate-plans/{id}/inclusions/{inclusion_id}
  → Edit existing inclusion

DELETE /rate-plans/{id}/inclusions/{inclusion_id}
  → Remove inclusion
```

---

## FLOW F — Edit Rate on an ACTIVE Rate Plan

**This is the most important flow for ongoing operations.**
Rate changes on live plans have real consequences — future bookings are affected.
System enforces a safe change process.

**Scenario: Revenue Manager wants to raise BAR Standard from LKR 12,000 to LKR 14,000.**

**Step 1: Open active plan**
```
Revenue Manager opens BAR Standard (status: ACTIVE)
Sees rate edit form — room price fields are EDITABLE (manager permission required).

⚠️  Banner shown at top of page:
    "This rate plan is LIVE and actively taking bookings.
     Changes you make here will affect future bookings immediately.
     To safely edit, use DRAFT mode. Or proceed with live edit (requires manager approval)."

Two options:
  [ Edit Safely in Draft Mode ]   ← recommended
  [ Edit Live (Manager PIN) ]     ← immediate, risky
```

**PATH 1 — Edit Safely in Draft Mode (recommended):**
```
Owner clicks "Edit Safely in Draft Mode"

System:
  1. Creates a copy of BAR Standard as a new DRAFT
     "BAR Standard [Draft — Sep 2026]"
  2. Owner edits the copy. Rate: LKR 12,000 → LKR 14,000
  3. Review changes in DRAFT — no risk
  4. When ready: "Activate this version"
  5. System deactivates old plan, activates new plan
  6. All new bookings use new rate
  7. Old bookings: unaffected (they captured rate_snapshot at booking time)

⚠️  Note: This creates a new rate_plan_id.
    All existing bookings still reference the old ID.
    rate_snapshot on each booking is immutable — old rate preserved.
```

**PATH 2 — Edit Live (Manager PIN):**
```
Revenue Manager enters 6-digit manager PIN to confirm.

Before saving, system calculates booking impact:
  "You are changing Deluxe King from LKR 18,000 to LKR 21,000.
   This rate plan has 14 future bookings.
   8 of them are for Deluxe King rooms.
   
   These bookings were made at LKR 18,000.
   Their rate is locked in their booking snapshot.
   They will NOT be affected by this change.
   
   NEW bookings after this change will be priced at LKR 21,000.
   
   [ Cancel ]   [ Confirm Change → ]"

If confirmed:
  1. PATCH /rate-plans/{id}/rooms/{room_type_id}
     body: { base_rate: 21000 }
  2. If derived children exist: cascade runs automatically
  3. Audit log entry created:
     change_type: RATE_CHANGED
     entity_changed: "RatePlanRoom"
     field_changed: "base_rate"
     old_value: "18000"
     new_value: "21000"
     changed_by: manager_id
     booking_impact_count: 8
  4. If Channel Manager (A1) active: OTA rate sync fires
```

**Rate change impact rule (immutable bookings):**
```
KEY PRINCIPLE:
  Changing an active rate plan does NOT change existing bookings.
  Each booking captured rate_snapshot at the moment of creation.
  rate_snapshot is read-only forever after creation.

This means:
  Guest booked Oct 12 at LKR 18,000.
  Revenue Manager raised rate on Oct 15 to LKR 21,000.
  Guest's booking: still LKR 18,000. Immutable.
  New bookings after Oct 15: LKR 21,000.

The system guarantees this automatically via rate_snapshot JSONB field on Booking.
```

**APIs needed:**
```
POST /rate-plans/{id}/create-draft-copy
  → Creates a DRAFT clone of an ACTIVE plan for safe editing
  → Returns: new draft rate_plan_id

GET /rate-plans/{id}/booking-impact
  → Returns: count of future bookings using this plan, by room type
  → Used for impact warning before confirming live edit

PATCH /rate-plans/{id}/rooms/{room_type_id}
  body: { base_rate }
  → Live rate update. Requires manager PIN header.
  → Triggers: audit log + cascade (if derived children) + OTA sync (if A1)

POST /rate-plans/{id}/activate
  → Activates a DRAFT plan (used after safe draft editing)
  → Can optionally deactivate the old plan at same time
```

---

## FLOW G — Archive a Rate Plan

**Trigger:** Rate plan is no longer needed. Cannot delete if past bookings exist.

**What staff sees:**
```
Rate plan actions menu → "Archive this rate plan"

Warning shown:
"Archiving BAR Standard will:
  ✅ Stop new bookings from using this plan immediately
  ✅ Preserve all 234 past bookings that used this plan
  ✅ Keep full rate history in audit log
  ⚠️  Remove this plan from OTA channels (sync will stop)
  ⚠️  If any rate plans derive from this, their rates will freeze

  Active bookings using this plan: 3
  These 3 bookings will continue normally. Their rates are locked in snapshots.

  [ Cancel ]   [ Archive Rate Plan → ]"
```

**Cannot DELETE (hard delete) rule:**
```
ALLOWED: Archive (soft delete — status = ARCHIVED)
NOT ALLOWED: Hard delete if any bookings reference rate_plan_id

Reason: Booking records keep rate_plan_id as FK for audit trail.
        Even if snapshot has all rate data, FK must remain resolvable.

EXCEPTION — Hard delete allowed only if:
  → Plan is in DRAFT status
  → Plan has 0 bookings (was never activated / never used)
```

**What system does:**
```
PATCH /rate-plans/{id}
  body: { status: "ARCHIVED", is_active: false }
  → Rate plan no longer appears in booking widget
  → OTA sync stopped for this plan (stop-sell sent if A1 active)
  → Derived children: their is_active = false, rates frozen at current values
  → Audit log entry: change_type: STATUS_CHANGED, old: ACTIVE, new: ARCHIVED

DELETE /rate-plans/{id}
  → Only if status = DRAFT AND booking_count = 0
  → Hard deletes all child records (RatePlanRoom, RatePlanChannel, etc.)
  → Cascades via FK ON DELETE CASCADE
```

**APIs needed:**
```
PATCH /rate-plans/{id}
  body: { status: "ARCHIVED" }

DELETE /rate-plans/{id}
  → 409 Conflict if any bookings reference this plan

GET /rate-plans/{id}/booking-count
  → Returns total and active booking count
  → Used for archive warning display
```

---

## FLOW H — Add a New OTA Channel to Existing Plan

**Trigger:** Hotel signs up for a new OTA (e.g., Airbnb). Wants to add existing rate plans to it.

**What staff sees:**
```
Open rate plan → "Channels" tab → [ + Add Channel ]

┌────────────────────────────────────────────────────────┐
│  Add Channel                                           │
│                                                        │
│  Channel: [ Airbnb ▼ ]                                 │
│  Commission: [ 3 ]%                                    │
│  OTA Rate Plan Code: [           ]                     │
│  ⓘ Get this from Airbnb host dashboard → Pricing       │
│                                                        │
│  [ Cancel ]          [ Add Channel → ]                 │
└────────────────────────────────────────────────────────┘

After save:
  ✅ Channel added.
  ⚠️  To sync rates automatically, enable Channel Manager (A1).
```

**Edge case — duplicate channel:**
```
Staff tries to add Booking.com to a plan that already has Booking.com.
System: "This rate plan already sells on Booking.com. 
         Go to Channels tab to edit the existing setup."
```

**What system does:**
```
POST /rate-plans/{id}/channels
  body: { channel_id: 5, commission_percentage: 3, channel_rate_plan_code: "" }
  → Unique constraint: one channel per rate plan. Duplicate rejected.
  → If Channel Manager (A1) active: sync fires to OTA immediately
  → Audit log: change_type: CHANNEL_ADDED
```

---

## Complete API Surface — Rate Plans

```
RATE PLAN LIFECYCLE
──────────────────────────────────────────────────────────────────
POST   /rate-plans/draft                     Create draft from template
GET    /rate-plans                           List all plans (with filters)
GET    /rate-plans/{id}                      Get full plan detail
PATCH  /rate-plans/{id}                      Update plan (name, status, visibility...)
DELETE /rate-plans/{id}                      Hard delete (DRAFT + 0 bookings only)
POST   /rate-plans/{id}/activate             DRAFT → ACTIVE
POST   /rate-plans/{id}/create-draft-copy    Safe edit: clone ACTIVE to DRAFT
GET    /rate-plans/templates                 List available templates + pre-fills
GET    /rate-plans/check-code               Uniqueness check for rate plan code
GET    /rate-plans/{id}/validation          Pre-activation validation check
GET    /rate-plans/{id}/booking-impact       Count of future bookings (for impact warning)
GET    /rate-plans/{id}/booking-count        Total + active booking count (for archive check)

ROOM PRICES
──────────────────────────────────────────────────────────────────
GET    /rate-plans/{id}/rooms               Room type prices for this plan
POST   /rate-plans/{id}/rooms               Bulk set room prices
PATCH  /rate-plans/{id}/rooms/{room_type_id} Update single room price (live edit, PIN required)

DERIVED RATE PLANS
──────────────────────────────────────────────────────────────────
GET    /rate-plans/available-parents        Plans that can be a parent (no circular risk)
GET    /rate-plans/derived/preview          Preview derived rates without saving
POST   /rate-plans/derived                  Create derivation (circular check first)
PATCH  /rate-plans/derived/{id}             Edit derivation formula
DELETE /rate-plans/derived/{id}             Remove derivation (child plan becomes manual)
GET    /rate-plans/derived/chain/{id}       Full derivation chain for a plan

CANCELLATION POLICIES
──────────────────────────────────────────────────────────────────
GET    /cancellation-policies               List all reusable policies (with usage count)
POST   /cancellation-policies               Create new policy
PATCH  /cancellation-policies/{id}          Update policy (warns if used by active plans)
DELETE /cancellation-policies/{id}          Delete (only if used by 0 plans)

DATE OVERRIDES
──────────────────────────────────────────────────────────────────
GET    /rate-plans/{id}/date-overrides              All overrides for a plan
POST   /rate-plans/{id}/date-overrides              Add seasonal/holiday override
PATCH  /rate-plans/{id}/date-overrides/{oid}        Edit override
DELETE /rate-plans/{id}/date-overrides/{oid}        Remove override
GET    /rate-plans/{id}/date-overrides/conflicts    Check date range for overlaps

PAYMENT
──────────────────────────────────────────────────────────────────
GET    /rate-plans/{id}/payment             Payment config for this plan
POST   /rate-plans/{id}/payment             Create payment config
PATCH  /rate-plans/{id}/payment             Update payment config

CHANNELS
──────────────────────────────────────────────────────────────────
GET    /channels                            All available channels (seeded system list)
GET    /rate-plans/{id}/channels            Channels this plan is sold on
POST   /rate-plans/{id}/channels            Add a channel
PATCH  /rate-plans/{id}/channels/{cid}      Edit commission / OTA code
DELETE /rate-plans/{id}/channels/{cid}      Remove channel

PACKAGE INCLUSIONS
──────────────────────────────────────────────────────────────────
GET    /rate-plans/{id}/inclusions          All inclusions for this plan
POST   /rate-plans/{id}/inclusions          Add inclusion
PATCH  /rate-plans/{id}/inclusions/{iid}    Edit inclusion
DELETE /rate-plans/{id}/inclusions/{iid}    Remove inclusion

POLICIES
──────────────────────────────────────────────────────────────────
GET    /rate-plans/{id}/policy              Policy (check-in, child, pet rules)
POST   /rate-plans/{id}/policy              Create policy
PATCH  /rate-plans/{id}/policy              Update policy

AUDIT LOG
──────────────────────────────────────────────────────────────────
GET    /rate-plans/{id}/audit-log           Full history for a plan (Phase 2 UI)
GET    /rate-plans/audit-log                All changes across all plans (manager view)
```

---

## Key Business Rules — Summary

```
1. DRAFT FIRST
   Every rate plan starts as DRAFT. Never goes live accidentally.
   Staff must explicitly activate. Guests never see DRAFT plans.

2. IMMUTABLE BOOKING SNAPSHOT
   When a guest books, rate_snapshot is captured and frozen.
   Changing a rate plan never changes existing bookings. Ever.
   Guest's booked rate is always protected.

3. CANCELLATION POLICY IS REUSABLE
   CancellationPolicy is a separate entity. Update once → all linked plans update.
   no_show_charge lives ONLY in CancellationPolicy. Never in RatePlanPolicy.

4. DERIVED RATES — PARENT OWNS THE FORMULA
   If a plan is derived: its base_rate in RatePlanRoom is READ-ONLY.
   Only DerivedRatePlan entity owns the formula.
   Parent changes → all children cascade automatically.

5. CIRCULAR CHAIN PREVENTION
   Before every DerivedRatePlan save: walk the full parent chain.
   If the child_rate_plan_id appears anywhere in the chain → reject.
   Must be application-level check. Database FK cannot catch this.

6. DATE OVERRIDE BEATS DAY RULE
   If a date has both a DateOverride and a DayRule (weekend multiplier):
   DateOverride wins. Always. Day rule is ignored for that specific date.

7. CTA EXISTS IN TWO PLACES — DIFFERENT USE CASES
   DateOverride.closed_to_arrival  → specific date blocked
   LOSRule.closed_to_arrival       → LOS doesn't meet requirement
   Both are checked independently. Either true = booking blocked.

8. CHANNEL IS A MASTER TABLE — NOT AN ENUM
   New OTA → insert one row in Channel table. Zero code change.
   RatePlanChannel references channel_id FK. Never a hardcoded string.

9. HARD DELETE FORBIDDEN IF BOOKINGS EXIST
   Rate plans with any booking (past or active) cannot be hard deleted.
   Archived instead. FK integrity for audit trail must be maintained.

10. MANAGER PERMISSION FOR LIVE RATE CHANGE
    Changing a rate on an ACTIVE plan requires manager PIN.
    Impact count shown before confirming: "8 future bookings affected."
    Bookings themselves are not changed — only new bookings use new rate.
```
