# Room Booking — 03: Room Availability Logic
> Hotel Module → Room Booking → Topic 3 of 7

---

## Overview

Guest "Dec 15-18, Double Room" search பண்றான் — system எப்படி available rooms calculate பண்றது என்று இது cover பண்றது. இது hotel-oda brain — wrong calculation = overbook or revenue loss.

---

## Core Formula

```
Available Rooms = Total Rooms of that type − Blocked Rooms

Blocked by:
  ├── Confirmed bookings (dates overlap)
  ├── Checked-in guests (currently in room)
  ├── Out-of-order rooms (maintenance)
  ├── Group blocks (held but not yet assigned)
  └── Manual blocks (staff manually held)
```

---

## How Availability Check Works

**Guest searches:** Double Room, Dec 15 Check-in → Dec 18 Check-out (3 nights)

**System logic — Check every Double room:**

```
Room 101 (Double):
  Dec 15 night → Any booking? YES (Guest A) → BLOCKED

Room 102 (Double):
  Dec 15 night → Any booking? NO
  Dec 16 night → Any booking? NO
  Dec 17 night → Any booking? NO
  All 3 nights free → AVAILABLE ✓

Room 103 (Double):
  Dec 15 night → Free
  Dec 16 night → Booking exists (Guest B) → BLOCKED
  Even one night blocked → Whole stay = BLOCKED ✗

Room 104 (Double):
  Dec 15 night → Out-of-Order (maintenance) → BLOCKED ✗
```

**Rule: ALL nights in the requested stay must be free. Even one night blocked = room unavailable.**

---

## Check-in / Check-out Date Logic

This is a common confusion — clarifying clearly:

```
Guest booking: Dec 15 (Check-in) → Dec 18 (Check-out)

Nights stayed:
  Night 1: Dec 15 (sleep here)
  Night 2: Dec 16 (sleep here)
  Night 3: Dec 17 (sleep here)
  Dec 18: Checkout morning — NOT a stay night

Total nights: 3
```

**Same-day back-to-back bookings are possible:**

```
Guest A: Dec 12 → Dec 15 checkout (leaves morning)
Guest B: Dec 15 → Dec 18 checkout (arrives afternoon)

Room 102 on Dec 15:
  Morning: Guest A checks out (room goes DIRTY)
  Afternoon: Housekeeping cleans (room goes CLEAN)
  Evening: Guest B checks in ✓

System logic: Dec 15 is A's checkout date, NOT a stay night
              Dec 15 is B's check-in — same room allowed ✓
```

---

## What Blocks Availability

### 1. Confirmed Bookings (Most Common Block)

```
Any booking with status IN [CONFIRMED, CHECKED_IN]
that overlaps the requested date range → Room blocked

Overlap check:
  Existing booking: Dec 13 → Dec 16
  New request:      Dec 15 → Dec 18
  
  Overlap? Dec 15, Dec 16 overlap → BLOCKED ✗

No overlap:
  Existing booking: Dec 12 → Dec 15 (checks out Dec 15)
  New request:      Dec 15 → Dec 18 (checks in Dec 15)
  
  Overlap? None (checkout date ≠ stay night) → AVAILABLE ✓
```

---

### 2. Out-of-Order Rooms

Staff marks a room out-of-order with a date range:

```
Room 105: Out-of-Order Dec 14 → Dec 17 (pipe leak repair)

Guest searches Dec 15-18:
  Dec 15, Dec 16, Dec 17 → Room 105 is OOO → BLOCKED ✗
  Dec 18 → OOO ended → Available (but Dec 15-17 blocked the search)

Staff inputs:
  Room number, reason, start date, end date, reported by
  
System:
  Auto-removes OOO rooms from availability for that period
  Housekeeping / maintenance notified
  Room returns to inventory automatically after end date
```

---

### 3. Group Blocks

Travel agent holds bulk rooms without specific assignment yet:

```
Agent: "Hold 10 Double rooms for Dec 15-18"
        ↓
Staff creates Group Block:
  Type: Double, Count: 10, Dates: Dec 15-18
  Status: BLOCKED (unassigned)
        ↓
System: Reduce Double room inventory by 10 for those dates

Before block: 30 Double rooms available
After block:  20 Double rooms available to general public

When rooming list comes → Specific rooms assigned from the 10 blocked
Unreleased rooms after cut-off → Released back to general inventory
```

---

### 4. Manual Block (Staff Hold)

Staff manually holds a specific room:

```
Use cases:
  ├── VIP guest preference — "Room 201 always for Mr. Kumar"
  ├── Owner's personal room — never sell
  ├── Pending renovation — block in advance
  └── Media / photography shoot

Input:
  Room, Start date, End date, Reason, Block type

System: Room removed from availability for that period
Staff can view / remove manual blocks anytime
```

---

## Room Type vs Specific Room — Two Levels

**Very important concept:**

