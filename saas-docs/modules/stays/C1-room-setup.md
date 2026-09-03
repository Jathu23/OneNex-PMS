# C1 — Room Setup
> Stays Module | Core Feature 1 of 9
> Status: Final | Version: 1.0
> Validated against: Oracle OPERA, Mews, Apaleo, Cloudbeds, HTNG Standard, Booking.com API, Expedia API

---

## 1. What Problem Does This Solve

Every hotel operation depends on one question the system must answer correctly:

```
"What rooms do you have, and which ones are available right now?"
```

If the system doesn't know the answer precisely:

```
Wrong room setup → Wrong availability → Double booking
Wrong room setup → Wrong pricing     → Revenue loss
Wrong room setup → Wrong OTA listing → Guest complaint
Wrong room setup → Wrong housekeeping → Room not ready
Wrong room setup → Wrong billing     → Tax invoice error
```

Room Setup is not just a configuration screen.
It is the foundation every other module depends on.

---

## 2. Design Philosophy

```
"Simple by Default. Powerful when needed."
```

One system must serve two very different users:

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

RoomType: name, bed arrangement,     RoomType: name, code, description,
          max_adults, base_rate                 category, view_type,
                                               bed arrangements, occupancy,
Room: room_number, floor,                      size, smoking, amenities,
      room_type                                translations, photos, OTA mapping

System hides everything else.        System shows everything.
```

---

## 3. Industry Validation

| System / Standard | Approach | Match |
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

## 4. The Two-Tier Model

### Level 1: Room Type (Template / Category)

```
"What KINDS of rooms do we have?"

Room Type = a mold. Define once. Apply to many rooms.

Example: "Standard Double"
  → Max 2 adults, 1 child
  → 1 Queen bed (160×200 cm)
  → 320 sq ft
  → Non-smoking
  → Amenities: AC, TV, WiFi, Hot water
  → Base rate: LKR 12,000/night (reference only — see billing note below)
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

Room 102 (same type, same floor — no differences):
  → Inherits everything from Standard Double type
  → Nothing different to store → zero extra records
```

### The Inheritance Rule

```
QUERY: "What does Room 305 offer?"

1. Get Room 305 → room_type = Deluxe
2. Get all Deluxe room type defaults
3. Check: does Room 305 have any overrides?
4. Override wins if exists. Type default otherwise.

Result:
  → 28 Standard Double rooms with no overrides = ZERO extra storage
  → Only exceptions (2 garden view rooms) need override records
  → Changing base rate on type = all 28 rooms updated instantly
```

---

## 5. Property Structure

Hotels have a physical hierarchy. Our system mirrors this exactly.

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
One building auto-created. Building concept hidden from UI.
Hotel owner sees: Floors → Rooms directly.
```

**Multi-building resort:**
```
Owner sees: Building selector → Floor → Rooms.
Floor map shows per building.
```

---

## 6. All Entities — Purpose and Fields

---

### Entity 1: `Building`

**Purpose:** Represents a physical structure within the property.
Single building hotels: auto-created, hidden from UI.
Multi-building properties: each building configured separately.

```
id                → Unique identifier
business_id         → Which hotel (multi-tenant isolation)
name              → "Main Block" / "Pool Wing" / "Garden Villas"
code              → "MB" / "PW" (used in room numbering: PW-01, PW-02)
display_order     → Controls order on floor map
created_at / updated_at / created_by / updated_by
```

```
Simple hotel:  1 auto-created building. Owner never sees this entity.
Resort:        Full building management. Each building named and mapped.
```

---

### Entity 2: `Floor`

**Purpose:** Floors within a building. Enables housekeeping zone assignment,
floor preference logic, and floor map display.

```
id                → Unique identifier
business_id          → Multi-tenant isolation
building_id       → Which building (FK → Building)
floor_number      → 1, 2, 3... (0 = ground floor, -1 = basement)
floor_name        → nullable.
                    null       = simple hotel ("Floor 1" auto-generated)
                    "Executive Floor" = luxury hotel named floor
display_order     → Display order on floor map
created_at / updated_at / created_by / updated_by
```

---

### Entity 3: `RoomType`

**Purpose:** The most important entity. Template that defines an entire category of rooms.
Change one field → all rooms of that type update instantly.

**Industry alignment:** "Unit Groups" in Apaleo, "Space Types" in Mews, "Room Types" in OPERA.

