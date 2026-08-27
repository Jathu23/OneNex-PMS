# Room Setup — Design Document
> Hotel Module | Setup & Configuration | Area 2
> Version: 1.0 | For Team Review
> Validated against: Apaleo, Oracle OPERA, Mews, Cloudbeds, HTNG Standard, Booking.com API, Expedia API

---

## 1. The Problem We Are Solving

Every hotel operation depends on one question the system must answer correctly:

```
"What rooms do you have, and which ones are available right now?"
```

If the system doesn't know the answer precisely:

```
Wrong room setup → Wrong availability → Double booking
Wrong room setup → Wrong pricing → Revenue loss
Wrong room setup → Wrong OTA listing → Guest complaint
Wrong room setup → Wrong housekeeping assignment → Room not ready
Wrong room setup → Wrong billing → Tax invoice error
```

Room Setup is not just a configuration screen.
It is the foundation every other module depends on.

---

## 2. Our Design Philosophy

```
"Simple by Default. Powerful when needed."
```

This means one system must serve two very different users:

```
USER A: Small Guesthouse Owner
  → 12 rooms, 2 room types
  → Not tech-savvy
  → Wants to go live in 30 minutes
  → Should never see fields they don't need

USER B: Luxury Resort Manager
  → 200 rooms, 8 room types, 3 buildings
  → Complex OTA connections
  → Pet-friendly rooms, connecting suites, accessible rooms
  → Needs full control of every detail
```

Same system. Same entities. Different depth of usage.

```
SMALL GUESTHOUSE:                    LUXURY RESORT:
Fill 4 fields → done.                Fill 30+ fields → full control.

RoomType: name, bed_type,            RoomType: name, code, description,
          max_adults, base_rate                 category, view_type,
                                               bed_type, flexible config,
Room: room_number, floor,                      occupancy, size, smoking,
      room_type                                amenities, translations,
                                               photos, OTA mapping

System hides everything else         System shows everything
```

---

## 3. Industry Validation

Before designing, we validated against real PMS systems and industry standards.

| System / Standard | Approach | Match with Our Design |
|---|---|---|
| Apaleo | "Unit Groups" + "Units" (two-tier) | ✅ Exact match |
| Oracle OPERA | Room Type + Room Number (two-tier) | ✅ Exact match |
| Mews | Space Types + Spaces (two-tier) | ✅ Exact match |
| Cloudbeds | Room Types + Rooms (two-tier) | ✅ Exact match |
| HTNG Standard | Room Type Mapping specification | ✅ Fields aligned |
| Booking.com API | Room type code, occupancy, bed, amenities | ✅ All required fields present |
| Expedia API | Room type + rate plan association | ✅ Covered |

**Conclusion:** The two-tier model (RoomType + PhysicalRoom) is the global industry standard. Our design is aligned with it.

---

## 4. The Two-Tier Model — Core Concept

### Level 1: Room Type (Template / Category)

```
"What KINDS of rooms do we have?"

Room Type = a mold. Define once. Apply to many rooms.

Example: "Standard Double"
  → Max 2 adults, 1 child
  → Queen bed
  → 320 sq ft
  → Non-smoking
  → Amenities: AC, TV, WiFi, Hot water
  → Base rate: LKR 12,000/night
  → Photos: [3 photos of this room category]

Every Standard Double room inherits ALL of this automatically.
Change one thing on the type → all rooms of that type update.
```

### Level 2: Physical Room (Actual Room Number)

```
"What are the ACTUAL rooms on each floor?"

Physical Room = one real room with a number on the door.

Example: Room 101
  → Type: Standard Double (inherits everything from type)
  → Floor: 1, Building: Main Block
  → Housekeeping Zone: Zone A
  → Notes: "AC noise — maintenance ticket open" (staff alert at check-in)
  → Override: View = Garden View (this specific room faces garden)

Room 102 (same type, same floor — but no override):
  → Inherits everything from Standard Double type
  → Nothing different to store
```

