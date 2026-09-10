# OneNex — Backend Architecture Blueprint

> Status: Living Document — actively updated through research & discussion. Not final yet.
> Last updated: 2026-09-05
> Owner: Architecture Team

---

## Part 1 — Runtime & Framework

**`.NET 10 LTS` — decided.**

| Version | Type | Support Until | Decision |
|---|---|---|---|
| .NET 8 | LTS | Nov 2026 | Too close to EOL at launch |
| .NET 9 | STS | May 2026 | Already expired |
| **.NET 10** | **LTS** | **Nov 2028** | ✅ Use this |

SaaS product-ku support window 3+ years minimum. `.NET 10 LTS` only valid choice.
C# 14 features (primary constructors, collection expressions) production-ready by 2026.

---

## Part 2 — Architecture: Confirmed

```
Modular Monolith + Clean Architecture per Module + Vertical Slice per Feature + CQRS + DDD
```

### Why NOT Microservices

| Problem | OneNex Impact |
|---|---|
| Distributed transactions | Folio = financial data. ACID mandatory. Distributed = saga complexity. |
| Network call overhead | Payment + Notification + CRM called constantly. Every hop = latency + failure point. |
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

Each module = separate C# project. Wrong cross-module reference = compile error. Boundaries are permanent.

### Architecture Pattern: Two Levels

**Level 1 — Clean Architecture per Module:**
Each module organized into Domain / Application / Infrastructure / Presentation layers.
Enforces separation of concerns at module level.

**Level 2 — Vertical Slice per Feature (within Application layer):**
Within each module, features are self-contained folders (Command + Handler + Validator + DTO together).
Cross-cutting concerns (auth, logging, validation, transactions) = MediatR Pipeline Behaviors — not duplicated per feature.

See: [Detailed Explanation → concepts.md] (to be written)

---

## Part 3 — Open Questions: Resolved

### PostgreSQL vs SQL Server → PostgreSQL
- Free (no license cost)
- Row-Level Security built-in (multi-tenant ready)
- JSONB for rate snapshots, audit logs
- Any cloud deployment
- No reconsideration.

### EF Core vs Dapper → Hybrid
```
Commands (write): EF Core — change tracking, migrations, relationships
Queries  (read):  EF Core for simple / Dapper for complex joins + reports
```
Night audit, analytics, reports = Dapper.
KDS screen queries, availability search = Dapper.
CRUD operations = EF Core.

### MediatR for CQRS → Yes (MediatR 12)
Pipeline Behaviors make cross-cutting concerns clean. Custom mediator = reinventing the wheel.

### Shared DB vs DB per module → Shared DB, separate schemas
```sql
stays.rooms, stays.bookings, stays.rate_plans
dining.orders, dining.tables, dining.menus
identity.users, identity.refresh_tokens
business.businesses, business.operations
```
- Single migration runner
- JOINs possible for reporting
- No distributed transactions
- Module boundaries enforced at application layer (NetArchTest catches violations in CI)
- Future: extract schema to separate DB if needed — zero business logic change

### Event Bus → In-memory first, MassTransit-ready
- V1: MediatR in-memory domain events
- V2: MassTransit + RabbitMQ (transport abstracted — swap in 1 day, business logic untouched)

### Background Jobs → Hangfire with PostgreSQL storage
- No extra infrastructure (already have PG)
- Dashboard built-in
- Retry logic built-in
- Night audit scheduling critical

### Logging & Monitoring → Serilog + OpenTelemetry
- OpenTelemetry = vendor neutral OTLP export
- Ship to Grafana / Seq / Application Insights without code change

### Testing Strategy → Unit + Integration + Architecture
- Detail in Part 11

---

## Part 4 — Full Tech Stack (Finalized)

