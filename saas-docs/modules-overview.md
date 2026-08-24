# SaaS Platform — Modules Overview
> Status: Draft | Last Updated: 2026-08-21

---

## 1. CORE PLATFORM
> Ella business-kukkum automatically included. Separate enable/disable illai.

| Sub Module | Description |
|-----------|-------------|
| Authentication | Login, Role Management, Staff PIN Login, Session Management |
| Subscription & Billing | Plan Management, Module Enable/Disable, Invoice, Payment Gateway, Trial |
| Multi-Branch | Branch Creation, Branch Config, Centralized Dashboard, Branch Reports |
| CRM | Customer Profiles, Purchase History, Loyalty Points, Segments, VIP/Blacklist |
| Notifications | Push, Email, SMS, WhatsApp, Custom Templates |
| Analytics & Reports | Live Dashboard, Sales Reports, Scheduled Reports, Export |
| Settings & Config | Business Profile, Tax, Currency, Working Hours, Receipt Customize |

---

## 2. RESTAURANT MODULE

| Sub Module | Description |
|-----------|-------------|
| POS | Order Taking, Bill Generation, Cash/Card/UPI, Receipt Print, Void/Refund, Discount, Split Bill |
| Menu Management | Categories, Items, Variants, Modifiers, Pricing, Availability Windows, Images |
| Order Management | Dine-in, Takeaway, Delivery — Order Status Tracking |
| Table Management | Floor Map, Table Status, Merge/Split, Assign to Staff |
| Table Reservation *(depends on Table Management)* | **Offline:** Staff manually books slot via system (phone call / walk-in); **Online:** Customer self-books via app/web — Booking Slots, Waitlist, Confirmation, No-show Tracking |
| KDS (Kitchen Display) | Single KDS, Multiple KDS, Station Routing, Order Priority, Ready Notification |
| QR Ordering | Table QR, Takeaway QR, Customer Self-Order, Pay-at-Table |
| Takeaway | Scheduled Pickup, Estimated Time, Status Updates |
| Delivery | In-house Delivery, Delivery Boy Assign, Route Tracking, 3rd Party Integration |
| Inventory | Ingredient Stock, Recipe Mapping, Auto Deduction on Order, Low Stock Alert, Wastage Log |
| Staff Management | Roles, Shift Management, Tip Tracking, Performance Reports |

---

## 3. HOTEL MODULE

| Sub Module | Description |
|-----------|-------------|
| Room Management | Room Types, Setup, Amenities, Status (Clean/Dirty/Maintenance) |
| Reservations | Online Booking, Walk-in, OTA Integration, Group Booking |
| Front Desk | Check-in, Check-out, Room Assignment, Early/Late Requests |
| Guest Folio | Charge Accumulation from all modules, Partial Payment, Folio Split, Final Bill |
| Housekeeping | Room Status, Task Assignment, Inspection, Lost & Found |
| Room Service | In-room QR → Restaurant KDS, Service Requests |
| Rate Management | Seasonal Rates, Promo Rates, Corporate Rates, OTA Rate Sync |
| Guest Management | Guest Profile, Stay History, Preferences, VIP Flags |
| Channel Manager | OTA Sync, Availability Update, Rate Push |

---

## 4. BAR MODULE

| Sub Module | Description |
|-----------|-------------|
| Bar POS | Quick Order, Tab Management, Open/Close Tab |
| Tab Management | Open Tab per Customer/Table, Add Items, Transfer Tab to Hotel Folio |
| Inventory | Liquor Stock, Bottle Tracking, Pour Cost, Low Stock Alert |
| Happy Hour | Time-based Pricing Rules, Auto Apply Discounts |

---

## 5. EVENT MODULE

| Sub Module | Description |
|-----------|-------------|
| Event Management | Event Creation, Types (Public/Private/Recurring), Capacity |
| Ticketing | Paid/Free/Invite-only, Ticket Tiers, QR Ticket Generation |
| Seating | Assigned Seating, Open Seating, Seating Map |
| Entry Management | QR Scan at Entry, Guest List Check-in |
| Event + Hotel Combo | Room Booking linked to Event, Package Deals |
| Event + Catering | Food Orders linked to Event via Restaurant Module |
| Refund & Cancellation | Policy Setup, Auto Refund |

---

## 6. RETAIL MODULE

| Sub Module | Description |
|-----------|-------------|
| Product Management | Products, Categories, Variants, Barcode |
| POS | Sale, Return, Exchange, Receipt |
| Inventory | Stock Management, Purchase Orders, Supplier Management, Stock Alerts |
| Pricing | Discount Rules, Promo Pricing, Bundle Offers |

---

## 7. SPA / WELLNESS MODULE

| Sub Module | Description |
|-----------|-------------|
| Service Management | Service List, Duration, Pricing |
| Appointment Booking | Online Booking, Walk-in, Staff Assign |
| Therapist Management | Availability, Schedule, Performance |
| Charges | Direct Pay or Charge to Hotel Room Folio |

---

## 8. INTERCONNECTION LAYER
> Modules interact with each other through this layer.

| Integration | Flow |
|-------------|------|
| Hotel ↔ Restaurant | Room guest → order food → charge to folio |
| Hotel ↔ Bar | Room guest → bar tab → charge to folio |
| Hotel ↔ Spa | Room guest → spa booking → charge to folio |
| Hotel ↔ Event | Room booking + event ticket combo |
| Restaurant ↔ Inventory | Order placed → auto deduct ingredients |
| Event ↔ Restaurant | Event catering orders → Restaurant KDS |
| All ↔ CRM | Every transaction → customer history update |
| All ↔ Analytics | Every module feeds into reports |
| All ↔ Notifications | Every action triggers relevant notification |

---

## Notes
- Table Management is the base sub-module — always present in Restaurant Module
- Table Reservation is an optional add-on that depends on Table Management
  - Cannot enable Table Reservation without Table Management
  - Table Management can be used alone (walk-in only, no reservations)
- Folio system is the key integration point for Hotel with all other modules
- All modules share Core Platform (Auth, CRM, Notifications, Analytics)
