# Stays Module — Master Plan
> Status: Planning | Last Updated: 2026-09-01
> Pattern: Two-Level Modularity — Core + Add-ons
> Philosophy: Simple by Default. Powerful when needed.

---

## What Is Stays?

Stays is the hotel/accommodation operation within OneNex.
When a Business owner enables "Stays", they get a full Property Management System (PMS).

```
Owner
  └── Business (Grand Hotel Colombo)
        └── Stays (Operation)
              ├── Core      ← always on, works immediately
              └── Add-ons   ← enable what you need
```

Stays can run standalone or alongside Dining, Bar, Wellness, Events.
When Stays + Dining both enabled → "Charge to Room" activates automatically (Core-to-Core integration via Folio).

---

## Competitor Research Summary

What we studied before defining our Core/Add-on split:

| Competitor | Approach | Key Insight |
|---|---|---|
| **Oracle OPERA** | Enterprise, all-inclusive | Everything bundled — too heavy for small/mid hotels |
| **Mews** | Lean core + paid add-ons | Kiosk, advanced housekeeping, RMS all behind add-ons |
| **Cloudbeds** | All-in-one platform | Channel manager + booking engine treated as core |
| **Apaleo** | API-first, minimal core | Reservations, folios, billing = core. Everything else via marketplace |
| **Little Hotelier** | SMB all-in-one | PMS + Channel Manager + Booking Engine bundled as one simple product |

**What competitors get wrong:**
- OPERA: Too complex for small/mid hotels. Requires long onboarding.
- Mews/Cloudbeds: Pricing grows fast as you add modules. Confusing tier structure.
- Apaleo: Too developer-focused. Non-technical hoteliers struggle.
- Little Hotelier: Great for tiny hotels, but no growth path as business scales.

**What we do better:**
- One clear Core that works immediately (no config required on day 1).
- Add-ons are business-level — owner enables per operation, not per module subscription.
- Pricing = per operation per business. No surprise per-module billing.
- Same platform scales from 5-room guesthouse to 500-room resort.

---

## Core Definition Rule

> **Can a hotel NOT operate without this feature?**
> YES = Core. NO = Add-on.

Core = absolute minimum to receive a guest, assign a room, charge them, and check them out.

---

## CORE (Always On When Stays Is Enabled)

### C1 — Property & Room Setup
Everything needed to describe the physical property.

- Room types & categories (Standard, Deluxe, Suite, Villa...)
- Room inventory (individual room records: number, floor, building, type)
- Room amenities per type (AC, TV, mini-bar, balcony...)
- Connecting rooms & accessible room flags
- Room photos & descriptions (for booking display)
- Out-of-order (OOO) room management — mark room unavailable with reason + date range

### C2 — Reservations (Core)
Individual booking management — the heart of Stays.

- Booking types: Walk-in, Phone, Staff-created, Direct online
- Full booking lifecycle: `Draft → Confirmed → Modified → Checked-in → Checked-out`
- Also handles: `Cancelled`, `No-show`
- Availability logic: `Available = Total Rooms − (Confirmed + OOO + Manual Blocks)`
- Cancellation policies: Free / Partial penalty / Non-refundable (per booking or per rate)
- Deposit & payment policies: Pay Now / Deposit / Pay at Hotel / Card guarantee
- Overbooking: Basic prevention (soft lock at capacity, hard lock override by manager only)
- No-show marking + charge application at Night Audit

### C3 — Front Desk
Daily check-in and check-out operations.

- Check-in (staff-executed): guest verification, room assignment, key issuance
- Check-out: folio review, payment, receipt/invoice print
- Walk-in handling: same-day availability check + booking creation
- Room assignment: manual pick from available + inspected rooms only
- Room upgrade: one-click upgrade (availability check, folio transfer)
- Room change during stay: auto-deactivates old key, transfers folio to new room
- Early check-in / late check-out: with room readiness check
- Guest document capture: passport / NIC number, nationality
- Arrival & departure list view (daily dashboard)

### C4 — Housekeeping (Basic)
Room status management — minimum viable for any hotel.

- Room statuses: VD (Vacant Dirty), VC (Vacant Clean), OD (Occupied Dirty), IP (In Progress), INS (Inspected), OOO (Out of Order), DND (Do Not Disturb)
- **Rule:** Front desk can only assign INS (Inspected) rooms — never VC
- Manual task assignment: assign cleaning tasks to housekeepers
- Status update: housekeeper marks room status (simple web or mobile input)
- Supervisor inspection: approve/reject → room moves to INS status

