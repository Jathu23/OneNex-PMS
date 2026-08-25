# Setup & Configuration — 09: Folio, Billing & Night Audit Setup
> Hotel Module → Setup & Configuration → Area 9 of 16
> Covers: Folio Structure + Invoice Format + Night Audit + Auto-Post Rules + No-Show + Settlement
> Foundation for: Guest Billing, Cross-Module Charges, Night Audit Operations, Corporate Invoicing

---

## Why This is Critical

```
Folio Setup defines:
  → How is a guest's bill structured?
  → What number format do invoices use?
  → When does night audit run?
  → Which charges auto-post vs manual?
  → How are cross-module charges routed?
  → What happens to no-show guests at midnight?

Wrong setup = billing chaos.
Right setup = system runs financials automatically, zero manual work.
```

---

## Existing Systems — Problems

| System | Problem |
|--------|---------|
| Oracle OPERA | Night audit is a complex multi-step manual process. Staff must run it in a specific order. One mistake = full re-run required. Junior staff scared to touch it. |
| Mews | Auto night audit but no configuration on what to post. Charges appear inconsistently. |
| Cloudbeds | No cross-module routing. Restaurant charges must be manually entered into room folio. |
| All systems | Invoice number format not configurable. No split folio setup. No comp/discount audit trail in setup rules. |

---

## Our Design Principles

### 1. Invoice / Folio Number Format
```
FORMAT SETUP:
  Prefix:        INV
  Year:          Yes (2 digit / 4 digit / none)
  Month:         Yes / No
  Separator:     - / / / none
  Sequence:      5 digits, reset yearly / monthly / never

Examples based on config:
  INV-2025-00001    (prefix + 4-digit year + 5-digit sequence)
  INV/25/001234     (prefix + 2-digit year + 6-digit, reset yearly)
  TGC-00001         (custom prefix, no date, never reset)

Preview shown live as hotel configures.
Each property: separate number sequence.
```

### 2. Night Audit — Configuration
```
NIGHT AUDIT SETUP:
  Run time:              23:59 (default — configurable)
  Auto-run:              YES (system runs automatically)
  Manual override:       YES (manager can trigger early if needed)

  What happens at night audit:
  ┌─────────────────────────────────────────────────────────┐
  │ NIGHT AUDIT — 11:59 PM                                  │
  │                                                         │
  │ 1. Post room charges to all OPEN folios                 │
  │    (every checked-in guest → +1 night room charge)      │
  │                                                         │
  │ 2. Post meal plan charges (if applicable)               │
  │    (Half Board → post breakfast + dinner charge)        │
  │                                                         │
  │ 3. Process no-show guests                               │
  │    (no check-in by audit time → auto-charge + cancel)   │
  │                                                         │
  │ 4. Auto-release held rooms past expiry                  │
  │    (soft locks expired → release to inventory)          │
  │                                                         │
  │ 5. Generate night audit report                          │
  │    (send to: GM + Accounts via email)                   │
  │                                                         │
  │ 6. Roll date → new business day starts                  │
  └─────────────────────────────────────────────────────────┘

Config: which of these steps are active?
Config: who receives the audit report?
```

### 3. Auto-Post Rules
```
CHARGE AUTO-POST CONFIG:

  Room charge:
    Post at:        Night audit (default) / Check-in / Daily at custom time
    Post label:     "Room Charge — Night of [date]"
    Tax:            Auto-apply GST as per Tax Config

  Meal plan charge:
    Breakfast:      Post at 10:30 AM (after breakfast service ends)
    Dinner:         Post at 10:30 PM
    Label:          "Breakfast Charge — [date]"

  Mini bar:
    Post at:        When staff logs consumption (instant)
    Label:          "Mini Bar — [item name] × qty"

  Cross-module (Restaurant / Bar / Spa):
    Route:          "Charge to Room" → instant folio entry
    Label:          "[Module]: [description] — [date time]"
    Requires:       Guest must be checked in at time of charge

  Parking:
    Post at:        Night audit (1 per night) OR at checkout
    Label:          "Parking — Night of [date]"

  Late checkout fee:
    Post at:        When late checkout is approved
    Label:          "Late Checkout Fee — [hours] hrs"
```

