# Setup & Configuration — 15: Group & Corporate Setup
> Hotel Module → Setup & Configuration → Area 15 of 16
> Covers: Group Blocks + Rooming List + Cut-off Logic + Corporate Accounts + City Ledger + Auto-Invoice
> Foundation for: Group Booking Operations, Corporate Billing, City Ledger, Master Invoicing

---

## Why Group & Corporate Setup Matters

```
Without proper setup:
  → Group block manually tracked in Excel → rooms double-booked
  → Corporate rate negotiated verbally → staff applies wrong rate
  → Group invoice: 45 rooms × 3 nights → manually calculated → errors
  → Cut-off date missed → unreleased rooms lost from inventory

With proper setup:
  → Group block: rooms reserved, cut-off auto-tracked
  → Corporate account: rate auto-applies when company books
  → Master invoice: auto-generated, itemized, sent to company
  → Cut-off: unreleased rooms auto-return to inventory
```

---

## Existing Systems — Problems

| System | Problem |
|--------|---------|
| Oracle OPERA | Group module powerful but extremely complex. 50+ screens. Small hotel never uses it fully. |
| Mews | Corporate accounts supported, but group blocks limited. Rooming list import not clean. |
| Cloudbeds | Very basic group handling. No corporate account management. No auto-invoice. |
| All systems | No cut-off auto-release. Someone must manually track and release rooms. Always forgotten. |

---

## PART 1: GROUP SETUP

### 1. What is a Group Block?
```
GROUP BLOCK = Pre-reserved pool of rooms for a specific event/arrival.

Example:
  "Silva Wedding — Dec 20-22"
  → Block 30 rooms (20 × Deluxe + 10 × Standard)
  → Guests book from this block (not from general inventory)
  → Rooms not in block remain available to other guests
  → Cut-off: Dec 13 (7 days before) — unreleased rooms return to inventory

NOT the same as individual bookings:
  Individual: 1 room, 1 guest, confirmed immediately
  Group block: Pool of rooms, guests fill slowly over days/weeks
```

### 2. Group Default Config
```
GROUP DEFAULTS SETUP (hotel sets once — applies to all groups):

  Default cut-off:          7 days before arrival
  Default deposit %:        30% of total estimated value
  Deposit due:              At block confirmation
  Balance due:              7 days before arrival (at cut-off)
  Min rooms for group rate: 10 rooms
  Rooming list due:         3 days before arrival
  Master bill:              Yes (one invoice for organizer)
  Auto-release at cut-off:  Yes (unreleased rooms → general inventory)
  Auto-alert before cutoff: 3 days warning to organizer + FD Manager
```

### 3. Group Block Creation
```
GROUP BLOCK PROFILE:
  Group Name:       Silva Wedding
  Group Type:       WEDDING / TOUR / CORPORATE_GROUP / CONFERENCE / OTHER
  Organizer:        Nimal Silva
  Organizer Phone:  077-XXXXXXX
  Organizer Email:  nimal@email.com
  Company:          (optional — if corporate group)

  Event Dates:
    Check-in:       Dec 20
    Check-out:      Dec 22
    Nights:         2

  Room Block:
    Deluxe Double:  20 rooms
    Standard:       10 rooms
    Total:          30 rooms

  Rate:
    Deluxe:         LKR 18,000/night (negotiated — vs rack LKR 22,000)
    Standard:       LKR 12,000/night (negotiated — vs rack LKR 15,000)

  Cut-off Date:     Dec 13 (auto-calculated: 7 days before)
  Deposit:          30% = LKR 1,260,000 (auto-calculated)
  Deposit Due:      At signing

  Special Requests:
    → Complimentary honeymoon suite for bride & groom
    → Welcome drinks on arrival
    → Bulk check-in: Dec 20, 2 PM

  Status:           TENTATIVE → CONFIRMED → PARTIAL → COMPLETE
```

### 4. Rooming List
```
ROOMING LIST = List of individual guests filling the group block

  Guest Name       Room Type    Check-in   Check-out  Special
  ──────────────────────────────────────────────────────────
  Kamal Perera     Deluxe       Dec 20     Dec 22     High floor
  Saman Fernando   Deluxe       Dec 20     Dec 22     -
  Priya Jayasena   Standard     Dec 21     Dec 22     Extra bed
  ...

Import options:
  → Upload Excel/CSV (system maps columns)
  → Manual entry one by one
  → Guest self-registration link (organizer shares → guests fill own details)

When guest added to rooming list:
  → Individual booking auto-created from group block
  → Room deducted from block allocation
  → Guest gets confirmation email/WhatsApp
```

### 5. Cut-off Logic
```
CUT-OFF DATE: Dec 13

What happens at cut-off:
  Block had:    30 rooms reserved
  Filled:       22 rooms (rooming list complete for 22)
  Unreleased:    8 rooms

  At cut-off:
  → 8 unreleased rooms → auto-return to general inventory
  → Available for regular bookings immediately
  → Organizer gets notification: "8 rooms released from your block"
  → If organizer wants more rooms after cut-off → subject to availability + rack rate

  System alert 3 days before cut-off:
  → To organizer: "8 rooms still unreleased — please confirm or release"
  → To FD Manager: "Silva Wedding: 8 rooms at cut-off risk"
```