### C5 — Guest Profile (Basic)
Essential guest data — enough for V1 operations.

- Name, contact (email, phone), nationality, ID number
- Stay history (auto-built from bookings)
- Basic preferences: bed type, floor, smoking/non-smoking
- VIP flag (manual) and Blacklist flag (with reason)
- Special occasions: birthday, anniversary (for front desk awareness)
- One unified guest profile across all Operations (Hotel + Restaurant + Spa = same guest)

### C6 — Guest Folio & Billing
Financial heart of Stays. Every charge flows through here.

- Folio auto-created at check-in
- Night Audit auto-posts room charge to every open folio every night
- Cross-module charges: Restaurant / Bar / Spa / Events → "Charge to Room" → instant folio entry
- Manual charge entry by staff (F&B, minibar, laundry, miscellaneous)
- Adjustments: credit/debit corrections with reason
- Partial payments during stay accepted
- Multi-tax support: VAT, Service Charge, GST — per rate plan
- Invoice generation: individual guest invoice, company invoice
- Checkout settlement: cash, card, bank transfer, company account
- Folio close at checkout

### C7 — Rate Plans (Basic)
Pricing setup — minimum to charge guests correctly.

- Base rate per room type
- Date-based rates: seasonal, peak/off-peak
- Weekday vs weekend rate distinction
- Meal plan inclusions: Room Only / Bed & Breakfast (BB) / Half Board (HB) / Full Board (FB) / All Inclusive (AI)
- Extra person charges
- Child policy per rate plan
- Rate override by staff (manager permission required)

### C8 — Night Audit
End-of-day automated process. Every hotel needs this.

- Auto-trigger at configured hotel closing time (e.g., 11:59 PM)
- Posts room charge to all open folios
- Marks confirmed no-shows, applies no-show charge if policy set
- Rolls the business date forward
- Generates daily flash report (occupancy, revenue, arrivals, departures)
- Balance reconciliation summary for manager review

### C9 — Basic Reports
Minimum reporting to run day-to-day operations.

- Daily Flash Report (auto, via Night Audit)
- Occupancy Report (by date range)
- Arrivals & Departures list
- Revenue Summary (room revenue by date)
- In-house guests list
- Housekeeping status report

---

## ADD-ONS (Enable When Needed)

### A1 — Channel Manager
Real-time sync with OTAs. Enables online distribution.

**Why Add-on:** Not every hotel sells via OTAs. Small guesthouses may take all bookings by phone.

- Real-time availability sync to all OTA channels (milliseconds, not batch)
- Rate sync: change once → all connected channels update
- OTA booking auto-import via webhooks
- V1 connections: Booking.com, Expedia, Agoda, Airbnb
- Commission tracking per channel
- Stop-sell per OTA per date
- Rate parity monitoring + alerts
- Phase 2: GDS (Sabre, Amadeus) connectivity
- Phase 2: Google Hotels integration (zero-commission direct bookings)

**Dependency:** None — standalone add-on.

### A2 — Direct Booking Engine
Hotel's own website booking. Captures commission-free direct bookings.

**Why Add-on:** Needs a hotel website to embed. Not all hotels have one at launch.

- White-label booking widget embeddable on hotel website
- Real-time availability display (same availability as PMS)
- Rate display with inclusions (BB, HB, packages)
- Guest self-booking → booking appears in PMS instantly
- Booking confirmation email auto-sent to guest
- Phase 2: Google Hotel Ads integration
- Phase 2: Promo codes / direct-only rates

**Dependency:** Recommended with Channel Manager (A1) for parity.

### A3 — Revenue Management
Dynamic pricing and revenue analytics. Moves beyond static rate plans.

**Why Add-on:** Small hotels run fine on fixed rates. Revenue management is for hotels actively optimizing occupancy & revenue.

- Yield management: auto price adjustment rules based on occupancy %
  - e.g., "If occupancy > 80% → raise rate by 15%"
- Length-of-stay (LOS) restrictions for peak periods
  - e.g., "Minimum 3 nights for New Year's week"
- Visual rate calendar with color-coded occupancy heatmap
- ADR, RevPAR, GOPPAR KPI dashboard with plain-language explanations
- Booking pace report (current vs same period last year)
- Phase 2: AI/dynamic pricing engine
- Phase 3: Competitor rate monitoring

**Dependency:** Benefits from Channel Manager (A1) for rate push.

### A4 — Group & Corporate
Group bookings and corporate account management.

**Why Add-on:** Only hotels that handle group reservations and corporate clients need this.

