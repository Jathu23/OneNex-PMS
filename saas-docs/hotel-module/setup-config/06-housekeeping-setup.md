# Setup & Configuration — 06: Housekeeping Setup
> Hotel Module → Setup & Configuration → Area 6 of 16
> Foundation for: Housekeeping Operations, Room Status Flow, Mini Bar Billing, Inspection Workflow

---

## Why Housekeeping Setup Matters

```
Housekeeping Setup defines the RULES:
  → Which rooms need inspection before assignment?
  → How long should cleaning a Standard room take?
  → What amenities go in each room type?
  → Who is responsible for which floor?
  → What is in the mini bar?

Wrong setup = housekeeper doesn't know what to do
             = front desk can't track room readiness
             = guest gets wrong amenities
             = mini bar billing errors
```

---

## Existing Systems — Problems

| System | Problem |
|--------|---------|
| Oracle OPERA | Housekeeping config buried deep in admin. Checklists not per room type — one generic list for all rooms. No amenity quantity setup. |
| Mews | No checklist feature at all. Housekeeping is just status update. |
| Cloudbeds | Basic status only. Zone assignment manual every day (not pre-configured). Mini bar completely separate or not supported. |
| All systems | No turnover time target. No way to know if a room is taking too long. Supervisor discovers problem only at check-in. |

---

## Our Design Principles

### 1. Zones — Pre-Configured, Not Daily Manual
```
ZONE SETUP (done once, stays until changed):

Zone A — Floor 1 & 2
  → Assigned to: Meena, Priya
  → Room count: 20 rooms
  → Shift: Morning (7 AM – 3 PM)

Zone B — Floor 3 & 4
  → Assigned to: Kavitha, Radha
  → Room count: 20 rooms
  → Shift: Morning (7 AM – 3 PM)

Zone C — Floor 5 (Suites)
  → Assigned to: Lakshmi (Senior)
  → Room count: 8 rooms
  → Shift: Morning (7 AM – 3 PM)

Zone D — Evening turndown
  → Assigned to: Sundari
  → Shift: Evening (4 PM – 11 PM)

Daily: system auto-assigns tasks based on zones.
Supervisor only overrides exceptions.
```

### 2. Checklist Per Room Type (Not Generic)
```
STANDARD DOUBLE — Cleaning Checklist:
  □ Strip and replace bed linen
  □ Clean bathroom (toilet, basin, shower)
  □ Replace towels (2 bath + 2 hand + 2 face)
  □ Restock toiletries (shampoo, soap, conditioner)
  □ Vacuum carpet / mop floor
  □ Dust all surfaces
  □ Check TV, AC, lights working
  □ Empty trash bins
  □ Replenish tea/coffee station
  □ Close curtains / arrange pillows
  □ Check for lost items → log if found

SUITE — Additional checklist items:
  □ Polish all wooden surfaces
  □ Arrange fruit basket (if included)
  □ Restock mini bar (check and replace)
  □ Arrange toiletries in luxury layout
  □ Iron curtains if needed
  □ Refill bath salts / premium amenities

Checklist defined in setup → housekeeper sees it on mobile app.
Cannot mark room VC (Vacant Clean) until all items checked.
```

### 3. Amenity Quantities Per Room Type
```
ROOM TYPE: Standard Double
  Shampoo:        2 bottles
  Conditioner:    2 bottles
  Body Wash:      2 bottles
  Soap:           2 bars
  Toothbrush:     0 (not provided)
  Toothpaste:     0
  Bath Towel:     2
  Hand Towel:     2
  Face Towel:     2
  Tea bags:       4
  Coffee:         2 sachets
  Sugar:          4 sachets
  Mineral Water:  2 bottles

ROOM TYPE: Suite
  Everything above × 2
  + Premium toiletry kit
  + Fruit basket
  + Complimentary wine (on arrival only)
  + Bathrobe: 2
  + Slipper sets: 2

System generates restocking list automatically.
Housekeeping store knows exactly what to pack per room type.
```

### 4. Mini Bar Setup
```
MINI BAR ITEMS:
  Item              Price    Category
  ─────────────────────────────────────
  Kingfisher Beer   ₹250     Alcohol
  Sprite 330ml      ₹120     Soft Drink
  Mineral Water 1L  ₹80      Water
  Pringles          ₹180     Snack
  Kitkat            ₹60      Chocolate
  Cashews           ₹150     Snack

Which room types have mini bar?
  → Standard: No
  → Deluxe: Yes
  → Suite: Yes (premium selection)

Mini bar charge → auto-posts to guest folio when staff logs consumption.
Phase 2: Housekeeper scans barcode on app → auto-posts instantly.
```

### 5. Turnover Time Targets
```
ROOM TYPE         TARGET TIME
──────────────────────────────
Standard Double    25 mins
Deluxe Room        35 mins
Junior Suite       45 mins
Suite              60 mins
Stay-over (DND)    15 mins (light service only)

If room exceeds target:
  → Supervisor alert on dashboard
  → Priority flag on floor map

This is how system detects slow rooms BEFORE check-in is affected.
```