### 4. No-Show Configuration
```
NO-SHOW RULES (per rate plan — set in Rate Plan Setup):
  Charge:    First night / Full stay / No charge / Custom %
  Timing:    At night audit / Next morning at X time
  Auto-cancel booking: Yes / No (if yes → room released to inventory)
  Notify guest: Yes (email/SMS with charge details)

NO-SHOW AUDIT FLOW:
  Night audit runs → finds bookings with:
    status = CONFIRMED
    check_in_date = today
    actual_checkin_time = NULL

  → Apply no-show charge per rate plan policy
  → Post charge to folio
  → Change booking status: CONFIRMED → NO_SHOW
  → Release room to inventory
  → Send notification to guest
  → Log in audit report
```

### 5. Folio Structure
```
FOLIO TYPES:
  ROOM_FOLIO    (default — one per booking, all charges go here)
  MASTER_FOLIO  (group bookings — one bill for all rooms)
  SPLIT_FOLIO   (guest requests: room charges on company, personal on guest)

SPLIT FOLIO SETUP:
  Allow split folio?           Yes / No
  Who can create split folio?  Front Desk Manager only / Any FD staff
  Split types allowed:
    → Room vs incidentals (most common)
    → Per department (company pays rooms, guest pays F&B)
    → Custom split

MASTER BILL (Corporate / Group):
  Who pays:        Corporate account / Group organizer
  Invoice format:  Consolidated (one total) / Itemized (per room)
  Delivery:        Email / WhatsApp / Portal download
  Payment terms:   Immediate / Net 30 / Net 60
```

### 6. Comp & Discount Rules (Folio Level)
```
COMP CONFIG:
  Allow comp items?             Yes / No
  Who can comp?                 Manager only (per role permission)
  Comp reason required?         Yes (must select from list)
  Comp reason options:
    → Service failure
    → VIP / loyalty gesture
    → Owner decision
    → Complaint resolution
    → Other (free text)

  Comp items logged:
    → What was comped
    → Who approved
    → Reason selected
    → Manager PIN override used? Yes/No

DISCOUNT CONFIG:
  Max discount % per role:      Configured in Staff Setup
  Discount reason required?     Yes / No
  Auto-flag if > X%:            Yes (alert to GM report)
```

### 7. Settlement Options
```
PAYMENT METHODS ACCEPTED AT SETTLEMENT:
  ✅ Cash
  ✅ Card (swipe / tap)
  ✅ UPI (QR code)
  ✅ Bank Transfer
  ✅ Corporate Account (charge to company, invoice later)

PARTIAL PAYMENT:
  Allow partial settlement?     Yes
  Remaining balance:            Must settle before checkout (enforced)

CITY LEDGER (Corporate):
  Corporate account set up → checkout without cash payment
  Invoice generated → sent to company on schedule
  Payment tracked separately in accounts module
```

### 8. Tax Posting on Folio
```
TAX DISPLAY ON FOLIO:
  Show tax breakdown?    Yes (line by line) / No (included in price)

  EXAMPLE — Folio with tax breakdown:
    Room Charge — Dec 20    ₹5,000
    GST 12% on Room         ₹600
    Breakfast — Dec 21      ₹500
    GST 5% on Food          ₹25
    ─────────────────────────────
    Subtotal                ₹5,500
    Total Tax               ₹625
    GRAND TOTAL             ₹6,125

Tax rates pulled from Tax Configuration (Area 14).
Auto-applied — no manual calculation.
```

---

## Data Model

