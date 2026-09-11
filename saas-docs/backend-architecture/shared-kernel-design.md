# OneNex — Shared.Kernel Design

> Status: Living Document
> Last updated: 2026-09-06
> Zero external dependencies. Every module builds on this.

---

## What is Shared.Kernel?

Building blocks every module uses. No business logic here — only foundations.

```
Shared.Kernel/
├── Ids/
│   └── IEntityId.cs            ← typed ID marker interface
├── Domain/
│   ├── Entity.cs
│   ├── AggregateRoot.cs
│   ├── IDomainEvent.cs
│   └── ValueObject.cs
├── ValueObjects/
│   ├── Money.cs
│   ├── Email.cs
│   ├── PhoneNumber.cs
│   ├── DateRange.cs
│   ├── TimeRange.cs
│   ├── Address.cs
│   └── Percentage.cs
├── Exceptions/
│   ├── DomainException.cs
│   ├── NotFoundException.cs
│   ├── ValidationException.cs
│   ├── ConflictException.cs
│   └── ForbiddenException.cs
└── Primitives/
    ├── AuditableEntity.cs
    ├── IDateTimeProvider.cs    ← system clock abstraction (testable)
    ├── ICurrentUser.cs         ← current user abstraction
    ├── IWriteRepository.cs     ← marker for Scrutor auto-scan (EF Core repos)
    ├── IReadRepository.cs      ← marker for Scrutor auto-scan (Dapper repos)
    ├── IDomainService.cs       ← marker for Scrutor auto-scan (domain services)
    └── PagedList.cs            ← pagination wrapper
```

---

## Typed IDs — Why & How

### The Problem Without Typed IDs

```csharp
// Plain Guid everywhere:
void AssignRoom(Guid bookingId, Guid roomId) { }

// Developer accidentally swaps:
AssignRoom(room.Id, booking.Id);  // ✅ compiles  ❌ wrong — silent bug
```

Compiler catch panna mudiyalai. Runtime-la wrong data — find panna romba kashtam.

### Solution — Each ID Gets Its Own Type

```csharp
public readonly record struct BookingId(Guid Value);
public readonly record struct RoomId(Guid Value);

void AssignRoom(BookingId bookingId, RoomId roomId) { }

AssignRoom(room.Id, booking.Id);  // ❌ COMPILE ERROR — caught immediately ✅
```

### Why record struct (Not ValueObject)

```
ValueObject (class) = heap allocation, GC overhead, heavy boilerplate
record struct       = stack allocation (like int/Guid), zero overhead, 1 line

ID-ku ValueObject = overkill.
record struct = type safety with zero cost.
```

### IEntityId — Marker Interface

```csharp
// Shared.Kernel/Ids/IEntityId.cs
namespace OneNex.Shared.Kernel.Ids;

public interface IEntityId<T> where T : struct
{
    T Value { get; }
}
```

### How Each Module Defines Its IDs

```csharp
// Stays.Domain/Ids/BookingId.cs
public readonly record struct BookingId(Guid Value) : IEntityId<Guid>
{
    public static BookingId New()  => new(Guid.NewGuid());
    public static BookingId Empty => new(Guid.Empty);

    // Guid-a direct use panna venum endral implicit conversion:
    public static implicit operator Guid(BookingId id) => id.Value;
    public static implicit operator BookingId(Guid id) => new(id);

    public override string ToString() => Value.ToString();
}

public readonly record struct RoomId(Guid Value) : IEntityId<Guid>
{
    public static RoomId New() => new(Guid.NewGuid());
    public static implicit operator Guid(RoomId id) => id.Value;
    public static implicit operator RoomId(Guid id) => new(id);
}

public readonly record struct GuestId(Guid Value) : IEntityId<Guid>
{
    public static GuestId New() => new(Guid.NewGuid());
    public static implicit operator Guid(GuestId id) => id.Value;
    public static implicit operator GuestId(Guid id) => new(id);
}
```

### EF Core — Value Converter

Application-la `BookingId`. DB-la `Guid`. EF Core bridge pannudum.

```csharp
// Infrastructure/Converters/EntityIdValueConverter.cs
public sealed class EntityIdValueConverter<TId> : ValueConverter<TId, Guid>
    where TId : struct, IEntityId<Guid>
{
    public EntityIdValueConverter()
        : base(
            id    => id.Value,                                         // App → DB
            value => (TId)Activator.CreateInstance(typeof(TId), value)! // DB → App
        ) { }
}

// Each module's DbContext OnModelCreating:
protected override void ConfigureConventions(ModelConfigurationBuilder config)
{
    config.Properties<BookingId>().HaveConversion<EntityIdValueConverter<BookingId>>();
    config.Properties<RoomId>().HaveConversion<EntityIdValueConverter<RoomId>>();
    config.Properties<GuestId>().HaveConversion<EntityIdValueConverter<GuestId>>();
}
```

