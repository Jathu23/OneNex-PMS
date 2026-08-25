# Setup & Configuration — 10: Notification Setup
> Hotel Module → Setup & Configuration → Area 10 of 16
> Covers: Guest Notifications + Staff Alerts + Templates + Timing + Channel Config
> Foundation for: Guest Communication, Staff Coordination, Escalation Alerts

---

## Why Notification Setup Matters

```
Without Notification Setup:
  → Guest doesn't know booking is confirmed → calls front desk → wasted time
  → Staff doesn't get housekeeping task alert → room not ready at check-in
  → GM doesn't know about complaint → escalation missed
  → Guest gets wrong language notification → bad experience

With proper setup:
  → Every event triggers the right message to the right person
  → Zero manual follow-up needed
  → Guest feels professionally handled throughout their stay
```

---

## Existing Systems — Problems

| System | Problem |
|--------|---------|
| Oracle OPERA | Notifications hardcoded. Can't customize template. Guest gets generic email with zero branding. |
| Mews | Good email notifications. No WhatsApp. No template editor for small hotels. |
| Cloudbeds | Email only. SMS extra cost add-on. No internal staff alerts built-in. |
| All systems | No timing control. Booking confirmation fires immediately — no option to delay. Pre-arrival reminder timing not configurable. |

---

## Our Design Principles

### 1. Two Notification Streams: Guest + Staff

```
GUEST NOTIFICATIONS          STAFF / INTERNAL NOTIFICATIONS
─────────────────────        ──────────────────────────────
Booking Confirmation         New Booking Alert
Payment Receipt              Check-in Due (arrival today)
Pre-Arrival Reminder         Room Not Ready Alert
Check-in Instructions        Maintenance Ticket Raised
Welcome Message              Maintenance Overdue Alert
Housekeeping DND Alert       No-Show Detected
Checkout Reminder            Low Inventory Warning
Invoice / Folio Summary      Night Audit Completed
Cancellation Confirmation    Complaint / Feedback Received
No-Show Charge Notice        VIP Guest Arriving
Feedback Request             Group Cut-off Approaching
```

### 2. Notification Channels Per Event
```
Each notification: independently configurable per channel

EVENT: Booking Confirmation
  Email:      ✅ ON
  SMS:        ✅ ON
  WhatsApp:   ✅ ON
  Push:       ☐ OFF

EVENT: Housekeeping Task Assigned (internal)
  Email:      ☐ OFF
  SMS:        ☐ OFF
  WhatsApp:   ✅ ON  (housekeeper's WhatsApp)
  Push:       ✅ ON  (mobile app)

EVENT: Night Audit Report (internal)
  Email:      ✅ ON  (GM + Accounts)
  SMS:        ☐ OFF
  WhatsApp:   ☐ OFF
  Push:       ☐ OFF
```

### 3. Template Editor
```
TEMPLATE: Booking Confirmation (Email)

Subject:  Your booking at {{hotel_name}} is confirmed!

Body:
  Dear {{guest_name}},

  Thank you for choosing {{hotel_name}}.
  Your reservation details:

  Booking ID:     {{booking_id}}
  Room Type:      {{room_type}}
  Check-in:       {{checkin_date}} at {{checkin_time}}
  Check-out:      {{checkout_date}} at {{checkout_time}}
  Nights:         {{total_nights}}
  Rate Plan:      {{rate_plan_name}}
  Total Amount:   {{total_amount}}
  Amount Paid:    {{amount_paid}}
  Balance Due:    {{balance_due}}

  For assistance: {{hotel_phone}} | {{hotel_email}}

  Warm regards,
  {{hotel_name}} Team

AVAILABLE VARIABLES:
  Guest:    {{guest_name}}, {{guest_phone}}, {{guest_email}}
  Booking:  {{booking_id}}, {{room_type}}, {{room_number}},
            {{checkin_date}}, {{checkout_date}}, {{total_nights}},
            {{rate_plan_name}}, {{total_amount}}, {{amount_paid}},
            {{balance_due}}, {{special_requests}}
  Hotel:    {{hotel_name}}, {{hotel_address}}, {{hotel_phone}},
            {{hotel_email}}, {{hotel_logo_url}}, {{checkin_time}},
            {{checkout_time}}
```

### 4. Timing Configuration
```
TIMING SETUP PER NOTIFICATION:

  Pre-Arrival Reminder:
    Send:    X days before check-in
    Options: 1 day / 2 days / 3 days / 7 days / custom
    Time:    10:00 AM (hotel's timezone)

  Checkout Reminder:
    Send:    Morning of checkout
    Time:    8:00 AM

  Feedback Request:
    Send:    X hours after checkout
    Options: 2 hrs / 6 hrs / 24 hrs / next day morning

  Check-in Instructions:
    Send:    Same day as check-in
    Time:    8:00 AM

  No-Show Charge Notice:
    Send:    Immediately after night audit processes no-show
```

