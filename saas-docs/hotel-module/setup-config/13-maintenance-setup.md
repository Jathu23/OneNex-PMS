# Setup & Configuration — 13: Maintenance Setup
> Hotel Module → Setup & Configuration → Area 13 of 16
> Covers: Issue Categories + Priority + SLA + OOO Auto-Trigger + Ticket Lifecycle
> Foundation for: Maintenance Operations, Room OOO Management, Issue History

---

## V1 Decision: SKIP — Phase 2 Feature

```
Maintenance Setup is NOT required for V1 launch.

Reason:
  → Hotel can operate without a formal ticket system
  → Staff use WhatsApp / verbal communication temporarily
  → Room OOO can be manually managed via Room Setup in V1
  → Maintenance tracking is a nice-to-have, not a blocker

V1 approach:
  → Staff marks room OOO manually (already in Room Setup)
  → No ticket system
  → No SLA enforcement
  → No history tracking

Phase 2:
  → Full maintenance ticket system
  → SLA enforcement
  → OOO auto-trigger per ticket
  → Room history

This file documents the full design for Phase 2 implementation.
```

---

## Why Maintenance Setup Matters (Phase 2)

```
Without Maintenance Setup:
  → Maintenance request on paper/WhatsApp → lost, forgotten
  → No priority — AC issue and leaking roof get same treatment
  → Room stays OOO forever because no one tracked fix status
  → Guest checks into broken room → complaint → reputation damage

With proper setup:
  → Every issue tracked from raised → assigned → fixed → verified
  → Priority determines response time (SLA enforced)
  → Room auto-OOO when ticket opened, auto-released when fixed
  → History per room — "Room 301 had AC issues 4 times this year"
```

---

## Existing Systems — Problems

| System | Problem |
|--------|---------|
| Oracle OPERA | Maintenance is just "mark room OOO". No ticket system, no SLA, no history. |
| Mews | Basic maintenance notes. No assignment, no tracking, no SLA. |
| Cloudbeds | No maintenance module at all. Hotels use WhatsApp groups. |
| All systems | No room history. Can't see if a room is repeatedly problematic. No cost tracking. |

---

## Our Design Principles

### 1. Issue Categories — Structured (Not Free Text)
```
CATEGORY SETUP (hotel defines their list):

  HVAC
    → AC not cooling
    → AC not working
    → Heater issue
    → Ventilation issue

  PLUMBING
    → Toilet not flushing
    → Water leak
    → No hot water
    → Drain blocked

  ELECTRICAL
    → Light not working
    → Power socket issue
    → TV not working
    → Geyser issue

  FURNITURE
    → Bed broken
    → Chair damaged
    → Wardrobe door stuck
    → Mirror cracked

  HOUSEKEEPING RELATED
    → Room smell issue
    → Pest sighting
    → Stain on carpet / wall

  GENERAL
    → Door lock issue
    → Window latch broken
    → Safe not opening
    → Phone not working

Category → Sub-category → specific issue
This structure enables reporting: "How many AC issues this quarter?"
```

### 2. Priority Levels + SLA
```
PRIORITY SETUP:

  EMERGENCY
    Definition:   Guest safety at risk / major damage
    Examples:     Water flood, gas leak, fire alarm, no power in room
    SLA:          Respond within 15 mins / Resolve within 1 hour
    Alert:        Maintenance Supervisor + GM immediately

  HIGH
    Definition:   Guest comfort severely affected
    Examples:     AC not working, toilet not flushing, no hot water
    SLA:          Respond within 30 mins / Resolve within 3 hours
    Alert:        Maintenance Supervisor

  MEDIUM
    Definition:   Guest comfort partially affected
    Examples:     TV not working, light bulb out, geyser slow
    SLA:          Respond within 2 hours / Resolve same day
    Alert:        Maintenance team

  LOW
    Definition:   No immediate guest impact
    Examples:     Cosmetic damage, furniture scratch, door squeak
    SLA:          Respond within 24 hours / Resolve within 3 days
    Alert:        Daily maintenance list

SLA breach → auto-escalate to next level above.
If Emergency not responded in 15 mins → GM gets WhatsApp alert.
```

### 3. Room OOO Auto-Trigger Rules
```
OOO AUTO-TRIGGER SETUP:
  "When should a maintenance ticket automatically mark room OOO?"

  Always OOO:
    → EMERGENCY tickets (immediate)
    → HIGH priority tickets (after X hours unresolved)

  Never OOO (just track, room stays available):
    → LOW priority (cosmetic issues)

  Hotel decides per category:
    HVAC - HIGH:      OOO immediately
    PLUMBING - HIGH:  OOO immediately
    ELECTRICAL - MED: OOO if unresolved after 4 hours
    FURNITURE - LOW:  Never OOO

OOO release rule:
  Ticket status = RESOLVED + VERIFIED
  → Room status auto-changes: OOO → VC (Vacant Clean)
  → Available for booking again
  → Front desk notified
```