```
id                    → Unique identifier
business_id              → Multi-tenant isolation

-- IDENTITY --
name                  → "Standard Double"
                        Guest-facing. Used in: OTA listing, booking widget,
                        confirmation email, folio, invoice.
short_code            → "STD-DBL"
                        Internal only: floor map, reports, housekeeping board.
description           → Guest-facing text shown at booking.
room_category         → BEDROOM / SUITE / VILLA / APARTMENT / DORMITORY
                        SYSTEM-LEVEL FIXED ENUM. Hotel cannot add custom values.
                        Why fixed: Booking.com / Expedia have fixed category filters
                        ("Suites" tab, "Villas" tab). Custom values break OTA sync.
                        HTNG industry standard — globally agreed values.
                        Hotel only selects from this list, never creates new values.

-- OCCUPANCY --
max_adults            → Maximum adult guests
max_children          → Maximum children (separate: different pricing)
max_occupancy         → Total people allowed (adults + children combined)
                        OTA requires all 3 for correct availability display.

-- PHYSICAL --
size_sqft             → nullable. Room size shown on OTA listing.

-- Bed config: NOT stored here. Managed via BedArrangement + BedConfiguration entities.
-- See Entity 10 (BedSizeOption), Entity 11 (BedArrangement), Entity 12 (BedConfiguration),
-- Entity 13 (RoomExtraBed), Entity 14 (RoomBedOverride).

-- POLICY --
smoking_policy        → SMOKING / NON_SMOKING / BOTH
                        Type-level default. Room can override.
is_pet_friendly       → bool. Does this room category allow pets?
                        Pet bookings filtered to pet-friendly rooms only.
view_type             → CITY / OCEAN / GARDEN / POOL / MOUNTAIN / NONE
                        SYSTEM-LEVEL FIXED ENUM. Hotel cannot add custom values.
                        Why fixed: OTA "view" filter recognizes only these values.
                        Custom values ("BACKWATER view") won't appear in OTA filter.
                        HTNG industry standard — dedicated field, NOT an amenity.
                        If no exact match: pick closest. Unique view → NONE (honest).
                        e.g. Paddy field view → GARDEN. Volcano view → NONE.

is_mobility_accessible → bool. Does this room TYPE allow wheelchair-accessible bookings?
                         Type-level flag — OTA accessibility filter uses this.
                         Accessible room type → true (all rooms of this type accessible).
                         Standard room type → false.

-- PRICING --
base_rate             → Setup reference rate only (LKR).
                        IMPORTANT: NOT the billing rate.
                        Purpose: pre-fills RatePlanRoom.base_rate when staff
                        creates a new rate plan (saves manual typing).
                        Authoritative billing rate → RatePlanRoom.base_rate.
                        If no rate plan exists → system blocks booking entirely.

-- DISPLAY --
display_order         → Booking widget order (cheapest first or custom)
is_active             → false = category hidden everywhere (renovation, removal)

-- AUDIT --
created_at / updated_at / created_by / updated_by
```

```
Simple hotel: name, max_adults, base_rate → add 1 bed config → done in 2 minutes.
Luxury hotel: all fields, multiple bed arrangements, translations, full amenity config.
```

---

### Entity 4: `Room` (Physical Room)

**Purpose:** One record = one real room with a number on the door.
Inherits everything from RoomType. Stores ONLY what differs from its type.

**Industry alignment:** "Units" in Apaleo, "Spaces" in Mews, "Room Number" in OPERA.

