# Business Module — Design Document

> Draft — Core module. Reviewed and finalized during architecture planning.
> This module manages business registration, operations enablement, hours, images, settings, and tax configuration.

---

## Module Responsibility

The Business module owns everything about a business entity:
- Business core identity and locale
- Business profile (contact details)
- Business address and geo coordinates
- Business images (logo, cover, gallery)
- Operations enablement (Dining, Stays, Bar, Wellness, Events, Retail)
- Add-on enablement per operation
- Subscription tracking per operation
- Operating hours (regular + exceptions)
- Business-wide settings
- Tax rate definitions and operation-level tax mapping

**What Business module does NOT own:**
- Staff identities (Identity module)
- Staff roles and permissions (Membership module)
- Actual dining/stays/bar operations (Operation modules)
- Payment processing (Payment Service)
- Notifications (Notification Service)

---

## Domain Events Published

```
BusinessCreatedEvent            → Membership module: auto-create owner membership record
BusinessOperationEnabledEvent   → Operation module: initialize module data (menus, tables...)
BusinessOperationDisabledEvent  → Operation module: cleanup / suspend
BusinessSuspendedEvent          → All modules: block access for this business_id
```

Business module publishes. It never calls other modules directly.

---

## Entities

### Table 1: `businesses`

Lean core identity. Loaded on every API call for tenant resolution — kept minimal on purpose.

```sql
CREATE TABLE businesses (
    id                  UUID            PRIMARY KEY DEFAULT gen_random_uuid(),
    owner_id            UUID            NOT NULL REFERENCES users(id),
    parent_business_id  UUID            REFERENCES businesses(id),  -- nullable: branch (Phase 2)

    -- Names
    legal_name          VARCHAR(200)    NOT NULL,  -- official registered name (invoices, contracts)
    trading_name        VARCHAR(200)    NOT NULL,  -- what customers see ("The Grand Café")

    -- URL identity
    slug                VARCHAR(100)    NOT NULL,  -- {slug}.onenex.com — globally unique

    -- Locale (needed on every request: currency formatting, timezone display)
    country_code        CHAR(2)         NOT NULL,  -- ISO 3166-1 alpha-2: LK, SG, GB, US
    timezone            VARCHAR(50)     NOT NULL,  -- IANA: Asia/Colombo, Asia/Singapore
    currency_code       CHAR(3)         NOT NULL,  -- ISO 4217: LKR, SGD, USD
    default_language    CHAR(2)         NOT NULL DEFAULT 'en',  -- ISO 639-1

    -- Lifecycle
    status              VARCHAR(20)     NOT NULL DEFAULT 'active',
    -- active | suspended | closed
    activated_at        TIMESTAMP WITH TIME ZONE,  -- when business first went live (≠ created_at)
    created_at          TIMESTAMP WITH TIME ZONE   NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMP WITH TIME ZONE   NOT NULL DEFAULT NOW(),

    CONSTRAINT businesses_slug_unique UNIQUE (slug)
);

CREATE INDEX idx_businesses_owner_id   ON businesses(owner_id);
CREATE INDEX idx_businesses_status     ON businesses(status);
CREATE INDEX idx_businesses_slug       ON businesses(slug);
```

**Design notes:**

| Field | Why |
|---|---|
| `legal_name` | Official registered name — tax invoices, legal documents |
| `trading_name` | Display name customers see — can differ from legal name |
| `slug` | URL identity — must be set from day 1 even though V1 uses `app.onenex.com` |
| `country_code` | Drives tax logic and locale defaults on every request |
| `timezone` | All timestamps stored UTC, displayed in business timezone |
| `currency_code` | Drives all billing and reporting |
| `activated_at` | Business can be created but not yet live — marks "first day open" |
| `parent_business_id` | Future-ready for branches (Phase 2). Nullable in V1. No branch logic in V1. |

**Status values:**
- `active` — operating normally
- `suspended` — payment issue or admin action — staff cannot access
- `closed` — permanently closed — data retained for compliance

---

### Table 2: `business_profiles`

Contact details. Loaded on demand — profile page, directory listings.

