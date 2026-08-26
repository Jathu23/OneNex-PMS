# Hotel Module — Setup & Configuration
> Status: In Progress | Discussing one by one
> This is the foundation — Setup defines rules, Operations follow those rules

---

## Why Setup & Configuration is Critical

```
Setup defines the RULES.
Operations follow those RULES automatically.

No setup   = No rules   = System doesn't know how to operate
Wrong setup = Wrong rules = Billing errors, policy violations, bad guest experience
Right setup = Right rules = System runs itself
```

---

## 16 Areas of Setup & Configuration

| # | Area | Status |
|---|------|--------|
| 1 | Property Setup | ✅ Done |
| 2 | Room Setup | ✅ Done |
| 3 | Rate Setup | ✅ Done (unified in 03-rate-plan-setup.md) |
| 4 | Policy Setup | ✅ Done (unified in 03-rate-plan-setup.md) |
| 5 | Payment Setup | ✅ Done (unified in 03-rate-plan-setup.md) |
| 6 | Housekeeping Setup | ✅ Done |
| 7 | Staff & Roles Setup | ✅ Done |
| 8 | Channel Setup (OTA) | ✅ Done (unified in 03-rate-plan-setup.md) |
| 9 | Folio, Billing & Night Audit Setup | ✅ Done |
| 10 | Notification Setup | ✅ Done |
| 11 | Guest Portal Setup | ✅ Done |
| 12 | Integrations Setup | 🔲 To discuss |
| 13 | Maintenance Setup | ✅ Done (Phase 2 — V1 skip) |
| 14 | Tax Configuration | ✅ Done (Sri Lanka — VAT + SC + TDL) |
| 15 | Group & Corporate Setup | ✅ Done |
| 16 | Report & Analytics Setup | 🔲 To discuss |

---

## Key Insight: Areas 3, 4, 5, 8 are One Unified Concept

```
Rate Setup + Policy Setup + Payment Setup + Channel Setup
= RATE PLAN

A Rate Plan defines the complete bookable behaviour of a room:
  - What price?         → Rate Setup
  - What rules?         → Policy Setup (cancellation, check-in time)
  - How to collect?     → Payment Setup (deposit, pay now, guarantee)
  - Where to sell?      → Channel Setup (OTA, direct, walk-in)

One Rate Plan = one complete package a guest can book.
These 4 areas must be discussed together.
```

---

## High Level Overview

### 1. Property Setup
```
├── Basic info (name, address, phone, email, website)
├── Property type (Guesthouse / Boutique / Business Hotel / Resort / Luxury)
├── Star rating
├── Logo & branding
├── Language & timezone
└── Currency
```

### 2. Room Setup ✅
```
├── Room types (name, description, max capacity, sq ft)
├── Actual rooms (room number, floor, type assignment)
├── Floor & building structure
├── Room amenities per type (AC, TV, WiFi, balcony...)
├── Room photos per type
├── Connecting rooms mapping
├── Accessible rooms (wheelchair, special needs)
└── Out-of-order default reasons list
```

### 3. Rate Setup
```
├── Base rate per room type
├── Rate plans (name, price, meal inclusion, visibility)
├── Date-based overrides (seasonal, weekend, holiday)
├── OTA-specific rates (per channel commission %)
└── Length of stay (LOS) restrictions
```

### 4. Policy Setup
```
├── Check-in / Check-out times + Early/Late rules
├── Cancellation policies (per rate plan)
├── No-show policy
├── Child policy
├── Pet policy
└── Overbooking policy
```

### 5. Payment Setup
```
├── Accepted payment methods
├── Deposit rules per rate plan
├── Credit card guarantee rules
├── Refund policy
└── Invoice setup (format, template)
```

### 6. Housekeeping Setup
```
├── Shift timings
├── Floor/zone assignment per housekeeper
├── Inspection required? (Yes/No)
├── Standard checklist per room type
├── Default amenity quantity per room type
├── Mini bar items list
└── Turnover time target
```

