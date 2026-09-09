# OneNex — Notification Module Design

> Status: Living Document — In Progress. Not final.
> Last updated: 2026-09-06
> Based on: NotifyNet (custom library built by team) — reimagined for OneNex

---

## Overview

Notification Module = self-contained engine that handles ALL outbound communications.
Other modules never touch email/SMS logic. They only publish one event.

```
Any Module → SendNotificationsEvent → Notification Module → Email / SMS / InApp
```

---

## Core Concept — Unified Event Pattern

Only ONE integration event. ONE handler. All modules use the same pattern.

```csharp
// Any module — this is ALL they write
await _eventBus.PublishAsync(new SendNotificationsEvent([
    new NotificationModel
    {
        ReceiverId       = guestId.ToString(),
        TenantId         = businessId.ToString(),
        NotificationType = NotificationTypes.Stays.BookingConfirmed,
        Payload          = new { guest_name, booking_ref, checkin_date },
        Priority         = NotificationPriority.High,
        IdempotencyKey   = $"booking-confirmed-{bookingId}"
    }
]), ct);

// Notification module handles everything else:
// → Template lookup → Liquid render → Email + SMS → Retry → Log
```

---

## NotifyNet — Foundation Reference

NotifyNet is a custom notification library built by the team (at J:\Asp.Net\NotifyNet_).
OneNex Notification Module = reimagined NotifyNet, built fresh for OneNex.

### What NotifyNet Has (Proven, Reuse Concepts)

| Feature | NotifyNet | OneNex Notification Module |
|---|---|---|
| In-memory Channel dispatch | ✅ | ✅ Keep |
| ChannelWorker (BackgroundService) | ✅ | ✅ Keep |
| ModeRouter (Immediate/Delay/Scheduled) | ✅ | ✅ Improved |
| NotificationExecutor (execute per channel) | ✅ | ✅ Keep |
| Per-channel retry (3 attempts, exponential) | ✅ | ✅ Keep |
| Hangfire fallback for persistent retry | ✅ | ✅ Keep |
| Idempotency key check (before send) | ✅ | ✅ Keep |
| Subscription / opt-out check | ✅ | ✅ Keep |
| Fluid/Liquid template system | ✅ | ✅ Keep |
| Tenant-specific templates + configs | ✅ | ✅ Keep |
| HMAC-signed unsubscribe tokens | ✅ | ✅ Keep |
| SMTP sender (MailKit) | ✅ | ✅ Keep |
| SendGrid sender | ✅ | ✅ Keep |
| TextLk SMS sender | ✅ | ✅ Keep |
| In-App notification storage | ✅ | ✅ Keep |
| SQL Server storage | ✅ | ❌ Replace with PostgreSQL |
| No unified integration event | ❌ | ✅ Fixed — SendNotificationsEvent |
| No module boundary (direct call) | ❌ | ✅ Fixed — IEventBus pattern |

### Key Improvements Over NotifyNet

**1. PostgreSQL (not SQL Server)**
```csharp
// EFCore: opt.UseNpgsql(connectionString)
// Hangfire: .UsePostgreSqlStorage(connectionString)
```

**2. High Priority = Hangfire Direct (crash-safe)**
```csharp
// ModeRouter improvement:
if (context.Priority == NotificationPriority.High)
    await _scheduler.EnqueueAsync(context, queue: "critical"); // persisted
else
    await _executor.ExecuteAsync(context, channel: 0, ct);    // fast
```

**3. Typed NotificationTypes constants**
```csharp
// Before (NotifyNet): NotificationType = 1001  ← magic number
// After (OneNex):     NotificationType = NotificationTypes.Stays.BookingConfirmed
```

**4. Unified integration event (cross-module pattern)**
```csharp
// Before: Each module had its own integration event + handler
// After:  One SendNotificationsEvent, one handler
```

---

## Architecture — Full Flow