```
id                        → Unique identifier
business_id                 → Multi-tenant isolation

-- LOCATION --
room_type_id              → FK → RoomType (inheritance link — most important field)
floor_id                  → FK → Floor
building_id               → FK → Building (denormalized for query speed)

-- IDENTITY --
room_number               → "101" / "PW-01" / "Penthouse" (unique per hotel)
display_name              → nullable.
                            null = room_number used everywhere.
                            "The Garden Suite" = set only for luxury named rooms.

-- OVERRIDES (null = inherit from type) --
smoking_override          → nullable bool
is_pet_friendly_override  → nullable bool
                            null  = inherit from RoomType
                            true  = this room allows pets (even if type doesn't)
                            false = this room blocks pets (even if type allows)
view_type_override        → nullable enum (CITY/OCEAN/GARDEN/POOL/MOUNTAIN/NONE)
                            null = inherit from RoomType.view_type
                            Set when one specific room faces a different direction.
                            e.g. Type = "Ocean View" but room 104 faces car park
                            → Override: NONE (prevents wrong OTA listing)

-- Bed override: NOT stored here. Managed via RoomBedOverride entity (Entity 14).
-- Use when one specific room's beds differ from room type default.
-- e.g. Type allows King/Twin arrangement, but Room 304 has wall-mounted beds → King only.

-- ACCESSIBILITY --
is_mobility_accessible_override → nullable bool.
                            null  = inherit from RoomType.is_mobility_accessible
                            true  = this specific room IS accessible
                                    (even if type is not — rare edge case)
                            false = this specific room is NOT accessible
                                    (e.g. accessible type, but one room has a step)
                            Same override pattern as smoking, pet, view_type.
                            Specific accessibility features (grab bars, roll-in shower...)
                            → handled via Amenity system (ACCESSIBILITY category).
                            Not a JSON field here — Amenity system gives OTA sync.

-- OPERATIONS --
internal_notes            → text nullable. STAFF ONLY. Guest never sees.
                            "AC noise in left corner"
                            "Connecting door to 102 — lock before assignment"
                            Shown as ALERT popup during room assignment.
                            Staff cannot assign room without seeing the note.

housekeeping_zone_id      → FK → HousekeepingZone
                            "Room 101 → Zone A → Meena's responsibility"
                            Housekeeping task auto-assigned based on this.

is_active                 → bool.
                            false = room permanently removed from inventory.
                            Use for: decommissioned rooms, converted to office.
                            For TEMPORARY blocks (painting, repair) → see OOO note below.

-- AUDIT --
created_at / updated_at / created_by / updated_by
```

**OOO (Out of Order) — V1 Approach:**
```
V1: Staff sets is_active = false for temporary blocks.
    Staff manually sets is_active = true when room is ready.
    Simple. Works for small hotels with staff discipline.

Phase 2: RoomBlock entity (date-ranged, auto-release).
    "Room 203 blocked Dec 1–5, auto-available Dec 6."
    For hotels that need automated OOO management.
```

---

### Entity 5: `AmenityCategory` — System Level Category Master

**Purpose:** System-seeded master list of amenity categories.
Business cannot add or remove categories. Fixed set — used to organise amenities in UI and filter display.

```
id
name              "COMFORT" / "TECH" / "BATHROOM" / "VIEW" /
                  "FOOD" / "SAFETY" / "OUTDOOR" / "PET" / "ACCESSIBILITY"
icon              text nullable. Icon library name. "sofa" / "wifi" / "shower"
                  Shown as category header icon in amenity setup UI.
icon_path         text nullable. Custom image path if icon library name not sufficient.
                  e.g. "/icons/categories/accessibility.svg"
                  null = use icon field (library name). Both can exist; icon_path takes priority.
display_order     int. Order categories appear in setup UI and booking widget.
```

---

### Entity 6: `Amenity` — System Level Amenity Master

**Purpose:** System-seeded master list of common amenities. No business_id here.
All businesses see this list and select from it.

```
id
category_id       FK → AmenityCategory
name              "Air Conditioning" / "WiFi" / "Rainshower" / "Nespresso Machine"
short_code        "AC" / "WIFI" / "RAIN"
icon              text nullable. Icon library name. "snowflake" / "wifi" / "shower"
                  Shown as amenity icon in booking widget and setup UI.
icon_path         text nullable. Custom image path.
                  e.g. "/icons/amenities/air-conditioning.svg"
                  null = use icon field. Both can exist; icon_path takes priority.
ota_mapping_code  JSON nullable.
                  OTAs use their own internal amenity IDs for API sync.
                  {"booking_com": "107", "expedia": "2001", "agoda": "55"}
                  null = amenity not pushed to OTA (internal label only).
                  Required when Channel Manager (A1) add-on is enabled.
```

**System-seeded amenities:**
```
COMFORT:       Air Conditioning, Heating, Fan, Blackout Curtains
TECH:          Smart TV, WiFi, Bluetooth Speaker, USB Charging Ports
BATHROOM:      Hot Water, Bathtub, Rainshower, Jacuzzi, Hair Dryer
VIEW:          Sea View, Garden View, Pool View, City View, Mountain View
FOOD:          Mini Bar, Coffee Machine, Kitchenette, Fruit Basket
SAFETY:        In-room Safe, Smoke Detector, Fire Extinguisher
PET:           Pet Friendly, Pet Bed, Pet Bowl
ACCESSIBILITY: Grab Bars, Roll-in Shower, Wide Doorway (90cm+),
               Lowered Bed, Visual Fire Alerts, Accessible Bathroom,
               Wheelchair Accessible Entrance, Elevator Access
```