### The Inheritance Rule

```
QUERY: "What does Room 305 offer?"

1. Get Room 305 → room_type = Deluxe
2. Get all Deluxe room type defaults
3. Check: does Room 305 have any overrides?
4. Override wins if exists. Type default otherwise.

This means:
  → 28 Standard Double rooms with no overrides = ZERO extra storage
  → Only exceptions (2 garden view rooms) need override records
  → Changing base rate on type = all 28 rooms updated instantly
```

---

## 5. Property Structure

Hotels are structured in a hierarchy. Our system mirrors this exactly.

```
HOTEL
  └── BUILDING          (Main Block, Pool Wing, Garden Villas)
        └── FLOOR        (Floor 1, Floor 2, Executive Floor)
              └── ROOM   (Room 101, Room 102, Garden Villa A)
                    │
                    └──→ ROOM TYPE  (Standard Double, Suite, Villa)
```

**Single building hotel** (most common):
```
One building auto-created. Floor and building concept hidden from UI.
Hotel owner sees: Floors → Rooms directly.
```

**Multi-building resort:**
```
Owner sees: Building selector → Floor → Rooms.
Floor map shows per building.
```

---

## 6. All 11 Entities — Purpose and Fields

---

### Entity 1: `Building`

**Purpose:** Represents a physical structure within the hotel property.
Single building hotels: auto-created, hidden from UI.
Multi-building properties: each building configured separately.

**Industry alignment:** Standard in all enterprise PMS systems for resort properties.

```
id                → Unique identifier
hotel_id          → Which hotel (multi-tenant isolation)
name              → "Main Block" / "Pool Wing" / "Garden Villas"
code              → "MB" / "PW" (used in room numbering: PW-01, PW-02)
display_order     → Controls order on floor map
created_at        → When created
updated_at        → Last modified
created_by        → Staff who created (audit trail)
updated_by        → Staff who last modified
```

**Simple / Advanced:**
```
Simple hotel:  1 auto-created building. Owner never sees this entity.
Resort:        Full building management. Each building named and mapped.
```

---

### Entity 2: `Floor`

**Purpose:** Floors within a building. Room numbers are floor-based.
Enables housekeeping zone assignment, high floor preference logic, and floor map display.

**Industry alignment:** Universal in all PMS systems.

```
id                → Unique identifier
hotel_id          → Multi-tenant isolation
building_id       → Which building (FK → Building)
floor_number      → 1, 2, 3... (or 0 = ground, -1 = basement)
floor_name        → nullable. "Executive Floor" / "Rooftop Level"
                    Simple hotels: null (floor_number used)
                    Luxury hotels: named floors for branding
display_order     → Display order on floor map
created_at / updated_at / created_by / updated_by
```

**Why floor_name is nullable:**
```
Simple hotel: "Floor 1", "Floor 2" (auto from floor_number)
Luxury hotel: "The Residence Level", "The Presidential Floor"
```

---

### Entity 3: `RoomType`

**Purpose:** The most important entity. Template that defines an entire category of rooms.
Change one field → all rooms of that type update instantly.

**Industry alignment:** Called "Unit Groups" in Apaleo, "Space Types" in Mews, "Room Types" in OPERA.
All required fields validated against Booking.com and Expedia API requirements.