---

## Domain — Base Classes

### IDomainEvent

```csharp
namespace OneNex.Shared.Kernel.Domain;

/// <summary>
/// Marker interface for all domain events.
/// Extends MediatR INotification — published via IPublisher after SaveChanges.
/// </summary>
public interface IDomainEvent : INotification
{
    Guid     EventId     { get; }
    DateTime OccurredOn  { get; }
    Guid     AggregateId { get; }   // used as partition key for ordered processing
}
```

---

### Entity

```csharp
namespace OneNex.Shared.Kernel.Domain;

/// <summary>
/// Base class for all domain entities.
/// Identity by ID — two entities with same ID = same entity (regardless of property values).
/// </summary>
public abstract class Entity<TId> : IEquatable<Entity<TId>>
    where TId : notnull
{
    public TId Id { get; protected init; } = default!;

    protected Entity() { }

    protected Entity(TId id)
    {
        Id = id;
    }

    // Equality by ID only
    public bool Equals(Entity<TId>? other)
    {
        if (other is null) return false;
        if (ReferenceEquals(this, other)) return true;
        if (other.GetType() != GetType()) return false;
        return EqualityComparer<TId>.Default.Equals(Id, other.Id);
    }

    public override bool Equals(object? obj) => Equals(obj as Entity<TId>);

    public override int GetHashCode() => EqualityComparer<TId>.Default.GetHashCode(Id);

    public static bool operator ==(Entity<TId>? left, Entity<TId>? right)
        => left is null ? right is null : left.Equals(right);

    public static bool operator !=(Entity<TId>? left, Entity<TId>? right)
        => !(left == right);
}
```

---

### AuditableEntity

```csharp
namespace OneNex.Shared.Kernel.Primitives;

/// <summary>
/// Adds created/updated audit timestamps.
/// Set automatically by EF Core SaveChanges interceptor — never set manually.
/// </summary>
public abstract class AuditableEntity<TId> : Entity<TId>
    where TId : notnull
{
    public DateTime CreatedAt  { get; private set; }
    public DateTime UpdatedAt  { get; private set; }
    public Guid?    CreatedBy  { get; private set; }   // userId
    public Guid?    UpdatedBy  { get; private set; }   // userId

    // Called by AuditInterceptor — not by business code
    internal void SetCreated(DateTime at, Guid? by) { CreatedAt = at; CreatedBy = by; }
    internal void SetUpdated(DateTime at, Guid? by) { UpdatedAt = at; UpdatedBy = by; }
}
```

---

### AggregateRoot

```csharp
namespace OneNex.Shared.Kernel.Domain;

/// <summary>
/// Base class for Aggregate Roots.
/// Owns a collection of domain events — raised during business operations,
/// published after SaveChanges via DomainEventInterceptor.
///
/// Rule: Only create repositories for AggregateRoot types.
///       Never for child entities.
/// </summary>
public abstract class AggregateRoot<TId> : AuditableEntity<TId>
    where TId : notnull
{
    private readonly List<IDomainEvent> _domainEvents = [];

    /// <summary>Read-only view of pending domain events.</summary>
    public IReadOnlyList<IDomainEvent> DomainEvents => _domainEvents.AsReadOnly();

    /// <summary>
    /// Call this inside domain methods to signal something happened.
    /// Events are NOT published here — stored until SaveChanges.
    /// </summary>
    protected void RaiseDomainEvent(IDomainEvent domainEvent)
        => _domainEvents.Add(domainEvent);

    /// <summary>Called by DomainEventInterceptor after successful SaveChanges.</summary>
    public void ClearDomainEvents()
        => _domainEvents.Clear();
}
```

---

## Aggregate Root — Concept & Rules

### Aggregate = Consistency Boundary

ஒரு system-ல் entities direct-ஆ interact பண்ணா rules break ஆகும். Aggregate = ஒரு group of objects that must change together as one unit.

**Example — Booking:**
- Room add பண்ணா → Booking total update ஆகணும்
- Guest change ஆனா → Booking அதை reflect பண்ணணும்
- இந்த changes எல்லாம் **ஒரே transaction-ல்** நடக்கணும் — இல்லன்னா data inconsistent ஆகிடும்

அந்த group of related objects = **Aggregate**. அந்த group-ஓட "boss" entity = **Aggregate Root**.

---

### OneNex Example — Booking Aggregate

```
Booking             ← Aggregate Root (all access goes through here)
  ├── BookingRoom[]     ← child entity (which room, which nights, rate)
  ├── BookingGuest[]    ← child entity (primary + additional guests)
  └── BookingAddOn[]    ← child entity (breakfast, airport transfer etc.)
```

