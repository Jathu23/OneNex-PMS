# SaaS Platform — Design Philosophy
> "Simple by Default. Powerful when needed."
> Status: Draft | Last Updated: 2026-08-21

---

## Core Principle

Features ellam system-la irukkum — but complexity hidden-aa irukkum.
User எவ்வளவு deep-a போனாலும் போகலாம், போகலன்னாலும் default-a ellam work aagum.

---

## 3 Setup Layers

### Layer 1 — Simple Setup
> Small Guesthouse / Budget Hotel / Small Restaurant

```
Setup: Minimum config. Works out of the box.
Operation: Basic flow only. Zero learning curve.
```

- Sensible defaults for everything
- No configuration required to get started
- Core operations work immediately after signup

---

### Layer 2 — Standard Setup
> Mid-size Hotel / Growing Restaurant

- Multiple room/item types
- Pricing variations (weekday/weekend, categories)
- Online booking/ordering
- Basic workflow management (housekeeping, staff)
- Guest/customer profiles
- Confirmation notifications

---

### Layer 3 — Advanced Setup
> Large Hotel / Restaurant Chain

- Rate plans & yield management
- OTA / 3rd party integrations
- Group bookings & block management
- Complex folio & charge routing
- Multi-property / multi-branch management
- Custom policies (cancellation, deposit, refund)
- Deep analytics & revenue reports

---

### Layer 4 — Luxury / Enterprise Setup
> 5-Star Resort / Enterprise Chain

- VIP guest profiles (preferences, history, special requests)
- Automated upselling & upgrade offers
- Pre-arrival experience workflows
- Dynamic / AI-based pricing engine
- Full white-labeling (own brand, own app)
- Custom workflow builder
- Public API access for 3rd party integrations
- SLA guarantees & dedicated support

---

## How the System Determines Layer

```
Onboarding Wizard:
  "What type of property are you?"

  ○ Small Guesthouse / B&B       → Layer 1 defaults
  ○ Mid-size Hotel               → Layer 2 defaults
  ○ Large Hotel / Chain          → Layer 3 defaults
  ○ Luxury Resort / Enterprise   → Layer 4 defaults

  "You can always unlock more features later."
```

- Defaults load automatically based on selection
- Advanced Settings always available behind a toggle
- Simple users never see complexity they don't need
- Advanced users can always go deeper

---

## Real Example — Room Pricing

**Simple User sees:**
```
Room Price: ₹2000/night
[Save]
```

**Advanced User sees:**
```
Base Rate:                  ₹2000
Weekend Rate:               ₹2800
Peak Season Rate:           ₹4500
Corporate Rate:             ₹1800
Early Bird (30 days):       15% off
Last Minute (2 days):       20% off
OTA Rate (Booking.com):     ₹2200
Minimum Stay:               1 night
Yield Management:           [Enable]
```

Same system. Same database. Different view based on setup level.

---

## Key Design Principles

| Principle | What it means |
|-----------|--------------|
| **Progressive Disclosure** | Show simple first, reveal complexity on demand |
| **Sensible Defaults** | Every setting has a smart default — works without config |
| **Non-destructive Complexity** | Advanced features don't break simple usage |
| **Upgrade Path** | Business grows → just unlock next layer, no migration needed |
| **Consistent Core** | Same data model for all layers — no separate systems |

---

## The Result

```
Small Guesthouse  → Uses 10% of features → Happy
Business Hotel    → Uses 40% of features → Happy
Luxury Resort     → Uses 90% of features → Happy

All on the SAME platform.
All paying different subscription tiers.
All getting exactly what they need.
```

---

## Applies To
This philosophy applies to **every module** in the platform:
- Hotel Module
- Restaurant Module
- Bar, Event, Retail, Spa — all follow the same layered approach
