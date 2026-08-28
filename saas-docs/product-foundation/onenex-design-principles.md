# OneNex — Design Principles

> The foundational rules that govern every design and product decision in OneNex.

---

## What OneNex Is

A **Hospitality Business Operating System** for business owners.

Any business type, any combination, any size — operated from one platform.
The owner owns the business. OneNex runs the operations.

---

## Core Architecture

```
OWNER (Account)
│
├── Business A: "Grand Hotel Colombo"
│   ├── Operation: Stays
│   ├── Operation: Dining
│   └── Operation: Wellness
│
└── Business B: "Beach Shack Galle"
    └── Operation: Dining
```

| Level | What | Example |
|---|---|---|
| **Owner** | Account holder | Hotel chain owner |
| **Business** | Physical location / entity | "Grand Hotel Colombo" |
| **Operation** | What that business does | Stays, Dining, Wellness |

---

## Principle 1 — Type is Derived, Never Declared

Owner never manually selects "Hotel type" or "Restaurant type."

```
Stays operation enabled              → System labels: "Hotel"
Dining only                          → "Restaurant"
Stays + Dining + Wellness            → "Hotel & Resort"
Events + Dining                      → "Event Venue"
```

> **Operations define what you are. You never declare it.**

---

## Principle 2 — Independent by Default, Connected by Choice

Every business starts completely standalone. No links, no dependencies.

Link = Owner explicitly chooses to share resources between two businesses.

**Shared resource types (Link dimensions):**

| Resource | What it enables |
|---|---|
| **Folio** | Hotel guest charges restaurant/spa bill to room |
| **Kitchen / Bar** | One central kitchen serves multiple businesses |
| **Staff** | Same staff works across multiple businesses |
| **Inventory** | Shared stock across businesses |
| **Guest CRM** | Same customer recognized across all businesses |

No link record = businesses don't know each other exist.
Link record = shared resources activate.

> **Connected when needed. Never forced.**

---

## Principle 3 — Folio is the Interconnection Hub

Operations never communicate directly with each other.
All charges flow to the Folio.

```
Guest checks in → Folio created
    │
    ├── Restaurant order  → posts to folio
    ├── Spa treatment     → posts to folio
    ├── Bar tab           → posts to folio
    └── Room charge       → posts to folio (nightly, via Night Audit)

Checkout → ONE bill. Everything settled.
```

Folio lives at **Business level** — not at operation level.
Owner-level view = aggregated reporting across all businesses.

> **Folio = the single financial truth for a guest's stay.**

---

## Principle 4 — Staff & Guests Live at Owner Level

```
OWNER
├── Staff Pool
│   └── Arun → assigned to: Grand Hotel + Garden Restaurant
│              Role @ Hotel: Floor Manager
│              Role @ Restaurant: Supervisor
│              ONE login. ONE schedule. Unified payroll.
│
└── Guest CRM
    └── Rahul → stayed at hotel 3x, visited restaurant 8x
               ONE profile. Full history across all businesses.
```

> **People (staff + guests) belong to the Owner, not to individual businesses.**

---

## Principle 5 — Simple by Default, Powerful when Needed

```
L1 Simple     → Small standalone restaurant. Basic POS, menu, orders.
L2 Standard   → Multi-table, reservations, basic inventory, reports.
L3 Advanced   → Multi-branch, yield management, channel sync.
L4 Enterprise → Full resort, OTA channels, corporate accounts, GDS.
```

All features exist in the system. Complexity is hidden, revealed on demand.

> **Small business uses 10% of features. Enterprise uses 90%. Same platform.**

---

## Principle 6 — Progressive Disclosure

Advanced features are never shown to those who don't need them.

```
Standalone restaurant owner:
→ Creates business → sets up dining → works. Done.
→ Never sees "link", "folio", "channel management", "yield rules."

Owner adds second business:
→ System asks: "Connect these businesses?"
→ Yes → link configuration appears.
→ No → both work standalone. Zero added complexity.
```

> **Features appear when relevant. Never before.**

---

## Principle 7 — No Assumptions About Customer Behavior

The system handles every scenario naturally. Nothing is assumed.

```
Hotel guest visits restaurant     → charge to folio (works)
Walk-in visits same restaurant    → direct bill (works)
Same table, different customers   → both scenarios handled naturally
```

> **System adapts to reality. Reality never adapts to the system.**

---

## Principle 8 — V1 Philosophy: Ship Simple, Stay Future-Ready

Every feature is evaluated against one question:

> "Can the business operate without this?"
> - YES → Phase 2 or Phase 3
> - NO  → V1

- **Data model** → always future-ready (even if V1 UI is simple)
- **UI** → only what's needed now
- **Architecture** → never painted into a corner

> **Ship simple. Iterate fast. Never break the foundation.**

---

## Full Architecture Diagram

```
┌──────────────────────────────────────────┐
│              OWNER ACCOUNT               │
│   Staff Pool  |  Guest CRM  |  Analytics │
└───────────────┬──────────────────────────┘
                │
     ┌──────────┴───────────┐
     │                      │
┌────▼──────────┐   ┌───────▼───────┐
│  Business A   │◄─►│  Business B   │   ← Link (optional, owner-chosen)
│               │   │               │
│  [Operation]  │   │  [Operation]  │
│  [Operation]  │   │               │
│               │   │               │
│   [Folio] ←──┼───┼── charges     │   ← Folio per business
└───────────────┘   └───────────────┘

     Simple by Default → Progressive Disclosure → Powerful when Needed
```

---

## Summary

| Principle | Rule |
|---|---|
| Type | Derived from operations. Never declared. |
| Businesses | Independent by default. Linked by choice. |
| Interconnection | Everything flows through Folio. |
| People | Staff and Guests live at Owner level. |
| Complexity | Hidden by default. Revealed progressively. |
| Behavior | No assumptions. All scenarios handled. |
| Shipping | Simple V1. Future-ready data model. |
