# Group & Corporate Management — Overview & Smart Features
> Hotel Module → Group & Corporate
> Status: V1 required (simplified) | Advanced features Phase 2+

---

## Existing Systems — Problems

| System | Weakness |
|--------|---------|
| Oracle OPERA | Good group management but very complex. Requires dedicated group coordinator. |
| Mews | Basic group bookings, limited corporate features, no travel manager portal |
| Cloudbeds | Very basic group support, no corporate account management |

**Core problem:** Groups and corporate managed partly in PMS, partly outside (emails, spreadsheets, phone calls). Nothing unified.

---

## Group Booking Threshold
5+ rooms booked together = Group (configurable per hotel)

**Who books groups:** Tour operators, wedding parties, corporate off-sites, sports teams, conference attendees, school trips

---

## PART 1 — GROUP BOOKINGS

---

## 1. Group Block Creation

```
STEP 1 — Request received:
  Group: "Reliance Offsite 2026"
  Dates: Dec 15-18 | Rooms: 25 Double + 5 Suites
  Organizer: Ravi Shah | Special: Conference room, team dinner

STEP 2 — Staff creates block:
  Group ID: GRP-2026-089
  Status: TENTATIVE (hold without contract)
  Hold until: Dec 1 (cut-off)

STEP 3 — Rate proposal:
  Double: ₹3,500/night (vs rack ₹4,000 — 12% off)
  Suite:  ₹10,000/night (vs rack ₹12,000 — 17% off)
  Inclusions: Breakfast + Conference room 1 day
  Cancellation: Separate group policy

STEP 4 — Agent accepts:
  Status: CONFIRMED
  Contract auto-generated (PDF) → Emailed to agent
  Deposit schedule activated
```

---

## 2. Rooming List Management

### V1 Approach (Basic)
Excel upload → System creates guest profiles in bulk. Staff reviews and approves.

### Phase 2 Approach (Smart)
```
Agent uploads Excel or fills online portal form:
  Row 1: Rajesh Kumar, Double, Dec 15-18, No pork
  Row 2: Priya Menon, Double, Dec 15-18, Vegetarian
  Row 3: Mr. & Mrs. Sharma, Suite, Dec 15-18, Anniversary ← auto-flag

System:
  ├── Creates guest profiles (all guests)
  ├── Links each to group block
  ├── Flags special requests automatically
  ├── Alerts housekeeping: Anniversary room decoration needed
  └── Reports: "28 of 30 rooms assigned. 2 TBC."

No manual entry by staff.
```

---

## 3. Cut-off Date Management

### V1 Approach (Basic)
Staff manually monitors cut-off dates. Alert in system.

### Phase 2 Approach (Auto)
```
Cut-off date arrives (Dec 1):
  Block: 30 rooms | Confirmed by agent: 25 | Unconfirmed: 5
  → Auto-release 5 rooms to general inventory
  → Alert to agent + sales manager

Before cut-off (6 days prior):
  → Alert to sales: "GRP-2026-089 cut-off in 6 days. 5 rooms unconfirmed."
  → Follow up prompt with agent contact

No manual tracking. No revenue lost from forgotten releases.
```

---

## 4. Group Billing — Flexible Options

```
Option A — Full Master Account:
  All charges → One invoice to company

Option B — Room to Master, Personal to Individual:
  Room + Breakfast → Company pays
  Minibar, Spa, Bar → Each guest pays personally

Option C — Split by Department:
  Engineering team (15 rooms) → Engineering cost center
  Sales team (15 rooms) → Sales cost center

Option D — Hybrid:
  Company deposit upfront → Balance per guest at checkout

V1: Option A only (full master)
Phase 2: All options available
```

---

## 5. Group Analytics *(Phase 2)*

```
Top Organizers:         Revenue, repeat bookings, avg group size
Peak Group Months:      Seasonality for group blocking strategy
Group Revenue %:        What % of total revenue comes from groups
Insight:                "Wedding Planner Pro is top source — offer loyalty rate"
```

---

## PART 2 — CORPORATE ACCOUNTS

