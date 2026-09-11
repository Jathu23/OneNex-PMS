# OneNex — EF Core Conventions

> Status: Living Document
> Last updated: 2026-09-07
> Covers: naming, schema, configuration pattern, value objects, soft delete, query filters

---

## Why Conventions Matter

EF Core default → `PascalCase` table/column names. PostgreSQL convention → `snake_case`.
Convention upfront-ல் set பண்ணாம விட்டா — DB-ல் `Bookings`, `TotalAmount`, `CheckInDate` — case-sensitive issues, ugly queries, inconsistency.

OneNex-ல் rules upfront — எல்லா module same pattern follow பண்ணும்.

---

## Package

```xml
<!-- Each module's Infrastructure.csproj -->
<PackageReference Include="EFCore.NamingConventions" Version="8.*" />
<PackageReference Include="Npgsql.EntityFrameworkCore.PostgreSQL" Version="9.*" />
```

---

## 1. DbContext per Module

ஒவ்வொரு module-க்கும் தனி DbContext. Modules share பண்றதில்லை.

```csharp
// Stays.Infrastructure/Persistence/StaysDbContext.cs
public sealed class StaysDbContext : DbContext
{
    private readonly ICurrentUser _currentUser;

    public StaysDbContext(
        DbContextOptions<StaysDbContext> options,
        ICurrentUser currentUser) : base(options)
    {
        _currentUser = currentUser;
    }

    public DbSet<Booking>  Bookings  => Set<Booking>();
    public DbSet<Room>     Rooms     => Set<Room>();
    public DbSet<Guest>    Guests    => Set<Guest>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // Schema per module
        modelBuilder.HasDefaultSchema("stays");

        // Auto-discover all IEntityTypeConfiguration<T> in this assembly
        modelBuilder.ApplyConfigurationsFromAssembly(typeof(StaysDbContext).Assembly);

        base.OnModelCreating(modelBuilder);
    }

    protected override void ConfigureConventions(ModelConfigurationBuilder config)
    {
        // Typed ID → Guid converters, auto-apply for all properties
        config.Properties<BookingId>().HaveConversion<EntityIdValueConverter<BookingId>>();
        config.Properties<RoomId>().HaveConversion<EntityIdValueConverter<RoomId>>();
        config.Properties<GuestId>().HaveConversion<EntityIdValueConverter<GuestId>>();
    }
}
```

Registration:

```csharp
// Stays.Infrastructure/DependencyInjection.cs
services.AddDbContext<StaysDbContext>(options =>
    options.UseNpgsql(connectionString)
           .UseSnakeCaseNamingConventions());   // ← all names auto snake_case
```

---

## 2. snake_case — Auto Conversion

`UseSnakeCaseNamingConventions()` — C# property names automatically converted:

```
C# Property       →   DB Column
──────────────────────────────────
BusinessId        →   business_id
TotalAmount       →   total_amount
CheckInDate       →   check_in_date
CreatedAt         →   created_at
IsDeleted         →   is_deleted
StayPeriod_Start  →   stay_period_start   (owned entity)
```

Manual override — specific column name வேண்டும் என்றால்:

```csharp
builder.Property(b => b.StayPeriod.Start)
    .HasColumnName("check_in");   // override auto name
```

---

## 3. Schema per Module

```csharp
modelBuilder.HasDefaultSchema("stays");
```

Result:

```
stays.bookings
stays.rooms
stays.guests
stays.rate_plans

dining.reservations
dining.tables
dining.menus

membership.members
membership.plans
```

Cross-module joins = never (modules communicate via events). Each schema = that module's territory.

---

## 4. IEntityTypeConfiguration — One File per Entity

**No data annotations in domain entities. Fluent API only in configuration files.**

