# OneNex Business Module — Final Implementation Design

> Revised architecture incorporating the identified gaps and decisions.
>
> **Core principle:** Business identity ≠ enabled operation ≠ subscription entitlement ≠ authorization.

## 1. Module Responsibility

The Business module owns business identity and business-level configuration.

### Business module owns

- Business core identity and locale(a place where something happens or is set, or that has particular events associated with it)
- Business profile and contact information
- Business address
- Business images
- Enabled operations
- Operation add-on enablement
- Business-level operating hours
- Business-hour exceptions
- Business-wide settings
- Tax definitions
- Operation-level tax mapping
- Business lifecycle state
- Slug history

### Business module does NOT own

- User/staff identity
- Membership, roles and permissions
- Actual Dining/Stays/Bar/Wellness/Events/Retail domain behavior
- Payment charging
- Subscription plans and invoices
- Notifications
- Operation-specific domain configuration

---

# 2. Operation Model — Enum, Not a Modules Table


Operation types are controlled by the application/domain layer.

```csharp
public enum OperationType     //finalize the enums
{
    Dining,
    Stays,
    Bar,
    Wellness,
    Events,
    Retail
}
```

The database stores stable string values:
The enum is the controlled catalog.

Adding a new core OneNex operation requires a code deployment/migration.
---

# 3. Core Database Schema

## 3.1 `businesses`

```sql
CREATE TABLE businesses (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    business_code       VARCHAR(30) NOT NULL UNIQUE,
    owner_id            UUID NOT NULL REFERENCES users(id),

    legal_name          VARCHAR(200) NOT NULL,
    trading_name        VARCHAR(200) NOT NULL,
    slug                VARCHAR(100) NOT NULL UNIQUE,

    country_code        CHAR(2) NOT NULL,
    timezone            VARCHAR(50) NOT NULL,
    currency_code       CHAR(3) NOT NULL,
    default_language    CHAR(2) NOT NULL DEFAULT 'en',

    contact_email       VARCHAR(254),
    contact_phone       VARCHAR(30),

    status              VARCHAR(20) NOT NULL DEFAULT 'active',
    onboarding_status   VARCHAR(30) NOT NULL DEFAULT 'setup',

    activated_at        TIMESTAMPTZ,
    suspended_at        TIMESTAMPTZ,
    closed_at           TIMESTAMPTZ,

    version             BIGINT NOT NULL DEFAULT 1,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    CONSTRAINT chk_business_status
        CHECK (status IN ('active','suspended','closed')),

    CONSTRAINT chk_onboarding_status
        CHECK (onboarding_status IN ('setup','ready','completed'))
);

CREATE INDEX idx_businesses_owner_id
    ON businesses(owner_id);

CREATE INDEX idx_businesses_status
    ON businesses(status);

CREATE INDEX idx_businesses_slug
    ON businesses(slug);
```

### Purpose

`businesses` is the lean tenant identity.

It answers:

> **Who is this business?**

It does not answer which OneNex operations are enabled or what subscription it has.

### Important fields

| Field | Purpose |
|---|---|
| `business_code` | Stable business identifier |
| `owner_id` | Original/legal owner reference |
| `slug` | URL identity |
| `country_code` | Locale/tax context |
| `timezone` | Business-local time |
| `currency_code` | Default currency |
| `status` | Business lifecycle |
| `onboarding_status` | Setup/readiness |
| `version` | Optimistic concurrency |

`owner_id` is **not** an authorization mechanism.
**But versioning is not mandatory** 
Since you're using DDD + CQRS, this becomes even more useful.

Your command can carry:

public record UpdateBusinessCommand(
    Guid BusinessId,
    string TradingName,
    long ExpectedVersion
);

Then the aggregate/repository can enforce:

ExpectedVersion == CurrentVersion
        ↓
       YES → apply change → version++
        ↓
       NO → concurrency conflict

---

## 3.2 `business_slug_history`

```sql
CREATE TABLE business_slug_history (
    id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    business_id  UUID NOT NULL REFERENCES businesses(id),
    slug         VARCHAR(100) NOT NULL,
    created_at   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    replaced_at  TIMESTAMPTZ
);

CREATE UNIQUE INDEX uq_business_slug_history_slug
    ON business_slug_history(slug);
```

### Purpose

When:

```text
grandhotel.onenex.com
```

changes to:

```text
grandhotel-colombo.onenex.com
```

the old slug remains in history.

This allows old bookmarks, QR codes and URLs to redirect.

---

## 3.3 `branches`

> **Decided (supersedes §34 "Branch / Location Direction" below):** a branch is a **location under one tenant** (Option B), not a separate business. `parent_business_id` is removed from `businesses` — it is replaced by this table.

