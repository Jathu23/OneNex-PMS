# OneNex — Auth: JWT + Multi-tenant + Permissions

> Status: Living Document
> Last updated: 2026-09-08
> Covers: JWT structure, Redis permission cache, permission system, token generation, refresh tokens

---

## Core Design Decision

```
JWT contains:  user_id + business_id ONLY — no role, no permissions
Permissions:   resolved from Redis cache at request time
```

**Why not permissions in JWT?**

```
JWT-ல் permissions போட்டா:
  Staff role மாத்தினா → old JWT expire ஆகும் வரை old permissions-தான் valid
  Staff suspend பண்ணினா → JWT expire ஆகும் வரை access இருக்கும்
  24h expiry = 24h security gap

Redis-ல் வச்சா:
  Role மாத்தினா → Redis key delete → next request-ல் உடனே new permissions
  Suspend பண்ணினா → Redis flag set → next request-ல் உடனே block
  Instant effect ✅
```

---

## Two Layers of Auth

```
Layer 1 — Authentication   "நீ யாரு?"
  JWT token validate — expired? tampered? valid signature?
  Handled by: JWT Middleware + [Authorize]

Layer 2 — Authorization    "உனக்கு permission இருக்கா?"
  Redis cache-ல் permission list check
  Handled by: [RequirePermission("bookings:cancel")] + PermissionAuthorizationHandler
```

---

## JWT Token Structure — Minimal

```json
{
  "sub":         "a1b2c3d4-...",    ← user_id only
  "business_id": "f9e8d7c6-...",   ← which business
  "name":        "Arun Kumar",
  "email":       "arun@hotel.com",
  "iat":         1725782400,
  "exp":         1725783300         ← 15 min (short — Redis handles permissions)
}
```

No role. No permissions. JWT = identity proof only.

Access token = **15 min** (short — permission changes take effect within 15 min max).

---

## Redis — Permission Cache

```
Key:   permissions:{userId}:{businessId}
Value: ["bookings:create", "bookings:cancel", "bookings:view", "rooms:view"]
TTL:   5 min

Key:   suspended:{userId}:{businessId}
Value: "true"
TTL:   no expiry (until manually cleared on un-suspend)
```

Permission change flow:

```
Admin removes "bookings:cancel" from staff role
        │
        ▼
MembershipModule → update DB
        │
        ▼
Delete Redis key: permissions:{userId}:{businessId}
        │
        ▼
Next request → cache miss → reload from DB → new permissions active ✅
```

Suspension flow:

```
Admin suspends staff
        │
        ▼
SET suspended:{userId}:{businessId} = "true"
        │
        ▼
Next request → suspended key found → 403 immediately ✅
```

---

## Permission System

### Two Role Dimensions (from Membership module)

```
business_role (fixed enum):
  owner | admin | member
  → governs business management (staff, settings)
  → owner = full bypass

operation_role (per operation):
  full | manager | supervisor | staff | viewer
  → governs everything within that operation
  → fully independent from business_role
```

### Permission Constants

```csharp
// Shared.Contracts/Auth/Permissions.cs
public static class Permissions
{
    public static class Stays
    {
        public const string BookingsCreate = "stays:bookings:create";
        public const string BookingsCancel = "stays:bookings:cancel";
        public const string BookingsView   = "stays:bookings:view";
        public const string RoomsView      = "stays:rooms:view";
        public const string RoomsUpdate    = "stays:rooms:update";
        public const string ReportsView    = "stays:reports:view";
    }

    public static class Dining
    {
        public const string OrdersCreate   = "dining:orders:create";
        public const string OrdersVoid     = "dining:orders:void";
        public const string OrdersView     = "dining:orders:view";
        public const string ReportsView    = "dining:reports:view";
    }

    public static class Business
    {
        public const string StaffManage    = "business:staff:manage";
        public const string SettingsUpdate = "business:settings:update";
    }

    public static readonly IReadOnlyList<string> All =
    [
        Stays.BookingsCreate, Stays.BookingsCancel, Stays.BookingsView,
        Stays.RoomsView, Stays.RoomsUpdate, Stays.ReportsView,
        Dining.OrdersCreate, Dining.OrdersVoid, Dining.OrdersView, Dining.ReportsView,
        Business.StaffManage, Business.SettingsUpdate
    ];
}
```

