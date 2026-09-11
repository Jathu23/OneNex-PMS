# OneNex — Observability

> Status: Living Document
> Last updated: 2026-09-11
> Covers: Serilog structured logging, Correlation ID, log enrichment, sinks, LoggingBehavior

---

## Goal

```
Every log line must answer:
  WHO         → UserId, BusinessId (from JWT, pushed by LoggingBehavior)
  WHAT        → request name, action taken
  WHEN        → timestamp
  WHERE       → MachineName, EnvironmentName
  HOW LONG    → elapsed ms (LoggingBehavior + SerilogRequestLogging)
  WHICH REQ   → CorrelationId (trace all logs for one HTTP request end-to-end)
```

---

## Why This Approach

### Why Serilog — not default .NET ILogger?

Default `ILogger` (Microsoft.Extensions.Logging) is an abstraction only — it has no structured output, no sinks, no enrichers. Console output is plain text; you cannot query it.

Serilog writes **structured events** (key-value pairs), not plain strings. Every log line is a JSON object that any log aggregator (Seq, Datadog, Loki, Azure Monitor) can index and query.

```
// Plain text — useless in production
"Handled CreateBookingCommand in 43ms"

// Structured — queryable
{ "RequestName": "CreateBookingCommand", "ElapsedMs": 43, "UserId": "...", "CorrelationId": "..." }
```

Serilog is the .NET industry standard for structured logging. It integrates with `ILogger<T>` — all existing `logger.LogInformation()` calls continue to work unchanged.

### Why CorrelationId Middleware — not a package?