```csharp
// Stays.Infrastructure/EntityConfigurations/BookingConfiguration.cs
public sealed class BookingConfiguration : IEntityTypeConfiguration<Booking>
{
    public void Configure(EntityTypeBuilder<Booking> builder)
    {
        builder.ToTable("bookings");

        builder.HasKey(b => b.Id);

        builder.Property(b => b.BusinessId).IsRequired();
        builder.Property(b => b.GuestId).IsRequired();

        // Enum → stored as string
        builder.Property(b => b.Status)
            .HasConversion<string>()
            .HasMaxLength(50)
            .IsRequired();

        builder.Property(b => b.CancelledReason)
            .HasMaxLength(500);

        // Value Object — owned entity (same table, no join)
        builder.OwnsOne(b => b.StayPeriod, period =>
        {
            period.Property(p => p.Start)
                .HasColumnName("check_in")
                .IsRequired();

            period.Property(p => p.End)
                .HasColumnName("check_out")
                .IsRequired();
        });

        builder.OwnsOne(b => b.TotalAmount, money =>
        {
            money.Property(m => m.Amount)
                .HasColumnName("total_amount")
                .HasPrecision(18, 4)
                .IsRequired();

            money.Property(m => m.Currency)
                .HasColumnName("total_currency")
                .HasMaxLength(3)
                .IsRequired();
        });

        // Child collection — Booking owns BookingRooms
        builder.HasMany(b => b.Rooms)
            .WithOne()
            .HasForeignKey(r => r.BookingId)
            .OnDelete(DeleteBehavior.Cascade);

        // Soft delete fields
        builder.Property(b => b.IsDeleted).IsRequired().HasDefaultValue(false);
        builder.Property(b => b.DeletedAt);
        builder.Property(b => b.DeletedBy);

        // Indexes
        builder.HasIndex(b => b.BusinessId);
        builder.HasIndex(b => new { b.BusinessId, b.Status });
        builder.HasIndex(b => b.GuestId);

        // Soft delete filter — never see deleted records
        builder.HasQueryFilter(b => !b.IsDeleted);

        // Optimistic concurrency — PostgreSQL native xmin column
        builder.UseXminAsConcurrencyToken();
    }
}
```

`ApplyConfigurationsFromAssembly` — new config file add பண்ணா automatic. DbContext-ல் manually register வேண்டாம்.

---

## 5. Value Objects — OwnsOne

Value Objects = same table, owned entity.

```csharp
// Money
builder.OwnsOne(b => b.TotalAmount, money =>
{
    money.Property(m => m.Amount)
        .HasColumnName("total_amount")
        .HasPrecision(18, 4)
        .IsRequired();

    money.Property(m => m.Currency)
        .HasColumnName("total_currency")
        .HasMaxLength(3)
        .IsRequired();
});

// DateRange
builder.OwnsOne(b => b.StayPeriod, period =>
{
    period.Property(p => p.Start).HasColumnName("check_in").IsRequired();
    period.Property(p => p.End).HasColumnName("check_out").IsRequired();
});

// Address (nullable — guest may not have address)
builder.OwnsOne(g => g.Address, addr =>
{
    addr.Property(a => a.Line1).HasColumnName("address_line1").HasMaxLength(200);
    addr.Property(a => a.City).HasColumnName("address_city").HasMaxLength(100);
    addr.Property(a => a.Country).HasColumnName("address_country").HasMaxLength(2);
    addr.Property(a => a.PostalCode).HasColumnName("address_postal_code").HasMaxLength(20);
    addr.Property(a => a.Latitude).HasColumnName("address_lat");
    addr.Property(a => a.Longitude).HasColumnName("address_lng");
});
```

DB result:

```
stays.bookings table:
  id, business_id, guest_id, status,
  check_in, check_out,           ← DateRange flattened
  total_amount, total_currency,  ← Money flattened
  is_deleted, deleted_at, deleted_by,
  created_at, updated_at, created_by, updated_by,
  xmin                           ← concurrency token (PostgreSQL system column)
```

No separate table for Money or DateRange. Clean, no joins.

---

## 6. ISoftDelete — Soft Delete

```csharp
// Shared.Kernel/Primitives/ISoftDelete.cs
public interface ISoftDelete
{
    bool      IsDeleted { get; }
    DateTime? DeletedAt { get; }
    Guid?     DeletedBy { get; }
}
```

Aggregate root-ல் implement:

