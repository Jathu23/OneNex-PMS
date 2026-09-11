# OneNex — Caching Strategy

> Status: Living Document
> Last updated: 2026-09-09
> Covers: ICacheService abstraction, dev vs production swap, CacheKeys, CacheTtl, what to cache, GetOrSetAsync pattern, production hosting options

---

## Core Design Decision

```
Handlers never touch IMemoryCache or IConnectionMultiplexer directly.
All caching goes through ICacheService abstraction.
Dev = IMemoryCache (no infra). Production = Valkey.
Zero handler code change when switching.
```

---

## Why Valkey (not Redis)?

```
Redis 7.4+ → changed license (SSPL — not truly open source)
Valkey     → Redis fork by Linux Foundation, BSD license (completely free)

Cost:     Zero. Dev, staging, production — no limits, no license fees.
Client:   Same StackExchange.Redis — zero code difference.
Switch:   docker run -d -p 6379:6379 valkey/valkey → done.
```

---

## ICacheService — The Abstraction

### Interface — Shared.Kernel

```csharp
// Shared.Kernel/Primitives/ICacheService.cs
namespace OneNex.Shared.Kernel.Primitives;

public interface ICacheService
{
    /// <summary>
    /// Get from cache. If missing, call factory, cache result, return it.
    /// Never call factory and cache separately — race condition risk.
    /// </summary>
    Task<T> GetOrSetAsync<T>(
        string            key,
        Func<Task<T>>     factory,
        TimeSpan?         ttl = null,
        CancellationToken ct  = default);

    /// <summary>Remove a specific cache entry (permission change, suspension).</summary>
    Task RemoveAsync(string key, CancellationToken ct = default);

    /// <summary>Set a flag key with optional TTL (suspension flags, feature flags).</summary>
    Task SetAsync<T>(
        string            key,
        T                 value,
        TimeSpan?         ttl = null,
        CancellationToken ct  = default);

    /// <summary>Check if key exists (suspension check — fast path).</summary>
    Task<bool> ExistsAsync(string key, CancellationToken ct = default);
}
```

---

## Two Implementations

### Dev — IMemoryCache (no Docker needed)

```csharp
// Shared.Infrastructure/Caching/InMemoryCacheService.cs
namespace OneNex.Shared.Infrastructure.Caching;

internal sealed class InMemoryCacheService(IMemoryCache cache)
    : ICacheService
{
    private static readonly TimeSpan DefaultTtl = TimeSpan.FromMinutes(5);

    public async Task<T> GetOrSetAsync<T>(
        string key, Func<Task<T>> factory, TimeSpan? ttl = null,
        CancellationToken ct = default)
    {
        if (cache.TryGetValue(key, out T? cached) && cached is not null)
            return cached;

        var value = await factory();

        cache.Set(key, value, ttl ?? DefaultTtl);

        return value;
    }

    public Task RemoveAsync(string key, CancellationToken ct = default)
    {
        cache.Remove(key);
        return Task.CompletedTask;
    }

    public Task SetAsync<T>(string key, T value, TimeSpan? ttl = null,
        CancellationToken ct = default)
    {
        cache.Set(key, value, ttl ?? DefaultTtl);
        return Task.CompletedTask;
    }

    public Task<bool> ExistsAsync(string key, CancellationToken ct = default)
        => Task.FromResult(cache.TryGetValue(key, out _));
}
```

### Production — Valkey (via StackExchange.Redis)

```csharp
// Shared.Infrastructure/Caching/ValkeyCacheService.cs
namespace OneNex.Shared.Infrastructure.Caching;

internal sealed class ValkeyCacheService(IConnectionMultiplexer valkey)
    : ICacheService
{
    private static readonly TimeSpan DefaultTtl = TimeSpan.FromMinutes(5);
    private IDatabase Db => valkey.GetDatabase();

    public async Task<T> GetOrSetAsync<T>(
        string key, Func<Task<T>> factory, TimeSpan? ttl = null,
        CancellationToken ct = default)
    {
        var cached = await Db.StringGetAsync(key);

        if (cached.HasValue)
            return JsonSerializer.Deserialize<T>(cached!)!;

        var value    = await factory();
        var json     = JsonSerializer.Serialize(value);

        await Db.StringSetAsync(key, json, ttl ?? DefaultTtl);

        return value;
    }

    public async Task RemoveAsync(string key, CancellationToken ct = default)
        => await Db.KeyDeleteAsync(key);

    public async Task SetAsync<T>(string key, T value, TimeSpan? ttl = null,
        CancellationToken ct = default)
    {
        var json = JsonSerializer.Serialize(value);
        await Db.StringSetAsync(key, json, ttl ?? DefaultTtl);
    }

    public async Task<bool> ExistsAsync(string key, CancellationToken ct = default)
        => await Db.KeyExistsAsync(key);
}
```

