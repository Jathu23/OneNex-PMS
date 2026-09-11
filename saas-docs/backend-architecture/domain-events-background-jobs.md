# OneNex — Domain Events & Background Jobs

> Status: Living Document
> Last updated: 2026-09-09
> Covers: IDomainEvent, AggregateRoot event collection, DomainEventInterceptor, IBackgroundTaskQueue, full dispatch flow

---

## Design Decision — No Outbox

```
Outbox Pattern  → events saved to DB table in same transaction → Hangfire polls → complex
Our Approach    → IBackgroundTaskQueue (in-memory) → simpler, sufficient for V1
```

**Trade-off accepted:**
- App crash between SaveChanges and queue dispatch → event lost (email not sent, OTA not notified)
- Business data (booking, room) already committed → safe
- Side effects retry manually if needed
- V1 scale does not justify Outbox complexity

---

## Building Blocks

### IDomainEvent — Shared.Kernel

```csharp
// Shared.Kernel/Domain/IDomainEvent.cs
namespace OneNex.Shared.Kernel.Domain;

/// <summary>
/// Marker interface for all domain events.
/// Extends MediatR INotification — published via IPublisher after SaveChanges.
/// </summary>
public interface IDomainEvent : INotification
{
    Guid     EventId    { get; }
    DateTime OccurredOn { get; }
    Guid     AggregateId { get; }
}
```

### AggregateRoot — Owns the Event List

```csharp
// Shared.Kernel/Domain/AggregateRoot.cs
public abstract class AggregateRoot<TId> : AuditableEntity<TId>
    where TId : notnull
{
    private readonly List<IDomainEvent> _domainEvents = [];

    /// <summary>Read-only view of pending domain events.</summary>
    public IReadOnlyList<IDomainEvent> DomainEvents => _domainEvents.AsReadOnly();

    /// <summary>
    /// Call inside domain methods to signal something happened.
    /// Events are NOT published here — stored until after SaveChanges.
    /// </summary>
    protected void RaiseDomainEvent(IDomainEvent domainEvent)
        => _domainEvents.Add(domainEvent);

    /// <summary>Called by DomainEventInterceptor after successful SaveChanges.</summary>
    public void ClearDomainEvents()
        => _domainEvents.Clear();
}
```

### Domain Event — Example

```csharp
// Stays.Domain/Events/BookingCreatedEvent.cs
public sealed record BookingCreatedEvent(
    Guid BookingId,
    Guid BusinessId,
    Guid GuestId,
    DateRange StayPeriod
) : IDomainEvent
{
    public Guid     EventId     { get; } = Guid.NewGuid();
    public DateTime OccurredOn  { get; } = DateTime.UtcNow;  // record init — ok here
    public Guid     AggregateId { get; } = BookingId;
}
```

### Entity Raises Event Internally

```csharp
// Stays.Domain/Entities/Booking.cs
public sealed class Booking : AggregateRoot<BookingId>
{
    public static Booking Create(
        BusinessId businessId,
        GuestId    guestId,
        DateRange  stayPeriod,
        Money      totalAmount)
    {
        var booking = new Booking
        {
            Id         = BookingId.New(),
            BusinessId = businessId,
            GuestId    = guestId,
            StayPeriod = stayPeriod,
            Status     = BookingStatus.Pending,
        };

        // Raised inside domain method — never from handler
        booking.RaiseDomainEvent(new BookingCreatedEvent(
            booking.Id.Value, businessId.Value, guestId.Value, stayPeriod));

        return booking;
    }

    public void Confirm()
    {
        if (Status != BookingStatus.Pending)
            throw new DomainException("Only pending bookings can be confirmed.");

        Status = BookingStatus.Confirmed;
        RaiseDomainEvent(new BookingConfirmedEvent(Id.Value, BusinessId.Value, GuestId.Value));
    }

    public void Cancel(string reason)
    {
        if (Status == BookingStatus.Cancelled)
            throw new DomainException("Booking already cancelled.");

        Status = BookingStatus.Cancelled;
        RaiseDomainEvent(new BookingCancelledEvent(Id.Value, BusinessId.Value, GuestId.Value, reason));
    }
}
```

Handler calls `booking.Cancel(reason)` — doesn't know what events were raised inside. Domain's concern.

---

## DomainEventInterceptor — Dispatches After Commit

```csharp
// Stays.Infrastructure/Interceptors/DomainEventInterceptor.cs
public sealed class DomainEventInterceptor(
    IBackgroundTaskQueue queue,
    IServiceScopeFactory scopeFactory,
    ILogger<DomainEventInterceptor> logger)
    : SaveChangesInterceptor
{
    // SavedChangesAsync — AFTER commit (not before)
    // Business data already safe in DB at this point
    public override async ValueTask<int> SavedChangesAsync(
        SaveChangesCompletedEventData eventData,
        int result,
        CancellationToken ct = default)
    {
        var aggregates = eventData.Context!.ChangeTracker
            .Entries<AggregateRoot<Guid>>()
            .Where(e => e.Entity.DomainEvents.Any())
            .Select(e => e.Entity)
            .ToList();

        var events = aggregates
            .SelectMany(a => a.DomainEvents)
            .ToList();

        // Clear from aggregates before dispatching — no double fire
        aggregates.ForEach(a => a.ClearDomainEvents());

        // Queue each event as background task
        // partitionKey = AggregateId → same aggregate events run sequentially
        foreach (var domainEvent in events)
        {
            var captured = domainEvent;

            await queue.QueueAsync(async taskCt =>
            {
                await using var scope = scopeFactory.CreateAsyncScope();
                var publisher = scope.ServiceProvider.GetRequiredService<IPublisher>();

                try
                {
                    await publisher.Publish(captured, taskCt);
                }
                catch (Exception ex)
                {
                    logger.LogError(ex,
                        "Domain event processing failed. {EventType} {AggregateId}",
                        captured.GetType().Name, captured.AggregateId);
                }
            }, partitionKey: captured.AggregateId.ToString());
        }

        return await base.SavedChangesAsync(eventData, result, ct);
    }
}
```