```csharp
public sealed class Booking : AggregateRoot<BookingId>, ISoftDelete
{
    public bool      IsDeleted { get; private set; }
    public DateTime? DeletedAt { get; private set; }
    public Guid?     DeletedBy { get; private set; }

    public void SoftDelete(Guid deletedBy, DateTime deletedAt)
    {
        if (IsDeleted)
            throw new DomainException("Booking is already deleted.");

        IsDeleted = true;
        DeletedAt = deletedAt;
        DeletedBy = deletedBy;

        RaiseDomainEvent(new BookingDeletedEvent(Id, BusinessId));
    }
}
```

**AuditInterceptor** — DELETE statement intercept பண்ணி UPDATE-ஆ convert:

```csharp
// In AuditInterceptor, before SaveChanges
foreach (var entry in context.ChangeTracker.Entries<ISoftDelete>())
{
    if (entry.State == EntityState.Deleted)
    {
        entry.State = EntityState.Modified;          // DELETE → UPDATE
        entry.Entity.SoftDelete(
            _currentUser.UserId,
            _clock.UtcNow);
    }
}
```

Command handler-ல்:

```csharp
// Handler calls Delete — interceptor converts to soft delete automatically
_writeRepo.Delete(booking);
await _context.SaveChangesAsync(ct);
// → UPDATE stays.bookings SET is_deleted=true, deleted_at=..., deleted_by=... WHERE id=...
```

---

## 7. Global Query Filter — Soft Delete Only

```csharp
// In each entity configuration
builder.HasQueryFilter(b => !b.IsDeleted);
```

Every query automatic filter:

```sql
-- Developer writes (via EF):
SELECT * FROM stays.bookings WHERE status = 'Confirmed'

-- EF Core generates:
SELECT * FROM stays.bookings
WHERE is_deleted = false       ← automatic, always
  AND status = 'Confirmed'
```

**Business isolation** — `business_id` filter is NOT in global query filter.
Handler-ல் explicit-ஆ filter பண்ணுவோம்:

```csharp
// In handlers — explicit, clear, intentional
var bookings = await _readRepo.ListAsync(
    filter: new BookingFilter
    {
        BusinessId = _currentUser.BusinessId,   // explicit
        Status     = BookingStatus.Confirmed
    }, ct);
```

Filter bypass (admin / system operations):

```csharp
// Explicitly opt-out — never accidental
var deletedBooking = await _context.Bookings
    .IgnoreQueryFilters()
    .FirstOrDefaultAsync(b => b.Id == id && b.IsDeleted, ct);
```

---

## 8. No Data Annotations — Rule

```csharp
// ❌ NEVER — annotations in domain entity
public sealed class Booking : AggregateRoot<BookingId>
{
    [Required]
    [MaxLength(500)]
    public string CancelledReason { get; private set; } = null!;
}

// ✅ ALWAYS — in configuration class
public void Configure(EntityTypeBuilder<Booking> builder)
{
    builder.Property(b => b.CancelledReason)
        .HasMaxLength(500);
}
```

Domain entity = zero infrastructure knowledge. Configuration file = all DB mapping decisions.

---

## 9. Enum Storage — String not Int

```csharp
// ❌ Int stored — migration nightmare when enum order changes
builder.Property(b => b.Status).HasConversion<int>();

// ✅ String stored — readable in DB, order-independent
builder.Property(b => b.Status)
    .HasConversion<string>()
    .HasMaxLength(50)
    .IsRequired();
```

DB-ல்: `'Pending'`, `'Confirmed'`, `'CheckedIn'`, `'Cancelled'` — readable, debuggable.

---

## 10. Concurrency — Optimistic Locking

PostgreSQL-ல் `xmin` system column — every UPDATE automatically increments. No manual column needed.

```csharp
builder.UseXminAsConcurrencyToken();
```

Two users same booking-ஐ update பண்ண try பண்ணா:

```
User A reads booking (xmin = 5)
User B reads booking (xmin = 5)

User A saves → xmin becomes 6
User B saves → EF sees xmin mismatch → DbUpdateConcurrencyException
```

Handler-ல் handle:

```csharp
try
{
    await _context.SaveChangesAsync(ct);
}
catch (DbUpdateConcurrencyException)
{
    return Error.Conflict("Booking.Conflict",
        "Booking was modified by another request. Please retry.");
}
```

---

## DB Schema Layout

