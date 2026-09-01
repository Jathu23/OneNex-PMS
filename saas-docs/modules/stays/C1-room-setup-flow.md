# C1 — Room Setup: Owner Journey & System Flow
> Stays Module | Core Feature 1 of 9
> Purpose: How a business owner sets up rooms — step by step.
> What they see → What they do → What the system does → What API is needed.

---

## Who Is Doing This

```
Business Owner (or Manager with setup permission)
  → First time setting up Stays operation on OneNex
  → OR returning to update room configuration

Trigger: Owner enabled "Stays" operation on their business.
         System auto-creates business_operation record.
         Room Setup screen appears as first mandatory step.
```

---

## The Full Flow at a Glance

```
Step 0: Owner enables Stays
        ↓
Step 1: Choose Property Template
        ↓
Step 2: Setup Property Structure (Buildings + Floors)
        ↓
Step 3: Create Room Types
        ↓
Step 4: Add Physical Rooms
        ↓
Step 5: Configure Amenities per Room Type
        ↓
Step 6: Upload Photos per Room Type
        ↓
Step 7: Mark Connecting Rooms (optional)
        ↓
Step 8: Review Setup Score → Go Live
```

Each step: owner cannot skip if previous step is incomplete.
Progress bar shown at top. Can save and come back anytime.

---

## Step 0 — Owner Enables Stays

**What owner does:**
Settings → Operations → Enable "Stays"

**What system does:**
```
1. Creates business_operation record:
     business_id: 123
     operation_type: STAYS
     status: SETUP_INCOMPLETE

2. Creates RoomSetupProgress record:
     hotel_id: 123
     overall_pct: 0
     (all counts = 0)

3. Auto-creates 1 Building record:
     name: "Main Building" (editable)
     code: "MB"
     hotel_id: 123

4. Redirects owner to Room Setup wizard.
```

**APIs needed:**
```
POST /operations/stays/enable
  → Creates business_operation + RoomSetupProgress + default Building
```

---

## Step 1 — Choose Property Template

**What owner sees:**
```
"What best describes your property?"

  ○ Small Guesthouse    → up to 20 rooms, simple setup
  ○ Business Hotel      → 20-100 rooms, corporate amenities
  ○ Resort / Luxury     → 100+ rooms, premium amenities, pool villas
  ○ Start from scratch  → I'll set up everything manually
```

**What owner does:**
Clicks one option. Hits "Continue."

**What system does:**
```
Based on selection, system pre-creates:

SMALL GUESTHOUSE template:
  RoomTypes created:
    → "Standard Room"  (max 2 adults, QUEEN bed, base_rate: 0)
    → "Deluxe Room"    (max 2 adults, KING bed, base_rate: 0)
  Amenities pre-loaded:
    → AC, WiFi, Hot Water, TV, In-room Safe (8 basic amenities)
  RoomTypeAmenity created:
    → Standard Room: AC=true, WiFi=true, Hot Water=true, TV=true
    → Deluxe Room:   AC=true, WiFi=true, Hot Water=true, TV=true, Mini Bar=true

BUSINESS HOTEL template:
  RoomTypes: Standard, Deluxe, Suite (3 types)
  Amenities: 20 pre-loaded (business-focused: desk, USB ports, iron...)

RESORT template:
  RoomTypes: Standard, Deluxe, Suite, Pool Villa (4-5 types)
  Amenities: 35 pre-loaded (premium: jacuzzi, rainshower, minibar...)

Owner edits these — doesn't start from zero.

START FROM SCRATCH selected:
  → NO RoomTypes created
  → NO RoomTypeAmenity records created
  → Default Building still exists (created in Step 0)
  → Amenity master list still pre-loaded (all hotels need the master list)
  → Owner goes to Step 2 with blank slate
  → Step 3: owner creates every RoomType manually from empty form
  → Step 5: owner picks amenities from master list for each type manually
  Use case: boutique property with unique room names that don't fit any template.
```

**APIs needed:**
```
GET  /room-setup/templates
  → Returns list of available templates with preview

POST /room-setup/apply-template
  body: { template: "GUESTHOUSE" }
  → Creates RoomTypes + Amenities + RoomTypeAmenity records
  → Updates RoomSetupProgress
```