**Why ACCESSIBILITY uses Amenity system (not a JSON field on Room):**
```
Amenity.ota_mapping_code carries the OTA-specific code for each feature.
  "Grab Bars" → {"booking_com": "445", "expedia": "2201"}

Staff checks "Grab Bars" on room type → auto-syncs to OTA with correct code.
JSON field on Room → no ota_mapping_code → OTA sync impossible.
```

---
### Entity 7: `BusinessAmenityCategory` — Business Level Amenity Category Master

**Purpose:** Business-level custom amenity categories — the "my own category" equivalent of the system-seeded AmenityCategory. Created once, reused by many custom amenities.

```
id
business_id       → FK → Business
name              → "Outdoor Extras" / "Business Traveller" (business's own category name)
icon              → text nullable. Icon library name.
icon_path         → text nullable. Custom image path. Takes priority over icon if both set.
display_order     → int. Order among the business's OWN custom categories.
is_active         → bool. Soft delete — hides category and cascades to its amenities in UI.
created_at / updated_at / created_by / updated_by
```
---
### Entity 8: `BusinessAmenity` — Business Level Amenity List

**Purpose:** Each business builds their own amenity list by:
1. Selecting from system amenities (Scenario 1)
2. Creating custom amenity under a system category (Scenario 2)
3. Creating custom amenity with custom category (Scenario 3)

```
id
business_id           FK → Business

-- Scenario 1: selected from system list
amenity_id            FK → Amenity nullable
                      set  = linked to system amenity
                      null = custom amenity (scenarios 2 & 3)

-- Scenario 2 & 3: custom amenity details
name                  text nullable
                      Scenario 1 → null (use Amenity.name)
                      Scenario 2 & 3 → business enters their own name

icon                  text nullable. Icon library name for custom amenity.
icon_path             text nullable. Custom image path for custom amenity.
                      null = use icon field. Both can exist; icon_path takes priority.

-- Scenario 2: custom amenity under system category
category_id           FK → AmenityCategory nullable
                      Scenario 1 → null (inherited from Amenity.category_id)
                      Scenario 2 → business picks from system categories
                      Scenario 3 → null (use custom_category_name)

-- Scenario 3: custom amenity + custom category
business_category_id  FK → BusinessAmenityCategory nullable
                      null for scenarios 1 & 2.
                      Scenario 3 → business enters their own category name.

ota_mapping_code      JSON nullable.
                      Scenario 1 → inherited from Amenity.ota_mapping_code
                      Scenario 2 & 3 → business fills manually or null (internal only)

is_active             bool. Soft delete.
```

**Which scenario applies — rules:**
```
amenity_id set                          → Scenario 1 (system amenity selected)
amenity_id null + category_id set       → Scenario 2 (custom name, system category)
amenity_id null + category_id null      → Scenario 3 (fully custom)
```

**Full list query for a business:**
```sql
SELECT
  ba.id,
  COALESCE(a.name,             ba.name)                AS amenity_name,
  COALESCE(ac.name,            ba.custom_category_name) AS category_name,
  COALESCE(a.icon,             ba.icon)                AS icon,
  COALESCE(a.icon_path,        ba.icon_path)           AS icon_path,
  COALESCE(a.ota_mapping_code, ba.ota_mapping_code)    AS ota_mapping_code
FROM BusinessAmenity ba
LEFT JOIN Amenity a          ON ba.amenity_id = a.id
LEFT JOIN AmenityCategory ac ON COALESCE(a.category_id, ba.category_id) = ac.id
WHERE ba.business_id = 42
  AND ba.is_active = true
ORDER BY ac.display_order, amenity_name
```

**Why ACCESSIBILITY uses Amenity system (not a JSON field on Room):**
```
BusinessAmenity.ota_mapping_code (inherited from Amenity) carries the OTA code.
  "Grab Bars" → {"booking_com": "445", "expedia": "2201"}

Hotel marks accessible room type → is_mobility_accessible = true (OTA filter).
Hotel marks specific features    → RoomTypeAmenity (ACCESSIBILITY category).
Room exception                   → RoomAmenityOverride.
```

---

### Entity 9: `RoomTypeAmenity`

**Purpose:** Defines which amenities each room TYPE has as defaults.
Junction table: links RoomType to BusinessAmenity.

**Why explicit YES and NO (not just skipping the record):**
```
OTA API asks: "Does Standard Double have TV?" → needs explicit YES or NO.
Absence of record ≠ NO.
Explicit has_amenity = false = NO.
```

