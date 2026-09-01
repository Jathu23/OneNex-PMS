# Part 1 — Discovery & Scope Document
## Restaurant Management System (Phase 1)

**Product:** RESTLY — SaaS Restaurant Management System  
**Document Type:** Part 1 Scope — Discovery & Solution Definition  
**Version:** 1.0  
**Status:** Draft  
**Audience:** Founders, Product Lead, Lead Developer

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Target Market](#2-target-market)
3. [Business Challenges — 12 Core Problems](#3-business-challenges)
4. [User Categories & Pain Points](#4-user-categories--pain-points)
5. [Problem Statement](#5-problem-statement)
6. [Solution Overview](#6-solution-overview)
7. [Phase 1 Module Scope — What We Build](#7-phase-1-module-scope)
8. [Out of Scope — Phase 1](#8-out-of-scope--phase-1)
9. [Unique Selling Points (USP)](#9-unique-selling-points)
10. [Competitive Advantage Summary](#10-competitive-advantage-summary)
11. [Success Metrics](#11-success-metrics)
12. [Phase 1 Build Boundaries (Developer Reference)](#12-phase-1-build-boundaries)

---

## 1. Executive Summary

RESTLY is a **modular, SaaS Restaurant Management System** built specifically for the Sri Lanka Food & Beverage market. It starts as a restaurant-first product and expands into a full hospitality platform over time.

**The core idea:** A business owner signs up, selects their business type (restaurant, café, bakery, bar), and gets a pre-configured system. They start with a basic POS and unlock modules as they grow — QR ordering, delivery, PickMe integration, table reservations, and more. Each feature they add changes how the system behaves without needing a developer.

**Phase 1 goal:** Ship a working, revenue-generating Restaurant Management System that serves Sri Lanka's Food & Beverage businesses — from a single-counter café to a 200-seat full-service restaurant — and get 20 paying customers within 6 months of launch.

**Why now:** No dedicated, affordable, Sri Lanka-first restaurant management platform exists. International systems are 3–10× too expensive. Local systems are outdated. WhatsApp and paper are the current "system" for most restaurants.

---

## 2. Target Market

### Phase 1 — Food & Beverage (Sri Lanka)

| Business Category | Description | Est. Count (SL) |
|---|---|---|
| Full-Service Restaurants | Seated table service, waiters, full menu | ~1,200+ |
| Cafés & Coffee Shops | Counter service, beverages + light food | ~800+ |
| Quick-Service / Fast-Casual | High volume, counter ordering, delivery-heavy | ~1,500+ |
| Bakeries & Dessert Outlets | Walk-in + pre-order (cakes, pastries) | ~600+ |
| Bars & Pubs | Tab management, beverages, late night | ~300+ |
| **Total addressable (SL)** | | **~4,400+** |

### Primary Beachhead Target (First 6 Months)

- **Colombo + suburbs** (Colombo 1–15, Nugegoda, Dehiwala, Rajagiriya, Moratuwa)
- **Business size:** 1–5 staff, 10–80 seats, single location
- **Owner profile:** Tech-comfortable, uses WhatsApp daily, may already use basic POS
- **Monthly revenue:** LKR 300,000 – 3,000,000
- **Pain:** Too many disconnected tools, no delivery integration, manual WhatsApp, no reports

### Secondary Target (Month 6–18)

- Medium businesses: 5–25 staff, 80–200 seats
- Multi-branch chains (2–5 locations)
- Hotel restaurants and resort F&B

---

## 3. Business Challenges

These 12 challenges were identified through competitor analysis, Sri Lanka market research, and F&B owner interviews.

---

### Challenge 1 — Everything is Disconnected

**What happens:** A restaurant uses 4–6 separate tools daily — a basic POS for billing, WhatsApp manually for takeaway orders, a paper book for reservations, phone calls for delivery, a separate Excel sheet for inventory. None of these talk to each other.

**Impact:** Staff double-entering data, mistakes, missed orders, no unified picture of the business.

**Who feels it:** Cashier, manager, owner.

---

### Challenge 2 — No Offline Capability

**What happens:** Sri Lanka has frequent power outages (load shedding, infrastructure issues). When power or internet goes down, most digital POS systems stop working. Restaurants revert to paper, lose orders, cannot print bills.

**Impact:** Revenue loss during outages. Cannot operate. Customer frustration.

**Who feels it:** Cashier, waiter, kitchen staff, owner.

---

### Challenge 3 — PickMe & UberEats Orders Are Manually Re-Entered

**What happens:** A PickMe order comes in on a tablet. A staff member reads it and manually types it into the POS. This wastes time, causes errors, and the order is invisible to the kitchen unless manually printed.

**Impact:** Wrong orders, delays, rider waiting, customer complaints, no consolidated reports.

**Who feels it:** Cashier, kitchen staff, manager.

---

### Challenge 4 — No Sri Lanka Payment Integration

**What happens:** International POS systems do not support LankaQR (Sri Lanka's national QR payment standard), KOKO BNPL (buy-now-pay-later popular with younger customers), or local payment gateways. Restaurants lose sales or use separate terminals.

**Impact:** Slow checkout, multiple terminals on the counter, no unified payment records.

**Who feels it:** Cashier, customer, owner (reconciliation).

---

### Challenge 5 — Reservations Run on WhatsApp & Phone Calls

**What happens:** A customer WhatsApps "Table for 4 tonight 7pm?" — staff checks a paper diary or memory, replies manually, forgets to block the table, double-books. No confirmation sent, no reminder, customer no-shows with zero accountability.

**Impact:** Double bookings, empty tables from no-shows, poor customer experience, revenue loss.

**Who feels it:** Host, manager, customer.

---

### Challenge 6 — International Systems Are 3–10× Too Expensive

**What happens:** Toast POS costs USD 110+/month. Oracle OPERA is enterprise-priced. Even mid-market options like Lightspeed cost USD 60–80/month. At USD 1 = LKR 310, this is LKR 18,600–34,000/month — unaffordable for most Sri Lanka restaurants.

**Impact:** Restaurant owners stay on free/pirated basic POS software with zero support, no updates, no features.

**Who feels it:** Owner (budget decision).

---

### Challenge 7 — No Trilingual UI

**What happens:** Most restaurant staff in Sri Lanka are comfortable in Sinhala or Tamil. International systems are English-only. Even local systems are English-only. This means staff training takes longer, errors happen more often.

**Impact:** High staff training time, errors from misread English labels, resistance to using the system.

**Who feels it:** All staff (waiter, cashier, kitchen staff).

---

### Challenge 8 — No Real-Time Kitchen Visibility

**What happens:** Orders are shouted across the kitchen, written on paper slips, or printed on basic KOT tickets. There is no visibility into how long an order has been waiting, which items are ready, or which table is overdue.

**Impact:** Long waits, wrong items served first, no data on kitchen efficiency.

**Who feels it:** Chef, kitchen staff, waiter, customer.

---

### Challenge 9 — SLTDA TDL Compliance Is Manual

**What happens:** SLTDA-licensed restaurants must collect Tourism Development Levy (TDL — 1% or 0.5%) and file quarterly returns. This is calculated manually from revenue records, prone to errors, and often filed late with penalties.

**Impact:** Compliance risk, accountant time wasted, potential fines.

**Who feels it:** Owner, accountant.

---

### Challenge 10 — No Customer Data or Loyalty

**What happens:** A customer visits 20 times. The restaurant has no record of who they are, what they order, their birthday, their preferred table. There is zero loyalty program. No way to run a campaign. The customer feels like a stranger every visit.

**Impact:** No repeat customer incentive, no data for marketing, lower lifetime value.

**Who feels it:** Owner, marketing/manager.

---

### Challenge 11 — Takeaway & Pre-Orders Are Chaos

**What happens:** Phone orders are written on paper. Scheduled pickups have no system — the kitchen makes everything at once. Pre-orders for custom cakes or catering are tracked in a notebook. Items get ready too early or too late.

**Impact:** Wrong timing, customer waits, food quality drops, wasted preparation.

**Who feels it:** Kitchen staff, cashier, customer.

---

### Challenge 12 — No Meaningful Business Reports

**What happens:** The owner cannot see: which items are most profitable, what time of day is peak, which waiter takes the most orders, what is the actual food cost vs. sales. The only "report" is the day's cash count.

**Impact:** No data for menu decisions, pricing, staffing, or growth planning.

**Who feels it:** Owner, manager.

---

## 4. User Categories & Pain Points

### 4.1 Primary Users (Daily System Users)

---

#### Business Owner / Director

**Who:** The person who invested in the restaurant. May or may not be present daily.

**Goals:**
- Know how the business is performing without being physically present
- Control costs (food cost, staff cost, wastage)
- Grow revenue (new channels, loyalty, delivery)
- Stay compliant (VAT, TDL, IRD)

**Pain points:**
- No real-time visibility when away from restaurant
- Cannot trust the cash count — no independent verification
- No idea which menu items are actually profitable
- Compliance paperwork takes hours each quarter

**System touchpoints:** Manager app — reports, revenue, staff performance, tax summary

---

#### Restaurant Manager / Shift Lead

**Who:** The person who runs operations day-to-day. Opens and closes the restaurant. Approves discounts and voids.

**Goals:**
- Smooth shift from open to close with no incidents
- Staff have the right information at the right time
- No lost orders, no billing mistakes
- End-of-day report closes cleanly

**Pain points:**
- Constantly called over for manual approvals (discounts, voids, bill corrections)
- Cannot see all tables at once without physically walking the floor
- No way to quickly see if kitchen is backed up
- Closing takes 45+ minutes doing manual reconciliation

**System touchpoints:** Manager dashboard, floor map overview, void/discount approvals, end-of-day report

---

#### Head Waiter / Captain

**Who:** Senior waiter who manages the floor, assigns tables to waiters, handles VIP guests.

**Goals:**
- Assign tables efficiently as customers arrive
- Handle walk-ins vs. reservations without conflict
- Know which tables need attention (overdue, food ready, long wait)

**Pain points:**
- No real-time view of all tables — memory-based
- Reservations in a paper book — easy to miss or double-book
- No way to tell kitchen which table is priority

**System touchpoints:** Floor map, reservation list, waiter assignment

---

#### Waiter / Server

**Who:** Front-of-house staff taking orders and serving food.

**Goals:**
- Take order quickly and accurately
- Know when food is ready without going to kitchen every 5 minutes
- Generate bill when customer asks — no waiting for cashier
- Not make mistakes that result in manager call-over

**Pain points:**
- Writing orders on paper → re-entering on POS → errors
- No notification when kitchen finishes an item — must check physically
- Bill calculation errors (service charge, VAT) cause customer complaints
- Cannot see their own performance (orders taken, tips)

**System touchpoints:** POS tablet, KDS waiter notification, bill generation

---

#### Cashier

**Who:** Handles billing and payment. May be same person as waiter in small restaurants.

**Goals:**
- Generate accurate bills fast
- Process multiple payment types without errors
- Handle split bills, discounts without confusion
- Close cash drawer cleanly at shift end

**Pain points:**
- SC + VAT calculation done manually — often wrong
- Split bill across 4 people takes 10+ minutes
- Card machine separate from POS — amounts not synced
- Cash drawer reconciliation at closing is manual and error-prone

**System touchpoints:** POS billing screen, payment processing, cash drawer management

---

#### Chef / Kitchen Manager

**Who:** Manages the kitchen. Sets station assignments. Monitors order flow.

**Goals:**
- Know exactly what is ordered and in what sequence
- Manage multiple stations without paper chaos
- Communicate "ready" status to floor staff without shouting
- Track which items are slow / unavailable

**Pain points:**
- Paper KOT slips get lost, wet, burn, or blow off the pass
- No visibility on how long each table has been waiting
- Cannot instantly communicate "item unavailable" to POS
- No data on which items take longest to prepare

**System touchpoints:** KDS (kitchen display), station management, menu 86 (item disable)

---

#### Kitchen Staff / Line Cook

**Who:** Prepares food for specific stations. Not always tech-literate.

**Goals:**
- See clearly what needs to be made
- Mark items done when ready
- Not be overwhelmed during service

**Pain points:**
- Paper slips pile up — unclear what is oldest
- Cannot read handwriting on slips
- No way to ask waiter a question about order without leaving station

**System touchpoints:** KDS screen (large text, color-coded)

---

#### Delivery Rider (Own Fleet)

**Who:** Rider employed by the restaurant to deliver orders.

**Goals:**
- Know which orders to pick up and where to deliver
- Navigate to customer efficiently
- Confirm delivery and get next assignment

**Pain points:**
- Called by phone for each delivery — no app
- No proof of delivery
- Not paid fairly if delivery count is not tracked

**System touchpoints:** Rider mobile app

---

### 4.2 Secondary Users (Admin / Setup)

| User | Frequency | Key Need |
|------|-----------|----------|
| Accountant | Monthly/Quarterly | VAT + TDL reports, IRD-ready format |
| System Admin (IT) | One-time setup | Hardware integration, printer setup |
| Supplier | Weekly | Purchase orders, low-stock alerts |

---

### 4.3 End Customers (Indirect Users)

| Customer Type | Channel | Key Need |
|---|---|---|
| Dine-in customer | Floor (via waiter) | Fast service, accurate bill, easy payment |
| QR ordering customer | Own phone at table | Browse menu, order, track, pay from phone |
| Takeaway customer | Counter / phone / online | Know when order is ready, accurate ETA |
| Delivery customer | PickMe / own ordering | Real-time tracking, accurate ETA |
| Pre-order customer (bakery) | Phone / WhatsApp / online | Confirmation, reminder, on-time pickup |

---

## 5. Problem Statement

### The Core Problem

**Sri Lanka's 4,400+ Food & Beverage businesses are running on paper, WhatsApp, and disconnected tools — with no affordable, unified system built for how Sri Lanka restaurants actually work.**

Specifically:
- No system handles dine-in + takeaway + delivery + QR + reservations in one place
- No system integrates with LankaQR, PickMe, or KOKO BNPL
- No system works offline during power cuts
- No system has Sinhala or Tamil UI
- No system automates SLTDA TDL quarterly compliance
- International systems cost LKR 18,000–34,000/month — 3–10× the market rate
- Local alternatives are outdated, single-feature tools with no support

The result: restaurant owners make business decisions based on gut feel instead of data. Staff make errors due to manual processes. Revenue is lost from missed delivery integrations. Customers get poor experiences. Compliance is missed.

### Who Suffers Most

**Primary:** Owner of a 15–80 seat restaurant in Colombo, managing 4–8 staff, doing LKR 500K–2M/month revenue, currently using basic POS + WhatsApp + paper diary.

**The moment of pain:** A PickMe order comes in on a separate tablet while the cashier is handling 3 walk-in customers. The order is re-entered manually, the kitchen doesn't see it for 6 minutes, the PickMe rider arrives before the food is ready, the customer gets a cold order and leaves a 1-star review. This happens 5–10 times per day.

### What We Solve

A **single, modular system** that connects every part of the restaurant operation — from the moment a customer books a table or places an order to the moment the bill is paid and the report is generated — while working offline, in Sinhala/Tamil/English, with LankaQR and PickMe built in, at a price every Sri Lanka restaurant can afford.

---

## 6. Solution Overview

### What We Are Building

A SaaS Restaurant Management System with a **modular feature toggle system**. The business owner chooses their business type during onboarding and gets a pre-configured system. They can enable additional modules as they grow.

### The Modular Model

```
Business signs up
      ↓
Selects business type → Pre-built config applied
(Full-Service / Café / QSR / Bakery / Bar)
      ↓
Core POS is always active
      ↓
Owner enables modules in Settings:
  ├── KDS (Kitchen Display)
  ├── KOT Printing
  ├── Table Management + Floor Map
  ├── QR Table Ordering
  ├── Takeaway Management
  ├── Scheduled Takeaway / Pre-Orders
  ├── Own Delivery Fleet
  ├── PickMe Integration
  ├── Table Reservations
  ├── Loyalty Program
  └── ...more
      ↓
System UI, workflows, and reports adapt automatically
```

### Multi-Screen System

| App / Screen | Who Uses It | Device |
|---|---|---|
| **POS App** | Waiter, cashier, head waiter | Tablet (10–12 inch) or desktop touch |
| **Manager App** | Manager, owner | Desktop, tablet, phone |
| **KDS Screen** | Kitchen staff, chef | Large monitor/TV in kitchen |
| **Order Display** | Customers (QSR) | TV screen facing customer area |
| **Rider App** | Delivery riders | Android phone |
| **Customer App (QR)** | Dine-in customers | Own phone (browser, no install) |
| **Online Ordering** | Takeaway/delivery customers | Own phone (browser) |

### How Pricing Works

| Tier | Monthly (LKR) | Who It's For |
|------|--------------|--------------|
| **Basic** | 3,000 – 5,000 | Small café, bakery counter — POS only |
| **Standard** | 8,000 – 12,000 | Restaurant with KDS, reservations, takeaway |
| **Growth** | 15,000 – 25,000 | Multi-channel: delivery + PickMe + CRM |

*No per-cover fees. No commission on orders. Flat monthly subscription.*

---

## 7. Phase 1 Module Scope

These are the modules we **will build in Phase 1**. Each module has a defined scope — what's included and what's deferred.

---

### Module 1 — POS Core ✅ MUST BUILD

**What it does:** The central order-taking and billing system.

**In scope:**
- Create new order (dine-in, takeaway, walk-in)
- Add items from menu (search, category browse)
- Item modifiers (size, add-ons, notes)
- Quantity, void single item
- Discount — fixed amount, percentage (manager code required)
- Bill generation — subtotal + Service Charge (if enabled) + VAT 18%
- Payment methods: Cash, Card, LankaQR, KOKO BNPL
- Cash change calculator
- Receipt — print or WhatsApp
- Shift open/close, cash drawer float
- Order types: Dine-in | Takeaway | Delivery
- Pre-pay mode (café/QSR) and Post-pay mode (restaurant/bar)

**Deferred:**
- Split bill (Phase 1B — priority)
- Tab management (Bar profile — Phase 1B)
- Kiosk / self-service mode (Phase 2)

---

### Module 2 — Table & Floor Management ✅ MUST BUILD

**What it does:** Visual floor map showing all tables and their status.

**In scope:**
- Floor map with tables (custom layout, drag-and-drop setup)
- Table statuses: Available / Occupied / Reserved / Cleaning
- Open a table — enter cover count
- Assign waiter to table
- Table timer (how long occupied)
- Merge tables (for large groups)
- Table status changes reflected in real-time across all devices

**Deferred:**
- Multiple floor sections (indoor/outdoor/terrace) — Phase 1B
- Custom table shapes (round, long) — Phase 1B

---

### Module 3 — Kitchen Display System (KDS) ✅ MUST BUILD

**What it does:** Digital display in kitchen showing orders in real-time.

**In scope:**
- Order cards appear instantly when order is sent from POS
- Items listed per order card with quantities
- Timer per order card (age of order)
- Color coding: grey (< 10 min) → amber (10–15 min) → red (> 15 min)
- Mark individual item DONE
- Mark full order DONE (BUMP ALL) → waiter notified
- Undo last bump
- Order type tag: Dine-in | Takeaway | PickMe | UberEats
- VIP / special notes visible on card

**Deferred:**
- Multiple KDS stations (grill, cold, bar) — Phase 1B (single station Phase 1)
- Station routing per menu category — Phase 1B

---

### Module 4 — KOT Printing ✅ MUST BUILD

**What it does:** Prints a Kitchen Order Ticket (paper) at the kitchen printer when an order is placed.

**In scope:**
- Auto-print on order send
- Manual reprint
- Table number, waiter name, item list, modifiers, timestamp
- Printer configuration (IP or USB thermal printer)

---

### Module 5 — Menu Management ✅ MUST BUILD

**What it does:** Full menu catalog management.

**In scope:**
- Categories (create, sort, enable/disable)
- Menu items (name, description, price, image, category)
- Item modifiers (choices + add-ons with price delta)
- Item enable/disable (86 an item when unavailable)
- Bulk price update
- VAT-inclusive vs. VAT-exclusive pricing flag

**Deferred:**
- Time-based pricing (happy hour) — Phase 1B (Bar profile)
- Channel-specific pricing (dine-in vs. delivery price) — Phase 2
- Menu import/export — Phase 1B

---

### Module 6 — Takeaway Management ✅ MUST BUILD

**What it does:** Manages takeaway orders from creation to pickup.

**In scope:**
- Create takeaway order (walk-in counter, phone order entry by staff)
- Estimated pickup time
- Customer name + phone
- Order sent to KDS with [TAKEAWAY] tag
- Status: Pending → In Kitchen → Ready → Collected
- WhatsApp notification: "Order received" + "Order ready for pickup"
- Takeaway board (kanban view)

**Deferred:**
- Customer self-service online ordering — Phase 1B
- Estimated time calculation from recipe prep times — Phase 1B

---

### Module 7 — Scheduled Takeaway / Pre-Orders ✅ MUST BUILD

**What it does:** Creates orders for future pickup — critical for bakeries and catering.

**In scope:**
- Create scheduled order with future date/time
- Customer details, items, deposit taken (optional)
- Auto-alert kitchen at correct time based on prep time
- Production queue view (especially for bakery)
- WhatsApp reminder to customer before pickup
- Pre-orders list with upcoming calendar view

---

### Module 8 — QR Table Ordering ✅ MUST BUILD

**What it does:** Customer scans QR on table, browses menu, orders directly from phone.

**In scope:**
- Unique QR per table (generated by system)
- Customer opens menu in browser (no app download)
- Browse categories, select items, add modifiers
- Place order → goes to KDS + POS
- Order status tracking on customer's phone
- Payment: LankaQR from phone (Phase 1B), otherwise pay at counter

**Deferred:**
- Card payment from phone — Phase 1B
- Customer loyalty login from QR menu — Phase 2

---

### Module 9 — Table Reservations ✅ MUST BUILD

**What it does:** Manage advance table bookings. (Full detail in Module-TableReservation-Detailed.md)

**In scope (Phase 1 — Launch):**
- Staff creates reservation (name, phone, party size, date, time, table)
- Online booking widget (embeddable on website)
- Availability check (no double booking)
- WhatsApp confirmation sent automatically
- 24-hour reminder WhatsApp
- Cancellation link in WhatsApp message
- Reservation timeline view (staff)
- Walk-in queue management
- Floor map shows reserved tables

**Deferred to Phase 2:**
- AI WhatsApp chatbot booking
- Credit card hold / deposit
- Automatic no-show detection

---

### Module 10 — PickMe Integration ✅ MUST BUILD

**What it does:** Receives PickMe Food orders directly into the system without manual re-entry.

**In scope:**
- Connect PickMe Business account via API
- Auto-receive incoming orders → appear in KDS + takeaway board
- Accept / reject order from system
- Menu sync (items + prices push to PickMe)
- Order status update → PickMe platform notified
- PickMe commission tracked in reports

**Deferred:**
- Auto-pricing (different price for PickMe) — Phase 2
- PickMe promotions management — Phase 2

---

### Module 11 — Inventory (Basic) ✅ MUST BUILD

**What it does:** Track ingredient stock and deduct based on recipes.

**In scope:**
- Ingredient catalog (name, unit, current stock level)
- Recipe costing — link menu items to ingredients + quantities
- Auto-deduction when order completed
- Low stock alert (threshold per ingredient)
- Manual stock update (delivery received, wastage)
- Basic purchase order to supplier (email/WhatsApp)
- Cost of goods sold (COGS) in reports

**Deferred:**
- Barcode scanning for stock intake — Phase 2
- Supplier portal — Phase 2
- Batch / shelf life tracking (Bakery) — Phase 1B

---

### Module 12 — Staff Management ✅ MUST BUILD

**What it does:** Manage staff accounts, roles, permissions, and shifts.

**In scope:**
- Staff accounts (name, role, phone, PIN login)
- Roles: Owner | Manager | Head Waiter | Waiter | Cashier | Chef | Kitchen Staff | Rider
- Permissions per role (what they can see/do in POS)
- Shift scheduling (which staff works which shift)
- Clock in / clock out via POS PIN
- Basic attendance log

**Deferred:**
- Payroll integration — Phase 2
- Performance metrics per waiter — Phase 1B

---

### Module 13 — Payments ✅ MUST BUILD

**What it does:** Process all payment types at checkout.

**In scope:**
- Cash (with change calculator)
- Card (manual entry, connect to card terminal)
- LankaQR (generate QR for customer to scan via any bank app)
- KOKO BNPL (customer pays in instalments)
- Split payment (partial cash + partial card)
- Tax calculation: VAT 18%, Service Charge 10% (configurable per tenant)
- TDL 1% / 0.5% calculation
- IRD-compliant receipt format
- Daily payment reconciliation report

**Deferred:**
- Alipay+ — Phase 1B
- Online card payment gateway — Phase 1B

---

### Module 14 — Reports & Analytics (Basic) ✅ MUST BUILD

**What it does:** Give the owner and manager actionable data.

**In scope:**
- Daily sales summary (total, by order type, by payment method)
- Shift report (per cashier/waiter)
- Menu performance (items sold, revenue per item)
- Tax report (VAT collected, SC collected, TDL)
- Void/discount report
- Basic inventory report (consumption vs. stock)
- Export to PDF / CSV

**Deferred:**
- Advanced analytics (trends, forecasting) — Phase 2
- Custom date range pivot — Phase 1B

---

### Module 15 — WhatsApp Notifications ✅ MUST BUILD

**What it does:** Send automated WhatsApp messages to customers at key events.

**In scope:**
- Reservation confirmation
- Reservation 24h reminder
- Takeaway order received
- Takeaway order ready for pickup
- Delivery dispatched
- Delivery delivered
- Receipt (on request)
- Template messages pre-approved with Meta

**Deferred:**
- Two-way WhatsApp inbox — Phase 2
- WhatsApp marketing campaigns — Phase 2
- AI WhatsApp chatbot — Phase 2

---

### Module 16 — Tax & Compliance ✅ MUST BUILD

**What it does:** Automate Sri Lanka tax calculations and compliance.

**In scope:**
- VAT 18% — calculated on every bill
- Service Charge 10% — configurable ON/OFF per tenant
- TDL 1% (turnover > LKR 500K/month) or 0.5% — calculated automatically
- IRD-compliant receipt format (business reg number, tax breakdown)
- Monthly VAT summary report
- Quarterly TDL filing report (ready to submit to SLTDA)

---

### Module 17 — Settings & Feature Toggles ✅ MUST BUILD

**What it does:** Business configuration and feature management.

**In scope:**
- Business profile (name, address, logo, business reg, contact)
- Business type selector (changes pre-config)
- Feature ON/OFF toggles (per module)
- Tax configuration (VAT rate, SC rate, TDL rate)
- Floor map setup (add/move/remove tables)
- Printer configuration
- WhatsApp number setup
- Staff roles and permissions
- Subscription plan and billing

---

### Module 18 — Offline Mode ✅ MUST BUILD

**What it does:** System continues to function during internet/power outages.

**In scope:**
- POS continues to work offline (create orders, bill, accept cash)
- KDS continues to display existing orders offline
- All offline actions queued locally (IndexedDB / Service Worker)
- On reconnection: auto-sync all queued actions to server
- Offline banner shown to staff

**Not supported offline:**
- PickMe / UberEats orders (platform-side)
- WhatsApp messages (queue and send on reconnect)
- LankaQR / card payment (require internet)

---

## 8. Out of Scope — Phase 1

The following are explicitly NOT built in Phase 1. They are planned for Phase 2 or later.

| Feature | Reason Deferred |
|---------|----------------|
| UberEats Integration | PickMe first, UberEats after market validation |
| Online Ordering Page (branded) | Requires payment gateway setup — Phase 1B |
| Loyalty Program | Valuable but not critical for MVP |
| CRM & WhatsApp Campaigns | Requires customer data accumulation first |
| AI WhatsApp Chatbot | Reservation Phase 2 feature |
| Multi-Branch Management | Single location first |
| Hotel PMS | Separate product phase |
| Kiosk / Self-Service Ordering | Hardware dependency, Phase 2 |
| Alipay+ / WeChat Pay | Niche, Phase 1B |
| Payroll & HR | Out of restaurant scope |
| Supplier Portal | Phase 2 |
| Advanced Analytics / BI | Phase 2 |
| Multiple KDS Stations | Phase 1B after single station proven |
| Room Service | Hotel module only |

---

## 9. Unique Selling Points

### USP 1 — One System, Every Channel

Dine-in table service, counter takeaway, scheduled pre-orders, QR ordering, delivery (own riders), PickMe — **all in one system, all in one report**. No other affordable system in Sri Lanka unifies all these channels.

> *"I used to have 4 tablets on my counter. Now I have one screen for everything."*

---

### USP 2 — Built for Sri Lanka, Not Imported

- **LankaQR** — the national QR payment standard, built-in
- **KOKO BNPL** — Sri Lanka's buy-now-pay-later service, built-in
- **PickMe Food** — direct API integration, no manual re-entry
- **SLTDA TDL** — quarterly filing automated, not manual
- **Sinhala + Tamil + English** — trilingual UI for all staff
- **LKR pricing** — no USD conversion

No international system offers even 3 of these. We offer all 6.

---

### USP 3 — Offline-First (Built for Sri Lanka Power Cuts)

Service Worker + IndexedDB architecture means the POS continues to take orders, print bills, and accept cash payments even when the internet is down. Orders sync automatically when connection returns.

> *"Load shedding happens. Your POS shouldn't stop."*

---

### USP 4 — Modular — Grows With the Business

A new café with 5 tables starts with just POS + basic KDS at LKR 3,000/month. As they grow, they unlock: takeaway → QR ordering → delivery → PickMe → reservations. Each unlock takes 30 seconds. No reinstall, no new system, no data migration.

> *"Start simple. Grow without switching systems."*

---

### USP 5 — Fair LKR Pricing, No Hidden Fees

- **No per-cover fees** (unlike OpenTable / SevenRooms charging LKR 600–1,800 per booking)
- **No commission on delivery** (unlike PickMe 30% / UberEats 25%)
- **Flat monthly LKR subscription** — no USD exchange risk
- **Free WhatsApp notifications** included in plan (no per-message charges)

> *"We don't earn more when you earn more. Flat fee, always."*

---

### USP 6 — SLTDA TDL on Autopilot

The system automatically calculates Tourism Development Levy on every bill, generates the quarterly filing report in SLTDA-required format, and sends an alert when the filing deadline approaches. What takes an accountant 2 hours per quarter takes 2 clicks.

---

## 10. Competitive Advantage Summary

| Factor | RESTLY | Toast / Lightspeed | Local POS (CXPOS etc.) | WhatsApp + Paper |
|---|---|---|---|---|
| All channels unified | ✅ | Partial | ❌ | ❌ |
| LankaQR + KOKO | ✅ | ❌ | ❌ | ❌ |
| PickMe integration | ✅ | ❌ | ❌ | ❌ |
| Offline mode | ✅ | ❌ | Partial | ✅ (paper) |
| Sinhala / Tamil UI | ✅ | ❌ | ❌ | N/A |
| SLTDA TDL auto | ✅ | ❌ | ❌ | ❌ |
| WhatsApp native | ✅ | ❌ | ❌ | Manual |
| LKR pricing | ✅ | ❌ (USD) | ✅ | Free |
| Monthly cost | LKR 3K–25K | LKR 18K–34K | LKR 2K–8K | Free |
| Real-time KDS | ✅ | ✅ | ❌ | ❌ |
| Table reservations | ✅ | Partial | ❌ | WhatsApp |
| Reports + tax | ✅ | ✅ | Basic | ❌ |

---

## 11. Success Metrics

### Phase 1 Launch Metrics (Month 1–6)

| Metric | Target |
|--------|--------|
| Paying customers | 20 by Month 6 |
| Monthly Recurring Revenue | LKR 200,000+ by Month 6 |
| Average revenue per customer | LKR 10,000/month |
| Churn rate | < 10% monthly |
| NPS (Net Promoter Score) | > 40 |
| Time to first order (onboarding) | < 30 minutes |
| System uptime | > 99% |
| Offline sync success rate | > 99.5% |
| WhatsApp delivery rate | > 95% |
| Support tickets per customer/month | < 2 |

### Phase 1 Product Metrics (Technical)

| Metric | Target |
|--------|--------|
| POS order creation time | < 60 seconds |
| KDS order display latency | < 2 seconds |
| Bill generation time | < 5 seconds |
| PickMe order sync latency | < 30 seconds |
| Reservation confirmation WhatsApp | < 10 seconds after booking |
| Page load time (POS app) | < 2 seconds on 4G |
| Offline queue sync on reconnect | < 30 seconds for last 50 orders |

---

## 12. Phase 1 Build Boundaries (Developer Reference)

### What Phase 1 Is

Phase 1 is a **working, deployable, revenue-generating SaaS product** for the Sri Lanka F&B market. It is not an MVP with placeholder screens — it must be production-quality for the modules listed in Section 7.

### Build Priority Order

| Week | Focus |
|------|-------|
| 1–2 | Core architecture, auth, multi-tenancy, feature flags, tenant onboarding |
| 3–4 | POS Core — counter mode + table service mode |
| 5–6 | Menu Management + KDS + KOT printing |
| 7–8 | Table Management + Floor Map + real-time (SignalR) |
| 9–10 | Takeaway Module + Scheduled Orders |
| 11–12 | PickMe Integration + Delivery (own fleet) |
| 13–14 | Table Reservations (Phase 1 scope) + WhatsApp |
| 15–16 | Payments (LankaQR, KOKO) + Tax & Billing |
| 17–18 | Reports + Settings + Offline Mode |
| 19–20 | QR Ordering + hardening + staging |
| 21–22 | Beta testing with 3–5 real restaurants |
| 23–24 | Bug fixes, performance, launch |

### Non-Negotiable Technical Requirements

These are not features — they are architectural requirements that must be in place from Day 1:

| Requirement | Why |
|-------------|-----|
| Multi-tenancy (TenantId on every record) | SaaS — every customer is isolated |
| Feature flag system (per tenant) | The modular model depends on this |
| Offline-first (Service Worker + IndexedDB) | Sri Lanka power cuts |
| SignalR real-time | KDS, floor map, order feed |
| WhatsApp Business API connected | Core to all notification flows |
| Result pattern (no business exceptions) | Clean error handling throughout |
| Architecture tests (NetArchTest) | Module boundaries enforced in CI |
| Schema-per-module (PostgreSQL) | Module isolation, clean migrations |
| Audit log (every POS action) | Void/discount accountability |

### The Modular Rule

> **A feature that is disabled must not appear anywhere in the UI, and its API endpoints must return 403 if called.**

This rule must hold for every module. Feature flags are checked at:
1. Backend command handler (returns `FeatureNotEnabled` error)
2. Frontend — navigation item hidden
3. Frontend — component not rendered
4. API gateway — route can be toggled off

### Definition of Done for Each Module

A module is considered complete when:
- [ ] Backend commands + queries implemented and tested
- [ ] API endpoints documented (Swagger)
- [ ] Frontend screens implemented and connected to API
- [ ] Feature flag gate works (ON = visible, OFF = hidden + blocked)
- [ ] Offline mode tested (POS core modules)
- [ ] WhatsApp notification sends correctly (where applicable)
- [ ] Works on tablet (1024px) and desktop (1280px+)
- [ ] Works on 4G mobile connection
- [ ] At least one real restaurant has tested it

---

## Document Relationships

| Document | What It Contains |
|----------|-----------------|
| **This document (Scope-Document.md)** | Problems, users, solution scope, USPs, success metrics |
| Architecture-Proposal.md | Technical architecture, patterns, code standards |
| Business-Categories-OperationalFlows.md | How each business type operates, feature matrix |
| Restaurant-Module-Detailed.md | Every sub-module, features, day-to-day flow |
| Module-TableReservation-Detailed.md | Reservation module — 4 phases, DB schema, flows |
| Part1-Market-Research.md | Global competitor analysis |
| Part1-SriLanka-Market-Research.md | Sri Lanka market, competitors, pricing |
| Product-Overview.md | A-to-Z product overview for founders |

---

*Part 1 Discovery — Scope Document*  
*RESTLY — SaaS Restaurant Management System*  
*Next: Part 2 — Requirements & UX Definition*
