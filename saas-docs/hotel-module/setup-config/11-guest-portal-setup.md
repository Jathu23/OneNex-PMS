# Setup & Configuration — 11: Guest Portal Setup
> Hotel Module → Setup & Configuration → Area 11 of 16
> Covers: Booking Widget + Online Check-in + Folio Self-View + Add-ons + Promo Codes
> Foundation for: Direct Bookings, Online Check-in, Guest Self-Service, Upsell Revenue

---

## Why Guest Portal Setup Matters

```
Guest Portal = Hotel's digital front door.

Without proper setup:
  → No direct booking → 100% OTA dependent → commission waste
  → No online check-in → all guests queue at front desk
  → Guest can't see their bill → calls front desk for every query
  → No special request capture → guest arrives expecting something hotel didn't know

With proper setup:
  → Direct bookings increase → OTA commission saved
  → Online check-in → front desk queue reduced 60%
  → Self-service folio view → zero calls for bill queries
  → Special requests captured at booking → staff prepared
```

---

## Existing Systems — Problems

| System | Problem |
|--------|---------|
| Oracle OPERA | No built-in booking portal. Needs third-party booking engine (extra cost + integration headache). |
| Mews | Good booking widget but limited customization. Brand colors only — no full white-label. |
| Cloudbeds | Booking engine included but online check-in is paid add-on. Guest portal is bare-bones. |
| All systems | No folio self-view. Guest must call front desk to know current bill. No digital request management. |

---

## Our Design Principles

### 1. Booking Widget — Embeddable
```
BOOKING WIDGET SETUP:
  Widget type:      Embedded (hotel's own website) /
                    Hosted page (platform URL) /
                    Both

  Widget config:
    Which room types to show?     All / Selected only
    Which rate plans to show?     All public / Selected
    Max advance booking:          365 days / custom
    Min advance booking:          0 hours / custom (e.g. book min 4 hrs ahead)
    Show availability calendar?   Yes / No
    Show price per night?         Yes / No
    Show rate plan details?       Yes / No (cancellation policy, inclusions)
    Promo code field?             Yes / No

  Embed code: one line of JavaScript
  Hotel pastes into their website → widget appears instantly
```

### 2. Booking Flow — What Guest Sees
```
STEP 1: Search
  Check-in date | Check-out date | Adults | Children
  → Shows available room types with photos + prices

STEP 2: Select Room
  Room type cards:
    Photo, name, description, amenities
    Rate plans available (Flexible / Non-Refundable / B&B...)
    Price per night + total
    Cancellation policy preview
  → Guest selects room + rate plan

STEP 3: Guest Details
  Required fields (hotel configures):
    Name, Email, Phone (minimum)
  Optional fields (hotel configures):
    Address, ID type + number, Nationality,
    Special requests (free text + preset options),
    Estimated arrival time,
    Add-ons (airport pickup, extra bed, flowers...)

STEP 4: Payment
  Based on rate plan payment config:
    PAY_NOW → full payment now
    DEPOSIT → % amount now
    CC_GUARANTEE → card details (no charge now)
    PAY_AT_HOTEL → no payment now

STEP 5: Confirmation
  Booking ID shown
  Confirmation email + SMS fired immediately
```

### 3. Required Guest Fields — Hotel Configures
```
FIELD CONFIG:
  Field               Required?   Shown?
  ─────────────────────────────────────────
  Name                ✅ Always   ✅
  Email               ✅ Always   ✅
  Phone               ✅ Always   ✅
  Adults count        ✅ Always   ✅
  Children count      ✅ Always   ✅
  Address             Hotel config (required / optional / hidden)
  ID Type + Number    Hotel config
  Nationality         Hotel config
  Special Requests    Hotel config
  Arrival Time (ETA)  Hotel config
  Add-ons             Hotel config (which add-ons to offer)

Special Request Presets (hotel defines):
  ☐ High floor preference
  ☐ Quiet room
  ☐ Extra pillows
  ☐ Early check-in (subject to availability)
  ☐ Late check-out (subject to availability)
  ☐ Anniversary / Birthday setup
  ☐ Airport pickup
  + Free text box
```

### 4. Online Check-in
```
ONLINE CHECK-IN SETUP:
  Enable?              Yes / No
  Available from:      X hours before check-in (e.g. 24 hrs)
  Available until:     Check-in time

  Guest flow:
    Pre-arrival email/WhatsApp → "Complete check-in online"
    → Guest clicks link → opens check-in form
    → Verify identity (upload ID photo)
    → Confirm arrival time
    → Any special requests?
    → Digital signature (accept T&C)
    → Done → "Your room will be ready at [check-in time]"

  Front desk sees:
    "Online check-in completed" flag on booking
    ID upload attached
    Arrival time confirmed
    → Room assignment can be done in advance
    → Guest arrives → minimal wait → key handover only

  Digital key (Phase 3):
    QR code or NFC → guest goes directly to room
```

### 5. Guest Folio Self-View
```
FOLIO SELF-VIEW SETUP:
  Enable?              Yes / No
  Access via:          Booking confirmation link (unique per booking)
                       OR guest login (email + booking ID)

  What guest can see:
    ✅ All charges posted so far
    ✅ Payments made
    ✅ Balance due
    ✅ Rate plan details
    ✅ Download invoice (PDF)
    ☐ Cannot see internal notes
    ☐ Cannot modify charges

  Real-time:
    Every charge posted → visible to guest immediately
    "Just had dinner at restaurant? See ₹850 added to your bill."

  Benefit:
    → Zero surprise at checkout
    → Guest can dispute mid-stay (easier to resolve)
    → Reduces checkout time significantly
```