---

## Step 2 — Property Structure (Buildings & Floors)

**What owner sees:**
```
Single building hotel (most common):
  Building section hidden entirely.
  Owner only sees:

  "How many floors does your property have?"
  Floors: [ 3 ] ← number input

  [Generate Floors] → Floor 1, Floor 2, Floor 3 created.

Multi-building (if owner clicks "Add another building"):
  Building A: [Main Block      ] Code: [MB]
  Building B: [Pool Wing       ] Code: [PW]

  Each building: floors configured separately.
```

**What owner does:**
Enters floor count (or adds buildings). Hits "Continue."

**What system does:**
```
Creates Floor records:
  floor_number: 1, building_id: MB, hotel_id: 123
  floor_number: 2, building_id: MB, hotel_id: 123
  floor_number: 3, building_id: MB, hotel_id: 123

If multi-building:
  Creates Building records first, then Floor records per building.

Updates RoomSetupProgress.
```

**APIs needed:**
```
GET  /room-setup/buildings
  → Returns buildings + floors for this hotel

POST /room-setup/buildings
  body: { name, code }
  → Creates Building

POST /room-setup/floors
  body: { building_id, floor_count }
  → Bulk creates Floor records

PATCH /room-setup/buildings/{id}
  → Edit building name / code

DELETE /room-setup/floors/{id}
  → Remove a floor (only if no rooms assigned)
```

---

## Step 3 — Create Room Types

**What owner sees:**
```
[Template room types already shown — owner edits them]

┌──────────────────────────────────────────────────────────────┐
│ Room Type 1: Standard Room              Code: [STD-DBL]  [✎] │
│                                                              │
│ BASIC                                                        │
│   Category:    [BEDROOM ▼]    Bed: [QUEEN ▼]                │
│   Bed flexible (Twin→King?): [No ▼]                         │
│   Max Adults:  [2]  Max Children: [1]  Max Total: [3]       │
│   Size: [320] sqft (optional)                               │
│                                                              │
│ GUEST-FACING                                                 │
│   Base Rate: LKR [12,000] /night  ← reference only          │
│   Description: [Comfortable room with queen bed...]          │
│   Display Name: [           ] ← luxury named rooms only     │
│                                                              │
│ POLICY                                                       │
│   View: [NONE ▼]  Smoking: [NON_SMOKING ▼]                  │
│   Pet Friendly: [No ▼]  Accessible: [No ▼]                  │
└──────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│ Room Type 2: Deluxe Room                       [✎] │
│   ...                                               │
└─────────────────────────────────────────────────────┘

[+ Add Room Type]
```

**What owner does:**
- Edits pre-filled room types (name, rate, bed type, description...)
- Sets `short_code` (auto-suggested from name: "Standard Room" → "STD-RM")
- Sets `display_name` only for luxury named rooms ("The Presidential Villa")
- Sets `bed_config_flexible` if TWIN bed can convert to KING
- Marks `is_accessible = Yes` for wheelchair-accessible room types
- Adds new types if template doesn't cover all
- Hits "Continue" when all types look correct

**When owner marks Accessible = Yes:**
```
System behavior:
  → RoomType.is_accessible = true
  → In Step 5 (Amenities): ACCESSIBILITY category auto-expands for this type
  → Owner prompted: "Mark the specific accessibility features available"
  → Owner checks: Grab Bars, Roll-in Shower, Wide Doorway etc.
  → These are stored as RoomTypeAmenity (ACCESSIBILITY category)
  → OTA filter "Accessible rooms" uses RoomType.is_accessible = true
  → Specific features synced via ota_mapping_code on each amenity (Channel Manager)
```

**System validation before Continue:**
```
Each RoomType must have:
  ✅ name
  ✅ bed_type
  ✅ max_adults
  ✅ max_occupancy

Missing any → inline error shown. Cannot proceed.
base_rate = 0 allowed (rate plans set actual billing rate).
```