Outside world = `Booking` மட்டும் தெரியும். `BookingRoom` யாரும் direct-ஆ touch பண்ண மாட்டாங்க.

---

### The 3 Rules

**Rule 1: Only AggregateRoot gets a Repository**

```
IBookingWriteRepository    ✅  Booking = AggregateRoot
IBookingRoomRepository     ❌  BookingRoom = child entity, no direct repo
IBookingGuestRepository    ❌  BookingGuest = child entity, no direct repo
```

**Rule 2: Children modified ONLY through the Root**

```csharp
// ❌ WRONG — bypasses aggregate, breaks consistency
await _roomRepo.AddAsync(new BookingRoom(bookingId, roomId, ...));

// ✅ CORRECT — through the aggregate root
var booking = await _writeRepo.GetByIdAsync(bookingId, ct);
booking.AddRoom(roomId, period, rate);   // domain method on Booking
await _context.SaveChangesAsync(ct);
```

**Rule 3: Load full aggregate when fetching for modification**

```csharp
// GetByIdAsync in Write Repo — always includes all children
var booking = await _context.Bookings
    .Include(b => b.Rooms)
    .Include(b => b.Guests)
    .Include(b => b.AddOns)
    .FirstOrDefaultAsync(b => b.Id == id, ct);
```

Why? EF change tracking needs to see all children to detect what changed. Include இல்லன்னா child-ஓட changes EF கு தெரியாது.

---

### How AddRoom Works Inside the Aggregate

```csharp
// Stays.Domain/Entities/Booking.cs
public sealed class Booking : AggregateRoot<BookingId>
{
    private readonly List<BookingRoom> _rooms = [];

    // Expose as read-only — outside cannot add directly
    public IReadOnlyList<BookingRoom> Rooms => _rooms.AsReadOnly();

    public void AddRoom(RoomId roomId, DateRange period, Money rate)
    {
        // Business rule check inside aggregate
        if (_rooms.Any(r => r.RoomId == roomId))
            throw new DomainException("Room is already added to this booking.");

        // Create child through internal factory — not public constructor
        var bookingRoom = BookingRoom.Create(Id, roomId, period, rate);
        _rooms.Add(bookingRoom);

        // Domain event raised inside domain method
        RaiseDomainEvent(new BookingRoomAddedEvent(Id, roomId));
    }

    public void RemoveRoom(RoomId roomId)
    {
        var room = _rooms.FirstOrDefault(r => r.RoomId == roomId)
            ?? throw new NotFoundException(nameof(BookingRoom), roomId.Value);

        _rooms.Remove(room);
        RaiseDomainEvent(new BookingRoomRemovedEvent(Id, roomId));
    }
}
```

---

### What a Child Entity Looks Like

```csharp
// Stays.Domain/Entities/BookingRoom.cs
// NOT AggregateRoot — no domain events, no repository
public sealed class BookingRoom : Entity<BookingRoomId>
{
    public BookingId BookingId { get; private set; }   // FK to parent
    public RoomId    RoomId    { get; private set; }
    public DateRange Period    { get; private set; } = null!;
    public Money     Rate      { get; private set; } = null!;

    private BookingRoom() { }   // EF Core

    // internal — only Booking (same assembly) can create BookingRoom
    // Outside cannot call BookingRoom.Create() directly
    internal static BookingRoom Create(
        BookingId bookingId, RoomId roomId, DateRange period, Money rate)
    {
        return new BookingRoom
        {
            Id        = BookingRoomId.New(),
            BookingId = bookingId,
            RoomId    = roomId,
            Period    = period,
            Rate      = rate
        };
    }
}
```

`internal static` — same module-ஓள் Booking மட்டும் BookingRoom create பண்ண முடியும். Other modules அல்லது handlers directly create பண்ண முடியாது.

---

### Full Flow — Command Handler to DB

```
Command Handler
      │
      ▼
IBookingWriteRepository.GetByIdAsync(id)
      │ ← EF Core, change tracking ON, full aggregate loaded (all Includes)
      ▼
Booking (AggregateRoot)
      ├── booking.AddRoom(roomId, period, rate)   ← business logic inside
      ├── BookingRoom created internally           ← child added to _rooms list
      └── RaiseDomainEvent(BookingRoomAddedEvent)  ← event queued, not published yet
      │
      ▼
DbContext.SaveChangesAsync()
      │
      ├── EF detects: Booking modified + new BookingRoom
      ├── Inserts BookingRoom row + updates Booking — one transaction
      └── DomainEventInterceptor fires after save
            └── Publishes BookingRoomAddedEvent → background queue
```

Children never saved separately. Root controls everything. One `SaveChangesAsync` = full aggregate persisted.

