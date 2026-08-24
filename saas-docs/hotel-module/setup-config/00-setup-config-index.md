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

## 13 Areas of Setup & Configuration

| # | Area | Status |
|---|------|--------|
| 1 | Property Setup | 🔲 To discuss |
| 2 | Room Setup | 🔲 To discuss |
| 3 | Rate Setup | 🔲 To discuss |
| 4 | Policy Setup | 🔲 To discuss |
| 5 | Payment Setup | 🔲 To discuss |
| 6 | Housekeeping Setup | 🔲 To discuss |
| 7 | Staff & Roles Setup | 🔲 To discuss |
| 8 | Channel Setup (OTA) | 🔲 To discuss |
| 9 | Folio & Billing Setup | 🔲 To discuss |
| 10 | Notification Setup | 🔲 To discuss |
| 11 | Guest Portal Setup | 🔲 To discuss |
| 12 | Night Audit Setup | 🔲 To discuss |
| 13 | Integrations Setup | 🔲 To discuss |

---

## High Level Overview (Before Deep Dive)

### 1. Property Setup
```
├── Basic info (name, address, phone, email, website)
├── Property type (Guesthouse / Boutique / Business Hotel / Resort / Luxury)
├── Star rating
├── Logo & branding
├── Language & timezone
└── Currency
```

### 2. Room Setup
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
├── Rate plans (name, price, meal inclusion, visibility, linked policy)
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
├── Tax configuration (GST per charge type)
└── Invoice setup (GSTIN, format, template)
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

### 9. Folio & Billing Setup
```
├── Folio number format
├── Night audit time
├── Auto-post rules
├── Cross-module charge routing
└── Folio display preferences
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

### 12. Night Audit Setup
```
├── Audit run time
├── Reports to auto-generate
├── Who receives each report
└── No-show auto-charge rules
```

### 13. Integrations Setup
```
├── Payment gateway credentials
├── SMS gateway credentials
├── Email provider
├── WhatsApp Business API
└── Accounting software (Phase 2)
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

---

## Files in This Folder
Each section gets its own detailed file after discussion:
- `01-property-setup.md`
- `02-room-setup.md`
- `03-rate-setup.md`
- `04-policy-setup.md`
- `05-payment-setup.md`
- `06-housekeeping-setup.md`
- `07-staff-roles-setup.md`
- `08-channel-setup.md`
- `09-folio-billing-setup.md`
- `10-notification-setup.md`
- `11-guest-portal-setup.md`
- `12-night-audit-setup.md`
- `13-integrations-setup.md`