**What system does:**
```
PATCH /room-types/{id} called on every field change (auto-save).
No "Save" button — changes persist immediately.
Owner can close browser and return — progress saved.

Updates RoomSetupProgress.room_types_complete.
```

**APIs needed:**
```
GET    /room-types
  → Returns all RoomTypes for this hotel

POST   /room-types
  body: { name, short_code, description, display_name,
          room_category, bed_type, bed_config_flexible,
          max_adults, max_children, max_occupancy,
          size_sqft, smoking_policy, is_pet_friendly,
          is_accessible, view_type, base_rate, display_order }
  → Creates new RoomType

PATCH  /room-types/{id}
  body: { any field }
  → Updates RoomType (auto-save on blur)

DELETE /room-types/{id}
  → Only allowed if no physical rooms of this type exist
  → If rooms exist: error "Remove rooms of this type first"

PATCH  /room-types/reorder
  body: { ordered_ids: [3, 1, 2] }
  → Updates display_order (drag-drop reorder in UI)
```

---

## Step 4 — Add Physical Rooms

**What owner sees:**
```
"Now let's add your actual room numbers."

QUICK ADD (most owners use this):
  Room Type: [Standard Room ▼]
  How many rooms?: [10]
  Starting floor: [Floor 1 ▼]
  Numbering format: [Floor-based: 101, 102... ▼]

  [Generate 10 Rooms] → Done in 1 second.

Then owner sees floor map:

FLOOR 1:
┌──────┬──────┬──────┬──────┬──────┐
│ 101  │ 102  │ 103  │ 104  │ 105  │
│ STD  │ STD  │ STD  │ STD  │ STD  │
└──────┴──────┴──────┴──────┴──────┘

Click any room → edit that room's details.
```

**What owner does:**
- Uses Quick Add to bulk-generate rooms
- Clicks individual rooms to mark exceptions:
  - "Room 104 is accessible"
  - "Room 101 has garden view (not ocean like other STD rooms)"
  - "Room 203: AC noise — note for staff"
  - "Rooms 201-202 are connecting"

**What system does:**
```
Bulk creation:
  Creates Room records:
    room_number: 101, room_type_id: STD, floor_id: F1, hotel_id: 123
    room_number: 102, room_type_id: STD, floor_id: F1, hotel_id: 123
    ... (10 records)

Individual room override (owner clicks room 104):
  PATCH /rooms/104
    { is_accessible_override: true }

  PATCH /rooms/101
    { view_type_override: "GARDEN" }

  PATCH /rooms/203
    { internal_notes: "AC noise in left corner — maintenance aware" }

Connecting rooms (owner clicks 201 then 202, "Mark as connecting"):
  POST /connecting-rooms
    { room_id: 201, connected_room_id: 202, connection_type: "CONNECTING_DOOR" }
  System auto-creates reverse record too (202 ↔ 201).

Updates RoomSetupProgress.rooms_complete.
```

**APIs needed:**
```
POST   /rooms/bulk-create
  body: { room_type_id, floor_id, count, numbering_format, start_number }
  → Creates multiple Room records in one call

GET    /rooms
  query: { floor_id, room_type_id }
  → Returns rooms for floor map display

GET    /rooms/{id}
  → Single room details + effective amenities (type + overrides merged)

PATCH  /rooms/{id}
  body: { any override field, internal_notes, is_accessible_override,
          view_type_override, smoking_override, is_pet_friendly_override }
  → Updates individual room

DELETE /rooms/{id}
  → Remove room (only if no active bookings)

POST   /connecting-rooms
  body: { room_id, connected_room_id, connection_type }
  → Creates connecting pair (system auto-creates reverse record)

DELETE /connecting-rooms
  body: { room_id, connected_room_id }
  → Removes connection (removes both directions)
```

---

## Step 5 — Configure Amenities per Room Type