---

### Entity vs AggregateRoot vs ValueObject — Quick Comparison

| | Entity | AggregateRoot | ValueObject |
|---|---|---|---|
| Identity | Has ID | Has ID | No ID |
| Repository | ❌ No | ✅ Yes | ❌ No |
| Domain Events | ❌ No | ✅ Yes | ❌ No |
| Direct access | Through Root | Direct | Through owner |
| Example | BookingRoom | Booking | Money, DateRange |

---

### DomainEventInterceptor (Infrastructure — not Shared.Kernel)

Lives in each module's Infrastructure project. Publishes events after DB save.

```csharp
// Example: Stays.Infrastructure/Interceptors/DomainEventInterceptor.cs
public sealed class DomainEventInterceptor : SaveChangesInterceptor
{
    private readonly IBackgroundTaskQueue _queue;
    private readonly IServiceScopeFactory _scopeFactory;
    private readonly ILogger<DomainEventInterceptor> _logger;

    public override async ValueTask<int> SavedChangesAsync(
        SaveChangesCompletedEventData eventData,
        int result,
        CancellationToken ct = default)
    {
        var aggregates = eventData.Context.ChangeTracker
            .Entries<AggregateRoot<Guid>>()
            .Where(e => e.Entity.DomainEvents.Any())
            .Select(e => e.Entity)
            .ToList();

        var events = aggregates.SelectMany(a => a.DomainEvents).ToList();
        aggregates.ForEach(a => a.ClearDomainEvents());

        // Background queue — same pattern as existing project
        // Partition key = AggregateId → same aggregate events = sequential
        foreach (var domainEvent in events)
        {
            var captured = domainEvent;
            await _queue.QueueAsync(async taskCt =>
            {
                try
                {
                    using var scope = _scopeFactory.CreateScope();
                    var publisher = scope.ServiceProvider.GetRequiredService<IPublisher>();
                    await publisher.Publish(captured, taskCt);
                }
                catch (Exception ex)
                {
                    _logger.LogError(ex,
                        "Domain event processing failed. {EventType} {AggregateId}",
                        captured.GetType().Name, captured.AggregateId);
                }
            }, partitionKey: captured.AggregateId.ToString());
        }

        return result;
    }
}
```

---

### ValueObject

```csharp
namespace OneNex.Shared.Kernel.Domain;

/// <summary>
/// Base class for Value Objects.
/// Equality by value — two VOs with same values = equal (no identity).
/// Must be immutable — no setters, only init.
/// </summary>
public abstract class ValueObject : IEquatable<ValueObject>
{
    protected abstract IEnumerable<object?> GetEqualityComponents();

    public bool Equals(ValueObject? other)
    {
        if (other is null) return false;
        if (other.GetType() != GetType()) return false;
        return GetEqualityComponents().SequenceEqual(other.GetEqualityComponents());
    }

    public override bool Equals(object? obj) => Equals(obj as ValueObject);

    public override int GetHashCode()
        => GetEqualityComponents()
            .Aggregate(0, (hash, obj) => HashCode.Combine(hash, obj?.GetHashCode() ?? 0));

    public static bool operator ==(ValueObject? left, ValueObject? right)
        => left is null ? right is null : left.Equals(right);

    public static bool operator !=(ValueObject? left, ValueObject? right)
        => !(left == right);
}
```

---

## Value Objects

### Money (Most Critical)

```csharp
namespace OneNex.Shared.Kernel.ValueObjects;

/// <summary>
/// Represents a monetary amount with currency.
/// Prevents currency mixing — adding USD + LKR = exception.
/// All billing, pricing, folio charges use this.
/// </summary>
public sealed class Money : ValueObject
{
    public decimal  Amount   { get; }
    public string   Currency { get; }   // ISO 4217: "USD", "LKR", "SGD"

    private Money() { }  // EF Core

    private Money(decimal amount, string currency)
    {
        if (amount < 0)
            throw new DomainException("Money amount cannot be negative.");
        if (string.IsNullOrWhiteSpace(currency) || currency.Length != 3)
            throw new DomainException("Currency must be a valid 3-letter ISO 4217 code.");

        Amount   = Math.Round(amount, 2);
        Currency = currency.ToUpperInvariant();
    }

    public static Money Of(decimal amount, string currency) => new(amount, currency);
    public static Money Zero(string currency) => new(0, currency);

    public Money Add(Money other)
    {
        if (Currency != other.Currency)
            throw new DomainException($"Cannot add {Currency} and {other.Currency}.");
        return new Money(Amount + other.Amount, Currency);
    }

    public Money Subtract(Money other)
    {
        if (Currency != other.Currency)
            throw new DomainException($"Cannot subtract {Currency} from {other.Currency}.");
        return new Money(Amount - other.Amount, Currency);
    }

    public Money Multiply(decimal factor) => new(Amount * factor, Currency);

    public Money ApplyPercentage(Percentage percentage)
        => new(Amount * (percentage.Value / 100m), Currency);

    public static Money operator +(Money a, Money b) => a.Add(b);
    public static Money operator -(Money a, Money b) => a.Subtract(b);
    public static Money operator *(Money a, decimal factor) => a.Multiply(factor);

    public bool IsZero => Amount == 0;

    public override string ToString() => $"{Amount:F2} {Currency}";

    protected override IEnumerable<object?> GetEqualityComponents()
    {
        yield return Amount;
        yield return Currency;
    }
}
```