**Why `SavedChangesAsync` (after commit)?**

```
SavingChangesAsync  → before commit  → Outbox approach (more reliable, more complex)
SavedChangesAsync   → after commit   → our approach (simpler, accepted trade-off for V1)
```

**Why `partitionKey = AggregateId`?**

```
Same booking → BookingCreatedEvent + BookingConfirmedEvent
partitionKey = bookingId → runs sequentially, no race condition
Different bookings → run in parallel
```

---

## Domain Event Handlers

```csharp
// Stays.Application/Events/BookingCreatedEventHandler.cs
public sealed class BookingCreatedEmailHandler(
    INotificationService notificationService)
    : INotificationHandler<BookingCreatedEvent>
{
    public async Task Handle(
        BookingCreatedEvent notification, CancellationToken ct)
    {
        await notificationService.SendBookingConfirmationAsync(
            notification.GuestId,
            notification.BookingId,
            notification.StayPeriod,
            ct);
    }
}

// Multiple handlers for same event — all run independently
public sealed class BookingCreatedOTASyncHandler(
    IOtaSyncService otaSync)
    : INotificationHandler<BookingCreatedEvent>
{
    public async Task Handle(
        BookingCreatedEvent notification, CancellationToken ct)
    {
        await otaSync.UpdateAvailabilityAsync(
            notification.BusinessId,
            notification.StayPeriod,
            ct);
    }
}
```

MediatR publishes to ALL registered handlers for that event. Each handler = one side effect. Failure in one doesn't affect others.

---

## Full Flow — End to End

```
POST /api/v1/stays/bookings
        │
        ▼
CreateBookingCommandHandler
  booking = Booking.Create(...)    ← RaiseDomainEvent(BookingCreatedEvent) inside
  room.MarkAsOccupied(period)
  await _context.SaveChangesAsync()
        │
        ├── INSERT stays.bookings   ✅
        └── UPDATE stays.rooms     ✅  ← ONE transaction committed
        │
        ▼
DomainEventInterceptor.SavedChangesAsync()  ← fires AFTER commit
  → booking.DomainEvents → [BookingCreatedEvent]
  → booking.ClearDomainEvents()
  → queue.QueueAsync(publish BookingCreatedEvent, partitionKey: bookingId)
        │
        ▼
201 Created → client gets fast response ✅

        │
        ▼ (background — immediately, same process)
IBackgroundTaskQueue processes:
  → IPublisher.Publish(BookingCreatedEvent)
        ├── BookingCreatedEmailHandler  → confirmation email
        ├── BookingCreatedOTASyncHandler → Airbnb / Booking.com
        └── BookingCreatedAnalyticsHandler → dashboard update

Handler failure:
  → LogError → continues
  → booking + room status already safe in DB ✅
  → side effect lost — acceptable V1 trade-off
```

---

## Where Files Live

```
Shared.Kernel/
  └── Domain/
      └── IDomainEvent.cs              ← marker interface

Stays.Infrastructure/
  └── Interceptors/
      ├── DomainEventInterceptor.cs    ← dispatches after SaveChanges
      └── AuditInterceptor.cs          ← sets CreatedAt, UpdatedAt

Stays.Domain/
  └── Events/
      ├── BookingCreatedEvent.cs
      ├── BookingConfirmedEvent.cs
      └── BookingCancelledEvent.cs

Stays.Application/
  └── Events/
      ├── BookingCreatedEmailHandler.cs
      ├── BookingCreatedOTASyncHandler.cs
      └── BookingCancelledRefundHandler.cs
```

---

## Rules

```
1.  IDomainEvent        → Shared.Kernel/Domain. Implements INotification.
2.  RaiseDomainEvent()  → protected, called only inside entity/aggregate methods. Never handler.
3.  ClearDomainEvents() → called only by DomainEventInterceptor. Never manually.
4.  SavedChangesAsync   → after commit. Business data safe before event dispatch.
5.  IBackgroundTaskQueue → in-memory. Same pattern as existing project.
6.  partitionKey        → AggregateId. Same aggregate = sequential. Different = parallel.
7.  Handler failure     → LogError, continue. Side effect lost — V1 trade-off accepted.
8.  One side effect     → one handler class. Never multiple concerns in one handler.
9.  Events folder       → Stays.Domain/Events/ (definition). Stays.Application/Events/ (handlers).
10. No Outbox           → V1 decision. Revisit if reliability requirements increase.
```