### 5. Staff Alert Routing
```
INTERNAL ALERT ROUTING:

  New Booking:
    → Front Desk Manager (if > ₹X value)
    → All Front Desk Staff

  Maintenance Ticket - Emergency:
    → Maintenance Supervisor (WhatsApp + Push)
    → GM (WhatsApp)
    → If not responded in 30 min → re-alert + escalate to GM

  Room Not Ready (check-in time passed, room still OD):
    → Housekeeping Supervisor
    → Front Desk Manager

  VIP Guest Arriving:
    → GM
    → Front Desk Manager
    → Housekeeping Supervisor

  Low Room Inventory (< X% available):
    → Owner / GM
    → Revenue Manager (if role exists)

  Night Audit Completed:
    → GM (email report)
    → Accounts (email report)

Alert routing linked to Staff Setup — uses department + role.
```

### 6. Guest Language Preference
```
LANGUAGE CONFIG:
  Default notification language:    English
  Auto-detect from booking source:  Yes / No
    (OTA booking from Japan → send in Japanese if template exists)

  Template languages available:
    → English (required)
    → Tamil, Hindi, Telugu (optional — hotel adds their own translation)
    → System provides English base, hotel translates other languages

V1: English only
Phase 2: Multi-language templates
```

### 7. WhatsApp Business Setup
```
WhatsApp Notification Config:
  Provider:         Meta WhatsApp Business API
                    OR Twilio / WATI / Interakt (aggregators)
  Phone number:     Hotel's WhatsApp Business number
  Template approval: Required by Meta before sending (system guides through)

  Message format (WhatsApp templates are pre-approved text):
  "Hi {{guest_name}}, your booking at {{hotel_name}} is confirmed.
   Check-in: {{checkin_date}}. Booking ID: {{booking_id}}.
   Reply HELP for assistance."

  WhatsApp templates: pre-approved by Meta → hotel selects from approved list
  Cannot send free-form marketing via WhatsApp (Meta policy)
```

---

## Data Model

```
NotificationEvent
  id
  code                  "BOOKING_CONFIRMED" / "PRE_ARRIVAL" / "CHECKOUT_REMINDER" /
                        "MAINTENANCE_TICKET_RAISED" / "NIGHT_AUDIT_DONE" ...
  name                  "Booking Confirmation"
  stream                GUEST / STAFF
  description

NotificationConfig
  id, hotel_id
  event_id              FK → NotificationEvent
  email_enabled         bool
  sms_enabled           bool
  whatsapp_enabled      bool
  push_enabled          bool
  is_active             bool

NotificationTiming
  config_id
  timing_type           IMMEDIATE / BEFORE_EVENT / AFTER_EVENT / SCHEDULED_TIME
  offset_value          int nullable (e.g. 2)
  offset_unit           HOURS / DAYS nullable
  scheduled_time        "10:00" nullable

NotificationTemplate
  id, hotel_id
  config_id
  channel               EMAIL / SMS / WHATSAPP / PUSH
  language              "en" / "ta" / "hi"
  subject               nullable (email only)
  body                  text (with {{variable}} placeholders)
  is_active             bool

NotificationRecipient (for staff/internal alerts)
  config_id
  recipient_type        ROLE / DEPARTMENT / SPECIFIC_STAFF
  role_id               nullable
  department            nullable
  staff_id              nullable

IntegrationCredentials (notification providers)
  hotel_id
  provider_type         EMAIL / SMS / WHATSAPP
  provider_name         "SendGrid" / "Twilio" / "WATI"
  credentials           JSON (encrypted — API keys, sender IDs)
  is_active             bool

NotificationLog (runtime)
  id, hotel_id
  event_code
  recipient_type        GUEST / STAFF
  recipient_id
  channel
  status                SENT / FAILED / PENDING
  sent_at               timestamp
  error_message         nullable
```

---

## Key Relationships

```
Hotel → NotificationConfig (many — one per event)
NotificationConfig → NotificationTemplate (one per channel per language)
NotificationConfig → NotificationTiming (one per config)
NotificationConfig → NotificationRecipient (for staff alerts — many)

Booking event → NotificationLog (runtime: what was sent, when, to whom)
IntegrationCredentials → email/SMS/WhatsApp provider (one per channel)
```

---

## V1 vs Phase Split

| Feature | V1 | Phase 2 | Phase 3 |
|---------|:--:|:-------:|:-------:|
| Booking confirmation (email + SMS) | ✅ | | |
| Payment receipt (email) | ✅ | | |
| Cancellation confirmation (email + SMS) | ✅ | | |
| Pre-arrival reminder (timing configurable) | ✅ | | |
| Checkout reminder | ✅ | | |
| No-show charge notice | ✅ | | |
| Night audit report (email to GM) | ✅ | | |
| Maintenance ticket alert (staff) | ✅ | | |
| Room not ready alert (staff) | ✅ | | |
| Template editor with variables | ✅ | | |
| Staff alert routing by role/department | ✅ | | |
| WhatsApp notifications (via WATI/Twilio) | ✅ | | |
| Multi-language templates | | ✅ | |
| Guest app push notifications | | ✅ | |
| Feedback request after checkout | | ✅ | |
| VIP arrival alert | | ✅ | |
| Low inventory alert | | ✅ | |
| Escalation chain (if not responded in X mins) | | ✅ | |
| AI-personalized guest messaging | | | ✅ |
