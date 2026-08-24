# Setup & Configuration — 02: Room Setup
> Hotel Module → Setup & Configuration → Area 2 of 13
> Foundation for: Availability Logic, Pricing, Housekeeping, OTA Sync

---

## Why Room Setup is the Foundation

```
Room Setup data correct-aa irundha:
  ├── Availability logic correct-aa work ஆகும்
  ├── Pricing correct room-ku apply ஆகும்
  ├── Housekeeping correct zone-la assign ஆகும்
  └── OTA sync correct inventory push ஆகும்

Everything depends on Room Setup.
Wrong setup = wrong operations everywhere.
```

---

## Two Levels of Room Setup

```
Level 1: ROOM TYPE  (Template / Category)
  "What kinds of rooms do we have?"
  Example: Standard Double, Deluxe Suite

Level 2: PHYSICAL ROOM  (Actual Room Number)
  "What are the actual room numbers?"
  Example: Room 101, 102... — each assigned to a type

Type = Template (define once, apply to many rooms)
Room = Instance (specific room with its own traits)
```

---

## Existing Systems — Problems

| System | Problem |
|--------|---------|
| Oracle OPERA | 3 confusing concepts (Physical/Pseudo/Virtual). Room code can't be changed after creation. Setup across multiple screens. |
| Mews | Forces "Space Type" terminology. No bulk room creation. |
| Cloudbeds | OTA mapping separate and painful. Limited room-specific overrides. |
| All systems | No visual floor map. Amenities in separate section (often forgotten). No setup completion indicator. No property templates. Photos not validated. |

---

## Our Design Principles

### 1. Property Templates
```
"What best describes your property?"
  ○ Small Guesthouse   → 2 room types pre-loaded, basic amenities
  ○ Business Hotel     → 4 room types, business amenities
  ○ Resort / Luxury    → 5+ types, premium amenities, pool villa option

Hotel edits template — doesn't start from scratch.
80% done by template, 20% customization.
```

### 2. Guided Wizard (Not a Form)
```
Step 1: Property Structure (buildings & floors)
Step 2: Room Types
Step 3: Physical Rooms (bulk creation)
Step 4: Amenities (inline — not separate)
Step 5: Photos
Step 6: Review & Go Live

Progress bar. One concept per step. Validates before moving forward.
```

### 3. Visual Floor Plan Builder
```
FLOOR 1:
┌──────┬──────┬──────┬──────┬──────┐
│ 101  │ 102  │ 103  │ 104  │ 105  │
│ STD  │ STD  │ DLX  │ DLX  │ ACC  │
└──────┴──────┴──────┴──────┴──────┘

Color by room type. Click to edit. Connecting rooms shown visually.
Same map used at check-in for room assignment.
```

### 4. Bulk Room Creation
```
Room Type: Standard Double | Count: [30]
Format: Floor-based
  Floor 1: Rooms 101-110
  Floor 2: Rooms 201-210
  Floor 3: Rooms 301-310
[Generate 30 Rooms] → Done in 10 seconds.
Then mark exceptions individually (no balcony, accessible, connecting).
```

### 5. Amenities Inline
```
During room type setup — amenities selected inline.
Not a separate section. Cannot be missed.
Type-level defaults. Room-level overrides.
```

### 6. Room-Specific Overrides
```
[✎ This room is different from type defaults]
Toggle individual attributes. Inherits everything else from type.
Visual indicator on floor map: "This room has overrides."
```

### 7. Photo Management (Quality Enforced)
```
Upload → Auto-validate resolution, size, aspect ratio
Auto-suggest: brightness enhancement
OTA-ready: Auto-generate OTA-specific versions (different size/format per OTA)
Upload once → Sync everywhere
Minimum 3 photos required before go-live (enforced)
```

### 8. Setup Completion Score
```
Room Setup: 73% complete
  ✅ Room Types: 4/4
  ✅ Physical Rooms: 60/60
  ⚠ Photos: 2/4 types (Suite & Pool Villa missing)
  ⚠ Amenities: 3/4 types (Pool Villa missing)

System blocks OTA connection until minimum 85% complete.
```

