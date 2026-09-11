# OneNex — Software Surfaces

> All surfaces (interfaces) that exist in OneNex.
> Each surface: specific WHO uses it, WHERE, and WHAT they do.
> Draft — needs team discussion before finalizing.

---

## Core Principle

```
ONE backend → Multiple surfaces
Each surface is purpose-built for a specific role + device + context.
```

---

## Surface 1: Owner Portal

```
WHO    → Business owner
WHERE  → Desktop / laptop (browser)
URL    → app.onenex.com
WHAT   → All businesses overview
         Analytics & reports (cross-business)
         Staff management (all businesses)
         Subscription & billing (OneNex plan)
         Business settings, operation setup
         Enable / disable add-ons per business

Operations: ALL
Device: Web — desktop first
```

---

## Surface 2: Operations Hub (Staff Web)

```
WHO    → Manager, Supervisor, Front Desk Agent
WHERE  → Desktop / laptop / tablet (browser)
URL    → {slug}.onenex.com
WHAT   → Day-to-day operations management

         Dining:    table management, reservations, orders overview
         Stays:     bookings calendar, check-in/out, folio management
         Wellness:  appointments calendar, therapist schedule
         Bar:       tab management, orders overview
         Events:    event management, attendee management
         Retail:    inventory, sales overview

Operations: All (view adapts per enabled operations)
Device: Web — desktop + tablet
```

---

## Surface 3: POS Terminal

```
WHO    → Cashier, Waiter, Bartender, Retail Staff
WHERE  → Fixed tablet at counter (touch screen)
URL    → {slug}.onenex.com/pos (or dedicated POS app)
WHAT   → Take orders
         Process payment (cash / card / digital)
         Apply discount
         Split bill
         Print / send receipt
         Void / refund

Operations: Dining, Bar, Retail, Events (ticket sales)
Device: Fixed tablet — touch optimized, full screen
Login: Short PIN (backed by staff identity)
```

---

## Surface 4: KDS — Kitchen Display System

```
WHO    → Chef, Kitchen Staff, Bartender (bar KDS)
WHERE  → Fixed screen in kitchen / bar (wall mounted)
WHAT   → View incoming orders in real-time
         See order details + special instructions
         Mark items as ready
         Manage ticket queue (oldest first)
         Station routing (grill sees grill items only)

Operations: Dining, Bar
Device: Fixed screen — Android tablet / smart TV
Interaction: Minimal touch — mostly view + simple tap
Real-time: SignalR push (instant — no refresh needed)
```

---

## Surface 5: Handheld — Floor Staff Ordering

```
WHO    → Floor waiter, Server
WHERE  → Their phone / handheld device (on the floor)
WHAT   → Take table orders at the table
         Send order directly to KDS
         View table status
         Note special requests
         (Payment handled at POS — not on handheld)

Operations: Dining, Bar
Device: Mobile phone (iOS / Android)
Phase: Phase 2 (V1 uses POS for order entry)
```

---

## Surface 6: Housekeeping App

```
WHO    → Housekeeper, Maintenance Staff
WHERE  → Their mobile phone (on the floor, room to room)
WHAT   → View assigned rooms for the shift
         Update room status:
           Dirty → Cleaning → Clean → Inspected
         Room cleaning checklist
         Report maintenance issues
         View guest special notes (VIP, allergies)

Operations: Stays
Device: Mobile phone (iOS / Android)
Real-time: Status update pushes to Operations Hub instantly
Phase: Phase 2 (V1: simple status update via Operations Hub)
```

---

## Surface 7: Customer Web & App

```
WHO    → Guest / Customer (end consumer)
WHERE  → Their own phone / browser
WHAT   →
  Dining:   QR scan → view menu → order → pay at table
  Stays:    Room booking, mobile check-in, view folio
  Wellness: Appointment booking, service catalog
  Events:   Ticket purchase, event info
  All:      Loyalty points, order history, preferences

Operations: All (based on what business has enabled)
Device: PWA (web app) first → Native app Phase 2
Phase: Phase 2 for most. QR ordering may be Phase 1 add-on.
```

---

## Surface 8: Self-Service Kiosk

```
WHO    → Walk-in customer (no staff assistance)
WHERE  → Fixed kiosk at entrance / counter (stand or wall)
WHAT   → Browse menu
         Place order
         Pay (card / digital)
         Get receipt
         No staff needed for basic transactions

Operations: Dining (QSR / fast food), Retail, Events
Device: Large touch tablet (fixed — stand/wall mounted)
Phase: Phase 3
```

---

## Surfaces × Operations Matrix

```
Surface                  Dining  Stays  Bar  Wellness  Events  Retail
──────────────────────────────────────────────────────────────────────
Owner Portal               ✓       ✓      ✓     ✓         ✓       ✓
Operations Hub             ✓       ✓      ✓     ✓         ✓       ✓
POS Terminal               ✓       -      ✓     ✓         ✓       ✓
KDS                        ✓       -      ✓     -         -       -
Handheld (floor staff)     ✓       -      ✓     -         -       -
Housekeeping App           -       ✓      -     -         -       -
Customer Web / App         ✓       ✓      -     ✓         ✓       ✓
Self-Service Kiosk         ✓       -      -     -         ✓       ✓
```

---

## V1 Build Priority

```
Phase 1 — Dining:
  ✓ Owner Portal
  ✓ Operations Hub (table management, orders overview, reports)
  ✓ POS Terminal (order + payment)
  ✓ KDS (kitchen display — real-time)

Phase 2 — Stays + Dining expand:
  ✓ Operations Hub → Stays (bookings, check-in/out, folio)
  ○ Handheld (floor waiter ordering)
  ○ Housekeeping App
  ○ Customer Web (QR ordering, room booking)

Phase 3+:
  ○ Native Customer App
  ○ Self-Service Kiosk
  ○ Bar, Wellness, Events, Retail surfaces
```

---

## Key Design Principles

| Principle | Detail |
|---|---|
| One backend, many surfaces | All surfaces connect to same API |
| Real-time sync | KDS sees order instantly when POS places it (SignalR) |
| Role-based views | Same data, different surface per role |
| Mobile-first for staff | Operations happen on the floor — mobile critical |
| Guest-first for customer | Polished, simple, branded experience |
| Device agnostic | Web-based where possible, native where needed |
| PIN login at terminals | Fast login at POS/KDS — full credentials on web |

---

## Open Questions (Discuss with Team)

- Operations Hub + POS — same app different view? Or completely separate?
- KDS — dedicated app or browser-based?
- Handheld ordering — Phase 1 or Phase 2?
- Customer web — PWA or native app first?
- Single codebase for all surfaces (Angular/React) or separate codebases?
- Offline capability — POS must work if internet drops?