```
id                    → Unique identifier
hotel_id              → Multi-tenant isolation

-- IDENTITY --
name                  → "Standard Double" (guest-facing, OTA listing)
short_code            → "STD-DBL" (internal: floor map, reports, housekeeping)
description           → Guest-facing text (booking widget, OTA description)
room_category         → BEDROOM / SUITE / VILLA / APARTMENT / DORMITORY
                        Industry standard (Apaleo): defines fundamental type
                        OTA uses this for category filters ("Suites" tab)

-- OCCUPANCY --
max_adults            → Maximum adult guests
max_children          → Maximum children (separate: different pricing)
max_occupancy         → Total people allowed (adults + children combined)
                        OTA requires all 3 for correct availability display

-- PHYSICAL --
size_sqft             → nullable. Room size. OTA listing shows this.
bed_type              → KING / QUEEN / TWIN / DOUBLE / BUNK
bed_config_flexible   → bool. Can TWIN convert to KING?
                        If true: room offered as both twin and king option

-- POLICY --
smoking_policy        → SMOKING / NON_SMOKING / BOTH
                        Type-level default. Room can override.
is_pet_friendly       → bool. Does this room category allow pets?
                        Pet bookings filtered to pet-friendly rooms only.
view_type             → CITY / OCEAN / GARDEN / POOL / MOUNTAIN / NONE
                        Industry standard field (HTNG).
                        OTA "view" filter uses this specifically.
                        (NOT an amenity — dedicated field for OTA compliance)

-- PRICING --
base_rate             → Default starting rate (LKR)
                        Rate Plans reference this as base

-- DISPLAY --
display_order         → Booking widget order (cheapest first or custom)
is_active             → false = category hidden everywhere (renovation, removal)

-- AUDIT --
created_at / updated_at / created_by / updated_by
```

**Simple / Advanced:**
```
Simple hotel: name, bed_type, max_adults, base_rate → done in 2 minutes
Luxury hotel: all fields, translations, ABS attributes, multiple rate plans
```

---

### Entity 4: `Room` (Physical Room)

**Purpose:** One record = one real room with a number on the door.
Inherits everything from RoomType. Stores ONLY what differs from its type.

**Industry alignment:** Called "Units" in Apaleo, "Spaces" in Mews, "Room Number" in OPERA.

```
id                        → Unique identifier
hotel_id                  → Multi-tenant isolation

-- LOCATION --
room_type_id              → FK → RoomType (inheritance link — most important field)
floor_id                  → FK → Floor
building_id               → FK → Building (denormalized for query speed)

-- IDENTITY --
room_number               → "101" / "PW-01" / "Penthouse" (unique per hotel)
display_name              → nullable. "The Garden Suite" / "The Presidential Villa"
                            null = room_number used everywhere
                            Set only for luxury named rooms

-- OVERRIDES (null = inherit from type) --
smoking_override          → nullable bool
is_pet_friendly_override  → nullable bool
                            null = inherit from RoomType.is_pet_friendly
                            true = this specific room allows pets
                            false = this specific room does NOT allow pets

-- ACCESSIBILITY --
is_accessible             → bool. Wheelchair accessible?
accessibility_features    → JSON nullable
                            ["grab_bars", "roll_in_shower", "wide_door",
                             "lowered_bed", "visual_alerts"]
                            Detailed features for guests with specific needs

-- OPERATIONS --
internal_notes            → text nullable. STAFF ONLY. Guest never sees.
                            "AC noise in left corner"
                            "Connecting door to 102 — lock before assignment"
                            Shown as ALERT popup during room assignment at front desk.
                            Cannot assign room without seeing the note.

housekeeping_zone_id      → FK → HousekeepingZone
                            "Room 101 → Zone A → Meena's responsibility"
                            Housekeeping task auto-assigned based on this

is_active                 → bool. false = room not bookable (OOO permanently)
                            Different from RoomType.is_active:
                              Type false = entire category unavailable
                              Room false = only this room unavailable

-- AUDIT --
created_at / updated_at / created_by / updated_by
```

---

### Entity 5: `Amenity`

**Purpose:** Master list of all amenities this hotel offers. Defined once. Reused across all room types.
Like a dictionary — define a word once, reference it everywhere.

**Industry alignment:** Amenity master lists are standard in all PMS and OTA APIs.

