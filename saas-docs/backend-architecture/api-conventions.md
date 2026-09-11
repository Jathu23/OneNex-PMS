# OneNex — API Conventions

> Status: Living Document
> Last updated: 2026-09-07
> Covers: controller structure, routes, request/response DTOs, HTTP method conventions

---

## Controller vs Minimal API — Decision

OneNex → **Controllers** chosen.

| | Controllers | Minimal API |
|---|---|---|
| Large-scale structure | ✅ Clean grouping | ❌ Gets messy at scale |
| Attribute-based auth | ✅ `[Authorize]`, `[RequirePermission]` | ⚠️ Verbose |
| Scalar / Swagger | ✅ Auto-discovery | ✅ Works but manual |
| Action filters | ✅ Full pipeline | ❌ Limited |
| Best for | Large modular apps | Small APIs, lambdas |

---

## Route Convention

```
api/v{version}/{module}/{resource}

api/v1/stays/bookings
api/v1/stays/bookings/{id}
api/v1/stays/bookings/{id}/cancel
api/v1/stays/bookings/{id}/confirm
api/v1/stays/rooms
api/v1/stays/rooms/{id}/availability

api/v1/dining/reservations
api/v1/dining/reservations/{id}
api/v1/dining/tables
api/v1/dining/tables/{id}/availability

api/v1/membership/members
api/v1/membership/members/{id}/plans
```

Rules:
- `kebab-case` for multi-word resources — `rate-plans`, `check-in`
- Module name in route — future microservice extraction-க்கு ready
- Version = URL segment — `v1`, `v2` (explicit, cacheable)
- State-changing actions = noun/{id}/verb pattern

---

## Request → Command → Response — Data Flow

```
Client JSON Body / Query String
        ↓
Request DTO        (WebAPI layer — thin, just holds incoming data)
        ↓  controller maps
Command / Query    (Application layer — carries typed IDs + intent)
        ↓  handler processes
Response DTO       (Application layer — query result shape)
        ↓  ApiController.Match() wraps
ApiResponse<T>     (to client — always same envelope)
```

**Request DTO** — WebAPI layer, no logic, just raw incoming values:

```csharp
// Stays.WebAPI/Models/Requests/CreateBookingRequest.cs
public sealed record CreateBookingRequest(
    Guid     RoomId,
    Guid     GuestId,
    DateOnly CheckIn,
    DateOnly CheckOut,
    string?  SpecialRequests);

// Stays.WebAPI/Models/Requests/CancelBookingRequest.cs
public sealed record CancelBookingRequest(string Reason);

// Stays.WebAPI/Models/Requests/ListBookingsRequest.cs
public sealed record ListBookingsRequest(
    string? Status     = null,
    int     PageNumber = 1,
    int     PageSize   = 20);
```

**Command** — Application layer, typed IDs, carries intent:

```csharp
// Stays.Application/Features/Bookings/Commands/CreateBooking/CreateBookingCommand.cs
public sealed record CreateBookingCommand(
    RoomId    RoomId,
    GuestId   GuestId,
    DateRange StayPeriod,
    string?   SpecialRequests) : ICommand<BookingId>;
```

**Response DTO** — Application layer, query handlers return this:

```csharp
// Stays.Application/Features/Bookings/Queries/GetBooking/BookingDetailDto.cs
public sealed record BookingDetailDto(
    Guid     BookingId,
    string   GuestName,
    string   RoomNumber,
    DateOnly CheckIn,
    DateOnly CheckOut,
    int      Nights,
    decimal  TotalAmount,
    string   Currency,
    string   Status,
    DateTime CreatedAt);
```

Domain entities never returned to client. Dapper maps directly to DTO.

---

## Controller — Full Example

