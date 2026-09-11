# OneNex — API Conventions & Error Handling

> Status: Living Document  
> Last updated: 2026-09-07  
> Covers: ApiResponse wrapper, ErrorOr pattern, ProblemDetails, ExceptionMiddleware

---

## Why a Custom Response Wrapper?

Raw controller returns (`Ok(data)`, `NotFound()`) — every endpoint different format. Client-side always guess பண்ணணும்.

OneNex-ல் **every response same shape**:

```json
// Success
{
  "success": true,
  "message": null,
  "data": { "bookingId": "3fa85f64-...", "status": "Confirmed" }
}

// Success with message
{
  "success": true,
  "message": "Booking confirmed successfully",
  "data": { "bookingId": "3fa85f64-..." }
}

// Error
{
  "success": false,
  "message": "Booking '3fa85f64' was not found.",
  "data": null
}

// Validation Error — multiple field errors
{
  "success": false,
  "message": "One or more validation errors occurred.",
  "data": null,
  "errors": {
    "checkInDate": ["Check-in date must be in the future"],
    "roomId": ["Room ID is required"],
    "nights": ["Must book at least 1 night"]
  }
}
```

Client-side `result.success` check பண்ணா போதும் — consistent always.

---

## ApiResponse — Wrapper Classes

```csharp
// Shared.Contracts/Responses/ApiResponse.cs
namespace OneNex.Shared.Contracts.Responses;

/// <summary>
/// Standard success/error envelope for all API responses.
/// Client always gets same shape — check success flag first.
/// </summary>
public sealed class ApiResponse<T>
{
    public bool                                    Success { get; init; }
    public string?                                 Message { get; init; }
    public T?                                      Data    { get; init; }
    public IReadOnlyDictionary<string, string[]>?  Errors  { get; init; }

    private ApiResponse() { }

    public static ApiResponse<T> Ok(T data, string? message = null)
        => new() { Success = true, Data = data, Message = message };

    public static ApiResponse<T> Fail(string message,
        IReadOnlyDictionary<string, string[]>? errors = null)
        => new() { Success = false, Message = message, Errors = errors };
}

/// <summary>
/// For commands that return no data (cancel, delete, confirm etc.)
/// </summary>
public sealed class ApiResponse
{
    public bool                                    Success { get; init; }
    public string?                                 Message { get; init; }
    public IReadOnlyDictionary<string, string[]>?  Errors  { get; init; }

    private ApiResponse() { }

    public static ApiResponse Ok(string? message = null)
        => new() { Success = true, Message = message };

    public static ApiResponse Fail(string message,
        IReadOnlyDictionary<string, string[]>? errors = null)
        => new() { Success = false, Message = message, Errors = errors };
}
```

---

## ErrorOr — Internal Result Pattern

### Why ErrorOr?

```csharp
// ❌ Exception-based — caller கிட்ட contract இல்ல
public async Task<Booking> Handle(CreateBookingCommand cmd)
{
    // throws NotFoundException, DomainException, ConflictException ...
    // caller-ku guess panna venum
}

// ✅ ErrorOr — return type-லயே contract தெரியும்
public async Task<ErrorOr<BookingId>> Handle(CreateBookingCommand cmd)
{
    // success → BookingId
    // failure → one or more Error objects
    // compiler-enforced — miss பண்ண முடியாது
}
```

ErrorOr = **internal** pattern. Handler → Controller-க்கு போகும் வரை. Controller-ல் `ApiResponse`-ஆ convert ஆகும்.

---

### Error Types → HTTP Status → ApiResponse

| ErrorOr Type | HTTP Status | When to use |
|---|---|---|
| `Error.NotFound` | 404 | Entity கிடைக்கல |
| `Error.Conflict` | 409 | Already exists, overlap |
| `Error.Validation` | 422 | Input invalid |
| `Error.Forbidden` | 403 | Permission இல்ல |
| `Error.Failure` | 400 | Business rule violation |
| `Error.Unexpected` | 500 | System error |

---

### Static Error Classes — OneNex Pattern

Magic strings தவிர்க்க, errors centralize பண்ணுவோம்:

```csharp
// Stays.Application/Errors/Errors.Booking.cs
namespace OneNex.Stays.Application.Errors;

public static partial class Errors
{
    public static class Booking
    {
        public static Error NotFound(Guid id) =>
            Error.NotFound(
                code:        "Booking.NotFound",
                description: $"Booking '{id}' was not found.");

        public static readonly Error AlreadyCheckedIn =
            Error.Conflict(
                code:        "Booking.AlreadyCheckedIn",
                description: "Booking is already checked in.");

        public static readonly Error CannotCancelCheckedIn =
            Error.Failure(
                code:        "Booking.CannotCancel",
                description: "Cannot cancel a booking that is already checked in.");

        public static readonly Error RoomNotAvailable =
            Error.Conflict(
                code:        "Booking.RoomNotAvailable",
                description: "Room is not available for the selected dates.");
    }

    public static class Room
    {
        public static Error NotFound(Guid id) =>
            Error.NotFound(
                code:        "Room.NotFound",
                description: $"Room '{id}' was not found.");

        public static readonly Error UnderMaintenance =
            Error.Conflict(
                code:        "Room.UnderMaintenance",
                description: "Room is currently under maintenance.");
    }

    public static class Guest
    {
        public static Error NotFound(Guid id) =>
            Error.NotFound(
                code:        "Guest.NotFound",
                description: $"Guest '{id}' was not found.");

        public static readonly Error EmailAlreadyExists =
            Error.Conflict(
                code:        "Guest.EmailExists",
                description: "A guest with this email already exists.");
    }
}
```

---

### Handler — Return ErrorOr\<T\>

```csharp
// Stays.Application/Features/Bookings/Commands/CancelBooking/CancelBookingCommandHandler.cs
public sealed class CancelBookingCommandHandler(
    IBookingWriteRepository writeRepo,
    ICurrentUser currentUser,
    IDateTimeProvider clock) : ICommandHandler<CancelBookingCommand>
{
    public async Task<ErrorOr<Success>> Handle(
        CancelBookingCommand command, CancellationToken ct)
    {
        // Not found → ErrorOr error (no throw)
        var booking = await writeRepo.FindByIdAsync(command.BookingId, ct);
        if (booking is null)
            return Errors.Booking.NotFound(command.BookingId.Value);

        // Permission check → ErrorOr error
        if (booking.BusinessId != currentUser.BusinessId)
            return Error.Forbidden();

        // Domain method → may throw DomainException (middleware catches)
        booking.Cancel(command.Reason, clock.UtcNow);

        await writeRepo.SaveChangesAsync(ct);
        return Result.Success;
    }
}

// Query handler — returns data
public sealed class GetBookingQueryHandler(
    IBookingReadRepository readRepo) : IQueryHandler<GetBookingQuery, BookingDetailDto>
{
    public async Task<ErrorOr<BookingDetailDto>> Handle(
        GetBookingQuery query, CancellationToken ct)
    {
        var booking = await readRepo.FindByIdAsync(query.BookingId, ct);
        if (booking is null)
            return Errors.Booking.NotFound(query.BookingId.Value);

        return booking;   // implicit conversion to ErrorOr<BookingDetailDto>
    }
}
```

---

## ApiController Base — ErrorOr → ApiResponse

```csharp
// Shared.Infrastructure/Controllers/ApiController.cs
namespace OneNex.Shared.Infrastructure.Controllers;

[ApiController]
public abstract class ApiController : ControllerBase
{
    // Command — no return data
    protected IActionResult Match(ErrorOr<Success> result, string? successMessage = null)
        => result.Match(
            _ => Ok(ApiResponse.Ok(successMessage)),
            errors => Problem(errors));

    // Query / Command — with return data
    protected IActionResult Match<T>(ErrorOr<T> result, string? successMessage = null)
        => result.Match(
            value  => Ok(ApiResponse<T>.Ok(value, successMessage)),
            errors => Problem(errors));

    // Command → 201 Created
    protected IActionResult MatchCreated<T>(
        ErrorOr<T> result,
        string actionName,
        Func<T, object> routeValues)
        => result.Match(
            value  => CreatedAtAction(
                          actionName,
                          routeValues(value),
                          ApiResponse<T>.Ok(value)),
            errors => Problem(errors));

    // Convert ErrorOr errors → ApiResponse fail
    private IActionResult Problem(List<Error> errors)
    {
        if (errors.Count == 0)
            return StatusCode(500, ApiResponse.Fail("An unexpected error occurred."));

        var firstError = errors[0];

        var statusCode = firstError.Type switch
        {
            ErrorType.NotFound   => StatusCodes.Status404NotFound,
            ErrorType.Conflict   => StatusCodes.Status409Conflict,
            ErrorType.Validation => StatusCodes.Status422UnprocessableEntity,
            ErrorType.Forbidden  => StatusCodes.Status403Forbidden,
            _                    => StatusCodes.Status400BadRequest
        };

        return StatusCode(statusCode,
            ApiResponse.Fail(firstError.Description));
    }
}
```