---

## Registration — Environment-based Swap

```csharp
// Shared.Infrastructure/DependencyInjection.cs
public static IServiceCollection AddSharedInfrastructure(
    this IServiceCollection services,
    IConfiguration configuration)
{
    // ... other registrations

    if (configuration.GetValue<bool>("Cache:UseInMemory"))
    {
        // Dev — no Docker needed
        services.AddMemoryCache();
        services.AddSingleton<ICacheService, InMemoryCacheService>();
    }
    else
    {
        // Production — Valkey
        var connectionString = configuration.GetConnectionString("Valkey")!;
        services.AddSingleton<IConnectionMultiplexer>(
            ConnectionMultiplexer.Connect(connectionString));
        services.AddSingleton<ICacheService, ValkeyCacheService>();
    }

    return services;
}
```

```json
// appsettings.Development.json
{
  "Cache": {
    "UseInMemory": true
  }
}

// appsettings.json (production)
{
  "Cache": {
    "UseInMemory": false
  },
  "ConnectionStrings": {
    "Valkey": "localhost:6379"
  }
}
```

---

## CacheKeys — No Magic Strings

```csharp
// Shared.Infrastructure/Caching/CacheKeys.cs
namespace OneNex.Shared.Infrastructure.Caching;

/// <summary>All cache keys in one place. No magic strings anywhere else.</summary>
public static class CacheKeys
{
    public static class Auth
    {
        // Permission list per user per business
        public static string Permissions(Guid userId, Guid businessId)
            => $"auth:permissions:{userId}:{businessId}";

        // Suspension flag per user per business
        public static string Suspended(Guid userId, Guid businessId)
            => $"auth:suspended:{userId}:{businessId}";
    }

    public static class Business
    {
        // Business info (loaded every API call — cache heavily)
        public static string Info(Guid businessId)
            => $"business:info:{businessId}";

        // Operating hours (checked frequently for reservation logic)
        public static string Hours(Guid businessId)
            => $"business:hours:{businessId}";
    }

    public static class Stays
    {
        // Room availability grid (changes on booking create/cancel)
        public static string RoomAvailability(Guid businessId, DateOnly date)
            => $"stays:availability:{businessId}:{date:yyyy-MM-dd}";
    }

    public static class Dining
    {
        // Menu items (changes rarely — long TTL ok)
        public static string Menu(Guid businessId)
            => $"dining:menu:{businessId}";

        // Table status map (changes on every order/close — short TTL)
        public static string TableStatus(Guid businessId)
            => $"dining:tables:{businessId}";
    }
}
```

---

## CacheTtl — Centralized TTLs

```csharp
// Shared.Infrastructure/Caching/CacheTtl.cs
namespace OneNex.Shared.Infrastructure.Caching;

/// <summary>All TTLs in one place. Tune here, takes effect everywhere.</summary>
public static class CacheTtl
{
    public static readonly TimeSpan Permissions   = TimeSpan.FromMinutes(5);   // auth:permissions
    // suspended key = no TTL (until manually cleared on un-suspend)

    public static readonly TimeSpan BusinessInfo  = TimeSpan.FromMinutes(30);  // changes rarely
    public static readonly TimeSpan BusinessHours = TimeSpan.FromHours(6);     // changes very rarely

    public static readonly TimeSpan RoomAvailability = TimeSpan.FromMinutes(2); // changes on booking
    public static readonly TimeSpan MenuItems         = TimeSpan.FromHours(1);  // changes on menu edit
    public static readonly TimeSpan TableStatus       = TimeSpan.FromSeconds(30); // real-time-ish
}
```

---

## Handler Usage — GetOrSetAsync Pattern