```
id
hotel_id              → Each hotel has their own amenity list
name                  → "Air Conditioning" / "Rainshower" / "Nespresso Machine"
short_code            → "AC" / "RAIN" / "NESPRESSO" (floor map, reports)
category              → COMFORT / TECH / BATHROOM / VIEW / FOOD / SAFETY / OUTDOOR / PET
icon                  → "snowflake" / "wifi" / "paw" (icon library name)
                        Shown as icon on booking widget — visual, scannable
is_active             → bool. Soft delete — preserve history
created_at / updated_at / created_by / updated_by
```

**Standard amenities (pre-loaded by property template):**
```
COMFORT:   Air Conditioning, Heating, Fan, Blackout Curtains
TECH:      Smart TV, WiFi, Bluetooth Speaker, USB Charging Ports
BATHROOM:  Hot Water, Bathtub, Rainshower, Jacuzzi, Hair Dryer
VIEW:      Sea View, Garden View, Pool View, City View, Mountain View
FOOD:      Mini Bar, Coffee Machine, Kitchenette, Fruit Basket
SAFETY:    In-room Safe, Smoke Detector, Fire Extinguisher
PET:       Pet Friendly, Pet Bed, Pet Bowl
```

---

### Entity 6: `RoomTypeAmenity`

**Purpose:** Defines which amenities each room TYPE has as defaults.
Junction table: links RoomType to Amenity.

**Why explicit YES and NO (not just skip the record):**
```
OTA API asks: "Does Standard Double have TV?" → needs explicit YES or NO.
Absence of record ≠ NO. Explicit has_amenity = false = NO.
```

```
room_type_id      FK → RoomType   ↘
amenity_id        FK → Amenity    → Composite primary key
has_amenity       → bool. YES or NO for this amenity on this type
value             → text nullable. Extra detail:
                    WiFi → "100 Mbps"
                    TV → "55 inch Smart TV"
                    AC → null (yes/no enough)
                    Shown at booking: "WiFi — 100 Mbps"
```

**Example records:**
```
room_type       amenity         has_amenity   value
────────────────────────────────────────────────────
Standard Double AC              true          null
Standard Double Mini Bar        false         null
Standard Double WiFi            true          "50 Mbps"
Standard Double Pet Friendly    false         null
Deluxe          AC              true          null
Deluxe          Mini Bar        true          null
Deluxe          WiFi            true          "100 Mbps"
Deluxe          Pet Friendly    false         null
Garden Suite    Pet Friendly    true          null
```

---

### Entity 7: `RoomAmenityOverride`

**Purpose:** Room-specific exception from type default.
Stored ONLY when ONE specific room differs from its type.
Zero records for rooms with no exceptions → clean, efficient.

```
room_id           FK → Room    ↘
amenity_id        FK → Amenity → Composite primary key
has_amenity       → Override value. Replaces type default for this room only.
value             → Override detail ("50 Mbps" when type has "100 Mbps")
reason            → text nullable. Internal. Why is this room different?
                    "Balcony sealed — safety repair pending"
                    "WiFi extender not installed in this corner"
                    Staff understands WHY, not just WHAT.
```

**Query logic (runs at booking, OTA sync, checklist):**
```
"What amenities does Room 305 have?"

1. Get all RoomTypeAmenity for Room 305's type
2. Get all RoomAmenityOverride for Room 305
3. For each amenity:
     IF override record exists → use override value
     ELSE → use type default
```

---

### Entity 8: `RoomPhoto`

**Purpose:** Photos for each room type. Upload once → system auto-generates all OTA-specific format/size versions.

**Why photos belong to RoomType, not Room:**
```
Photos show the CATEGORY of room — not specific room 101.
All Standard Double rooms are represented by the same photos.
Exception (Phase 3): individual room photo for ABS selling.
```

