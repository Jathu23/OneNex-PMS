# OneNex — CQRS Markers & Pipeline Patterns

> Status: Living Document
> Last updated: 2026-09-08
> Covers: ICommand, IQuery, ICommandHandler, IQueryHandler, TransactionBehavior, Transaction vs Domain Event boundary

---

## CQRS Markers — Why Needed

MediatR default-ஆ `IRequest<T>` use பண்ணும். Command vs Query distinguish பண்ண முடியாது.

```csharp
// Without markers — pipeline behavior Command vs Query தெரியாது
public record CancelBookingCommand : IRequest<ErrorOr<Success>> { }
public record GetBookingQuery      : IRequest<ErrorOr<BookingDetailDto>> { }

// TransactionBehavior — இரண்டுக்கும் run ஆகும் — wrong
// Query-க்கு transaction = unnecessary DB lock + overhead
```

Solution: marker interfaces — Command vs Query compile-time-ல் separate ஆகும்.

---

## Marker Interfaces — Shared.Kernel

```csharp
// Shared.Kernel/CQRS/ICommand.cs
namespace OneNex.Shared.Kernel.CQRS;

/// <summary>
/// Marker for commands that return no data (cancel, delete, confirm).
/// TransactionBehavior wraps all ICommand handlers automatically.
/// ErrorOr<Success> already baked in — no repeat needed per handler.
/// </summary>
public interface ICommand : IRequest<ErrorOr<Success>> { }

/// <summary>
/// Marker for commands that return data (create → returns new ID).
/// TransactionBehavior wraps all ICommand<T> handlers automatically.
/// </summary>
public interface ICommand<TResponse> : IRequest<ErrorOr<TResponse>> { }

// Shared.Kernel/CQRS/IQuery.cs
/// <summary>
/// Marker for queries — read only, no DB write.
/// TransactionBehavior SKIPS IQuery handlers — no transaction overhead.
/// </summary>
public interface IQuery<TResponse> : IRequest<ErrorOr<TResponse>> { }
```

---

## Handler Interfaces — Shared.Kernel

```csharp
// Shared.Kernel/CQRS/ICommandHandler.cs
namespace OneNex.Shared.Kernel.CQRS;

/// <summary>For commands with no return data (cancel, delete, confirm).</summary>
public interface ICommandHandler<TCommand>
    : IRequestHandler<TCommand, ErrorOr<Success>>
    where TCommand : ICommand { }

/// <summary>For commands that return data (create → BookingId).</summary>
public interface ICommandHandler<TCommand, TResponse>
    : IRequestHandler<TCommand, ErrorOr<TResponse>>
    where TCommand : ICommand<TResponse> { }

// Shared.Kernel/CQRS/IQueryHandler.cs
/// <summary>For all queries — read only.</summary>
public interface IQueryHandler<TQuery, TResponse>
    : IRequestHandler<TQuery, ErrorOr<TResponse>>
    where TQuery : IQuery<TResponse> { }
```

---

## Usage — Command & Query Definitions

```csharp
// ── Commands ──────────────────────────────────────────────────

// No return data
public sealed record CancelBookingCommand(
    BookingId BookingId,
    string    Reason) : ICommand;

public sealed record DeleteBookingCommand(
    BookingId BookingId) : ICommand;

public sealed record ConfirmBookingCommand(
    BookingId BookingId) : ICommand;

// Returns data
public sealed record CreateBookingCommand(
    RoomId    RoomId,
    GuestId   GuestId,
    DateRange StayPeriod,
    string?   SpecialRequests) : ICommand<BookingId>;   // ← returns BookingId

// ── Queries ───────────────────────────────────────────────────

public sealed record GetBookingQuery(
    BookingId BookingId) : IQuery<BookingDetailDto>;

public sealed record ListBookingsQuery(
    Guid    BusinessId,
    string? Status     = null,
    int     PageNumber = 1,
    int     PageSize   = 20) : IQuery<PagedList<BookingListItemDto>>;
```

---

## Usage — Handler Definitions