---

### Email

```csharp
namespace OneNex.Shared.Kernel.ValueObjects;

public sealed class Email : ValueObject
{
    public string Value { get; }

    private Email() { }

    private Email(string value)
    {
        if (string.IsNullOrWhiteSpace(value))
            throw new DomainException("Email cannot be empty.");
        if (!IsValidEmail(value))
            throw new DomainException($"'{value}' is not a valid email address.");

        Value = value.ToLowerInvariant().Trim();
    }

    public static Email Of(string value) => new(value);
    public static implicit operator string(Email email) => email.Value;

    private static bool IsValidEmail(string email)
    {
        try { _ = new System.Net.Mail.MailAddress(email); return true; }
        catch { return false; }
    }

    public override string ToString() => Value;

    protected override IEnumerable<object?> GetEqualityComponents()
    {
        yield return Value;
    }
}
```

---

### PhoneNumber

```csharp
namespace OneNex.Shared.Kernel.ValueObjects;

public sealed class PhoneNumber : ValueObject
{
    public string Value { get; }   // stored in E.164 format: +94771234567

    private PhoneNumber() { }

    private PhoneNumber(string value)
    {
        if (string.IsNullOrWhiteSpace(value))
            throw new DomainException("Phone number cannot be empty.");

        var cleaned = value.Trim();
        if (!cleaned.StartsWith('+') || cleaned.Length < 8 || cleaned.Length > 16)
            throw new DomainException("Phone must be in E.164 format: +[country][number]");

        Value = cleaned;
    }

    public static PhoneNumber Of(string value) => new(value);
    public static implicit operator string(PhoneNumber phone) => phone.Value;
    public override string ToString() => Value;

    protected override IEnumerable<object?> GetEqualityComponents()
    {
        yield return Value;
    }
}
```

---

### DateRange

```csharp
namespace OneNex.Shared.Kernel.ValueObjects;

/// <summary>
/// Represents a date range (check-in to check-out, availability window, etc.)
/// Start inclusive, End exclusive — industry standard for hotel stays.
/// e.g. CheckIn Jan 10, CheckOut Jan 12 = 2 nights (Jan 10, Jan 11)
/// </summary>
public sealed class DateRange : ValueObject
{
    public DateOnly Start { get; }
    public DateOnly End   { get; }

    public int Nights => End.DayNumber - Start.DayNumber;

    private DateRange() { }

    private DateRange(DateOnly start, DateOnly end)
    {
        if (end <= start)
            throw new DomainException("End date must be after start date.");

        Start = start;
        End   = end;
    }

    public static DateRange Of(DateOnly start, DateOnly end) => new(start, end);

    public static DateRange Of(DateTime start, DateTime end)
        => new(DateOnly.FromDateTime(start), DateOnly.FromDateTime(end));

    public bool Overlaps(DateRange other)
        => Start < other.End && End > other.Start;

    public bool Contains(DateOnly date)
        => date >= Start && date < End;

    public override string ToString() => $"{Start:yyyy-MM-dd} → {End:yyyy-MM-dd} ({Nights} nights)";

    protected override IEnumerable<object?> GetEqualityComponents()
    {
        yield return Start;
        yield return End;
    }
}
```

---

### TimeRange

```csharp
namespace OneNex.Shared.Kernel.ValueObjects;

/// <summary>
/// Represents a time window within a day.
/// Used for: business hours, dining reservations, wellness appointments.
/// </summary>
public sealed class TimeRange : ValueObject
{
    public TimeOnly Start { get; }
    public TimeOnly End   { get; }

    public int DurationMinutes => (int)(End - Start).TotalMinutes;

    private TimeRange() { }

    private TimeRange(TimeOnly start, TimeOnly end)
    {
        if (end <= start)
            throw new DomainException("End time must be after start time.");

        Start = start;
        End   = end;
    }

    public static TimeRange Of(TimeOnly start, TimeOnly end) => new(start, end);
    public static TimeRange Of(int startHour, int startMin, int endHour, int endMin)
        => new(new TimeOnly(startHour, startMin), new TimeOnly(endHour, endMin));

    public bool Overlaps(TimeRange other)
        => Start < other.End && End > other.Start;

    public bool Contains(TimeOnly time)
        => time >= Start && time < End;

    public override string ToString() => $"{Start:HH:mm} – {End:HH:mm}";

    protected override IEnumerable<object?> GetEqualityComponents()
    {
        yield return Start;
        yield return End;
    }
}
```

