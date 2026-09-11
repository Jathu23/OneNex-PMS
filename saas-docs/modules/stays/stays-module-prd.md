# Stays Module — Product Requirements Document
> Status: Awaiting Product Team Input
> Purpose: Product team fills this → Backend team builds from this → No mid-development changes
> Rule: Every question must have a final answer before development starts
> Note: C1 (Room Setup), C7 (Rate Plans), C5 (Guest Profile) already have technical designs — product team confirms or corrects those sections.

---

## How to Use This Document

- Read each section carefully
- Answer every question — no skipping
- Mark confirmed sections: **CONFIRMED** or **NEEDS CHANGE**
- For new sections (C2, C3, C4, C6, C8): answer fully
- When all sections = CONFIRMED → doc is locked → backend builds

---

## Section 0 — What Is the Stays Operation?

### 0.1 Who Uses Stays?

The Stays operation is for businesses that offer accommodation.

**Q1:** What types of properties will OneNex Stays support in V1?

| Property Type | V1 / Phase 2 / Not needed |
|---|---|
| Hotel (standard rooms, multiple floors) | |
| Guesthouse / B&B | |
| Boutique hotel | |
| Resort | |
| Serviced apartment | |
| Hostel (dorm beds) | |
| Villa / holiday home (single property) | |

**Product Team Answer:**
> *(Write answer here)*

---

### 0.2 Who Operates the Stays System?

| Role | Needs access? | What they do |
|---|---|---|
| Business Owner | Yes / No | Configure, full access |
| Front Desk Manager | Yes / No | Check-in, check-out, reservations |
| Front Desk Staff | Yes / No | Check-in, check-out |
| Housekeeping Supervisor | Yes / No | Assign tasks, inspect rooms |
| Housekeeper | Yes / No | Update room status |
| Reservations Staff | Yes / No | Create/modify bookings |
| Night Auditor | Yes / No | Run night audit |

**Product Team Answer:**
> *(Write answer here)*

---

## Section 1 — Property & Room Setup (C1)
> C1 technical design exists. Product team confirms or requests changes.

### 1.1 Property Structure

Current design:
```
Business
  └── Building (optional grouping)
        └── Floor
              └── Room Type (template — e.g., "Deluxe Sea View")
                    └── Room (physical — e.g., "Room 101")
```

**Q1:** Is Building an optional grouping (not all hotels have buildings)?
- [ ] Yes — optional, small hotels can skip
- [ ] No — always required

**Q2:** Is Floor required or optional?
- [ ] Required — every room must have a floor
- [ ] Optional — some properties don't use floors

**Product Team Answer:**
> *(Write answer here)*

---

### 1.2 Room Types

Current design: Room Type = template (Deluxe, Superior, Suite etc.)

**Q1:** Can one physical room belong to multiple room types?
- [ ] No — one room = one room type only
- [ ] Yes — a room can be switched between types

**Q2:** Room categories — what fixed categories does OneNex support?

| Category | Include? |
|---|---|
| Standard | |
| Deluxe | |
| Superior | |
| Suite | |
| Presidential Suite | |
| Studio | |
| Villa | |
| Dormitory | |
| Custom (owner-defined) | |

**Q3:** What details does a Room Type need?

| Detail | Required / Optional / Not needed |
|---|---|
| Name | |
| Category (Standard, Deluxe etc.) | |
| Max occupancy (adults) | |
| Max occupancy (children) | |
| Max occupancy (infants) | |
| Base price | |
| Room size (sqm) | |
| Bed configuration (King, Twin etc.) | |
| View type (Sea, Garden, City) | |
| Floor (min/max) | |
| Smoking / Non-smoking | |
| Accessibility (wheelchair etc.) | |
| Pet-friendly | |
| Photos | |
| Short description | |
| Long description | |

**Product Team Answer:**
> *(Write answer here)*

---

### 1.3 Individual Rooms

**Q1:** What details does each physical room need?

| Detail | Required / Optional / Not needed |
|---|---|
| Room number | |
| Room name (optional, e.g., "The Penthouse") | |
| Floor | |
| Building | |
| Room type (which template it follows) | |
| Connecting room (links to adjacent room) | |
| Notes for staff (internal, not shown to guest) | |