```sql
CREATE TABLE business_profiles (
    id              UUID            PRIMARY KEY DEFAULT gen_random_uuid(),
    business_id     UUID            NOT NULL UNIQUE REFERENCES businesses(id),  -- 1:1

    phone           VARCHAR(20),
    email           VARCHAR(255),
    website_url     VARCHAR(500),
    description     TEXT,

    updated_at      TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);
```

**Why split from `businesses`?**
```
businesses       → loaded on EVERY API call (tenant resolution, JWT context) — must stay lean
business_profiles → loaded only when needed: profile read/update, directory listing
```

---

### Table 3: `business_addresses`

Physical location. Loaded on demand — maps, delivery, receipts.

```sql
CREATE TABLE business_addresses (
    id              UUID            PRIMARY KEY DEFAULT gen_random_uuid(),
    business_id     UUID            NOT NULL UNIQUE REFERENCES businesses(id),  -- 1:1

    address_line1   VARCHAR(255),
    address_line2   VARCHAR(255),
    city            VARCHAR(100),
    state           VARCHAR(100),  -- state / province / region
    postal_code     VARCHAR(20),
    -- country_code lives in businesses table (needed for locale on every request)

    -- Geo coordinates (maps, delivery radius)
    latitude        DECIMAL(10, 7),
    longitude       DECIMAL(10, 7),

    updated_at      TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);
```

> `country_code` stays in `businesses` — needed for tax/locale on every request.
> Address table = display and geo only.

---

### Table 4: `business_images`

Logo, cover, gallery images. 1:many. Typed.

```sql
CREATE TABLE business_images (
    id              UUID            PRIMARY KEY DEFAULT gen_random_uuid(),
    business_id     UUID            NOT NULL REFERENCES businesses(id),

    image_type      VARCHAR(20)     NOT NULL,
    -- logo | cover | gallery

    url             VARCHAR(500)    NOT NULL,
    alt_text        VARCHAR(200),
    display_order   SMALLINT        NOT NULL DEFAULT 0,
    is_active       BOOLEAN         NOT NULL DEFAULT true,

    created_at      TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_business_images_business_type
    ON business_images(business_id, image_type);
```

**Image types:**

| Type | Description |
|---|---|
| `logo` | Main brand logo — one active at a time |
| `cover` | Hero / banner image |
| `gallery` | Multiple images — `display_order` controls sequence |

---

### Table 5: `business_operations`

Which operations are enabled per business. Subscription state only — Business module's actual concern.

Operation-specific config (order prefix, kitchen buffer, check-in time etc.) lives inside each operation's own module — not here.

```sql
CREATE TABLE business_operations (
    id                  UUID            PRIMARY KEY DEFAULT gen_random_uuid(),
    business_id         UUID            NOT NULL REFERENCES businesses(id),
    operation_type      VARCHAR(20)     NOT NULL,
    -- dining | stays | bar | wellness | events | retail

    -- Subscription
    subscription_plan   VARCHAR(20)     NOT NULL,
    -- starter | professional | enterprise
    subscription_status VARCHAR(20)     NOT NULL,
    -- trial | active | past_due | cancelled | suspended
    billing_cycle       VARCHAR(10)     NOT NULL,
    -- monthly | annual
    trial_ends_at       TIMESTAMP WITH TIME ZONE,
    next_billing_at     TIMESTAMP WITH TIME ZONE,

    -- Lifecycle
    enabled_at          TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    disabled_at         TIMESTAMP WITH TIME ZONE,  -- null = currently active

    CONSTRAINT business_operations_unique UNIQUE (business_id, operation_type)
);

CREATE INDEX idx_business_operations_business_id ON business_operations(business_id);
CREATE INDEX idx_business_operations_status      ON business_operations(subscription_status);
```

**Module boundary rule:**

```
Business module  → Is Dining enabled? What plan? What billing cycle?
Dining module    → dining_settings table: order prefix, kitchen buffer, receipt footer...
Stays module     → stays_settings table: check-in time, check-out time, night audit time...
```

Each operation module creates its own settings row when it handles `BusinessOperationEnabledEvent`.
Business module does not know what config any operation needs.

---

### Table 6: `business_operation_addons`

Add-ons enabled per operation. Normalized from JSONB array — proper table for queryability and audit.

