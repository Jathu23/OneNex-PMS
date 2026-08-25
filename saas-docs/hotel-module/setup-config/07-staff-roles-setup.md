# Setup & Configuration — 07: Staff & Roles Setup
> Hotel Module → Setup & Configuration → Area 7 of 16
> Foundation for: Access Control, Audit Trail, PIN Login, Manager Overrides, Multi-Property Access

---

## Why Staff & Roles Setup Matters

```
Without Staff & Roles Setup:
  → Everyone has full access → billing errors, unauthorized discounts
  → Housekeeper accidentally modifies rate plans
  → Front desk voids charges without manager approval
  → No audit trail — who did what?

With proper setup:
  → Each role sees only what they need
  → Sensitive actions require manager PIN override
  → Every action logged against a staff member
  → New staff onboarded in 2 minutes (assign role → done)
```

---

## Existing Systems — Problems

| System | Problem |
|--------|---------|
| Oracle OPERA | Permissions extremely granular (200+ checkboxes per user). Configuring a single staff profile takes 45 mins. Small hotels just give everyone admin access. |
| Mews | Good role system but limited to predefined roles. Can't customize permissions within a role. |
| Cloudbeds | Basic 3-level access (Admin / Manager / Staff). No department structure. All "Staff" see the same things regardless of department. |
| All systems | No PIN-based quick login for shared terminals. Staff must log in/out fully every time. Slows down front desk. |

---

## Our Design Principles

### 1. Pre-Built Role Templates (Start Fast)
```
"What is this staff member's role?"

  ○ Owner / General Manager    → Full access, all modules
  ○ Front Desk Manager         → Bookings, check-in/out, folio, reports
  ○ Front Desk Staff           → Bookings, check-in/out, folio (limited)
  ○ Housekeeping Supervisor    → Assign tasks, inspect rooms, manage staff
  ○ Housekeeper                → Update room status, complete checklists
  ○ Restaurant Manager         → Menu, orders, billing, reports
  ○ Restaurant Staff           → Take orders, process bills
  ○ Maintenance Staff          → View and close maintenance tickets
  ○ Accountant                 → Reports, invoices, folio settlement
  ○ Custom Role                → Define from scratch

Role selected → permissions pre-configured → ready in 30 seconds.
Hotel can customize any pre-built role.
```

### 2. Permission Areas (What Can Each Role Do?)
```
BOOKING
  view | create | modify | cancel | no-show mark

CHECK-IN / CHECK-OUT
  process_checkin | process_checkout | early_checkin | late_checkout

FOLIO & BILLING
  view | post_charge | give_discount | comp_item | settle | void | split_folio
  discount_max_percentage: 0% / 10% / 20% / unlimited

ROOMS
  view | assign_room | mark_ooo | release_ooo

RATE PLANS
  view | modify_rates | create_rate_plan

HOUSEKEEPING
  assign_tasks | update_room_status | inspect_rooms | manage_zones

MAINTENANCE
  view_tickets | create_ticket | close_ticket | escalate

REPORTS
  view_daily | view_revenue | view_full | export

STAFF MANAGEMENT
  view_staff | add_staff | edit_staff | deactivate_staff

SYSTEM SETTINGS
  view_settings | modify_settings
```

### 3. Predefined Roles — Permission Matrix
```
                        Owner  FD Mgr  FD Staff  HK Sup  HK Staff  Accounts
──────────────────────────────────────────────────────────────────────────────
Booking: Create           ✅     ✅       ✅        ☐        ☐        ☐
Booking: Cancel           ✅     ✅       ⚠ (mgr)   ☐        ☐        ☐
Check-in / out            ✅     ✅       ✅        ☐        ☐        ☐
Folio: Post charge        ✅     ✅       ✅        ☐        ☐        ✅
Folio: Give discount      ✅     ✅       ⚠ 10%    ☐        ☐        ☐
Folio: Comp item          ✅     ✅       ☐         ☐        ☐        ☐
Folio: Void               ✅     ⚠ (mgr)  ☐        ☐        ☐        ☐
Room: Mark OOO            ✅     ✅       ☐         ✅        ☐        ☐
Rate Plans: Modify        ✅     ✅       ☐         ☐        ☐        ☐
HK: Assign tasks          ✅     ✅       ☐         ✅        ☐        ☐
HK: Update status         ✅     ✅       ✅        ✅        ✅        ☐
HK: Inspect rooms         ✅     ✅       ☐         ✅        ☐        ☐
Reports: Full access      ✅     ✅       ☐         ☐        ☐        ✅
System settings           ✅     ☐        ☐         ☐        ☐        ☐

⚠ = Requires manager PIN override
```

### 4. PIN System — Quick Login for Shared Terminals
```
PROBLEM: Front desk uses one shared tablet/PC.
  Full login/logout every shift change = slow.

SOLUTION: PIN-based session switching.

  Each staff member: 4-6 digit PIN
  Tablet stays on "Who are you?" screen
  Staff taps name → enters PIN → instant access
  Auto-logout after 10 mins inactivity → back to PIN screen

Manager Override PIN:
  Some actions prompt: "Manager approval required"
  → Manager enters their PIN (doesn't need to switch session)
  → Action approved and logged against manager
  → Front desk continues their session

Actions requiring manager PIN:
  → Cancellation with refund > ₹5,000
  → Discount > 10%
  → Comp any item (₹0 charge)
  → Void a posted charge
  → Override OOO room assignment
  → Check-in without payment
```