Constants — no magic strings. Typo = compile error.

---

## JWT Authentication Setup

```csharp
// Shared.Infrastructure/DependencyInjection.cs
services
    .AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer           = true,
            ValidateAudience         = true,
            ValidateLifetime         = true,
            ValidateIssuerSigningKey = true,

            ValidIssuer      = configuration["Jwt:Issuer"],
            ValidAudience    = configuration["Jwt:Audience"],
            IssuerSigningKey = new SymmetricSecurityKey(
                Encoding.UTF8.GetBytes(configuration["Jwt:SecretKey"]!))
        };
    });
```

```json
// appsettings.json
{
  "Jwt": {
    "Issuer":         "onenex-identity",
    "Audience":       "onenex-api",
    "SecretKey":      "from-environment-never-hardcode",
    "ExpiryMinutes":  15
  }
}
```

---

## Permission Authorization — Redis-based

### RequirePermission Attribute

```csharp
// Shared.Infrastructure/Auth/RequirePermissionAttribute.cs
public sealed class RequirePermissionAttribute : AuthorizeAttribute
{
    public RequirePermissionAttribute(string permission)
        : base(policy: $"permission:{permission}") { }
}
```

### PermissionPolicyProvider — Dynamic Policy Creation

```csharp
// Shared.Infrastructure/Auth/PermissionPolicyProvider.cs
public sealed class PermissionPolicyProvider(
    IOptions<AuthorizationOptions> options) : IAuthorizationPolicyProvider
{
    private readonly DefaultAuthorizationPolicyProvider _fallback
        = new(options);

    public Task<AuthorizationPolicy?> GetPolicyAsync(string policyName)
    {
        if (policyName.StartsWith("permission:"))
        {
            var permission = policyName["permission:".Length..];
            var policy = new AuthorizationPolicyBuilder()
                .AddRequirements(new PermissionRequirement(permission))
                .Build();

            return Task.FromResult<AuthorizationPolicy?>(policy);
        }

        return _fallback.GetPolicyAsync(policyName);
    }

    public Task<AuthorizationPolicy> GetDefaultPolicyAsync()
        => _fallback.GetDefaultPolicyAsync();

    public Task<AuthorizationPolicy?> GetFallbackPolicyAsync()
        => _fallback.GetFallbackPolicyAsync();
}
```

### PermissionAuthorizationHandler — Redis Check

```csharp
// Shared.Infrastructure/Auth/PermissionAuthorizationHandler.cs
public sealed class PermissionAuthorizationHandler(
    IConnectionMultiplexer redis,
    ICurrentUser currentUser) : AuthorizationHandler<PermissionRequirement>
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

        // Check suspension first — immediate block
        var suspendedKey = $"suspended:{currentUser.UserId}:{currentUser.BusinessId}";
        var isSuspended  = await redis.GetDatabase().KeyExistsAsync(suspendedKey);

        if (isSuspended)
        {
            context.Fail();
            return;
        }

        // Check permission from cache
        var permKey     = $"permissions:{currentUser.UserId}:{currentUser.BusinessId}";
        var cachedPerms = await redis.GetDatabase().StringGetAsync(permKey);

        HashSet<string> permissions;

        if (cachedPerms.HasValue)
        {
            // Cache hit — deserialize
            permissions = JsonSerializer.Deserialize<HashSet<string>>(cachedPerms!)!;
        }
        else
        {
            // Cache miss — load from DB via Membership module
            permissions = await LoadAndCachePermissionsAsync(
                currentUser.UserId,
                currentUser.BusinessId);
        }

        if (permissions.Contains(requirement.Permission))
            context.Succeed(requirement);
        else
            context.Fail();
    }

    private async Task<HashSet<string>> LoadAndCachePermissionsAsync(
        Guid userId, Guid businessId)
    {
        // Call Membership module via IPermissionService (Shared.Contracts)
        var permissions = await _permissionService
            .GetPermissionsAsync(userId, businessId);

        var key   = $"permissions:{userId}:{businessId}";
        var value = JsonSerializer.Serialize(permissions);

        // Cache with 5 min TTL
        await redis.GetDatabase().StringSetAsync(
            key, value, TimeSpan.FromMinutes(5));

        return permissions.ToHashSet();
    }
}

// Requirement record
public sealed record PermissionRequirement(string Permission)
    : IAuthorizationRequirement;
```

