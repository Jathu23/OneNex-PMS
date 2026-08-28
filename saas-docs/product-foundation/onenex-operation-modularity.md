# OneNex — Operation Modularity (Core + Add-ons)

> Within each operation, features are split into Core (always on) and Add-ons (enable when needed).
> This is the second level of modularity in OneNex.

---

## Two Levels of Modularity

```
Level 1: Owner enables OPERATIONS
         Dining, Stays, Wellness, Bar, Events, Retail

Level 2: Within each operation, enable ADD-ONS
         QR Ordering, Delivery, Table Reservation, KDS...
```

Same platform. Same code. Business uses only what they need.

---

## Feature Types Within an Operation

```
TYPE 1: CORE
→ Always present. Cannot disable.
→ Operation cannot function without these.

TYPE 2: ADD-ONS
→ Default off.
→ Owner enables when their business needs it.
→ Enabling triggers a setup wizard.
→ Each add-on has dependencies (checked before enabling).
```

---

## Dining Operation — Core vs Add-ons

### Core (Always On)

| Feature | Why Core |
|---|---|
| Menu Management | No dining without a menu |
| Order Management | No dining without orders |
| Basic Table Management | No dining without tables |
| Billing & Payment | No dining without payment |
| Basic Reports | Minimum operational visibility |

### Add-ons (Enable When Needed)

| Add-on | Setup Required | Dependency |
|---|---|---|
| Table Reservation | Time slots, party size limits, booking rules | Table Management |
| QR Self-Ordering | Generate QR codes, customer-facing menu layout | Menu Management |
| Takeaway | Counter name, order number format, prep time | Order Management |
| Delivery | Delivery zones, fees, estimated time | Order Management |
| KDS (Kitchen Display) | Kitchen stations, item routing rules | Order Management |
| Inventory Tracking | Ingredients, recipes, stock levels | Menu Management |

---

## UX Pattern — Owner Settings View

```
Dining Operation Settings
│
├── Core Features          ← always visible, always working
│   ├── Menu               [Manage →]
│   ├── Tables             [Manage →]
│   ├── Orders             [Manage →]
│   └── Billing            [Manage →]
│
└── Add-ons
    ┌──────────────────────────────────┐
    │ Table Reservation   [Enable →]   │
    │ QR Self-Ordering    [Enable →]   │
    │ Takeaway            [Enable →]   │
    │ Delivery            [Enable →]   │
    │ KDS                 [Enable →]   │
    │ Inventory Tracking  [Enable →]   │
    └──────────────────────────────────┘

Enable → Setup wizard → Done → Feature active.
```

---

## Dependency Rules

When enabling an add-on, system checks dependencies first:

```
QR Ordering       → requires: Menu Management       (core ✓ always available)
Table Reservation → requires: Table Management      (core ✓ always available)
KDS               → requires: Order Management      (core ✓ always available)
Delivery          → requires: Order Management      (core ✓ always available)
Inventory         → requires: Menu Management       (core ✓ always available)
```

If dependency not met → system explains what needs to be enabled first.

---

## Same Pattern — All Operations

### Stays Operation

| Core | Add-ons |
|---|---|
| Rooms & Room Types | OTA Channel Management |
| Booking Management | Online Booking Engine |
| Check-in / Check-out | Housekeeping App |
| Folio & Billing | Revenue Management |
| Basic Reports | Corporate Rate Management |

### Wellness Operation

| Core | Add-ons |
|---|---|
| Services & Treatments | Online Booking |
| Appointment Management | Package Deals |
| Staff (Therapist) Assignment | Loyalty Integration |
| Billing | Customer App |

### Bar Operation

| Core | Add-ons |
|---|---|
| Drink Menu | Tab Management |
| Orders | Happy Hour Rules |
| Billing | QR Ordering |

---

## Business Growth Model

```
Small restaurant (day 1):
→ Core only → Menu + Orders + Tables + Billing
→ Simple. Works perfectly. Zero overwhelm.

Growing restaurant (6 months in):
→ Enable Table Reservation → more organized bookings
→ Enable QR Ordering → faster service
→ Enable Inventory → control food cost

Established restaurant (1 year):
→ Enable Delivery → new revenue channel
→ Enable KDS → kitchen efficiency
→ Full operation. Same platform.
```

> Start simple. Add power as needed. Never forced to use what you don't need.

---

## Design Principles Reinforced

- **Simple by Default:** Core works for any business out of the box
- **Powerful when needed:** Add-ons unlock complexity progressively
- **No assumptions:** Business decides what they need, when they need it
- **Progressive disclosure:** Add-on settings only appear after enabling