- Group room blocks: create block, assign subrooms, set cut-off date
- Rooming list management (upload or manual entry)
- Group rate + group billing master account
- One-click group check-in (pre-assign rooms, activate all at once)
- Corporate account management: contract rates, expiry alerts
- Corporate rate auto-apply when booking under company name
- Monthly auto-invoice to corporate accounts
- Phase 2: Agent/travel agent portal
- Phase 2: Multi-property corporate contract (one contract across all linked hotels)
- Phase 2: Employee self-booking portal for corporate clients

**Dependency:** None — standalone add-on.

### A5 — Housekeeping Pro
Advanced housekeeping operations beyond basic status management.

**Why Add-on:** Basic status management (C4) covers small hotels. Larger hotels need smart assignment and mobile workflow.

- Smart task assignment engine:
  - Priority ordering: VIP arrivals > Early check-ins > Standard rooms
  - Floor grouping: minimize housekeeper travel time
  - Workload balancing: equal distribution across available staff
- Mobile app for housekeepers: room checklist + photo upload
- Priority countdown alerts: notification if VIP room not started within X minutes
- Formal inspection checklist: approve/reject workflow with checklist items
- Lost & found: digital log with photo, room number, date, status
- Mini-bar tracking: scan or tap items → auto-post to guest folio
- Performance tracking per housekeeper (rooms cleaned, time per room)
- Linen & supply request from mobile
- Phase 3: Eco/green opt-out with loyalty points reward

**Dependency:** None — extends Core housekeeping (C4).

### A6 — Guest CRM
Full 360° guest intelligence and relationship management.

**Why Add-on:** Basic guest profile (C5) covers operations. CRM is for hotels investing in guest loyalty and personalization.

- 360° guest profile: aggregates data from all Operations (Hotel, Dining, Spa, Events)
- Auto VIP scoring: system calculates score based on stay frequency, spend, LTV — not manual tagging
- Preference learning: system observes behavior and updates preferences automatically
  - e.g., Guest always picks high floor → auto-flag as "prefers high floor"
- Guest segmentation: Business / Leisure / Budget / Luxury / Special Occasion
- Pre-arrival journey automation:
  - D-7: Welcome email + upgrade offer
  - D-3: Local recommendations email
  - D-1: Check-in reminder + directions
  - D+1 post checkout: Thank you + review request
- Complaint tracking: log complaints with room number → system prevents re-assigning same room to same guest
- Room blacklisting: guest-specific room blocks ("this guest complained about room 204's noise")
- GDPR compliance: guest data view/export/delete rights

**Dependency:** Benefits from all Operations enabled (richer profile data).

### A7 — Guest Communication
Automated messaging throughout the guest journey.

**Why Add-on:** Some hotels prefer direct phone calls. Automation is for properties with volume.

- Pre-arrival confirmation & welcome email (auto on booking confirmed)
- Check-in reminder (day before arrival)
- Upsell offers: room upgrade, packages, add-on services
- In-stay: housekeeping request, service requests via message
- Checkout reminder
- Post-checkout: thank you message + review request (TripAdvisor, Google)
- Custom message templates (hotel branded)
- Multi-channel: email + SMS + WhatsApp (WhatsApp Phase 2)
- Phase 2: AI-personalized message content per guest segment

**Dependency:** None — standalone. Enhanced with Guest CRM (A6) for personalization.

### A8 — Digital Guest Experience
Self-service check-in/out and digital key.

**Why Add-on:** Requires hardware (kiosk) or guest app. Not universal.

- Online pre-check-in: guest submits ID + preferences before arrival
- Mobile check-out: guest pays from app, walks out (express checkout)
- Digital room key: app-based unlock (Phase 2 — needs hardware integration)
- Kiosk check-in: self-service terminal at lobby (Phase 2)
- In-room QR code: scan for room service, housekeeping request, restaurant menu, folio view
- Real-time folio view on guest app: guest tracks charges live during stay
- In-app chat with front desk
- Phase 3: Smart room controls (IoT — lights, temperature, curtains)
- Phase 3: Biometric check-in

**Dependency:** None. Better with Guest Communication (A7) for message flow.

### A9 — Maintenance
Property maintenance and repair management.

**Why Add-on:** Small hotels handle maintenance informally. Structured maintenance is for larger properties.

- Maintenance request creation: any staff can raise from web/mobile
- Work order assignment to maintenance team
- Priority levels: Urgent (guest in room) / Normal / Scheduled
- OOO room auto-link: maintenance on a room automatically marks it OOO
- OOO auto-releases when maintenance marked complete
- Preventive maintenance calendar: scheduled tasks (monthly generator check, etc.)
- Equipment register: AC units, elevators, boilers with service history
- Maintenance cost tracking
- Phase 2: Vendor/contractor management portal

