# OneNex — .NET Aspire Local Dev Setup

> Status: Living Document
> Last updated: 2026-09-09
> Covers: Aspire AppHost, PostgreSQL + Valkey wiring, dashboard, ServiceDefaults

---

## What is .NET Aspire?

```
Without Aspire:
  Terminal 1 → docker run postgres
  Terminal 2 → docker run valkey
  Terminal 3 → dotnet run --project src/WebAPI
  Manually manage connection strings in appsettings.Development.json
  Manually check if postgres started before app

With Aspire:
  dotnet run --project AppHost
  → starts PostgreSQL container
  → starts Valkey container
  → starts the app (waits for DB to be ready)
  → opens dashboard (logs, traces, health of all services)
  → connection strings injected automatically
```

One command. Everything wired. Dashboard included.

---

## Projects — What Gets Added

```
src/
  ├── OneNex.AppHost/           ← Aspire orchestrator (runs only locally)
  │   └── Program.cs
  │
  └── OneNex.ServiceDefaults/   ← shared config (health checks, telemetry)
      └── Extensions.cs
```

These two projects are dev-only tooling — not deployed to production.

---

## Packages

```xml
<!-- OneNex.AppHost/OneNex.AppHost.csproj -->
<Project Sdk="Microsoft.NET.Sdk">
  <Sdk Name="Aspire.AppHost.Sdk" Version="9.*" />

  <ItemGroup>
    <PackageReference Include="Aspire.Hosting.PostgreSQL" Version="9.*" />
    <PackageReference Include="Aspire.Hosting.Valkey"     Version="9.*" />

    <!-- Reference the main WebAPI project -->
    <ProjectReference
      Include="..\OneNex.WebAPI\OneNex.WebAPI.csproj"
      IsAspireProjectResource="false" />
  </ItemGroup>
</Project>
```

```xml
<!-- OneNex.ServiceDefaults/OneNex.ServiceDefaults.csproj -->
<Project Sdk="Microsoft.NET.Sdk">
  <ItemGroup>
    <PackageReference Include="Microsoft.Extensions.ServiceDiscovery" Version="9.*" />
    <PackageReference Include="OpenTelemetry.Exporter.OpenTelemetryProtocol" Version="1.*" />
    <PackageReference Include="OpenTelemetry.Extensions.Hosting"              Version="1.*" />
    <PackageReference Include="OpenTelemetry.Instrumentation.AspNetCore"      Version="1.*" />
    <PackageReference Include="OpenTelemetry.Instrumentation.Http"            Version="1.*" />
  </ItemGroup>
</Project>
```

---

## AppHost — Program.cs

```csharp
// OneNex.AppHost/Program.cs
var builder = DistributedApplication.CreateBuilder(args);

// PostgreSQL — Aspire starts container, manages port
var postgres = builder.AddPostgres("postgres")
    .WithDataVolume()                  // persists data between restarts
    .WithPgAdmin();                    // pgAdmin UI at localhost:xxxx (dev only)

// OneNex main database
var onenexDb = postgres.AddDatabase("onenex-db", databaseName: "onenex_main");

// Valkey — Aspire starts container
var valkey = builder.AddValkey("valkey")
    .WithDataVolume();                 // persists data between restarts

// Main WebAPI — waits for postgres + valkey to be ready
builder.AddProject<Projects.OneNex_WebAPI>("onenex-api")
    .WithReference(onenexDb)          // injects connection string automatically
    .WithReference(valkey)            // injects Valkey connection string automatically
    .WaitFor(onenexDb)                // app starts only after DB is ready
    .WaitFor(valkey);

builder.Build().Run();
```

`WithReference()` → Aspire injects connection strings as environment variables. No manual appsettings editing.

---

## ServiceDefaults — Shared Config

```csharp
// OneNex.ServiceDefaults/Extensions.cs
namespace Microsoft.Extensions.Hosting;

public static class Extensions
{
    /// <summary>
    /// Adds health checks, OpenTelemetry, service discovery.
    /// Called in WebAPI Program.cs — one line.
    /// </summary>
    public static IHostApplicationBuilder AddServiceDefaults(
        this IHostApplicationBuilder builder)
    {
        builder.ConfigureOpenTelemetry();
        builder.AddDefaultHealthChecks();
        builder.Services.AddServiceDiscovery();

        builder.Services.ConfigureHttpClientDefaults(http =>
        {
            http.AddStandardResilienceHandler();
            http.AddServiceDiscovery();
        });

        return builder;
    }

    public static IHostApplicationBuilder ConfigureOpenTelemetry(
        this IHostApplicationBuilder builder)
    {
        builder.Logging.AddOpenTelemetry(logging =>
        {
            logging.IncludeFormattedMessage = true;
            logging.IncludeScopes           = true;
        });

        builder.Services.AddOpenTelemetry()
            .WithMetrics(metrics =>
            {
                metrics.AddAspNetCoreInstrumentation()
                       .AddHttpClientInstrumentation()
                       .AddRuntimeInstrumentation();
            })
            .WithTracing(tracing =>
            {
                tracing.AddAspNetCoreInstrumentation()
                       .AddHttpClientInstrumentation();
            });

        builder.AddOpenTelemetryExporters();

        return builder;
    }

    private static IHostApplicationBuilder AddOpenTelemetryExporters(
        this IHostApplicationBuilder builder)
    {
        var useOtlpExporter = !string.IsNullOrWhiteSpace(
            builder.Configuration["OTEL_EXPORTER_OTLP_ENDPOINT"]);

        if (useOtlpExporter)
            builder.Services.AddOpenTelemetry().UseOtlpExporter();

        return builder;
    }

    public static IHostApplicationBuilder AddDefaultHealthChecks(
        this IHostApplicationBuilder builder)
    {
        builder.Services.AddHealthChecks()
            .AddCheck("self", () => HealthCheckResult.Healthy(), ["live"]);

        return builder;
    }

    public static WebApplication MapDefaultEndpoints(this WebApplication app)
    {
        // /health/live  → is app alive?
        // /health/ready → is app ready to serve traffic? (DB connected, etc.)
        app.MapHealthChecks("/health/live",  new HealthCheckOptions
            { Predicate = r => r.Tags.Contains("live") });

        app.MapHealthChecks("/health/ready", new HealthCheckOptions
            { Predicate = r => r.Tags.Contains("ready") });

        return app;
    }
}
```