```csharp
// Stays.WebAPI/Controllers/BookingsController.cs
[ApiController]
[Route("api/v1/stays/bookings")]
[Authorize]
public sealed class BookingsController(
    ISender      sender,
    ICurrentUser currentUser) : ApiController
{
    // GET api/v1/stays/bookings/{id}
    [HttpGet("{id:guid}")]
    public async Task<IActionResult> GetById(Guid id, CancellationToken ct)
    {
        var result = await sender.Send(
            new GetBookingQuery(new BookingId(id)), ct);

        return Match(result);
    }

    // GET api/v1/stays/bookings?status=Confirmed&pageNumber=1&pageSize=20
    [HttpGet]
    public async Task<IActionResult> List(
        [FromQuery] ListBookingsRequest request, CancellationToken ct)
    {
        var result = await sender.Send(
            new ListBookingsQuery(
                currentUser.BusinessId,
                request.Status,
                request.PageNumber,
                request.PageSize), ct);

        return Match(result);
    }

    // POST api/v1/stays/bookings
    [HttpPost]
    public async Task<IActionResult> Create(
        CreateBookingRequest request, CancellationToken ct)
    {
        var command = new CreateBookingCommand(
            new RoomId(request.RoomId),
            new GuestId(request.GuestId),
            DateRange.Of(request.CheckIn, request.CheckOut),
            request.SpecialRequests);

        var result = await sender.Send(command, ct);

        return MatchCreated(result,
            actionName:  nameof(GetById),
            routeValues: id => new { id });
    }

    // POST api/v1/stays/bookings/{id}/confirm
    [HttpPost("{id:guid}/confirm")]
    public async Task<IActionResult> Confirm(Guid id, CancellationToken ct)
    {
        var result = await sender.Send(
            new ConfirmBookingCommand(new BookingId(id)), ct);

        return Match(result, successMessage: "Booking confirmed successfully.");
    }

    // POST api/v1/stays/bookings/{id}/cancel
    [HttpPost("{id:guid}/cancel")]
    public async Task<IActionResult> Cancel(
        Guid id, CancelBookingRequest request, CancellationToken ct)
    {
        var result = await sender.Send(
            new CancelBookingCommand(new BookingId(id), request.Reason), ct);

        return Match(result, successMessage: "Booking cancelled successfully.");
    }

    // POST api/v1/stays/bookings/{id}/check-in
    [HttpPost("{id:guid}/check-in")]
    public async Task<IActionResult> CheckIn(Guid id, CancellationToken ct)
    {
        var result = await sender.Send(
            new CheckInCommand(new BookingId(id)), ct);

        return Match(result, successMessage: "Guest checked in successfully.");
    }

    // DELETE api/v1/stays/bookings/{id}
    [HttpDelete("{id:guid}")]
    public async Task<IActionResult> Delete(Guid id, CancellationToken ct)
    {
        var result = await sender.Send(
            new DeleteBookingCommand(new BookingId(id)), ct);

        return Match(result);
    }
}
```

Controller job: Request → Command, Send, Match. Zero business logic.

---

## HTTP Method Conventions

| Method | Use | Example |
|---|---|---|
| `GET` | Fetch — no side effects | `GET /bookings/{id}` |
| `POST` | Create new resource | `POST /bookings` |
| `POST` | State-changing action | `POST /bookings/{id}/cancel` |
| `PUT` | Full replace | `PUT /rooms/{id}` |
| `PATCH` | Partial update | `PATCH /guests/{id}` |
| `DELETE` | Remove (soft delete) | `DELETE /bookings/{id}` |

**State-changing actions → `POST /resource/{id}/action`**

```
POST /bookings/{id}/confirm      ✅
POST /bookings/{id}/cancel       ✅
POST /bookings/{id}/check-in     ✅
POST /bookings/{id}/check-out    ✅

PATCH /bookings/{id} { "status": "Cancelled" }   ❌ ambiguous, hard to validate
```

---

## Module Folder Structure

```
Stays.WebAPI/
├── Controllers/
│   ├── BookingsController.cs
│   ├── RoomsController.cs
│   └── GuestsController.cs
├── Models/
│   └── Requests/
│       ├── CreateBookingRequest.cs
│       ├── CancelBookingRequest.cs
│       └── ListBookingsRequest.cs
└── Program.cs

Stays.Application/
└── Features/
    └── Bookings/
        ├── Commands/
        │   ├── CreateBooking/
        │   │   ├── CreateBookingCommand.cs
        │   │   ├── CreateBookingCommandHandler.cs
        │   │   └── CreateBookingCommandValidator.cs
        │   └── CancelBooking/
        │       ├── CancelBookingCommand.cs
        │       ├── CancelBookingCommandHandler.cs
        │       └── CancelBookingCommandValidator.cs
        └── Queries/
            ├── GetBooking/
            │   ├── GetBookingQuery.cs
            │   ├── GetBookingQueryHandler.cs
            │   └── BookingDetailDto.cs        ← response DTO here
            └── ListBookings/
                ├── ListBookingsQuery.cs
                ├── ListBookingsQueryHandler.cs
                └── BookingListItemDto.cs
```

Response DTOs → Application layer (Query folder-ல்).
Request DTOs → WebAPI layer (Models/Requests folder-ல்).

---

## Program.cs — Registration

```csharp
// Stays.WebAPI/Program.cs
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers();

builder.Services
    .AddStaysApplication()      // MediatR, validators, behaviors
    .AddStaysInfrastructure();  // DbContext, repositories, interceptors

var app = builder.Build();

app.UseExceptionHandling();   // first — wraps everything
app.UseAuthentication();
app.UseAuthorization();
app.MapControllers();

app.Run();
```

---

## Rules

```
1. Controllers only — no Minimal API
2. Route = api/v1/{module}/{resource} — module name always in route
3. kebab-case URLs — rate-plans, check-in, room-types
4. Request DTOs → WebAPI layer (thin, no logic)
5. Response DTOs → Application layer (query folder, next to handler)
6. Controller → maps request to command/query, sends, matches — no business logic
7. State changes → POST /resource/{id}/action (not PATCH with status field)
8. Domain entities → never returned to client (always DTO)
9. ICurrentUser → inject in controller only when needed for mapping (not for business logic)
```