| Layer | Technology | Reason |
|---|---|---|
| Runtime | .NET 10 LTS | 3-year support window |
| Framework | ASP.NET Core 10 | Latest stable |
| Language | C# 14 | Primary constructors, collection expressions |
| ORM (writes) | EF Core 10 | Migrations, change tracking, relationships |
| ORM (reads) | Dapper 2.x | Raw SQL performance for complex queries |
| Database | PostgreSQL 17 | ACID, RLS, JSONB, free, any cloud |
| Cache | Valkey 8 | Redis fork (open-source) — drop-in replacement after Redis license change 2024 |
| Background Jobs | Hangfire 2.x | PG storage, dashboard, retries |
| CQRS | MediatR 12 | Pipeline behaviors, clean dispatch |
| Real-time | SignalR (built-in) | KDS, floor map, room status updates |
| Validation | FluentValidation 12 | Pipeline behavior integration |
| Result Pattern | ErrorOr 2.x | No exceptions for expected business errors |
| Auth | ASP.NET Core Identity (extended) | Not replaced — extended with custom fields |
| JWT | RS256 (asymmetric keys) | Key rotation without re-deploy |
| API Docs | Scalar | Modern Swagger UI replacement |
| Logging | Serilog | Structured JSON logging |
| Observability | OpenTelemetry | Traces + Metrics + Logs (OTLP) |
| Local Dev | .NET Aspire | Orchestrates PG + Redis + App + Dashboard |
| Containers | Docker + Compose | Production parity in dev environment |
| Testing: Unit | xUnit + NSubstitute + FluentAssertions | — |
| Testing: Integration | Testcontainers (real PostgreSQL) | No DB mocking |
| Testing: Architecture | NetArchTest | Module boundary enforcement in CI |
| Fake Data | Bogus | Realistic test data generation |

---

## Part 5 — Solution Structure

```
OneNex.slnx
│
src/
├── Shared.Kernel/                    ← Zero dependencies. True foundation.
│   ├── CQRS/
│   │   ├── ICommand.cs
│   │   ├── ICommandHandler.cs
│   │   ├── IQuery.cs
│   │   └── IQueryHandler.cs
│   ├── ValueObjects/
│   │   ├── Money.cs
│   │   ├── Email.cs
│   │   ├── PhoneNumber.cs
│   │   ├── DateRange.cs
│   │   ├── TimeRange.cs
│   │   ├── Address.cs
│   │   ├── Percentage.cs
│   │   ├── CountryCode.cs
│   │   └── CurrencyCode.cs
│   ├── Exceptions/
│   │   ├── DomainException.cs
│   │   ├── NotFoundException.cs
│   │   ├── ValidationException.cs
│   │   ├── ForbiddenException.cs
│   │   └── ConflictException.cs
│   └── Primitives/
│       ├── Entity.cs
│       ├── AuditableEntity.cs
│       ├── AggregateRoot.cs
│       ├── IDomainEvent.cs
│       ├── ICurrentUser.cs
│       ├── IDateTimeProvider.cs
│       └── PagedList.cs
│
├── Shared.Contracts/                 ← Module-to-module interfaces only
│   └── IntegrationEvents/
│       └── (integration events added as modules grow)
│
├── Shared.Infrastructure/            ← Cross-cutting: behaviors, middleware, auth
│   ├── Api/
│   │   ├── ApiController.cs          ← Base controller with Match() helpers
│   │   └── ApiResponse.cs
│   ├── Auth/
│   │   └── CurrentUser.cs
│   ├── Behaviors/
│   │   ├── LoggingBehavior.cs        ← Logging → Validation → Transaction (pipeline order)
│   │   ├── ValidationBehavior.cs
│   │   └── TransactionBehavior.cs
│   ├── Middleware/                   ← HTTP concerns live here, NOT in WebAPI
│   │   ├── ExceptionHandlingMiddleware.cs
│   │   └── CorrelationIdMiddleware.cs
│   ├── Time/
│   │   └── SystemDateTimeProvider.cs
│   └── DependencyInjection.cs        ← AddSharedInfrastructure(), UseCorrelationId(), UseExceptionHandling()
│
├── Modules/
│   ├── Identity/                     ← Core module
│   │   ├── Identity.Domain/
│   │   ├── Identity.Application/
│   │   │   ├── AssemblyReference.cs  ← Assembly marker for MediatR scan
│   │   │   └── Features/
│   │   │       └── {Feature}/
│   │   │           ├── {Feature}Command.cs / {Feature}Query.cs
│   │   │           ├── {Feature}Handler.cs
│   │   │           ├── {Feature}Validator.cs
│   │   │           └── {Feature}Dto.cs
│   │   ├── Identity.Infrastructure/
│   │   │   └── DependencyInjection.cs  ← AddIdentityModule()
│   │   └── Identity.Presentation/
│   │       ├── Controllers/
│   │       ├── Hubs/
│   │       └── DependencyInjection.cs  ← AddIdentityPresentation()
│   │
│   ├── Business/
│   │   └── (same 4-project structure)
│   │
│   ├── Membership/
│   │   └── (same 4-project structure)
│   │
│   └── Operations/                   ← Operation modules grouped here
│       ├── Stays/
│       │   ├── Stays.Domain/
│       │   ├── Stays.Application/
│       │   ├── Stays.Infrastructure/
│       │   └── Stays.Presentation/
│       │       ├── Controllers/
│       │       └── Hubs/             ← SignalR (room status, housekeeping)
│       └── Dining/
│           └── (same 4-project structure)
│
└── Host/
    ├── OneNex.WebAPI/                ← Entry point. Thin — wires everything, owns nothing.
    │   ├── Program.cs
    │   └── Extensions/
    │       └── ModuleExtensions.cs   ← AddModules() + AddModulePresentations()
    │
    ├── OneNex.AppHost/               ← [Optional] .NET Aspire — local dev orchestrator only
    │   └── Program.cs                ← Starts API + Postgres + Valkey in one command
    │
    └── OneNex.ServiceDefaults/       ← [Optional] Aspire shared defaults
        └── Extensions.cs             ← Health checks (/health, /alive), OpenTelemetry, service discovery

tests/
├── Architecture.Tests/               ← NetArchTest boundary enforcement
└── Modules/
    ├── Identity/
    │   ├── Identity.Unit.Tests/
    │   └── Identity.Integration.Tests/
    ├── Business/
    ├── Membership/
    ├── Stays/
    └── Dining/
```