```
room_type_id          FK → RoomType        ↘
business_amenity_id   FK → BusinessAmenity → Composite primary key
has_amenity           bool. YES or NO for this amenity on this room type.
value                 text nullable. Extra detail.
                      WiFi → "100 Mbps"
                      TV   → "55 inch Smart TV"
                      AC   → null (yes/no is enough)
                      Shown at booking: "WiFi — 100 Mbps"
```

**Example records:**
```
room_type        amenity          has_amenity   value
─────────────────────────────────────────────────────
Standard Double  AC               true          null
Standard Double  Mini Bar         false         null
Standard Double  WiFi             true          "50 Mbps"
Standard Double  Pet Friendly     false         null
Deluxe King      AC               true          null
Deluxe King      Mini Bar         true          null
Deluxe King      WiFi             true          "100 Mbps"
Garden Suite     Pet Friendly     true          null
```

---

### Entity 10: `RoomAmenityOverride`

**Purpose:** Room-specific exception from type default.
Stored ONLY when ONE specific room differs from its type.
Zero records for rooms with no exceptions → clean, efficient.

```
room_id               FK → Room            ↘
business_amenity_id   FK → BusinessAmenity → Composite primary key
has_amenity           bool. Override value. Replaces type default for this room only.
value                 text nullable. Override detail.
                      "50 Mbps" when type has "100 Mbps"
reason                text nullable. Internal staff note. Why is this room different?
                      "WiFi extender not installed in this corner"
                      "Balcony sealed — safety repair pending"
```

**Query logic (runs at booking, OTA sync, checklist):**
```
"What amenities does Room 305 have?"

1. Get all RoomTypeAmenity for Room 305's type
2. Get all RoomAmenityOverride for Room 305
3. For each amenity:
     IF override record exists → use override value
     ELSE                      → use type default
```

---

### Entity 11: `BedSizeOption` — System Level Standard Sizes

**Purpose:** System-seeded standard bed dimensions per bed type.
Business selects from this list when configuring beds. Custom size = override fields.

```
id
bed_type          KING / QUEEN / DOUBLE / SINGLE /
                  BUNK / SOFA_BED / FUTON / TATAMI
label             "Standard" / "US King" / "Compact King"
width_cm          int
length_cm         int
is_default        bool. Which size pre-selects automatically when bed_type chosen.
display_order     int
```

**System-seeded sizes:**
```
KING     → Standard (180×200, default) | US King (193×203) | Compact (170×200)
QUEEN    → Standard (160×200, default) | US Queen (153×203)
DOUBLE   → Standard (140×200, default)
SINGLE   → Standard (90×200,  default) | Compact (80×190)
BUNK     → Standard (90×190,  default)
SOFA_BED → Standard (120×190, default) | Single Sofa (90×180)
FUTON    → Standard (100×200, default)
TATAMI   → Standard (100×200, default)
```

---

### Entity 12: `BedArrangement` — Convertible Bed Setup Options

**Purpose:** Defines different ways the beds in a room type can be arranged.
Only needed when the same physical beds can be set up in multiple ways.
One arrangement = one way to configure the beds before guest arrival.

```
id
room_type_id      FK → RoomType
name              "King Setup" / "Twin Setup" / "Standard"
is_default        bool. Which arrangement shows first at booking.
display_order     int
```

**When to use:**
```
Room has 2 single beds that can be joined:
  BedArrangement: "King Setup"   is_default: true
  BedArrangement: "Twin Setup"   is_default: false

Room has fixed beds (no conversion possible):
  BedArrangement: "Standard"     is_default: true   ← one arrangement only
  UI hides arrangement selector at booking — guest just sees bed info.
```

---

### Entity 13: `BedConfiguration` — Actual Beds Per Arrangement

**Purpose:** Defines the actual beds in each BedArrangement.
One-to-many: one arrangement can have multiple bed types (e.g. 1 King + 2 Singles).

```
id
room_type_id        FK → RoomType
bed_arrangement_id  FK → BedArrangement

bed_type            KING / QUEEN / DOUBLE / SINGLE /
                    BUNK / SOFA_BED / FUTON / TATAMI
bed_count           int. How many beds of this type in this arrangement.

bed_size_option_id  FK → BedSizeOption nullable
                    set  = standard size selected from list
                    null = custom size (use width_cm + length_cm)

width_cm            int nullable. Custom width. Used when bed_size_option_id = null.
length_cm           int nullable. Custom length. Used when bed_size_option_id = null.
```

