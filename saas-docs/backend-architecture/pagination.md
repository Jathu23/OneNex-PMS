# OneNex — Pagination

> Status: Living Document
> Last updated: 2026-09-07
> Covers: PagedList, offset pagination, Dapper pattern, response shape

---

## Approach — Offset Pagination

```
Offset (SKIP/TAKE)  → V1 choice ✅  — simple, predictable, sufficient
Cursor (Keyset)     → V2 if needed  — better perf at very large offsets
```

Hospitality SaaS data volumes-க்கு offset sufficient.

---

## PagedList\<T\> — Shared.Kernel

Every module's list query இந்த type return பண்ணும்.

```csharp
// Shared.Kernel/Primitives/PagedList.cs
namespace OneNex.Shared.Kernel.Primitives;

public sealed class PagedList<T>
{
    public IReadOnlyList<T> Items           { get; private init; } = [];
    public int              TotalCount      { get; private init; }
    public int              PageNumber      { get; private init; }
    public int              PageSize        { get; private init; }
    public int              TotalPages      => (int)Math.Ceiling(TotalCount / (double)PageSize);
    public bool             HasNextPage     => PageNumber < TotalPages;
    public bool             HasPreviousPage => PageNumber > 1;

    private PagedList() { }

    public static PagedList<T> Create(
        IReadOnlyList<T> items,
        int totalCount,
        int pageNumber,
        int pageSize)
        => new()
        {
            Items      = items,
            TotalCount = totalCount,
            PageNumber = pageNumber,
            PageSize   = pageSize
        };

    public static PagedList<T> Empty(int pageNumber, int pageSize)
        => Create([], 0, pageNumber, pageSize);
}
```

---

## Dapper — Count + Data in One Round Trip

```csharp
// Stays.Infrastructure/Repositories/Read/BookingReadRepository.cs
public async Task<PagedList<BookingListItemDto>> ListAsync(
    ListBookingsQuery query, CancellationToken ct = default)
{
    var offset = (query.PageNumber - 1) * query.PageSize;

    // Two SQL statements — one round trip via QueryMultiple
    var sql = """
        SELECT COUNT(*)
        FROM stays.bookings
        WHERE business_id = @BusinessId
          AND is_deleted  = false
          AND (@Status IS NULL OR status = @Status::text);

        SELECT
            b.id              AS BookingId,
            g.full_name       AS GuestName,
            r.room_number     AS RoomNumber,
            b.check_in        AS CheckIn,
            b.check_out       AS CheckOut,
            (b.check_out - b.check_in) AS Nights,
            b.total_amount    AS TotalAmount,
            b.total_currency  AS Currency,
            b.status          AS Status,
            b.created_at      AS CreatedAt
        FROM stays.bookings b
        JOIN stays.guests g ON g.id = b.guest_id
        JOIN stays.rooms  r ON r.id = b.room_id
        WHERE b.business_id = @BusinessId
          AND b.is_deleted  = false
          AND (@Status IS NULL OR b.status = @Status::text)
        ORDER BY b.created_at DESC
        LIMIT @PageSize OFFSET @Offset;
        """;

    var parameters = new
    {
        BusinessId = query.BusinessId,
        Status     = query.Status,
        PageSize   = query.PageSize,
        Offset     = offset
    };

    using var conn  = await _connectionFactory.OpenAsync(ct);
    using var multi = await conn.QueryMultipleAsync(sql, parameters);

    var totalCount = await multi.ReadSingleAsync<int>();
    var items      = (await multi.ReadAsync<BookingListItemDto>()).ToList();

    return PagedList<BookingListItemDto>.Create(
        items,
        totalCount,
        query.PageNumber,
        query.PageSize);
}
```

Two queries, one DB round trip. Count + data parallel via `QueryMultiple`.

---

## Query — Pagination Parameters

```csharp
// Stays.Application/Features/Bookings/Queries/ListBookings/ListBookingsQuery.cs
public sealed record ListBookingsQuery(
    Guid    BusinessId,
    string? Status     = null,
    int     PageNumber = 1,
    int     PageSize   = 20) : IQuery<PagedList<BookingListItemDto>>;
```

---

## Validator — Page Constraints