`Serilog.Enrichers.CorrelationId` package exists, but it reads `HttpContext.TraceIdentifier` (ASP.NET's internal ID — not sent to clients). We need:
1. Client-provided ID — frontend sends `X-Correlation-Id`, we use it (end-to-end trace across services)
2. Echo in response header — client can log the same ID
3. Full control over the header name and fallback behaviour

Writing 30 lines of middleware gives us all three. The package gives us none of them.

### Why LoggingBehavior in MediatR pipeline — not in each handler?

Without the pipeline behavior, every handler would need:
```csharp
// Repeated in every handler — 50+ handlers eventually
logger.LogInformation("Handling CreateBookingCommand");
var sw = Stopwatch.StartNew();
// ... actual work ...
logger.LogInformation("Handled CreateBookingCommand in {ElapsedMs}ms", sw.ElapsedMilliseconds);
```

One `LoggingBehavior` registered once → automatic for every command and query. Zero boilerplate in handlers.

It also pushes `UserId` and `BusinessId` into `LogContext` — any `logger.Log*()` call inside the handler gets these automatically without the handler needing to know about them.

### Why separate layers — not one giant logging middleware?

Each layer has a distinct responsibility:

| Layer | Responsibility | Why separate |
|-------|---------------|--------------|
| Serilog engine | Structured output, sinks | Replaces the logger — always runs |
| CorrelationId MW | Request identity | Must be first — before any log fires |
| SerilogRequestLogging | HTTP traffic summary | One line per request — not per handler |
| LoggingBehavior | Business operation trace | Runs inside CQRS — knows UserId/BusinessId |
| ExceptionHandling MW | Unexpected error logging | Catches what falls through everything else |

If all this was one middleware, it would need to know about MediatR, JWT, business context — violating single responsibility.

### Why only log 500s — not all exceptions?

`DomainException`, `NotFoundException`, `ValidationException` etc. are **expected outcomes** — a user tried to book a room that doesn't exist. That is not an error; it is the system working correctly. Logging it as `LogError` would flood the error dashboard with noise and make real bugs invisible.

Only unexpected exceptions (unhandled, uncaught — true bugs) deserve `LogError`.

---

## Packages

```xml
<!-- Shared.Infrastructure/Shared.Infrastructure.csproj -->
<PackageReference Include="Serilog.AspNetCore"             Version="9.*" />
<PackageReference Include="Serilog.Sinks.File"             Version="6.*" />
<PackageReference Include="Serilog.Enrichers.Environment"  Version="3.*" />
<PackageReference Include="Serilog.Enrichers.Thread"       Version="4.*" />
```

> `Serilog.AspNetCore` already includes the Console sink — no separate `Serilog.Sinks.Console` needed.
> CorrelationId is handled by our own `CorrelationIdMiddleware` — `Serilog.Enrichers.CorrelationId` package not used.

---

## Layers — How Logging Is Structured

```
┌─────────────────────────────────────────────────────────────┐
│  1. Serilog (engine)      — structured logging foundation   │
│  2. CorrelationId MW      — unique ID per HTTP request      │
│  3. SerilogRequestLogging — one summary line per request    │
│  4. LoggingBehavior       — CQRS command/query trace        │
│  5. Dapper logging        — SQL + elapsed (dev: decorator,  │
│                              prod: OpenTelemetry span)      │
│  6. EF Core logging       — SQL + elapsed (built-in)        │
│  7. ExceptionHandling MW  — 500 errors logged with ID       │
└─────────────────────────────────────────────────────────────┘
```

---

## Layer 1 — Serilog Engine

### Bootstrap Logger

Captures startup crashes before DI and `appsettings.json` fully load.

```csharp
// Program.cs — before WebApplication.CreateBuilder()
Log.Logger = new LoggerConfiguration()
    .WriteTo.Console()
    .CreateBootstrapLogger();
```

If `AddSharedInfrastructure()` or any module registration throws — this logger catches it.

### Full Serilog via UseSerilog()

Replaces the default .NET `ILogger` with Serilog. Config is read from `appsettings.json`.

```csharp
builder.Host.UseSerilog((context, services, config) =>
    config
        .ReadFrom.Configuration(context.Configuration)  // appsettings.json "Serilog" section
        .ReadFrom.Services(services)                     // DI-registered enrichers
        .Enrich.FromLogContext()                         // LogContext.PushProperty() values
        .Enrich.WithMachineName()                        // which server handled the request
        .Enrich.WithEnvironmentName()                    // Development / Production
        .Enrich.WithThreadId());                         // useful for diagnosing async issues
```

### Startup / Shutdown Wrapper

```csharp
try
{
    // ... full app setup and app.Run()
}
catch (Exception ex)
{
    Log.Fatal(ex, "OneNex API failed to start.");
}
finally
{
    Log.CloseAndFlush();  // flush buffered logs before process exits
}
```

---

## Layer 2 — Correlation ID Middleware

Every HTTP request gets a unique ID. All log lines for that request share it.

```
Request 1 → CorrelationId = "a1b2c3"
  [a1b2c3] Handling CreateBookingCommand | UserId: xxx
  [a1b2c3] INSERT stays.bookings
  [a1b2c3] Handled CreateBookingCommand in 43ms
  [a1b2c3] HTTP POST /api/v1/stays/bookings → 201 in 48.2ms

Request 2 → CorrelationId = "d4e5f6"
  [d4e5f6] Handling CancelBookingCommand | UserId: yyy
  [d4e5f6] Handled CancelBookingCommand in 12ms
  [d4e5f6] HTTP POST /api/v1/stays/bookings/bf3c/cancel → 404 in 8.1ms
```

Filter logs by CorrelationId → see exactly what happened in one request.

```csharp
// Shared.Infrastructure/Middleware/CorrelationIdMiddleware.cs
public sealed class CorrelationIdMiddleware(RequestDelegate next)
{
    private const string CorrelationIdHeader = "X-Correlation-Id";

    public async Task InvokeAsync(HttpContext context)
    {
        // Use client-provided ID (frontend tracing) or generate a new one
        string correlationId =
            context.Request.Headers[CorrelationIdHeader].FirstOrDefault()
            ?? Guid.NewGuid().ToString();

        // Store in HttpContext.Items — ExceptionHandlingMiddleware reads this
        context.Items[CorrelationIdHeader] = correlationId;

        // Echo back to client — frontend can log it too
        context.Response.OnStarting(() =>
        {
            context.Response.Headers[CorrelationIdHeader] = correlationId;
            return Task.CompletedTask;
        });

        // Push into Serilog LogContext — ALL logs in this request get it automatically
        using (LogContext.PushProperty("CorrelationId", correlationId))
        {
            await next(context);
        }
    }
}
```

---

## Layer 3 — Serilog Request Logging

One structured log line per HTTP request (replaces ASP.NET Core's default noisy request logs).

```csharp
app.UseSerilogRequestLogging(options =>
{
    options.MessageTemplate =
        "HTTP {RequestMethod} {RequestPath} → {StatusCode} in {Elapsed:0.0}ms | CorrelationId: {CorrelationId}";

    options.EnrichDiagnosticContext = (diagnosticContext, httpContext) =>
    {
        diagnosticContext.Set(
            "CorrelationId",
            httpContext.Items["X-Correlation-Id"]?.ToString() ?? "none");
    };
});
```

**Output:**
```
[14:32:01 INF] [a1b2c3] HTTP POST /api/v1/stays/bookings → 201 in 48.2ms | CorrelationId: a1b2c3
[14:32:05 INF] [d4e5f6] HTTP GET  /api/v1/stays/rooms    → 200 in 12.1ms | CorrelationId: d4e5f6
```

---

## Layer 4 — LoggingBehavior (MediatR Pipeline)

Every command and query is logged automatically. Pushes `UserId` and `BusinessId` into `LogContext` — any `logger.Log*()` call inside the handler gets these properties automatically.

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
        CancellationToken cancellationToken)
    {
        string requestName = typeof(TRequest).Name;

        using (LogContext.PushProperty("UserId",
                   currentUser.IsAuthenticated ? currentUser.UserId : (object)"anonymous"))
        using (LogContext.PushProperty("BusinessId",
                   currentUser.IsAuthenticated ? currentUser.BusinessId : (object)"none"))
        {
            logger.LogInformation("Handling {RequestName}", requestName);

            Stopwatch stopwatch = Stopwatch.StartNew();
            TResponse response  = await next(cancellationToken);
            stopwatch.Stop();

            logger.LogInformation(
                "Handled {RequestName} in {ElapsedMs}ms",
                requestName,
                stopwatch.ElapsedMilliseconds);

            return response;
        }
    }
}
```

**Pipeline position:** `LoggingBehavior → ValidationBehavior → Handler`

---

## Layer 5 — Dapper Query Logging

EF Core-க்கு built-in SQL logging இருக்கு (`Microsoft.EntityFrameworkCore.Database.Command`). Dapper-க்கு இல்ல — Dapper deliberate ஆ logging provide பண்றதில்ல (Marc Gravell: "logging is a deployment-time concern, not a library concern").

Solution: `IDbConnection`-ஐ wrap பண்ற `LoggingDbConnection` decorator. Dev-ல factory இந்த wrapper return பண்றது — repository-ல எந்த code-உம் மாறாது.

```
Dev:
  NpgsqlConnectionFactory.OpenAsync()
    → new LoggingDbConnection(conn, logger)   ← intercepts every Dapper call
    → Dapper executes SQL via the wrapper
    → Log: SQL + params + elapsed time