---

### Percentage

```csharp
namespace OneNex.Shared.Kernel.ValueObjects;

/// <summary>
/// Represents a percentage value (0 to 100).
/// Used for: tax rates, service charges, discounts, cancellation penalties.
/// </summary>
public sealed class Percentage : ValueObject
{
    public decimal Value { get; }   // 0.00 to 100.00

    private Percentage() { }

    private Percentage(decimal value)
    {
        if (value < 0 || value > 100)
            throw new DomainException("Percentage must be between 0 and 100.");

        Value = Math.Round(value, 2);
    }

    public static Percentage Of(decimal value) => new(value);
    public static Percentage Zero => new(0);

    public decimal AsDecimalFactor => Value / 100m;

    public override string ToString() => $"{Value}%";

    protected override IEnumerable<object?> GetEqualityComponents()
    {
        yield return Value;
    }
}
```

---

### Address

```csharp
namespace OneNex.Shared.Kernel.ValueObjects;

public sealed class Address : ValueObject
{
    public string  Line1    { get; }
    public string? Line2    { get; }
    public string  City     { get; }
    public string? State    { get; }
    public string  Country  { get; }   // ISO 3166-1 alpha-2: "LK", "SG", "IN"
    public string? PostalCode { get; }
    public double? Latitude  { get; }
    public double? Longitude { get; }

    private Address() { }

    private Address(
        string line1, string? line2, string city,
        string? state, string country, string? postalCode,
        double? latitude, double? longitude)
    {
        if (string.IsNullOrWhiteSpace(line1))    throw new DomainException("Address line1 required.");
        if (string.IsNullOrWhiteSpace(city))     throw new DomainException("City required.");
        if (string.IsNullOrWhiteSpace(country))  throw new DomainException("Country required.");

        Line1      = line1;
        Line2      = line2;
        City       = city;
        State      = state;
        Country    = country.ToUpperInvariant();
        PostalCode = postalCode;
        Latitude   = latitude;
        Longitude  = longitude;
    }

    public static Address Of(
        string line1, string? line2, string city,
        string? state, string country, string? postalCode,
        double? latitude = null, double? longitude = null)
        => new(line1, line2, city, state, country, postalCode, latitude, longitude);

    public bool HasCoordinates => Latitude.HasValue && Longitude.HasValue;

    protected override IEnumerable<object?> GetEqualityComponents()
    {
        yield return Line1;
        yield return Line2;
        yield return City;
        yield return State;
        yield return Country;
        yield return PostalCode;
    }
}
```

---

## Exceptions

### DomainException

```csharp
namespace OneNex.Shared.Kernel.Exceptions;

/// <summary>
/// Thrown when a domain rule is violated.
/// Maps to HTTP 400 Bad Request.
/// Use for: business rule violations within domain methods.
/// </summary>
public sealed class DomainException : Exception
{
    public DomainException(string message) : base(message) { }
}
```

### NotFoundException

```csharp
namespace OneNex.Shared.Kernel.Exceptions;

/// <summary>
/// Thrown when a requested resource does not exist.
/// Maps to HTTP 404 Not Found.
/// </summary>
public sealed class NotFoundException : Exception
{
    public NotFoundException(string resource, object id)
        : base($"{resource} with id '{id}' was not found.") { }

    public NotFoundException(string message) : base(message) { }
}
```

### ValidationException

```csharp
namespace OneNex.Shared.Kernel.Exceptions;

/// <summary>
/// Thrown by ValidationBehavior when FluentValidation fails.
/// Maps to HTTP 422 Unprocessable Entity.
/// </summary>
public sealed class ValidationException : Exception
{
    public IReadOnlyDictionary<string, string[]> Errors { get; }

    public ValidationException(IDictionary<string, string[]> errors)
        : base("One or more validation errors occurred.")
    {
        Errors = errors.AsReadOnly();
    }
}
```

### ConflictException

```csharp
namespace OneNex.Shared.Kernel.Exceptions;

/// <summary>
/// Thrown when an operation conflicts with existing state.
/// Maps to HTTP 409 Conflict.
/// Use for: duplicate bookings, overlapping schedules, room already occupied.
/// </summary>
public sealed class ConflictException : Exception
{
    public ConflictException(string message) : base(message) { }
}
```

### ForbiddenException