```sql
CREATE TABLE branches (
    id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    business_id       UUID NOT NULL REFERENCES businesses(id),

    branch_code       VARCHAR(30) NOT NULL,
    name              VARCHAR(200) NOT NULL,

    is_headquarters   BOOLEAN NOT NULL DEFAULT FALSE,
    timezone          VARCHAR(50),   -- NULL = inherit businesses.timezone

    status            VARCHAR(20) NOT NULL DEFAULT 'active',

    version           BIGINT NOT NULL DEFAULT 1,
    created_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    CONSTRAINT uq_branch_code
        UNIQUE (business_id, branch_code),

    CONSTRAINT chk_branch_status
        CHECK (status IN ('active','suspended','closed'))
);

CREATE INDEX idx_branches_business_id
    ON branches(business_id);

CREATE UNIQUE INDEX uq_business_single_headquarters
    ON branches(business_id)
    WHERE is_headquarters = TRUE;
```

### Purpose

`branches` answers:

> **Where does this business operate, and which one am I logged into right now?**

It does not answer who can access a given branch — that is a Membership concern (`staff_branch_access`, see the Membership module).

### Rules

```text
Every business has at least one branch — the headquarters (HQ) —
created automatically when the business is created (see §27).

Exactly one branch per business may have is_headquarters = TRUE
(enforced by the partial unique index).

A branch cannot be deleted — only disabled (status = 'closed'),
same lifecycle discipline as businesses. Domain data created under
a branch is never deleted when the branch is closed.
```

### Relationship to `business_addresses` and `business_hours`

Each branch is a physical location, so it needs its own address and — often — its own hours:

```text
business_addresses.branch_id  (nullable)
    NULL      → business's default/legal address
    NOT NULL  → that specific branch's address

business_hours.branch_id  (nullable)
    NULL      → applies business-wide (fallback)
    NOT NULL  → overrides for that specific branch
```

See §5 and §11 for the updated column definitions.

---

# 4. Business Profile

## `business_profiles`

