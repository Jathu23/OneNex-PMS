# OneNex — Backend Architecture

> Draft — Needs team discussion before finalizing.
> ASP.NET Core backend architecture decisions for OneNex.

---

## Architecture Choice: Modular Monolith + Clean Architecture + CQRS + DDD

---

## Why Modular Monolith (Not Pure Monolith, Not Microservices)

### Why NOT Microservices

| Problem | OneNex Impact |
|---|---|
| Distributed transactions | Folio = financial data. ACID mandatory. Distributed = saga complexity. |
| Network calls | Payment + Notification + CRM called constantly. Every hop = latency + failure. |
| Multi-tenant RLS | Must implement in every service separately. Drift risk = data leak. |
| Night audit | Cross-module batch operation. Distributed coordination = nightmare. |
| SignalR real-time | Hub needs direct in-process access to module data. |
| Team size | Infrastructure overhead kills product development speed. |

### Why NOT Pure Monolith

Pure Monolith + Clean Architecture = intention only. Nothing enforces boundaries.

```csharp
// Nothing stops this in a pure monolith:
var room = _staysRepository.GetRoom(roomId); // Dining accessing Stays internals
// Compiles. Runs. Wrong. Over time → Big Ball of Mud.
```

### Why Modular Monolith

```
Pure Monolith    = "We promised to keep it clean."
Modular Monolith = "The compiler keeps it clean."
```

Each module is a separate C# project. Wrong cross-module references = compile error. Boundaries are permanent.

---

## Two Levels of Vertical Slicing

### Level 1 — Module Slices (By Operation/Domain)

```
src/
├── Modules/
│   │
│   ├── [OPERATION MODULES]
│   ├── Dining/              ← full vertical slice
│   ├── Stays/               ← full vertical slice
│   ├── Bar/
│   ├── Wellness/
│   ├── Events/
│   └── Retail/
│
│   ├── [SHARED SERVICE MODULES]
│   ├── Auth/
│   ├── Payments/
│   ├── Notifications/
│   ├── CRM/
│   └── Staff/
│
└── Shared.Contracts/        ← interfaces only — modules communicate through this
```

### Level 2 — Feature Slices (Within Each Module)

Within Dining module:

```
Dining/
├── Domain/
│   ├── Entities/            (Order, Table, MenuItem)
│   ├── Events/              (OrderPlacedEvent, TableStatusChangedEvent)
│   └── ValueObjects/        (OrderStatus)
│
├── Application/
│   └── Features/
│       ├── Orders/
│       │   ├── Commands/
│       │   │   ├── PlaceOrder/
│       │   │   │   ├── PlaceOrderCommand.cs
│       │   │   │   ├── PlaceOrderHandler.cs
│       │   │   │   ├── PlaceOrderValidator.cs
│       │   │   │   └── PlaceOrderDto.cs
│       │   │   └── CancelOrder/
│       │   └── Queries/
│       │       └── GetTableOrders/
│       ├── Menu/
│       │   ├── Commands/
│       │   └── Queries/
│       └── Tables/
│
├── Infrastructure/
│   ├── Repositories/
│   ├── EntityConfigurations/
│   └── DiningDbContext.cs
│
└── API/
    ├── Controllers/
    └── Hubs/               (SignalR — KDS real-time)
```

---

## Module Communication Rules

```csharp
// ❌ Wrong — direct internal access
_staysRepository.GetRoom(roomId);

// ✅ Correct — through shared interface
_staysService.GetAvailability(date); // IStaysService in Shared.Contracts
```

Modules know only interfaces. Never implementations.

---

## Domain Events — Loose Coupling Within Monolith

```
Order placed → OrderPlacedEvent (in-memory)
    ├── KDS Handler     → display on kitchen screen (SignalR)
    ├── Inventory Handler → deduct ingredients
    └── Notification Handler → alert kitchen staff

Modules don't call each other. They publish events. Others listen.
Future: in-memory events → message queue. Zero business logic change.
```

---

## Real-World Flow — Place Order

```
POST /api/dining/orders
      ↓
OrdersController              (API layer)
      ↓
PlaceOrderCommand             (CQRS)
      ↓
PlaceOrderHandler             (business logic)
  → validate
  → save (IDiningRepository)
  → publish OrderPlacedEvent
      ↓
  ┌───┴──────────────────┐
  ▼                      ▼
KDS Handler          Notification Handler
(SignalR → kitchen)  (alert staff)
```

Everything for this flow lives inside Dining module. No other module touched.

---

## Tech Stack

| Technology | Purpose |
|---|---|
| ASP.NET Core 8 | Framework |
| PostgreSQL | Primary DB — ACID for folio/financial |
| EF Core | ORM |
| MediatR | CQRS implementation |
| SignalR | Real-time (KDS, table status, room status) |
| Redis | Caching + session |
| Hangfire | Background jobs (night audit, scheduled notifications) |
| FluentValidation | Input validation |

### Why PostgreSQL over SQL Server

| | PostgreSQL | SQL Server |
|---|---|---|
| Cost | Free | Expensive license |
| Multi-tenant RLS | Built-in | Complex to implement |
| JSON support | Excellent | Limited |
| Cloud | Any cloud | Mostly Azure |

---

## Future Scaling Path

```
Modular Monolith today

IF specific module needs independent scaling:
→ Extract that module as standalone microservice
→ Module boundaries already defined
→ Business logic: ZERO change
→ Just deployment change

Example: Notifications hit 10M/day
→ Extract Notification module → standalone service
→ Everything else stays in monolith
```

---

## Open Questions (Discuss with Team)

- PostgreSQL vs SQL Server — team preference? Existing infrastructure?
- EF Core vs Dapper — or hybrid (EF Core writes, Dapper reads)?
- MediatR for CQRS — or custom mediator pattern?
- Shared database vs database per module?
- Event bus — in-memory only (V1) or introduce RabbitMQ/Azure Service Bus early?
- Background jobs — Hangfire vs Azure Functions vs custom hosted service?
- Logging & monitoring stack — Serilog + Seq? Application Insights?
- Testing strategy — unit per feature slice? Integration tests per module?