**Dependency:** None — standalone add-on.

### A10 — Advanced Analytics & BI
Deep revenue and operational intelligence beyond basic reports.

**Why Add-on:** Basic reports (C9) covers operational needs. Analytics is for revenue-driven management.

- Revenue analysis by source / channel / room type / rate plan
- Booking pace vs last year comparison
- Forecasting: occupancy + revenue projections (30/60/90 day)
- Housekeeping productivity analytics
- Guest satisfaction trend analysis
- Segment analysis: where are guests coming from?
- Phase 2: Custom report builder (drag-and-drop)
- Phase 2: Scheduled email delivery of reports
- Phase 3: AI-generated insights ("RevPAR dropped 12% this week because…")
- Phase 3: Competitor rate benchmarking

**Dependency:** None — extends Core reports (C9).

### A11 — Meeting & Banquet
Conference rooms and event space management.

**Why Add-on:** Only hotels with function spaces need this.

- Meeting room / function space setup: capacity, layout options, AV equipment list
- Space availability calendar
- Event booking: client, date, time, space, layout, pax count
- AV equipment booking with space
- Catering linkage: connect to Dining module for food & beverage orders
- Event timeline management: setup → event → breakdown
- Banquet billing: charge to client account or guest folio
- Phase 2: Online event inquiry/quote form
- Phase 2: BEO (Banquet Event Order) document generation

**Dependency:** Benefits from Dining module (Catering integration).

### A12 — Loyalty Program
Guest loyalty points and rewards system.

**Why Add-on:** Loyalty is a marketing investment, not an operational need.

- Points earning: per night stayed, per dollar spent (configurable rates)
- Points redemption: against room rate or ancillary charges
- Tier levels: Silver / Gold / Platinum (configurable names and thresholds)
- Member-exclusive rates: special rates visible only to logged-in loyalty members
- Bonus point promotions: double points weekends, anniversary bonuses
- Lifetime value tracking
- Phase 2: Cross-operation points (Hotel + Restaurant + Spa = one pool)
- Phase 3: Partner program (earn points at other businesses in Link network)
- Phase 3: TMC/corporate loyalty integration

**Dependency:** Requires Guest CRM (A6) for full profile linkage.

---

## V1 Scope — What Ships First

### Core — V1 Scope

| Component | V1 Must Have | Phase 2 |
|---|---|---|
| C1 Property & Room Setup | Full | — |
| C2 Reservations | Walk-in, Phone, Staff-created. Full lifecycle. Cancellation + No-show. | OTA-sourced (needs A1) |
| C3 Front Desk | Full | Mobile check-in (needs A8) |
| C4 Housekeeping Basic | Room statuses, manual assignment, basic inspection | Smart routing (A5) |
| C5 Guest Profile Basic | Name, contact, stay history, preferences, VIP/blacklist | 360° profile (A6) |
| C6 Folio & Billing | Auto-folio, night audit, cross-module charges, manual entry, checkout, GST invoice | Folio split, multi-currency |
| C7 Rate Plans | Base rates, seasonal, meal plans, extra person, child policy | Complex yield rules (A3) |
| C8 Night Audit | Full auto-process | — |
| C9 Basic Reports | Flash report, occupancy, arrivals, revenue summary | Advanced (A10) |

### Add-ons — V1 Scope

| Add-on | V1 | Phase 2 | Phase 3 |
|---|---|---|---|
| A1 Channel Manager | Booking.com + Expedia (webhook import + availability push) | Full rate sync, more OTAs, rate parity | GDS |
| A2 Booking Engine | Basic widget, real-time availability, confirmation email | Promo codes, Google Hotels | — |
| A3 Revenue Management | Basic yield rules, LOS restrictions, ADR/RevPAR dashboard | Rate calendar visual, booking pace | AI pricing, competitor monitoring |
| A4 Group & Corporate | Group blocks, rooming list, corporate rate manual apply, basic invoice | Agent portal, auto cut-off, multi-property contract | TMC/GDS |
| A5 Housekeeping Pro | Mobile app (basic), smart assignment, lost & found | Inspection checklist, minibar scan, performance tracking | Eco/green points |
| A6 Guest CRM | — (Phase 2) | 360° profile, auto VIP, pre-arrival journey, GDPR | AI personalization |
| A7 Guest Communication | Basic email templates (confirmation, pre-arrival, post-checkout) | WhatsApp, SMS, upsell automation, AI content | — |
| A8 Digital Guest Experience | Online pre-check-in, in-room QR, express checkout | Kiosk, digital key | Smart room controls, biometric |
| A9 Maintenance | Full (request, work order, OOO link, preventive calendar) | Vendor portal | — |
| A10 Advanced Analytics | Revenue by source/channel, booking pace, forecasting (basic) | Custom report builder, scheduled delivery | AI insights, competitor benchmarking |
| A11 Meeting & Banquet | Space setup, booking, basic billing | Online inquiry, BEO document, catering link | — |
| A12 Loyalty | — (Phase 3) | Cross-operation points | Partner program |