**Q2:** Can a room's details differ from its Room Type?
```
Room Type: Deluxe — max 2 adults
Room 501 (Deluxe): actually fits 3 (larger than others)
Can Room 501 override the Room Type's occupancy?
```
- [ ] Yes — room-level overrides allowed for specific fields
- [ ] No — room must follow room type exactly

**Product Team Answer:**
> *(Write answer here)*

---

### 1.4 Room Status

Which room statuses does V1 need?

| Status | Code | Include in V1? |
|---|---|---|
| Vacant Dirty | VD | Yes / No |
| Vacant Clean | VC | Yes / No |
| Occupied Dirty | OD | Yes / No |
| Occupied Clean | OC | Yes / No |
| Inspected | INS | Yes / No |
| Do Not Disturb | DND | Yes / No |
| Out of Order | OOO | Yes / No |
| Out of Service | OOS | Yes / No |

**Rule to confirm:**
> Front desk can only assign rooms with status = INSPECTED to arriving guests. Correct?

- [ ] Yes — INSPECTED only
- [ ] No — VC (Vacant Clean) is enough, no inspection required
- [ ] Depends on hotel setting — configurable

**Product Team Answer:**
> *(Write answer here)*

---

### 1.5 Amenities

Current design: Amenities are a shared list (TV, WiFi, Pool, Gym...) assigned to Room Types.

**Q1:** Are amenities managed at Business level (shared across all room types) or per Room Type separately?
- [ ] Business level — one master list, room types pick from it
- [ ] Per Room Type — each type has its own amenity list

**Q2:** Does OneNex provide a default amenity list or does owner build from scratch?
- [ ] OneNex provides standard list (WiFi, TV, AC, Pool...) + owner can add custom
- [ ] Owner builds from scratch
- [ ] OneNex provides list, owner cannot add custom ones

**Product Team Answer:**
> *(Write answer here)*

---

### 1.6 Room Out of Order (OOO)

```
Scenario: Room 101 has a broken AC. Needs 3 days to fix.
It must not be bookable during those 3 days.
```

**Q1:** How does owner mark a room as OOO?
- [ ] Mark it inactive — no dates, just "unavailable until fixed"
- [ ] Set a date range — OOO from X to Y, automatically reopens
- [ ] Both options available

**Product Team Answer:**
> *(Write answer here)*

---

## Section 2 — Rate Plans (C7)
> C7 technical design exists. Product team confirms or requests changes.

### 2.1 What Is a Rate Plan?

Current design: Rate Plan = one complete pricing unit:
```
Name + Description
+ Base price per room type
+ Cancellation policy
+ Payment terms (Pay Now / Deposit / Pay at Hotel)
+ Which channels it applies to (Direct / Booking.com / etc.)
+ Optional inclusions (Breakfast, Airport transfer...)
+ Date overrides (seasonal pricing)
+ LOS restrictions (minimum stay rules)
```

**Q1:** Does this match what the product team envisions?
- [ ] Yes — confirmed
- [ ] No — needs changes (specify below)

**Product Team Answer:**
> *(Write answer here)*

---

### 2.2 Rate Plan Status

Current design: Rate plans have 3 statuses:
```
DRAFT   → being built, not visible, not bookable
ACTIVE  → live, bookable
ARCHIVED → retired, not bookable, but old bookings reference it
```

**Q1:** Is DRAFT status needed? (Allows owner to build a rate plan without it going live immediately)
- [ ] Yes — DRAFT is important
- [ ] No — a rate plan is either Active or not (simpler)

**Product Team Answer:**
> *(Write answer here)*

---

### 2.3 Derived Rate Plans

Current design:
```
BAR (Best Available Rate) = base
  → Corporate Rate = BAR − 15%
    → VIP Corporate = Corporate − 5%

Change BAR → all derived rates auto-update
```

**Q1:** Is derived/linked rate plans needed in V1?
- [ ] Yes — V1 must have
- [ ] No — Phase 2, V1 each rate plan is independent

**Product Team Answer:**
> *(Write answer here)*

---

### 2.4 Cancellation Policies

Current design: Cancellation policies are reusable — one policy can be applied to many rate plans.

```
Policy: "Flexible"
  → Free cancellation up to 24 hours before check-in
  → After that: 1 night charge

Applied to: BAR, Corporate, Walk-in rates
```