```
INotificationService.SendAsync()
        │
        ▼
  NotificationValidator          ← WHO / WHAT / WHEN validate
        │
        ▼
  Channel.WriteAsync()           ← in-memory push (0.1ms) — caller returns immediately
        │
  [Background — ChannelWorker reads]
        │
        ▼
    ModeRouter
   ┌────┴──────────────────────────┐
   ▼                               ▼
Immediate (Normal/Low)      High Priority / Delay / Scheduled
   │                               │
   ▼                               ▼
NotificationExecutor          HangfireScheduler
   │                          (persisted to PostgreSQL)
   │                               │
   └──────────────┬────────────────┘
                  ▼
         [Per channel — independent]
         ┌────────────────────────────────────┐
         │ 1. Idempotency check               │
         │ 2. Subscription / opt-out check    │
         │ 3. Template lookup (DB)            │
         │ 4. Fluid/Liquid render             │
         │ 5. Tenant config resolve           │
         │ 6. Receiver resolve (3-tier)       │
         │ 7. Send (Email / SMS / InApp)      │
         │ 8. Log result                      │
         └────────────────────────────────────┘
                  │
            Fail? → In-process retry (3 attempts, 2s/4s backoff)
                  │
            Still fail? → Hangfire persistent retry (5 min delay)
```

### Receiver Resolution — 3 Tier

```
Tier 1: DirectContact provided? → use it directly (no DB lookup)
Tier 2: ReceiverId provided → query notification_users table
Tier 3: IReceiverResolver → OneNexReceiverResolver → query OneNex identity DB
```

### Response Time

```
Without background queue: API waits for email send → 200-500ms
With background queue:    API returns after Channel.WriteAsync() → ~10ms
Email sending happens in background — user doesn't wait
```

---

## Module Boundary Rules

```
Other modules know:
  ✅ SendNotificationsEvent     (from Notification.Contracts)
  ✅ NotificationModel          (from Notification.Contracts)
  ✅ NotificationTypes          (from Notification.Contracts)
  ❌ INotificationService internals
  ❌ Hangfire
  ❌ Email/SMS providers
  ❌ Template system

Notification module knows:
  ✅ Everything above
  ✅ NotifyNet-based engine
  ✅ Hangfire scheduling
  ✅ Email/SMS/InApp sending
  ✅ Template rendering
```

---

## Project Structure

```
src/Modules/Notification/
│
├── Notification.Contracts/          ← No dependencies. Referenced by all modules.
│   ├── NotificationModel.cs
│   ├── DirectContact.cs
│   ├── SendNotificationsEvent.cs
│   ├── NotificationTypes.cs
│   └── Enums/
│       ├── NotificationMode.cs
│       ├── NotificationPriority.cs
│       └── NotificationChannel.cs
│
├── Notification.Infrastructure/     ← The engine (NotifyNet reimagined)
│   ├── Abstractions/
│   │   ├── INotificationService.cs
│   │   ├── IEmailSender.cs
│   │   ├── ISmsSender.cs
│   │   ├── IInAppSender.cs
│   │   ├── INotificationExecutor.cs
│   │   ├── INotificationScheduler.cs
│   │   └── IReceiverResolver.cs
│   ├── Pipeline/
│   │   ├── NotificationJobContext.cs
│   │   ├── NotificationService.cs
│   │   ├── NotificationValidator.cs
│   │   ├── ChannelWorker.cs
│   │   ├── ModeRouter.cs
│   │   └── NotificationExecutor.cs  ← TODO: write this (template + send + log + retry)
│   ├── Persistence/
│   │   ├── NotificationDbContext.cs
│   │   └── Entities/
│   │       ├── NotificationTemplate.cs
│   │       ├── NotificationLog.cs
│   │       ├── NotificationChannelStrategy.cs
│   │       ├── NotificationTenantConfig.cs
│   │       ├── NotificationSubscription.cs
│   │       ├── NotificationUser.cs
│   │       ├── NotificationUserContact.cs
│   │       └── InAppNotification.cs
│   ├── Senders/
│   │   ├── Email/
│   │   │   ├── SmtpEmailSender.cs   ← TODO
│   │   │   └── SendGridEmailSender.cs ← TODO
│   │   ├── Sms/
│   │   │   └── TextLkSmsSender.cs   ← TODO
│   │   └── InApp/
│   │       └── EfInAppSender.cs     ← TODO
│   ├── Scheduling/
│   │   └── HangfireScheduler.cs     ← TODO
│   └── DependencyInjection.cs
│
├── Notification.Application/
│   ├── Handlers/
│   │   └── SendNotificationsEventHandler.cs   ← THE ONE HANDLER
│   ├── Resolvers/
│   │   └── OneNexReceiverResolver.cs
│   └── DependencyInjection.cs
│
└── Notification.Presentation/
    └── Controllers/
        ├── UnsubscribeController.cs            ← TODO
        └── NotificationManagementController.cs ← TODO (templates, logs, strategies)
```