```csharp
// Query handler — cache transparent to caller
public sealed class GetBusinessInfoQueryHandler(
    IBusinessReadRepository readRepo,
    ICacheService cache,
    ICurrentUser currentUser)
    : IQueryHandler<GetBusinessInfoQuery, BusinessInfoDto>
{
    public async Task<ErrorOr<BusinessInfoDto>> Handle(
        GetBusinessInfoQuery query, CancellationToken ct)
    {
        var key = CacheKeys.Business.Info(currentUser.BusinessId);

        var result = await cache.GetOrSetAsync(
            key,
            factory: () => readRepo.GetBusinessInfoAsync(currentUser.BusinessId, ct),
            ttl: CacheTtl.BusinessInfo,
            ct: ct);

        if (result is null)
            return Errors.Business.NotFound(currentUser.BusinessId);

        return result;
    }
}
```

```csharp
// Command handler — invalidate cache after mutation
public sealed class UpdateBusinessSettingsCommandHandler(
    IBusinessWriteRepository writeRepo,
    ICacheService cache,
    ICurrentUser currentUser)
    : ICommandHandler<UpdateBusinessSettingsCommand>
{
    public async Task<ErrorOr<Success>> Handle(
        UpdateBusinessSettingsCommand command, CancellationToken ct)
    {
        var business = await writeRepo.GetByIdAsync(
            currentUser.BusinessId, ct);

        if (business is null)
            return Errors.Business.NotFound(currentUser.BusinessId);

        business.UpdateSettings(command.Settings);
        await writeRepo.SaveChangesAsync(ct);

        // Invalidate — next read will reload from DB
        await cache.RemoveAsync(
            CacheKeys.Business.Info(currentUser.BusinessId), ct);

        return Result.Success;
    }
}
```

---

## Permission Cache — Auth Flow

The `PermissionAuthorizationHandler` uses `ICacheService` (not raw Redis):

```csharp
// Shared.Infrastructure/Auth/PermissionAuthorizationHandler.cs
public sealed class PermissionAuthorizationHandler(
    ICacheService cache,
    IPermissionService permissionService,
    ICurrentUser currentUser)
    : AuthorizationHandler<PermissionRequirement>
{
    protected override async Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        PermissionRequirement requirement)
    {
        if (!currentUser.IsAuthenticated)
        {
            context.Fail();
            return;
        }

        // Suspension check — fast path (key exists = suspended)
        var suspendedKey = CacheKeys.Auth.Suspended(
            currentUser.UserId, currentUser.BusinessId);

        if (await cache.ExistsAsync(suspendedKey))
        {
            context.Fail();
            return;
        }

        // Permission check via GetOrSetAsync — loads from DB on cache miss
        var permKey = CacheKeys.Auth.Permissions(
            currentUser.UserId, currentUser.BusinessId);

        var permissions = await cache.GetOrSetAsync(
            permKey,
            factory: () => permissionService.GetPermissionsAsync(
                currentUser.UserId, currentUser.BusinessId),
            ttl: CacheTtl.Permissions);

        if (permissions.Contains(requirement.Permission))
            context.Succeed(requirement);
        else
            context.Fail();
    }
}
```

Permission change flow (Membership module):
```csharp
// When admin changes staff permissions:
await _membershipRepo.UpdateRolePermissionsAsync(...);
await _cache.RemoveAsync(
    CacheKeys.Auth.Permissions(staffUserId, businessId));
// Next request → cache miss → reload → new permissions active instantly ✅
```

Suspension flow:
```csharp
// When admin suspends staff:
await _cache.SetAsync(
    CacheKeys.Auth.Suspended(staffUserId, businessId),
    true,
    ttl: null);  // no TTL — until manually cleared on un-suspend
// Next request → ExistsAsync = true → 403 immediately ✅
```

---

## What to Cache vs Not Cache

### Cache ✅

| Data | Key | TTL | Reason |
|---|---|---|---|
| User permissions | `auth:permissions:{u}:{b}` | 5 min | Checked every protected request |
| Suspension flag | `auth:suspended:{u}:{b}` | No TTL | Instant block needed |
| Business info | `business:info:{b}` | 30 min | Loaded every API call |
| Business hours | `business:hours:{b}` | 6 hr | Changes rarely |
| Menu items | `dining:menu:{b}` | 1 hr | Changes on menu edit only |
| Room availability | `stays:availability:{b}:{date}` | 2 min | Changes on booking create/cancel |

### Never Cache ❌