---

## Add-on Dependency Map

```
A1 Channel Manager     ──→ benefits A2, A3
A2 Booking Engine      ──→ standalone (recommends A1 for parity)
A3 Revenue Management  ──→ standalone (benefits from A1 for push)
A4 Group & Corporate   ──→ standalone
A5 Housekeeping Pro    ──→ extends C4 (Core)
A6 Guest CRM           ──→ standalone (richer with all Ops enabled)
A7 Guest Communication ──→ standalone (enhanced with A6)
A8 Digital Experience  ──→ standalone (enhanced with A7)
A9 Maintenance         ──→ standalone
A10 Advanced Analytics ──→ extends C9 (Core)
A11 Meeting & Banquet  ──→ standalone (enhanced with Dining Op)
A12 Loyalty            ──→ requires A6 (Guest CRM)
```

---

## Stays vs Other Operations — Integration Points

| From | To | How |
|---|---|---|
| Dining | Stays | "Charge to room" → auto folio entry (Core-to-Core, no add-on needed) |
| Bar | Stays | Same as above |
| Wellness/Spa | Stays | Same as above |
| Events | Stays | Same as above |
| Stays | Dining | Group/Corporate clients can have restaurant charges billed to master account |
| Stays | Meeting & Banquet (A11) | Function space bookings feed into Dining for catering |

All cross-operation charges flow through **Guest Folio** — the integration hub.

---

## Design Decisions Locked In

1. **Business type derived, never declared** — enabling Stays operation → business becomes a hotel automatically.
2. **Room assignment = INS (Inspected) rooms only** — VC rooms not available to front desk.
3. **Night Audit is Core** — every hotel needs end-of-day auto-posting. Not an add-on.
4. **Folio = Integration hub** — all cross-operation charges post here. No module-to-module direct billing.
5. **Guest Profile is shared across Operations** — same guest in Hotel, Restaurant, Spa = one profile.
6. **Channel Manager = Add-on** — not every hotel sells online. Small guesthouses take walk-in + phone only.
7. **Revenue Management = Add-on** — basic rate plans (C7) cover most hotels. Dynamic pricing is a power feature.
8. **Data model V1 = future-ready** — V1 ships simple, but schema supports Phase 2 without migration pain.

---

## What Was Already Deeply Designed (Carry Forward)

All existing research in `hotel-module/` is valid and maps to this structure.
When designing each section in detail, refer back to:

| Old Doc | Maps To |
|---|---|
| `room-booking/01-07` | C2 Reservations + A4 Group & Corporate |
| `front-desk/01` | C3 Front Desk + A8 Digital Guest Experience |
| `guest-management/01` | C5 Guest Profile (Basic) + A6 Guest CRM |
| `guest-folio/01-02` | C6 Guest Folio & Billing |
| `housekeeping/01` | C4 Housekeeping Basic + A5 Housekeeping Pro |
| `rate-revenue/01` | C7 Rate Plans Basic + A3 Revenue Management |
| `channel-management/01` | A1 Channel Manager + A2 Booking Engine |
| `group-corporate/01` | A4 Group & Corporate |
| `setup-config/` | Feeds into all Core + Add-on setup screens |

---

## Next Steps (In Order)

1. **C2 Reservations** — detailed entity design, booking lifecycle state machine, availability algorithm
2. **C6 Folio & Billing** — entity design, folio structure, night audit logic, cross-module posting
3. **C3 Front Desk** — UX flow design, room scoring engine detail
4. **C7 Rate Plans** — rate entity design, rate resolution logic (which rate wins?)
5. **C4 + A5 Housekeeping** — task assignment algorithm, mobile flow
6. **A1 Channel Manager** — OTA webhook design, availability sync architecture
7. **A3 Revenue Management** — yield rule engine design
8. All others in priority order per business need