### 6. Group Master Invoice
```
GROUP MASTER INVOICE:
  One invoice for entire group → sent to organizer

  The Grand Colombo
  Invoice #: INV-2025-00445
  To: Nimal Silva / Silva Wedding

  ROOM CHARGES:
  20 × Deluxe (Dec 20-21):   LKR 360,000
  20 × Deluxe (Dec 21-22):   LKR 360,000
  10 × Standard (Dec 20-21): LKR 120,000
  10 × Standard (Dec 21-22): LKR 120,000
  ─────────────────────────────────────────
  Room Subtotal:              LKR 960,000
  Service Charge (10%):       LKR  96,000
  VAT (18%):                  LKR 190,080
  TDL (1%):                   LKR   9,600
  ─────────────────────────────────────────
  TOTAL:                    LKR 1,255,680

  Less: Deposit Paid:       LKR  (376,704)
  ─────────────────────────────────────────
  BALANCE DUE:              LKR   878,976

  OR itemized per room (hotel configures):
    Each room → separate line with guest name
```

---

## PART 2: CORPORATE SETUP

### 1. What is a Corporate Account?
```
CORPORATE ACCOUNT = A registered company that books rooms regularly.

Benefits:
  → Negotiated rate (lower than rack)
  → Direct billing (company pays, not individual guest)
  → Monthly invoice (not per-booking payment)
  → No deposit required per booking
  → Streamlined check-in (company details pre-filled)

Examples:
  → MAS Holdings (staff travel to Colombo frequently)
  → John Keells Holdings (executive travel)
  → Embassy of Germany (diplomat stays)
```

### 2. Corporate Account Profile
```
CORPORATE ACCOUNT:
  Company Name:       MAS Holdings (Pvt) Ltd
  Account Code:       CORP-001
  Industry:           Apparel / Manufacturing
  Contact Person:     Chamari Wijesinghe (Travel Manager)
  Contact Phone:      011-XXXXXXX
  Contact Email:      travel@mas.lk
  Company VAT No:     VAT/LKR/XXXXXXXX   (for invoice ITC)
  Company Address:    [for invoice]
  Credit Limit:       LKR 500,000
  Payment Terms:      Net 30
  Status:             ACTIVE

  Negotiated Rates:
    Standard Room:    LKR 12,000/night  (rack: LKR 15,000)
    Deluxe Room:      LKR 16,000/night  (rack: LKR 22,000)
    Rate valid:       Jan 1 – Dec 31, 2025
    Rate plan type:   CORPORATE (private — not shown to public)

  Billing Arrangement:
    Who pays:         Company (master bill)
    Invoice:          Monthly consolidated
    Delivery:         Email to chamari@mas.lk + accounts@mas.lk
    Payment method:   Bank transfer
```

### 3. Corporate Rate — How It Works
```
CORPORATE RATE FLOW:

  MAS Holdings staff arrives → gives company name at front desk
  → Staff selects "MAS Holdings" corporate account
  → Rate auto-applies: Deluxe at LKR 16,000 (not rack LKR 22,000)
  → Check-in proceeds
  → At checkout: "Charge to company account" (no payment from guest)
  → Folio linked to MAS Holdings city ledger
  → At month end: consolidated invoice generated → sent to company
```

### 4. City Ledger / Corporate Credit
```
CITY LEDGER = Running tab for corporate accounts

  MAS Holdings — October 2025:
    Oct 3:   Chamari Wijesinghe — 2 nights Deluxe   LKR 32,000
    Oct 8:   Rohan Perera — 1 night Standard         LKR 12,000
    Oct 15:  Pradeep Silva — 3 nights Deluxe         LKR 48,000
    Oct 22:  Chamari Wijesinghe — 1 night Standard   LKR 12,000
    ──────────────────────────────────────────────────────────
    October Total:                                   LKR 104,000
    Service Charge (10%):                            LKR  10,400
    VAT (18%):                                       LKR  20,592
    TDL (1%):                                        LKR   1,040
    ──────────────────────────────────────────────────────────
    INVOICE TOTAL:                                   LKR 136,032

  Invoice auto-generated Nov 1 → emailed to company
  Payment due: Dec 1 (Net 30)

  Credit limit tracking:
    Limit: LKR 500,000
    Current outstanding: LKR 136,032
    Available: LKR 363,968

    If new booking would exceed credit limit → FD Manager alert
```

### 5. Corporate Invoice Auto-Schedule
```
AUTO-INVOICE SETUP (per corporate account):
  Generate:       Monthly / Weekly / Per stay
  Generate on:    1st of each month (for previous month)
  Send to:        chamari@mas.lk, accounts@mas.lk
  Format:         PDF (itemized — each stay per line)
  Payment terms:  Net 30

  Auto-reminder if unpaid:
    Day 25: "Invoice due in 5 days"
    Day 30: "Invoice due today"
    Day 35: "Invoice overdue — 5 days"
    Day 45: "Invoice overdue — flag to FD Manager"
```