**What owner sees:**
```
[Tab: Standard Room] [Tab: Deluxe Room] [Tab: Suite]

STANDARD ROOM — Amenities

COMFORT          TECH            BATHROOM
[✅] AC          [✅] WiFi       [✅] Hot Water
[  ] Heating     [✅] Smart TV   [  ] Bathtub
[  ] Fan         [  ] USB Ports  [  ] Rainshower

FOOD             SAFETY          ACCESSIBILITY
[  ] Mini Bar    [✅] In-room Safe [  ] Grab Bars
[  ] Coffee M.   [✅] Smoke Det.  [  ] Roll-in Shower

WiFi value: [50 Mbps     ] ← extra detail field (optional)
TV value:   [55 inch     ]
```

**What owner does:**
- Checks/unchecks amenities per room type
- Adds optional value for specific amenities (WiFi speed, TV size)
- Switches tabs to configure each room type

**What system does:**
```
Every toggle = PATCH /room-type-amenities (auto-save)

Toggle WiFi ON for Standard Room:
  Upsert RoomTypeAmenity:
    room_type_id: STD, amenity_id: WIFI, has_amenity: true, value: "50 Mbps"

Toggle Mini Bar OFF for Standard Room:
  Upsert RoomTypeAmenity:
    room_type_id: STD, amenity_id: MINIBAR, has_amenity: false, value: null

Why upsert (not insert):
  Record may already exist from template.
  Toggle again → update existing record. No duplicates.

Updates RoomSetupProgress.amenities_types_done.
```

**APIs needed:**
```
GET    /amenities
  query: { category }
  → Returns all amenities for this hotel (master list)

GET    /room-type-amenities/{room_type_id}
  → Returns all amenity configs for this room type

PATCH  /room-type-amenities
  body: { room_type_id, amenity_id, has_amenity, value }
  → Upsert — create or update amenity config for this type

POST   /amenities
  body: { name, category, icon }
  → Owner adds a custom amenity not in master list

GET    /room-amenity-overrides/{room_id}
  → Returns amenity exceptions for a specific room (Phase 2)

PATCH  /room-amenity-overrides
  body: { room_id, amenity_id, has_amenity, value, reason }
  → Room-specific override (Phase 2)
```

**Note — `ota_mapping_code` on Amenity:**
```
Owner does NOT set ota_mapping_code during Room Setup.
It is populated when Channel Manager (A1) add-on is enabled.
Channel Manager setup screen: each amenity → map to OTA-specific code.
Room Setup wizard has no awareness of OTA codes — that is A1's concern.
```

**Note — Pet Policy 3 Levels:**
```
Level 1 (Room Type)     → set in Step 3: is_pet_friendly on RoomType
Level 2 (Room override) → set in Step 4: is_pet_friendly_override on Room
Level 3 (Rate Plan)     → NOT in Room Setup.
  Configured in C7 Rate Plans: pet_fee, pet_types_allowed, max_pets.
  Room Setup = eligibility (which rooms allow pets).
  Rate Plans = billing rules (how much, which pet types, max count).
```

**Note — Multi-language Room Type Names:**
```
V1: English only.
    RoomTypeTranslation auto-created (language_code="en") from RoomType.name.
    Owner does nothing extra.

Phase 2: "Add translations" button appears in Step 3.
    Owner adds Sinhala + Tamil name per room type.
    Stored in RoomTypeTranslation table.
    OTA listings served in guest's language automatically.
```

---

## Step 6 — Upload Photos per Room Type

**What owner sees:**
```
[Tab: Standard Room] [Tab: Deluxe Room] [Tab: Suite]

STANDARD ROOM — Photos

[  Drop photos here or click to upload  ]
Min 3 photos required before going live.

Uploaded:
┌──────────┬──────────┬──────────┐
│  [photo] │  [photo] │  [photo] │
│ INTERIOR │ BATHROOM │   VIEW   │ ← photo_type label
│  ★ Cover │          │          │
└──────────┴──────────┴──────────┘
  [Set as cover] [Delete]

⚠ Suite: 0 photos — required before go-live
```

**What owner does:**
- Drags photos into upload area
- Selects photo_type for each photo (INTERIOR / BATHROOM / VIEW...)
- Marks one photo as cover (main listing photo)
- Switches tabs to upload for each room type