**Q1:** Should cancellation policies be reusable (one policy → many rate plans)?
- [ ] Yes — reusable (current design)
- [ ] No — each rate plan has its own standalone cancellation policy

**Q2:** What cancellation penalty types are needed in V1?

| Penalty Type | V1 / Phase 2 / Not needed |
|---|---|
| No penalty (fully flexible) | |
| Fixed amount (e.g., LKR 5000) | |
| Percentage of booking value (e.g., 50%) | |
| First night charge | |
| Full booking non-refundable | |

**Product Team Answer:**
> *(Write answer here)*

---

### 2.5 Payment Terms Per Rate Plan

```
Rate Plan: "Non-Refundable"
  → Guest must pay full amount at booking

Rate Plan: "Flexible"
  → Guest pays 20% deposit now, rest at hotel
```

**Q1:** What payment term options are needed per rate plan?

| Payment Term | V1 / Phase 2 / Not needed |
|---|---|
| Pay full amount at booking | |
| Pay deposit % at booking, rest at hotel | |
| Pay at hotel (no upfront payment) | |
| Credit card guarantee only (no charge until arrival) | |

**Product Team Answer:**
> *(Write answer here)*

---

## Section 3 — Guest Profile (C5)
> C5 technical design exists. Product team confirms or requests changes.

### 3.1 Guest Profile Scope

Current design:
```
GuestProfile = Business level (NOT operation level)
Same guest profile works across Stays, Dining, Spa under same Business.
```

**Q1:** Is this correct — one guest profile per Business (shared across all operations)?
- [ ] Yes — confirmed
- [ ] No — separate guest profile per operation

**Product Team Answer:**
> *(Write answer here)*

---

### 3.2 Guest Information

**Q1:** What information is captured for a guest in V1?

| Field | Required / Optional / Not needed |
|---|---|
| First name | |
| Last name | |
| Email | |
| Phone | |
| Date of birth | |
| Nationality | |
| NIC number (Sri Lanka locals) | |
| Passport number (foreign guests) | |
| Address | |
| Gender | |
| Language preference | |
| Room preferences (high floor, quiet room...) | |
| Dietary requirements | |
| Special occasions (anniversary, birthday) | |
| VIP flag | |
| Blacklist flag | |
| Notes (internal, staff only) | |

**Product Team Answer:**
> *(Write answer here)*

---

### 3.3 Duplicate Guest Detection

Current design:
```
System detects duplicate when:
  Same email  OR
  Same phone  OR
  Same name + same NIC/Passport
→ Warns staff before creating new profile
```

**Q1:** Is this detection logic correct?
- [ ] Yes — confirmed
- [ ] No — different logic needed (specify):

**Q2:** When duplicate detected — what happens?
- [ ] Staff sees warning, can still create new (override)
- [ ] Staff sees warning, must merge or use existing (cannot create duplicate)
- [ ] System auto-merges

**Product Team Answer:**
> *(Write answer here)*

---

### 3.4 Guest Blacklist

```
Scenario: Guest caused damage in a past stay.
Manager blacklists them.
Next time this guest tries to book — staff gets a warning.
```

**Q1:** Who can blacklist a guest?
- [ ] Only Manager and above
- [ ] Any staff
- [ ] Only Business Owner

**Q2:** What happens when blacklisted guest tries to book?
- [ ] System blocks booking automatically
- [ ] System warns staff but allows booking (staff decides)
- [ ] System warns staff and requires manager approval

**Product Team Answer:**
> *(Write answer here)*

---

## Section 4 — Reservations (C2)
> No technical design yet. Product team defines fully.

### 4.1 Booking Types

Which booking types must V1 support?

| Type | V1 / Phase 2 / Not needed | Description |
|---|---|---|
| Walk-in | | Guest arrives without booking — room assigned immediately |
| Phone booking | | Staff takes booking over phone |
| Online (direct website) | | Guest books via hotel's own booking page |
| OTA (Booking.com, MakeMyTrip) | | Booking comes from third-party site |
| Group booking | | Multiple rooms, one booking reference |
| Corporate booking | | Company account, contract rate |

**Product Team Answer:**
> *(Write answer here)*

---

### 4.2 Booking Lifecycle

What states does a booking go through?

```
Suggested flow:
Created → Confirmed → Checked-In → Checked-Out
                   ↘ Cancelled
                   ↘ No-Show
```