**Examples:**
```
Family Suite — 1 arrangement (fixed, no conversion):
  BedArrangement: "Standard"
    BedConfiguration row 1: KING   × 1, 180×200
    BedConfiguration row 2: SINGLE × 2, 90×200

Deluxe Room — 2 arrangements (convertible):
  BedArrangement: "King Setup"
    BedConfiguration row 1: KING   × 1, 180×200
  BedArrangement: "Twin Setup"
    BedConfiguration row 1: SINGLE × 2, 90×200
```

**OTA display — auto generated from rows:**
```
Family Suite  → "1 King bed (180×200 cm) + 2 Single beds (90×200 cm)"
Deluxe Room   → "King bed (180×200 cm) or 2 Single beds (90×200 cm)"
Dormitory     → "6 Bunk beds (90×190 cm)"
```

**Booking UI:**
```
One arrangement only   → show bed info as text. No selector shown.
Multiple arrangements  → show "Bed Arrangement" selector.

Deluxe Room:
  Bed Arrangement:
  ● King bed (180×200 cm)      ← default
  ○ 2 Single beds (90×200 cm)
```

**Housekeeping instruction — auto generated:**
```
Guest selected "King Setup" → housekeeping board shows:
  Room 304 | Arrival Oct 5 | "Join the two single beds"

Guest selected "Twin Setup" → housekeeping board shows:
  Room 304 | Arrival Oct 5 | "Separate the beds"
```

---

### Entity 14: `RoomExtraBed` — Add-on Beds at Booking Time

**Purpose:** Extra beds that can be added to a room at booking time on request.
These are not permanently in the room — staff brings them in for the stay.

```
id
room_type_id        FK → RoomType
extra_bed_type      ROLLAWAY / BABY_COT / SOFA_BED
max_extra_beds      int. Maximum number of this extra bed type allowed in the room.
charge_per_night    decimal. 0.00 = free.
is_available        bool.
```

**Booking UI:**
```
Extra beds (optional):
  [ ] Rollaway bed   + LKR 500/night
  [ ] Baby cot       Free
```

**Housekeeping task — auto generated:**
```
Booking has baby cot request → housekeeping board:
  Room 304 | Arrival Oct 5 | "Place baby cot in room"
```

---

### Entity 15: `RoomBedOverride` — Room-Level Bed Exception

**Purpose:** When one specific physical room's beds differ from its room type defaults.
Stored ONLY when a specific room differs. Zero records = inherit from type.
Consistent with RoomAmenityOverride pattern.

```
id
room_id                   FK → Room

-- Option A: disable a specific arrangement for this room
disabled_arrangement_id   FK → BedArrangement nullable
                          "Room 304 cannot do King Setup — wall-mounted beds"

-- Option B: override the entire bed config for this room
override_bed_type         enum nullable
override_bed_count        int nullable
override_width_cm         int nullable
override_length_cm        int nullable

reason                    text nullable. Internal staff note.
                          "Wall-mounted beds — cannot join"
                          "Beds replaced with King during Oct 2025 renovation"
```

**Query logic (runs at booking, room assignment):**
```
"What bed arrangements are available for Room 304?"

1. Get all BedArrangements for Room 304's RoomType
2. Get all RoomBedOverride for Room 304
3. Remove disabled arrangements
4. Apply any bed config overrides

Result → show only valid options to guest at booking
```

---

### Entity 16: `RoomPhoto`

**Purpose:** Photos for each room type.
Upload once → system auto-generates all OTA-specific format/size versions.

**Why photos belong to RoomType, not Room:**
```
Photos show the CATEGORY of room — not specific room 101.
All Standard Double rooms are represented by the same photos.
Exception (Phase 3): individual room photo for Attribute-Based Selling.
```

```
id
room_type_id      → FK → RoomType
room_id           → FK → NULLABLE
url               → Original high-resolution upload (cloud storage)
thumbnail_url     → Auto-generated small version (fast loading)
photo_type        → INTERIOR / BATHROOM / VIEW / EXTERIOR / AMENITY / FLOOR_PLAN
                    Required by Booking.com + Expedia API for correct categorization.
                    INTERIOR:   main room shot (OTAs use as listing thumbnail)
                    BATHROOM:   bathroom photos (high conversion impact)
                    VIEW:       view from balcony or window
                    EXTERIOR:   building / property exterior
                    AMENITY:    specific amenity shot (pool, minibar, TV)
                    FLOOR_PLAN: room layout diagram
                    Without photo_type: OTA photo sync may fail or mislabel photos.
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
Min resolution:   1024×768 px (enforced at upload)
Min photos:       3 per room type (setup score blocked if missing)
Max file size:    10 MB per photo
Format:           JPG, PNG, WEBP accepted
```