### 6. Inspection Rules
```
INSPECTION SETUP:
  Inspection required before room assignment? [YES / NO]

  If YES:
    Who can inspect?    → Supervisor role only
    All rooms?          → Yes / No
    If No, which types? → Suites only / All except Standard / Custom

  Inspection checklist (separate from cleaning checklist):
    □ Linen — white, crisp, no stains
    □ Bathroom — spotless, no hair, no soap residue
    □ Floor — no dust, no debris
    □ Amenities — correct quantities placed
    □ Mini bar — stocked correctly (if applicable)
    □ AC — set to 24°C
    □ Lights — all working
    □ TV — tested, working
    □ Room fragrance — applied
    □ Overall presentation — ready for VIP?

  Fail → room goes back to OD (Occupied Dirty) for re-clean.
  Pass → room becomes INS (Inspected) → assignable at front desk.
```

### 7. Shift Setup
```
SHIFT CONFIG:
  Morning Shift:   7:00 AM – 3:00 PM
  Evening Shift:   3:00 PM – 11:00 PM
  Night Shift:     11:00 PM – 7:00 AM (optional)

  Turndown Service: 6:00 PM – 9:00 PM
    → Applies to: Suites only / All rooms / Custom

  Shift overlap allowed? Yes (30-min handover window)
```

---

## Data Model

```
HousekeepingZone
  id, hotel_id
  name                  "Zone A — Floor 1 & 2"
  floor_ids             JSON [floor_1_id, floor_2_id]
  shift                 MORNING / EVENING / NIGHT
  shift_start           "07:00"
  shift_end             "15:00"

ZoneAssignment
  zone_id, staff_id
  effective_from        date
  effective_until       date nullable (null = ongoing)

HousekeepingChecklist
  id, hotel_id, room_type_id
  checklist_type        CLEANING / INSPECTION / TURNDOWN
  items                 JSON [{ order, label, is_required }]

AmenityStock
  id, hotel_id, room_type_id
  item_name             "Shampoo"
  category              TOILETRY / LINEN / BEVERAGE / FOOD / OTHER
  quantity_per_room     int
  unit                  BOTTLE / BAR / PIECE / SET

MiniBarItem
  id, hotel_id
  name                  "Kingfisher Beer"
  category              ALCOHOL / SOFT_DRINK / WATER / SNACK / CHOCOLATE
  price                 decimal
  is_active             bool

RoomTypeMiniBar
  room_type_id, minibar_item_id
  quantity              int

HousekeepingConfig
  hotel_id
  inspection_required           bool
  inspection_role               SUPERVISOR / ANY_STAFF
  apply_inspection_to           ALL / SUITES_ONLY / CUSTOM
  turndown_service_enabled      bool
  turndown_applicable_to        ALL / SUITES_ONLY / CUSTOM

TurnoverTarget
  hotel_id, room_type_id
  stay_type                     CHECKOUT / STAYOVER
  target_minutes                int

ShiftConfig
  id, hotel_id
  shift_name            "Morning"
  shift_type            MORNING / EVENING / NIGHT / TURNDOWN
  start_time            "07:00"
  end_time              "15:00"
  is_active             bool
```

---

## Key Relationships

```
Hotel → HousekeepingZone → ZoneAssignment → Staff
Hotel → HousekeepingChecklist (per room type, per checklist type)
Hotel → AmenityStock (per room type)
Hotel → MiniBarItem → RoomTypeMiniBar → RoomType
Hotel → HousekeepingConfig (one config per hotel)
Hotel → TurnoverTarget (per room type)

Room → HousekeepingZone (FK — which zone this room belongs to)
HousekeepingTask → HousekeepingChecklist (which checklist applies)
MiniBarConsumption → MiniBarItem → FolioCharge (auto-post)
```

---

## V1 vs Phase Split

| Feature | V1 | Phase 2 | Phase 3 |
|---------|:--:|:-------:|:-------:|
| Zone setup (floor grouping) | ✅ | | |
| Zone → staff assignment | ✅ | | |
| Shift config (morning / evening) | ✅ | | |
| Inspection required toggle | ✅ | | |
| Basic inspection checklist | ✅ | | |
| Cleaning checklist per room type | ✅ | | |
| Amenity quantities per room type | ✅ | | |
| Mini bar items + pricing | ✅ | | |
| Mini bar per room type assignment | ✅ | | |
| Turnover time targets | ✅ | | |
| Turndown service config | ✅ | | |
| Mobile app checklist (housekeeper ticks on phone) | | ✅ | |
| Photo upload on inspection | | ✅ | |
| Supervisor alert if turnover time exceeded | | ✅ | |
| Performance tracking per housekeeper | | ✅ | |
| Mini bar barcode scan → auto-folio | | ✅ | |
| Smart task assignment (workload balancing) | | ✅ | |
| Eco / green opt-out with loyalty points | | | ✅ |