**Q1:** Is this flow correct?
- [ ] Yes — confirmed
- [ ] No — different states needed (specify):

**Q2:** Can a Confirmed booking be modified (date change, room change)?
- [ ] Yes — staff can modify
- [ ] Yes — but only before check-in
- [ ] No — must cancel and rebook

**Q3:** Can a Checked-In booking be modified?
- [ ] Yes — extend stay, change room
- [ ] No — no changes after check-in

**Product Team Answer:**
> *(Write answer here)*

---

### 4.3 Availability Logic

How is room availability calculated?

```
Current design:
Available = Total rooms − (Confirmed bookings + OOO rooms + Manual blocks)
```

**Q1:** Is this logic correct?
- [ ] Yes — confirmed
- [ ] No — needs adjustment (specify):

**Q2:** At what point does a booking "block" availability?
- [ ] Only when Confirmed (Created status doesn't block)
- [ ] From the moment it's Created (holds the room)
- [ ] Configurable — owner decides

**Q3:** How far in advance can bookings be made?
- [ ] No limit
- [ ] Maximum X months (specify: _____)
- [ ] Configurable by owner

**Product Team Answer:**
> *(Write answer here)*

---

### 4.4 Booking Information

What information is captured when a booking is created?

| Field | Required / Optional |
|---|---|
| Guest name | |
| Guest phone | |
| Guest email | |
| Room type (requested) | |
| Specific room (assigned) | |
| Check-in date | |
| Check-out date | |
| Number of adults | |
| Number of children | |
| Number of infants | |
| Rate plan selected | |
| Special requests | |
| Booking source (walk-in / phone / online / OTA) | |
| OTA confirmation number | |
| Internal notes | |
| Payment method | |
| Deposit amount collected | |

**Product Team Answer:**
> *(Write answer here)*

---

### 4.5 No-Show Handling

```
Scenario: Guest had a booking for today. Did not arrive. No cancellation.
```

**Q1:** When is a booking marked as No-Show?
- [ ] Staff manually marks it
- [ ] Auto-marked at a set time (e.g., midnight of arrival date)
- [ ] Auto-marked but staff confirms

**Q2:** What happens to the room when No-Show is marked?
- [ ] Room is released immediately, available for new booking
- [ ] Room is held until staff confirms release

**Q3:** Is there a No-Show charge?
- [ ] Yes — charge according to rate plan's no-show policy
- [ ] No — no charge for no-show in V1

**Product Team Answer:**
> *(Write answer here)*

---

### 4.6 Overbooking

```
Scenario: Hotel has 10 rooms. 11 bookings accepted.
Intentional (hotel strategy) or accidental?
```

**Q1:** Does OneNex allow intentional overbooking?
- [ ] Yes — owner sets overbooking % per room type
- [ ] No — system never allows booking beyond available rooms

**Q2:** If overbooking happens by mistake — what does the system do?
- [ ] Block it — cannot confirm booking if no rooms available
- [ ] Warn staff — shows alert, staff can override
- [ ] Allow it silently

**Product Team Answer:**
> *(Write answer here)*

---

## Section 5 — Front Desk (C3)
> No technical design yet. Product team defines fully.

### 5.1 Check-In Flow

```
Guest arrives at front desk:
```

**Q1:** What does the check-in screen need to show?

| Information | Show at check-in? |
|---|---|
| Guest name and photo | Yes / No |
| Booking details (dates, room type, rate) | Yes / No |
| Past stay history | Yes / No |
| Special requests from booking | Yes / No |
| Preferences from guest profile | Yes / No |
| VIP flag / Blacklist warning | Yes / No |
| ID verification prompt | Yes / No |
| Room assignment (which room) | Yes / No |
| Deposit paid / balance due | Yes / No |
| Key card issue confirmation | Yes / No |

**Q2:** Can a guest check in before the official check-in time?
- [ ] Yes — if a clean/inspected room is available
- [ ] No — strictly enforced check-in time
- [ ] Yes — with an early check-in charge (configurable)

**Q3:** Can a guest check in for multiple rooms at once (group)?
- [ ] Yes — V1 supports group check-in
- [ ] No — Phase 2, V1 is one room at a time

**Product Team Answer:**
> *(Write answer here)*

---

### 5.2 Check-Out Flow

```
Guest comes to front desk to check out:
```

**Q1:** What does the check-out process involve?

| Step | V1 / Phase 2 / Not needed |
|---|---|
| Show full folio (all charges) | |
| Guest reviews charges | |
| Payment collection | |
| GST / Tax invoice generation + print/email | |
| Room key return confirmation | |
| Room status change to Dirty after checkout | |
| Thank you / feedback prompt | |

**Q2:** Can a guest check out late (after official check-out time)?
- [ ] Yes — late check-out available, configurable charge
- [ ] No — strict check-out time enforced
- [ ] Yes — free late check-out if room not needed

**Q3:** Can a guest do express/self checkout?
- [ ] Yes — V1 supports (staff or guest-initiated)
- [ ] Phase 2

**Product Team Answer:**
> *(Write answer here)*

---

### 5.3 Room Change

```
Scenario: Guest is in Room 101, wants to move to Room 205.
Reasons: noisy, AC broken, upgrade requested.
```

**Q1:** What happens when a room change is done?
- [ ] Folio transfers automatically to new room
- [ ] Staff manually transfers charges
- [ ] Both folio and booking update automatically

**Q2:** Who can perform a room change?
- [ ] Any front desk staff
- [ ] Supervisor / Manager only
- [ ] Configurable by owner

**Product Team Answer:**
> *(Write answer here)*

---

### 5.4 Walk-In Check-In

```
Guest walks in with no booking, wants a room for tonight.
```

**Q1:** Walk-in flow:
- [ ] Staff checks availability → selects room → creates booking + immediately checks in (one flow)
- [ ] Staff must create booking first → then check in separately
- [ ] Walk-in = same as booking creation, just instant confirmation

**Product Team Answer:**
> *(Write answer here)*

---

## Section 6 — Housekeeping (C4)
> No technical design yet. Product team defines fully.

### 6.1 Housekeeping Tasks

**Q1:** Who assigns housekeeping tasks in V1?
- [ ] Supervisor manually assigns room to housekeeper
- [ ] System auto-assigns based on floor/section
- [ ] Both options available

**Q2:** What triggers a housekeeping task?
| Trigger | V1 / Phase 2 |
|---|---|
| Guest checks out (room needs cleaning) | |
| Guest requests cleaning (DND off) | |
| Supervisor manually creates task | |
| Daily schedule (clean all occupied rooms) | |

**Product Team Answer:**
> *(Write answer here)*

---

### 6.2 Inspection

Current design: Housekeeper cleans → Supervisor inspects → Room becomes INSPECTED → Only then assignable.

**Q1:** Is formal inspection required in V1?
- [ ] Yes — housekeeper marks clean, supervisor inspects and approves
- [ ] No — housekeeper marks clean = room is ready (no inspection step)
- [ ] Configurable — owner decides if inspection required

**Product Team Answer:**
> *(Write answer here)*

---

### 6.3 Housekeeping View

**Q1:** What does the housekeeping screen show?

| Info | Show? |
|---|---|
| All rooms with current status | Yes / No |
| Which rooms are assigned to which housekeeper | Yes / No |
| Priority rooms (VIP arriving, early check-in requested) | Yes / No |
| Rooms that are OOO (maintenance) | Yes / No |
| Estimated time since guest checked out | Yes / No |

**Product Team Answer:**
> *(Write answer here)*

---

## Section 7 — Folio & Billing (C6)
> No technical design yet. Product team defines fully.

### 7.1 What Is a Folio?

```
A Folio = running tab for a guest's stay.
Auto-created at check-in. Closed at checkout.
All charges post here.
```

**Q1:** Is this the correct understanding?
- [ ] Yes — confirmed
- [ ] No — needs adjustment:

**Q2:** What charges can be added to a folio?

| Charge Type | V1 / Phase 2 |
|---|---|
| Room charge (nightly rate) | |
| Restaurant/Dining charge (room charge from dining) | |
| Bar charge | |
| Spa/Wellness charge | |
| Mini bar charge | |
| Laundry charge | |
| Transport / taxi charge | |
| Manual charge (staff adds anything) | |
| Damage charge | |
| Discount / complimentary | |

**Product Team Answer:**
> *(Write answer here)*

---

### 7.2 Room Charge from Other Operations

```
Guest has lunch at hotel restaurant.
Tells waiter: "Charge to room 205."
→ Amount appears on folio automatically.
```

**Q1:** Is this automatic room charge required in V1?
- [ ] Yes — must work seamlessly in V1
- [ ] No — Phase 2, V1 = staff manually adds charge to folio

**Product Team Answer:**
> *(Write answer here)*

---

### 7.3 Folio at Checkout

**Q1:** What happens at checkout?

| Step | V1 / Phase 2 / Not needed |
|---|---|
| Show full itemized folio to guest | |
| Accept payment (cash / card / online) | |
| Apply discount or complimentary | |
| Generate GST invoice (PDF) | |
| Email invoice to guest | |
| Print invoice | |

**Q2:** Can a guest pay part now, part later?
- [ ] Yes — partial payment allowed
- [ ] No — full payment required at checkout

**Q3:** Can checkout be done with an outstanding balance (credit / account)?
- [ ] Yes — for corporate accounts (bill sent later)
- [ ] No — must fully settle before checkout

**Product Team Answer:**
> *(Write answer here)*

---

### 7.4 Folio Split

```
Scenario: 2 guests sharing a room.
Guest A pays for room charges only.
Guest B pays for all food/beverage charges.
```

**Q1:** Is folio splitting required in V1?
- [ ] Yes — V1 must support
- [ ] No — Phase 2, V1 = one folio per stay

**Product Team Answer:**
> *(Write answer here)*

---

## Section 8 — Night Audit (C8)
> No technical design yet. Product team defines fully.

### 8.1 What Is Night Audit?

```
Night Audit = automated job that runs every night (usually midnight or 1 AM):
1. Posts room charge to every open folio (for tonight's stay)
2. Rolls the business date to next day
3. Generates audit report
```

**Q1:** Is this understanding correct?
- [ ] Yes — confirmed
- [ ] No — needs adjustment:

**Q2:** When does night audit run?
- [ ] Exactly midnight (auto)
- [ ] Staff triggers it manually at end of day
- [ ] Configurable time (e.g., 1 AM)

**Q3:** Can night audit be run if there are unresolved issues (e.g., unpaid folios)?
- [ ] Yes — runs regardless, shows warnings
- [ ] No — blocks until all issues resolved
- [ ] Warns staff, staff can force-run

**Q4:** What does the night audit report include?

| Item | V1 / Phase 2 |
|---|---|
| Rooms occupied tonight | |
| Room revenue posted | |
| Arrivals today | |
| Departures today | |
| No-shows today | |
| Outstanding folios (unpaid balances) | |
| Cash / card payments collected | |
| Discounts / complimentary given | |

**Product Team Answer:**
> *(Write answer here)*

---

## Section 9 — Basic Reports (C9)

### 9.1 What Reports Are Needed in V1?

| Report | V1 / Phase 2 / Not needed |
|---|---|
| Occupancy % (tonight, this month) | |
| ADR (Average Daily Rate) | |
| RevPAR (Revenue per Available Room) | |
| Arrivals list (today) | |
| Departures list (today) | |
| In-house guests list | |
| Room revenue summary | |
| Payment method breakdown (cash vs card vs online) | |
| Cancellation summary | |
| No-show summary | |

**Q1:** Who can access reports?
- [ ] Owner only
- [ ] Owner + Manager
- [ ] All staff (view only)
- [ ] Permission-based (configurable)

**Product Team Answer:**
> *(Write answer here)*

---

## Section 10 — Add-ons Scope Confirmation (A1–A12)

Confirm which add-ons are in V1 and which are Phase 2.

| Add-on | V1 / Phase 2 / Phase 3 | Priority if V1 |
|---|---|---|
| A1 — Channel Manager (OTA sync — Booking.com, MakeMyTrip) | | |
| A2 — Online Booking Engine (direct website booking) | | |
| A3 — Revenue Management (yield, auto pricing) | | |
| A4 — Group & Corporate Bookings | | |
| A5 — Housekeeping Advanced (smart assign, mobile app) | | |
| A6 — Guest Portal (guest-facing app) | | |
| A7 — Maintenance Management | | |
| A8 — Meeting & Banquet | | |
| A9 — Advanced Folio (split, master, transfer) | | |
| A10 — Loyalty & Offers | | |
| A11 — OTA Parity Monitor | | |
| A12 — Advanced Analytics (pickup, pace, forecast) | | |

**Product Team Answer:**
> *(Fill each row)*

---

## Section 11 — Out of Scope (Confirm NOT in V1)

| Feature | Confirm out of V1 |
|---|---|
| Mobile key (guest unlocks room from phone) | Yes out / No bring in |
| Biometric check-in | Yes out / No bring in |
| Self check-in kiosk | Yes out / No bring in |
| GDS integration (Sabre, Amadeus) | Yes out / No bring in |
| Revenue management AI/dynamic pricing | Yes out / No bring in |
| Multi-property consolidated dashboard | Yes out / No bring in |
| Loyalty points program | Yes out / No bring in |
| Pre-arrival automated email sequence | Yes out / No bring in |
| Guest sentiment analysis | Yes out / No bring in |

**Product Team Answer:**
> *(Confirm each row)*

---

## Section 12 — Special Scenarios (Must Answer)

**Scenario 1:**
> Guest books Room 101 (Deluxe) online. Arrives and asks for an upgrade to a Suite.
> Front desk upgrades them. Rate plan changes.
> **What happens to the rate?** Does the folio update automatically? Who approves the upgrade?

**Product Team Answer:**
> *(Write answer here)*

---

**Scenario 2:**
> Guest checks in for 5 nights. After night 2, they want to extend by 2 more nights.
> Room 101 is available. Rate plan they booked has expired for those extra dates.
> **What rate applies for the extension nights?**

**Product Team Answer:**
> *(Write answer here)*

---

**Scenario 3:**
> Two housekeepers are assigned to the same floor.
> Both mark Room 205 as clean at the same time.
> **What happens?** Which status wins?

**Product Team Answer:**
> *(Write answer here)*

---

**Scenario 4:**
> Guest checks in. Goes to restaurant. Charges LKR 3,500 to room.
> Guest then disputes this charge at checkout — says they didn't eat there.
> **How does staff handle this?** Can a charge be voided from folio? Who can void it?

**Product Team Answer:**
> *(Write answer here)*

---

**Scenario 5:**
> Night audit runs at midnight. One folio has a zero balance (complimentary stay).
> **Does room charge still post to it?** Or is it skipped?

**Product Team Answer:**
> *(Write answer here)*

---

**Scenario 6:**
> Hotel has 20 rooms. All occupied tonight. Walk-in guest arrives.
> **What does the system show?** Is there any way to accommodate them?

**Product Team Answer:**
> *(Write answer here)*

---

**Scenario 7:**
> Staff creates a booking by mistake for wrong dates.
> Realizes error 10 minutes later.
> **Can they cancel/delete it without any record?** Or is there always an audit trail?

**Product Team Answer:**
> *(Write answer here)*

---

**Scenario 8:**
> Guest folio has charges from Dining and Room.
> At checkout, guest pays with 2 methods — LKR 10,000 cash + rest by card.
> **How is this handled?** Can one folio accept multiple payment methods?

**Product Team Answer:**
> *(Write answer here)*

---

## Section 13 — Sign-off

Once all sections are answered, product team signs off here.

| Section | Status | Signed by | Date |
|---|---|---|---|
| Section 0 — What Is Stays | PENDING / CONFIRMED | | |
| Section 1 — Room Setup (C1) | PENDING / CONFIRMED | | |
| Section 2 — Rate Plans (C7) | PENDING / CONFIRMED | | |
| Section 3 — Guest Profile (C5) | PENDING / CONFIRMED | | |
| Section 4 — Reservations (C2) | PENDING / CONFIRMED | | |
| Section 5 — Front Desk (C3) | PENDING / CONFIRMED | | |
| Section 6 — Housekeeping (C4) | PENDING / CONFIRMED | | |
| Section 7 — Folio & Billing (C6) | PENDING / CONFIRMED | | |
| Section 8 — Night Audit (C8) | PENDING / CONFIRMED | | |
| Section 9 — Basic Reports (C9) | PENDING / CONFIRMED | | |
| Section 10 — Add-ons Scope | PENDING / CONFIRMED | | |
| Section 11 — Out of Scope | PENDING / CONFIRMED | | |
| Section 12 — Special Scenarios | PENDING / CONFIRMED | | |

**All CONFIRMED → Backend team starts building. Zero changes after this.**