---

## Code Written So Far

### Notification.Contracts (Complete)

**`Enums/NotificationMode.cs`**
```csharp
namespace OneNex.Notification.Contracts.Enums;

public enum NotificationMode
{
    Immediate = 1,
    Delay     = 2,
    Scheduled = 3
}
```

**`Enums/NotificationPriority.cs`**
```csharp
namespace OneNex.Notification.Contracts.Enums;

public enum NotificationPriority
{
    Low    = 1,
    Normal = 2,
    High   = 3
}
```

**`Enums/NotificationChannel.cs`**
```csharp
namespace OneNex.Notification.Contracts.Enums;

public enum NotificationChannel
{
    Email = 1,
    Sms   = 2,
    InApp = 3,
    Push  = 4   // Phase 2
}
```

**`DirectContact.cs`**
```csharp
namespace OneNex.Notification.Contracts;

public sealed class DirectContact
{
    public List<string> Emails    { get; init; } = [];
    public List<string> Phones    { get; init; } = [];
    public List<string> FcmTokens { get; init; } = [];  // Phase 2 — push
}
```

**`NotificationModel.cs`**
```csharp
namespace OneNex.Notification.Contracts;

public sealed class NotificationModel
{
    // ── WHO ──────────────────────────────────────────────────────
    public string?        ReceiverId    { get; init; }
    public DirectContact? DirectContact { get; init; }

    // ── WHAT ─────────────────────────────────────────────────────
    public required int   NotificationType { get; init; }
    public object?        Payload          { get; init; }

    // ── CONTEXT ──────────────────────────────────────────────────
    public string?        TenantId      { get; init; }

    // ── WHEN ─────────────────────────────────────────────────────
    public NotificationMode    Mode        { get; init; } = NotificationMode.Immediate;
    public TimeSpan?           Delay       { get; init; }
    public DateTimeOffset?     ScheduledAt { get; init; }

    // ── OPTIONS ──────────────────────────────────────────────────
    public NotificationPriority Priority       { get; init; } = NotificationPriority.Normal;
    public string?              IdempotencyKey { get; init; }
}
```

**`SendNotificationsEvent.cs`**
```csharp
namespace OneNex.Notification.Contracts;

public sealed record SendNotificationsEvent(
    IReadOnlyList<NotificationModel> Notifications
) : IIntegrationEvent;
```

**`NotificationTypes.cs`**
```csharp
namespace OneNex.Notification.Contracts;

public static class NotificationTypes
{
    public static class Stays
    {
        public const int BookingConfirmed    = 1001;
        public const int BookingCancelled    = 1002;
        public const int PreArrivalReminder  = 1003;
        public const int CheckInReady        = 1004;
        public const int CheckoutReceipt     = 1005;
        public const int NoShowAlert         = 1006;
        public const int WaitlistAvailable   = 1007;
    }

    public static class Dining
    {
        public const int ReservationConfirmed = 2001;
        public const int ReservationReminder  = 2002;
        public const int OrderReady           = 2003;
        public const int BillReady            = 2004;
    }

    public static class Membership
    {
        public const int StaffInvitation      = 3001;
        public const int InvitationExpiring   = 3002;
        public const int PasswordReset        = 3003;
        public const int AccountSuspended     = 3004;
        public const int WelcomeOnboard       = 3005;
    }

    public static class Business
    {
        public const int NightAuditComplete   = 4001;
        public const int SubscriptionRenewing = 4002;
        public const int PaymentFailed        = 4003;
        public const int LowInventoryAlert    = 4004;
    }
}
```

---

### Notification.Infrastructure — Pipeline (Complete)

