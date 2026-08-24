# Room Booking — 07: Overbooking
> Hotel Module → Room Booking → Topic 7 of 7

---

## Overview

Hotel-la இருக்கற physical room count-விட அதிகமான bookings accept பண்றது overbooking. இதுல இரண்டு types இருக்கு: **Intentional** (strategy) and **Accidental** (mistake). Both-ஐயும் எப்படி handle பண்றது என்று இது cover பண்றது.

---

## Part 1: Intentional Overbooking

### Why Hotels Overbook on Purpose

```
Hotel has: 100 rooms
Historical data shows:
  Average no-show rate:       3%
  Average cancellation rate:  4%
  Combined:                   7% rooms typically go empty

If hotel sells exactly 100 rooms:
  → 7 guests won't show up or cancel last minute
  → 7 rooms empty on that night
  → 7 nights × ₹5,000 = ₹35,000 revenue lost

Solution: Sell 107 rooms (overbook by 7%)
  → 7 cancel/no-show → 100 guests arrive → Full house ✓
  → Revenue maximized
  
Risk: If all 107 show up → 7 guests need to be relocated
```

---

### Overbooking Levels

```
Conservative:   Overbook 2-3%   → Safe, small gain
Standard:       Overbook 5-8%   → Industry common
Aggressive:     Overbook 10-15% → High revenue, high relocation risk

What to base it on:
  ├── Historical no-show rate (by season, day of week)
  ├── Historical cancellation rate (by source, rate plan)
  ├── Booking lead time (last-minute bookings = more no-shows)
  ├── Source mix (OTA = more cancellations than direct)
  └── Season (peak = lower overbook %, off-peak = higher)
```

---

### Overbooking in Our System — Layer Design

```
Layer 1 (Simple):   Overbooking DISABLED — safe default
                    System never sells beyond physical room count

Layer 2 (Standard): Overbooking DISABLED — default
                    Can be enabled manually by admin

Layer 3 (Advanced): Overbooking CONFIGURABLE per room type
                    Staff sets overbook % manually

Layer 4 (Luxury):   AI-driven overbooking optimization
                    System auto-calculates optimal % per room type per date
                    Based on historical patterns + current booking pace
```

---

### Configuring Overbooking (Layer 3+)

```
Hotel admin sets per room type:

  Double Room:  Allow 105%  (sell up to 105 of 100 rooms)
  Single Room:  Allow 110%  (high no-show history for singles)
  Suite:        Allow 100%  (never overbook premium rooms — bad guest experience)
  Family Room:  Allow 103%  (low no-show, conservative)

System behavior:
  Double rooms: 100 physical
  Max sellable: 105
  
  When 100 sold → Still shows 5 available (overbook buffer)
  When 105 sold → Shows sold out
  When 101-105 sold → Staff warning: "Overbooking active"
```

---

### Walk Procedure — When All Guests Arrive

Worst case: 105 rooms sold, all 105 guests show up.

**Who gets "walked" (relocated to another hotel)?**

```
Priority — walked in this order (last priority = first walked):
  1. Single-night stays (over multi-night)
  2. Same-day / last-minute bookings
  3. OTA bookings (over direct bookings)
  4. Non-loyalty members (over loyalty members)
  5. Low-tier loyalty (over high-tier)
  
NEVER walked (protected):
  ├── VIP guests
  ├── Top-tier loyalty members
  ├── Direct bookers (rewarding loyalty to hotel)
  ├── Long-stay guests (already in hotel, not new arrival)
  └── Guests with special needs / medical requirements
```

**Walk process:**
```
Identified guest to be walked
        ↓
Apologize sincerely — explain situation honestly
        ↓
Arrange alternative:
  ├── Comparable or better hotel nearby (same or higher star rating)
  ├── Hotel pays transport to partner hotel
  ├── Hotel pays room rate difference (if partner costs more)
  └── Complimentary meal at partner hotel (goodwill)
        ↓
Compensation for the walked guest:
  ├── Full refund for the walked night(s)
  ├── Complimentary upgrade on next visit
  ├── Loyalty points bonus
  └── Handwritten apology from manager
        ↓
Document in system:
  ├── Booking marked: WALKED
  ├── Partner hotel where relocated
  ├── Compensation given
  └── Follow-up scheduled
```

---

## Part 2: Accidental Overbooking

This is a mistake — must be prevented by system design.

### Race Condition — Same Room Sold Twice

```
Scenario: 1 Double room remaining

Moment 0.00s: Guest A (website) enters checkout page
Moment 0.01s: Guest B (Booking.com API) books same room
Moment 0.02s: Staff C (phone call) creates booking for same room

Without protection:
  All 3 systems see "1 room available" simultaneously
  All 3 confirm → Same last room sold 3 times → Accidentally overbooked 😱
```