---

## Controller — Clean Usage

```csharp
// Stays.WebAPI/Controllers/BookingsController.cs
[Route("api/v1/bookings")]
public sealed class BookingsController(ISender sender) : ApiController
{
    // GET api/v1/bookings/{id}
    [HttpGet("{id:guid}")]
    public async Task<IActionResult> GetById(Guid id, CancellationToken ct)
    {
        var result = await sender.Send(new GetBookingQuery(new BookingId(id)), ct);
        return Match(result);
    }

    // POST api/v1/bookings
    [HttpPost]
    public async Task<IActionResult> Create(CreateBookingRequest request, CancellationToken ct)
    {
        var command = new CreateBookingCommand(
            new RoomId(request.RoomId),
            new GuestId(request.GuestId),
            DateRange.Of(request.CheckIn, request.CheckOut));

        var result = await sender.Send(command, ct);

        // 201 Created with Location header
        return MatchCreated(result,
            actionName:  nameof(GetById),
            routeValues: id => new { id });
    }

    // POST api/v1/bookings/{id}/cancel
    [HttpPost("{id:guid}/cancel")]
    public async Task<IActionResult> Cancel(Guid id, CancelBookingRequest request, CancellationToken ct)
    {
        var command = new CancelBookingCommand(new BookingId(id), request.Reason);
        var result  = await sender.Send(command, ct);

        return Match(result, successMessage: "Booking cancelled successfully.");
    }

    // DELETE api/v1/bookings/{id}
    [HttpDelete("{id:guid}")]
    public async Task<IActionResult> Delete(Guid id, CancellationToken ct)
    {
        var result = await sender.Send(new DeleteBookingCommand(new BookingId(id)), ct);
        return Match(result);
    }
}
```

---

## ExceptionHandlingMiddleware — Safety Net

ErrorOr cover பண்ணாத errors (DomainException, unexpected) இங்க catch ஆகும்:

```csharp
// Shared.Infrastructure/Middleware/ExceptionHandlingMiddleware.cs
namespace OneNex.Shared.Infrastructure.Middleware;

public sealed class ExceptionHandlingMiddleware(
    RequestDelegate next,
    ILogger<ExceptionHandlingMiddleware> logger)
{
    public async Task InvokeAsync(HttpContext context)
    {
        try
        {
            await next(context);
        }
        catch (ValidationException ex)
        {
            // FluentValidation failures → 422 with field errors
            logger.LogWarning("Validation failed: {Errors}", ex.Errors);

            context.Response.StatusCode  = StatusCodes.Status422UnprocessableEntity;
            context.Response.ContentType = "application/json";

            await context.Response.WriteAsJsonAsync(
                ApiResponse.Fail("One or more validation errors occurred.", ex.Errors));
        }
        catch (DomainException ex)
        {
            // Domain invariant violations → 400
            logger.LogWarning(ex, "Domain rule violated");

            context.Response.StatusCode  = StatusCodes.Status400BadRequest;
            context.Response.ContentType = "application/json";

            await context.Response.WriteAsJsonAsync(
                ApiResponse.Fail(ex.Message));
        }
        catch (ForbiddenException ex)
        {
            context.Response.StatusCode  = StatusCodes.Status403Forbidden;
            context.Response.ContentType = "application/json";

            await context.Response.WriteAsJsonAsync(
                ApiResponse.Fail(ex.Message));
        }
        catch (Exception ex)
        {
            // Unexpected errors → 500, hide internal details from client
            logger.LogError(ex, "Unhandled exception");

            context.Response.StatusCode  = StatusCodes.Status500InternalServerError;
            context.Response.ContentType = "application/json";

            await context.Response.WriteAsJsonAsync(
                ApiResponse.Fail("An unexpected error occurred. Please try again later."));
        }
    }
}

// Registration extension
public static class MiddlewareExtensions
{
    public static IApplicationBuilder UseExceptionHandling(this IApplicationBuilder app)
        => app.UseMiddleware<ExceptionHandlingMiddleware>();
}
```