```
PostgreSQL — onenex_main
├── stays.*
│   ├── stays.bookings
│   ├── stays.booking_rooms
│   ├── stays.booking_guests
│   ├── stays.rooms
│   ├── stays.guests
│   └── stays.rate_plans
│
├── dining.*
│   ├── dining.reservations
│   ├── dining.tables
│   └── dining.menu_items
│
├── membership.*
│   ├── membership.members
│   └── membership.plans
│
└── shared.*
    └── shared.businesses

PostgreSQL — onenex_notification
├── notification.*
└── hangfire.*
```

---

## Conventions Summary

| Convention | Decision |
|---|---|
| Naming | `snake_case` via `UseSnakeCaseNamingConventions()` |
| Schema | One per module — `HasDefaultSchema("stays")` |
| Config | `IEntityTypeConfiguration<T>` — one file per entity |
| Auto-load | `ApplyConfigurationsFromAssembly()` — one line |
| Value Objects | `OwnsOne()` — flattened into same table |
| Enums | `HasConversion<string>()` — readable in DB |
| Soft Delete | `ISoftDelete` + interceptor converts DELETE → UPDATE |
| Query Filter | `!IsDeleted` only — business_id filtered explicitly in handlers |
| Annotations | ❌ Never in domain — Fluent API only |
| Concurrency | `UseXminAsConcurrencyToken()` — PostgreSQL native |
| Cross-module joins | ❌ Never — modules communicate via events only |

---

## Dapper — Read Path

> Dapper = reads only. EF Core = writes only. Never mix.

### Why Two Libraries

EF Core-ல query பண்ணும்போது — change tracker, identity map, lazy loading என நிறைய overhead இருக்கு. CQRS read side-ல அவை தேவையில்ல — ஒரு DTO return பண்ணினா போதும்.

Dapper = raw ADO.NET + object mapping. Query result → DTO — straight. EF-ஐ விட 3–5x faster for reads.

```
Command Handler → EF Core → AggregateRoot → SaveChangesAsync
Query Handler   → Dapper  → DTO           → return
```

### IDbConnectionFactory

```csharp
// Shared.Kernel/Primitives/IDbConnectionFactory.cs
public interface IDbConnectionFactory
{
    Task<IDbConnection> OpenAsync(CancellationToken ct = default);
}
```

```csharp
// Shared.Infrastructure/Persistence/NpgsqlConnectionFactory.cs
internal sealed class NpgsqlConnectionFactory(string connectionString) : IDbConnectionFactory
{
    public async Task<IDbConnection> OpenAsync(CancellationToken ct = default)
    {
        var conn = new NpgsqlConnection(connectionString);
        await conn.OpenAsync(ct);
        return conn;
    }
}
```

Dev-ல — `LoggingDbConnection`-ஆல wrap பண்ணி return பண்றது (observability doc பார்க்கவும்).
Prod-ல — `conn` directly return.

### Repository — No Base Class

Industry consensus (kgrzybek/modular-monolith-with-ddd, ardalis/CleanArchitecture): thin repository — Dapper directly, no base class wrapper methods.

Base class = connection management overhead + false abstraction. Real performance comes from SQL design, not wrapper design.

```csharp
// Stays.Infrastructure/Repositories/RoomReadRepository.cs
internal sealed class RoomReadRepository(IDbConnectionFactory factory)
    : IRoomReadRepository
{
    public async Task<IEnumerable<RoomAvailabilityDto>> GetAvailableAsync(
        Guid businessId, DateRange dates, CancellationToken ct)
    {
        await using IDbConnection conn = await factory.OpenAsync(ct);

        return await conn.QueryAsync<RoomAvailabilityDto>("""
            SELECT r.id, r.name, r.type, r.base_price
            FROM stays.rooms r
            WHERE r.business_id = @businessId
              AND r.status = 'Available'
              AND r.id NOT IN (
                  SELECT b.room_id
                  FROM stays.bookings b
                  WHERE b.check_in  < @checkOut
                    AND b.check_out > @checkIn
                    AND b.is_deleted = false
              )
            ORDER BY r.base_price
            """,
            new { businessId, checkIn = dates.Start, checkOut = dates.End });
    }
}
```

### Dapper Rules

#### Rule 1 — SELECT specific columns, never `SELECT *`