**`Pipeline/NotificationJobContext.cs`**
```csharp
namespace OneNex.Notification.Infrastructure.Pipeline;

public sealed class NotificationJobContext
{
    public int                  NotificationType { get; set; }
    public string?              TenantId         { get; set; }
    public string?              ReceiverId       { get; set; }
    public DirectContact?       DirectContact    { get; set; }
    public string?              PayloadJson      { get; set; }
    public NotificationMode     Mode             { get; set; }
    public TimeSpan?            Delay            { get; set; }
    public DateTimeOffset?      ScheduledAt      { get; set; }
    public NotificationPriority Priority         { get; set; }
    public string?              IdempotencyKey   { get; set; }
    public int                  Channel          { get; set; } = 0; // 0 = all channels
}
```

**`Pipeline/NotificationValidator.cs`**
```csharp
namespace OneNex.Notification.Infrastructure.Pipeline;

internal static class NotificationValidator
{
    public static void Validate(IReadOnlyList<NotificationModel> notifications)
    {
        var errors = new List<string>();

        foreach (var n in notifications)
        {
            var hasReceiver = !string.IsNullOrWhiteSpace(n.ReceiverId);
            var hasContact  = n.DirectContact is not null &&
                              (n.DirectContact.Emails.Count > 0 ||
                               n.DirectContact.Phones.Count > 0);

            if (!hasReceiver && !hasContact)
                errors.Add("Each notification requires ReceiverId or DirectContact.");

            if (n.NotificationType == 0)
                errors.Add("NotificationType cannot be 0.");

            if (n.Mode == NotificationMode.Delay && n.Delay is null)
                errors.Add("Delay is required when Mode = Delay.");

            if (n.Mode == NotificationMode.Scheduled && n.ScheduledAt is null)
                errors.Add("ScheduledAt is required when Mode = Scheduled.");

            if (n.Mode == NotificationMode.Scheduled &&
                n.ScheduledAt <= DateTimeOffset.UtcNow)
                errors.Add("ScheduledAt must be a future time.");
        }

        if (errors.Count > 0)
            throw new NotificationValidationException(errors);
    }
}
```

**`Pipeline/NotificationService.cs`**
```csharp
namespace OneNex.Notification.Infrastructure.Pipeline;

internal sealed class NotificationService : INotificationService
{
    private readonly Channel<NotificationJobContext> _channel;

    public NotificationService(Channel<NotificationJobContext> channel)
        => _channel = channel;

    public async Task SendAsync(
        IReadOnlyList<NotificationModel> notifications,
        CancellationToken ct = default)
    {
        NotificationValidator.Validate(notifications);

        foreach (var n in notifications)
        {
            var context = new NotificationJobContext
            {
                NotificationType = n.NotificationType,
                TenantId         = n.TenantId,
                ReceiverId       = n.ReceiverId,
                DirectContact    = n.DirectContact,
                PayloadJson      = n.Payload is not null
                                   ? JsonSerializer.Serialize(n.Payload)
                                   : null,
                Mode             = n.Mode,
                Delay            = n.Delay,
                ScheduledAt      = n.ScheduledAt,
                Priority         = n.Priority,
                IdempotencyKey   = n.IdempotencyKey
            };

            await _channel.Writer.WriteAsync(context, ct);
        }
    }
}
```

**`Pipeline/ChannelWorker.cs`**
```csharp
namespace OneNex.Notification.Infrastructure.Pipeline;

internal sealed class ChannelWorker : BackgroundService
{
    private readonly Channel<NotificationJobContext> _channel;
    private readonly IServiceScopeFactory            _scopeFactory;
    private readonly ILogger<ChannelWorker>          _logger;

    public ChannelWorker(
        Channel<NotificationJobContext> channel,
        IServiceScopeFactory scopeFactory,
        ILogger<ChannelWorker> logger)
    {
        _channel      = channel;
        _scopeFactory = scopeFactory;
        _logger       = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        await foreach (var context in _channel.Reader.ReadAllAsync(stoppingToken))
        {
            _ = Task.Run(async () =>
            {
                using var scope  = _scopeFactory.CreateScope();
                var router = scope.ServiceProvider.GetRequiredService<ModeRouter>();
                try
                {
                    await router.RouteAsync(context, stoppingToken);
                }
                catch (Exception ex)
                {
                    _logger.LogError(ex,
                        "Notification routing failed. Type={Type} Tenant={Tenant}",
                        context.NotificationType, context.TenantId);
                }
            }, stoppingToken);
        }
    }
}
```

