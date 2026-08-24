# Room Booking — 01: Booking Types
> Hotel Module → Room Booking → Topic 1 of 7

---

## Overview

Oru hotel-la booking பண்ண வேற வேற sources இருக்கும். Each source-ku different flow, different rate, different billing இருக்கும். Itha system design பண்ணும்போது ellatha oru data model-la handle பண்ண வேணும் — but create பண்ற flow மட்டும் differ ஆகும்.

---

## 6 Booking Types

### 1. Walk-in

Guest advance booking எதுவும் இல்லாம directly hotel-ku வருவான்.

**Flow:**
```
Guest front desk-la வருவான்
        ↓
Staff real-time availability check பண்ணுவாங்க
        ↓
Available room types காட்டுவாங்க
        ↓
Guest room type select பண்ணுவான்
        ↓
Rate confirm → Guest details enter
        ↓
Deposit collect (policy based)
        ↓
Booking created → Immediate or future check-in
```

**Key Points:**
- No advance notice — room ready-aa இருக்கணும்
- Rate: Rack rate (full price, no discount) — பெரும்பாலும்
- Immediate check-in or future date — both possible
- High revenue per room (no OTA commission, no discount)

---

### 2. Phone / Staff Created

Guest phone பண்ணி book பண்றான் — Staff system-la manually create பண்றாங்க.

**Flow:**
```
Guest calls hotel
        ↓
Staff availability check பண்றாங்க (system-la)
        ↓
Dates, room type, guest count discuss பண்றாங்க
        ↓
Rate negotiate / confirm பண்றாங்க
        ↓
Staff system-la booking create பண்றாங்க
        ↓
Guest details (name, phone, email) enter பண்றாங்க
        ↓
Auto confirmation SMS / Email → Guest-ku போகும்
        ↓
Booking saved — Guest future date-la வருவான்
```

**Key Points:**
- Staff-side action, guest-side app வேண்டாம்
- Rate: Negotiable — staff discount apply பண்ணலாம்
- Deposit: Phone-la card details collect பண்ணலாம் or pay at hotel
- Personal touch — guest-ku direct conversation

---

### 3. Online Direct (Website / App)

Guest hotel's own website or our customer app-la self-book பண்றான்.

**Flow:**
```
Guest hotel website / app open பண்றான்
        ↓
Check-in date, Check-out date, Guests count enter
        ↓
Available room types + rates show ஆகும்
        ↓
Room select → Rate plan select (flexible / non-refundable)
        ↓
Guest details fill பண்றான்
        ↓
Payment (deposit or full — policy based)
        ↓
Auto confirmation email / SMS
        ↓
Booking system-la appear ஆகும் (no staff involvement)
```

**Key Points:**
- 100% self-serve — no staff needed
- Best rate guaranteed (cheaper than OTA — no commission)
- Rate: Published rate, promo codes apply பண்ணலாம்
- Commission-free → Hotel-ku full revenue
- Real-time availability shown

---

### 4. OTA (Online Travel Agencies)

Guest Booking.com, Airbnb, Expedia, Agoda போன்ற platforms-la book பண்றான்.

**Flow:**
```
Guest Booking.com-la hotel search பண்றான்
        ↓
Our hotel list ஆகுது (availability synced)
        ↓
Guest book பண்றான் (Booking.com-la)
        ↓
Booking.com → Our system-ku auto push பண்றது (API)
        ↓
Our system: Booking create + Availability update
        ↓
Staff notification: "New Booking.com reservation — Room 102, Dec 15"
        ↓
Guest arrives → Normal check-in flow
```

**Key Points:**
- Real-time availability sync mandatory — stale data = accidental overbook
- Rate: OTA rate (includes commission — 10-20% goes to OTA)
- Payment: OTA collect பண்றாங்க and remit to hotel (or hotel collects at checkout — policy based)
- Modification/cancellation: Goes through OTA → sync to our system
- Highest volume source — but lowest margin

---

### 5. Group Booking

Travel agent, corporate company, event organizer — bulk rooms book பண்றாங்க.

**Flow:**
```
Agent / Organizer contact பண்றாங்க: "20 rooms, Dec 15-18 வேணும்"
        ↓
Staff availability check + rate negotiate
        ↓
Contract agree பண்றாங்க (group rate, deposit schedule, cut-off date)
        ↓
Staff system-la Group Block create பண்றாங்க
  (20 Double rooms — held, not yet assigned to specific guests)
        ↓
Rooms inventory-la blocked (general public-ku sold out ஆகும்)
        ↓
Cut-off date-ku முன்னாடி: Rooming list submit பண்றாங்க
  (யாரு room 101-la, யாரு 102-la என்று)
        ↓
Cut-off date pass ஆனா → Released rooms back to general inventory
        ↓
Individual guests arrive → Normal check-in
        ↓
Billing: Master account (company pays all) or individual
```

**Key Points:**
- Minimum threshold (eg: 5+ rooms = group booking)
- Group rate — lower per room but guaranteed volume
- Cut-off date — by this date agent must confirm all rooms, else released
- Rooming list — individual guest assignment
- One master folio (company pays) or split folios (each guest pays)

---

### 6. Corporate Booking

Specific company-oda employees stay பண்றாங்க — pre-negotiated contract rate-la.

**Flow:**
```
Company ABC signs corporate agreement with hotel
  (Rate: ₹3,000/night, Min 50 room nights/month)
        ↓
Employee travel பண்ண வேணும் → Phone / Company portal-la book
        ↓
System: Corporate account detect → Contract rate auto-apply
        ↓
Employee check-in → Normal flow
        ↓
Billing: 
  Option A: Employee pays personally
  Option B: Charge to company master account
        ↓
Month-end: Consolidated invoice → Company-ku
```

**Key Points:**
- Pre-negotiated rate (lower than rack rate)
- Volume commitment from company side
- Monthly consolidated billing option
- Employee doesn't need to pay every time (company credit)
- Corporate account manager (hotel staff) manages relationship

---

## Booking Source Tracking — Important

Every booking-la **Source field** mandatory:

```
Source values:
  WALK_IN
  PHONE
  DIRECT_ONLINE
  BOOKING_COM
  AIRBNB
  EXPEDIA
  AGODA
  GROUP
  CORPORATE
  OTHER_OTA
```

**Why tracking matters:**
- Which channel brings most revenue?
- Which channel has highest cancellation rate?
- OTA commission cost vs direct booking savings
- Staff time spent per channel
- Marketing ROI calculation

---

## Summary Comparison

| Booking Type | Who Books | Rate Type | Commission | Staff Effort |
|-------------|-----------|-----------|------------|-------------|
| Walk-in | Guest (in-person) | Rack Rate | None | Medium |
| Phone | Guest (via call) | Negotiable | None | High |
| Online Direct | Guest (self-serve) | Published | None | None |
| OTA | Guest (via OTA) | OTA Rate | 10-20% | Low |
| Group | Agent/Company | Group Rate | Sometimes | High |
| Corporate | Employee | Contract Rate | None | Low |

---

## Data Model Note

All 6 types → Same `Booking` table in database.
Difference: `source` field + `booking_type` field.
Rate calculation and billing flow differ per type — but core booking data same.
