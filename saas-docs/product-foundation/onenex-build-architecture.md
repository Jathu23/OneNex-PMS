# OneNex — Build Architecture & Layer Design

> How OneNex is structured from a system architecture perspective.
> What to build first, what depends on what, and why.

---

## The 3-Layer Architecture

"Kernel" என்பது ஒண்ணு இல்ல. 3 layers இருக்கு.

```
┌─────────────────────────────────┐
│       PLATFORM FEATURES         │  ← Built after operations exist
├─────────────────────────────────┤
│        SHARED SERVICES          │  ← Used by all operations
├─────────────────────────────────┤
│          PURE KERNEL            │  ← Must exist before anything
└─────────────────────────────────┘
         ↑ built first
```

---

## Layer 1 — Pure Kernel

**Cannot write a single line of feature code without these.**

| Component | What it does |
|---|---|
| Multi-tenant Core | Owner → Business → Operation data model. Data isolation between owners. |
| Auth & RBAC | Who can login. What they can do. Where (which business). |
| SaaS Billing | Subscription management — our billing to customers. |

Build: **Before everything else.**

---

## Layer 2 — Shared Services

**Every operation uses these. Each operation never reimplements them.**

| Service | Used by | Notes |
|---|---|---|
| Payment Service | All operations | Every operation collects money. Payment flow differs per operation. |
| Notification Service | All operations | Email / SMS / Push. Every operation sends alerts. |
| Staff Service | All operations | Staff lives at Owner level. Works across businesses. |
| Customer CRM | All operations | Same guest recognized everywhere. |
| File / Media Service | All operations | Menu images, room photos, documents. |

Build: **Alongside Phase 1 (first operation). Design first, build incrementally.**

---

## Layer 3 — Platform Features

**Only possible after operations exist and produce data.**

| Feature | Requires |
|---|---|
| Loyalty Engine | Customer CRM + 2+ operations |
| Offer / Promo Engine | Operations + Payment service |
| Analytics & Reports | Data from running operations |
| Audit Log | All layers running |
| Link features (cross-business folio, shared staff) | 2+ businesses + shared services |

Build: **Phase 2+ — after first operation is stable.**

---

## Feature Classification

| Feature | Layer | Build When |
|---|---|---|
| Multi-tenant data model | Pure Kernel | Phase 0 |
| Auth & RBAC | Pure Kernel | Phase 0 |
| SaaS subscription billing | Pure Kernel | Phase 0 |
| Payment processing | Shared Service | Phase 1 |
| Notifications | Shared Service | Phase 1 |
| Staff management | Shared Service | Phase 1 |
| Customer CRM | Shared Service | Phase 1 |
| Loyalty | Platform Feature | Phase 2+ |
| Offers / Promotions | Platform Feature | Phase 2+ |
| Analytics | Platform Feature | Phase 2+ |
| Cross-business linking | Platform Feature | Phase 2+ |

---

## Recommended Build Order

```
Phase 0 — Pure Kernel
├── Multi-tenant data model (Owner, Business, Operation)
├── Auth & RBAC
├── Design system / UI component library
└── CI/CD pipeline (dev → staging → prod)

Phase 1 — Shared Services + First Operation (Dining/Restaurant)
├── Payment Service
├── Notification Service
├── Staff Service (basic)
├── Customer CRM (basic)
├── Owner onboarding flow
├── Business + Dining operation setup
├── Table management
├── Menu management
├── Basic POS (order → payment)
└── End-of-day report

Phase 2 — Restaurant Complete
├── Table reservations
├── KDS (kitchen display system)
├── Inventory basics
├── Multi-branch
├── Staff scheduling
└── Basic analytics

Phase 3 — Hotel Module (Stays)
├── Stays operation (rooms, booking, check-in/out)
├── Folio system
├── Front desk
└── Housekeeping

Phase 4 — Link Concept
├── Cross-business folio charging
├── Shared staff across businesses
└── Combined analytics

Phase 5+ — Expand
├── Bar, Spa, Events, Retail operations
├── Loyalty engine
├── Offer / Promo engine
├── Channel management (OTA)
└── Customer-facing app
```

---

## Key Architectural Principles

**1. Design shared services before building operations**
Even if not fully built, the architecture must reserve the right shape for shared services.
Otherwise, retrofitting later is painful.

**2. Operations never reimplement shared concerns**
Payment logic lives in Payment Service — not in Restaurant module, not in Hotel module.
Notification templates live in Notification Service — not scattered per module.

**3. Pure Kernel is non-negotiable**
No feature development until multi-tenancy, auth, and RBAC are solid.
A data leak between tenants is a product-killing event.

**4. Platform Features need data to exist**
Loyalty without customers using multiple operations = meaningless.
Build Platform Features only when there's real data to power them.