```sql
CREATE TABLE business_operation_addons (
    id                      UUID            PRIMARY KEY DEFAULT gen_random_uuid(),
    business_operation_id   UUID            NOT NULL REFERENCES business_operations(id),

    addon_type              VARCHAR(50)     NOT NULL,
    -- dining:  reservation | qr_ordering | kds | delivery | takeaway
    -- stays:   housekeeping | maintenance | channel_manager
    -- bar:     kds

    is_active               BOOLEAN         NOT NULL DEFAULT true,
    config                  JSONB           NOT NULL DEFAULT '{}',

    enabled_at              TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    disabled_at             TIMESTAMP WITH TIME ZONE,

    CONSTRAINT business_operation_addons_unique UNIQUE (business_operation_id, addon_type)
);

CREATE INDEX idx_addons_operation_id ON business_operation_addons(business_operation_id);
CREATE INDEX idx_addons_type         ON business_operation_addons(addon_type);
```

**Why proper table (not JSONB array):**

```
JSONB ["reservation", "kds"]  → no FK, no audit trail, no per-addon config, no queryability
Proper table                  → "how many businesses have KDS?" is a simple query
                              → enabled_at per addon, config per addon, full audit trail
```

---

### Table 7: `business_hours`

Regular weekly schedule. 7 rows per business (auto-created on business creation).

```sql
CREATE TABLE business_hours (
    id              UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    business_id     UUID        NOT NULL REFERENCES businesses(id),

    day_of_week     SMALLINT    NOT NULL,
    -- 0=Sunday, 1=Monday, 2=Tuesday, 3=Wednesday, 4=Thursday, 5=Friday, 6=Saturday

    is_open         BOOLEAN     NOT NULL DEFAULT true,
    open_time       TIME,                      -- null if is_open=false OR is_open_all_day=true
    close_time      TIME,                      -- null if is_open=false OR is_open_all_day=true
    closes_next_day BOOLEAN     NOT NULL DEFAULT false,
    -- bar opens 18:00, closes 03:00 → close_time='03:00', closes_next_day=true
    is_open_all_day BOOLEAN     NOT NULL DEFAULT false,
    -- hotel front desk 24hr → is_open_all_day=true, open_time/close_time null

    CONSTRAINT business_hours_unique    UNIQUE (business_id, day_of_week),
    CONSTRAINT business_hours_day_range CHECK  (day_of_week BETWEEN 0 AND 6)
);
```

---

### Table 8: `business_hour_exceptions`

Date-range overrides. Single day or multi-day. Exception always wins over regular schedule.

```sql
CREATE EXTENSION IF NOT EXISTS btree_gist;

CREATE TABLE business_hour_exceptions (
    id              UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    business_id     UUID        NOT NULL REFERENCES businesses(id),

    start_date      DATE        NOT NULL,
    end_date        DATE        NOT NULL,
    -- single day  → start_date = end_date  (Christmas Day)
    -- date range  → start_date < end_date  (Ramadan schedule, renovation)

    is_closed       BOOLEAN     NOT NULL DEFAULT false,
    open_time       TIME,                      -- null if is_closed or is_open_all_day
    close_time      TIME,                      -- null if is_closed or is_open_all_day
    closes_next_day BOOLEAN     NOT NULL DEFAULT false,
    is_open_all_day BOOLEAN     NOT NULL DEFAULT false,

    reason          VARCHAR(100),
    -- "Christmas Day", "Ramadan Schedule", "New Year's Eve", "Renovation"

    CONSTRAINT exception_dates_valid       CHECK (end_date >= start_date),
    CONSTRAINT no_overlapping_exceptions   EXCLUDE USING gist (
        business_id WITH =,
        daterange(start_date, end_date, '[]') WITH &&
    )
);

CREATE INDEX idx_hour_exceptions_business_dates
    ON business_hour_exceptions(business_id, start_date, end_date);
```

**Query logic — "Is this business open at 7pm on 2025-12-25?"**

```sql
-- Step 1: check for exception covering this date
SELECT * FROM business_hour_exceptions
WHERE business_id = ?
  AND start_date <= '2025-12-25'
  AND end_date   >= '2025-12-25';
-- Found: is_closed = true → CLOSED. Stop.

-- Step 2: no exception found → fall back to regular schedule
SELECT * FROM business_hours
WHERE business_id = ? AND day_of_week = 4;  -- Thursday
-- is_open=true, open_time=09:00, close_time=22:00 → 19:00 is within range → OPEN
```