**`Pipeline/ModeRouter.cs`**
```csharp
namespace OneNex.Notification.Infrastructure.Pipeline;

internal sealed class ModeRouter
{
    private readonly INotificationExecutor  _executor;
    private readonly INotificationScheduler _scheduler;

    public ModeRouter(INotificationExecutor executor, INotificationScheduler scheduler)
    {
        _executor  = executor;
        _scheduler = scheduler;
    }

    public async Task RouteAsync(NotificationJobContext context, CancellationToken ct)
    {
        switch (context.Mode)
        {
            case NotificationMode.Immediate:
                // High priority = Hangfire (crash-safe, persisted)
                if (context.Priority == NotificationPriority.High)
                {
                    await _scheduler.EnqueueAsync(context, queue: "critical");
                    return;
                }
                // Normal/Low = direct (fast path)
                await _executor.ExecuteAsync(context, channel: 0, ct);
                break;

            case NotificationMode.Delay:
                await _scheduler.ScheduleDelayAsync(context, context.Delay!.Value);
                break;

            case NotificationMode.Scheduled:
                await _scheduler.ScheduleAtAsync(context, context.ScheduledAt!.Value);
                break;
        }
    }
}
```

---

### Notification.Application (Complete)

**`Handlers/SendNotificationsEventHandler.cs`**
```csharp
namespace OneNex.Notification.Application.Handlers;

internal sealed class SendNotificationsEventHandler
    : INotificationHandler<SendNotificationsEvent>
{
    private readonly INotificationService _notifications;

    public SendNotificationsEventHandler(INotificationService notifications)
        => _notifications = notifications;

    public Task Handle(SendNotificationsEvent evt, CancellationToken ct)
        => _notifications.SendAsync(evt.Notifications, ct);
}
```

**`Resolvers/OneNexReceiverResolver.cs`**
```csharp
namespace OneNex.Notification.Application.Resolvers;

internal sealed class OneNexReceiverResolver : IReceiverResolver
{
    private readonly IIdentityService _identity; // from Shared.Contracts

    public OneNexReceiverResolver(IIdentityService identity)
        => _identity = identity;

    public async Task<NotificationReceiver?> ResolveAsync(
        string receiverId, string? tenantId, CancellationToken ct)
    {
        var contacts = await _identity.GetUserContactsAsync(Guid.Parse(receiverId), ct);
        if (contacts is null) return null;

        return new NotificationReceiver
        {
            Emails = contacts.Emails,
            Phones = contacts.Phones
            // FcmTokens = Phase 2 — push notifications
        };
    }
}
```

**`DependencyInjection.cs`**
```csharp
namespace OneNex.Notification.Application;

public static class DependencyInjection
{
    public static IServiceCollection AddNotificationModule(
        this IServiceCollection services,
        IConfiguration config)
    {
        // In-memory channel
        services.AddSingleton(_ =>
            Channel.CreateBounded<NotificationJobContext>(
                new BoundedChannelOptions(capacity: 2000)
                {
                    FullMode     = BoundedChannelFullMode.Wait,
                    SingleWriter = false,
                    SingleReader = false
                }));

        // Core pipeline
        services.AddSingleton<INotificationService, NotificationService>();
        services.AddScoped<ModeRouter>();
        services.AddHostedService<ChannelWorker>();

        // Executor + Scheduler (TODO: implement)
        services.AddScoped<INotificationExecutor, NotificationExecutor>();
        services.AddScoped<INotificationScheduler, HangfireScheduler>();

        // Receiver resolver
        services.AddScoped<IReceiverResolver, OneNexReceiverResolver>();

        // Email provider — config-driven
        var emailProvider = config["Notification:EmailProvider"] ?? "Smtp";
        if (emailProvider == "SendGrid")
            services.AddSendGridEmailSender(config);
        else
            services.AddSmtpEmailSender(config);

        // SMS
        services.AddTextLkSmsSender(config);

        // Hangfire (PostgreSQL storage)
        services.AddHangfire(cfg => cfg
            .UsePostgreSqlStorage(config.GetConnectionString("DefaultConnection")));
        services.AddHangfireServer(opt =>
            opt.Queues = ["critical", "default", "low"]);

        // MediatR — discovers SendNotificationsEventHandler
        services.AddMediatR(cfg =>
            cfg.RegisterServicesFromAssembly(typeof(SendNotificationsEventHandler).Assembly));

        return services;
    }
}
```