Prod:
  NpgsqlConnectionFactory.OpenAsync()
    → conn directly                           ← zero overhead
```

### Dev Output

```
[14:32:01 DBG] Dapper 47ms
               SQL: SELECT id, name, type, base_price
                    FROM stays.rooms
                    WHERE business_id = @businessId AND status = 'Available'
               Params: businessId=a9b8c7d6-...

[14:32:01 DBG] Dapper 12ms
               SQL: SELECT b.id, b.guest_id, b.check_in, b.check_out
                    FROM stays.bookings b
                    WHERE b.business_id = @businessId AND b.status = @status
               Params: businessId=a9b8c7d6-... | status=Confirmed
```

### appsettings control

```json
// appsettings.Development.json — Dapper SQL visible in dev
{
  "Serilog": {
    "MinimumLevel": {
      "Override": {
        "OneNex.Shared.Infrastructure.Persistence": "Debug"
      }
    }
  }
}
```

```json
// appsettings.json (prod) — SQL hidden, timing only if needed
{
  "Serilog": {
    "MinimumLevel": {
      "Override": {
        "OneNex.Shared.Infrastructure.Persistence": "Information"
      }
    }
  }
}
```

> Dev-ல `Debug` → SQL + params + elapsed தெரியும்.
> Prod-ல `Information` → elapsed மட்டும் (SQL hidden — security).

### Production — OpenTelemetry Spans

Dev: Serilog `Debug` log போதும்.
Prod: OpenTelemetry `ActivitySource` span — Grafana Tempo-ல visual timeline.

```csharp
// Module repository
private static readonly ActivitySource Activity = new("OneNex.Stays");

