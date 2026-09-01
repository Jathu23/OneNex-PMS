# OneNex — Backend Module Boundaries

> Status: DRAFT — Needs personal review + team discussion before finalizing.
> This is the most critical architectural document. Issues here affect everything.

---

## Foundation Principle

```
One module = one business domain.
"What changes together, lives together."
Modules communicate ONLY through:
  → Shared.Contracts interfaces (sync)
  → Domain Events via MediatR (async)
Never directly. Never across database tables.
```

---

## Key Architecture Decisions

```
1. No Organization layer
   REMOVED:  users → organizations → businesses → operations
   CORRECT:  users → businesses → operations

2. Subscription = per Operation, per Business
   Owner gets ONE invoice (sum of all operation plans)

3. Folio lives inside Stays module V1
   Extract to Folio Hub (Phase 3) when cross-operation charging needed

4. Two levels of modularity
   Level 1: Owner enables Operations
   Level 2: Within each operation — Core (always on) + Add-ons (enable when needed)
```

---

## Full Module Map

```
┌───────────────────────────────────────────────────────────────────┐
│                     PLATFORM MODULES (Phase 2+)                    │
│          Analytics      Loyalty      Offers      Folio Hub         │
├───────────────────────────────────────────────────────────────────┤
│                     OPERATION MODULES                              │
│   Dining (P1)   Stays (P2)   Bar   Wellness   Events   Retail     │
├─────────────────────────┬─────────────────────────────────────────┤
│     SHARED SERVICES     │            CORE MODULES                  │
│   Payment               │   Identity     Business    Membership    │
│   Notification          │                                          │
└─────────────────────────┴─────────────────────────────────────────┘
          ↑ build first                     ↑ build first (Phase 0)
```

---

## JWT Flow — 3 Phases

```
PHASE 1: Login (Identity Module)
→ Verify email + password
→ JWT_1: { user_id, email }   ← no business context yet

PHASE 2: Business context (Business Module)
→ Subdomain slug → business_id  (staff at {slug}.onenex.com)
   OR owner selects from list   (owner at app.onenex.com)
→ JWT_2: { user_id, business_id }

PHASE 3: Permissions loaded (Membership Module)
→ Load role + permissions for this user in this business
→ JWT_3: { user_id, business_id, role, permissions[] }

ALL API CALLS USE JWT_3.
Every DB query: WHERE business_id = jwt.business_id
Cross-tenant data: impossible by design.
```

---

## Onboarding Flow — Module by Module

```
Owner sign up + login         → Identity Module
Create first business         → Business Module
Enable operation (dining...)  → Business Module
                                → JWT_2 + JWT_3 issued here
Operation setup (tables/menu) → Dining Module (or relevant operation)
Invite staff + set roles      → Membership Module
Owner dashboard               → Business Module
```

---

## CORE MODULES (Phase 0 — Build Before Everything)

---

### Module 1: Identity

**One line:** WHO you are. Globally. Nothing else.

**Responsibility:**
Manages the global identity of every human who touches OneNex.
Same registration flow for owner, staff, and customer.
Knows nothing about businesses, roles, or permissions.

**Owns:**
```
users:
  id                  PK
  email               unique globally
  phone               unique globally, nullable
  name
  password_hash
  email_verified      boolean
  created_at
```

**Features:**
```
├── Register (email + password + name + phone)
├── Email verification (OTP / magic link)
├── Login → JWT Phase 1 { user_id, email }
├── Password reset (forgot password flow)
├── Token refresh
└── Token revoke (logout)
```

**Does NOT know about:**
- Businesses, operations, roles, permissions
- Who is owner vs staff vs customer
- Anything business-specific

**Communicates via:**
- `INotificationService` → send verification email / reset email

---

### Module 2: Business

**One line:** WHAT businesses the owner runs and what they operate.

**Responsibility:**
Manages every business an owner creates, which operations that business runs,
how businesses are structured (branches), and how businesses are linked.
Also manages subscription per operation.

**Owns:**
```
businesses:
  id                  PK
  owner_id            → users.id
  name
  slug                unique globally (→ subdomain)
  address, city, country
  timezone, currency, logo
  status              (active / inactive)
  created_at

business_operations:
  id                  PK
  business_id         → businesses.id
  operation_type      (dining / stays / bar / wellness / events / retail)
  subscription_plan   (basic / standard / pro / enterprise)
  subscription_status (trial / active / expired / cancelled)
  billing_cycle       (monthly / yearly)
  enabled_at
  config              JSONB  (operation-specific settings)

business_branches:
  id                  PK
  parent_business_id  → businesses.id
  name
  address, city, country
  timezone, currency
  status

business_links:
  id                  PK
  business_a_id       → businesses.id
  business_b_id       → businesses.id
  shared_resources    JSONB  (folio / kitchen / staff / inventory / crm)
  created_at
```