```csharp
// ListBookingsQueryValidator.cs
public sealed class ListBookingsQueryValidator
    : AbstractValidator<ListBookingsQuery>
{
    public ListBookingsQueryValidator()
    {
        RuleFor(x => x.PageNumber)
            .GreaterThan(0)
            .WithMessage("Page number must be at least 1.");

        RuleFor(x => x.PageSize)
            .InclusiveBetween(1, 100)
            .WithMessage("Page size must be between 1 and 100.");
    }
}
```

Max 100 per page — `?pageSize=10000` abuse prevent பண்ணுவோம்.

---

## Handler

```csharp
// ListBookingsQueryHandler.cs
public sealed class ListBookingsQueryHandler(
    IBookingReadRepository readRepo,
    ICurrentUser currentUser)
    : IQueryHandler<ListBookingsQuery, PagedList<BookingListItemDto>>
{
    public async Task<ErrorOr<PagedList<BookingListItemDto>>> Handle(
        ListBookingsQuery query, CancellationToken ct)
    {
        var result = await readRepo.ListAsync(
            query with { BusinessId = currentUser.BusinessId }, ct);

        return result;
    }
}
```

---

## Controller

```csharp
// GET api/v1/stays/bookings?pageNumber=2&pageSize=20&status=Confirmed
[HttpGet]
public async Task<IActionResult> List(
    [FromQuery] ListBookingsRequest request, CancellationToken ct)
{
    var query = new ListBookingsQuery(
        currentUser.BusinessId,
        request.Status,
        request.PageNumber,
        request.PageSize);

    var result = await sender.Send(query, ct);
    return Match(result);
}
```

```csharp
// Stays.WebAPI/Models/Requests/ListBookingsRequest.cs
public sealed record ListBookingsRequest(
    string? Status     = null,
    int     PageNumber = 1,
    int     PageSize   = 20);
```

---

## Response DTO

```csharp
// BookingListItemDto.cs — lean shape for list view
public sealed record BookingListItemDto(
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

List view = minimal data. Full detail = separate `GetBookingQuery` → `BookingDetailDto`.

---

## Client Response Shape

```json
HTTP 200 OK
{
  "success": true,
  "message": null,
  "data": {
    "items": [
      {
        "bookingId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
        "guestName": "Arun Kumar",
        "roomNumber": "101",
        "checkIn": "2026-10-15",
        "checkOut": "2026-10-18",
        "nights": 3,
        "totalAmount": 450.00,
        "currency": "USD",
        "status": "Confirmed",
        "createdAt": "2026-09-07T10:30:00Z"
      },
      { ... }
    ],
    "totalCount": 150,
    "pageNumber": 1,
    "pageSize": 20,
    "totalPages": 8,
    "hasNextPage": true,
    "hasPreviousPage": false
  }
}
```

Client-க்கு navigate பண்ண வேண்டியது எல்லாம் response-ல் இருக்கும்.

---

## Empty Page Response

```json
HTTP 200 OK
{
  "success": true,
  "message": null,
  "data": {
    "items": [],
    "totalCount": 0,
    "pageNumber": 1,
    "pageSize": 20,
    "totalPages": 0,
    "hasNextPage": false,
    "hasPreviousPage": false
  }
}
```

Empty list = 200, not 404. Data இல்லன்னு resource missing இல்ல.

---

## Read Repository Interface

```csharp
public interface IBookingReadRepository
{
    Task<PagedList<BookingListItemDto>> ListAsync(
        ListBookingsQuery query, CancellationToken ct = default);

    Task<BookingDetailDto?> FindByIdAsync(
        BookingId id, CancellationToken ct = default);

    Task<bool> ExistsAsync(
        BookingId id, CancellationToken ct = default);
}
```

---

## Rules

```
1. PagedList<T>    → Shared.Kernel — every module uses same type
2. Offset          → V1 (LIMIT + OFFSET), Cursor → V2 if needed
3. Default page    → 20 items
4. Max page size   → 100 (validator enforces)
5. Count + data    → one round trip (QueryMultiple)
6. Empty list      → 200 OK with empty items array (not 404)
7. List DTO        → lean, only what list view needs
8. Detail DTO      → separate query, full data
9. ORDER BY        → always explicit (created_at DESC default)
10. business_id    → filter explicitly in query (not global filter)
```