public async Task<IEnumerable<RoomAvailabilityDto>> GetAvailableAsync(...)
{
    using Activity? span = Activity.StartActivity("stays.rooms.get-available");
    span?.SetTag("businessId", businessId.ToString());

    await using IDbConnection conn = await factory.OpenAsync(ct);
    return await conn.QueryAsync<RoomAvailabilityDto>(sql, new { businessId, ... });
}
```

Grafana Tempo output:
```
CreateBookingCommand [243ms]
  ├── stays.rooms.get-available    [47ms]   ← Dapper span
  ├── stays.pricing.calculate      [89ms]   ← Dapper span
  └── ef-core: SaveChanges         [107ms]  ← EF auto-instrumented
```

`ServiceDefaults`-ல `AddSource("OneNex.*")` add பண்ணினா — எல்லா module spans automatic capture ஆகும்.

---

## Layer 7 — Exception Handling Middleware

Unexpected exceptions (500) are logged with `CorrelationId` for traceability.
Expected domain errors (400/404/403/409/422) are **not** logged — they are business-expected outcomes.

```csharp
default:  // unexpected exception
    statusCode = StatusCodes.Status500InternalServerError;
    body       = new { success = false, message = "An unexpected error occurred." };

    logger.LogError(exception,
        "Unhandled exception. CorrelationId: {CorrelationId}",
        context.Items["X-Correlation-Id"]);
    break;
```

| Exception | HTTP | Logged? |
|-----------|------|---------|
| `ValidationException` | 422 | No |
| `DomainException` | 400 | No |
| `NotFoundException` | 404 | No |
| `ForbiddenException` | 403 | No |
| `ConflictException` | 409 | No |
| Any other | 500 | **Yes — LogError** |

---

## Middleware Order — Matters

```csharp
// Program.cs — pipeline order is critical
app.UseCorrelationId();          // 1st — assigns CorrelationId before anything logs
app.UseSerilogRequestLogging();  // 2nd — HTTP request log line (CorrelationId is now set)
app.UseExceptionHandling();      // 3rd — exception logs also have CorrelationId
app.UseHttpsRedirection();
app.UseAuthentication();
app.UseAuthorization();
app.MapControllers();
app.MapDefaultEndpoints();
```

> CorrelationId must be FIRST. All middleware and handlers registered below it automatically get the ID in every log line.

---

## appsettings.json — Serilog Config

```json
{
  "Serilog": {
    "MinimumLevel": {
      "Default": "Information",
      "Override": {
        "Microsoft":                     "Warning",
        "Microsoft.EntityFrameworkCore": "Warning",
        "System":                        "Warning"
      }
    },
    "WriteTo": [
      {
        "Name": "Console",
        "Args": {
          "outputTemplate": "[{Timestamp:HH:mm:ss} {Level:u3}] [{CorrelationId}] {Message:lj}{NewLine}{Exception}"
        }
      },
      {
        "Name": "File",
        "Args": {
          "path": "logs/onenex-.log",
          "rollingInterval": "Day",
          "retainedFileCountLimit": 7,
          "outputTemplate": "{Timestamp:yyyy-MM-dd HH:mm:ss.fff} [{Level:u3}] [{CorrelationId}] {Message:lj}{NewLine}{Exception}"
        }
      }
    ],
    "Enrich": ["FromLogContext", "WithMachineName", "WithEnvironmentName"]
  }
}
```

## appsettings.Development.json

```json
{
  "Cache": {
    "UseInMemory": true
  },
  "Serilog": {
    "MinimumLevel": {
      "Override": {
        "Microsoft.EntityFrameworkCore.Database.Command": "Information"
      }
    }
  }
}
```

> `Microsoft.EntityFrameworkCore.Database.Command: Information` → EF Core SQL queries visible in dev only.
> `Cache:UseInMemory: true` → InMemoryCacheService (no Docker needed in dev).

---

## What a Log Looks Like

### Console (Dev)

```
[14:32:01 INF] [a1b2c3d4] Handling CreateBookingCommand
[14:32:01 INF] [a1b2c3d4] Handled CreateBookingCommand in 43ms
[14:32:01 INF] [a1b2c3d4] HTTP POST /api/v1/stays/bookings → 201 in 48.2ms | CorrelationId: a1b2c3d4

