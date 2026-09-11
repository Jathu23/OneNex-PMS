# OneNex — Validation Flow

> Status: Living Document
> Last updated: 2026-09-07
> Covers: FluentValidation, ValidationBehavior, domain validation, where each lives

---

## Two Types of Validation — Different Purposes

```
Type 1 — Input Validation (FluentValidation)
  "Is this a valid request to even process?"
  → Format, required fields, length, range, cross-field
  → Runs BEFORE handler — no DB touch
  → Returns 422 with field-level errors

Type 2 — Domain Validation (DomainException / ErrorOr)
  "Can this business operation happen right now?"
  → Business rules, state checks, availability
  → Runs INSIDE handler / domain method — needs DB
  → Returns 400 (domain rule) or 409 (conflict)
```

---

## ValidationBehavior — MediatR Pipeline Step

```csharp
// Shared.Infrastructure/Behaviors/ValidationBehavior.cs
public sealed class ValidationBehavior<TRequest, TResponse>(
    IEnumerable<IValidator<TRequest>> validators)
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : IBaseRequest
{
    public async Task<TResponse> Handle(
        TRequest request,
        RequestHandlerDelegate<TResponse> next,
        CancellationToken ct)
    {
        // No validators registered for this request → skip
        if (!validators.Any())
            return await next();

        // Run all validators in parallel
        var context = new ValidationContext<TRequest>(request);

        var results = await Task.WhenAll(
            validators.Select(v => v.ValidateAsync(context, ct)));

        var failures = results
            .SelectMany(r => r.Errors)
            .Where(f => f is not null)
            .ToList();

        if (failures.Count == 0)
            return await next();   // passed → continue to handler

        // Build field → messages dictionary
        var errors = failures
            .GroupBy(f => f.PropertyName)
            .ToDictionary(
                g => g.Key,
                g => g.Select(e => e.ErrorMessage).ToArray());

        throw new ValidationException(errors);
        // → ExceptionHandlingMiddleware → 422 ApiResponse with errors dict
    }
}
```

---

## Registration

```csharp
// Stays.Application/DependencyInjection.cs
services.AddValidatorsFromAssembly(typeof(ApplicationAssemblyMarker).Assembly);

// MediatR pipeline — order matters
services.AddMediatR(cfg =>
{
    cfg.RegisterServicesFromAssembly(typeof(ApplicationAssemblyMarker).Assembly);
    cfg.AddBehavior(typeof(IPipelineBehavior<,>), typeof(LoggingBehavior<,>));
    cfg.AddBehavior(typeof(IPipelineBehavior<,>), typeof(ValidationBehavior<,>));
    cfg.AddBehavior(typeof(IPipelineBehavior<,>), typeof(TransactionBehavior<,>));
});
```

Pipeline order:
```
Request
  │
  ▼
LoggingBehavior       → log request
  │
  ▼
ValidationBehavior    → validate input (fail fast, no DB, no transaction)
  │
  ▼
TransactionBehavior   → begin transaction (only if validation passed)
  │
  ▼
Handler
```

Validation fail → transaction never opens. DB never touched. Fast failure.

---

## Where Validators Live — Vertical Slice

```
Stays.Application/
└── Features/
    └── Bookings/
        └── Commands/
            ├── CreateBooking/
            │   ├── CreateBookingCommand.cs
            │   ├── CreateBookingCommandHandler.cs
            │   └── CreateBookingCommandValidator.cs   ← same folder
            │
            └── CancelBooking/
                ├── CancelBookingCommand.cs
                ├── CancelBookingCommandHandler.cs
                └── CancelBookingCommandValidator.cs   ← same folder
```

Validator = command-ஓட same folder. Feature-க்கு போனா எல்லாம் ஒரே இடத்துல தெரியும்.

---

## Validator Examples

### CancelBookingCommandValidator

```csharp
public sealed class CancelBookingCommandValidator
    : AbstractValidator<CancelBookingCommand>
{
    public CancelBookingCommandValidator()
    {
        RuleFor(x => x.BookingId.Value)
            .NotEmpty()
            .WithMessage("Booking ID is required.");

        RuleFor(x => x.Reason)
            .NotEmpty()
            .WithMessage("Cancellation reason is required.")
            .MaximumLength(500)
            .WithMessage("Reason cannot exceed 500 characters.");
    }
}
```

### CreateBookingCommandValidator

```csharp
public sealed class CreateBookingCommandValidator
    : AbstractValidator<CreateBookingCommand>
{
    public CreateBookingCommandValidator()
    {
        RuleFor(x => x.RoomId.Value)
            .NotEmpty()
            .WithMessage("Room is required.");

        RuleFor(x => x.GuestId.Value)
            .NotEmpty()
            .WithMessage("Guest is required.");

        RuleFor(x => x.CheckIn)
            .NotEmpty()
            .WithMessage("Check-in date is required.")
            .GreaterThanOrEqualTo(DateOnly.FromDateTime(DateTime.UtcNow))
            .WithMessage("Check-in date must be today or in the future.");

        RuleFor(x => x.CheckOut)
            .NotEmpty()
            .WithMessage("Check-out date is required.")
            .GreaterThan(x => x.CheckIn)
            .WithMessage("Check-out must be after check-in.");

        RuleFor(x => x.Nights)
            .GreaterThan(0)
            .WithMessage("Must book at least 1 night.")
            .LessThanOrEqualTo(365)
            .WithMessage("Cannot book more than 365 nights.");
    }
}
```