```
LEVEL 1 — Room Type Availability (At Booking Time)
  Question: "How many Double rooms available Dec 15-18?"
  Answer: "5 Double rooms available"
  
  → Guest books "a Double Room" (not Room 102 specifically)
  → Booking is for room TYPE, not specific room

LEVEL 2 — Specific Room Assignment (At Check-in Time)
  Guest arrives Dec 15
  Staff sees: Rooms 102, 107, 109, 112, 115 available (all Double, all clean)
  Staff assigns: Room 107 (guest prefers high floor)
  
  → Now Room 107 specifically linked to this booking
```

**Why this design?**

```
Scenario: Guest booked "Double Room" on Dec 1 for Dec 15

Dec 14 evening: Room 102 still occupied (Guest leaving Dec 15)
Dec 15 morning: Room 102 checked out → goes Dirty
Dec 15 noon:    Housekeeping cleans Room 102 → Clean
Dec 15 2pm:     New guest arrives → Staff assigns Room 102 ✓

If we assigned Room 102 at booking time (Dec 1):
  → What if Room 102 had a maintenance issue Dec 14?
  → Staff would need to manually reassign
  → Unnecessary constraint
  
Flexible assignment at check-in = better housekeeping planning
                                  upgrade opportunity
                                  preference matching possible
```

---

## Availability Calendar (Staff View)

Visual representation — staff sees this:

```
Room    | Dec 13 | Dec 14 | Dec 15 | Dec 16 | Dec 17 | Dec 18 |
--------|--------|--------|--------|--------|--------|--------|
101 Dbl | [Guest A────────]|        |        |        |        |
102 Dbl |        |        | [Guest B─────────────────]|        |
103 Dbl |        | [OOO Out-of-Order──────] |        |        |
104 Dbl |        |        |        | [Group─────────] |        |
105 Dbl |        |        |        |        |        |        | ← Free

Color Codes:
  Blue   = Confirmed booking / Checked-in
  Red    = Out-of-Order
  Yellow = Group block (unassigned)
  Orange = Manual block
  Green  = Available
```

---

## Availability Count Calculation

```
Hotel total: 30 Double rooms

On Dec 15:
  Confirmed bookings occupying Double rooms: 12
  Checked-in Double room guests:              8
  Out-of-order Double rooms:                  2
  Group block (unassigned):                   3
  Manual blocks:                              1
  ─────────────────────────────────────────────
  Total blocked:                             26
  
  Available for new bookings: 30 - 26 = 4 Double rooms
```

---

## Real-Time Sync — Critical for OTA

```
Our system shows: 4 Double rooms available
Booking.com shows: 4 Double rooms (synced)
Expedia shows: 4 Double rooms (synced)

Guest A books via our website → 3 remaining
  → Immediately sync to Booking.com → 3
  → Immediately sync to Expedia → 3

Guest B books via Booking.com → 2 remaining
  → Booking.com pushes to our system → 2
  → Our system pushes to Expedia → 2

This must happen in SECONDS — not minutes.
Delay = risk of same room sold twice.
```

---

## Race Condition Problem & Solution

**Problem: Two guests book the last room at the same moment**

```
1 Double room remaining

Guest A (website)    → starts checkout → processing payment
Guest B (Booking.com)→ booking arrives at same time
Staff C (phone call) → manually creating booking

All 3 hit the same last room → All 3 confirm → OVERBOOKED 😱
```

**Solution: Inventory Locking**

```
SOFT LOCK (Temporary Hold):
  Guest A enters checkout page
  → System: Soft lock on that room (5-10 minute hold)
  → Guest B arrives: "Room not available" shown
  → Guest A completes payment: HARD LOCK (permanent booking)
  → Soft lock released (only that specific hold gone)

  If Guest A abandons (closes page):
  → Soft lock expires after 5-10 min → Room available again

HARD LOCK (Confirmed Booking):
  Payment complete → Booking confirmed
  → Permanent block until check-out or cancellation
```

---

## Overbooking Buffer (Advanced — Layer 3+)

Some hotels intentionally keep a small buffer:

```
Hotel setting: "Never show 0 available — always keep 1 room as buffer"

Even if 29 of 30 rooms are sold:
  Show available: 1 (buffer room held back from OTA)
  
Why: OTA sync has tiny delay — buffer prevents accidental overbook
     Also: Walk-in guest can still get the buffer room

This is separate from intentional overbooking (covered in Topic 7)
```

---

## Availability Check — Summary Flow

```
Guest requests: Room Type X, Date A → Date B
                      ↓
System gets: All rooms of Type X (total count)
                      ↓
For each room, check all nights (Date A to Date B-1):
  
  Blocked if:
  ├── CONFIRMED or CHECKED_IN booking overlaps → exclude
  ├── OUT_OF_ORDER overlaps → exclude
  ├── GROUP_BLOCK overlaps → exclude
  └── MANUAL_BLOCK overlaps → exclude
                      ↓
Remaining rooms = Available
                      ↓
Available count > 0 → Show room + price options
Available count = 0 → "Sold out for these dates"
                      → Suggest nearby dates (if enabled)
```
