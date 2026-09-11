# OneNex — Dependency Injection Structure

> Status: Living Document
> Last updated: 2026-09-07
> Covers: Program.cs clean pattern, extension methods, module DI, layer DI, Options pattern, Scrutor auto-scan

---

## Goal — Program.cs Stays Clean

```csharp
// ❌ Problem — everything in Program.cs
builder.Services.AddDbContext<StaysDbContext>(...);
builder.Services.AddMediatR(...);
builder.Services.AddScoped<IBookingWriteRepository, BookingWriteRepository>();
// ... 200+ more lines — unmaintainable
```

```csharp
// ✅ Solution — Program.cs only orchestrates
var builder = WebApplication.CreateBuilder(args);

builder.Services
    .AddSharedInfrastructure(builder.Configuration)
    .AddStaysModule(builder.Configuration)
    .AddDiningModule(builder.Configuration)
    .AddNotificationModule(builder.Configuration);

var app = builder.Build();

app.UseExceptionHandling();
app.UseAuthentication();
app.UseAuthorization();
app.MapControllers();

app.Run();
```

Program.cs = 15 lines. Module internals = module's own concern.

---

## Registration Hierarchy

```
Program.cs
  │
  ├── AddSharedInfrastructure()
  │     └── Shared.Infrastructure/DependencyInjection.cs
  │
  ├── AddStaysModule()
  │     └── Stays.WebAPI/DependencyInjection.cs  ← aggregator
  │           ├── AddStaysApplication()
  │           │     └── Stays.Application/DependencyInjection.cs
  │           └── AddStaysInfrastructure()
  │                 └── Stays.Infrastructure/DependencyInjection.cs
  │
  ├── AddDiningModule()           ← same pattern
  │
  └── AddNotificationModule()    ← same pattern
```

Each layer registers only its own concerns. No layer knows another layer's internals.

---

## IDbConnectionFactory — Dapper Connection

All Dapper read repositories இதை inject பண்ணும். Connection string handling ஒரே இடத்துல.

### Interface — Shared.Kernel

```csharp
// Shared.Kernel/Primitives/IDbConnectionFactory.cs
namespace OneNex.Shared.Kernel.Primitives;

public interface IDbConnectionFactory
{
    IDbConnection CreateConnection();   // returns closed connection — Dapper opens automatically
}
```

### Implementation — Shared.Infrastructure

```csharp
// Shared.Infrastructure/Persistence/NpgsqlConnectionFactory.cs
namespace OneNex.Shared.Infrastructure.Persistence;

public sealed class NpgsqlConnectionFactory(string connectionString)
    : IDbConnectionFactory
{
    private readonly string _connectionString = connectionString
        ?? throw new ArgumentNullException(
               nameof(connectionString), "Connection string cannot be null.");

    public IDbConnection CreateConnection()
        => new NpgsqlConnection(_connectionString);
}
```

`CreateConnection()` — closed connection return பண்ணும். Dapper internally open பண்ணும் — explicit `OpenAsync()` வேண்டாம்.

### Usage in Read Repository

```csharp
public sealed class BookingReadRepository
    : DapperRepository, IBookingReadRepository, IReadRepository
{
    public BookingReadRepository(IDbConnectionFactory db) : base(db) { }

    public async Task<BookingDetailDto?> FindByIdAsync(
        BookingId id, CancellationToken ct = default)
    {
        using var conn = _db.CreateConnection();   // closed — Dapper opens it

        return await conn.QuerySingleOrDefaultAsync<BookingDetailDto>(
            """
            SELECT b.id, g.full_name AS guest_name, r.room_number,
                   b.check_in, b.check_out, b.status, b.total_amount
            FROM stays.bookings b
            JOIN stays.guests g ON g.id = b.guest_id
            WHERE b.id = @Id AND b.is_deleted = false
            """,
            new { Id = id.Value });
    }
}
```

### Notification Module — Separate DB

Notification-க்கு different connection string. Keyed registration (.NET 8+):

```csharp
// Notification.Infrastructure/DependencyInjection.cs
services.AddKeyedSingleton<IDbConnectionFactory>("notification",
    new NpgsqlConnectionFactory(
        configuration.GetConnectionString("Notification")!));

// Usage — inject with key
public sealed class NotificationReadRepository(
    [FromKeyedServices("notification")] IDbConnectionFactory db)
    : DapperRepository(db), INotificationReadRepository, IReadRepository
{ }
```