```csharp
namespace OneNex.Shared.Kernel.Exceptions;

/// <summary>
/// Thrown when user lacks permission for an operation.
/// Maps to HTTP 403 Forbidden.
/// </summary>
public sealed class ForbiddenException : Exception
{
    public ForbiddenException(string message = "You do not have permission to perform this action.")
        : base(message) { }
}
```

---

## IDateTimeProvider — System Clock Abstraction

### Why Not `DateTime.UtcNow` Directly?

```csharp
// ❌ WRONG — used directly everywhere
public void Cancel(string reason)
{
    CancelledAt = DateTime.UtcNow;   // untestable — always real clock
}

// ✅ CORRECT — injected, mockable in tests
public void Cancel(string reason, DateTime cancelledAt)
{
    CancelledAt = cancelledAt;   // tests can pass any time
}
```

`DateTime.UtcNow` direct-ஆ use பண்ணா unit tests-ல் time-dependent logic test பண்ண முடியாது. "Booking cancel பண்ண 24 hours-க்கு முன்னால் பண்ணா full refund" — இந்த rule test பண்ண வேணும் என்னா time control வேணும்.

---

### Interface — Shared.Kernel

```csharp
// Shared.Kernel/Primitives/IDateTimeProvider.cs
namespace OneNex.Shared.Kernel.Primitives;

/// <summary>
/// Abstraction over system clock.
/// Inject this instead of DateTime.UtcNow — enables time control in tests.
/// All modules use this. Never call DateTime.UtcNow or DateTimeOffset.UtcNow directly.
/// </summary>
public interface IDateTimeProvider
{
    DateTime UtcNow    { get; }      // full timestamp
    DateOnly TodayUtc  { get; }      // date only (availability checks, night audit)
    TimeOnly TimeUtc   { get; }      // time only (business hours checks)
}
```

---

### Implementation — Infrastructure (not Shared.Kernel)

```csharp
// Shared.Infrastructure/Time/SystemDateTimeProvider.cs
namespace OneNex.Shared.Infrastructure.Time;

internal sealed class SystemDateTimeProvider : IDateTimeProvider
{
    public DateTime UtcNow   => DateTime.UtcNow;
    public DateOnly TodayUtc => DateOnly.FromDateTime(DateTime.UtcNow);
    public TimeOnly TimeUtc  => TimeOnly.FromDateTime(DateTime.UtcNow);
}
```