**Rule: exception always wins over regular schedule.**

**Usage examples:**

```
Ramadan 2025 (30 days):
  start_date='2025-03-01', end_date='2025-03-30'
  open_time='18:00', close_time='23:00'
  reason='Ramadan Schedule'

Christmas Day (single day, closed):
  start_date='2025-12-25', end_date='2025-12-25'
  is_closed=true
  reason='Christmas Day'

New Year's Eve (single day, late close):
  start_date='2025-12-31', end_date='2025-12-31'
  open_time='09:00', close_time='03:00', closes_next_day=true
  reason="New Year's Eve"

Renovation (week closed):
  start_date='2025-08-10', end_date='2025-08-17'
  is_closed=true
  reason='Annual Renovation'
```

---

### Table 9: `business_settings`

Business-wide display preferences only. No operation-specific config here.

```sql
CREATE TABLE business_settings (
    id          UUID    PRIMARY KEY DEFAULT gen_random_uuid(),
    business_id UUID    NOT NULL UNIQUE REFERENCES businesses(id),  -- 1:1

    settings    JSONB   NOT NULL DEFAULT '{}',

    updated_at  TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);
```

**JSONB contains global-only settings — applies equally to ALL operations:**

```json
{
  "date_format": "DD/MM/YYYY",
  "time_format": "12h",
  "week_start_day": 1,
  "receipt_show_logo": true
}
```

**Rule:**
```
If a setting is operation-specific   → business_operations.config (owned by that operation)
If a setting needs querying          → proper column / proper table
If a setting is truly global/display → business_settings JSONB
```

---

### Table 10: `business_tax_rates`

Tax rate master list. Defined at business level (jurisdiction). Rates are referenced by billing in operation modules.

```sql
CREATE TABLE business_tax_rates (
    id          UUID            PRIMARY KEY DEFAULT gen_random_uuid(),
    business_id UUID            NOT NULL REFERENCES businesses(id),

    name        VARCHAR(100)    NOT NULL,  -- "VAT", "Service Charge", "GST", "Tourism Levy"
    code        VARCHAR(20),              -- short code for receipts: "VAT", "SC", "TL"
    rate        DECIMAL(5,2)    NOT NULL, -- 15.00 = 15%

    is_active   BOOLEAN         NOT NULL DEFAULT true,
    -- deactivated rates kept for historical order reference

    created_at  TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_business_tax_rates_business_id ON business_tax_rates(business_id);
```

---

### Table 11: `business_operation_tax_rates`

Junction table — which tax rates apply to which operation.

```sql
CREATE TABLE business_operation_tax_rates (
    id                      UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    business_operation_id   UUID        NOT NULL REFERENCES business_operations(id),
    tax_rate_id             UUID        NOT NULL REFERENCES business_tax_rates(id),

    is_auto_applied         BOOLEAN     NOT NULL DEFAULT true,
    -- true  = automatically applied to every order in this operation
    -- false = optional, staff selects manually (e.g., government exemption cases)

    CONSTRAINT operation_tax_rate_unique UNIQUE (business_operation_id, tax_rate_id)
);
```

**Example data — Sri Lanka hotel:**

```
business_tax_rates:
  [1] VAT              18%
  [2] Service Charge   10%
  [3] Tourism Levy      2%

business_operation_tax_rates:
  Dining  → [1] VAT ✓ auto,  [2] SC ✓ auto,  [3] Tourism Levy ✗
  Stays   → [1] VAT ✓ auto,  [2] SC ✓ auto,  [3] Tourism Levy ✓ auto
  Retail  → [1] VAT ✓ auto,  [2] SC ✗,        [3] Tourism Levy ✗
```

---

## Entity Relationships

```
users
  │
  └── businesses (owner_id)
        │
        ├── business_profiles             (1:1)
        ├── business_addresses            (1:1)
        ├── business_images               (1:many, typed)
        ├── business_hours                (1:7 — one per day)
        ├── business_hour_exceptions      (1:many — date-range overrides)
        ├── business_settings             (1:1)
        ├── business_tax_rates            (1:many — master list)
        └── business_operations           (1:many — one per operation type)
              │
              ├── business_operation_addons      (1:many)
              └── business_operation_tax_rates   (1:many → references tax_rates)
```

