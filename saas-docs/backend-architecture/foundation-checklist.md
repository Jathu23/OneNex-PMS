# OneNex — Foundation Checklist

> Status: Living Document
> Last updated: 2026-09-11
> Purpose: Senior architect checklist vs current foundation — what's done, what's missing, how each gap will be solved.

---

## Legend

| Symbol | Meaning |
|--------|---------|
| ✅ | Implemented — code exists and working |
| 🔜 | Partial — interface/decision exists, implementation pending |
| ❌ | Missing — needs to be built before first module |
| ⏭️ | Deferred — intentionally out of V1 scope |
| 📋 | Decided — architectural decision made, code comes at module level |

---

## 0. Non-Functional Targets

> These are not code — they are decisions that every technical choice is measured against.
> Must be documented before first module is built.

| Item | Status | Decision / Plan |
|------|--------|----------------|
| Concurrent users / peak load | ❌ | Define per vertical (Stays, Dining). Informs connection pool size, cache TTL, queue depth. |
| Data volume growth per core table | ❌ | Bookings table: estimate rows/day. Informs partition strategy, archiving policy. |
| Latency targets p50/p95/p99 | ❌ | Critical ops (booking create, availability check) need explicit targets. Informs whether cache is required or optional. |
| Uptime / SLA / RTO / RPO | ❌ | Drives backup frequency, health check alerting thresholds, failover design. |
| Read-heavy vs write-heavy per module | ❌ | Stays availability = read-heavy → aggressive caching. Booking creation = write-heavy → optimistic concurrency. |

---

## 1. Module & Boundary Architecture

| Item | Status | How It's Done |
|------|--------|--------------|
| Boundary enforcement — compiler-level | ✅ | Separate `.csproj` per module (`Identity.Application`, `Identity.Infrastructure`, `Identity.Presentation`). Wrong cross-module reference = compile error. |
| Physical structure | ✅ | `src/Modules/Platform/Identity/` → three sub-projects. Same pattern for every module. |
| Shared vs vertical rule | ✅ | `Shared.Kernel` = domain primitives only (no infra). `Shared.Infrastructure` = cross-cutting tech (logging, cache, behaviors). `Shared.Contracts` = public DTOs between modules. |
| Internal layer convention | ✅ | Every module: `Application` (CQRS handlers) → `Infrastructure` (DbContext, repos) → `Presentation` (controllers). |
| Public contract exposure | 🔜 | `Shared.Contracts` project exists. Empty now — populated as modules expose cross-module DTOs. No module references another module's Application/Infrastructure directly. |

---

## 2. Data Architecture & Persistence

| Item | Status | How It's Done / Plan |
|------|--------|---------------------|
| DB/schema separation | 📋 | Single PostgreSQL DB. Each module gets its own schema via EF `HasDefaultSchema("identity")`, `HasDefaultSchema("stays")` etc. in module DbContext `OnModelCreating`. |
| ORM context — one per module | 📋 | Each module's `Infrastructure` project registers its own `DbContext`. No shared `DbContext`. Module DbContext inherits from EF `DbContext` directly (not a shared base). |
| Repository pattern | ✅ | `IReadRepository` (Dapper — fast reads) and `IWriteRepository<TEntity>` (EF Core — writes) interfaces in `Shared.Kernel`. |
| Read/write path split | 🔜 | Interfaces exist. `IDbConnectionFactory` + `NpgsqlConnectionFactory` + `LoggingDbConnection` implementation pending. Queries use Dapper via `IDbConnectionFactory`. Commands use EF `SaveChangesAsync`. Dev: `LoggingDbConnection` decorator logs SQL + params + elapsed. Prod: raw connection + OpenTelemetry `ActivitySource` spans per repository method. |
| Cross-module FK — ID only | 📋 | Rule: modules reference other modules by ID only (e.g. `GuestId` as `Guid`, never a navigation property). No DB-level FK across schemas. Enforced by convention + code review. |
| Concurrency control | ❌ | **Plan:** Add `byte[] RowVersion` (PostgreSQL `xmin` or explicit timestamp) to `Entity<TId>`. EF `IsRowVersion()` in base configuration. Handlers catch `DbUpdateConcurrencyException` → throw `ConflictException`. |
| Soft-delete | ❌ | **Plan:** Add `ISoftDelete` interface (`bool IsDeleted`, `DateTime? DeletedAt`) to `Shared.Kernel`. Global query filter `HasQueryFilter(e => !e.IsDeleted)` in module DbContext base. Hard-delete remains available for audit/compliance records. |
| Audit trail | 🔜 | `AuditableEntity<TId>` has `CreatedAt`, `UpdatedAt`, `CreatedBy`, `UpdatedBy` fields. **Missing:** `AuditInterceptor` (SaveChanges interceptor that reads `ICurrentUser` and fills these automatically). |
| Default indexing rules | 📋 | Convention: every tenant-scoped entity gets `HasIndex(e => e.BusinessId)`. Every FK gets an index. Common filter columns indexed in `IEntityTypeConfiguration` per entity. |
| Migration approach | 📋 | EF Core migrations per module (`--project Identity.Infrastructure --context IdentityDbContext`). Schema prefix prevents collision. Rollback = EF `Down()` migration. Production: auto-apply on startup disabled — explicit migration script per release. |
| Reference data seeding | 📋 | `IEntityTypeConfiguration.Configure()` includes `HasData()` for lookup/master data. Environment-specific seeds via `app.Environment.IsDevelopment()` guard. |