```sql
-- ❌ unused columns → network waste + memory waste
SELECT * FROM stays.rooms WHERE business_id = @businessId

-- ✅ only what the DTO needs
SELECT id, name, type, base_price, status
FROM stays.rooms
WHERE business_id = @businessId
```

#### Rule 2 — Parameters always, never string interpolation

```csharp
// ❌ SQL injection + query plan cache miss
string sql = $"SELECT id FROM stays.rooms WHERE business_id = '{businessId}'";

// ✅ parameterized — Postgres caches query plan
string sql = "SELECT id, name FROM stays.rooms WHERE business_id = @businessId";
await conn.QueryAsync<RoomDto>(sql, new { businessId });
```

Stack Overflow documented a real incident: dynamic SQL without parameters → query plan cache filled 2GB of memory.

#### Rule 3 — Optional filters → `DynamicParameters`

```csharp
// ❌ string concatenation — breaks query plan caching
string sql = "SELECT id FROM stays.rooms WHERE business_id = @businessId";
if (status.HasValue) sql += " AND status = @status";

// ✅ DynamicParameters — clean, safe, plan-cacheable
var p = new DynamicParameters();
p.Add("businessId", businessId);
if (status.HasValue)
    p.Add("status", status.Value.ToString());

string sql = """
    SELECT id, name FROM stays.rooms
    WHERE business_id = @businessId
      AND (@status IS NULL OR status = @status)
    """;

return await conn.QueryAsync<RoomDto>(sql, p);
```

#### Rule 4 — Buffered vs Unbuffered

Default `buffered: true` — all rows loaded into memory, connection freed immediately. Correct for 95% of queries.

`buffered: false` — rows streamed one-by-one, connection held open. Use only for large exports (500+ rows).

```csharp
// Normal query — buffered (default)
return await conn.QueryAsync<RoomDto>(sql, new { businessId });

// Export / report — unbuffered
return await conn.QueryAsync<ReportRowDto>(
    sql, new { businessId, month },
    buffered: false);
```

#### Rule 5 — JOIN, not multiple queries

```csharp
// ❌ 2 DB round trips
var room    = await conn.QuerySingleOrDefaultAsync<RoomDto>(roomSql, new { roomId });
var reviews = await conn.QueryAsync<ReviewDto>(reviewSql, new { roomId });

// ✅ 1 round trip — Dapper multi-mapping
string sql = """
    SELECT r.id, r.name, rv.id, rv.rating, rv.comment
    FROM stays.rooms r
    LEFT JOIN stays.reviews rv ON rv.room_id = r.id
    WHERE r.id = @roomId
    """;

var lookup = new Dictionary<Guid, RoomDetailDto>();
await conn.QueryAsync<RoomDetailDto, ReviewDto, RoomDetailDto>(
    sql,
    (room, review) =>
    {
        if (!lookup.TryGetValue(room.Id, out var existing))
            lookup[room.Id] = existing = room;
        if (review is not null)
            existing.Reviews.Add(review);
        return existing;
    },
    new { roomId },
    splitOn: "id");

return lookup.Values.FirstOrDefault();
```

#### Rule 6 — Always async

```csharp
// ❌ blocks thread — kills throughput
conn.Query<RoomDto>(sql, param);

// ✅ async — frees thread while DB responds
await conn.QueryAsync<RoomDto>(sql, param);
```

#### Rule 7 — Dapper = reads only

```
Query Handler  → Dapper → DTO                 ✅
Command Handler → EF Core → AggregateRoot    ✅
Command Handler → Dapper → write             ❌ never — EF change tracker bypassed
```

Command handler-ல Dapper write பண்ணா — domain events fire ஆகாது, audit fields set ஆகாது, concurrency token check ஆகாது.

### Dapper Rules Summary

| Rule | Pattern |
|------|---------|
| Columns | `SELECT id, name, ...` — never `SELECT *` |
| Params | Anonymous object / `DynamicParameters` — never interpolation |
| Optional filters | `DynamicParameters` + `@param IS NULL OR col = @param` |
| Large result sets | `buffered: false` for 500+ rows |
| Multiple fetches | JOIN + multi-mapping — never two separate queries |
| Async | `QueryAsync` only — never `Query` |
| Write path | EF Core only — Dapper = reads only |