---

## Module Boundary

Business module owns all 11 tables. Other modules access data ONLY through `IBusinessService` in `Shared.Contracts`:

```csharp
public interface IBusinessService
{
    Task<BusinessDto>    GetBusiness(Guid businessId);
    Task<bool>           IsOperationEnabled(Guid businessId, string operationType);
    Task<bool>           IsAddonEnabled(Guid businessId, string operationType, string addonType);
    Task<string>         GetOperationConfig(Guid businessId, string operationType);
    Task<bool>           IsBusinessOpen(Guid businessId, DateTime at);
    Task<List<TaxRateDto>> GetOperationTaxRates(Guid businessId, string operationType);
}
```

No other module queries `businesses`, `business_tax_rates`, or any other Business table directly. Ever.

---

## V1 Scope

| Table | V1 | Notes |
|---|---|---|
| `businesses` | Build | Core identity |
| `business_profiles` | Build | Contact details |
| `business_addresses` | Build | Physical address + geo |
| `business_images` | Build | Logo, cover, gallery |
| `business_operations` | Build | Operations + subscriptions |
| `business_operation_addons` | Build | Add-ons per operation |
| `business_hours` | Build | Regular weekly schedule |
| `business_hour_exceptions` | Build | Holiday/seasonal overrides |
| `business_settings` | Build | Global display prefs |
| `business_tax_rates` | Build | Tax master list |
| `business_operation_tax_rates` | Build | Tax per operation mapping |
| `business_links` | Phase 2 | Cross-business resource sharing |
| Full branch logic | Phase 2 | `parent_business_id` field ready |

---

## API Endpoints

### Business Management

| Method | Endpoint | Description | Auth |
|---|---|---|---|
| `POST` | `/api/businesses` | Create new business | Owner JWT |
| `GET` | `/api/businesses` | List owner's businesses | Owner JWT |
| `GET` | `/api/businesses/{id}` | Get business details | Staff JWT |
| `PUT` | `/api/businesses/{id}` | Update core business info | Owner JWT |
| `GET` | `/api/businesses/resolve/{slug}` | Resolve business by slug | Public (tenant resolution) |

### Profile, Address, Images

| Method | Endpoint | Description | Auth |
|---|---|---|---|
| `GET` | `/api/businesses/{id}/profile` | Get contact details | Staff JWT |
| `PUT` | `/api/businesses/{id}/profile` | Update contact details | Owner/Manager JWT |
| `GET` | `/api/businesses/{id}/address` | Get address | Staff JWT |
| `PUT` | `/api/businesses/{id}/address` | Update address | Owner JWT |
| `GET` | `/api/businesses/{id}/images` | List images | Staff JWT |
| `POST` | `/api/businesses/{id}/images` | Upload image | Owner/Manager JWT |
| `PUT` | `/api/businesses/{id}/images/{imageId}` | Update image metadata | Owner/Manager JWT |
| `DELETE` | `/api/businesses/{id}/images/{imageId}` | Remove image | Owner JWT |

### Operations & Add-ons

| Method | Endpoint | Description | Auth |
|---|---|---|---|
| `POST` | `/api/businesses/{id}/operations` | Enable an operation | Owner JWT |
| `GET` | `/api/businesses/{id}/operations` | List enabled operations | Staff JWT |
| `DELETE` | `/api/businesses/{id}/operations/{type}` | Disable an operation | Owner JWT |
| `POST` | `/api/businesses/{id}/operations/{type}/addons` | Enable add-on | Owner JWT |
| `DELETE` | `/api/businesses/{id}/operations/{type}/addons/{addon}` | Disable add-on | Owner JWT |

### Hours

| Method | Endpoint | Description | Auth |
|---|---|---|---|
| `GET` | `/api/businesses/{id}/hours` | Get weekly schedule | Staff JWT |
| `PUT` | `/api/businesses/{id}/hours` | Update weekly schedule | Owner/Manager JWT |
| `GET` | `/api/businesses/{id}/hours/exceptions` | List exceptions | Staff JWT |
| `POST` | `/api/businesses/{id}/hours/exceptions` | Create exception | Owner/Manager JWT |
| `PUT` | `/api/businesses/{id}/hours/exceptions/{exId}` | Update exception | Owner/Manager JWT |
| `DELETE` | `/api/businesses/{id}/hours/exceptions/{exId}` | Delete exception | Owner/Manager JWT |