**Features:**
```
├── Create business (name, slug, address, timezone, currency)
├── Enable operation → subscription starts (trial or paid)
├── Disable operation
├── Add branch (same brand, new location)
├── Link two businesses (enable shared resources)
├── Subdomain slug → business_id routing (for JWT Phase 2)
├── JWT Phase 2 issue { user_id, business_id }
└── Owner dashboard (all businesses + operations overview)
```

**Subscription logic:**
```
Each business_operation has its own plan.
Owner's monthly invoice = sum of all active operation plans.
ONE invoice. Multiple line items.
```

---

### Module 3: Membership

**One line:** WHO works WHERE with WHAT permission.

**Responsibility:**
Manages staff access to businesses. Handles roles, permissions, and terminal PIN login.
This is access control ONLY — not HR, not scheduling, not payroll.

**Owns:**
```
staff_memberships:
  id                  PK
  user_id             → users.id
  business_id         → businesses.id
  role                (owner / gm / manager / supervisor / staff / trainee)
  custom_permissions  JSONB  (override specific permissions)
  pin_hash            (for POS / terminal fast login)
  status              (invited / active / inactive)
  invited_at
  joined_at
```

**Features:**
```
├── Invite staff (owner sends email → staff accepts → membership created)
├── Assign role per business
├── Override specific permissions per staff member
├── PIN setup for terminal / POS login
├── PIN validation (terminal login flow)
├── Revoke / deactivate access
├── JWT Phase 3 issue { user_id, business_id, role, permissions[] }
└── RBAC permission checks → exposed via IRbacService in Shared.Contracts
```

**Role hierarchy:**
```
owner      → full access to everything
gm         → full access within one business
manager    → manage staff + operations, override pricing
supervisor → floor control, some overrides
staff      → daily operations only
trainee    → view only / very limited
```

**Permission enforcement (2 layers):**
```
Layer 1: UI → hides features staff can't access
Layer 2: API → rejects request even if UI is bypassed
         → Every API call: IRbacService.HasPermission(jwt, action)
```

**Does NOT own:**
- Staff scheduling, shifts, payroll, performance → Phase 2, inside operation modules

---

## SHARED SERVICES (Phase 0 — Build with Core)

---

### Module 4: Payment Service

**One line:** All money movement. PCI compliance isolated here.

**Responsibility:**
Single place for all payment processing across all operations.
No operation module handles payment gateway directly — they all call IPaymentService.

**Features:**
```
├── Payment gateway abstraction (Stripe / local gateway)
├── Charge (card / cash / digital wallet)
├── Refund / partial refund
├── Immutable payment audit trail
├── Receipt generation
└── Payment status tracking
```

**Communication:**
- Exposed via `IPaymentService` interface in Shared.Contracts
- Called by: Dining, Stays, Bar, Wellness, Events, Retail

---

### Module 5: Notification Service

**One line:** All outbound communication from OneNex.

**Responsibility:**
Single place for all emails, SMS, and push notifications.
No module sends emails directly — they call INotificationService.
Business branding applied here (logo, colors in email templates).

**Features:**
```
├── Email (booking confirm, receipt, invite, password reset, reminders)
├── SMS (OTP, check-in alert, order ready)
├── Push notifications (Phase 2)
├── Template management (per business branding)
└── Delivery tracking (sent / failed / opened)
```

**Communication:**
- Exposed via `INotificationService` interface in Shared.Contracts
- Called by: Identity, Business, Membership, all Operation modules

---

## OPERATION MODULES

**Rule:** Each operation = fully self-contained, end-to-end module.
No other module touches its internals. Communicates only via events + interfaces.

Each operation has:
- **Core features** — always on, operation cannot work without them
- **Add-ons** — off by default, owner enables via setup wizard

---

### Module 6: Dining (Phase 1 — Build First)

**One line:** Complete restaurant / F&B operation management.

**Core (always on):**
```
Menu Management
  ├── Categories, items, prices, photos
  ├── Variants (size: small/large)
  ├── Modifiers (extra cheese, no onion)
  └── Combo deals, time-based availability

Table Management
  ├── Floor plan layout
  ├── Table status: available / occupied / reserved / cleaning
  ├── Merge tables, split tables
  └── Capacity per table

Order Management
  ├── Create order → link to table (or takeaway)
  ├── Add / remove / modify items
  ├── Transfer order to different table
  ├── Order status: placed → preparing → ready → served
  └── Void / cancel order (with permission)

Billing & Payment
  ├── Generate bill (itemized)
  ├── Apply discount (% or fixed)
  ├── Split bill (by person / by item)
  ├── Payment → calls IPaymentService
  ├── Receipt / invoice
  └── Refund / void (with permission)

Basic Reports
  ├── Daily sales summary
  ├── Top selling items
  ├── Table turnover rate
  └── Void / discount report
```

