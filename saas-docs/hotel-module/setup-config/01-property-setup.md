# Setup & Configuration — 01: Property Setup
> Hotel Module → Setup & Configuration → Area 1 of 16
> Foundation for: OTA Listings, GST Invoicing, Night Audit Timing, Guest Portal, All Operations

---

## Why Property Setup is First

```
Property Setup defines WHO the hotel is.

Without Property Setup:
  → No identity for OTA listings
  → No legal info for GST invoices
  → No timezone = night audit runs at wrong time
  → No currency = pricing broken
  → No logo = guest portal looks unfinished

Property Setup = System's first question: "Tell me about your hotel."
```

---

## Existing Systems — Problems

| System | Problem |
|--------|---------|
| Oracle OPERA | Property config scattered across 8+ admin screens. Star rating in one place, legal info in another, branding in a third. Takes days to complete setup. |
| Mews | Good basic info setup. But property facilities (pool, gym, parking) not well organized — just a free-text field. |
| Cloudbeds | Basic info only. No setup completion score. Hotel goes live with half-empty profile. OTA listing looks poor. |
| All systems | No onboarding wizard. Blank form on first login. Small hotel owner doesn't know what to fill. Leaves important fields empty. |

---

## Our Design Principles

### 1. Onboarding Wizard — First Screen After Signup
```
Welcome to [Platform Name]!
Let's set up your property in 5 minutes.

Step 1: What kind of property do you have?
  ○ Small Guesthouse / Homestay   (1-20 rooms)
  ○ Boutique Hotel                (21-50 rooms)
  ○ Business Hotel                (51-150 rooms)
  ○ Resort                        (any size, leisure focus)
  ○ Luxury / Heritage Hotel       (premium segment)

Selection → auto-loads property template:
  Room types pre-suggested
  Facilities pre-checked
  Rate plan templates matched
  OTA channels suggested
```

### 2. Property Profile — Structured Sections
```
SECTION 1: Basic Identity
  Hotel Name, Legal Name (for invoices), Star Rating

SECTION 2: Location & Contact
  Address (street, city, state, pincode)
  GPS coordinates (auto-fetch from address)
  Phone, Email, Website

SECTION 3: Legal & Tax
  GSTIN, PAN Number
  FSSAI Number (if restaurant enabled)
  Trade License Number

SECTION 4: Branding
  Logo, Cover Photo
  Brand colors (auto-extract from logo)

SECTION 5: Property Facilities
  Pool, Gym, Spa, Restaurant, Bar, Parking...
  Each with: chargeable? / operating hours

SECTION 6: Operational Defaults
  Default Check-in time, Check-out time
  Night Audit time
  Breakfast timings (if meal plans active)
```

### 3. Property Facilities — Structured (Not Free Text)
```
FACILITY SETUP:
  [✅] Swimming Pool
       → Chargeable? No (complimentary)
       → Hours: 6 AM – 10 PM
       → Indoor / Outdoor: Outdoor

  [✅] Parking
       → Chargeable? Yes — ₹200/night
       → Valet? No
       → Capacity: 40 cars

  [✅] Conference Room
       → Chargeable? Yes
       → Capacity: 80 persons
       → AV equipment: Yes

  [☐] Airport Shuttle
  [☐] Spa
  [☐] Gym

Shown on OTA listing, booking portal, guest app automatically.
```

### 4. Setup Completion Score (Property Level)
```
Property Setup: 68% complete
  ✅ Basic Info: Done
  ✅ Location: Done
  ⚠ Legal (GSTIN missing)
  ✅ Branding: Done
  ⚠ Facilities: 2 of 5 incomplete (hours missing)
  ⚠ Cover Photos: 0 uploaded

System blocks OTA connection until minimum 80% complete.
GST invoice blocked until GSTIN entered.
```

### 5. Multi-Property Support
```
One Owner Account → Multiple Properties

Owner Dashboard:
  ├── The Grand Chennai      (150 rooms) ← active
  ├── The Grand Bangalore    (90 rooms)  ← active
  └── The Grand Coimbatore   (40 rooms)  ← setup in progress

Each property: separate data, separate config, separate billing.
Owner sees consolidated report across all properties.
Staff accounts: scoped to one or more properties.
```

---

## Data Model