**What system does:**
```
On upload:
  1. Validate: resolution ≥ 1024×768, size ≤ 10MB, format JPG/PNG/WEBP
  2. Store original to cloud storage → get url
  3. Generate thumbnail → thumbnail_url
  4. Queue background job: generate OTA-specific versions
     (booking_com 720×540, expedia 800×533, airbnb 1024×683)
  5. Create RoomPhoto record:
     room_type_id: STD, url: "...", thumbnail_url: "...",
     photo_type: INTERIOR, is_cover: false, display_order: 3

  If is_cover = true:
     First set all existing photos for this type: is_cover = false
     Then set this photo: is_cover = true

Updates RoomSetupProgress.photos_types_done.
```

**APIs needed:**
```
POST   /room-photos/upload
  body: multipart/form-data { room_type_id, file, photo_type }
  → Validates + uploads + creates RoomPhoto record
  → Returns: { photo_id, url, thumbnail_url, status: "processing" }
  → OTA versions generated async (background job)

GET    /room-photos/{room_type_id}
  → Returns all photos for this room type

PATCH  /room-photos/{photo_id}
  body: { photo_type, caption, is_cover, display_order }
  → Update photo metadata

PATCH  /room-photos/{room_type_id}/reorder
  body: { ordered_ids: [3, 1, 2] }
  → Update display_order for carousel

DELETE /room-photos/{photo_id}
  → Remove photo
  → If it was cover: system auto-promotes next photo as cover

GET    /room-photos/{photo_id}/ota-status
  → Check if OTA versions generated yet
  → { booking_com: "ready", expedia: "processing", airbnb: "ready" }
```

---

## Step 7 — Mark Connecting Rooms (Optional)

**What owner sees:**
```
"Do you have rooms with connecting doors?"

[Yes, set up connecting rooms]  [Skip for now]

If Yes:

FLOOR 1:
┌──────┬──────┬──────┬──────┐
│ 101  │ 102  │ 103  │ 104  │
└──────┴──────┴──────┴──────┘

Click 101, then click 102:
"Connect these rooms?"
  ○ Connecting Door (can walk between rooms)
  ○ Adjacent Only (side by side, no door)
[Confirm]

201 ↔ 202 now shows a link icon between them on floor map.
```

**What system does:**
```
POST /connecting-rooms
  { room_id: 101, connected_room_id: 102, connection_type: "CONNECTING_DOOR" }

System creates 2 records:
  room_id: 101, connected_room_id: 102
  room_id: 102, connected_room_id: 101
(Bidirectional — query from either room works)
```

APIs: already defined in Step 4.

---

## Step 8 — Review Setup Score & Go Live

**What owner sees:**
```
Room Setup Complete!

Overall Score: 87%

  ✅ Room Types:      4 / 4   (30 pts)
  ✅ Physical Rooms:  42 / 42 (30 pts)
  ✅ Photos:          3 / 4   (19 pts)  ← Suite missing 1 photo
  ✅ Amenities:       4 / 4   (15 pts)  ← but Suite has only 2 amenities

  Score 87% — minimum 80% met ✅

[Connect OTA Channels]   → Available (score ≥ 80%)
[Go Live - Accept Bookings] → Available

Recommendation:
"Add more photos for Suite to improve your OTA listing quality."
```

**What system does:**
```
Recalculates RoomSetupProgress.overall_pct:
  room_types_complete: 4/4  → 30 × (4/4)  = 30 pts
  rooms_complete:     42/42  → 30 × (42/42) = 30 pts
  photos_types_done:   3/4  → 25 × (3/4)  = 18.75 pts
  amenities_types_done: 4/4 → 15 × (4/4)  = 15 pts
  overall_pct = 93.75% → rounds to 87% (weighted)

If score < 80%:
  [Connect OTA] → BLOCKED
  [Go Live]     → BLOCKED
  Shows exactly what is missing.

If score ≥ 80%:
  Updates business_operation.status: SETUP_COMPLETE
  Go Live unlocked.
```

**APIs needed:**
```
GET    /room-setup/progress
  → Returns RoomSetupProgress with breakdown
  → { overall_pct, room_types, rooms, photos, amenities, missing_items[] }

POST   /room-setup/complete
  → Marks setup as done
  → Updates business_operation.status = SETUP_COMPLETE
  → Triggers: "Room Setup Complete" notification to owner
```