---

### Entity 17: `ConnectingRoom`

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
  2. Check: both rooms available Dec 20-22?
  3. Show available pairs to front desk
  4. Staff assigns pair → family walks between rooms freely
```

---

### Entity 18: `RoomSetupProgress`

**Purpose:** Tracks setup completion. Auto-calculated score.
Blocks OTA connection and go-live if minimum threshold not met.
Shows owner exactly what is missing — not a vague error.

**Our innovation:** No existing PMS has this built-in.
Solves the real problem: hotels go live with incomplete setup → OTA listing issues.

```
business_id              → One record per hotel
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
                          (rooms_complete/total     × 30) +
                          (photos_done/total         × 25) +
                          (amenities_done/total      × 15)
last_calculated_at    → Recalculates on every setup change
```

**What the owner sees:**
```
Room Setup: 73% complete
  ✅ Room Types:   4 / 4 complete
  ✅ Rooms:       60 / 60 created
  ⚠  Photos:       2 / 4 types done  ← Suite and Pool Villa missing photos
  ⚠  Amenities:   3 / 4 types done  ← Pool Villa amenities not configured

[Connect OTA]  → BLOCKED — minimum 80% required
[Go Live]      → BLOCKED — minimum 80% required

"Add photos for Suite and Pool Villa to unlock OTA connection."
```

---

### Entity 19: `RoomTypeTranslation`

**Purpose:** Multi-language support for room type names and descriptions.
Required for OTA listings in different languages and multi-language guest portals.

```
id
room_type_id      → FK → RoomType
language_code     → "en" / "si" / "ta" / "zh" / "de"
name              → Room type name in this language
description       → Room description in this language
is_active         → bool

V1:     English only (auto-populated from RoomType.name and description)
Phase 2: Full multi-language — hotel enters Sinhala, Tamil translations
```

---

## 7. Pet Policy — Full Coverage (3 Levels)

Pet support must exist at 3 levels. Rate plan alone is not enough.

```
LEVEL 1: Room Type
  is_pet_friendly    → Does this room CATEGORY allow pets?

LEVEL 2: Physical Room (Override)
  is_pet_friendly_override
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
  Guest books with 1 dog:

  Step 1: Rate Plan check → pet_allowed = true, type = DOG ✅
  Step 2: Room filter     → only is_pet_friendly rooms shown
  Step 3: Assignment      → staff assigns pet room (system warns otherwise)
  Step 4: Folio           → pet_fee auto-posts every night via night audit
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
  │                 │                                    │     └── amenity_id → Amenity (5)
  │                 │                                    ├── RoomPhoto (many)
  │                 │                                    └── RoomTypeTranslation (many)
  │                 ├── RoomAmenityOverride (many)
  │                 │     └── amenity_id → Amenity (5)
  │                 ├── ConnectingRoom (many ↔ many)
  │                 └── housekeeping_zone_id → HousekeepingZone (Housekeeping module)
  │
  └── RoomSetupProgress (1)
```

---

## 9. How Room Setup Connects to Every Other Module

```
ROOM TYPE ──→ RATE PLAN (C7)
  Rate plan created per room type.
  No room type → rate plan has nothing to attach to.
  RoomType.base_rate pre-fills rate plan. Billing uses rate plan value.

ROOM ──→ AVAILABILITY (C2 Reservations)
  Available = total physical rooms − confirmed bookings − OOO rooms
  System counts actual room numbers, not types.
  "Room 101 booked Dec 20-22" → that specific room removed from count.

ROOM ──→ HOUSEKEEPING (C4)
  housekeeping_zone_id on Room entity.
  "Room 101 → Zone A → Meena's responsibility"
  Task created for room 101 → auto-assigned to Zone A housekeeper.

ROOM ──→ FRONT DESK (C3)
  Staff assigns specific room number at check-in.
  internal_notes → shown as alert popup during assignment.
  is_pet_friendly → system warns if pet guest assigned to non-pet room.

ROOM TYPE ──→ OTA SYNC (A1 Channel Manager)
  Booking.com receives: "Standard Double: 18 available tonight."
  OTA listing uses: name, description, photos (+ photo_type), amenities (+ ota_mapping_code), view_type.