### 4. Ticket Lifecycle
```
TICKET FLOW:

  RAISED
    → Who can raise? Any staff / Guest (via portal) / Housekeeping
    → Details: Room, Category, Description, Priority (auto or manual)
    → Photos: optional attachment

  ASSIGNED
    → Auto-assign to maintenance dept? Yes / No
    → Manual assign by supervisor
    → Assigned staff gets WhatsApp + Push alert

  IN PROGRESS
    → Staff updates status when work starts
    → Timer starts for SLA tracking

  RESOLVED
    → Staff marks resolved + adds resolution note
    → Photo of fix (optional)

  VERIFIED
    → Supervisor or FD verifies fix
    → Room released from OOO (if applicable)

  CLOSED
    → Ticket archived
    → Room history updated

  REOPENED
    → If same issue recurs within X days → linked to original ticket
    → Pattern detection: "This is the 3rd AC issue in Room 301"
```

### 5. Who Can Raise a Ticket?
```
TICKET CREATION PERMISSION:
  Any staff member:     Yes / No
  Housekeeping:         Always (during room cleaning they find issues)
  Front Desk:           Always (guest complaint → ticket)
  Guest (via portal):   Yes / No (hotel decides)

  If guest raises via portal:
    → Ticket created
    → Auto-response sent to guest: "We're on it! Expected resolution: X hrs"
    → Staff alerted
    → Guest notified when resolved
```

### 6. Preventive Maintenance Schedule
```
PREVENTIVE MAINTENANCE SETUP (Phase 2+):

  Schedule:
    AC Filter cleaning:   Every 30 days
    Fire extinguisher:    Every 90 days
    Elevator inspection:  Every 180 days
    Pest control:         Every 60 days

  Auto-creates ticket on schedule date.
  Assigned to maintenance team.
  Must complete before due date.
```

### 7. Room Maintenance History
```
PER ROOM HISTORY:
  Room 301 — Maintenance History:
    ┌──────────┬──────────────┬──────────┬────────────┬──────────┐
    │ Date     │ Issue        │ Priority │ Resolved   │ Cost     │
    ├──────────┼──────────────┼──────────┼────────────┼──────────┤
    │ Jan 5    │ AC not cool  │ HIGH     │ Jan 5 3hrs │ ₹800     │
    │ Feb 12   │ AC not cool  │ HIGH     │ Feb 12 2hr │ ₹1,200   │
    │ Mar 8    │ AC not cool  │ HIGH     │ Mar 8 4hrs │ ₹2,500   │
    └──────────┴──────────────┴──────────┴────────────┴──────────┘
    → System flags: "Room 301 AC recurring issue — consider replacement"
```

---

## Data Model

```
MaintenanceCategory
  id, hotel_id
  name              "HVAC"
  sub_categories    JSON [{ name, examples[] }]
  is_active         bool

MaintenancePriority
  id, hotel_id
  level             EMERGENCY / HIGH / MEDIUM / LOW
  respond_within_mins   int
  resolve_within_mins   int
  alert_roles       JSON [role_ids to alert]
  escalate_after_mins   int
  escalate_to_roles JSON [role_ids]

MaintenanceConfig
  hotel_id
  allow_guest_tickets           bool
  allow_any_staff_tickets       bool
  auto_assign_to_dept           bool

OOOTriggerRule
  id, hotel_id
  category_id
  priority_level                EMERGENCY / HIGH / MEDIUM / LOW
  trigger_ooo                   IMMEDIATELY / AFTER_HOURS / NEVER
  trigger_after_hours           int nullable

MaintenanceTicket (runtime)
  id, hotel_id
  room_id
  category_id
  sub_category                  text
  description                   text
  priority                      EMERGENCY / HIGH / MEDIUM / LOW
  status                        RAISED / ASSIGNED / IN_PROGRESS /
                                RESOLVED / VERIFIED / CLOSED / REOPENED
  raised_by_staff_id            nullable
  raised_by_guest               bool
  assigned_to_staff_id          nullable
  resolution_note               nullable
  photos                        JSON [url list]
  ooo_triggered                 bool
  raised_at                     timestamp
  assigned_at                   timestamp nullable
  resolved_at                   timestamp nullable
  verified_at                   timestamp nullable
  sla_respond_due               timestamp
  sla_resolve_due               timestamp
  sla_breached                  bool
  parent_ticket_id              nullable

PreventiveMaintenance (Phase 2)
  id, hotel_id
  task_name                     "AC Filter Cleaning"
  frequency_days                int
  applies_to                    ALL_ROOMS / ROOM_TYPE / SPECIFIC_ROOMS
  room_type_id                  nullable
  last_done_date                date nullable
  next_due_date                 date
```

---

## V1 vs Phase Split

| Feature | V1 | Phase 2 | Phase 3 |
|---------|:--:|:-------:|:-------:|
| Manual room OOO (via Room Setup) | ✅ | | |
| Issue categories + sub-categories | | ✅ | |
| Priority levels (4) + SLA config | | ✅ | |
| Ticket lifecycle (Raised → Closed) | | ✅ | |
| Room OOO auto-trigger per ticket | | ✅ | |
| OOO auto-release when ticket verified | | ✅ | |
| Any staff can raise ticket | | ✅ | |
| SLA breach escalation alert | | ✅ | |
| Room maintenance history | | ✅ | |
| Photo attachment on ticket | | ✅ | |
| Guest raises ticket via portal | | ✅ | |
| Cost tracking per ticket | | ✅ | |
| Preventive maintenance schedule | | ✅ | |
| Recurring issue detection + alert | | ✅ | |
| Vendor management (external contractors) | | ✅ | |
| AI predictive maintenance alerts | | | ✅ |