| Data | Reason |
|---|---|
| Financial data (invoices, folio) | Stale = wrong charge. Always DB. |
| Order status | Real-time accuracy needed |
| Inventory counts | Consistency critical — transaction boundary |
| User passwords / tokens | Security risk |
| Data from other business | business_id isolation must be in key always |

---

## Valkey Local Setup — Docker (30 seconds)

```bash
# Start Valkey
docker run -d --name valkey -p 6379:6379 valkey/valkey:latest

# Test
docker exec -it valkey valkey-cli ping
# → PONG

# Stop
docker stop valkey
```

```json
// appsettings.Development.json — switch from InMemory to Valkey
{
  "Cache": {
    "UseInMemory": false
  },
  "ConnectionStrings": {
    "Valkey": "localhost:6379"
  }
}
```

---

## Dev Strategy — Progression

```
Phase 1 — Start coding (UseInMemory = true)
  IMemoryCache under the hood.
  No Docker. Works immediately.
  Handlers: _cache.GetOrSetAsync(...) — same code.

Phase 2 — Integration testing (UseInMemory = false)
  docker run valkey → flip config flag → done.
  Zero handler code change.

Phase 3 — Production
  Managed Valkey (or self-hosted) → same config flag.
  Zero handler code change.
```

---

## Production Hosting — Options

**Key point:** Valkey software = FREE always. Pay பண்றது server/hosting-க்கு மட்டும் — software license-க்கு இல்ல. PostgreSQL மாதிரி.

### Option 1 — Same VPS-ல் Run (Cheapest)

```
உன் App Server (e.g. $20/mo VPS)
  ├── ASP.NET Core app
  └── Valkey (same server-ல் install)     ← extra cost = $0
```

App-கும் Valkey-கும் same machine. Small/medium load-க்கு perfect.

```bash
# Ubuntu server-ல் install
apt install valkey
systemctl enable valkey
systemctl start valkey
```

**Extra cost: $0** — already paying for server.

---

### Option 2 — Upstash (Serverless, Free tier இருக்கு)

```
Upstash Valkey
  Free tier  → 10,000 commands/day          → $0
  Pay-as-go  → $0.20 per 100,000 commands   → very cheap for early stage
  Max cap    → ~$120/mo (unlimited commands) → predictable billing
```

Connection string மட்டும் paste பண்ணா போதும். No server management.

**Good for:** Early stage — traffic குறைவா இருக்கும் போது free, scale ஆனா pay.

---

### Option 3 — AWS ElastiCache for Valkey

```
cache.t4g.micro  → ~$12/mo    (dev/staging)
cache.t4g.small  → ~$24/mo    (small prod)
cache.r7g.large  → ~$130/mo   (serious prod, multi-AZ)
```

AWS native, multi-AZ, auto-failover. Enterprise grade.

**Good for:** Already on AWS, need reliability guarantees at scale.

---

### Recommended Progression for OneNex

```
Dev          → UseInMemory = true        → $0, no Docker needed
Staging      → Upstash free tier         → $0
Production   → Same VPS or Upstash       → $0–$20/mo
Scale up     → AWS ElastiCache           → when load justifies it
```

Upstash start: account create → connection string copy → paste in appsettings → done.
Zero infra management. Pay only when you grow.

---

## Package References

```xml
<!-- Shared.Infrastructure/Shared.Infrastructure.csproj -->
<PackageReference Include="StackExchange.Redis"           Version="2.*" />
<PackageReference Include="Microsoft.Extensions.Caching.Memory" Version="10.*" />
```

No extra Valkey-specific package — StackExchange.Redis works with Valkey as-is.

---

## Rules

```
1.  ICacheService         → inject this everywhere. Never IMemoryCache or IConnectionMultiplexer.
2.  GetOrSetAsync         → always use this pattern. Never get + set separately (race condition).
3.  CacheKeys             → Shared.Infrastructure/Caching/CacheKeys.cs — no magic strings anywhere.
4.  CacheTtl              → Shared.Infrastructure/Caching/CacheTtl.cs — no hardcoded TimeSpan.
5.  Cache invalidation    → on mutation command, after SaveChangesAsync, before return.
6.  business_id in key    → always. Never cache data without scoping to business.
7.  Financial data        → never cache. Always read from DB.
8.  Suspension key        → SetAsync with ttl=null. Never expires until un-suspend clears it.
9.  UseInMemory=true      → dev default. No Docker, no infra, same code.
10. UseInMemory=false     → Valkey. Same ICacheService, different implementation.
```