### Why Middleware lives in Shared.Infrastructure, not WebAPI

```
❌ Blueprint draft (WebAPI/Middleware/) — rejected reason:
   Middleware tied to one host → breaks if second host added
   WebAPI becomes fat with cross-cutting concerns

✅ Actual decision (Shared.Infrastructure/Middleware/):
   ExceptionHandlingMiddleware and CorrelationIdMiddleware are
   HTTP cross-cutting concerns — any host can reuse them
   WebAPI stays thin: Program.cs only wires, owns nothing
```

### Why ObservabilityExtensions was removed from WebAPI

Observability (Serilog, OpenTelemetry, traces, metrics) is handled by
`OneNex.ServiceDefaults/Extensions.cs` via `builder.AddServiceDefaults()`.
No separate ObservabilityExtensions.cs needed in WebAPI.

---

## Part 6 — Design Patterns & Rules (Non-Negotiable)

### Rule 1: Result Pattern — No Business Exceptions

```csharp
// ❌ Wrong — exceptions for expected business scenarios
throw new Exception("Room not available");

// ✅ Correct — ErrorOr result pattern
public async Task<ErrorOr<BookingDto>> Handle(CreateBookingCommand cmd, CancellationToken ct)
{
    var room = await _roomRepo.GetByIdAsync(cmd.RoomId, ct);
    if (room is null)
        return Error.NotFound("Room.NotFound", "Room not found");

    if (!room.IsAvailable(cmd.CheckIn, cmd.CheckOut))
        return Error.Conflict("Room.NotAvailable", "Selected dates not available");

    var booking = Booking.Create(room, cmd.CheckIn, cmd.CheckOut, cmd.GuestId);
    await _bookingRepo.AddAsync(booking, ct);
    return BookingDto.From(booking);
}
```

