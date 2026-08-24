# Housekeeping — Overview & Smart Features
> Hotel Module → Housekeeping
> Status: V1 required (simplified) | Advanced features Phase 2+

---

## Existing Systems — Problems

| System | Weakness |
|--------|---------|
| Oracle OPERA | Paper-based or basic mobile, no intelligent assignment |
| Mews | Good mobile app, still manual task assignment |
| Cloudbeds | Very basic — just room status, no task management |

**Core problem:** Most hotels still run housekeeping on paper checklists and walkie-talkies.
Even with software — no intelligence. Who cleans which room = supervisor instinct, not data.

---

## Room Status — Foundation of Housekeeping

| Status | Code | Meaning |
|--------|------|---------|
| Vacant Dirty | VD | Guest checked out → needs full cleaning |
| Occupied Dirty | OD | Guest staying → daily refresh needed |
| Do Not Disturb | DND | Guest put DND → skip today |
| In Progress | IP | Housekeeper currently cleaning |
| Vacant Clean | VC | Cleaned → awaiting supervisor inspection |
| Inspected | INS | Approved by supervisor → available for check-in |
| Out of Order | OOO | Maintenance issue → not for sale |
| Occupied Clean | OC | Cleaned while guest was out, guest returned |

**Rule: Front desk can only assign INSPECTED rooms. Never just VC.**

---

## 1. Smart Task Assignment *(Phase 2)*

### V1 Approach (Simple)
Supervisor manually assigns rooms to housekeepers each morning.

### Phase 2 Approach (Smart)
```
System auto-generates optimized assignment:

Factors:
  ├── Priority (VIP arriving, early check-in requests)
  ├── Floor grouping (same floor = less travel time)
  ├── Workload balance (equal load per housekeeper)
  ├── Room type weight (Suite = more time than Standard)
  └── Estimated completion vs arrival times

HOUSEKEEPING ASSIGNMENT — Dec 15

Housekeeper: Lakshmi (Floor 3-4)
  🔴 PRIORITY
    Room 301 — VD — Guest arrives 11 AM
    Room 312 — VD — Early check-in 12 PM
  🟡 NORMAL
    Room 305 — OD | Room 308 — OD | Room 310 — VD
  Estimated: 3.5 hours | Priority rooms done by 10:30 AM
```

---

## 2. Mobile App for Housekeepers

### V1 Approach (Simple)
Basic status update — housekeeper marks room done via simple screen.

### Phase 2 Approach (Full App)
```
Housekeeper opens app → Sees task list → Taps [Start]
  → Status: IN PROGRESS (front desk sees live)
  → Timer running

Done → Checklist:
  ☑ Bed made
  ☑ Bathroom cleaned
  ☑ Towels replaced
  ☑ Mini bar checked (items consumed logged here)
  ☑ AC set to default
  [Submit]

→ Status: VACANT CLEAN
→ Mini bar items → Auto-posted to guest folio
→ Supervisor notified for inspection
```

---

## 3. Priority Alert System *(Phase 2)*

```
Auto-alerts when priority rooms not started in time:

🔴 Room 301 — VIP arriving 2 PM (current time 8 AM → 6 hrs)
🟠 Room 312 — Early check-in 12 PM (current time 8 AM → 4 hrs)

Alert at 9:30 AM if Room 301 not started:
"⚠ Room 301 not started — VIP guest arrives in 90 min!"
```

---

## 4. Inspection Workflow

### V1 Approach (Simple)
Supervisor marks room inspected manually in system.

### Phase 2 Approach (Formal Workflow)
```
Housekeeper done → Status: VACANT CLEAN
        ↓
Supervisor notified → Goes to room → Opens app
        ↓
Inspection checklist:
  ☑ Overall cleanliness
  ☑ Bathroom spotless
  ☑ Amenities restocked
  ☑ Mini bar verified
  ☑ No damage
  [ ] Issue? → Photo + description
        ↓
[Approve] → INSPECTED → Front desk notified
[Reject]  → Back to housekeeper → Fix → Re-inspect
```

---

## 5. Lost & Found

### V1 Approach (Simple)
Staff logs lost item: Room, description, found by, date. Searchable list.

### Phase 2 Approach
```
Housekeeper finds item:
  → [Report] in app → Photo → Description
  → System assigns: Lost Item #LF-045
  → Tagged: Room 301, Dec 15, Found by Lakshmi

Guest calls:
  → Staff searches by room + date + item type
  → Photo confirms → Return / courier arranged

Auto-flag: Items unclaimed after 90 days → Disposal process
```

---

## 6. Linen & Supplies Management *(Phase 2)*

```
Each room: Standard supply quota defined
Housekeeper logs consumed items during cleaning
System tracks:
  ├── Daily consumption per room
  ├── Laundry request auto-generated
  └── Low stock alerts to supervisor
```

---

## 7. Green / Eco Option *(Phase 3)*

```
Guest option (at check-in or app):
  "Daily housekeeping?"
  ○ Yes — daily cleaning
  ○ No — skip (earn loyalty points)
  ○ Every other day

Guest skips → System removes room from today's list
            → Loyalty points added automatically
```

---

## 8. Performance Tracking *(Phase 2)*

```
Per housekeeper (monthly):
  ├── Avg rooms per shift
  ├── Avg time per room type
  ├── Inspection pass rate (first attempt)
  └── Guest complaints linked to their rooms

Data-driven evaluation — not supervisor opinion.
```

---

## 9. Turnover Time Tracking *(Phase 2)*

```
Room 301:
  Checkout:           11:05 AM
  Cleaning started:   11:45 AM  (40 min gap)
  Cleaning done:      12:30 PM  (45 min)
  Inspected:          12:50 PM  (20 min)
  Available:          12:50 PM  ✅

Identifies bottlenecks:
  Long gap before start → Assignment delay?
  Long cleaning time → Training issue?
  Long inspection wait → Supervisor availability?
```

---

## V1 vs Later

```
V1 MUST HAVE:
  ✅ Room status management (all statuses)
  ✅ Basic task list per housekeeper (manual assignment)
  ✅ Status update by housekeeper (simple web/mobile)
  ✅ Supervisor inspection (approve/reject — simple)
  ✅ Front desk auto-notified when room inspected
  ✅ Lost & found (basic digital log)
  ✅ Manual mini bar charge entry

PHASE 2:
  ➕ Smart auto-assignment with scoring
  ➕ Priority alert system with countdown
  ➕ Full mobile app with checklist + photo
  ➕ Mini bar scan → auto-folio post
  ➕ Linen & supply tracking
  ➕ Performance analytics per housekeeper
  ➕ Turnover time tracking

PHASE 3:
  ➕ Green / eco option with loyalty points
  ➕ Predictive workload planning
  ➕ Laundry integration
```

---

## Comparison Summary

| Feature | Existing Systems | Our System |
|---------|-----------------|------------|
| Task assignment | Manual | V1: Manual | Phase 2: Smart auto |
| Housekeeper interface | Paper / basic | V1: Simple web | Phase 2: Mobile app |
| Priority rooms | Manual awareness | Phase 2: Auto-alerts |
| Inspection | Ad-hoc | V1: Simple approve | Phase 2: Checklist + photo |
| Lost & found | Paper register | V1: Basic digital log |
| Mini bar | Paper slip | V1: Manual entry | Phase 2: App scan → auto-folio |
| Performance | Observation | Phase 2: Data-driven |
| Eco option | None | Phase 3 |