```csharp
// Command — no return
public sealed class CancelBookingCommandHandler(
    IBookingWriteRepository writeRepo,
    ICurrentUser currentUser,
    IDateTimeProvider clock)
    : ICommandHandler<CancelBookingCommand>   // ← clean, ErrorOr<Success> implied
{
    public async Task<ErrorOr<Success>> Handle(
        CancelBookingCommand command, CancellationToken ct)
    {
        var booking = await writeRepo.FindByIdAsync(command.BookingId, ct);
        if (booking is null)
            return Errors.Booking.NotFound(command.BookingId.Value);

        if (booking.BusinessId != currentUser.BusinessId)
            return Error.Forbidden();

        booking.Cancel(command.Reason, clock.UtcNow);
        await writeRepo.SaveChangesAsync(ct);

        return Result.Success;
    }
}

// Command — returns data
public sealed class CreateBookingCommandHandler(
    IBookingWriteRepository writeRepo,
    IRoomWriteRepository roomWriteRepo,
    ICurrentUser currentUser,
    IDateTimeProvider clock)
    : ICommandHandler<CreateBookingCommand, BookingId>   // ← returns BookingId
{
    public async Task<ErrorOr<BookingId>> Handle(
        CreateBookingCommand command, CancellationToken ct)
    {
        var room = await roomWriteRepo.GetByIdAsync(command.RoomId, ct);

        if (!room.IsAvailableFor(command.StayPeriod))
            return Errors.Room.NotAvailable;

        var booking = Booking.Create(
            currentUser.BusinessId,
            command.GuestId,
            command.StayPeriod);

        // Room status — SAME transaction (invariant — see section below)
        room.MarkAsOccupied(command.StayPeriod);

        await writeRepo.AddAsync(booking, ct);
        await writeRepo.SaveChangesAsync(ct);
        // ↑ SaveChanges:
        //   INSERT booking + UPDATE room — one transaction
        //   DomainEventInterceptor → BookingCreatedEvent → background queue

        return booking.Id;
    }
}

// Query
public sealed class GetBookingQueryHandler(
    IBookingReadRepository readRepo,
    ICurrentUser currentUser)
    : IQueryHandler<GetBookingQuery, BookingDetailDto>
{
    public async Task<ErrorOr<BookingDetailDto>> Handle(
        GetBookingQuery query, CancellationToken ct)
    {
        var booking = await readRepo.FindByIdAsync(query.BookingId, ct);
        if (booking is null)
            return Errors.Booking.NotFound(query.BookingId.Value);

        return booking;
    }
}
```

---

## Pipeline Behaviors

MediatR pipeline — behaviors run in registration order:

```
Request
  │
  ▼
LoggingBehavior       → log request + response time (ALL requests)
  │
  ▼
ValidationBehavior    → FluentValidation (ALL requests)
  │
  ▼
TransactionBehavior   → DB transaction (ICommand ONLY — IQuery skips)
  │
  ▼
Handler
```

### LoggingBehavior

```csharp
// Shared.Infrastructure/Behaviors/LoggingBehavior.cs
public sealed class LoggingBehavior<TRequest, TResponse>(
    ILogger<LoggingBehavior<TRequest, TResponse>> logger,
    ICurrentUser currentUser)
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : IBaseRequest
{
    public async Task<TResponse> Handle(
        TRequest request,
        RequestHandlerDelegate<TResponse> next,
        CancellationToken ct)
    {
        var requestName = typeof(TRequest).Name;

        logger.LogInformation(
            "Handling {RequestName} | UserId: {UserId} | BusinessId: {BusinessId}",
            requestName,
            currentUser.IsAuthenticated ? currentUser.UserId : "Anonymous",
            currentUser.IsAuthenticated ? currentUser.BusinessId : "None");

        var stopwatch = Stopwatch.StartNew();

        var response = await next();

        stopwatch.Stop();

        logger.LogInformation(
            "Handled {RequestName} in {ElapsedMs}ms",
            requestName,
            stopwatch.ElapsedMilliseconds);

        return response;
    }
}
```

### ValidationBehavior

```csharp
// Shared.Infrastructure/Behaviors/ValidationBehavior.cs
public sealed class ValidationBehavior<TRequest, TResponse>(
    IEnumerable<IValidator<TRequest>> validators)
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : IBaseRequest
{
    public async Task<TResponse> Handle(
        TRequest request,
        RequestHandlerDelegate<TResponse> next,
        CancellationToken ct)
    {
        if (!validators.Any())
            return await next();

        var context  = new ValidationContext<TRequest>(request);
        var results  = await Task.WhenAll(
            validators.Select(v => v.ValidateAsync(context, ct)));

        var failures = results
            .SelectMany(r => r.Errors)
            .Where(f => f is not null)
            .ToList();

        if (failures.Count == 0)
            return await next();

        var errors = failures
            .GroupBy(f => f.PropertyName)
            .ToDictionary(
                g => g.Key,
                g => g.Select(e => e.ErrorMessage).ToArray());

        throw new ValidationException(errors);
    }
}
```

### TransactionBehavior — ICommand Only

```csharp
// Shared.Infrastructure/Behaviors/TransactionBehavior.cs
public sealed class TransactionBehavior<TRequest, TResponse>(
    IDbContextFactory dbContextFactory)
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : ICommand   // ← IQuery வந்தா இந்த behavior-ஐ MediatR consider பண்ணாது
{
    public async Task<TResponse> Handle(
        TRequest request,
        RequestHandlerDelegate<TResponse> next,
        CancellationToken ct)
    {
        await using var transaction = await dbContextFactory
            .CreateDbContext()
            .Database
            .BeginTransactionAsync(ct);
        try
        {
            var response = await next();      // handler runs

            await transaction.CommitAsync(ct);  // success → save all
            return response;
        }
        catch
        {
            await transaction.RollbackAsync(ct); // fail → undo all
            throw;
        }
    }
}
```