### Registration

```csharp
// Shared.Infrastructure/DependencyInjection.cs
services.AddSingleton<IAuthorizationPolicyProvider, PermissionPolicyProvider>();
services.AddScoped<IAuthorizationHandler, PermissionAuthorizationHandler>();
```

---

## ICurrentUser — Reads JWT Claims Only

```csharp
// Shared.Kernel/Primitives/ICurrentUser.cs
public interface ICurrentUser
{
    Guid   UserId          { get; }
    Guid   BusinessId      { get; }
    string Name            { get; }
    bool   IsAuthenticated { get; }
}

// Shared.Infrastructure/Auth/CurrentUser.cs
internal sealed class CurrentUser(IHttpContextAccessor accessor) : ICurrentUser
{
    private ClaimsPrincipal? User => accessor.HttpContext?.User;

    public Guid UserId
        => Guid.Parse(User!.FindFirstValue(ClaimTypes.NameIdentifier)
            ?? throw new ForbiddenException("User not authenticated."));

    public Guid BusinessId
        => Guid.Parse(User!.FindFirstValue("business_id")
            ?? throw new ForbiddenException("Business context missing from token."));

    public string Name
        => User?.FindFirstValue(ClaimTypes.Name) ?? string.Empty;

    public bool IsAuthenticated
        => User?.Identity?.IsAuthenticated ?? false;

    // No HasPermission() here — permission check = Redis, handled by authorization handler
}
```

`ICurrentUser` = JWT claims மட்டும். Permission check = `PermissionAuthorizationHandler` Redis-ல் பண்ணும்.

---

## Controller Usage

```csharp
[ApiController]
[Route("api/v1/stays/bookings")]
[Authorize]
public sealed class BookingsController(ISender sender) : ApiController
{
    [HttpGet]
    [RequirePermission(Permissions.Stays.BookingsView)]
    public async Task<IActionResult> List(...) { }

    [HttpPost]
    [RequirePermission(Permissions.Stays.BookingsCreate)]
    public async Task<IActionResult> Create(...) { }

    [HttpPost("{id:guid}/cancel")]
    [RequirePermission(Permissions.Stays.BookingsCancel)]
    public async Task<IActionResult> Cancel(...) { }
}
```

---

## Token Generation — Identity Module

```csharp
// Identity.Infrastructure/Services/JwtTokenService.cs
public sealed class JwtTokenService(
    IOptions<JwtSettings> settings,
    IDateTimeProvider clock) : ITokenService
{
    private readonly JwtSettings _settings = settings.Value;

    public TokenResult GenerateToken(User user)
    {
        // Minimal claims — no role, no permissions
        var claims = new List<Claim>
        {
            new(ClaimTypes.NameIdentifier, user.Id.Value.ToString()),
            new(ClaimTypes.Name,           user.FullName),
            new(ClaimTypes.Email,          user.Email.Value),
            new("business_id",             user.BusinessId.Value.ToString()),
        };

        var expires = clock.UtcNow.AddMinutes(_settings.ExpiryMinutes);  // 15 min
        var key     = new SymmetricSecurityKey(
                          Encoding.UTF8.GetBytes(_settings.SecretKey));
        var creds   = new SigningCredentials(key, SecurityAlgorithms.HmacSha256);

        var token = new JwtSecurityToken(
            issuer:             _settings.Issuer,
            audience:           _settings.Audience,
            claims:             claims,
            expires:            expires,
            signingCredentials: creds);

        return new TokenResult(
            new JwtSecurityTokenHandler().WriteToken(token),
            expires);
    }
}
```