### 6. Add-ons / Upsells
```
ADD-ON SETUP (hotel defines what to offer at booking):

  ADD-ON: Airport Pickup
    Price:    ₹800 one-way
    When:     At booking (guest selects)
    Routing:  Auto-posts to folio OR separate invoice

  ADD-ON: Early Check-in (11 AM)
    Price:    ₹500 flat
    When:     At booking OR via portal pre-arrival
    Subject to: Room availability (system checks)

  ADD-ON: Birthday Decoration
    Price:    ₹1,500
    When:     At booking
    Note:     Auto-creates housekeeping special task

  ADD-ON: Extra Bed
    Price:    ₹800/night
    When:     At booking
    Routing:  Auto-adds to room charge calculation

Each add-on: charge auto-posts to folio when selected.
No manual staff entry needed.
```

### 7. Promo Code / Special Rate Access
```
PROMO CODE SETUP:
  Code:             SUMMER25
  Discount:         25% off base rate
  Rate plan:        Applied to "Flexible" plan only
  Valid:            Jun 1 – Aug 31
  Usage limit:      100 bookings / unlimited
  Min stay:         2 nights
  Visibility:       Hidden (only accessible with code)

Guest enters code at booking widget → rate updates instantly.

Use cases:
  → Email campaign to past guests
  → Social media promotions
  → Corporate negotiated codes
  → Travel agent codes
```

---

## Data Model

```
GuestPortalConfig
  hotel_id
  booking_widget_enabled        bool
  widget_embed_type             EMBEDDED / HOSTED / BOTH
  hosted_page_url               nullable (auto-generated)
  max_advance_booking_days      int (365)
  min_advance_booking_hours     int (0)
  show_availability_calendar    bool
  show_price_breakdown          bool
  show_rate_plan_details        bool
  promo_code_enabled            bool

  online_checkin_enabled        bool
  checkin_available_hours_before int (24)
  checkin_id_upload_required    bool
  checkin_digital_signature     bool

  folio_self_view_enabled       bool
  folio_access_method           LINK / LOGIN

BookingFieldConfig
  hotel_id
  field_name                    "address" / "id_number" / "nationality" / "eta" ...
  is_required                   bool
  is_shown                      bool

SpecialRequestPreset
  id, hotel_id
  label                         "High floor preference"
  display_order                 int
  is_active                     bool

AddOn
  id, hotel_id
  name                          "Airport Pickup"
  description
  price                         decimal
  price_unit                    FLAT / PER_NIGHT / PER_PERSON
  available_at                  BOOKING / PRE_ARRIVAL / BOTH
  folio_post_type               INSTANT / AT_CHECKOUT
  charge_label                  "Airport Pickup — Arrival"
  subject_to_availability       bool
  creates_task                  bool
  task_department               nullable (HOUSEKEEPING / MAINTENANCE / FD)
  is_active                     bool

PromoCode
  id, hotel_id
  code                          "SUMMER25"
  discount_type                 PERCENTAGE / FLAT
  discount_value                decimal
  applicable_rate_plan_ids      JSON nullable (null = all)
  valid_from                    date
  valid_until                   date
  min_nights                    int nullable
  usage_limit                   int nullable
  usage_count                   int (auto-incremented)
  is_active                     bool

OnlineCheckIn (runtime)
  id, booking_id
  completed_at                  timestamp
  id_document_url               nullable
  estimated_arrival_time        time nullable
  digital_signature_url         nullable
  special_requests              text nullable
```

---

## Key Relationships

```
Hotel → GuestPortalConfig (one per hotel)
Hotel → BookingFieldConfig (many — one per optional field)
Hotel → SpecialRequestPreset (many)
Hotel → AddOn (many)
Hotel → PromoCode (many)

Booking → OnlineCheckIn (one — if guest completes online check-in)
AddOn → FolioCharge (auto-posts when selected)
AddOn → HousekeepingTask (if creates_task = true)
PromoCode → Booking (used_promo_code_id FK on booking)
```

---

## V1 vs Phase Split

| Feature | V1 | Phase 2 | Phase 3 |
|---------|:--:|:-------:|:-------:|
| Booking widget (embedded + hosted page) | ✅ | | |
| Room + rate plan selection | ✅ | | |
| Guest details form (configurable fields) | ✅ | | |
| Special request presets | ✅ | | |
| Payment at booking (Pay Now / Deposit / Guarantee) | ✅ | | |
| Booking confirmation page | ✅ | | |
| Promo code support | ✅ | | |
| Add-ons at booking (airport pickup, extra bed) | ✅ | | |
| Folio self-view (via link) | ✅ | | |
| Folio PDF download | ✅ | | |
| Online check-in (ID upload + ETA) | | ✅ | |
| Guest login portal (booking history) | | ✅ | |
| Pre-arrival add-on upsell | | ✅ | |
| Guest messaging (chat with hotel via portal) | | ✅ | |
| Loyalty points display | | ✅ | |
| Digital room key (QR / NFC) | | | ✅ |
| AI room upgrade suggestions | | | ✅ |
| Guest app (iOS + Android) | | | ✅ |