---

## 6. Corporate Account Setup

```
CORPORATE ACCOUNT — Infosys Ltd

Valid:              Jan 1 - Dec 31, 2026
Rate:               ₹3,000/night (all types — 25% off rack)
Breakfast:          Included
Cancellation:       Free cancel up to 24 hours
Volume commitment:  150 room nights/year

USAGE TRACKER:
  Used: 112 nights | Remaining: 38 nights
  Revenue: ₹3,36,000 | Commission saved vs OTA: ₹48,000

ALERTS:
  ⚠ Expiry in 45 days → Account manager notified
  ⚠ 80% volume used → Notify company (encourage more bookings)
```

---

## 7. Corporate Booking Flow

```
V1 — Phone / Email:
  Employee gives company name
  → System detects active contract → Rate auto-applied
  → No manual lookup, no negotiation
  → Booking confirmed in 30 seconds

Phase 2 — Self-booking portal:
  Company HR gives employees a booking link
  Employee selects dates → Corporate rate shown automatically
  Books directly → No staff needed

Phase 3 — TMC integration:
  Company uses Travel Management Company (FCM, American Express Travel)
  TMC books via GDS → Corporate rate auto-applies
```

---

## 8. Corporate Monthly Invoice — Auto

```
AUTO INVOICE — Infosys Ltd | November 2026

Booking  Guest        Dates      Nights  Amount
#1234    Rajesh K.    Nov 3-5    2       ₹6,000
#1456    Priya M.     Nov 8-9    1       ₹3,000
#1678    Arjun S.     Nov 12-15  3       ₹9,000
─────────────────────────────────────────────────
Room subtotal:                   6       ₹18,000
GST (12%):                               ₹2,160
─────────────────────────────────────────────────
TOTAL DUE:                              ₹20,160

Auto-generated: 1st of every month
Auto-emailed:   travel@infosys.com
Payment due:    30 days
Overdue:        Auto-reminder → Account manager alerted
```

---

## 9. Multi-Property Corporate *(Phase 2)*

```
Infosys contract across all our hotel properties:
  Hotel A (Bangalore): ₹3,000/night
  Hotel B (Chennai):   ₹2,800/night
  Hotel C (Hyderabad): ₹3,200/night

ONE contract → Works across all properties
Volume counted across all hotels (150 nights total, any location)
ONE consolidated monthly invoice for all properties

Native advantage of our multi-tenant SaaS platform.
```

---

## V1 vs Later

```
V1 MUST HAVE:
  GROUP:
    ✅ Group block creation (staff-created)
    ✅ Room type hold with dates
    ✅ Cut-off date alert (manual)
    ✅ Rooming list (Excel upload — basic)
    ✅ Group rate setup
    ✅ Master billing (full master only)

  CORPORATE:
    ✅ Corporate account setup (rate + dates)
    ✅ Rate auto-apply on booking
    ✅ Monthly invoice generation (basic)
    ✅ Usage tracking (nights used vs committed)

PHASE 2:
  ➕ Agent self-service portal
  ➕ Auto cut-off release
  ➕ Flexible group billing (department split)
  ➕ Group analytics
  ➕ Employee self-booking portal
  ➕ Multi-property corporate contract
  ➕ Auto invoice + payment tracking

PHASE 3:
  ➕ TMC / GDS integration
  ➕ Corporate loyalty program
  ➕ AI group revenue optimization
```

---

## Comparison Summary

| Feature | Existing Systems | Our System |
|---------|-----------------|------------|
| Group creation | Email + manual entry | Structured form, auto block |
| Rooming list | Manual entry | Excel upload → bulk create (Phase 2: portal) |
| Cut-off management | Staff memory | Auto-release + alerts (Phase 2) |
| Group billing | Basic only | Flexible split (Phase 2) |
| Corporate rates | Paper contract + manual | Auto-detect + auto-apply |
| Corporate invoice | Manual Excel | Auto-generated + emailed |
| Multi-property corporate | Not available | Phase 2 (platform native) |
| Self-booking portal | Not available | Phase 2 |