```
id
room_type_id      → FK → RoomType
url               → Original high-resolution upload (cloud storage)
thumbnail_url     → Auto-generated small version (fast loading)
is_cover          → bool. Main/first photo shown. One per room type.
display_order     → Carousel order. Cover always shown first.
ota_versions      → JSON. Auto-generated at upload time:
                    {
                      "booking_com": "720×540 jpg url",
                      "expedia":     "800×533 jpg url",
                      "airbnb":      "1024×683 jpg url"
                    }
                    Upload once → all OTA formats ready automatically.
caption           → nullable. "View from the balcony" (shown below photo)
created_at / updated_at / created_by
```

**Validation rules:**
```
Min resolution:    1024×768 px (enforced at upload)
Min photos:        3 per room type (setup score blocked if missing)
Max file size:     10 MB per photo
Format:            JPG, PNG, WEBP accepted
Auto-enhancement:  Brightness and contrast suggestion on upload
```

---

### Entity 9: `ConnectingRoom`

**Purpose:** Records physical rooms that have connecting doors between them.
Used for family bookings, group requests needing adjacent rooms.

**Industry alignment:** Called "Component Rooms" in Oracle OPERA and HTNG standard.

```
room_id               FK → Room  ↘
connected_room_id     FK → Room  → Composite primary key

Bidirectional:
  Room 201 ↔ Room 202:
    Record 1: room_id=201, connected_room_id=202
    Record 2: room_id=202, connected_room_id=201
  Both stored. Query from either direction works.

connection_type
  → CONNECTING_DOOR: physical door between rooms (can walk through)
  → ADJACENT_ROOM:   side by side, no door, but bookable as a pair
```

**How it works at booking:**
```
Guest: "2 connecting rooms for family, Dec 20-22"

System:
  1. Find all CONNECTING_DOOR pairs
  2. Check: both rooms free on Dec 20-22?
  3. Show available pairs to front desk
  4. Staff assigns pair → family walks between rooms freely
```

---

### Entity 10: `RoomSetupProgress`

**Purpose:** Tracks setup completion. Auto-calculated score.
Blocks OTA connection and go-live if minimum threshold not met.
Shows owner exactly what is missing — not a vague error.

**Industry alignment:** No existing PMS has this built-in. Our innovation.
Solves the real problem: hotels go live with incomplete setup and face OTA listing issues.

```
hotel_id              → One record per hotel
room_types_total      → Count of RoomType records (auto)
room_types_complete   → Types with: name + occupancy + bed_type + base_rate + min 1 photo
rooms_total           → Count of Room records (auto)
rooms_complete        → Rooms with: room_number + floor + type assigned
photos_types_total    → Total room types (need photos)
photos_types_done     → Types with minimum 3 photos uploaded
amenities_types_total → Total room types (need amenity config)
amenities_types_done  → Types with at least basic amenities configured
overall_pct           → Auto-calculated:
                          (room_types_complete/total × 30) +
                          (rooms_complete/total × 30) +
                          (photos_done/total × 25) +
                          (amenities_done/total × 15)
last_calculated_at    → Recalculates on every setup change
```

**What staff sees:**
```
Room Setup: 73% complete
  ✅ Room Types:   4 / 4 complete
  ✅ Rooms:       60 / 60 created
  ⚠ Photos:        2 / 4 types done  ← Suite and Pool Villa missing photos
  ⚠ Amenities:     3 / 4 types done  ← Pool Villa amenities not configured

[Connect OTA]  → BLOCKED — minimum 80% required
[Go Live]      → BLOCKED — minimum 80% required

"Add photos for Suite and Pool Villa to unlock OTA connection."
```

---

### Entity 11: `RoomTypeTranslation`

**Purpose:** Multi-language support for room type names and descriptions.
Required for OTA listings in different languages and multi-language guest portals.

**Industry alignment:** Apaleo explicitly includes translated versions per language.
Sri Lanka context: English (OTA) + Sinhala + Tamil (local guests).

```
id
room_type_id      → FK → RoomType
language_code     → "en" / "si" / "ta" / "zh" / "de"
name              → Room type name in this language
description       → Room description in this language
is_active         → bool

V1:  English only (auto-populated from RoomType.name and description)
Phase 2: Full multi-language — hotel enters Sinhala, Tamil translations
```