```
FolioConfig
  hotel_id
  invoice_prefix              "INV"
  invoice_year_format         NONE / TWO_DIGIT / FOUR_DIGIT
  invoice_month_included      bool
  invoice_separator           "-" / "/" / ""
  invoice_sequence_digits     int (5)
  invoice_sequence_reset      NEVER / YEARLY / MONTHLY
  invoice_current_sequence    int (auto-incremented)

  allow_split_folio           bool
  split_folio_role            ANY_FD / MANAGER_ONLY
  allow_master_bill           bool
  allow_comp                  bool
  comp_reason_required        bool
  comp_reasons                JSON [list of reasons]
  discount_reason_required    bool
  discount_alert_above_pct    int nullable

NightAuditConfig
  hotel_id
  run_time                    "23:59"
  auto_run                    bool
  post_room_charges           bool
  post_meal_plan_charges      bool
  process_no_shows            bool
  auto_release_expired_holds  bool
  generate_report             bool
  report_recipients           JSON [staff_id list]

AutoPostRule
  id, hotel_id
  charge_type                 ROOM / BREAKFAST / DINNER / MINI_BAR /
                              PARKING / LATE_CHECKOUT / CROSS_MODULE
  post_trigger                NIGHT_AUDIT / INSTANT / CUSTOM_TIME
  post_time                   nullable "10:30"
  charge_label_template       "Room Charge — Night of {date}"
  tax_apply                   bool

FolioCharge (runtime — schema must be setup-ready from day 1)
  id, folio_id
  charge_type
  description
  amount
  tax_amount
  tax_rate
  posted_by_staff_id
  source_module               HOTEL / RESTAURANT / BAR / SPA / MAINTENANCE
  source_reference_id         nullable
  is_comp                     bool
  comp_reason                 nullable
  comp_approved_by_staff_id   nullable
  discount_pct                nullable
  discount_approved_by        nullable
  posted_at                   timestamp
  night_audit_id              nullable FK

NightAuditRun (runtime log)
  id, hotel_id
  run_date                    date
  started_at, completed_at    timestamp
  triggered_by                AUTO / MANUAL
  triggered_by_staff_id       nullable
  rooms_charged               int
  no_shows_processed          int
  total_revenue_posted        decimal
  status                      RUNNING / COMPLETED / FAILED
  report_url                  nullable
```

---

## Key Relationships

```
Hotel → FolioConfig (one per hotel)
Hotel → NightAuditConfig (one per hotel)
Hotel → AutoPostRule (many — one per charge type)

Booking → Folio (one default, optionally split)
Folio → FolioCharge (many)
FolioCharge → Staff (who posted)
FolioCharge → NightAuditRun (if auto-posted by audit)
FolioCharge → source module reference (Restaurant / Bar / Spa)

NightAuditConfig → NightAuditRun (runtime execution log)
FolioConfig → Invoice number generation (sequence auto-increments)
```

---

## V1 vs Phase Split

| Feature | V1 | Phase 2 | Phase 3 |
|---------|:--:|:-------:|:-------:|
| Invoice number format config | ✅ | | |
| Night audit auto-run at configured time | ✅ | | |
| Room charge auto-post at night audit | ✅ | | |
| No-show auto-charge + release | ✅ | | |
| Cross-module charge routing (Restaurant/Bar/Spa) | ✅ | | |
| Comp with reason + manager approval | ✅ | | |
| Discount limit + reason | ✅ | | |
| Tax auto-apply on folio charges | ✅ | | |
| Split folio | ✅ | | |
| Settlement: Cash + Card + UPI | ✅ | | |
| Night audit report (email to GM) | ✅ | | |
| Folio PDF generation | ✅ | | |
| Master bill (corporate / group) | ✅ | | |
| Meal plan charge auto-post timing | ✅ | | |
| City ledger / corporate credit | | ✅ | |
| Folio dispute workflow | | ✅ | |
| Payment gateway integration (online settlement) | | ✅ | |
| Accounting software export (Tally, Zoho Books) | | ✅ | |
| Multi-currency folio | | | ✅ |
