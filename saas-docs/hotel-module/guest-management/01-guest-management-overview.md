# Guest Management — Overview & Smart Features
> Hotel Module → Guest Management
> Approach: Existing systems analyzed → Better ideas documented
> Implementation: Phase 2+ (skip in V1, implement later)

---

## Existing Systems — Common Problems

| System | Weakness |
|--------|---------|
| Oracle OPERA | Guest data siloed — only hotel stays, no cross-module link |
| Mews | Better UI but still module-isolated, no behavioral intelligence |
| Cloudbeds | Basic profiles only, no segmentation, no automation |

**Core problem:** Guest profiles are data stores, not intelligence engines.
Data sits there — system doesn't USE it proactively.

---

## Features Covered

| # | Feature | Priority |
|---|---------|---------|
| 1 | 360° Guest Profile | Phase 2 |
| 2 | Cross-Module Guest Identity | Phase 2 |
| 3 | Automated VIP Detection | Phase 2 |
| 4 | Preference Learning (Auto) | Phase 3 |
| 5 | Pre-Arrival Engagement Journey | Phase 2 |
| 6 | Guest Segmentation | Phase 3 |
| 7 | Complaint & Recovery Management | Phase 2 |
| 8 | Privacy & GDPR Controls | Phase 2 |

---

## 1. Guest Profile — 360° View

### Existing Approach
```
Name, phone, email, past stay dates. That's it.
No cross-module data. No intelligence.
```

### Our Approach — Full 360° Profile

```
┌─────────────────────────────────────────────────────────┐
│ GUEST PROFILE — Rajesh Kumar                            │
│ ID: G-10234 | Member since: Jan 2023 | 🥇 Gold Tier    │
├─────────────────────────────────────────────────────────┤
│ CONTACT                                                  │
│  📱 +91 98765 43210  ✉ rajesh@gmail.com                │
│  🎂 Born: Mar 15, 1985 | 🏢 Company: Infosys Ltd       │
├─────────────────────────────────────────────────────────┤
│ LIFETIME VALUE (All Modules)                            │
│  Total spend:   ₹4,82,000                               │
│  Hotel stays:   14 nights | 6 visits                    │
│  Restaurant:    ₹38,000 (22 visits)                     │
│  Spa:           ₹18,500 (5 sessions)                    │
│  Events:        ₹12,000 (3 tickets)                     │
├─────────────────────────────────────────────────────────┤
│ PREFERENCES (learned from behavior)                     │
│  Room: High floor | City view | King bed                │
│  AC: 22°C always | Pillow: Extra firm                   │
│  Food: No pork | Prefers South Indian | Filter coffee   │
│  Parking: Always needs | Late check-in (avg 7 PM)       │
├─────────────────────────────────────────────────────────┤
│ BEHAVIORAL PATTERNS (system-detected)                   │
│  • Accepts upgrades: 4/4 times ✅                       │
│  • Visits bar on Tuesday nights                         │
│  • Books spa on Day 2 of every stay                     │
│  • Always books direct (never OTA)                      │
├─────────────────────────────────────────────────────────┤
│ COMPLAINT HISTORY                                       │
│  ⚠ Nov 2025: AC noise — Room 203 (avoid this room)     │
│  ✅ Aug 2025: Praised staff "Priya"                     │
└─────────────────────────────────────────────────────────┘
```

---

## 2. Cross-Module Guest Identity

**Problem:** Same guest = 3 separate profiles in Hotel, Restaurant, Spa.
Hotel doesn't know their restaurant spend. Restaurant doesn't know they're a VIP hotel guest.

**Solution:** Single guest identity across ALL modules.

```
Rajesh walks into restaurant (not staying today)
Waiter enters phone number → System: GOLD MEMBER — 14 hotel nights
Waiter sees VIP status → Treats accordingly → Visit added to profile
Hotel team sees: "Rajesh visited restaurant 3x this month" → Re-engage trigger
```

---

## 3. Automated VIP Detection

**Problem:** VIP tags added manually, inconsistently, often forgotten.

**Solution:** Auto VIP scoring engine.

```
Score based on:
  ├── Lifetime spend
  ├── Visit frequency
  ├── Loyalty tier
  ├── Direct booking history
  └── Complaint history (negative factor)

Score → Tier:
  0-40:   Regular Guest
  41-60:  Valued Guest (small perks)
  61-80:  VIP (priority, upgrades)
  81-100: Ultra VIP → GM notified on booking, full protocol triggered
```

---

## 4. Preference Learning — Automatic

**Problem:** Preferences require manual entry. Most never get recorded.

**Solution:** System observes behavior and learns.

```
Guest never says "I like high floors"

System observes:
  Stay 1: Assigned floor 3 → Requested change to floor 7
  Stay 2: Assigned floor 2 → Called, moved to floor 8
  Stay 3: Auto-assigned floor 8 → 5-star rating, no complaints

System auto-adds: "Prefers high floor (confidence: HIGH)"
Future stays: High floor rooms auto-suggested
```

Same logic for: dining preferences, spa timing, minibar items, AC temperature.

---

## 5. Pre-Arrival Engagement Journey

**Problem:** After booking confirmation, guest hears nothing until check-in. Missed upsell + delight opportunity.

**Solution:** Automated 7-day pre-arrival journey.

```
T-7 days: Welcome email + Gold member benefits reminder
T-3 days: Upsell offer ("Pre-book your usual spa session?")
T-1 day:  Arrival prep ("Online check-in available — skip counter")
T-4 hrs:  Room ready notification (if ready early)
Day of:   Digital key sent (Layer 3+)
```

---

## 6. Guest Segmentation

**Problem:** All guests treated the same in marketing.

**Solution:** Auto-segments based on behavior.

| Segment | Signals | Targeted Offer |
|---------|---------|---------------|
| Business Traveler | Weekday, solo, early checkout, company billing | Corporate rate, loyalty fast-track |
| Leisure | Weekend, couple/family, spa usage | Romance package, activity deals |
| Budget | OTA bookings, basic room, low F&B | Direct booking discount |
| Luxury | Suites, high spend, direct bookings | Exclusive packages, member events |
| Special Occasion | Anniversary/birthday detected | Surprise amenity, celebration dinner |

---

## 7. Complaint & Recovery Management

**Problem:** Complaints noted ad-hoc, never systematically tracked. Same issue repeats on next visit.

**Solution:** Complaints linked to guest profile + room + staff.

```
Complaint logged:
  → Guest profile updated
  → Maintenance ticket auto-created
  → Room flagged (avoid assigning same room next visit)
  → Compensation offered
  → Recovery tracked (did guest rate well after recovery?)
```

---

## 8. Privacy & GDPR Controls

```
Guest rights (via app or front desk):
  [View my data]    → Full profile download
  [Correct my data] → Update preferences, contact
  [Delete my data]  → Anonymize personal data (keep financial records 7 yrs)

Data retention:
  Inactive 3 years → Auto-archive
  Inactive 5 years → Auto-delete
  Financial records → 7 years (statutory)
```

---

## Comparison Summary

| Feature | Existing Systems | Our System |
|---------|-----------------|------------|
| Profile depth | Basic contact + stays | 360° — all modules linked |
| Preferences | Manual entry | Auto-learned from behavior |
| Cross-module identity | Siloed | Single identity everywhere |
| VIP detection | Manual tagging | Auto-scoring engine |
| Pre-arrival | Confirmation email only | 7-day automated journey |
| Segmentation | None | Auto-segments + targeted offers |
| Complaints | Ad-hoc notes | Tracked, linked, prevents repeat |
| Privacy | Minimal | Full GDPR-compliant guest control |