---

## Two Error Paths — When Each Fires

```
Path 1 — ErrorOr (Handler level)
─────────────────────────────────
Used for: expected business outcomes
  • "Booking not found"       → return Errors.Booking.NotFound(id)
  • "Room already booked"     → return Errors.Booking.RoomNotAvailable
  • "No permission"           → return Error.Forbidden()

Flow: Handler → ErrorOr<T> → Controller.Match() → ApiResponse (4xx)


Path 2 — Exception (Domain / Infrastructure level)
────────────────────────────────────────────────────
Used for: invariant violations & unexpected errors
  • DomainException    → Money negative, date invalid    → 400
  • ValidationException → FluentValidation pipeline fail → 422
  • ForbiddenException  → auth middleware                → 403
  • Exception           → DB down, null ref etc.         → 500

Flow: throw → bubble up → ExceptionHandlingMiddleware → ApiResponse (4xx/5xx)
```

---

## Real Response Examples

### GET /api/v1/bookings/abc — Found

```json
HTTP 200 OK
{
  "success": true,
  "message": null,
  "data": {
    "bookingId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
    "guestName": "Arun Kumar",
    "roomNumber": "101",
    "checkIn": "2026-10-15",
    "checkOut": "2026-10-18",
    "nights": 3,
    "totalAmount": "450.00 USD",
    "status": "Confirmed"
  }
}
```

### GET /api/v1/bookings/xyz — Not Found

```json
HTTP 404 Not Found
{
  "success": false,
  "message": "Booking 'xyz' was not found.",
  "data": null
}
```

### POST /api/v1/bookings — Validation Failed

```json
HTTP 422 Unprocessable Entity
{
  "success": false,
  "message": "One or more validation errors occurred.",
  "data": null,
  "errors": {
    "checkInDate": ["Check-in date must be in the future"],
    "roomId": ["Room ID is required"],
    "nights": ["Must book at least 1 night"]
  }
}
```

### POST /api/v1/bookings — Room Not Available

```json
HTTP 409 Conflict
{
  "success": false,
  "message": "Room is not available for the selected dates.",
  "data": null
}
```

### POST /api/v1/bookings/{id}/cancel — Success

```json
HTTP 200 OK
{
  "success": true,
  "message": "Booking cancelled successfully.",
  "data": null
}
```

### POST /api/v1/bookings — Created

```json
HTTP 201 Created
Location: /api/v1/bookings/3fa85f64-...
{
  "success": true,
  "message": null,
  "data": { "bookingId": "3fa85f64-..." }
}
```

---

## Registration — Program.cs

```csharp
// WebAPI/Program.cs
var app = builder.Build();

// First middleware — catches everything below
app.UseExceptionHandling();

app.UseAuthentication();
app.UseAuthorization();

app.MapControllers();
```

---

## Full Request → Response Flow

```
HTTP Request
    │
    ▼
ExceptionHandlingMiddleware    ← wraps everything, catches unhandled throws
    │
    ▼
Auth Middleware (JWT validate)
    │
    ▼
Controller Action
    │
    ▼
sender.Send(command)
    │
    ├── ValidationBehavior     → FluentValidation fail → throw ValidationException
    │                                                  → middleware catches → 422 ApiResponse
    ├── LoggingBehavior        → log request
    └── TransactionBehavior    → begin DB transaction
    │
    ▼
CommandHandler.Handle()
    │
    ├── FindByIdAsync → null?         → return Error (ErrorOr path)
    ├── Permission check fail?        → return Error.Forbidden (ErrorOr path)
    ├── booking.Cancel()
    │     └── DomainException?        → throw (middleware catches → 400 ApiResponse)
    └── SaveChangesAsync
    │
    ▼
return ErrorOr<Success>
    │
    ▼
Controller.Match(result)
    ├── Success → Ok(ApiResponse.Ok("Booking cancelled successfully."))   → 200
    └── Error   → StatusCode(4xx, ApiResponse.Fail(message))              → 4xx
    │
    ▼
HTTP Response — always ApiResponse shape ✅
```