---

## Shared.Infrastructure — DependencyInjection.cs

Cross-cutting concerns — every module needs these.

```csharp
// Shared.Infrastructure/DependencyInjection.cs
namespace OneNex.Shared.Infrastructure;

public static class DependencyInjection
{
    public static IServiceCollection AddSharedInfrastructure(
        this IServiceCollection services,
        IConfiguration configuration)
    {
        services.AddHttpContextAccessor();

        // ICurrentUser — reads JWT claims from HttpContext
        services.AddScoped<ICurrentUser, CurrentUser>();

        // IDateTimeProvider — testable system clock
        services.AddSingleton<IDateTimeProvider, SystemDateTimeProvider>();

        // IDbConnectionFactory — Dapper connection (Singleton — factory is stateless)
        services.AddSingleton<IDbConnectionFactory>(
            new NpgsqlConnectionFactory(
                configuration.GetConnectionString("Default")!));

        return services;
    }

    // Middleware extension — app.UseExceptionHandling()
    public static IApplicationBuilder UseExceptionHandling(
        this IApplicationBuilder app)
    {
        app.UseMiddleware<ExceptionHandlingMiddleware>();
        return app;
    }
}
```

---

## Stays.Application — DependencyInjection.cs

MediatR handlers + validators live here → registered here.

```csharp
// Stays.Application/DependencyInjection.cs
namespace OneNex.Stays.Application;

public static class DependencyInjection
{
    public static IServiceCollection AddStaysApplication(
        this IServiceCollection services)
    {
        // MediatR — scan this assembly for all handlers
        services.AddMediatR(cfg =>
        {
            cfg.RegisterServicesFromAssembly(
                typeof(DependencyInjection).Assembly);

            // Pipeline behaviors — order matters
            cfg.AddBehavior(
                typeof(IPipelineBehavior<,>),
                typeof(LoggingBehavior<,>));
            cfg.AddBehavior(
                typeof(IPipelineBehavior<,>),
                typeof(ValidationBehavior<,>));
            cfg.AddBehavior(
                typeof(IPipelineBehavior<,>),
                typeof(TransactionBehavior<,>));
        });

        // FluentValidation — scan this assembly for all validators
        services.AddValidatorsFromAssembly(
            typeof(DependencyInjection).Assembly);

        return services;
    }
}
```

---

## Stays.Infrastructure — DependencyInjection.cs

DbContext + repositories + interceptors — infrastructure concerns.

### Package

```xml
<!-- Stays.Infrastructure/Stays.Infrastructure.csproj -->
<PackageReference Include="Scrutor" Version="5.*" />
```

### Assembly Marker

```csharp
// Stays.Infrastructure/StaysInfrastructureMarker.cs
namespace OneNex.Stays.Infrastructure;

/// <summary>
/// Assembly anchor for Scrutor scanning.
/// No logic — just marks this assembly for reflection.
/// </summary>
internal sealed class StaysInfrastructureMarker { }
```

### Marker Interfaces — Shared.Kernel

```csharp
// Shared.Kernel/Primitives/IWriteRepository.cs
namespace OneNex.Shared.Kernel.Primitives;

/// <summary>
/// Marker interface — enables Scrutor auto-scan for EF Core write repositories.
/// No methods. Implement alongside the concrete repository interface.
/// </summary>
public interface IWriteRepository { }

// Shared.Kernel/Primitives/IReadRepository.cs
/// <summary>
/// Marker interface — enables Scrutor auto-scan for Dapper read repositories.
/// No methods. Implement alongside the concrete repository interface.
/// </summary>
public interface IReadRepository { }
```

### Each Repository — Implements Marker

```csharp
// Write repo — two interfaces: concrete + marker
public sealed class BookingWriteRepository
    : IBookingWriteRepository, IWriteRepository
{
    // EF Core implementation
}

// Read repo — two interfaces: concrete + marker
public sealed class BookingReadRepository
    : IBookingReadRepository, IReadRepository
{
    // Dapper implementation
}
```

New repo add = class create + marker implement. DI file never touch.

### DependencyInjection.cs — With Auto-Scan