Registration (each module's DI or shared infra DI):

```csharp
services.AddSingleton<IDateTimeProvider, SystemDateTimeProvider>();
```

---

### Test Double — Unit Tests

```csharp
// Tests/Helpers/FakeDateTimeProvider.cs
public sealed class FakeDateTimeProvider : IDateTimeProvider
{
    public FakeDateTimeProvider(DateTime utcNow)
        => UtcNow = utcNow;

    public DateTime UtcNow   { get; set; }
    public DateOnly TodayUtc => DateOnly.FromDateTime(UtcNow);
    public TimeOnly TimeUtc  => TimeOnly.FromDateTime(UtcNow);

    // Advance time within a test
    public void Advance(TimeSpan by) => UtcNow = UtcNow.Add(by);
}

// Usage in test:
var clock = new FakeDateTimeProvider(new DateTime(2026, 01, 10, 14, 0, 0, DateTimeKind.Utc));
var handler = new CancelBookingCommandHandler(writeRepo, clock);

clock.Advance(TimeSpan.FromHours(25));   // simulate next day
// now test: late cancellation should charge penalty
```

---

### Usage in Domain / Handlers

```csharp
// Command Handler — inject IDateTimeProvider
public sealed class CancelBookingCommandHandler(
    IBookingWriteRepository writeRepo,
    IDateTimeProvider clock) : ICommandHandler<CancelBookingCommand>
{
    public async Task<ErrorOr<Success>> Handle(
        CancelBookingCommand command, CancellationToken ct)
    {
        var booking = await writeRepo.GetByIdAsync(command.BookingId, ct);

        // Pass clock.UtcNow — domain method controls the logic
        booking.Cancel(command.Reason, clock.UtcNow);

        await _context.SaveChangesAsync(ct);
        return Result.Success;
    }
}

// Domain entity — clock injected as parameter (pure, testable)
public void Cancel(string reason, DateTime cancelledAt)
{
    if (Status == BookingStatus.CheckedIn)
        throw new DomainException("Cannot cancel a checked-in booking.");

    // Business rule: cancel < 24h before check-in = penalty
    var hoursUntilCheckIn = (StayPeriod.Start.ToDateTime(TimeOnly.MinValue) - cancelledAt).TotalHours;
    var isLateCancellation = hoursUntilCheckIn < 24;

    Status      = BookingStatus.Cancelled;
    CancelledAt = cancelledAt;
    Penalty     = isLateCancellation ? CancellationPenalty.OneNight : CancellationPenalty.None;

    RaiseDomainEvent(new BookingCancelledEvent(Id, BusinessId, reason, isLateCancellation));
}
```

Domain method-ல் `IDateTimeProvider` inject பண்ண வேண்டாம் — handler-ல் `clock.UtcNow` எடுத்து parameter-ஆ pass பண்ணுங்க. Domain method pure-ஆ இருக்கும், test பண்ண easy.

---

### Rule

```
NEVER use DateTime.UtcNow or DateTimeOffset.UtcNow directly in any class.
ALWAYS inject IDateTimeProvider and use clock.UtcNow.

Exception: DomainEvent record initializer (EventId + OccurredOn) — acceptable.
```

---

## Usage Examples

### How an Aggregate Looks

```csharp
// Stays.Domain/Entities/Booking.cs
public sealed class Booking : AggregateRoot<Guid>
{
    public Guid        BusinessId { get; private set; }
    public Guid        GuestId    { get; private set; }
    public DateRange   StayPeriod { get; private set; } = null!;
    public Money       TotalAmount { get; private set; } = null!;
    public BookingStatus Status   { get; private set; }

    private Booking() { }   // EF Core

    public static Booking Create(
        Guid businessId, Guid guestId,
        DateRange stayPeriod, Money totalAmount)
    {
        var booking = new Booking
        {
            Id          = Guid.NewGuid(),
            BusinessId  = businessId,
            GuestId     = guestId,
            StayPeriod  = stayPeriod,
            TotalAmount = totalAmount,
            Status      = BookingStatus.Pending
        };

        booking.RaiseDomainEvent(new BookingCreatedEvent(
            booking.Id, businessId, guestId, stayPeriod, totalAmount));

        return booking;
    }

    public void Confirm()
    {
        if (Status != BookingStatus.Pending)
            throw new DomainException("Only pending bookings can be confirmed.");

        Status = BookingStatus.Confirmed;

        RaiseDomainEvent(new BookingConfirmedEvent(Id, BusinessId, GuestId));
    }

    public void Cancel(string reason)
    {
        if (Status == BookingStatus.CheckedIn)
            throw new DomainException("Cannot cancel a booking that is already checked in.");

        Status = BookingStatus.Cancelled;

        RaiseDomainEvent(new BookingCancelledEvent(Id, BusinessId, GuestId, reason));
    }
}
```

### How a Domain Event Looks

```csharp
// Stays.Domain/Events/BookingConfirmedEvent.cs
public sealed record BookingConfirmedEvent(
    Guid BookingId,
    Guid BusinessId,
    Guid GuestId
) : IDomainEvent
{
    public Guid     EventId     { get; } = Guid.NewGuid();
    public DateTime OccurredOn  { get; } = DateTime.UtcNow;
    public Guid     AggregateId => BookingId;   // partition key
}
```

### How Money is Used

```csharp
// Never primitives for money
var roomRate  = Money.Of(150.00m, "USD");
var tax       = roomRate.ApplyPercentage(Percentage.Of(10));   // 15.00 USD
var total     = roomRate + tax;                                 // 165.00 USD
var nights    = Money.Of(3 * 150.00m, "USD");

// Currency mixing = compile-time safe, runtime exception
var wrong = Money.Of(100, "USD") + Money.Of(100, "LKR");      // throws DomainException
```

---

## Rules

```
1. AggregateRoot only  → has repositories
2. Entity              → accessed only through its AggregateRoot
3. Value Object        → immutable, no ID, equality by value
4. Domain Events       → raised inside domain methods, published after SaveChanges
5. No business logic   → in Shared.Kernel. Only building blocks.
6. No external deps    → Shared.Kernel references nothing except MediatR (for INotification)
```

---

## Inheritance Hierarchy

```
ValueObject                        ← equality by value, immutable (Money, Email, etc.)
Entity<TId>                        ← equality by ID
  └── AuditableEntity<TId>         ← + CreatedAt, UpdatedAt, CreatedBy, UpdatedBy
        └── AggregateRoot<TId>     ← + DomainEvents list, RaiseDomainEvent()

record struct (IEntityId<Guid>)    ← typed IDs (BookingId, RoomId, GuestId...)
                                     NOT a ValueObject — lightweight stack type
```

## ID Decision Summary

```
Plain Guid          → ❌ no type safety, silent swap bugs
Full ValueObject    → ❌ overkill for IDs, heavy boilerplate
record struct       → ✅ type safety + zero overhead (OneNex choice)

DB stores:          Guid (EF Core converter handles this automatically)
App uses:           BookingId, RoomId, GuestId — compiler-enforced
```