### 5. Department Structure
```
DEPARTMENTS:
  MANAGEMENT          → Owner, GM, Revenue Manager
  FRONT_DESK          → FD Manager, FD Staff, Night Auditor
  HOUSEKEEPING        → HK Supervisor, Housekeeper
  FOOD_AND_BEVERAGE   → Restaurant Manager, Waitstaff, Bar Staff
  MAINTENANCE         → Maintenance Supervisor, Technician
  ACCOUNTS            → Accountant, Finance Manager

Department assignment:
  → Groups staff on reports
  → Filters task assignment (maintenance ticket → maintenance dept only)
  → Department head gets escalation alerts
```

### 6. Staff Profile
```
STAFF PROFILE:
  Name:           Meena Kumari
  Employee ID:    EMP-0042
  Department:     Housekeeping
  Role:           Housekeeping Supervisor
  Phone:          9876543210
  Email:          meena@thegrand.com (optional)
  PIN:            ●●●● (hashed)
  Shift:          Morning (Mon–Sat, 7 AM – 3 PM)
  Joined:         12 Jan 2024
  Status:         Active

Multi-property:
  Can access: The Grand Chennai ✅ / The Grand Bangalore ☐
```

---

## Data Model

```
Staff
  id, hotel_id
  name
  employee_id           nullable
  department            MANAGEMENT / FRONT_DESK / HOUSEKEEPING /
                        FOOD_AND_BEVERAGE / MAINTENANCE / ACCOUNTS
  role_id               FK → Role
  phone
  email                 nullable
  pin_hash              (bcrypt hashed)
  photo_url             nullable
  hire_date             date
  is_active             bool

Role
  id, hotel_id
  name                  "Front Desk Manager"
  description
  is_system_role        bool (system roles can't be deleted, only customized)

RolePermissions
  role_id
  -- Booking --
  booking_view          bool
  booking_create        bool
  booking_modify        bool
  booking_cancel        bool
  booking_noshow        bool
  -- Check-in/out --
  checkin_process       bool
  checkout_process      bool
  -- Folio --
  folio_view            bool
  folio_post_charge     bool
  folio_give_discount   bool
  folio_discount_max_pct int  (0 = cannot discount, 100 = unlimited)
  folio_comp_item       bool
  folio_settle          bool
  folio_void            bool
  folio_split           bool
  -- Rooms --
  room_view             bool
  room_assign           bool
  room_mark_ooo         bool
  -- Rates --
  rate_view             bool
  rate_modify           bool
  -- Housekeeping --
  hk_assign_tasks       bool
  hk_update_status      bool
  hk_inspect_rooms      bool
  hk_manage_zones       bool
  -- Maintenance --
  maint_view            bool
  maint_create          bool
  maint_close           bool
  -- Reports --
  report_daily          bool
  report_revenue        bool
  report_full           bool
  report_export         bool
  -- Staff --
  staff_view            bool
  staff_manage          bool
  -- Settings --
  settings_view         bool
  settings_modify       bool

ManagerOverrideConfig
  hotel_id
  override_required_for JSON [
    { action: "cancel_refund", threshold_amount: 5000 },
    { action: "discount", threshold_pct: 10 },
    { action: "comp_item" },
    { action: "void_charge" },
    { action: "ooo_override" },
    { action: "checkin_without_payment" }
  ]

StaffShift
  staff_id
  days_of_week          JSON [MON, TUE, WED, THU, FRI, SAT]
  shift_start           "07:00"
  shift_end             "15:00"

StaffPropertyAccess
  staff_id, hotel_id    (multi-property — which properties this staff can access)

AuditLog
  id, hotel_id
  staff_id
  action                "folio_void"
  entity_type           "FolioCharge"
  entity_id
  details               JSON
  override_by_staff_id  nullable (if manager PIN was used)
  timestamp
```

---

## Key Relationships

```
Hotel → Role (many — some system, some custom)
Hotel → Staff (many)
Staff → Role (FK — one role per staff)
Role → RolePermissions (one-to-one)
Staff → StaffShift (one active shift)
Staff → StaffPropertyAccess (multi-property)
Staff → AuditLog (every action logged)

ManagerOverrideConfig (one per hotel — defines what needs override)
```

---

## V1 vs Phase Split

| Feature | V1 | Phase 2 | Phase 3 |
|---------|:--:|:-------:|:-------:|
| Pre-built role templates (9 roles) | ✅ | | |
| Permission matrix per role | ✅ | | |
| Staff profiles (name, role, dept, PIN) | ✅ | | |
| PIN-based quick login | ✅ | | |
| Manager PIN override for sensitive actions | ✅ | | |
| Audit log (who did what, when) | ✅ | | |
| Department structure | ✅ | | |
| Shift assignment (days + hours) | ✅ | | |
| Multi-property staff access | ✅ | | |
| Custom role creation (from scratch) | ✅ | | |
| Discount limit per role | ✅ | | |
| Biometric login (fingerprint) | | ✅ | |
| Staff performance reports | | ✅ | |
| Payroll integration | | | ✅ |
| HR module (leave, attendance) | | | ✅ |