Exceptions = only for truly unexpected failures (DB down, null where impossible).

### Rule 2: Repository only for Aggregate Roots

```
✅ IBookingRepository    (Booking = Aggregate Root)
✅ IRoomRepository       (Room = Aggregate Root)
❌ ILineItemRepository   (LineItem = child of Order — access via Order only)
```

### Rule 3: Domain Logic in Domain, Not in Handlers

```csharp
// ❌ Wrong — business logic in handler
if (booking.CheckOut - booking.CheckIn < policy.MinimumStay)
    return Error.Validation("Minimum stay not met");

// ✅ Correct — factory method or domain method owns the rule
var result = Booking.Create(room, checkIn, checkOut, guestId, policy);
// Booking.Create validates minimum stay internally — handler doesn't know the rule
```

### Rule 4: Module Isolation — Hard Boundary

```csharp
// ❌ Wrong — Dining module directly using Stays internals
using Stays.Infrastructure.Repositories;
var room = new StaysDbContext().Rooms.Find(id); // crosses boundary

// ✅ Correct — through Shared.Contracts
private readonly IStaysService _staysService; // from Shared.Contracts
var availability = await _staysService.GetAvailabilityAsync(date);
```

NetArchTest in CI: build fails if violated. Automated, not trust-based.

### Rule 5: Value Objects over Primitives

```csharp
// ❌ Primitive obsession — what currency? what precision?
decimal amount, string currency

// ✅ Value Object — self-validating, type-safe, no currency mixing
Money price = new(150.00m, "USD");
Money total = price + tax; // guaranteed same currency
```

### Rule 6: Commands and Queries never mix

```
Command handler: writes to DB, publishes domain events, returns void or created resource ID
Query handler:   reads from DB, NEVER writes anything, returns DTO (never domain entity to caller)
```

---

## Part 7 — MediatR Pipeline Behaviors

**Order matters. Executes top to bottom:**

```
Request
  → LoggingBehavior       (log request name, user, businessId, response time — ALL requests)
  → ValidationBehavior    (FluentValidation — reject before handler if invalid — ALL requests)
  → TransactionBehavior   (wrap in DB transaction — Commands ONLY, not Queries)
  → Handler
Response
```

### Why TransactionBehavior is Commands ONLY

Commands = state changes (multiple DB operations must be atomic).
Queries = reads only (transactions add lock overhead, zero benefit).

See: [Detailed Explanation — Transaction vs Query] (below in this doc, to be added)

### Registration

```csharp
services.AddMediatR(cfg =>
{
    cfg.RegisterServicesFromAssemblies(/* all module assemblies */);
    cfg.AddOpenBehavior(typeof(LoggingBehavior<,>));
    cfg.AddOpenBehavior(typeof(ValidationBehavior<,>));
    cfg.AddOpenBehavior(typeof(TransactionBehavior<,>));
});
```

---

## Part 8 — Auth Pipeline

```
Request arrives
  → JWT RS256 validated (ASP.NET Core built-in middleware)
  → [RequirePermission("stays:reservations:create")] on controller action
  → PermissionAuthorizationHandler (custom IAuthorizationHandler)
  → Check Redis: permissions:{userId}:{businessId}
  → Cache miss: load from DB → serialize → cache (5 min TTL)
  → Check Redis: suspended:{userId}:{businessId}
  → If suspended → 403 immediately
  → Proceed to controller → MediatR dispatch
```

Permission revoked → Redis key deleted → next request reloads. Instant effect, no token reissue needed.

---

## Part 9 — Global Error Handling

One middleware handles all exceptions. RFC 7807 ProblemDetails format:

```
DomainException       → 400 Bad Request
ValidationException   → 422 Unprocessable Entity
NotFoundException     → 404 Not Found
ConflictException     → 409 Conflict
ForbiddenException    → 403 Forbidden
Unhandled Exception   → 500 + CorrelationId (stack trace NEVER exposed)
```