---

## Refresh Token

```
Access Token  → 15 min (short — Redis handles instant permission revoke)
Refresh Token → 30 days, DB stored (hashed), rotated on use
```

```csharp
// Identity.Domain/Entities/RefreshToken.cs
public sealed class RefreshToken : Entity<RefreshTokenId>
{
    public UserId   UserId        { get; private set; }
    public string   TokenHash     { get; private set; } = null!;  // hashed — never plain
    public DateTime ExpiresAt     { get; private set; }
    public bool     IsRevoked     { get; private set; }
    public bool     IsExpired     => DateTime.UtcNow > ExpiresAt;
    public bool     IsValid       => !IsRevoked && !IsExpired;

    public static RefreshToken Create(UserId userId, DateTime expiresAt, string tokenHash)
        => new()
        {
            Id        = RefreshTokenId.New(),
            UserId    = userId,
            TokenHash = tokenHash,
            ExpiresAt = expiresAt
        };

    public void Revoke() => IsRevoked = true;
}
```

---

## Full Auth Flow

```
POST /api/v1/identity/auth/login
{ "email": "arun@hotel.com", "password": "..." }
        │
        ▼
LoginCommandHandler
  → validate credentials
  → build JWT with {user_id, business_id} only
  → create refresh token → hash → save to DB
  → return { accessToken (15m), refreshToken (30d) }
        │
        ▼
Client stores tokens

──────────────────────────────────────────────

POST /api/v1/stays/bookings/{id}/cancel
Authorization: Bearer eyJhbGci...
        │
        ▼
JWT Middleware
  → signature valid? ✅
  → not expired?     ✅
  → extract {user_id, business_id} → ClaimsPrincipal set
        │
        ▼
[Authorize] → authenticated? ✅
        │
        ▼
[RequirePermission("stays:bookings:cancel")]
        │
        ▼
PermissionAuthorizationHandler
  → check suspended:{userId}:{businessId} → not set ✅
  → check permissions:{userId}:{businessId} → cache hit ✅
  → "stays:bookings:cancel" in list? ✅
  → context.Succeed()
        │
        ▼
Controller → ICurrentUser ready
  currentUser.UserId     → from JWT
  currentUser.BusinessId → from JWT
        │
        ▼
Handler runs → business scoped to BusinessId
```

---

## Rules

```
1.  JWT             → user_id + business_id ONLY. No role. No permissions.
2.  Access token    → 15 min expiry (short — Redis handles instant revoke)
3.  Refresh token   → 30 days, DB stored hashed, rotated on use
4.  Permissions     → Redis cache, 5 min TTL, loaded from Membership module on miss
5.  Suspension      → Redis flag, no TTL, instant block
6.  Permission key  → permissions:{userId}:{businessId}
7.  Suspend key     → suspended:{userId}:{businessId}
8.  Permission change→ delete Redis key → next request reloads
9.  Constants       → Shared.Contracts/Auth/Permissions.cs — no magic strings
10. [Authorize]     → every controller class (default-on)
11. [AllowAnonymous]→ explicit opt-out (login, register only)
12. business_id     → always from JWT — never from request body
13. Middleware order → UseAuthentication → UseAuthorization — never swap
14. SecretKey       → environment variable only. Never in appsettings.json
```