---

## End-to-End Walkthrough — POST /api/v1/bookings/{id}/cancel

ஒரு real API call எப்படி start to finish வருது என்று step-by-step பாக்கலாம்.

---

### Step 1 — Client Sends Request

```http
POST /api/v1/bookings/3fa85f64-5717-4562-b3fc-2c963f66afa6/cancel
Authorization: Bearer eyJhbGciOiJSUzI1NiJ9...
Content-Type: application/json

{
  "reason": "Guest changed plans"
}
```

---

### Step 2 — ExceptionHandlingMiddleware Wraps

Request enter ஆகும் போதே middleware-ல் wrap ஆகும். இனி எந்த layer-லயும் unhandled exception வந்தாலும் இங்க catch ஆகும்.

```
try {
    await next(context);   // ← everything below runs inside this
} catch { ... }
```

---

### Step 3 — JWT Auth Middleware

`Authorization: Bearer ...` validate ஆகும்.

```
Token valid? → claims extract:
  - user_id:     "a1b2c3d4-..."
  - business_id: "f9e8d7c6-..."
  - name:        "Arun Kumar"

ICurrentUser ready for the request scope.
```

Token invalid → 401 Unauthorized உடனே return. Handler-க்கே போகாது.

---

### Step 4 — Controller Receives

```csharp
public async Task<IActionResult> Cancel(
    Guid id,                      // 3fa85f64-... (from URL)
    CancelBookingRequest request, // { Reason = "Guest changed plans" } (from body)
    CancellationToken ct)
{
    var command = new CancelBookingCommand(new BookingId(id), request.Reason);
    var result  = await sender.Send(command, ct);   // → MediatR pipeline

    return Match(result, successMessage: "Booking cancelled successfully.");
}
```

Controller job: request → command, send → match. Business logic zero.

---

### Step 5 — MediatR Pipeline Behaviors

`sender.Send(command)` → behaviors in order:

**LoggingBehavior** fires first:
```
→ Log: "Handling CancelBookingCommand | BookingId: 3fa85f64 | UserId: a1b2c3d4"
```

**ValidationBehavior** fires next:
```
CancelBookingCommandValidator runs:
  ✅ Reason not empty
  ✅ Reason length ≤ 500 chars
→ Validation passed, continue
```

If validation failed here → `throw ValidationException` → ExceptionHandlingMiddleware catches → **422 returned immediately, handler never runs.**

**TransactionBehavior** fires last:
```
ICommand detected → BEGIN DB TRANSACTION
→ Handler runs inside transaction
```

---

### Step 6 — Handler Executes

```csharp
public async Task<ErrorOr<Success>> Handle(
    CancelBookingCommand command, CancellationToken ct)
{
    // 1. Fetch from DB (EF Core, change tracking ON)
    var booking = await writeRepo.FindByIdAsync(command.BookingId, ct);

    // 2. Not found check
    if (booking is null)
        return Errors.Booking.NotFound(command.BookingId.Value);
        // → ErrorOr<Success> with Error inside
        // → handler returns here, no exception

    // 3. Business permission check
    if (booking.BusinessId != currentUser.BusinessId)
        return Error.Forbidden();

    // 4. Domain method — business logic inside entity
    booking.Cancel(command.Reason, clock.UtcNow);
    //   → Status       = Cancelled
    //   → CancelledAt  = 2026-09-07 10:30:00 UTC
    //   → _domainEvents.Add(new BookingCancelledEvent(...))

    // 5. Persist
    await context.SaveChangesAsync(ct);
    //   → AuditInterceptor:        UpdatedAt = now, UpdatedBy = userId
    //   → EF:                      UPDATE stays.bookings SET status='Cancelled'...
    //   → COMMIT TRANSACTION
    //   → DomainEventInterceptor:  BookingCancelledEvent → BackgroundQueue

    // 6. Return success
    return Result.Success;
}
```