---

## Database Tables

```
notification_templates          ← Liquid templates per type+channel+tenant
notification_channel_strategies ← Which channels active per notification type
notification_tenant_configs     ← Per-tenant SMTP/SendGrid/TextLk credentials (JSON)
notification_logs               ← Full delivery audit trail (attempt, status, error)
notification_subscriptions      ← Receiver opt-out preferences
notification_users              ← User identities (ReceiverId lookup)
notification_user_contacts      ← Email/phone per user
inapp_notifications             ← In-app notification storage (read/unread)
```

---

## DB Template Example

```
Type: 1001 (Stays.BookingConfirmed)
Channel: Email
Subject: "{{ guest_name }}, your booking {{ booking_ref }} is confirmed!"
Body:    "<h1>Hello {{ guest_name }}</h1>
          <p>Check-in: {{ checkin_date }}</p>
          <p>Room: {{ room_type }}</p>
          <a href='{{ unsubscribe_url }}'>Unsubscribe</a>"

Type: 1001 (Stays.BookingConfirmed)
Channel: SMS
Body:    "Hi {{ guest_name }}, booking {{ booking_ref }} confirmed. Check-in: {{ checkin_date }}"
```

---

## Independent Deployment Strategy

### V1 — In-Process (Monolith)

```
OneNex.WebAPI (single process)
  ├── All modules
  └── Notification Module (runs in-process)
       └── MediatR handles SendNotificationsEvent
```

### V2 — Standalone Service (when volume demands)

```
OneNex.WebAPI
  └── All modules → MassTransit → RabbitMQ

OneNex.NotificationService (separate deployment)
  └── MassTransit Consumer → same handler logic → NotifyNet engine
```

**Handler code = zero change for V2 switch.**
Only add MassTransit consumer wrapper (5 lines) + change DI transport.

---

## How Other Modules Use It — Reference

### Stays Module — Booking Confirmed

```csharp
await _eventBus.PublishAsync(new SendNotificationsEvent([

    // Immediate confirmation
    new NotificationModel
    {
        ReceiverId       = booking.GuestId.ToString(),
        TenantId         = booking.BusinessId.ToString(),
        NotificationType = NotificationTypes.Stays.BookingConfirmed,
        Payload          = new
        {
            guest_name    = guest.FullName,
            booking_ref   = booking.Reference,
            checkin_date  = booking.CheckIn.ToString("MMM dd, yyyy"),
            checkout_date = booking.CheckOut.ToString("MMM dd, yyyy"),
            room_type     = room.TypeName
        },
        Priority         = NotificationPriority.High,
        IdempotencyKey   = $"booking-confirmed-{booking.Id}"
    },

    // Pre-arrival reminder — scheduled automatically
    new NotificationModel
    {
        ReceiverId       = booking.GuestId.ToString(),
        TenantId         = booking.BusinessId.ToString(),
        NotificationType = NotificationTypes.Stays.PreArrivalReminder,
        Payload          = new { guest_name = guest.FullName },
        Mode             = NotificationMode.Scheduled,
        ScheduledAt      = booking.CheckIn.AddDays(-1),
        IdempotencyKey   = $"pre-arrival-{booking.Id}"
    }

]), ct);
```

### Membership Module — Staff Invitation

```csharp
await _eventBus.PublishAsync(new SendNotificationsEvent([
    new NotificationModel
    {
        DirectContact    = new DirectContact { Emails = [invitation.Email] },
        TenantId         = invitation.BusinessId.ToString(),
        NotificationType = NotificationTypes.Membership.StaffInvitation,
        Payload          = new
        {
            business_name = business.Name,
            role          = invitation.Role,
            invite_link   = invitation.InviteUrl,
            expires_in    = "72 hours"
        },
        Priority         = NotificationPriority.High,
        IdempotencyKey   = $"staff-invite-{invitation.Id}"
    }
]), ct);
```

---

## Database Architecture — Decided

### Two Databases — Isolation Strategy