`where TRequest : ICommand` — MediatR இந்த constraint-ஐ பாத்து IQuery request-க்கு இந்த behavior-ஐ pipeline-ல் add பண்ணாது. Zero overhead.

---

## Transaction vs Domain Event — The Boundary

### The One Question

> **"இது fail ஆனா, business data inconsistent ஆகுமா?"**

```
YES → Same transaction-ல் வை   (invariant — must be consistent)
NO  → Domain event-ல் வை       (side effect — eventually consistent OK)
```

---

### Booking Create — What Goes Where

```csharp
// CreateBookingCommandHandler — Transaction-ல்:
var booking = Booking.Create(...);
room.MarkAsOccupied(command.StayPeriod);   // ← Transaction-ல் (invariant)

await _context.SaveChangesAsync(ct);
// ONE transaction:
//   INSERT stays.bookings          ✅
//   UPDATE stays.rooms (Occupied)  ✅ — both or nothing
//
// AFTER save, DomainEventInterceptor:
//   BookingCreatedEvent → background queue → side effects below
```

```
BookingCreatedEvent (domain event — background):
  ✅ Send confirmation email     — fail ஆனாலும் booking + room status valid
  ✅ Notify OTA (Airbnb)         — fail ஆனாலும் booking + room status valid
  ✅ Update analytics dashboard  — fail ஆனாலும் booking + room status valid
  ✅ Trigger housekeeping task   — fail ஆனாலும் booking + room status valid
```

---

### Decision Table — Transaction or Domain Event?

| Action | Transaction | Domain Event | Reason |
|---|---|---|---|
| Room status → Occupied | ✅ | ❌ | Fail = double booking |
| Booking record create | ✅ | ❌ | Core data |
| Inventory deduct | ✅ | ❌ | Fail = oversell |
| Folio/billing record | ✅ | ❌ | Fail = financial inconsistency |
| Confirmation email | ❌ | ✅ | Fail = inconvenience, not data loss |
| OTA sync | ❌ | ✅ | Fail = retry பண்ணலாம் |
| Staff notification | ❌ | ✅ | Fail = acceptable |
| Analytics update | ❌ | ✅ | Fail = acceptable |
| Housekeeping schedule | ❌ | ✅ | Fail = acceptable |

---

### Visual Flow — What Happens in DB

```
CreateBookingCommand
        │
        ▼
TransactionBehavior → BEGIN TRANSACTION
        │
        ▼
Handler:
  room.IsAvailableFor(period)?   → No  → return Error.Conflict (ROLLBACK auto)
                                   Yes → continue
  Booking.Create(...)
  room.MarkAsOccupied(period)
  SaveChangesAsync()
    ├── AuditInterceptor    → CreatedAt, CreatedBy set
    ├── INSERT stays.bookings
    ├── UPDATE stays.rooms SET status = 'Occupied'
    └── DomainEventInterceptor → BookingCreatedEvent → BackgroundQueue
        │
        ▼
TransactionBehavior → COMMIT
        │
        ▼
Client ← 201 Created { bookingId: "..." }  ← fast response ✅

        │
        ▼ (background — after response sent)
BackgroundQueue → BookingCreatedEvent
  → EmailHandler      → confirmation email
  → OTASyncHandler    → Airbnb / Booking.com update
  → AnalyticsHandler  → dashboard update

If background handlers fail:
  → Booking + room status already committed ✅ — business data safe
  → Retry logic / manual intervention for side effects
```

---

## Registration — Application DI

```csharp
// Stays.Application/DependencyInjection.cs
services.AddMediatR(cfg =>
{
    cfg.RegisterServicesFromAssembly(typeof(DependencyInjection).Assembly);

    // Order matters — top runs first
    cfg.AddBehavior(typeof(IPipelineBehavior<,>), typeof(LoggingBehavior<,>));
    cfg.AddBehavior(typeof(IPipelineBehavior<,>), typeof(ValidationBehavior<,>));
    cfg.AddBehavior(typeof(IPipelineBehavior<,>), typeof(TransactionBehavior<,>));
    //              TransactionBehavior — only matches ICommand (constraint)
    //              IQuery requests → this behavior not added to pipeline
});
```

---

## Rules

```
1. ICommand       → no return data (cancel, delete, confirm)
2. ICommand<T>    → returns data (create → ID, update → updated DTO)
3. IQuery<T>      → read only, always returns data
4. ICommandHandler / IQueryHandler → always sealed
5. ErrorOr already baked into markers — never repeat per handler
6. TransactionBehavior → ICommand constraint — IQuery gets zero overhead
7. Transaction    → business invariants (room status, inventory, billing)
8. Domain Event   → side effects (email, OTA sync, analytics, notifications)
9. The test       → "Fail ஆனா business data inconsistent ஆகுமா?" YES=transaction, NO=event
```