### Cross-Field & Conditional Rules

```csharp
// Cross-field
RuleFor(x => x.CheckOut)
    .GreaterThan(x => x.CheckIn)
    .WithMessage("Check-out must be after check-in.");

// Conditional — only validate when field is relevant
RuleFor(x => x.EarlyCheckInTime)
    .NotEmpty()
    .WithMessage("Early check-in time required when early check-in requested.")
    .When(x => x.RequestEarlyCheckIn);
```

---

## What Goes Where — Decision Table

| Check | Where | Why |
|---|---|---|
| Field empty / null | FluentValidation | Format — no DB needed |
| String max length | FluentValidation | Format — no DB needed |
| Number range | FluentValidation | Range — no DB needed |
| Date format valid | FluentValidation | Format — no DB needed |
| Check-in before check-out | FluentValidation | Cross-field — pure logic |
| Page size 1–100 | FluentValidation | Range — no DB needed |
| Room available on dates | Handler (ErrorOr) | Needs DB query |
| Guest email unique | Handler (ErrorOr) | Needs DB query |
| Booking in correct state | Domain method | Entity knows its own state |
| Money not negative | Domain VO | Invariant — enforced everywhere |
| Currency mixing | Domain VO | Invariant — enforced everywhere |
| End date after start date | Domain VO (DateRange) | Invariant — enforced everywhere |

**Rule:** FluentValidation = DB touch இல்லாம் decide பண்ண முடிஞ்சதை மட்டும்.
DB வேணும்னா → handler (ErrorOr) or domain method (DomainException).

---

## Query Validators — Optional

```csharp
// GetBookingsQueryValidator.cs
public sealed class GetBookingsQueryValidator
    : AbstractValidator<GetBookingsQuery>
{
    public GetBookingsQueryValidator()
    {
        RuleFor(x => x.PageNumber)
            .GreaterThan(0)
            .WithMessage("Page number must be at least 1.");

        RuleFor(x => x.PageSize)
            .InclusiveBetween(1, 100)
            .WithMessage("Page size must be between 1 and 100.");

        RuleFor(x => x.CheckIn)
            .LessThan(x => x.CheckOut)
            .WithMessage("Filter check-in must be before check-out.")
            .When(x => x.CheckIn.HasValue && x.CheckOut.HasValue);
    }
}
```

No validator registered → ValidationBehavior skips silently.
Validator = opt-in per command/query. Mandatory-ஆ இல்ல.

---

## Response When Validation Fails

```json
HTTP 422 Unprocessable Entity
{
  "success": false,
  "message": "One or more validation errors occurred.",
  "data": null,
  "errors": {
    "roomId":  ["Room is required."],
    "checkIn": ["Check-in date must be today or in the future."],
    "nights":  ["Must book at least 1 night."]
  }
}
```

Field names = C# property names (camelCase in JSON).

---

## Full Flow — Validation Fail

```
POST /api/v1/bookings
{ "roomId": "", "checkIn": "2025-01-01" }
         │
         ▼
Controller → CreateBookingCommand
         │
         ▼
LoggingBehavior → log
         │
         ▼
ValidationBehavior
  ├── roomId empty       → "Room is required."
  ├── checkIn past date  → "Check-in date must be today or in the future."
  └── failures.Count > 0 → throw ValidationException
         │
         ▼
ExceptionHandlingMiddleware
  └── catch ValidationException → 422 ApiResponse

── Handler never runs      ──
── DB never touched        ──
── Transaction never opens ──
```

## Full Flow — Validation Pass, Domain Fail

```
POST /api/v1/bookings/{id}/cancel
{ "reason": "Guest changed plans" }   ← valid input
         │
         ▼
ValidationBehavior → all rules pass ✅
         │
         ▼
TransactionBehavior → BEGIN TRANSACTION
         │
         ▼
Handler
  ├── FindByIdAsync → booking found ✅
  ├── booking.Cancel(reason, now)
  │     └── Status == CheckedIn
  │           → throw DomainException("Cannot cancel a checked-in booking.")
  │
  ▼
TransactionBehavior → ROLLBACK
  │
  ▼
ExceptionHandlingMiddleware
  └── catch DomainException → 400 ApiResponse

{
  "success": false,
  "message": "Cannot cancel a checked-in booking.",
  "data": null
}
```

---

## Rules

```
1. FluentValidation  → input format/structure only — no DB calls
2. DB checks         → handler (ErrorOr) or domain method (DomainException)
3. Validator file    → same folder as its command/query (vertical slice)
4. No validator      → ValidationBehavior skips (opt-in, not mandatory)
5. Domain VOs        → always enforce own invariants (Money, DateRange, etc.)
6. Validation fail   → 422 with field-level errors dict
7. Domain fail       → 400 with single message
8. Conflict fail     → 409 with single message (via ErrorOr in handler)
```