**Solution: Inventory Locking System**

```
SOFT LOCK (Temporary Hold):
  Guest A opens checkout page
  → System immediately: SOFT LOCK that room type inventory (5-10 minutes)
  → Guest B arrives: "Sorry, no Double rooms available" ← Blocked
  → Staff C: Same block ← Blocked
  
  Guest A completes payment (within 5 min):
  → HARD LOCK activated (permanent confirmed booking)
  → Soft lock removed
  
  Guest A abandons checkout (closes tab, times out):
  → Soft lock EXPIRES after 5-10 minutes
  → Room available again for others
  
  Note: Soft lock is on ROOM TYPE count, not specific room
  (Since specific room assignment happens at check-in)

HARD LOCK (Permanent — Confirmed Booking):
  Payment complete → Booking entry created in DB
  → Room type count permanently decremented
  → Released only on cancellation or check-out
```

---

### OTA Sync Failure — Availability Desync

```
Scenario:
  Our system: 1 Double room available
  Booking.com: 1 Double room (synced correctly)
  
  Network issue → Sync fails for 15 minutes
  
  During downtime:
    Guest A books via our website → Confirmed → 0 rooms (our system updated)
    Guest B books via Booking.com → Confirmed (stale data — still shows 1) → -1 rooms
  
  Sync resumes: CONFLICT — both bookings confirmed, only 1 room
  → Accidental overbook from sync failure
```

**Prevention strategies:**

```
1. Real-time push sync:
   Our system → Immediately push to all OTAs on every booking
   Not batch (every 5 min) → Real-time (within seconds)

2. Sync failure detection:
   If sync to OTA fails → Immediate alert to staff
   Staff can manually close OTA availability during outage

3. Safe mode (automatic):
   If sync failure detected → Auto-close availability on all OTAs
   "If we can't keep OTAs updated, don't sell there" — safety first
   Resume when sync restored

4. Buffer room strategy:
   Never push 0 availability to OTAs
   Keep 1 room as buffer — only sold via direct/phone
   Example: 5 rooms available → Push 4 to OTA, keep 1 as buffer
   → Even if sync is 1 minute late, buffer absorbs the risk
```

---

### Double-Booking Detection

System checks on every new booking:

```
Before confirming any booking:
  
  SELECT COUNT(*) FROM bookings
  WHERE room_type = 'DOUBLE'
  AND status IN ('CONFIRMED', 'CHECKED_IN')
  AND check_in_date < new_checkout_date
  AND check_out_date > new_checkin_date
  
  COUNT ≥ physical_rooms + overbook_buffer
  → REJECT: "No rooms available"
  
  COUNT < physical_rooms + overbook_buffer
  → PROCEED: Create booking
  
  This check runs ATOMICALLY (database transaction)
  → No two requests can pass this check simultaneously for the same last room
```

---

## Overbooking Reports — What Management Tracks

```
Reports needed:
  ├── Current overbooking status (how many over per room type, per date)
  ├── Walk history: How often? Which guests? Which dates?
  ├── Walk cost: Transport + rate difference + compensation totals
  ├── Overbook revenue gain: Extra rooms sold × rate
  ├── Net gain: Revenue gain − Walk cost (is it worth it?)
  ├── No-show rate by source (calibrate overbook % accurately)
  ├── Cancellation patterns (refine strategy per season)
  └── OTA sync failure log
```

---

## Overbooking — Layer Summary

| Layer | Intentional Overbook | Accidental Prevention |
|-------|---------------------|----------------------|
| Layer 1 | Disabled | Basic locking |
| Layer 2 | Disabled (can enable) | Locking + OTA sync |
| Layer 3 | Configurable % per room type | Full locking + sync + buffer |
| Layer 4 | AI-optimized per date | All above + predictive conflict detection |

---

## Summary

```
INTENTIONAL OVERBOOKING:
  Hotel sells more rooms than physically available
  Based on historical no-show + cancellation data
  System enforces overbook % limit per room type
  If all guests arrive → Walk procedure (relocate + compensate)
  Goal: Maximize occupancy revenue

ACCIDENTAL OVERBOOKING:
  Mistake — same room sold to multiple guests simultaneously
  
  Prevention 1 — Race condition:
    Soft lock (temporary) when guest enters checkout
    Hard lock (permanent) when payment confirmed
    
  Prevention 2 — OTA sync failure:
    Real-time sync (not batch)
    Sync failure → Alert staff → Auto-close OTA availability
    Buffer room — last room never sold on OTA
    
  Prevention 3 — Database level:
    Atomic transaction check before every booking creation
    Prevents any two simultaneous bookings for same last room
```