ROOM TYPE ──→ FOLIO & BILLING (C6)
  Night audit posts room charge based on RatePlanRoom.base_rate.
  Pet fee posts based on rate plan pet_fee every night.
  Tax calculation uses room charge amount.

ROOM TYPE ──→ HOUSEKEEPING SETUP
  Cleaning checklist defined per room type.
  Amenity restock quantities defined per room type.
```

---

## 10. Scale — Same Entities, Different Depth

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

Entities used:      All 18        All 18       All 18       All 18
Fields used:        20%           50%          75%          100%
```

Same 18 entities. Every hotel size. No schema changes between phases.

---

## 11. V1 Scope

```
MUST HAVE — V1 (hotel cannot operate without):
  ✅ RoomType            → name, category, max_adults, max_children,
                           max_occupancy, base_rate (reference only),
                           is_pet_friendly, view_type, is_active
  ✅ Room                → room_number, room_type_id, floor_id,
                           internal_notes,
                           is_pet_friendly_override, view_type_override,
                           is_mobility_accessible_override,
                           is_active (OOO via manual toggle in V1)
  ✅ Building            → auto-created, single hotel
  ✅ Floor               → floor_number
  ✅ AmenityCategory     → system seeded, icon + icon_path
  ✅ Amenity             → system seeded, icon + icon_path, ota_mapping_code
  ✅ BusinessAmenity     → select from system + custom amenities (all 3 scenarios)
  ✅ RoomTypeAmenity     → has_amenity bool + value (both required)
  ✅ BedSizeOption       → system seeded standard sizes per bed type
  ✅ BedArrangement      → at least 1 per room type required (even if only "Standard")
  ✅ BedConfiguration    → at least 1 row per arrangement required
  ✅ RoomExtraBed        → optional per room type (rollaway, baby cot)
  ✅ RoomPhoto           → photo_type per photo, min 3 per type, go-live blocked otherwise
  ✅ ConnectingRoom      → simple pair mapping
  ✅ RoomSetupProgress   → completion score + blocking logic

PHASE 2:
  → RoomBlock            → Date-ranged OOO with auto-release
  → RoomAmenityOverride  → Room-specific amenity exceptions
  → RoomBedOverride      → Room-specific bed arrangement exceptions
  → RoomTypeTranslation  → Sinhala, Tamil, other languages
  → display_name         → Luxury named rooms ("The Presidential Villa")
  → Visual floor plan builder (drag-drop)
  → Bulk room generation wizard

PHASE 3:
  → Room-specific photos (Attribute-Based Selling)
  → AI-based room scoring for smart assignment
  → Attribute-Based Selling (guests pay extra for specific attributes)
```

---

## 12. Our Innovations vs Competitors

```
FEATURE                              US    OPERA   MEWS   CLOUDBEDS
─────────────────────────────────────────────────────────────────────
Setup completion score + blocking    ✅     ❌       ❌      ❌
Room notes surfaced at assignment    ✅     ❌       ❌      ❌
Pet policy at 3 levels               ✅     Partial  ❌      ❌
photo_type for OTA compliance        ✅     ✅       ❌      ❌
ota_mapping_code on amenities        ✅     ✅       ❌      ❌
view_type as dedicated field (HTNG)  ✅     ✅       ❌      ❌
Property templates (80% pre-filled)  ✅     ❌       ❌      ❌
Simple by default (4 fields = live)  ✅     ❌       Partial ✅
```

---

## 13. Entity Quick Reference

| # | Entity | Records (100-room hotel) | Purpose |
|---|--------|--------------------------|---------|
| 1 | Building | 1-3 | Property structure |
| 2 | Floor | 5-10 | Floor within building |
| 3 | RoomType | 3-6 | Room category template |
| 4 | Room | 100 | Actual physical rooms |
| 5 | Amenity | 20-40 | Master amenity dictionary |
| 6 | RoomTypeAmenity | 60-150 | Amenities per type (has_amenity + value) |
| 7 | RoomAmenityOverride | 5-20 | Room-specific exceptions (Phase 2) |
| 8 | RoomPhoto | 15-50 | Photos per room type (+ photo_type) |
| 9 | ConnectingRoom | 10-40 | Connected room pairs |
| 10 | RoomSetupProgress | 1 | Completion tracking + go-live blocking |
| 11 | RoomTypeTranslation | 3-18 | Multi-language names (Phase 2) |