```sql
CREATE TABLE business_profiles (
    id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    business_id  UUID NOT NULL UNIQUE REFERENCES businesses(id),
    phone        VARCHAR(30),
    email        VARCHAR(254),
    website_url  VARCHAR(500),
    description  TEXT,
    updated_at   TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

If multiple contacts become necessary, introduce:

```text
business_contacts
```

with:

```text
contact_type
name
phone
email
```

---

# 5. Business Address

## `business_addresses`

```sql
CREATE TABLE business_addresses (
    id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    business_id   UUID NOT NULL REFERENCES businesses(id),
    branch_id     UUID REFERENCES branches(id),   -- NULL = business default/legal address
    address_line1 VARCHAR(255),
    address_line2 VARCHAR(255),
    city          VARCHAR(100),
    state         VARCHAR(100),
    postal_code   VARCHAR(20),
    latitude      DECIMAL(10,7),
    longitude     DECIMAL(10,7),
    updated_at    TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE UNIQUE INDEX uq_business_default_address
    ON business_addresses(business_id)
    WHERE branch_id IS NULL;

CREATE UNIQUE INDEX uq_branch_address
    ON business_addresses(branch_id)
    WHERE branch_id IS NOT NULL;
```

V1 supports one address per branch, plus one business-level default (`branch_id IS NULL`) used when a branch has not set its own address.

Branches are a V1 concept — see §3.3.

---

# 6. Business Images

## `business_images`

```sql
CREATE TABLE business_images (
    id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    business_id   UUID NOT NULL REFERENCES businesses(id),
    image_type    VARCHAR(20) NOT NULL,
    storage_key   VARCHAR(500) NOT NULL,
    url           VARCHAR(500),
    mime_type     VARCHAR(100),  
    A MIME type (Multipurpose Internet Mail Extensions) is a standardized way to indicate the nature and format of a file so that browsers, APIs, or applications know how to handle it. Eg: image/png
    file_size     BIGINT,
    width         INTEGER,
    height        INTEGER,
    alt_text      VARCHAR(200),
    display_order SMALLINT NOT NULL DEFAULT 0,
    is_active     BOOLEAN NOT NULL DEFAULT TRUE,
    created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    CONSTRAINT chk_image_type
        CHECK (image_type IN ('logo','cover','gallery'))
);

CREATE INDEX idx_business_images_business_type
    ON business_images(business_id, image_type);

CREATE UNIQUE INDEX uq_business_active_logo
    ON business_images(business_id)
    WHERE image_type = 'logo'
      AND is_active = TRUE;
```

The partial unique index guarantees only one active logo while allowing historical logos.

---

# 7. Business Operations — Single Source of Truth

`business_operations` replaces both the previous modules/business_modules concept and the operation enablement portion of the original model.

It answers:

> **What OneNex operation is enabled for this business?**

It does **not** own subscription billing state.

## `business_operations`

```sql
CREATE TABLE business_operations (
    id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    business_id      UUID NOT NULL REFERENCES businesses(id),
    operation_type   VARCHAR(30) NOT NULL,

    status           VARCHAR(20) NOT NULL DEFAULT 'enabled',
    enabled_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    disabled_at      TIMESTAMPTZ,

    version          BIGINT NOT NULL DEFAULT 1,
    created_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    CONSTRAINT uq_business_operation
        UNIQUE (business_id, operation_type),

    CONSTRAINT chk_operation_status
        CHECK (status IN ('enabled','suspended','disabled'))
);

CREATE INDEX idx_business_operations_business
    ON business_operations(business_id);

CREATE INDEX idx_business_operations_status
    ON business_operations(status);
```

### Example

Business A initially:

```text
business_operations

Business A | dining | enabled
```

Later:

```text
Business A | dining | enabled
Business A | stays  | enabled
```

The `businesses` record does not change.

This is exactly what is needed when a business starts with Dining and later adds Stays.

---

# 8. Enabled vs Configured vs Ready

A critical distinction:

> **Enabled does not mean Ready.**

Example:

```text
Stays enabled
        ↓
No room types
        ↓
Not ready for reservations
```

Recommended conceptual states:

| State | Meaning |
|---|---|
| `enabled` | Operation has been activated |
| `configuring` | Setup is in progress |
| `ready` | Operation-specific readiness requirements pass |
| `suspended` | Temporarily unavailable |
| `disabled` | Not currently offered |

The Business module can own enablement/lifecycle.

The actual readiness rules belong to the operation module.

For example:

### Stays module

```text
room types configured?
rates configured?
availability configured?
policies configured?
```

Only the Stays module should decide whether Stays is fully ready.

---

# 9. Operation Add-ons

## `business_operation_addons`

```sql
CREATE TABLE business_operation_addons (
    id                    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    business_operation_id UUID NOT NULL REFERENCES business_operations(id),
    addon_type            VARCHAR(50) NOT NULL,

    is_active             BOOLEAN NOT NULL DEFAULT TRUE,
    config                JSONB NOT NULL DEFAULT '{}',

    enabled_at            TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    disabled_at           TIMESTAMPTZ,

    created_at            TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at            TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    CONSTRAINT uq_business_operation_addon
        UNIQUE (business_operation_id, addon_type)
);

CREATE INDEX idx_addons_operation_id
    ON business_operation_addons(business_operation_id);
```

Example:

```text
Dining
 ├── reservation
 ├── qr_ordering
 ├── kds
 ├── delivery
 └── takeaway
```

Add-on dependency:

```text
KDS
 ↓
requires Dining
```

Dependencies are enforced in the application/domain layer.

---

# 10. Subscription and Billing — Separate Module

Remove these fields from `business_operations`:

```text
subscription_plan
subscription_status
billing_cycle
trial_ends_at
next_billing_at
```

Those fields mix two different concepts.

## Business Operations

Answers:

> Is Dining enabled?

## Subscription

Answers:

> Is this business commercially entitled to Dining?

## Billing

Answers:

> What should be charged and has payment succeeded?

The Subscription & Billing module should own concepts such as:

```text
subscriptions
subscription_items
plans
invoices
payment status
billing periods
trial
cancellation
```

Conceptually:

```text
subscriptions
    id
    business_id
    plan_id
    status
    billing_cycle
    starts_at
    trial_ends_at
    current_period_start
    current_period_end
    cancelled_at
    created_at
    updated_at
```

```text
subscription_items
    id
    subscription_id
    operation_type
    quantity / limits / entitlement data
    status
    starts_at
    ends_at
```

The exact billing schema belongs to Subscription & Billing.

---

# 11. Business Hours

A single `open_time` / `close_time` pair is insufficient.

Example:

```text
Monday
11:00–15:00
17:00–23:00
```

Therefore use a parent schedule + interval table.

## `business_hours`

```sql
CREATE TABLE business_hours (
    id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    business_id   UUID NOT NULL REFERENCES businesses(id),
    branch_id     UUID REFERENCES branches(id),   -- NULL = business-wide fallback
    day_of_week   SMALLINT NOT NULL,
    is_open       BOOLEAN NOT NULL DEFAULT TRUE,

    created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    CHECK (day_of_week BETWEEN 0 AND 6)
);

CREATE UNIQUE INDEX uq_business_default_hours
    ON business_hours(business_id, day_of_week)
    WHERE branch_id IS NULL;

CREATE UNIQUE INDEX uq_branch_hours
    ON business_hours(branch_id, day_of_week)
    WHERE branch_id IS NOT NULL;
```

## `business_hour_intervals`

```sql
CREATE TABLE business_hour_intervals (
    id                 UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    business_hours_id  UUID NOT NULL REFERENCES business_hours(id),

    open_time          TIME NOT NULL,
    close_time         TIME NOT NULL,
    closes_next_day    BOOLEAN NOT NULL DEFAULT FALSE,

    display_order      SMALLINT NOT NULL DEFAULT 0,

    UNIQUE (business_hours_id, display_order)
);
```

### Example

```text
Monday
 ├── 11:00 → 15:00
 └── 17:00 → 23:00
```

For overnight:

```text
18:00 → 03:00
closes_next_day = true
```

---

# 12. Operation-Specific Hours

Overall business hours are not necessarily operation hours.

Example:

```text
Hotel
 └── Business hours: 24/7

Dining
 └── 07:00–22:00

Spa
 └── 09:00–20:00
```

Therefore:

- Business module owns overall business hours.
- Dining owns Dining-specific hours.
- Stays owns Stays-specific hours.
- Bar owns Bar-specific hours.

The final transaction check may combine:

```text
Business active?
        +
Operation enabled?
        +
Operation ready?
        +
Operation open?
```

---

# 13. Business Hour Exceptions

```sql
CREATE EXTENSION IF NOT EXISTS btree_gist;

CREATE TABLE business_hour_exceptions (
    id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    business_id      UUID NOT NULL REFERENCES businesses(id),

    start_date       DATE NOT NULL,
    end_date         DATE NOT NULL,

    is_closed        BOOLEAN NOT NULL DEFAULT FALSE,
    is_open_all_day  BOOLEAN NOT NULL DEFAULT FALSE,

    reason           VARCHAR(200),

    created_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    CONSTRAINT chk_exception_dates
        CHECK (end_date >= start_date),

    CONSTRAINT no_overlapping_business_exceptions
        EXCLUDE USING gist (
            business_id WITH =,
            daterange(start_date, end_date, '[]') WITH &&
        )
);
```

### Rule

> Exception always wins over regular schedule.

Example:

```text
Normal:
Monday 09:00–22:00

Christmas:
Closed

Result:
Christmas exception wins.
```

If exceptions need different opening intervals, introduce exception interval rows similar to `business_hour_intervals`.

---

# 14. Business Settings

```sql
CREATE TABLE business_settings (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    business_id UUID NOT NULL UNIQUE REFERENCES businesses(id),

    settings    JSONB NOT NULL DEFAULT '{}',

    version     BIGINT NOT NULL DEFAULT 1,
    updated_by  UUID REFERENCES users(id),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

Use JSONB for genuinely global/display settings:

```json
{
  "date_format": "DD/MM/YYYY",
  "time_format": "12h",
  "week_start_day": 1,
  "receipt_show_logo": true
}
```

### Rule

```text
Operation-specific
    → operation module

Needs querying/filtering/constraints
    → proper column/table

Truly global/display preference
    → business_settings JSONB
```

---

# 15. Tax Model

Tax registration, tax definitions, applicability and transaction snapshots are separate concepts.

---

## 15.1 Tax Profile

```sql
CREATE TABLE business_tax_profiles (
    id                        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    business_id               UUID NOT NULL UNIQUE REFERENCES businesses(id),

    tax_identification_number VARCHAR(100),
    tax_registration_number   VARCHAR(100),
    tax_regime                VARCHAR(100),

    default_tax_inclusive     BOOLEAN NOT NULL DEFAULT FALSE,

    status                    VARCHAR(20) NOT NULL DEFAULT 'active',

    created_at                TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at                TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

This stores the business's tax registration/profile information.

---

# 16. Tax / Charge Definitions

A business may have:

- VAT
- GST
- Tourism Levy
- Service Charge
- Government fee

These are not all technically the same kind of charge.

Therefore distinguish them.

## `business_tax_rates`

```sql
CREATE TABLE business_tax_rates (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    business_id         UUID NOT NULL REFERENCES businesses(id),

    name                VARCHAR(100) NOT NULL,
    code                VARCHAR(50) NOT NULL,

    charge_type         VARCHAR(30) NOT NULL,

    rate                NUMERIC(7,4) NOT NULL,

    calculation_method  VARCHAR(30) NOT NULL DEFAULT 'percentage',
    calculation_order   SMALLINT NOT NULL DEFAULT 1,
    is_compound         BOOLEAN NOT NULL DEFAULT FALSE,

    effective_from      DATE NOT NULL,
    effective_to        DATE,

    is_active            BOOLEAN NOT NULL DEFAULT TRUE,

    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    CONSTRAINT uq_business_tax_code_version
        UNIQUE (business_id, code, effective_from),

    CONSTRAINT chk_charge_type
        CHECK (
            charge_type IN
            ('tax','levy','service_charge','fee')
        ),

    CONSTRAINT chk_calculation_method
        CHECK (
            calculation_method IN
            ('percentage','fixed')
        ),

    CONSTRAINT chk_rate
        CHECK (rate >= 0 AND rate <= 100),

    CONSTRAINT chk_effective_dates
        CHECK (
            effective_to IS NULL
            OR effective_to >= effective_from
        )
);

CREATE INDEX idx_business_tax_rates_lookup
    ON business_tax_rates(
        business_id,
        code,
        effective_from,
        effective_to
    );
```

---

# 17. Why Tax Rates Must Be Versioned

Suppose:

```text
VAT
18%
effective from 2026-01-01
```

Later it becomes:

```text
VAT
20%
effective from 2027-01-01
```

Do **not** update the old row:

```text
18% → 20%
```

Instead:

```text
VAT version 1
18%
2026-01-01 → 2026-12-31

VAT version 2
20%
2027-01-01 → NULL
```

This allows historical transactions to reproduce their original calculation.

---

# 18. Operation Tax Mapping

## `business_operation_tax_rates`

```sql
CREATE TABLE business_operation_tax_rates (
    id                    UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    business_operation_id UUID NOT NULL
        REFERENCES business_operations(id),

    tax_rate_id           UUID NOT NULL
        REFERENCES business_tax_rates(id),

    is_auto_applied       BOOLEAN NOT NULL DEFAULT TRUE,

    priority              SMALLINT NOT NULL DEFAULT 1,

    UNIQUE (business_operation_id, tax_rate_id)
);

CREATE INDEX idx_operation_tax_rates_operation
    ON business_operation_tax_rates(business_operation_id);
```

### Example

Sri Lankan hotel:

```text
Tax definitions

VAT             18%
Service Charge  10%
Tourism Levy     2%
```

Mapping:

```text
Dining
 ├── VAT            auto
 ├── Service Charge auto
 └── Tourism Levy   no

Stays
 ├── VAT            auto
 ├── Service Charge auto
 └── Tourism Levy   auto

Retail
 ├── VAT            auto
 ├── Service Charge no
 └── Tourism Levy   no
```

---

# 19. Tax Calculation Order

When several charges apply, calculation order matters.

For example:

```text
Base amount
    ↓
Service Charge
    ↓
VAT
```

The exact order must follow the applicable tax rules.

`calculation_order` makes the sequence explicit.

`is_compound` alone is not enough to describe complex calculation behavior.

---

# 20. Tax Applicability — V1 Boundary

Operation-level mapping is appropriate for V1.

However, real-world businesses can require more granular rules.

Dining:

```text
Food
Alcohol
Takeaway
Delivery
```

Stays:

```text
Room
Minibar
Spa
Laundry
```

may all have different tax treatment.

Therefore V1 uses:

```text
Operation → Tax
```

Later, if necessary:

```text
Product/Service/Category
        ↓
Tax Rule
        ↓
Tax Rate
```

Do not introduce this complexity until actual operation requirements justify it.

---

# 21. Transaction Tax Snapshot

Historical transactions must not depend on today's tax configuration.

When an order/invoice is created, snapshot the calculation:

```text
tax_code
tax_name
charge_type
rate
calculation_method
calculation_order
taxable_amount
tax_amount
is_inclusive
```

Example:

```text
Order #1001

VAT
18%
Taxable amount: 10,000
Tax: 1,800
```

If the business later changes VAT to 20%, Order #1001 remains 18%.

---

# 22. Business Lifecycle

| Status | Meaning | Behavior |
|---|---|---|
| `active` | Operating normally | Normal access |
| `suspended` | Temporarily blocked | Operational actions blocked; data retained |
| `closed` | Permanently closed | No new operational transactions; historical data retained |

### Suspended

Possible reasons:

```text
Payment issue
Administrative action
Compliance issue
Temporary closure
```

The business remains in the database.

### Closed

Closed means the business is permanently no longer operating.

Future reservations/orders/billing need an explicit closure workflow.

---

# 23. Disable Operation — Never Delete Domain Data

Suppose:

```text
Business A
 └── Stays
       ├── rooms
       ├── reservations
       ├── guests
       └── invoices
```

Owner disables Stays.

Do **not** delete:

```text
rooms
reservations
guests
invoices
```

Instead:

```text
business_operations
Stays → disabled
```

The Stays module then prevents new Stays activity according to its rules.

If Stays is re-enabled:

```text
Stays → enabled
```

and existing domain data remains available.

---

# 25. Domain Events and Outbox

The Business module should not directly call other modules.

Instead:

```text
Business change
      ↓
Database transaction
      ↓
Business data + Outbox record
      ↓
COMMIT
      ↓
Outbox dispatcher
      ↓
Event
      ↓
Consumer module
```

This prevents:

```text
Business created successfully
BUT
event was lost
```

---

# 26. Important Domain Events

Recommended events:

```text
BusinessCreatedEvent
BusinessOperationEnabledEvent
BusinessOperationDisabledEvent
BusinessSuspendedEvent
BusinessReactivatedEvent
BusinessSlugChangedEvent
BusinessTaxRateChangedEvent
BranchCreatedEvent
BranchUpdatedEvent
BranchDeactivatedEvent
```

Example:

```csharp
public record BusinessCreatedEvent(
    Guid BusinessId,
    Guid OwnerId,
    string TradingName,
    string Slug
) : IDomainEvent;

public record BusinessOperationEnabledEvent(
    Guid BusinessId,
    string OperationType
) : IDomainEvent;

public record BusinessOperationDisabledEvent(
    Guid BusinessId,
    string OperationType
) : IDomainEvent;

public record BranchCreatedEvent(
    Guid BusinessId,
    Guid BranchId,
    string BranchCode,
    bool IsHeadquarters
) : IDomainEvent;

public record BranchDeactivatedEvent(
    Guid BusinessId,
    Guid BranchId
) : IDomainEvent;
```

`BranchCreatedEvent` is consumed by the Membership module: the business owner's membership gets implicit access to every branch (bypass), and — per product policy — existing non-owner staff are **not** auto-granted the new branch; access must be explicitly assigned (mirrors how new operations are not auto-granted to existing custom-permission staff).

---

# 27. Business Creation Flow

```text
Authenticate user
        ↓
Validate business information
        ↓
Validate slug uniqueness
        ↓
Validate business_code uniqueness
        ↓
Create businesses
        ↓
Create business_settings
        ↓
Create default business hours
        ↓
Create headquarters branch (is_headquarters = TRUE)
        ↓
Create Outbox BusinessCreatedEvent + BranchCreatedEvent
        ↓
COMMIT
        ↓
Outbox publishes events
        ↓
Membership creates owner membership
        ↓
Membership grants owner implicit access to all branches (bypass — see Membership module)
```

No operation needs to be automatically enabled unless product policy explicitly requires it.

Every business is created with exactly one branch (the HQ). Additional branches are created explicitly via `POST /api/businesses/{id}/branches` (§36).

---

# 28. Enabling an Operation

Example: owner wants to enable Stays.

```text
Authorize business.operation.manage
        ↓
Business not closed?
        ↓
Subscription entitlement valid?
        ↓
Dependencies valid?
        ↓
Create/reactivate business_operations
        ↓
Create Outbox event
        ↓
COMMIT
        ↓
Publish BusinessOperationEnabledEvent
        ↓
Stays module initializes its data
        ↓
Stays module determines readiness
```

---

# 29. Disabling an Operation

```text
Authorize business.operation.manage
        ↓
Check active reservations/orders/etc.
        ↓
Run operation-specific closure policy
        ↓
Mark operation disabled
        ↓
Create Outbox event
        ↓
COMMIT
        ↓
Operation module blocks new activity
        ↓
Historical data remains
```

---

# 29a. Creating a Branch

```text
Authorize business.branch.manage
        ↓
Business not closed?
        ↓
Validate branch_code uniqueness within business
        ↓
Create branches row (is_headquarters = FALSE)
        ↓
Create Outbox BranchCreatedEvent
        ↓
COMMIT
        ↓
Publish BranchCreatedEvent
        ↓
Membership: owner gains bypass access automatically;
            other staff need explicit staff_branch_access grants
```

The headquarters branch cannot be deleted or have `is_headquarters` reassigned through this flow — HQ transfer (if ever needed) is a deliberate, separate admin action, not part of V1.

---

# 30. Authorization

Do not use:

```text
Owner JWT
Manager JWT
Staff JWT
```

as the authorization model.

Instead:

```text
Authentication
     ↓
Who is the user?
     ↓
Business context
     ↓
Does user have membership?
     ↓
RBAC permission check
     ↓
Execute operation
```

Example permissions:

```text
business.view
business.update
business.profile.update
business.settings.update
business.operation.manage
business.addon.manage
business.branch.view
business.branch.manage
```

This aligns Business with the dynamic RBAC design.

Note: `business.branch.manage` authorizes *creating/editing/closing* a branch (a Business-module concern). It is distinct from `staff_branch_access` (a Membership-module concern), which authorizes *which staff can see/operate a given branch's data*. Creating a branch does not, by itself, grant anyone access to it beyond the owner bypass.

---

# 31. `IBusinessService`

Other modules should not directly query Business tables.

```csharp
public interface IBusinessService
{
    Task<BusinessDto> GetBusiness(Guid businessId);

    Task<bool> IsBusinessActive(Guid businessId);

    Task<bool> IsOperationEnabled(
        Guid businessId,
        OperationType operationType);

    Task<bool> IsAddonEnabled(
        Guid businessId,
        OperationType operationType,
        string addonType);

    Task<bool> IsBusinessOpen(
        Guid businessId,
        DateTime at);

    Task<IReadOnlyList<TaxRateDto>>
        GetOperationTaxRates(
            Guid businessId,
            OperationType operationType);

    Task<IReadOnlyList<BranchDto>> GetBranches(Guid businessId);

    Task<BranchDto> GetBranch(Guid branchId);

    Task<bool> IsBranchActive(Guid branchId);

    Task<Guid> GetHeadquartersBranch(Guid businessId);
}
```

Other modules consume contracts/application services.

They do not query:

```text
businesses
business_operations
business_tax_rates
branches
```

directly. In particular, Membership's `staff_branch_access` stores only `branch_id` references and calls `IBusinessService.IsBranchActive` / `GetBranches` rather than joining into Business tables.

---

# 32. Entity Relationships

```text
users
  │
  └── businesses
        ├── branches                           (1:many — exactly 1 is_headquarters)
        ├── business_profiles                  (1:1)
        ├── business_addresses                 (1:many — 1 default + 1 per branch)
        │      └── branches (optional FK)
        ├── business_images                    (1:many)
        ├── business_slug_history              (1:many)
        ├── business_hours                     (1:7 default + 1:7 per branch)
        │      ├── business_hour_intervals     (1:many)
        │      └── branches (optional FK)
        ├── business_hour_exceptions           (1:many)
        ├── business_settings                  (1:1)
        ├── business_tax_profiles              (1:1)
        ├── business_tax_rates                 (1:many)
        └── business_operations                (1:many)
               ├── business_operation_addons  (1:many)
               └── business_operation_tax_rates
                         │
                         └── business_tax_rates
```

Branch-level staff scoping (`staff_branch_access`) lives in the Membership module, not here — see Membership module → Branch Access Control.

---

# 33. Audit Requirements

Important changes should be auditable.

At minimum:

```text
Business created
Business updated
Business suspended
Business closed
Business reactivated

Operation enabled
Operation disabled

Branch created
Branch updated
Branch closed

Add-on enabled
Add-on disabled

Tax rate created
Tax rate changed
Tax rate deactivated

Operation tax mapping changed

Business settings changed

Business slug changed

Business hours changed
```

A centralized OneNex audit module is preferable if one already exists.

---

# 34. Branch / Location Direction — DECIDED (V1)

> Superseded: this used to defer the decision to Phase 2. It is now decided and built in V1 — see §3.3 `branches`.

**Option B — Location under one tenant** was chosen:

```text
Business
 ├── Jaffna Branch (HQ)
 ├── Colombo Branch
 └── Kandy Branch
```

`parent_business_id` has been removed from `businesses`. Branches are modeled as their own entity (`branches`, §3.3), owned by exactly one business, never as a second row in `businesses`.

This model was chosen (over Option A — separate tenant per location) because it gives multi-location operators, in one login:

- Shared ownership and single staff identity across locations
- Consolidated reporting at the business level
- Location-specific staff scope (Membership module's `staff_branch_access`)
- Location-specific hours (`business_hours.branch_id`)
- Location-specific addresses (`business_addresses.branch_id`)

A single business-scoped login (`onenex.ai/{business-slug}`) never exposes another business's data — but it does span all of that business's branches, filtered by which branches the logged-in staff member has access to (§ Membership module: Branch Access Control). There is no "switch business" control inside a business portal; switching businesses means returning to the Owner Portal / login and re-selecting (see the Identity module's Business Context & Portal Access flow).

---

# 35. Final V1 Table List

| Table | Purpose | Status |
|---|---|---|
| `businesses` | Tenant/business identity | Build |
| `branches` | Business locations (HQ + additional branches) | Build |
| `business_slug_history` | Old slug history/redirect | Build |
| `business_profiles` | Profile/contact | Build |
| `business_addresses` | Business default + per-branch address | Build |
| `business_images` | Logo/cover/gallery | Build |
| `business_operations` | Enabled operations — single source of truth | Build |
| `business_operation_addons` | Add-ons per operation | Build |
| `business_hours` | Weekly schedule | Build |
| `business_hour_intervals` | Multiple intervals per day | Build |
| `business_hour_exceptions` | Holiday/seasonal overrides | Build |
| `business_settings` | Global display settings | Build |
| `business_tax_profiles` | Tax registration/profile | Build |
| `business_tax_rates` | Effective-dated tax/charge definitions | Build |
| `business_operation_tax_rates` | Operation → tax mapping | Build |
| `Outbox` | Reliable cross-module events | Platform |
| `Audit` | Configuration/security history | Platform |

---

# 36. Final API Shape

Authorization is permission-based.

| Method | Endpoint | Permission |
|---|---|---|
| POST | `/api/businesses` | `business.create` |
| GET | `/api/businesses` | `business.view` |
| GET | `/api/businesses/{id}` | `business.view` + membership |
| PUT | `/api/businesses/{id}` | `business.update` |
| GET | `/api/businesses/resolve/{slug}` | Public tenant resolution — returns `businessId` + branch list for the Owner Portal's business/branch picker |
| GET/PUT | `/api/businesses/{id}/profile` | `business.profile.view/update` |
| GET/PUT | `/api/businesses/{id}/address` | `business.address.view/update` — `?branchId=` optional |
| GET/POST/PUT/DELETE | `/api/businesses/{id}/images` | `business.images.manage` |
| GET/POST/DELETE | `/api/businesses/{id}/operations` | `business.operation.view/manage` |
| POST/DELETE | `/api/businesses/{id}/operations/{type}/addons` | `business.addon.manage` |
| GET | `/api/businesses/{id}/branches` | `business.branch.view` + membership |
| POST | `/api/businesses/{id}/branches` | `business.branch.manage` |
| GET/PUT | `/api/businesses/{id}/branches/{branchId}` | `business.branch.view/manage` |
| DELETE | `/api/businesses/{id}/branches/{branchId}` | `business.branch.manage` — soft-close only, HQ blocked |
| GET/PUT | `/api/businesses/{id}/hours` | `business.hours.view/manage` — `?branchId=` optional |
| GET/POST/PUT/DELETE | `/api/businesses/{id}/hours/exceptions` | `business.hours.manage` |
| GET/PUT | `/api/businesses/{id}/settings` | `business.settings.view/update` |
| GET/POST/PUT/DELETE | `/api/businesses/{id}/tax-rates` | `business.tax.view/manage` |
| GET/PUT | `/api/businesses/{id}/operations/{type}/tax-rates` | `business.tax.view/manage` |

---

# 37. Project Structure

```text
Modules/Business/
├── Domain/
│   ├── Entities/
│   │   ├── Business.cs
│   │   ├── Branch.cs
│   │   ├── BusinessProfile.cs
│   │   ├── BusinessAddress.cs
│   │   ├── BusinessImage.cs
│   │   ├── BusinessSlugHistory.cs
│   │   ├── BusinessOperation.cs
│   │   ├── BusinessOperationAddon.cs
│   │   ├── BusinessHours.cs
│   │   ├── BusinessHourInterval.cs
│   │   ├── BusinessHourException.cs
│   │   ├── BusinessSettings.cs
│   │   ├── BusinessTaxProfile.cs
│   │   ├── BusinessTaxRate.cs
│   │   └── BusinessOperationTaxRate.cs
│   ├── Events/
│   │   ├── BusinessCreatedEvent.cs
│   │   ├── BusinessOperationEnabledEvent.cs
│   │   ├── BusinessOperationDisabledEvent.cs
│   │   ├── BusinessSuspendedEvent.cs
│   │   ├── BranchCreatedEvent.cs
│   │   └── BranchDeactivatedEvent.cs
│   └── ValueObjects/
│       ├── BusinessSlug.cs
│       ├── OperationType.cs
│       └── BusinessStatus.cs
│
├── Application/
│   └── Features/
│       ├── Businesses/
│       ├── Branches/
│       ├── Profile/
│       ├── Address/
│       ├── Images/
│       ├── Operations/
│       ├── Hours/
│       ├── Settings/
│       └── TaxRates/
│
├── Infrastructure/
│   ├── Repositories/
│   ├── EntityConfigurations/
│   └── BusinessDbContext.cs
│
└── API/
    └── Controllers/
```

---

# 38. Final Mental Model

```text
BUSINESS
= Who is the tenant?

BUSINESS_OPERATION
= What OneNex operation has this tenant enabled?

SUBSCRIPTION
= What is this tenant commercially entitled to/pay for?

OPERATION MODULE
= How does Dining/Stays/Bar/etc. actually work?

MEMBERSHIP + RBAC
= Which person is allowed to do what?

TAX CONFIGURATION
= Which charges apply, at what rate, and from when?

TRANSACTION SNAPSHOT
= What exact tax/configuration was used when the transaction happened?

BRANCH
= Where does this business operate — which location is this portal session scoped to?
```

This separation allows the business identity to remain stable while operations, subscriptions, permissions, tax rules and domain capabilities evolve independently.

---

# 39. Final Architecture Summary

The resulting OneNex Business architecture is:

```text
                         ┌─────────────────────┐
                         │      BUSINESS       │
                         │                     │
                         │ Tenant Identity     │
                         │ Lifecycle           │
                         │ Locale              │
                         │ Profile             │
                         │ Address             │
                         │ Settings            │
                         └──────────┬──────────┘
                                    │
                   ┌────────────────┼────────────────┬────────────────┐
                   │                │                │                │
                   ▼                ▼                ▼                ▼
          business_operations   Tax Config       Business Hours     Branches
                   │                                                   │
                                                            ┌──────────┴──────────┐
                                                            ▼                     ▼
                                                     Branch Address        Branch Hours
                   │
          ┌────────┼─────────┐
          ▼        ▼         ▼
       Dining    Stays      Bar
          │        │
          ▼        ▼
     Operation-specific
     domain/configuration


Subscription & Billing
        │
        └── Commercial entitlement
             (separate from business_operations)

Membership + RBAC
        │
        └── Who may access/use each capability

Outbox
        │
        └── Reliable cross-module event delivery
```