---

## Data Model

```
-- GROUP SETUP --

GroupConfig
  hotel_id
  default_cutoff_days           int (7)
  default_deposit_pct           decimal (30)
  min_rooms_for_group_rate      int (10)
  rooming_list_due_days         int (3)
  auto_release_at_cutoff        bool
  cutoff_alert_days_before      int (3)

GroupBlock
  id, hotel_id
  name                          "Silva Wedding"
  group_type                    WEDDING / TOUR / CORPORATE_GROUP /
                                CONFERENCE / OTHER
  organizer_name
  organizer_phone
  organizer_email
  company_id                    nullable FK → CorporateAccount
  checkin_date, checkout_date
  cutoff_date
  deposit_pct                   decimal
  deposit_amount                decimal (auto-calculated)
  deposit_paid                  decimal
  deposit_paid_at               timestamp nullable
  rooming_list_due_date         date
  master_bill                   bool
  invoice_format                CONSOLIDATED / ITEMIZED_PER_ROOM
  status                        TENTATIVE / CONFIRMED / PARTIAL /
                                COMPLETE / CANCELLED
  special_requests              text nullable

GroupBlockAllocation
  id, group_block_id
  room_type_id
  allocated_count               int
  negotiated_rate               decimal
  filled_count                  int (auto-calculated)
  released_count                int (auto at cutoff)

GroupBooking
  id, group_block_id
  booking_id                    FK → Booking (auto-created)
  guest_name, phone, email
  room_type_id
  special_requests              nullable
  added_via                     MANUAL / IMPORT / SELF_REGISTRATION

-- CORPORATE SETUP --

CorporateAccount
  id, hotel_id
  company_name                  "MAS Holdings (Pvt) Ltd"
  account_code                  "CORP-001"
  industry                      nullable
  contact_name
  contact_phone
  contact_email
  company_vat_number            nullable
  company_address               text
  credit_limit                  decimal
  current_outstanding           decimal (auto-tracked)
  payment_terms_days            int (30)
  invoice_frequency             MONTHLY / WEEKLY / PER_STAY
  invoice_day_of_month          int nullable (1)
  invoice_emails                JSON [email list]
  invoice_format                CONSOLIDATED / ITEMIZED
  is_active                     bool

CorporateRatePlan
  corporate_account_id
  rate_plan_id                  FK → RatePlan (visibility = CORPORATE)
  valid_from                    date
  valid_until                   date

CorporateInvoice
  id, hotel_id, corporate_account_id
  period_from, period_to        date
  invoice_number
  subtotal, sc_amount, vat_amount, tdl_amount, total_amount
  amount_paid                   decimal
  status                        DRAFT / SENT / PAID / OVERDUE
  generated_at                  timestamp
  sent_at                       timestamp nullable
  due_date                      date
  paid_at                       timestamp nullable

CityLedgerEntry
  id, corporate_account_id
  booking_id                    FK → Booking
  folio_id                      FK → Folio
  amount                        decimal
  invoice_id                    nullable FK → CorporateInvoice
  posted_at                     timestamp
```

---

## Key Relationships

```
Hotel → GroupConfig (one)
Hotel → GroupBlock (many)
GroupBlock → GroupBlockAllocation (one per room type)
GroupBlock → GroupBooking → Booking (individual bookings auto-created)
GroupBlock → CorporateAccount (optional — if corporate group)

Hotel → CorporateAccount (many)
CorporateAccount → CorporateRatePlan → RatePlan
CorporateAccount → CityLedgerEntry (all stays accumulate)
CorporateAccount → CorporateInvoice (monthly billing)
Booking → CityLedgerEntry (when charged to corporate account)
```

---

## V1 vs Phase Split

| Feature | V1 | Phase 2 | Phase 3 |
|---------|:--:|:-------:|:-------:|
| Group block creation + room allocation | ✅ | | |
| Group cut-off date + auto-release | ✅ | | |
| Cut-off alert (3 days before) | ✅ | | |
| Rooming list (manual entry + CSV import) | ✅ | | |
| Group negotiated rate | ✅ | | |
| Group deposit tracking | ✅ | | |
| Group master invoice (consolidated) | ✅ | | |
| Corporate account profile | ✅ | | |
| Corporate negotiated rate plan | ✅ | | |
| City ledger (charge to company) | ✅ | | |
| Monthly corporate invoice auto-generate | ✅ | | |
| Credit limit tracking + alert | ✅ | | |
| Corporate invoice payment reminder | ✅ | | |
| Guest self-registration link (rooming list) | | ✅ | |
| Corporate portal login (online booking) | | ✅ | |
| Itemized per-room group invoice | | ✅ | |
| Input tax credit tracking (corporate VAT) | | ✅ | |
| Group analytics (conversion, fill rate) | | ✅ | |
| Contract management (upload + expiry alert) | | ✅ | |
| RFP (Request for Proposal) workflow | | | ✅ |