---

## WebAPI — Program.cs Changes

```csharp
// OneNex.WebAPI/Program.cs
var builder = WebApplication.CreateBuilder(args);

// ServiceDefaults — health checks + OpenTelemetry + service discovery
builder.AddServiceDefaults();    // ← one line, Aspire wires telemetry

// Connection strings come from Aspire (environment variables) automatically
// No manual appsettings editing needed in dev

builder.Services.AddSharedInfrastructure(builder.Configuration);
builder.Services
    .AddStaysModule(builder.Configuration)
    .AddDiningModule(builder.Configuration)
    .AddMembershipModule(builder.Configuration)
    .AddNotificationModule(builder.Configuration);

builder.Services.AddControllers();
builder.Services.AddOpenApi();

var app = builder.Build();

app.MapDefaultEndpoints();        // /health/live + /health/ready

app.UseCorrelationId();
app.UseSerilogRequestLogging();
app.UseExceptionHandling();

if (app.Environment.IsDevelopment())
    app.MapScalarApiReference();

app.UseAuthentication();
app.UseAuthorization();
app.MapControllers();

app.Run();
```

---

## Aspire Dashboard

```
dotnet run --project OneNex.AppHost
  │
  ▼
Dashboard: https://localhost:17xxx

Shows:
  ├── Resources tab   → postgres ✅ | valkey ✅ | onenex-api ✅ (health status)
  ├── Console tab     → live logs from all services in one place
  ├── Traces tab      → distributed traces (one HTTP request → all spans)
  └── Metrics tab     → request count, latency, error rate
```

All services' logs in one place. No switching between terminals.

---

## Connection String Flow

```
Without Aspire (manual):
  appsettings.Development.json → "Default": "Host=localhost;Port=5432;..."
  developer must know port, credentials, remember to start Docker

With Aspire:
  AppHost starts postgres container → random port assigned
  AppHost injects → ConnectionStrings__onenex-db = "Host=localhost;Port=54321;..."
  App reads from environment → works automatically
  Developer types nothing
```

---

## Run Command

```bash
# Start everything
dotnet run --project src/OneNex.AppHost

# Output:
# Login to the dashboard at: https://localhost:17137
# onenex-api: https://localhost:5001
# postgres:   started ✅
# valkey:     started ✅
```

---

## Dev Without Aspire (still works)

Aspire = optional dev tooling. App still runs standalone:

```bash
# Manual — start infra separately
docker run -d -p 5432:5432 -e POSTGRES_PASSWORD=dev postgres:17
docker run -d -p 6379:6379 valkey/valkey

# Run app directly
dotnet run --project src/OneNex.WebAPI
```

```json
// appsettings.Development.json — manual connection strings
{
  "ConnectionStrings": {
    "Default": "Host=localhost;Port=5432;Database=onenex_main;Username=postgres;Password=dev"
  },
  "Cache": {
    "UseInMemory": true    // skip Valkey entirely in dev if preferred
  }
}
```

---

## Project Structure — Final

```
OneNex/
  ├── src/
  │   ├── OneNex.AppHost/              ← Aspire orchestrator
  │   │   └── Program.cs
  │   │
  │   ├── OneNex.ServiceDefaults/      ← shared health + telemetry
  │   │   └── Extensions.cs
  │   │
  │   ├── Shared.Kernel/
  │   ├── Shared.Infrastructure/
  │   ├── Stays.Domain/
  │   ├── Stays.Application/
  │   ├── Stays.Infrastructure/
  │   ├── Dining.Domain/
  │   ├── Dining.Application/
  │   ├── Dining.Infrastructure/
  │   └── OneNex.WebAPI/               ← single entry point
  │
  └── tests/
      └── Architecture.Tests/
```

---

## Rules

```
1.  AppHost          → dev only. Never deployed. Orchestrates local infra.
2.  ServiceDefaults  → added to WebAPI once. Health checks + OpenTelemetry.
3.  WithReference()  → Aspire injects connection strings. No manual appsettings in dev.
4.  WaitFor()        → app waits for DB ready. No connection refused on startup.
5.  WithDataVolume() → postgres + valkey data persists between restarts.
6.  WithPgAdmin()    → pgAdmin UI in dev. Remove from staging/prod.
7.  dotnet run AppHost → one command, everything starts.
8.  Standalone still works → manual Docker + appsettings for devs who prefer it.
9.  Health checks    → /health/live (process alive) + /health/ready (deps ready).
10. Dashboard        → logs + traces + metrics for all services in one browser tab.
```
