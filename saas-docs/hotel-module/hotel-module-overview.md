# Hotel Module — Full Overview
> Status: Draft | Last Updated: 2026-08-21
> Heart of Hotel Module: Room Booking / Reservation

---

## Big Modules

| # | Module | Core Purpose |
|---|--------|-------------|
| 1 | Room & Property Management | Rooms setup, types, amenities, status |
| 2 | Reservations / Booking | Every type of booking management |
| 3 | Front Desk | Check-in, check-out, daily operations |
| 4 | Guest Management | Guest profiles, preferences, history |
| 5 | Guest Folio & Billing | Charge accumulation, payment, invoice |
| 6 | Housekeeping | Cleaning, room status, task management |
| 7 | Maintenance | Repairs, work orders, preventive care |
| 8 | Rate & Revenue Management | Pricing, yield, demand-based rates |
| 9 | Channel Management | OTA sync, Booking.com, Airbnb, GDS |
| 10 | Night Audit | End-of-day process, auto charge posting |
| 11 | Group & Corporate | Group bookings, corporate accounts |
| 12 | Meeting & Banquet | Conference rooms, event spaces |
| 13 | Guest Communication | Pre/during/post stay messaging |
| 14 | Loyalty Program | Points, tiers, member benefits |
| 15 | Analytics & Reports | Occupancy, revenue, KPIs, forecasting |
| 16 | Digital Guest Experience | Mobile check-in, digital key, in-room app |
| 17 | Staff Management | Roles, shifts, performance |
| 18 | Accounting Integration | Financial posting, tax, accounting software |

---

## Sub-Features Per Module

### 1. Room & Property Management
- Room types & categories
- Floor / building structure
- Room amenities tracking
- Room status (Available / Occupied / Dirty / Out-of-Order / Inspected)
- Room photos & descriptions
- Connecting rooms, accessible rooms
- Out-of-order management

---

### 2. Reservations / Booking ← Heart
- Individual booking (walk-in, phone, online, staff-created)
- Group booking & room block
- Corporate booking
- OTA booking (auto-import)
- Booking modification & cancellation
- Overbooking prevention & waitlist
- Deposit & advance payment policies
- Package / bundle deals
- Early bird & last-minute rates
- Cancellation policy per booking type
- No-show handling

---

### 3. Front Desk
- Check-in (staff, kiosk, mobile)
- Check-out (staff, express, mobile)
- Room assignment & upgrade
- Walk-in handling
- Early check-in / late check-out
- Room change during stay
- Guest document verification
- Key card management

---

### 4. Guest Management
- Guest profile (contact, ID, preferences)
- Stay history
- Room preferences (floor, view, bed type)
- Dietary restrictions & allergies
- VIP / Blacklist flags
- Loyalty tier & points
- Special occasions (birthday, anniversary)
- Lifetime value tracking
- Guest segmentation

---

### 5. Guest Folio & Billing
- Room charge auto-posting (nightly)
- Charges from Restaurant, Bar, Spa, Events
- Manual charge entry
- Partial payments during stay
- Folio split (multiple guests, company split)
- Invoice generation
- Multi-currency support
- Tax calculation & compliance
- Credit / debit adjustments
- Final bill & settlement

---

### 6. Housekeeping
- Room cleaning task assignment
- Priority-based task queue
- Mobile app for housekeepers
- Room status real-time update
- Inspection workflow (supervisor sign-off)
- Linen & supply tracking
- Lost & found management
- Turnover time tracking

---

### 7. Maintenance
- Maintenance request creation
- Work order assignment
- Priority levels (urgent / normal / scheduled)
- Preventive maintenance calendar
- Equipment tracking
- Cost per maintenance item
- Out-of-order room linking

---

### 8. Rate & Revenue Management
- Base rate setup
- Seasonal rates
- Weekend / weekday rates
- Corporate rates
- Early bird & last-minute rates
- Length-of-stay (LOS) restrictions
- Yield management (auto price by occupancy %)
- Dynamic / AI-based pricing *(Layer 4)*
- Competitor rate monitoring *(Layer 4)*
- RevPAR, ADR, GOPPAR tracking

---

### 9. Channel Management
- OTA integration (Booking.com, Airbnb, Expedia, Agoda)
- Real-time availability sync
- Rate parity management
- Commission tracking
- GDS connectivity *(Layer 3/4)*
- Direct booking engine (hotel's own website)
- Channel performance analytics

---

### 10. Night Audit
- End-of-day automated process
- Nightly room charge auto-posting to all folios
- Day closing & rollover
- Daily flash report generation
- Balance reconciliation
- No-show marking & charge

---

### 11. Group & Corporate
- Group block creation
- Subblock management
- Group rate & billing
- Rooming list generation
- Master account / company billing
- Corporate account management
- Contract rate management

---

### 12. Meeting & Banquet *(Layer 2+)*
- Meeting room / function space setup
- Space availability & booking
- Event timeline management
- AV equipment booking
- Catering linkage (via Restaurant Module)
- Banquet billing

---

### 13. Guest Communication
- Pre-arrival confirmation & welcome email
- Check-in reminder (day before)
- Upsell offers (upgrade, add-ons)
- Special occasion messages
- In-stay service requests
- Post-checkout thank you & survey
- Review request (TripAdvisor, Google)
- Custom message templates

---

### 14. Loyalty Program *(Layer 2+)*
- Points earning per stay / spend
- Points redemption
- Tier levels (Silver, Gold, Platinum)
- Member-exclusive rates
- Bonus point promotions
- Partner program integration

---

### 15. Analytics & Reports
- Live dashboard (occupancy, arrivals, revenue)
- Daily flash report
- Occupancy rate & ADR report
- RevPAR analysis
- Revenue by source / channel
- Housekeeping productivity
- Guest satisfaction scores
- Booking pace (vs last year)
- Forecasting reports
- Custom report builder *(Layer 3+)*

---

### 16. Digital Guest Experience *(Layer 2+)*
- Customer-facing web / app
- Online booking
- Mobile check-in & check-out
- Digital room key / keyless entry *(Layer 4)*
- In-room QR → service requests, room service
- Real-time bill view
- In-app chat with front desk
- Smart room controls *(Layer 4)*

---

### 17. Staff Management
- Staff roles & permissions
- Shift scheduling
- Department-wise access
- Task assignment tracking
- Performance reports

---

### 18. Accounting Integration
- Revenue posting to accounting software
- Tax reports
- Accounts receivable management
- Integration with QuickBooks, Xero *(Layer 3+)*

---

## Layer Mapping

```
Layer 1 — Simple (Small Guesthouse):
  Room Management, Basic Reservations, Front Desk,
  Basic Folio, Housekeeping, Night Audit, Basic Reports

Layer 2 — Standard (Mid Hotel):
  + Guest Management, Rate Plans, OTA Channels,
    Guest Communication, Loyalty, Meeting Rooms,
    Digital Experience (basic)

Layer 3 — Advanced (Large Hotel):
  + Revenue Management, Group & Corporate,
    Channel Manager (GDS), Advanced Analytics,
    Maintenance, Accounting Integration

Layer 4 — Luxury / Enterprise (5-Star Resort):
  + AI Pricing, Digital Key, Smart Room Controls,
    Competitor Monitoring, Advanced Personalization,
    Custom Workflows, API Access
```

---

## Notes
- Room Booking / Reservation = Heart of Hotel Module. Everything else depends on it.
- Guest Folio = Key integration point between Hotel and other modules (Restaurant, Bar, Spa, Events)
- All modules follow the "Simple by Default, Powerful when needed" design philosophy
- See: `design-philosophy.md` for layer details