```
Hotel (core entity = tenant)
  id (= tenant_id in multi-tenant system)
  name                    "The Grand Chennai"
  legal_name              "Grand Hospitality Pvt Ltd"
  property_type           GUESTHOUSE / BOUTIQUE / BUSINESS / RESORT / LUXURY
  star_rating             1 / 2 / 3 / 4 / 5 / UNRATED
  description             (guest-facing, shown on booking portal)
  short_description       (OTA listing tagline)
  established_year        int nullable
  total_rooms             int (auto-calculated from Room Setup)
  is_active               bool
  setup_complete_pct      int (auto-calculated)

PropertyAddress
  hotel_id
  street_line1, street_line2
  city, state, country, pincode
  latitude, longitude     (auto-fetched or manual)
  google_place_id         (for Google Business sync — Phase 2)

PropertyContact
  hotel_id
  contact_type            FRONT_DESK / EMERGENCY / OWNER / MANAGER
  name, phone, email
  is_primary              bool

PropertyLegal
  hotel_id
  gstin
  pan_number
  fssai_number            nullable
  trade_license_number    nullable
  invoice_prefix          "INV" (folio invoice number format)

PropertyBranding
  hotel_id
  logo_url
  favicon_url
  cover_photo_url
  primary_color           hex nullable
  secondary_color         hex nullable
  email_header_url        nullable

PropertyLocale
  hotel_id
  timezone                "Asia/Kolkata"
  default_language        "en" / "ta" / "hi"
  currency_code           "INR"
  currency_symbol         "₹"
  date_format             "DD/MM/YYYY"
  time_format             "12h" / "24h"

PropertyOperationalDefaults
  hotel_id
  default_checkin_time    "15:00"
  default_checkout_time   "11:00"
  night_audit_time        "23:59"
  breakfast_start         "07:00"  nullable
  breakfast_end           "10:30"  nullable

PropertyFacility
  id, hotel_id
  facility_type           POOL / GYM / SPA / RESTAURANT / BAR / PARKING /
                          CONFERENCE_ROOM / AIRPORT_SHUTTLE / WIFI /
                          EV_CHARGING / LAUNDRY / KIDS_CLUB / BEACH_ACCESS
  is_chargeable           bool
  charge_amount           nullable decimal
  charge_unit             PER_NIGHT / PER_USE / PER_PERSON
  is_24_hours             bool
  operating_hours         JSON nullable
  indoor_outdoor          INDOOR / OUTDOOR / BOTH nullable
  capacity                int nullable
  description             text nullable

PropertySocialLinks
  hotel_id
  platform                INSTAGRAM / FACEBOOK / TRIPADVISOR / GOOGLE_BUSINESS
  url                     text
```

---

## Key Relationships

```
Hotel (1) → PropertyAddress (1)
Hotel (1) → PropertyLegal (1)
Hotel (1) → PropertyBranding (1)
Hotel (1) → PropertyLocale (1)
Hotel (1) → PropertyOperationalDefaults (1)
Hotel (1) → PropertyFacility (many)
Hotel (1) → PropertyContact (many)
Hotel (1) → PropertySocialLinks (many)

Hotel (1) → RoomTypes (many)         ← Room Setup
Hotel (1) → RatePlans (many)         ← Rate Plan Setup
Hotel (1) → Staff (many)             ← Staff Setup

Owner Account (1) → Hotels (many)    ← Multi-property
```

---

## V1 vs Phase Split

| Feature | V1 | Phase 2 | Phase 3 |
|---------|:--:|:-------:|:-------:|
| Basic info (name, type, star rating, address, contact) | ✅ | | |
| Legal info (GSTIN, PAN) | ✅ | | |
| Logo upload | ✅ | | |
| Timezone + currency + language | ✅ | | |
| Default check-in / check-out time | ✅ | | |
| Night audit time | ✅ | | |
| Property facilities (simple list with hours) | ✅ | | |
| Setup completion score | ✅ | | |
| Onboarding wizard | ✅ | | |
| Multi-property (same owner, multiple hotels) | ✅ | | |
| Brand colors + email header customization | | ✅ | |
| Google Business sync | | ✅ | |
| Rich text property description with photos | | ✅ | |
| Google Maps auto-fetch from address | | ✅ | |
| Custom domain for booking portal | | | ✅ |
| White-label options | | | ✅ |