---

## 3. Multi-Tenancy

| Item | Status | How It's Done / Plan |
|------|--------|---------------------|
| Data isolation model | 📋 | Discriminator column: `BusinessId` (`Guid`) on every tenant-scoped entity. Single DB, single schema per module. Simpler than schema-per-tenant; sufficient for V1 scale. |
| Where scoping is enforced | ❌ | **Plan:** Module DbContext base class (or each module DbContext) overrides `SaveChangesAsync` + applies global query filter `HasQueryFilter(e => e.BusinessId == _currentUser.BusinessId)`. `ICurrentUser` injected into DbContext constructor. Bypass = impossible without removing the filter explicitly (debug only). |
| Tenant context propagation | 🔜 | `ICurrentUser.BusinessId` reads from JWT claim `business_id`. JWT is issued per business login. **Missing:** DbContext integration (filter wired to `ICurrentUser`). |
| Per-tenant resource fairness | ⏭️ | Deferred — V2. If one tenant's query floods the DB, rate-limiting at API gateway level is the V1 mitigation. |

---

## 4. Communication & Data Flow

| Item | Status | How It's Done / Plan |
|------|--------|---------------------|
| In-process cross-module communication | ✅ | MediatR `ISender`. Module A publishes an `ICommand` or `IQuery` — Module B's handler responds. No direct project reference. |
| Sync vs async | ✅ | MediatR: all handlers are `async Task`. Handlers do not block threads. |
| Transactional consistency across modules | 📋 | V1: In-memory `IBackgroundTaskQueue`. Domain events fired after `SaveChangesAsync` commits (in `DomainEventInterceptor`). If handler crashes after commit, event is lost. Acceptable for V1. V2: Outbox pattern (event persisted in same transaction, processed by background worker). |
| Domain vs integration events | 📋 | V1: same `IDomainEvent` + `IBackgroundTaskQueue` mechanism for both. V2: integration events get separate Outbox table + message broker (RabbitMQ / Azure Service Bus). |
| Middleware order | ✅ | `CorrelationId` → `SerilogRequestLogging` → `ExceptionHandling` → `UseAuthentication` → `UseAuthorization` → `MapControllers`. Documented in `Program.cs` with comments. |
| Background job / double-run prevention | ❌ | `BackgroundTaskQueue` is in-memory, `SingleReader=true` (sequential). No Hangfire/Quartz. No idempotency key on domain events. **Plan for V1:** Keep in-memory, document that events are best-effort. **Plan for V2:** Outbox + idempotency key per event. |

---

## 5. Time & Currency