### 7. Staff & Roles Setup
```
├── Staff profiles
├── Role definitions (permissions per role)
├── Department structure
├── Shift assignment
└── PIN setup
```

### 8. Channel Setup (OTA)
```
├── Which OTAs to connect
├── API credentials per OTA
├── Commission % per channel
├── Availability buffer
└── Stop sell rules
```

### 9. Folio, Billing & Night Audit Setup
```
├── Folio number format
├── Night audit run time
├── Auto-post rules
├── Cross-module charge routing
├── Folio display preferences
├── No-show auto-charge rules
└── Reports to auto-generate at audit
```

### 10. Notification Setup
```
├── Which notifications enabled
├── Channel per notification (Email/SMS/WhatsApp)
├── Template per notification
├── Staff alert rules
└── Report delivery config
```

### 11. Guest Portal Setup
```
├── Enable/Disable online booking
├── Which rate plans visible to public
├── Booking widget embed
├── Required guest fields
└── Loyalty program enable/disable
```

### 12. Integrations Setup
```
├── Payment gateway credentials
├── SMS gateway credentials
├── Email provider
├── WhatsApp Business API
└── Accounting software (Phase 2)
```

### 13. Maintenance Setup
```
├── Maintenance request categories (AC / Plumbing / Electrical...)
├── Priority levels (Low / Medium / High / Emergency)
├── Default SLA per priority (e.g., Emergency = 30 mins)
├── Maintenance team assignment
└── OOO auto-trigger rules (room auto Out-of-Order when open ticket)
```

### 14. Tax Configuration
```
├── GST rates per charge type (room, food, beverage, service)
├── Tax-inclusive vs tax-exclusive pricing
├── GSTIN setup per property
├── HSN/SAC codes per charge type
└── Tax exemption rules (corporate, government)
```

### 15. Group & Corporate Setup
```
├── Group block default settings (cut-off days, deposit %)
├── Corporate account profiles
├── Contract rate templates
├── Billing arrangement defaults (master bill, split bill)
└── Auto-invoice schedule (monthly/weekly)
```

### 16. Report & Analytics Setup
```
├── Which reports auto-generate (daily/weekly/monthly)
├── Who receives each report (email delivery)
├── Dashboard KPI selection (what shows on home screen)
├── Data retention period
└── Export format preference (PDF / Excel / CSV)
```

---

## How Setup Drives Operations (Examples)

| Setup Rule | Operation Result |
|-----------|-----------------|
| Check-out: 11 AM, Late: ₹500/hr | Guest requests 1 PM late → System auto-calculates ₹1,000 charge |
| Night audit: 11:59 PM | Every night → Room charges auto-post to all open folios |
| Booking.com: 10% commission | New booking from Booking.com → OTA rate auto-applied |
| Cancellation: Free until 48 hrs | Guest cancels 72 hrs before → Full refund auto-triggered |
| Room 412: No-smoking, high floor | Check-in → System flags if wrong room type assigned |
| Emergency SLA: 30 mins | Maintenance ticket raised → Alert fires if not responded in 30 mins |
| GST: 18% on room above ₹7,500 | Booking at ₹8,000 → GST auto-calculated and added to folio |
| Group cut-off: 7 days | 7 days before arrival → Unreleased rooms auto-return to inventory |

---

## Files in This Folder

Each section gets its own detailed file after discussion:
- `01-property-setup.md` ✅
- `02-room-setup.md` ✅
- `03-rate-plan-setup.md` ✅  ← Covers Areas 3 + 4 + 5 + 8 (Rate + Policy + Payment + Channel)
- `06-housekeeping-setup.md`
- `07-staff-roles-setup.md`
- `09-folio-billing-night-audit-setup.md`
- `10-notification-setup.md`
- `11-guest-portal-setup.md`
- `12-integrations-setup.md`
- `13-maintenance-setup.md`
- `14-tax-configuration.md`
- `15-group-corporate-setup.md`
- `16-report-analytics-setup.md`