Consistent format across all modules — one place to change, all modules benefit.

---

## Part 10 — Observability: Three Pillars

```
Logs    → Serilog (structured JSON) → OTLP → Grafana / Seq
Traces  → OpenTelemetry auto-instrumentation (EF Core, HttpClient, MediatR, SignalR)
Metrics → OpenTelemetry (request rate, DB query time, cache hit %, queue depth)
```

**Correlation ID** — every request gets a UUID header. Flows through all logs and traces.
Production incident = one ID → full picture across all module logs.

**Structured logging rule:**
```csharp
// ❌ Wrong — string interpolation loses structure
Log.Information($"Booking {bookingId} created for guest {guestId}");

// ✅ Correct — structured properties, queryable in Seq/Grafana
Log.Information("Booking created {BookingId} for guest {GuestId} at {BusinessId}",
    booking.Id, guest.Id, businessId);
```

---

## Part 11 — Testing Strategy

```
          [ E2E — minimal, critical paths only ]
         [ Architecture Tests — module boundaries ]
        [ Integration Tests — real DB per module ]
       [ Unit Tests — handlers, domain, value objects ]
```

| Type | What to test | Tool | Rule |
|---|---|---|---|
| Unit | Handler logic, Domain methods, Value Objects | xUnit + NSubstitute + FluentAssertions | No real DB, no real HTTP |
| Integration | Full API endpoint → real PostgreSQL | Testcontainers + WebApplicationFactory | NO DB mocking |
| Architecture | Module boundary rules, naming conventions | NetArchTest | Runs in CI, fails build on violation |
| E2E | Check-in flow, billing, night audit | Playwright | Phase 2 only |

**Integration test non-negotiable:** Testcontainers spins a real PostgreSQL container per test run.
Mocked DB = false confidence. We got burned. Real DB always.

---

## Part 12 — Scalability Roadmap

```
V1 — Launch:
  Single server + Single PostgreSQL + Single Valkey
  Comfortable: ~5,000 concurrent users
  All modules in one Modular Monolith process

V2 — Growth (when metrics demand, not before):
  PostgreSQL Read Replicas → route heavy reads
  Valkey Cluster → maintain cache hit rate at scale
  CDN → static assets, menu photos, room images
  Comfortable: ~50,000 concurrent users

V3 — Scale (extract only what needs it):
  Notification module → standalone service (high volume, async)
  Channel Manager sync → standalone worker (real-time OTA sync load)
  Core business logic stays in monolith
  Comfortable: ~500,000 concurrent users

V4 — If ever needed:
  Full microservices
  Module boundaries already defined since V1
  Business logic: zero change required
  Pure deployment/infrastructure change
```

**Key rule:** Never extract a module before metrics prove it's necessary.
99% of hospitality SaaS never reach V3. Architect for V3, build V1. Not the other way.

---

## Part 13 — Coding Conventions (Non-Negotiable)

| Rule | Standard |
|---|---|
| Files | One class per file. Always. |
| Class naming | PascalCase |
| Private fields | `_camelCase` with underscore prefix |
| Constants | `SCREAMING_SNAKE_CASE` |
| Async | All IO operations async. Never `.Result` or `.Wait()`. |
| Null safety | Nullable reference types enabled project-wide. No implicit nulls. |
| DTOs / Commands / Queries | `record` types (immutable by design) |
| No magic strings | All permission codes, event names, schema names = typed constants |
| Feature folders | Feature-first organization within Application layer (not layer-first) |

---

## Part 14 — Local Developer Experience (.NET Aspire)

```csharp
// AppHost/Program.cs — ONE command starts everything
var postgres = builder.AddPostgres("onenex-db")
    .WithPgAdmin();
var redis = builder.AddValkey("onenex-cache");
var api = builder.AddProject<Projects.OneNex_WebAPI>("api")
    .WithReference(postgres)
    .WithReference(redis);
```

`dotnet run --project AppHost` starts:
- PostgreSQL with pgAdmin
- Valkey (Redis-compatible)
- OneNex API
- Aspire Dashboard (logs, traces, health checks, metrics)