```
Main DB (onenex):
  stays.* dining.* identity.* membership.* business.*
  Load: Business operations only

Notification DB (onenex_notification):
  notification.templates
  notification.logs
  notification.channel_strategies
  notification.tenant_configs
  notification.subscriptions
  notification.users
  notification.user_contacts
  inapp.notifications
  hangfire.job              ← Hangfire tables HERE (not main DB)
  hangfire.jobqueue
  hangfire.state
  hangfire.server
```

Hangfire load = stays inside notification DB. Main business operations = unaffected.

### Why Not SQLite for Hangfire

```
SQLite rejected for production:
  ❌ No multi-server support (file lock conflict)
  ❌ Write locking under high notification volume
  ❌ No replication / backup built-in
  ✅ Dev only — InMemory Hangfire used instead
```

### Hangfire — LISTEN/NOTIFY (Low Load)

```csharp
services.AddHangfire(cfg => cfg
    .UsePostgreSqlStorage(
        config.GetConnectionString("NotificationConnection"),
        new PostgreSqlStorageOptions
        {
            QueuePollInterval = TimeSpan.Zero  // LISTEN/NOTIFY — not polling
            // Near-zero DB load when no jobs pending
        }));
```

### Connection Strings

```json
{
  "ConnectionStrings": {
    "DefaultConnection":      "Host=...;Database=onenex;...",
    "NotificationConnection": "Host=...;Database=onenex_notification;..."
  }
}
```

### Dev vs Production

```
Dev:        Hangfire InMemory (zero infra setup)
Production: Hangfire PostgreSQL → NotificationConnection DB
```

```csharp
if (env.IsDevelopment())
    services.AddHangfire(cfg => cfg.UseInMemoryStorage());
else
    services.AddHangfire(cfg =>
        cfg.UsePostgreSqlStorage(config.GetConnectionString("NotificationConnection")));
```

---

## TODO — Still To Build

### Infrastructure (Next Session)

- [ ] `NotificationExecutor.cs` — core execution:
  - Idempotency check (query notification_logs)
  - Subscription/opt-out check
  - Channel strategy lookup (which channels for this type)
  - Template lookup + Fluid/Liquid render
  - Tenant config resolve (per-tenant SMTP/SMS creds)
  - Receiver resolve (3-tier: DirectContact → DB → IReceiverResolver)
  - Send via IEmailSender / ISmsSender / IInAppSender
  - Log result (INSERT on first attempt, UPDATE on retry)
  - Per-channel retry: 3 attempts, exponential backoff (2s, 4s)
  - If all fail: Hangfire persistent retry (5 min delay)

- [ ] `HangfireScheduler.cs`
  - EnqueueAsync (fire-and-forget with queue name)
  - ScheduleDelayAsync (TimeSpan-based)
  - ScheduleAtAsync (DateTimeOffset-based)

- [ ] `Persistence/NotificationDbContext.cs` + all entity configs

- [ ] `Senders/Email/SmtpEmailSender.cs` (MailKit)
- [ ] `Senders/Email/SendGridEmailSender.cs`
- [ ] `Senders/Sms/TextLkSmsSender.cs`
- [ ] `Senders/InApp/EfInAppSender.cs`

### Presentation (Later)

- [ ] `UnsubscribeController.cs` — handle HMAC token, call OptOut
- [ ] `NotificationManagementController.cs` — template CRUD, log query, strategy management

### Phase 2

- [ ] Push notifications (FCM)
- [ ] Email open/click tracking (webhook)
- [ ] Notification preferences UI (per guest/staff)
- [ ] Multi-language templates

---

## Key Design Decisions

| Decision | Choice | Reason |
|---|---|---|
| Unified event | `SendNotificationsEvent` | One handler, zero per-type boilerplate |
| Channel → Hangfire split | Normal/Low = Channel, High = Hangfire | Speed + reliability balance |
| PostgreSQL | EF Core Npgsql + Hangfire.PostgreSql | OneNex infra consistency |
| Liquid templates | Fluid library | Sandboxed, safe, designer-friendly |
| Idempotency scope | Per channel + key | Email success, SMS retry → no double email |
| Retry strategy | 3x in-process → Hangfire fallback | Fast recovery + persistent safety net |
| Module boundary | Notification.Contracts (thin) | Other modules zero knowledge of engine |
| Receiver resolver | IIdentityService via Shared.Contracts | No direct DB cross-module access |