---

### Step 7 — Back in Controller

```csharp
return Match(result, successMessage: "Booking cancelled successfully.");

// result.IsError = false → success path
// → Ok(ApiResponse.Ok("Booking cancelled successfully."))
// → HTTP 200
```

---

### Step 8 — Client Gets Response

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "success": true,
  "message": "Booking cancelled successfully.",
  "data": null
}
```

---

### What If Something Goes Wrong? — All Failure Scenarios

#### Scenario A — Booking Not Found

Handler:
```csharp
var booking = await writeRepo.FindByIdAsync(command.BookingId, ct);
// → null

return Errors.Booking.NotFound(command.BookingId.Value);
```

Controller `Match()`:
```
result.IsError = true
firstError.Type = ErrorType.NotFound → statusCode = 404
→ StatusCode(404, ApiResponse.Fail("Booking '3fa85f64...' was not found."))
```

Client:
```http
HTTP/1.1 404 Not Found
{
  "success": false,
  "message": "Booking '3fa85f64-...' was not found.",
  "data": null
}
```

---

#### Scenario B — Validation Failed (empty reason)

```http
POST /cancel
{ "reason": "" }
```

ValidationBehavior → FluentValidation:
```
Reason is empty → FAIL
→ throw ValidationException { Errors: { "reason": ["Reason is required"] } }
```

Handler never runs. Exception → ExceptionHandlingMiddleware:

```http
HTTP/1.1 422 Unprocessable Entity
{
  "success": false,
  "message": "One or more validation errors occurred.",
  "data": null,
  "errors": {
    "reason": ["Reason is required"]
  }
}
```

---

#### Scenario C — Domain Rule Violated (already checked in)

Handler → booking.Cancel() inside domain entity:
```csharp
public void Cancel(string reason, DateTime cancelledAt)
{
    if (Status == BookingStatus.CheckedIn)
        throw new DomainException("Cannot cancel a checked-in booking.");
}
```

Exception thrown → TransactionBehavior catches → **ROLLBACK** → exception bubbles → ExceptionHandlingMiddleware:

```http
HTTP/1.1 400 Bad Request
{
  "success": false,
  "message": "Cannot cancel a checked-in booking.",
  "data": null
}
```

---

#### Scenario D — Unexpected Error (DB down)

```csharp
await context.SaveChangesAsync(ct);
// → SqlException: connection refused
```

Exception → TransactionBehavior ROLLBACK → ExceptionHandlingMiddleware:
```
logger.LogError(ex, "Unhandled exception")   // full stack trace logged
→ generic message to client (never expose internals)
```

```http
HTTP/1.1 500 Internal Server Error
{
  "success": false,
  "message": "An unexpected error occurred. Please try again later.",
  "data": null
}
```

---

### All Scenarios Summary

| What happened | Where caught | HTTP | success |
|---|---|---|---|
| Success | — | 200 | true |
| Created | — | 201 | true |
| Not found | Handler (ErrorOr) | 404 | false |
| No permission | Handler (ErrorOr) | 403 | false |
| Conflict / overlap | Handler (ErrorOr) | 409 | false |
| Validation failed | Middleware | 422 | false |
| Domain rule violated | Middleware | 400 | false |
| DB / system error | Middleware | 500 | false |

Client side எப்பவும் `success` check பண்ணா போதும் — true/false. Format மாறாது.

---

## Rules

```
1. Every endpoint returns ApiResponse or ApiResponse<T> — no raw Ok(data)
2. ErrorOr used internally in handlers — never expose to client directly  
3. Static error classes (Errors.Booking.*) — no magic strings in handlers
4. DomainException → domain invariants only (Money, DateRange, business rules inside entity)
5. ErrorOr errors → application decisions (not found, conflict, permission)
6. Middleware → safety net for domain exceptions and unexpected errors
7. 500 errors → log full exception, return generic message to client (no stack trace)
```