```csharp
// Stays.Infrastructure/DependencyInjection.cs
namespace OneNex.Stays.Infrastructure;

public static class DependencyInjection
{
    public static IServiceCollection AddStaysInfrastructure(
        this IServiceCollection services,
        IConfiguration configuration)
    {
        // Typed settings
        services.Configure<StaysSettings>(
            configuration.GetSection(StaysSettings.SectionName));

        // Interceptors — manual (specific scoping + wiring needed)
        services.AddScoped<AuditInterceptor>();
        services.AddScoped<DomainEventInterceptor>();

        // DbContext — manual (complex setup with interceptors)
        services.AddDbContext<StaysDbContext>((sp, options) =>
            options
                .UseNpgsql(configuration.GetConnectionString("Default"))
                .UseSnakeCaseNamingConventions()
                .AddInterceptors(
                    sp.GetRequiredService<AuditInterceptor>(),
                    sp.GetRequiredService<DomainEventInterceptor>()));

        // ✅ Auto-scan — all Write + Read repositories in this assembly
        services.Scan(scan => scan
            .FromAssemblyOf<StaysInfrastructureMarker>()

            .AddClasses(c => c.AssignableTo<IWriteRepository>())
                .AsImplementedInterfaces()
                .WithScopedLifetime()

            .AddClasses(c => c.AssignableTo<IReadRepository>())
                .AsImplementedInterfaces()
                .WithScopedLifetime());

        return services;
    }
}
```

---

## Module Aggregator — Stays.WebAPI

```csharp
// Stays.WebAPI/DependencyInjection.cs
namespace OneNex.Stays.WebAPI;

public static class DependencyInjection
{
    public static IServiceCollection AddStaysModule(
        this IServiceCollection services,
        IConfiguration configuration)
    {
        services
            .AddStaysApplication()
            .AddStaysInfrastructure(configuration);

        return services;
    }
}
```

Program.cs `AddStaysModule()` call பண்ணா — Application + Infrastructure இரண்டும் ஒரே shot-ல் register ஆகும்.

---

## Settings — Options Pattern

`IConfiguration` directly handlers-ல் inject பண்ண வேண்டாம். Typed settings use பண்ணுவோம்.

```csharp
// Stays.Infrastructure/Settings/StaysSettings.cs
public sealed class StaysSettings
{
    public const string SectionName = "Stays";

    public int  MaxBookingDays       { get; init; } = 365;
    public int  DefaultCheckInHour   { get; init; } = 14;    // 2:00 PM
    public int  DefaultCheckOutHour  { get; init; } = 11;    // 11:00 AM
    public bool AllowSameDayBooking  { get; init; } = true;
}
```

```json
// appsettings.json
{
  "ConnectionStrings": {
    "Default": "Host=localhost;Database=onenex_main;Username=...;Password=..."
  },
  "Stays": {
    "MaxBookingDays": 365,
    "DefaultCheckInHour": 14,
    "DefaultCheckOutHour": 11,
    "AllowSameDayBooking": true
  },
  "Notification": {
    "ConnectionString": "Host=localhost;Database=onenex_notification;..."
  }
}
```

```csharp
// Usage in handler — inject IOptions<StaysSettings>
public sealed class CreateBookingCommandHandler(
    IBookingWriteRepository writeRepo,
    IOptions<StaysSettings> staysOptions,
    ICurrentUser currentUser,
    IDateTimeProvider clock)
{
    private readonly StaysSettings _settings = staysOptions.Value;

    public async Task<ErrorOr<BookingId>> Handle(...)
    {
        if (command.Nights > _settings.MaxBookingDays)
            return Errors.Booking.ExceedsMaxDays;
        // ...
    }
}
```

---

## Auto-Scan vs Manual — Decision Table

| What | How | Why |
|---|---|---|
| Write Repositories | ✅ Scrutor (`IWriteRepository`) | Same lifetime, convention-based |
| Read Repositories | ✅ Scrutor (`IReadRepository`) | Same lifetime, convention-based |
| Domain Services | ✅ Scrutor (`IDomainService`) | Same lifetime, convention-based |
| MediatR Handlers | ✅ MediatR scan | Built into `AddMediatR()` |
| FluentValidation | ✅ FV scan | Built into `AddValidatorsFromAssembly()` |
| DbContext | ❌ Manual | Needs interceptors wired explicitly |
| Interceptors | ❌ Manual | Specific scoping + order matters |
| ICurrentUser | ❌ Manual | Interface → impl mapping |
| IDateTimeProvider | ❌ Manual | Singleton lifetime |
| IOptions\<Settings\> | ❌ Manual | Needs config section name |
| IDbConnectionFactory | ❌ Manual | Needs connection string |