### Settings

| Method | Endpoint | Description | Auth |
|---|---|---|---|
| `GET` | `/api/businesses/{id}/settings` | Get settings | Staff JWT |
| `PUT` | `/api/businesses/{id}/settings` | Update settings | Owner/Manager JWT |

### Tax Rates

| Method | Endpoint | Description | Auth |
|---|---|---|---|
| `GET` | `/api/businesses/{id}/tax-rates` | List tax rates | Staff JWT |
| `POST` | `/api/businesses/{id}/tax-rates` | Create tax rate | Owner JWT |
| `PUT` | `/api/businesses/{id}/tax-rates/{rateId}` | Update tax rate | Owner JWT |
| `DELETE` | `/api/businesses/{id}/tax-rates/{rateId}` | Deactivate tax rate | Owner JWT |
| `GET` | `/api/businesses/{id}/operations/{type}/tax-rates` | Get taxes for operation | Staff JWT |
| `PUT` | `/api/businesses/{id}/operations/{type}/tax-rates` | Assign taxes to operation | Owner JWT |

---

## Key Business Rules

### Business Creation
1. Owner must be authenticated (Identity JWT)
2. `slug` must be unique across all businesses — validate before save
3. `legal_name` + `trading_name` both required — can be the same value
4. On creation: publish `BusinessCreatedEvent` → Membership module auto-creates owner record
5. On creation: auto-create `business_settings` row (1:1, always exists)
6. On creation: auto-create 7 `business_hours` rows (one per day, default open Mon–Sun 09:00–22:00)
7. First operation enabled → set `activated_at` on businesses

### Slug Rules
- Lowercase, letters + numbers + hyphens only
- Min 3 chars, max 100 chars — cannot start or end with hyphen
- Reserved words blocked: `app`, `api`, `admin`, `www`, `mail`, `support`, `onenex`
- Slug change allowed by owner only — old slug not reserved after change

### Operation Rules
- Same operation type cannot be enabled twice per business (UNIQUE constraint)
- Each new operation starts `subscription_status = 'trial'`
- Add-on requires parent operation to be enabled first
- Add-on dependencies enforced in application layer before enabling

### Tax Rate Rules
- Multiple active rates allowed per business
- Deactivating a rate does NOT delete it — historical orders reference it
- Rate of 0.00 valid (tax-exempt businesses)
- Billing in operation modules reads rates through `IBusinessService.GetOperationTaxRates()`

### Hours Rules
- Exception always wins over regular schedule
- Overlapping date ranges rejected at DB level (EXCLUDE constraint)
- `closes_next_day=true` for operations that close after midnight

---

## Domain Events

```csharp
public record BusinessCreatedEvent(
    Guid BusinessId,
    Guid OwnerId,
    string TradingName,
    string Slug
) : IDomainEvent;

public record BusinessOperationEnabledEvent(
    Guid BusinessId,
    string OperationType,
    string SubscriptionPlan
) : IDomainEvent;

public record BusinessOperationDisabledEvent(
    Guid BusinessId,
    string OperationType
) : IDomainEvent;

public record BusinessSuspendedEvent(
    Guid BusinessId,
    string Reason
) : IDomainEvent;
```

---

## Project Structure