**Add-ons (enable when needed):**
```
KDS — Kitchen Display System
  ├── Orders push to kitchen screen in real-time (SignalR)
  ├── Station routing (grill station sees grill items only)
  └── Mark item ready → notify floor staff

Table Reservation
  ├── Book table (date, time, party size)
  ├── Time slot management
  ├── Status: confirmed / arrived / no-show / cancelled
  └── Walk-in vs reservation floor view

QR Self-Ordering
  ├── Generate QR code per table
  ├── Customer scans → views menu on phone
  ├── Customer places order → goes directly to KDS
  └── Customer pays from phone → calls IPaymentService

Takeaway
  ├── Counter orders (customer name, pickup number)
  ├── Prep time estimate
  └── Ready notification (SMS via INotificationService)

Delivery
  ├── Delivery zones + fees
  ├── Estimated delivery time
  └── Order tracking status

Inventory Tracking
  ├── Ingredient management
  ├── Recipe → ingredient mapping
  ├── Auto-deduct stock when item sold
  └── Low stock alerts
```

**Events published:**
```
OrderPlaced       → INotificationService (kitchen staff alert)
OrderReady        → INotificationService (floor staff alert)
OrderVoided       → Analytics (Phase 2)
PaymentCompleted  → Analytics (Phase 2)
```

---

### Module 7: Stays (Phase 2)

**One line:** Complete hotel / accommodation operation management.

**Core (always on):**
```
Room & Property Setup
  ├── Room types (single, double, suite...)
  ├── Room configuration (floor, view, amenities, photos)
  ├── Room status: VD / OD / DND / IP / VC / INS / OOO / OC
  └── Out-of-order management

Rate Plans
  ├── Base rate per room type
  ├── Seasonal / weekend / weekday rates
  ├── Inclusions (breakfast, airport transfer)
  ├── Length-of-stay restrictions (minimum nights)
  └── Cancellation policy per rate plan

Booking Management
  ├── Booking types: walk-in / phone / staff-created
  ├── Booking status: confirmed → checked-in → checked-out / cancelled / no-show
  ├── Modify booking (dates, room type, rate)
  ├── Cancel booking (apply policy → refund via IPaymentService)
  ├── No-show handling (charge policy, mark no-show)
  └── Availability calendar

Front Desk Operations
  ├── Check-in → folio auto-created
  ├── Room assignment (manual or smart assign)
  ├── Early check-in / late check-out
  ├── Room change during stay (folio transfer)
  ├── Guest document verification
  └── Check-out → folio settled → invoice

Folio & Billing  ← V1: lives inside Stays module
  ├── Folio auto-created at check-in
  ├── Night audit: auto room charge posted every night (Hangfire job)
  ├── Manual charge entry (extras, minibar, damages)
  ├── Partial payment during stay
  ├── Final settlement → calls IPaymentService
  └── GST invoice generation

Basic Reports
  ├── Occupancy rate
  ├── ADR (Average Daily Rate)
  ├── RevPAR
  ├── Arrivals / departures today
  └── Revenue by room type
```

**Add-ons (enable when needed):**
```
Housekeeping App
  ├── Task assignment per housekeeper
  ├── Priority queue (VIP arriving, early check-in first)
  ├── Mobile status update: dirty → cleaning → clean → inspected
  ├── Room checklist (items to verify)
  └── Report maintenance issue

OTA Channel Management
  ├── Connect: Booking.com, MakeMyTrip (V1)
  ├── Real-time availability sync (milliseconds — SignalR push)
  ├── OTA booking auto-import via webhooks
  └── Commission tracking per channel

Online Booking Engine
  ├── Customer self-booking via business website / app
  ├── Rate selection, room selection
  ├── Payment (deposit / full → IPaymentService)
  └── Booking confirmation → INotificationService

Revenue Management
  ├── Yield rules (auto price by occupancy %)
  ├── Visual rate calendar with color coding
  └── LOS restrictions for peak periods

Corporate Rate Management
  ├── Corporate account setup
  ├── Contract rate auto-apply
  └── Monthly auto-invoice generation
```

**Events published:**
```
BookingCreated    → INotificationService (confirmation email)
GuestCheckedIn    → INotificationService (welcome SMS)
                  → Housekeeping (room prep task if needed)
NightAuditRun     → Analytics (Phase 2)
GuestCheckedOut   → INotificationService (invoice email)
                  → CRM (visit recorded — Phase 2)
BookingCancelled  → INotificationService (cancellation confirm)
```