| Item | Status | How It's Done / Plan |
|------|--------|---------------------|
| UTC storage convention | ✅ | `IDateTimeProvider.UtcNow` — all handlers inject this, never `DateTime.Now` or `DateTime.UtcNow` directly. Testable clock. |
| Timezone source of truth | ❌ | **Decision needed:** Property-level timezone (hotel's local time) stored as IANA timezone string (e.g. `"Asia/Colombo"`). Conversions happen at API response layer (server converts UTC → local before sending). Storage always UTC. |
| Date-only vs DateTime semantics | ✅ | `DateRange` value object uses `DateOnly` (check-in/out). `TimeRange` uses `TimeOnly` (operating hours). `DateTime` reserved for audit timestamps only. |
| DST handling for recurring schedules | ❌ | **Plan:** Recurring schedules stored as `TimeOnly` + IANA timezone string. Expansion to actual `DateTime` slots happens at query time using `TimeZoneInfo.ConvertTime`. No pre-expansion stored. |
| Currency storage + rounding | ✅ | `Money` value object: `decimal Amount` + `string Currency` (ISO 4217). Same-currency enforcement on arithmetic. No implicit conversion. Negative amounts rejected at construction. |
| Translated content | ⏭️ | Deferred. V2: separate `_translations` table per entity (entity_id, locale, field, value). |
| Date/number formatting | ⏭️ | Client owns formatting. Server sends raw values (UTC timestamps, numeric amounts). |

---

## 6. API Design & Contracts

| Item | Status | How It's Done / Plan |
|------|--------|---------------------|
| API style | ✅ | REST. `ApiController` base class (inherits `ControllerBase`, sets `[ApiController]`). Route: `[Route("api/v1/[controller]")]`. |
| Standard response shape | ✅ | `ApiResponse<T>` — `{ success, data, message }` for success. `{ success, message, errors? }` for errors. Consistent across all endpoints. |
| Validation | ✅ | Two layers: FluentValidation at pipeline edge (`ValidationBehavior`) + `DomainException` for business rule violations in domain. Both return structured error responses. |
| API versioning | ❌ | `v1` is hardcoded in route string `"api/v1/[controller]"`. **Plan:** Use `Asp.Versioning` package. `[ApiVersion("1.0")]` on controllers. URL segment versioning (`/api/v1/`, `/api/v2/`). Deprecation header on old versions. |
| Pagination / filtering / sorting | 🔜 | `PagedList<T>` exists (`Items`, `TotalCount`, `Page`, `PageSize`, `HasNextPage`). Convention for query parameters (`?page=1&pageSize=20`) not yet formalized. |

---

## 7. Security

| Item | Status | How It's Done / Plan |
|------|--------|---------------------|
| Auth mechanism + token lifetime | 🔜 | `CurrentUser` reads `ClaimsPrincipal` from JWT. JWT issued by Identity module. **Missing:** JWT bearer setup in `DependencyInjection.cs` (`AddAuthentication().AddJwtBearer(...)`). Access token: 15 min. Refresh token: 7 days (stored in Redis, rotated on use). |
| Authorization model | 🔜 | Permission-based. `ICurrentUser` provides `UserId` + `BusinessId`. **Missing:** `RequirePermissionAttribute`, `PermissionRequirement`, `PermissionPolicyProvider`, `PermissionAuthorizationHandler`. Permissions cached in Valkey per user (key: `auth:permissions:{userId}`). |
| Secrets management | ❌ | **Plan:** Dev → .NET User Secrets (`dotnet user-secrets`). Staging/Prod → environment variables injected by CI/CD (no secrets in appsettings files, no secrets in git). JWT signing key never in source control. |
| Column-level encryption | ⏭️ | Deferred. V2 for PII fields (passport numbers etc.) if compliance requires. |
| Input sanitization | ✅ | FluentValidation rejects malformed input at API edge. `DomainException` enforces business rules. EF Core parameterized queries prevent SQL injection. No raw SQL in handlers. |

---

## 8. Performance & Caching

| Item | Status | How It's Done / Plan |
|------|--------|---------------------|
| Caching layers + invalidation | ✅ | `ICacheService` abstraction. Dev: `InMemoryCacheService` (`IMemoryCache`). Prod: `ValkeyCacheService` (StackExchange.Redis). `CacheKeys` static methods for consistent key naming. `CacheTtl` for centralized TTL constants. Invalidation: explicit `RemoveAsync(key)` on write commands. |
| Connection pooling | 📋 | Npgsql default pool: 100 connections. For V1 scale, sufficient. Explicit `Minimum Pool Size` / `Maximum Pool Size` added to connection string when NFT numbers are defined (Section 0). |
| N+1 prevention | 📋 | Convention: queries (read path) use Dapper via `IDbConnectionFactory` — explicit SQL, no lazy loading. JOIN + multi-mapping instead of multiple queries. EF Core (write path) has lazy loading disabled by default. Handlers must not loop over entities calling `.Load()`. |
| Async/await policy | ✅ | `.editorconfig`: `async` methods must end with `Async` (error severity). `ConfigureAwait` suppressed (ASP.NET Core — no sync context issue). |

---

## 9. Resilience & Error Handling

| Item | Status | How It's Done / Plan |
|------|--------|---------------------|
| Global exception handling | ✅ | `ExceptionHandlingMiddleware`. Maps: `ValidationException→422`, `DomainException→400`, `NotFoundException→404`, `ForbiddenException→403`, `ConflictException→409`, everything else→500. Only 500s are `LogError`. |
| Retry / timeout for HTTP clients | ✅ | `AddStandardResilienceHandler()` in `ServiceDefaults` (Aspire). Covers retry with exponential backoff, timeout, circuit breaker for all `HttpClient` instances. |
| Idempotency for booking/payment | ❌ | **Plan:** `Idempotency-Key` header (UUID sent by client). Middleware checks Valkey for existing response with that key. If found → return cached response without re-processing. If not → process, store result in Valkey (TTL 24h), return. Critical for booking creation and payment capture. |
| Graceful degradation | ⏭️ | Deferred. V1: if Valkey is down, cache miss → DB query. Non-critical dependency (email notifications) failures logged, not surfaced to user. |
| Third-party adapter pattern | ❌ | **Plan:** All third-party integrations (payment gateway, email, SMS) behind an interface in `Shared.Kernel` or module kernel. Concrete adapter in module `Infrastructure`. Never call SDK directly from handler. |

---

## 10. Observability

| Item | Status | How It's Done |
|------|--------|--------------|
| Structured logging + required fields | ✅ | Serilog. Every log line carries: `CorrelationId` (middleware), `UserId` + `BusinessId` (LoggingBehavior), `MachineName`, `EnvironmentName`, `ThreadId` (enrichers). Output: Console + rolling File. |
| Core metrics | ✅ | OpenTelemetry (`ServiceDefaults`): ASP.NET Core instrumentation (request count, duration, error rate) + HTTP client instrumentation + runtime metrics (GC, thread pool). OTLP exporter activated via `OTEL_EXPORTER_OTLP_ENDPOINT` env var. |
| Distributed tracing | ✅ | OpenTelemetry tracing with `AddAspNetCoreInstrumentation` + `AddHttpClientInstrumentation`. Same OTLP exporter. Compatible with Jaeger / Grafana Tempo / Azure Monitor. |
| Health checks | ✅ | `/health` (all checks) + `/alive` (liveness only). Registered via `ServiceDefaults.AddDefaultHealthChecks()`. Module-specific checks (DB connectivity) added per module. |

---

## 11. Configuration & Environments

| Item | Status | How It's Done / Plan |
|------|--------|---------------------|
| Config per environment | ✅ | `appsettings.json` (base) + `appsettings.Development.json` (overrides). Dev: `Cache:UseInMemory=true`, EF SQL logging `Information`. Prod: Valkey, file sink only. |
| Per-tenant configurability | ⏭️ | Deferred. V2: tenant config stored in DB, cached in Valkey. |
| Feature flags | ⏭️ | Deferred. Simple env-var toggle if needed for V1. |

---

## 12. Testing Strategy

| Item | Status | Plan |
|------|--------|------|
| Test pyramid shape | ❌ | **Plan:** Unit tests for domain logic (value objects, entities, domain services) — pure, no infra. Integration tests for handlers (real Postgres via Testcontainers, no mocks). E2E tests for critical flows (booking creation → confirmation) via HTTP. |
| Integration test — real DB | ❌ | **Plan:** Testcontainers (PostgreSQL container spun up per test run). Each test class gets a clean schema via EF `EnsureCreated()` + seeded data. No in-memory DB — catches real SQL issues. |
| Test data strategy | ❌ | **Plan:** Builder pattern per aggregate (e.g. `BookingBuilder`). No shared fixtures — each test is self-contained. |

---

## 14. Coding Standards

| Item | Status | How It's Done |
|------|--------|--------------|
| Naming conventions | ✅ | `.editorconfig`: interfaces `I` prefix (error), private fields `_` prefix (error), async methods `Async` suffix (error), constants PascalCase (error). |
| DI lifetime conventions | ✅ | Documented in `DependencyInjection.cs` comments: Singleton = stateless shared. Scoped = per-request (ICurrentUser). Transient = stateless per-use (behaviors). |
| Nullable reference types | ✅ | `<Nullable>enable</Nullable>` in all projects. CS8600–CS8625 treated as errors. `TreatWarningsAsErrors=true` in `Directory.Build.props`. |
| PR / definition-of-done | ❌ | **Plan:** PR requires: 0 build warnings, all new handlers have a corresponding integration test, no direct DbContext access from Presentation layer, migration script included if schema changes. |

---

## 16. File / Media / Search / Backup / Compliance

| Item | Status | Plan |
|------|--------|------|
| File/media + CDN | ⏭️ | Deferred. V2: Azure Blob / S3 + CDN. Interface in module kernel. |
| Search architecture | ⏭️ | V1: PostgreSQL full-text search (`to_tsvector`). V2: Meilisearch or Typesense if needed. |
| Backup / restore | ❌ | Infrastructure concern — not in application code. Plan: daily automated snapshot (managed DB service), weekly restore drill, RTO/RPO from Section 0. |
| Data residency | ⏭️ | Deferred. V1: single region. |
| Data retention / deletion | ❌ | **Plan:** Soft-delete for business data. Hard-delete (GDPR right-to-erasure) via explicit `PurgeAsync` command per module that bypasses soft-delete filter. Retention policy (e.g. archive bookings > 2 years) as background job. |

---

## 17. Scalability & Dependency Hygiene

| Item | Status | How It's Done / Plan |
|------|--------|---------------------|
| Stateless-by-design | ✅ | JWT for identity (no server session). Valkey for distributed cache (no `IMemoryCache` in prod). `IBackgroundTaskQueue` is in-process only — acceptable for V1 monolith. If horizontally scaled before Outbox: switch to Valkey-backed queue. |
| Shared library versioning | 📋 | All modules reference `Shared.Kernel` / `Shared.Infrastructure` via project reference (same solution). No NuGet packaging needed until modules split into separate repos. Breaking changes in Shared trigger all-module rebuild — caught at compile time. |

---

## Priority Order — What to Build Next

Sorted by "blocks first module" dependency:

```
1.  ❌ JWT auth setup in DependencyInjection.cs          — blocks every protected endpoint
2.  ❌ PermissionAuthorizationHandler + Attribute         — blocks authorization
3.  ❌ ISoftDelete + global query filter                  — blocks any entity that needs soft-delete
4.  ❌ AuditInterceptor                                   — blocks CreatedBy/UpdatedBy being filled
5.  ❌ IDbConnectionFactory + NpgsqlConnectionFactory + LoggingDbConnection — blocks all Dapper read queries and dev SQL visibility
6.  ❌ Idempotency middleware                             — blocks booking/payment endpoints
7.  ❌ Concurrency token (RowVersion on Entity<TId>)      — blocks double-booking prevention
8.  ❌ API versioning (Asp.Versioning package)            — blocks stable public API
9.  ❌ Test project setup (Testcontainers)                — blocks integration tests
10. ❌ Secrets management convention (user-secrets)       — blocks secure local dev
```