---

## Complete API Surface — Summary

```
BUILDINGS & FLOORS
  POST   /room-setup/apply-template
  GET    /room-setup/buildings
  POST   /room-setup/buildings
  PATCH  /room-setup/buildings/{id}
  POST   /room-setup/floors
  DELETE /room-setup/floors/{id}

ROOM TYPES
  GET    /room-types
  POST   /room-types
  PATCH  /room-types/{id}
  DELETE /room-types/{id}
  PATCH  /room-types/reorder

ROOMS
  POST   /rooms/bulk-create
  GET    /rooms
  GET    /rooms/{id}
  PATCH  /rooms/{id}
  DELETE /rooms/{id}

CONNECTING ROOMS
  POST   /connecting-rooms
  DELETE /connecting-rooms

AMENITIES
  GET    /amenities
  POST   /amenities
  GET    /room-type-amenities/{room_type_id}
  PATCH  /room-type-amenities
  GET    /room-amenity-overrides/{room_id}     ← Phase 2
  PATCH  /room-amenity-overrides               ← Phase 2

PHOTOS
  POST   /room-photos/upload
  GET    /room-photos/{room_type_id}
  PATCH  /room-photos/{photo_id}
  PATCH  /room-photos/{room_type_id}/reorder
  DELETE /room-photos/{photo_id}
  GET    /room-photos/{photo_id}/ota-status

PROGRESS & GO LIVE
  GET    /room-setup/progress
  POST   /room-setup/complete
```

---

## Ongoing Operation — Marking a Room Out of Order (OOO)

This is NOT part of initial setup. Happens day-to-day during hotel operations.

**What staff does:**
```
Rooms → Floor Map → Click Room 203 → "Mark Out of Order"

Reason: [Bathroom tile cracked — plumber booked Dec 2]
Duration: From [Dec 1] To [Dec 5]
```

**V1 behavior:**
```
Staff sets Room.is_active = false.
Room disappears from availability immediately.
System does NOT auto-release — staff must manually set is_active = true on Dec 6.
Simple. Suitable for small hotels with discipline.
```

**Phase 2 behavior (RoomBlock):**
```
System creates RoomBlock record:
  room_id: 203
  block_type: OOO_MAINTENANCE
  start_date: Dec 1
  end_date: Dec 5
  reason_note: "Bathroom tile cracked — plumber booked Dec 2"

Dec 6 00:01: system auto-releases. Room available immediately.
Zero manual action. Full audit trail.
If Maintenance (A9) add-on enabled: links to work order ticket.
When ticket marked COMPLETE → RoomBlock auto-releases early.
```

**APIs needed (V1):**
```
PATCH /rooms/{id}
  body: { is_active: false }
  → Marks room permanently inactive (OOO in V1)

PATCH /rooms/{id}
  body: { is_active: true }
  → Restores room to active

Note: same PATCH /rooms/{id} used — no separate OOO endpoint in V1.
```

**APIs needed (Phase 2):**
```
POST   /room-blocks
  body: { room_id, block_type, start_date, end_date, reason_note }
  → Creates date-ranged OOO block

GET    /room-blocks
  query: { room_id, active_only: true }
  → Returns active blocks for a room or all rooms

PATCH  /room-blocks/{id}
  body: { end_date, is_active }
  → Extend, shorten, or cancel a block early

DELETE /room-blocks/{id}
  → Cancel block immediately (room returns to available)
```

---

## Key Rules the System Enforces

```
1. Cannot create Room without RoomType existing
2. Cannot create RoomType without Building + Floor existing
3. Cannot delete RoomType if Rooms of that type exist
4. Cannot delete Room if active/future bookings exist
5. Cannot proceed to Rate Plans (C7) if room_types_complete = 0
6. Cannot connect OTA if overall_pct < 80%
7. ConnectingRoom always creates bidirectional pair
8. RoomPhoto cover change: auto-demotes previous cover
9. Photo upload: OTA versions generated async (not blocking the owner)
10. All changes auto-save — no "Save" button anywhere in setup
```