**Note on Folio — Phase 3 extraction:**
```
V1:      Folio lives inside Stays (hotel-only context, simple)
Phase 3: Extract to "Folio Hub" module when:
         → Hotel guest charges restaurant/bar/spa bill to room folio
         → Link concept activates cross-operation charging
         → Folio Hub becomes the central billing point
```

---

### Modules 8–11: Bar, Wellness, Events, Retail (Phase 3+)

Same pattern as Dining and Stays: Core + Add-ons. Fully self-contained.

```
Bar Module
  Core:    Drink menu, orders, tabs, billing
  Add-ons: Tab management, Happy Hour rules, Bar KDS, QR ordering

Wellness Module
  Core:    Services/treatments, appointments, therapist assignment, billing
  Add-ons: Online booking, package deals, loyalty integration

Events Module
  Core:    Event management, ticketing, capacity, billing
  Add-ons: Catering link (via Dining), seating plans, attendee management

Retail Module
  Core:    Product catalog, POS, basic inventory, billing
  Add-ons: Advanced inventory, loyalty, online store
```

---

## PLATFORM MODULES (Phase 2+)

---

### Customer CRM (Phase 2)

```
Responsibility: Single guest/customer profile across all businesses.

Owns:
  customer_profiles: user_id, organization-level data,
                     preferences, visit history, loyalty points

Features:
  ├── Cross-business guest recognition
  ├── Visit history (stayed at hotel 3x, visited restaurant 8x)
  ├── Preferences (room type, dietary, special occasions)
  └── VIP / blacklist flags

V1: Basic profile stored in Identity module
Phase 2: Extract to full CRM module
```

---

### Folio Hub (Phase 3)

```
Responsibility: Cross-operation billing when businesses are linked.

When activated:
  Hotel guest → charges restaurant bill → posted to hotel folio
  Hotel guest → charges spa → posted to hotel folio
  Checkout → ONE bill (rooms + food + spa + bar)

Extract from Stays module when link concept activates.
```

---

### Analytics & Reports (Phase 2)

```
Responsibility: Cross-operation insights.

Features:
  ├── Cross-business revenue dashboard
  ├── Operation-level KPIs (ADR, RevPAR, table turnover...)
  ├── Staff performance reports
  └── Custom report builder (Phase 3)
```

---

### Loyalty & Offers (Phase 3)

```
Responsibility: Points engine + promotional rules.

Features:
  ├── Points earning rules per operation
  ├── Points redemption
  ├── Offer / promotion engine
  └── Tier management (Silver / Gold / Platinum)
```

---

## Module Communication Rules

```
✅ ALLOWED:

Sync (via Shared.Contracts interfaces):
  Any module → IPaymentService.Charge(...)
  Any module → INotificationService.Send(...)
  Any module → IRbacService.HasPermission(...)

Async (via MediatR Domain Events):
  Dining:  OrderPlaced event → KDS Handler + Notification Handler
  Stays:   GuestCheckedIn event → Housekeeping Handler + Notification Handler

❌ NOT ALLOWED:
  Module A → Module B's Repository
  Module A → Module B's Domain classes
  Module A → Module B's database tables
  Module A → Module B directly (no direct object reference)
```

---

## V1 Build Order

```
Phase 0 — Core (before any feature code):
  ✓ Identity Module
  ✓ Business Module
  ✓ Membership Module
  ✓ Payment Service
  ✓ Notification Service

Phase 1 — First Operation:
  ✓ Dining Module
    Core: Menu + Tables + Orders + Billing + Basic Reports
    Add-on: KDS (minimum for V1)

Phase 2 — Hotel + Expand:
  ✓ Stays Module
    Core: Rooms + Rate Plans + Bookings + Front Desk + Folio + Reports
    Add-on: Housekeeping App (minimum for V1)
  ✓ Customer CRM (basic)
  ✓ Analytics (basic)

Phase 3+:
  ○ Folio Hub (extracted from Stays)
  ○ Bar, Wellness, Events, Retail modules
  ○ OTA Channel Management
  ○ Loyalty & Offers
  ○ Advanced Analytics
```

---

## Open Questions (Review Before Finalizing)

- JWT Phase 2 + 3 — one combined API call or two separate calls?
- business_links shared_resources JSONB — is this enough or do we need separate link config tables per resource type?
- Bar V1 — inside Dining module (simpler) or separate module from day 1?
- Customer CRM V1 — exact fields stored in Identity vs Business vs future CRM module?
- PIN hash algorithm — bcrypt or argon2?
- Branch vs new Business — exact rule: same brand + owner = branch, different brand = new business?
- business_operations config JSONB — define exactly what goes in it per operation type
- Night audit job (Stays) — exact time? Configurable per business timezone?