### 9. OTA Room Mapping Built-In
```
Our Room Type        Booking.com Room Type
Standard Double  →  [Standard Double Room ▼]
Deluxe Double    →  [Deluxe Double Room ▼]

Booking.com room types pulled via API — dropdown, not typing.
Conflict detection: warns if two types map to same OTA type.
```

### 10. Room Notes Surfaced at Check-in
```
Staff assigns Room 203:
┌─────────────────────────────────────────┐
│ ⚠ ROOM NOTE — Room 203                 │
│ "AC noise — Maintenance #M-456 open"    │
│ [Assign anyway]  [Choose different room]│
└─────────────────────────────────────────┘
Can't assign without seeing the note.
```

### 11. Attribute-Based Selling (ABS) — Phase 3
```
Rooms tagged with attributes: [High Floor] [City View] [Corner] [Quiet Wing]
Guest pays extra for guaranteed attributes — not just room type.
Foundation built in setup. Enabled in Phase 3.
```

---

## Room Type — All Fields

```
id, hotel_id
name                     "Standard Double"
short_code               "STD-DBL"
description              (shown to guests)
max_adults               2
max_children             1
max_occupancy            3
size_sqft                320
bed_type                 KING / QUEEN / TWIN / DOUBLE / BUNK
bed_config_flexible      bool (twin→king convertible)
smoking_policy           SMOKING / NON_SMOKING / BOTH
floor_preference         HIGH / LOW / ANY
base_rate                ₹4,000
display_order            1
is_active                bool
template_source          GUESTHOUSE / BUSINESS / RESORT / CUSTOM
```

---

## Physical Room — All Fields

```
id, hotel_id, room_type_id, floor_id
room_number              "101"
display_name             "101" or "Garden Suite A" (luxury)
smoking_override         nullable (null = inherit from type)
view_type_override       nullable
is_accessible            bool
accessibility_features   JSON (grab_bars, roll_in_shower, wide_door...)
is_active                bool
internal_notes           text (shown at check-in as alert)
housekeeping_zone_id     FK
setup_complete           bool
```

---

## Supporting Entities

```
Building        (id, hotel_id, name, code, display_order)
Floor           (id, building_id, number, name, display_order)
ConnectingRoom  (room_id, connected_room_id, type: DOOR/ADJACENT)
Amenity         (id, hotel_id, name, category, icon)
RoomTypeAmenity (room_type_id, amenity_id, value)
RoomAmenityOverride (room_id, amenity_id, has_amenity, value)
RoomAttribute   (room_id, attribute_type, is_sellable, extra_charge) ← Phase 3
RoomPhoto       (id, room_type_id, url, thumbnail_url, is_cover,
                 display_order, ota_versions JSON)
SetupProgress   (hotel_id, section, completion_percentage, missing_items JSON)
```

---

## Key Relationships

```
Hotel → Buildings → Floors → Rooms → RoomType
RoomType → Amenities (type-level defaults)
Room → AmenityOverrides (room-level exceptions only)
Room ↔ Room (ConnectingRoom — bidirectional)
RoomType → Photos
RoomType → RatePlans (in Rate Setup)
Room → HousekeepingZone (in Housekeeping Setup)
```

---

## Comparison: Existing vs Our System

| Problem | Our Solution |
|---------|-------------|
| Complex terminology | Simple: Room Type → Room |
| Start from scratch | Property templates (80% pre-filled) |
| No visual layout | Interactive floor plan builder |
| Rooms added one by one | Bulk generation with floor-based numbering |
| Amenities separate | Inline in room type setup |
| Room overrides complex | Simple toggle with clear indicator |
| Photos not validated | Quality check + OTA format auto-conversion |
| No completion indicator | Setup score + blocked go-live if incomplete |
| OTA mapping separate | Built-in dropdown + conflict detection |
| Notes never seen at check-in | Alert shown during room assignment |