---

## DI File per Layer — Folder Location

```
src/
├── Shared.Kernel/
│   ├── Primitives/
│   │   ├── IWriteRepository.cs       ← marker interface
│   │   └── IReadRepository.cs        ← marker interface
│   └── (no DI file — zero dependencies)
│
├── Shared.Infrastructure/
│   └── DependencyInjection.cs        ← AddSharedInfrastructure()
│                                        UseExceptionHandling()
│
├── Stays.Application/
│   ├── DependencyInjection.cs        ← AddStaysApplication()
│   └── StaysApplicationMarker.cs     ← assembly anchor
│
├── Stays.Infrastructure/
│   ├── DependencyInjection.cs        ← AddStaysInfrastructure() + Scrutor scan
│   ├── StaysInfrastructureMarker.cs  ← assembly anchor for Scrutor
│   └── Settings/
│       └── StaysSettings.cs
│
├── Stays.WebAPI/
│   ├── DependencyInjection.cs        ← AddStaysModule() (aggregator)
│   └── Program.cs                    ← orchestration only
│
├── Dining.Application/
│   └── DependencyInjection.cs        ← AddDiningApplication()
│
├── Dining.Infrastructure/
│   ├── DependencyInjection.cs        ← AddDiningInfrastructure() + Scrutor scan
│   └── DiningInfrastructureMarker.cs
│
└── Notification/
    └── DependencyInjection.cs        ← AddNotificationModule()
```

---

## Full Program.cs — Final Shape

```csharp
// Stays.WebAPI/Program.cs
var builder = WebApplication.CreateBuilder(args);

// Shared — cross-cutting concerns
builder.Services.AddSharedInfrastructure(builder.Configuration);

// Modules — each self-contained
builder.Services
    .AddStaysModule(builder.Configuration)
    .AddDiningModule(builder.Configuration)
    .AddMembershipModule(builder.Configuration)
    .AddNotificationModule(builder.Configuration);

// API infrastructure
builder.Services.AddControllers();
builder.Services.AddOpenApi();

var app = builder.Build();

// Middleware pipeline — order matters
app.UseExceptionHandling();   // first — wraps everything

if (app.Environment.IsDevelopment())
    app.MapScalarApiReference();

app.UseAuthentication();
app.UseAuthorization();
app.MapControllers();

app.Run();
```

---

## Domain Services — Same Auto-Scan Pattern

Domain services-க்கும் same pattern apply பண்ணலாம்:

```csharp
// Shared.Kernel/Primitives/IDomainService.cs
public interface IDomainService { }

// Stays.Application — domain service implements marker
public sealed class RoomAvailabilityService
    : IRoomAvailabilityService, IDomainService { }

// Stays.Application/DependencyInjection.cs — add to scan
services.Scan(scan => scan
    .FromAssemblyOf<StaysApplicationMarker>()
    .AddClasses(c => c.AssignableTo<IDomainService>())
        .AsImplementedInterfaces()
        .WithScopedLifetime());
```

---

## Rules

```
1.  Program.cs         → orchestration only. No service details. ~20 lines max.
2.  Each layer         → own DependencyInjection.cs with single extension method
3.  Naming             → AddXxxApplication(), AddXxxInfrastructure(), AddXxxModule()
4.  Module aggregator  → AddXxxModule() combines Application + Infrastructure
5.  Shared.Kernel      → no DI file (zero dependencies — just interfaces and base classes)
6.  Shared.Infra       → AddSharedInfrastructure() + UseExceptionHandling() extension
7.  MediatR + validators → Application DI (handlers and validators live in Application)
8.  DbContext          → Infrastructure DI, manual (interceptors need explicit wiring)
9.  Interceptors       → Infrastructure DI, manual (scoping + order matters)
10. Repositories       → Scrutor auto-scan via IWriteRepository / IReadRepository markers
11. Domain services    → Scrutor auto-scan via IDomainService marker
12. Settings           → IOptions<T> typed class per module. Never raw IConfiguration in handlers.
13. New repo added     → implement marker interface. DI file untouched. Auto-picked up.
14. New module added   → one line in Program.cs. Internals fully encapsulated.
15. Middleware         → extension methods on IApplicationBuilder, not in services
```