---

## 7. Pet Policy — Full Coverage (3 Levels)

Pet support must exist at 3 levels. Rate plan alone is not enough.

```
LEVEL 1: Room Type
  is_pet_friendly    → Does this room CATEGORY allow pets?
  
LEVEL 2: Physical Room (Override)
  is_pet_friendly_override   → This specific room exception
  null  = inherit from type
  true  = pet allowed (even if type doesn't)
  false = pet not allowed (even if type does)

LEVEL 3: Rate Plan Policy (booking rules)
  pet_allowed          → Does this rate plan permit pet bookings?
  pet_fee              → LKR 1,500/night (auto-posts to folio)
  pet_types_allowed    → ["DOG", "CAT", "ANY"]
  pet_size_limit       → SMALL / MEDIUM / LARGE / ANY
  max_pets             → Max 2 pets per booking

AMENITY:
  "Pet Friendly" in Amenity master → OTA filter support

HOW IT WORKS TOGETHER:
  Guest books: "1 room, 1 dog"

  Step 1: Rate Plan check → pet_allowed = true, type = DOG allowed ✅
  Step 2: Room filter → only is_pet_friendly rooms shown
  Step 3: Assignment → staff assigns pet room only (system warns otherwise)
  Step 4: Folio → pet_fee auto-posts every night
```

---

## 8. Full Entity Relationship Map

```
Hotel (1)
  ├── Building (many)
  │     └── Floor (many)
  │           └── Room (many)
  │                 ├── room_type_id ──────────────→ RoomType (1)
  │                 │                                    ├── RoomTypeAmenity (many)
  │                 │                                    │     └── amenity_id → Amenity
  │                 │                                    ├── RoomPhoto (many)
  │                 │                                    └── RoomTypeTranslation (many)
  │                 ├── RoomAmenityOverride (many)
  │                 │     └── amenity_id → Amenity
  │                 ├── ConnectingRoom (many ↔ many)
  │                 └── housekeeping_zone_id → HousekeepingZone (Area 6)
  │
  └── RoomSetupProgress (1)
```

---

## 9. Basic to Advanced — Same Entities, Different Depth

```
                    GUESTHOUSE    BUSINESS     RESORT       LUXURY
                    (12 rooms)    (80 rooms)   (200 rooms)  (300 rooms)
                    ──────────    ─────────    ──────────   ─────────
Buildings           1 (hidden)    1            2-3          3-5
Floors              2             8            4            10
RoomTypes           2             4            6            8+
Rooms               12            80           200          300
Amenities           8             20           35           50+
Photos/type         3             5            8            12+
ConnectingRooms     0             4 pairs      20 pairs     40+ pairs
Translations        English       English      EN + SI      EN + SI + TA + ZH
Pet rooms           0             5            20           30+

Entities used:      All 11        All 11       All 11       All 11
Fields used:        20%           50%          75%          100%
```

**Same 11 entities. Every hotel size. No schema changes between phases.**

---

## 10. V1 Scope — What to Build First

```
MUST HAVE (V1 — hotel cannot operate without):
  ✅ RoomType          → name, category, max_adults, max_children,
                         max_occupancy, bed_type, base_rate,
                         is_pet_friendly, view_type, is_active
  ✅ Room              → room_number, room_type_id, floor_id,
                         is_accessible, internal_notes,
                         is_pet_friendly_override, is_active
  ✅ Building          → auto-created, single hotel
  ✅ Floor             → floor_number
  ✅ Amenity           → master list (pre-loaded by template)
  ✅ RoomTypeAmenity   → basic amenities per type
  ✅ RoomPhoto         → min 3 photos per type (go-live blocked otherwise)
  ✅ ConnectingRoom    → simple pair mapping
  ✅ RoomSetupProgress → completion score + blocking logic

PHASE 2:
  → RoomAmenityOverride (room-specific exceptions)
  → RoomTypeTranslation (Sinhala, Tamil, other languages)
  → display_name for luxury named rooms
  → Accessibility detailed features
  → OTA photo auto-format generation
  → Visual floor plan builder (drag-drop)
  → Bulk room generation wizard

PHASE 3:
  → Room-specific photos (ABS selling)
  → AI-based room scoring for smart assignment
  → Attribute-Based Selling (sellable room attributes)
```