New developer onboarding: `git clone` → `dotnet run` → working. No manual setup.

---

## Part 15 — Build Order

```
Week 1-2:  Solution skeleton + Shared.Kernel (Entity, ValueObjects, Exceptions)
Week 2-3:  Shared.Contracts (empty interfaces — fill as modules grow)
Week 3-4:  Infrastructure foundation (DbContexts per module, Valkey, Serilog, OTel, Error middleware)
Week 4-5:  MediatR + Pipeline Behaviors (Logging, Validation, Transaction)
Week 5-6:  JWT Auth + Permission pipeline + Redis permission cache
Week 6-7:  Identity Module (users, refresh tokens, token rotation)
Week 7-8:  Business Module (businesses, operations, settings)
Week 8-9:  Membership Module (staff, roles, permissions — most complex)
Week 9+:   Stays Module (C1 Room Setup first → C7 Rate Plans → C5 Guest Profile → ...)
```

Foundation = 6 weeks minimum. Not negotiable. Shortcuts here = rewrites later.

---

## Part 16 — Domain Event Strategy (Decided)

### V1 — In-Memory Channel + Channel Worker

Same pattern as existing Booking project. Simple, proven, fast.

```
Domain Event raised in Aggregate
    → SaveChangesAsync() override collects events
    → BackgroundTaskQueue.QueueAsync() ← in-memory channel push
    → Response returned immediately (~20ms)

[Channel Worker — background thread]
    → MediatR.Publish(domainEvent)
    → Domain Event Handler runs
    → IEventBus.PublishAsync(IntegrationEvent)  ← cross-module
    → Integration Handler → Hangfire.Enqueue()  ← actual work
```

**Partition key = AggregateId** — same aggregate events = sequential. Different aggregates = parallel.

### V2 — Outbox Pattern (Future)

Add when production monitoring shows events being lost.

```
Same flow + one addition:
  SaveChangesAsync() → business data + outbox_messages row (SAME transaction)
  OutboxProcessor (BackgroundService) reads unpublished → MediatR.Publish()
  Mark as published

Migration effort: Add outbox table + processor. Zero handler changes.
```

```sql
CREATE TABLE outbox_messages (
    id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    type         TEXT NOT NULL,
    payload      JSONB NOT NULL,
    created_at   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    published_at TIMESTAMPTZ NULL,
    error        TEXT NULL
);
```

### Why Not Hangfire for Domain Events

```
Hangfire.Enqueue() happens AFTER transaction commit.
If commit succeeds but enqueue fails → event lost. Same problem as in-memory.
Hangfire = for WORK triggered by handlers. Not for event publishing.
```

### Hangfire DB Location (Decided)

```
Hangfire is NOT in main DB.

Main DB (onenex):
  stays.* dining.* identity.* membership.* business.*
  outbox_messages (V2 — same transaction as business data)

Notification DB (onenex_notification):
  notification.*
  hangfire.*     ← Hangfire lives here (notification work isolated)
```

### Response Time — No Impact

```
Request path (user waits):     ~20ms
  DB save + event queue push + Hangfire enqueue (fast DB INSERT)

Background (user doesn't wait):
  Email send, SMS, OTA sync, report generate → Hangfire worker
```

---

## Open Items (To Be Discussed & Added)

- [ ] Outbox Pattern for domain events (guarantees delivery even if handler crashes)
- [ ] Rate limiting strategy (per business, per user, per endpoint)
- [ ] Multi-tenancy: Row-Level Security in PostgreSQL — how to implement
- [ ] API versioning strategy (URL-based vs header-based)
- [ ] Soft delete pattern — how to handle across all modules
- [ ] Audit log pattern — who changed what, when
- [ ] File upload strategy (room photos, menu images) — S3 / Blob storage
- [ ] Localization / i18n at API level
- [ ] Health check endpoints design
- [ ] Deployment architecture (bare metal vs Kubernetes vs PaaS)