[14:32:05 INF] [e5f6a7b8] Handling GetRoomAvailabilityQuery
[14:32:05 INF] [e5f6a7b8] Handled GetRoomAvailabilityQuery in 8ms
[14:32:05 INF] [e5f6a7b8] HTTP GET /api/v1/stays/rooms → 200 in 9.1ms | CorrelationId: e5f6a7b8
```

### Structured JSON (Production / Log Aggregator)

```json
{
  "Timestamp": "2026-09-11T14:32:01.123Z",
  "Level": "Information",
  "Message": "Handled CreateBookingCommand in 43ms",
  "CorrelationId": "a1b2c3d4",
  "UserId": "f1e2d3c4-b5a6-7890-abcd-ef1234567890",
  "BusinessId": "a9b8c7d6-e5f4-3210-fedc-ba9876543210",
  "MachineName": "onenex-prod-01",
  "EnvironmentName": "Production",
  "ThreadId": 12
}
```

---

## Where Files Live

```
src/
├── Host/
│   └── OneNex.WebAPI/
│       ├── Program.cs                      ← UseSerilog() + UseSerilogRequestLogging()
│       ├── appsettings.json                ← Serilog sinks + level config
│       └── appsettings.Development.json    ← Cache:UseInMemory + EF/Dapper SQL override
│
└── Shared.Infrastructure/
    ├── Middleware/
    │   ├── CorrelationIdMiddleware.cs      ← assigns CorrelationId to every request
    │   └── ExceptionHandlingMiddleware.cs  ← logs 500s with CorrelationId
    ├── Behaviors/
    │   └── LoggingBehavior.cs              ← pushes UserId + BusinessId, measures elapsed
    └── Persistence/
        ├── NpgsqlConnectionFactory.cs      ← opens Npgsql connection (prod: direct)
        └── LoggingDbConnection.cs          ← dev decorator: intercepts Dapper, logs SQL

src/Modules/Stays/Stays.Infrastructure/
    └── Repositories/
        └── RoomReadRepository.cs           ← ActivitySource span per query (prod tracing)
```

---

## Rules

```
1.  Serilog              → replaces default .NET logger. UseSerilog() in Program.cs.
2.  Bootstrap logger     → WriteTo.Console() only — catches startup crashes before config loads.
3.  CorrelationId        → FIRST middleware. All logs in one request share the same ID.
4.  X-Correlation-Id     → read from request header (frontend tracing) or generate new. Echoed in response.
5.  LogContext           → PushProperty() in middleware + LoggingBehavior. Auto-enriches all logs.
6.  UserId + BusinessId  → pushed by LoggingBehavior from ICurrentUser. Never log raw JWT.
7.  MinimumLevel         → Information default. Microsoft/EF Core = Warning (noise reduction).
8.  Dev override (EF)    → EF Core SQL = Information (see queries in dev). Cache = InMemory.
9.  Dev override (Dapper)→ LoggingDbConnection decorator — SQL + params + elapsed. Debug level.
10. Prod (Dapper)        → ActivitySource spans per query — visible in Grafana Tempo. No SQL in logs.
11. Structured logging   → always use {PropertyName} placeholders. NEVER string interpolation.
12. Log sinks            → Console + File. Production: add centralized sink (Seq, Datadog, etc.).
13. Error logging        → only unexpected exceptions (500) are LogError. Domain errors are not logged.
14. ThreadId enricher    → included — useful for diagnosing async/concurrent issues.
```