---

## 11. How Room Setup Connects to Every Other Module

```
ROOM TYPE ──→ RATE PLAN
  Rate Plan created per room type.
  "Standard Double — Flexible" = rate plan for Standard rooms.
  No room type → rate plan has nothing to attach to.

ROOM (physical) ──→ AVAILABILITY
  Availability = total physical rooms − blocked rooms.
  System counts actual room numbers, not types.
  "Room 101 booked Dec 20-22" → that specific room removed from count.
  No physical rooms → system cannot count inventory.

ROOM ──→ HOUSEKEEPING
  housekeeping_zone_id on Room entity.
  "Room 101 → Zone A → Meena's responsibility."
  Task created for room 101 → auto-assigned to Zone A housekeeper.

ROOM ──→ FRONT DESK
  Staff assigns specific room number at check-in.
  internal_notes → shown as alert popup during assignment.
  is_pet_friendly → system warns if pet guest assigned to non-pet room.

ROOM TYPE ──→ OTA SYNC
  Booking.com receives: "Standard Double: 18 available tonight."
  System counts physical rooms of that type that are free.
  OTA listing uses: name, description, photos, amenities, view_type.

ROOM TYPE ──→ FOLIO & BILLING
  Night audit posts: room charge based on room type base rate.
  Pet fee posts based on is_pet_friendly and rate plan pet_fee.
  Tax calculation uses room charge amount.

ROOM TYPE ──→ HOUSEKEEPING SETUP
  Cleaning checklist defined per room type.
  Amenity restock quantities defined per room type.
  Mini bar items assigned per room type.
```

---

## 12. Summary — Why This Design is Right

```
INDUSTRY ALIGNED:
  Two-tier model (RoomType + Room) — used by OPERA, Mews, Apaleo, Cloudbeds.
  All OTA-required fields present (Booking.com + Expedia validated).
  HTNG standard fields covered.

DESIGN PHILOSOPHY ALIGNED:
  Simple hotel: fill 4 fields, go live in 10 minutes.
  Luxury resort: full control of every detail.
  Same 11 entities serve both.

FUTURE READY:
  V1 data model supports Phase 2 and Phase 3 features.
  No schema changes needed — only new data and feature flags.
  ABS (Phase 3) foundation built into entity design today.

OUR INNOVATIONS (not in any existing system):
  → Setup Completion Score with blocking logic
  → Room notes surfaced as alerts at assignment
  → Pet policy at 3 levels (Type + Room + Rate Plan)
  → Photo OTA-format auto-generation
  → Property templates (80% pre-filled setup)
```

---

## 13. Entity Quick Reference

| # | Entity | Records (100-room hotel) | Purpose |
|---|--------|--------------------------|---------|
| 1 | Building | 1-3 | Property structure |
| 2 | Floor | 5-10 | Floor within building |
| 3 | RoomType | 3-6 | Room category template |
| 4 | Room | 100 | Actual room numbers |
| 5 | Amenity | 20-40 | Master amenity dictionary |
| 6 | RoomTypeAmenity | 60-150 | Amenities per type |
| 7 | RoomAmenityOverride | 5-20 | Room-specific exceptions |
| 8 | RoomPhoto | 15-50 | Photos per room type |
| 9 | ConnectingRoom | 10-40 | Connected room pairs |
| 10 | RoomSetupProgress | 1 | Completion tracking |
| 11 | RoomTypeTranslation | 3-18 | Multi-language names |