```
Modules/Business/
├── Domain/
│   ├── Entities/
│   │   ├── Business.cs
│   │   ├── BusinessProfile.cs
│   │   ├── BusinessAddress.cs
│   │   ├── BusinessImage.cs
│   │   ├── BusinessOperation.cs
│   │   ├── BusinessOperationAddon.cs
│   │   ├── BusinessHours.cs
│   │   ├── BusinessHourException.cs
│   │   ├── BusinessSettings.cs
│   │   ├── BusinessTaxRate.cs
│   │   └── BusinessOperationTaxRate.cs
│   ├── Events/
│   │   ├── BusinessCreatedEvent.cs
│   │   ├── BusinessOperationEnabledEvent.cs
│   │   ├── BusinessOperationDisabledEvent.cs
│   │   └── BusinessSuspendedEvent.cs
│   └── ValueObjects/
│       ├── BusinessSlug.cs
│       ├── OperationType.cs
│       └── SubscriptionStatus.cs
│
├── Application/
│   └── Features/
│       ├── Businesses/
│       │   ├── Commands/
│       │   │   ├── CreateBusiness/
│       │   │   └── UpdateBusiness/
│       │   └── Queries/
│       │       ├── GetBusiness/
│       │       ├── GetOwnerBusinesses/
│       │       └── ResolveBusinessBySlug/
│       ├── Profile/
│       │   ├── Commands/UpdateBusinessProfile/
│       │   └── Queries/GetBusinessProfile/
│       ├── Address/
│       │   ├── Commands/UpdateBusinessAddress/
│       │   └── Queries/GetBusinessAddress/
│       ├── Images/
│       │   ├── Commands/
│       │   │   ├── UploadBusinessImage/
│       │   │   ├── UpdateBusinessImage/
│       │   │   └── RemoveBusinessImage/
│       │   └── Queries/GetBusinessImages/
│       ├── Operations/
│       │   ├── Commands/
│       │   │   ├── EnableOperation/
│       │   │   ├── DisableOperation/
│       │   │   ├── EnableAddon/
│       │   │   └── DisableAddon/
│       │   └── Queries/GetBusinessOperations/
│       ├── Hours/
│       │   ├── Commands/
│       │   │   ├── UpdateBusinessHours/
│       │   │   ├── CreateHourException/
│       │   │   ├── UpdateHourException/
│       │   │   └── DeleteHourException/
│       │   └── Queries/
│       │       ├── GetBusinessHours/
│       │       ├── GetHourExceptions/
│       │       └── CheckBusinessOpen/
│       ├── Settings/
│       │   ├── Commands/UpdateBusinessSettings/
│       │   └── Queries/GetBusinessSettings/
│       └── TaxRates/
│           ├── Commands/
│           │   ├── CreateTaxRate/
│           │   ├── UpdateTaxRate/
│           │   ├── DeactivateTaxRate/
│           │   └── AssignOperationTaxRates/
│           └── Queries/
│               ├── GetBusinessTaxRates/
│               └── GetOperationTaxRates/
│
├── Infrastructure/
│   ├── Repositories/
│   │   ├── BusinessRepository.cs
│   │   └── BusinessOperationRepository.cs
│   ├── EntityConfigurations/
│   │   ├── BusinessConfiguration.cs
│   │   ├── BusinessProfileConfiguration.cs
│   │   ├── BusinessAddressConfiguration.cs
│   │   ├── BusinessImageConfiguration.cs
│   │   ├── BusinessOperationConfiguration.cs
│   │   ├── BusinessOperationAddonConfiguration.cs
│   │   ├── BusinessHoursConfiguration.cs
│   │   ├── BusinessHourExceptionConfiguration.cs
│   │   ├── BusinessSettingsConfiguration.cs
│   │   ├── BusinessTaxRateConfiguration.cs
│   │   └── BusinessOperationTaxRateConfiguration.cs
│   └── BusinessDbContext.cs
│
└── API/
    └── Controllers/
        ├── BusinessesController.cs
        ├── BusinessProfileController.cs
        ├── BusinessAddressController.cs
        ├── BusinessImagesController.cs
        ├── BusinessOperationsController.cs
        ├── BusinessHoursController.cs
        ├── BusinessSettingsController.cs
        └── BusinessTaxRatesController.cs
```

---

## Open Questions (Discuss with Team)

- Slug change — soft redirect for old slug or immediate 404?
- `business_operations.config` JSONB — validate schema in application layer per operation type?
- Image upload — direct to S3/Blob from client (presigned URL) or through API?
- `activated_at` — set automatically on first operation enable, or owner manually confirms "go live"?
- Branch concept (Phase 2) — branch inherits parent hours/settings or fully independent?
- Subscription billing — Business module tracks state, but who triggers actual payment charge? Payment Service or external billing provider (Stripe)?
- `btree_gist` extension — confirm available on target PostgreSQL hosting environment.
