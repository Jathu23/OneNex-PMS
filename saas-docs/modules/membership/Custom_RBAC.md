# ONENEX

#  MEMBERSHIP IMPLEMENTATION DOCUMENT

*Production-oriented design for Multi-Business + Business Role + Operation Role + Dynamic Permissions + Overrides + Caching*

---

| **Document** | **Value** |
|---|---|
| **Status** | Final implementation baseline |
| **Architecture** | Modular Monolith + Clean Architecture + CQRS + DDD |
| **Primary stack** | ASP.NET Core + EF Core + PostgreSQL + Redis + MediatR |
| **Audience** | Backend, frontend, database, DevOps and QA teams |
| **Scope** | Membership, roles, permissions, overrides, authorization, cache, audit, tenant isolation |

---

# 1. Executive Summary and Final Decision

This document combines the two supplied OneNex RBAC designs into one implementation baseline.

The final model uses the second design's explicit business-membership and operation-role structure, while incorporating the first design's:

- Effective-permission resolver
- Authorization versioning
- Audit trail
- Two-level caching
- Cache invalidation
- Future scope support
- Command-level defense in depth

## Final Authorization Model

```text
User
  ↓
Staff Membership (Business)
  ↓
Business Role (owner / admin / member)
  ↓
Operation Access (Dining / Stays / Maintenance / ...)
  ↓
Operation Role (full / manager / supervisor / staff / viewer / custom)
  ↓
Role Permissions
  ↓
Staff Permission Overrides
  ↓
Effective Permission Set
  ↓
L1 Memory Cache → L2 Redis → PostgreSQL
  ↓
Permission Authorization
  ↓
Domain / Resource Rules
```

This preserves the key OneNex separation:

```text
Authentication
    → Who is the user?

Membership
    → Which business does the user belong to?

Authorization
    → What can the user do?

Domain Rules
    → Is this specific business operation valid?
```

---

## 1.1 Architectural Decisions

| **Decision** | **Final Choice** | **Reason** |
|---|---|---|
| **Business relationship** | `staff_memberships` | Tenant membership must be explicit and security-critical. |
| **Business role** | Fixed enum: `owner / admin / member` | Business administration is distinct from operations. |
| **Operation role** | Role per enabled operation | Dining, Stays, Maintenance, etc. need independent access. |
| **Permissions** | Stable `module:resource:action` keys | Readable, globally unique and code-friendly. |
| **Custom roles** | Business-scoped | Businesses can define their own role combinations. |
| **Overrides** | Per-staff `ALLOW / DENY` | Handles edge cases without creating unnecessary roles. |
| **JWT** | Identity/context only; no permission list | Permissions change independently and must not become stale JWT data. |
| **Authorization** | ASP.NET Core policy + shared authorization service | Native framework pipeline plus reusable application checks. |
| **Cache** | L1 `IMemoryCache` + L2 Redis | Fast local reads and shared distributed cache. |
| **Freshness** | Version + invalidation + TTL | TTL alone can leave stale authorization. |
| **Audit** | Authorization audit table | Security-sensitive access changes need traceability. |
| **Future scope** | Optional scope columns now | Allows branch/property/resource restrictions later without redesigning overrides. |

---

## 1.2 What Is NOT Part of This RBAC System

- User authentication and credential storage → **Identity/Auth module**
- Business registration and business master data → **Business module**
- Actual order, booking, maintenance or payment business rules → **respective operation modules**
- Frontend permission checking as a security boundary → **frontend checks are UX only**
- Database RLS as the sole authorization mechanism → **application authorization remains authoritative; RLS can be added as defense in depth**

---

# 2. Core Concepts

## 2.1 Five Questions OneNex Must Answer

1. Who is this person?
2. Which business are they operating in?
3. Which branch (location) of that business are they operating in?
4. What permissions do they effectively have in that business/branch?
5. Is this particular operation/resource action allowed?

---

## 2.2 Business Role vs Operation Role

| **Layer** | **Examples** | **Controls** |
|---|---|---|
| **Business role** | `owner`, `admin`, `member` | Staff management, business settings, role management, subscription administration |
| **Operation role** | Dining: `manager`; Stays: `viewer` | Actions inside a specific operation |
| **Branch access** | Jaffna ✓, Colombo ✓, Kandy ✗ | Which physical location's data the staff member can see or act on |
| **Permission** | `dining:orders:void` | One concrete capability |
| **Override** | `ALLOW / DENY` one permission | One employee's exception to normal role permissions |
| **Domain rule** | Order is OPEN and belongs to Business A | Whether the requested business action is valid after authorization |

Branch access is a **separate, independent dimension** from business role and operation role — exactly like operation access, but scoped to *where* rather than *what*. `business_role = owner` bypasses branch checks entirely (owner sees every branch). `admin` and `member` see only the branches explicitly granted via `staff_branch_access`, regardless of their operation roles.

### Example

```text
Abi

Business:
    Grand Hotel

Business Role:
    member

Branch Access:
    Jaffna Branch (HQ)  → granted
    Colombo Branch      → granted
    Kandy Branch        → not granted

Operation Access:
    Dining       → manager
    Stays        → viewer
    Maintenance  → staff
    Bar          → no access

Override:
    dining:orders:refund → ALLOW
```

Abi can manage Dining orders — but only for the Jaffna and Colombo branches. A `dining:orders:void` request for a Kandy order is rejected at the branch-access check, before the operation-role/permission check even runs.

Effective permissions are the union of the applicable operation-role permissions, then modified by explicit `DENY / ALLOW` overrides.

---

# 3. Database Design

The final design uses the following authorization/membership tables.

Existing `Users`, `Businesses`, `Branches` and operation tables are assumed to belong to their respective modules.

| **Table** | **Purpose** | **V1** |
|---|---|---|
| `staff_memberships` | User ↔ Business relationship and business role/status | Required |
| `staff_invitations` | Secure invitation lifecycle | Required |
| `permissions` | Master permission catalog | Required |
| `roles` | System and business custom roles | Required |
| `role_permissions` | Role ↔ permission mapping | Required |
| `staff_operation_access` | Membership ↔ operation role | Required |
| `staff_branch_access` | Membership ↔ branch grant (which locations a staff member can operate in) | Required |
| `staff_custom_permissions` | Per-staff ALLOW/DENY overrides | Required |
| `authorization_states` | Authorization version per membership | Required |
| `authorization_audits` | Security audit history | Required |
| `authorization_outbox` | Durable change events for future scale | Recommended |

---

# 4. Detailed Table Definitions

## 4.1 `staff_memberships`

This is the **tenant/business boundary**.

Never authorize a business operation before confirming an active membership.

| **Field** | **Type** | **Null** | **Key / Rule** | **Description** |
|---|---|---|---|---|
| `id` | uuid | NO | PK | Membership identifier |
| `user_id` | uuid | NO | FK `users.id` | Authenticated person |
| `business_id` | uuid | NO | FK `businesses.id` | Business/tenant |
| `business_role` | varchar(20) | NO | CHECK `owner/admin/member` | Business administration role |
| `status` | varchar(20) | NO | CHECK `active/inactive/suspended` | Membership state |
| `pin_hash` | varchar(255) | YES | — | Hashed staff/POS PIN if applicable |
| `joined_at` | timestamptz | NO | — | Membership activation time |
| `created_at` | timestamptz | NO | — | Created timestamp |
| `updated_at` | timestamptz | NO | — | Last change |
| `row_version` | bigint | NO | — | Optimistic concurrency/version field |

### Constraints

```sql
UNIQUE (user_id, business_id)

INDEX (business_id, status)

INDEX (user_id, status)
```

---

## 4.2 `staff_invitations`

| **Field** | **Type** | **Null** | **Key / Rule** | **Description** |
|---|---|---|---|---|
| `id` | uuid | NO | PK | Invitation ID |
| `business_id` | uuid | NO | FK `businesses.id` | Target business |
| `invited_by_user_id` | uuid | NO | FK `users.id` | Who invited |
| `email` | varchar(320) | NO | — | Invite destination |
| `default_role_id` | uuid | YES | FK `roles.id` | Initial operation role, if applicable |
| `token_hash` | varchar(255) | NO | UNIQUE | Never store raw invitation token |
| `expires_at` | timestamptz | NO | — | Recommended 72 hours |
| `status` | varchar(20) | NO | — | `pending / accepted / expired / cancelled` |
| `accepted_at` | timestamptz | YES | — | Acceptance timestamp |
| `created_at` | timestamptz | NO | — | Created timestamp |

Invitation tokens are:

- Single-use
- Short-lived
- Stored hashed

---

## 4.3 `permissions`

| **Field** | **Type** | **Null** | **Key / Rule** | **Description** |
|---|---|---|---|---|
| `id` | uuid | NO | PK | Permission ID |
| `code` | varchar(150) | NO | UNIQUE | Stable authorization key |
| `module` | varchar(50) | NO | — | Owning module/operation |
| `resource` | varchar(80) | NO | — | Protected resource |
| `action` | varchar(80) | NO | — | Capability/action |
| `name` | varchar(150) | NO | — | Human-readable name |
| `description` | varchar(500) | YES | — | Developer/admin description |
| `is_active` | boolean | NO | DEFAULT `true` | Soft activation flag |
| `created_at` | timestamptz | NO | — | Created timestamp |

```sql
UNIQUE (code)
```

### Example Permission Codes

```text
dining:orders:void
dining:orders:handle
stays:bookings:checkin
maintenance:tasks:complete
```

Permission codes are application-owned constants and seeded.

Business users assign permissions to roles but should not arbitrarily invent authorization semantics.

---

## 4.4 `roles`

| **Field** | **Type** | **Null** | **Key / Rule** | **Description** |
|---|---|---|---|---|
| `id` | uuid | NO | PK | Role ID |
| `business_id` | uuid | YES | FK `businesses.id` | NULL for global system roles; non-null for custom roles |
| `code` | varchar(80) | NO | — | Stable role code |
| `name` | varchar(120) | NO | — | Display name |
| `role_type` | varchar(20) | NO | `system/custom` | Role ownership |
| `operation_type` | varchar(50) | YES | — | Dining/Stays/etc. for operation roles |
| `is_system` | boolean | NO | DEFAULT `false` | Immutable seeded role |
| `is_active` | boolean | NO | DEFAULT `true` | Soft deletion/deactivation |
| `created_at` | timestamptz | NO | — | Created timestamp |
| `updated_at` | timestamptz | NO | — | Last update |

### System Roles

```text
business_id = NULL
```

System roles are global and immutable.

### Custom Roles

```text
business_id = owning business ID
```

Custom roles belong only to that business.

### Recommended Uniqueness

```text
UNIQUE(business_id, code)
```

for custom roles, while system roles have `business_id = NULL` and globally unique codes.

---

## 4.5 `role_permissions`

| **Field** | **Type** | **Null** | **Key / Rule** | **Description** |
|---|---|---|---|---|
| `role_id` | uuid | NO | PK/FK `roles.id` | Role |
| `permission_id` | uuid | NO | PK/FK `permissions.id` | Permission |
| `created_at` | timestamptz | NO | — | Assignment timestamp |

### Constraints

```sql
PRIMARY KEY (role_id, permission_id)

INDEX (permission_id, role_id)
```

---

## 4.6 `staff_operation_access`

| **Field** | **Type** | **Null** | **Key / Rule** | **Description** |
|---|---|---|---|---|
| `id` | uuid | NO | PK | Access row ID |
| `staff_membership_id` | uuid | NO | FK `staff_memberships.id` | Membership |
| `operation_type` | varchar(50) | NO | — | Dining, Stays, Maintenance, etc. |
| `role_id` | uuid | YES | FK `roles.id` | Operation role |
| `access_status` | varchar(20) | NO | `enabled/disabled` | Explicit operation access state |
| `created_at` | timestamptz | NO | — | Created timestamp |
| `updated_at` | timestamptz | NO | — | Last update |

### Constraint

```sql
UNIQUE (staff_membership_id, operation_type)
```

### Important Rule

```text
disabled
    ↓
No operation permission
    ↓
Regardless of role_id
```

Keeping `access_status` separate from `role_id` is preferable to representing no access using a magic role.

---

## 4.6a `staff_branch_access`

Grants a staff member access to a specific branch (location) of the business. Owner bypasses this check entirely (implicit access to every branch). Admin and member must have an explicit row per branch — access is **not** derived from `business_role` or from any operation role.

| **Field** | **Type** | **Null** | **Key / Rule** | **Description** |
|---|---|---|---|---|
| `id` | uuid | NO | PK | Access row ID |
| `staff_membership_id` | uuid | NO | FK `staff_memberships.id` | Membership |
| `branch_id` | uuid | NO | FK `branches.id` (Business module) | Granted branch |
| `granted_by_user_id` | uuid | NO | FK `users.id` | Who granted this branch |
| `granted_at` | timestamptz | NO | — | Grant timestamp |

### Constraints

```sql
UNIQUE (staff_membership_id, branch_id)

INDEX (staff_membership_id)
```

### Default Grant on Invite

Same pattern as `staff_operation_access`: when a staff member is invited, the system auto-creates one `staff_branch_access` row per branch **currently active on the business at invite time**. The owner can then customize — revoke Kandy, keep Jaffna and Colombo — exactly as shown in the mockups.

```text
Owner invites Kamal as "Manager":
  Jaffna Branch (HQ)  → auto-granted
  Colombo Branch      → auto-granted
  Kandy Branch        → auto-granted
  [all branches active at invite time]

Owner customizes: revokes Kamal's Kandy Branch access.
```

### New Branch Created After Staff Onboarded

A new branch is **not** retroactively granted to existing non-owner staff (`BranchCreatedEvent` triggers no automatic `staff_branch_access` inserts for admin/member — only the owner's bypass covers it automatically). An explicit grant is required. This mirrors the open question already on file for new operations (§ "New operation enabled after staff already onboarded").

### Important Rule

```text
No staff_branch_access row for (membership, branch)
    AND
business_role != owner
    ↓
403 on any request scoped to that branch
    ↓
Regardless of operation role or permission overrides
```

---

## 4.7 `staff_custom_permissions`

| **Field** | **Type** | **Null** | **Key / Rule** | **Description** |
|---|---|---|---|---|
| `id` | uuid | NO | PK | Override ID |
| `staff_membership_id` | uuid | NO | FK `staff_memberships.id` | Employee in a business |
| `permission_id` | uuid | NO | FK `permissions.id` | Permission being overridden |
| `effect` | varchar(10) | NO | `ALLOW/DENY` | Override result |
| `scope_type` | varchar(30) | YES | — | Future: business/branch/property/resource |
| `scope_id` | uuid | YES | — | Future scoped resource ID |
| `reason` | varchar(500) | YES | — | Why override exists |
| `expires_at` | timestamptz | YES | — | Optional temporary override |
| `created_by_user_id` | uuid | NO | FK `users.id` | Who changed it |
| `created_at` | timestamptz | NO | — | Created timestamp |
| `updated_at` | timestamptz | NO | — | Last update |

### Constraints

```sql
UNIQUE (
    staff_membership_id,
    permission_id,
    scope_type,
    scope_id
)

CHECK (effect IN ('ALLOW', 'DENY'))
```

### Purpose

Overrides are scoped to **one staff member**.

Other staff members with the same role are unaffected.

---

## 4.8 `authorization_states`

| **Field** | **Type** | **Null** | **Key / Rule** | **Description** |
|---|---|---|---|---|
| `staff_membership_id` | uuid | NO | PK/FK | Authorization subject |
| `version` | bigint | NO | DEFAULT `1` | Increment on every authz change |
| `updated_at` | timestamptz | NO | — | Last authorization change |

The cache is associated with this version.

Versioning is a safety mechanism in addition to cache invalidation and TTL.

---

## 4.9 `authorization_audits`

| **Field** | **Type** | **Null** | **Description** |
|---|---|---|---|
| `id` | uuid | NO | Audit ID |
| `business_id` | uuid | NO | Business context |
| `staff_membership_id` | uuid | YES | Affected employee membership |
| `actor_user_id` | uuid | NO | User who performed the change |
| `event_type` | varchar(50) | NO | `ASSIGN_ROLE`, `REMOVE_ROLE`, `GRANT_PERMISSION`, `DENY_PERMISSION`, etc. |
| `entity_type` | varchar(50) | NO | Role, permission, membership, override |
| `entity_id` | uuid | YES | Affected record |
| `before_json` | jsonb | YES | Previous state snapshot |
| `after_json` | jsonb | YES | New state snapshot |
| `reason` | varchar(500) | YES | Optional explanation |
| `ip_address` | inet | YES | Request source |
| `user_agent` | varchar(1000) | YES | Client metadata |
| `created_at` | timestamptz | NO | Audit timestamp |

---

## 4.10 `authorization_outbox`

**Recommended**

| **Field** | **Type** | **Null** | **Description** |
|---|---|---|---|
| `id` | uuid | NO | PK |
| `aggregate_type` | varchar(80) | NO | Authorization aggregate |
| `aggregate_id` | uuid | NO | Membership ID |
| `event_type` | varchar(100) | NO | `AuthorizationChanged` |
| `payload` | jsonb | NO | Event data |
| `occurred_at` | timestamptz | NO | Creation timestamp |
| `published_at` | timestamptz | YES | Null until delivered |
| `attempt_count` | int | NO | Retry count |
| `last_error` | text | YES | Last delivery error |

V1 may use Redis Pub/Sub directly if operational simplicity is more important.

The outbox is the safer path when reliable event delivery becomes necessary.

---

# 5. Sample Data

## 5.1 Businesses, Branches and Users

| **ID** | **Name / Email** | **Type** |
|---|---|---|
| B001 | Grand Hotel | Business |
| B002 | City Apartments | Business |
| BR001 | Jaffna Branch (HQ) | Branch of B001 |
| BR002 | Colombo Branch | Branch of B001 |
| BR003 | Kandy Branch | Branch of B001 |
| U001 | Abi | User |
| U002 | Kamal | User |
| U003 | John | User |

---

## 5.2 Memberships

| **ID** | **User** | **Business** | **Business Role** | **Status** |
|---|---|---|---|---|
| M001 | Abi | Grand Hotel | member | active |
| M002 | Kamal | Grand Hotel | admin | active |
| M003 | John | Grand Hotel | member | active |
| M004 | Abi | City Apartments | admin | active |

The same user can have different business roles because membership is scoped by business.

---

## 5.2a Branch Access

| **Membership** | **Branch** | **Access** |
|---|---|---|
| M001 (Abi, member) | Jaffna Branch (HQ) | granted |
| M001 (Abi, member) | Colombo Branch | granted |
| M001 (Abi, member) | Kandy Branch | not granted |
| M002 (Kamal, admin) | Jaffna Branch (HQ) | granted |
| M002 (Kamal, admin) | Colombo Branch | granted |
| M002 (Kamal, admin) | Kandy Branch | granted |
| M003 (John, member) | Jaffna Branch (HQ) | granted |

Owner (not shown — no `staff_branch_access` rows needed) always has access to all three branches via bypass.

---

## 5.3 Operation Access

| **Membership** | **Operation** | **Role** | **Access** |
|---|---|---|---|
| M001 | Dining | manager | enabled |
| M001 | Stays | viewer | enabled |
| M001 | Maintenance | staff | enabled |
| M001 | Bar | — | disabled |
| M003 | Dining | staff | enabled |
| M003 | Stays | viewer | enabled |

---

## 5.4 Role Permissions

| **Role** | **Permission** |
|---|---|
| Dining manager | `dining:orders:view` |
| Dining manager | `dining:orders:create` |
| Dining manager | `dining:orders:handle` |
| Dining manager | `dining:orders:void` |
| Dining manager | `dining:orders:approve` |
| Dining viewer | `dining:orders:view` |
| Dining staff | `dining:orders:view` |
| Dining staff | `dining:orders:create` |

---

## 5.5 Overrides

| **Membership** | **Permission** | **Effect** | **Reason** | **Expires** |
|---|---|---|---|---|
| M001 | `dining:orders:refund` | ALLOW | Senior staff temporary responsibility | 2026-12-31 |
| M001 | `dining:orders:void` | DENY | Training period | NULL |

---

## 5.6 Effective Result for Abi

```text
ALLOW

dining:orders:view
dining:orders:create
dining:orders:handle
dining:orders:approve
dining:orders:refund

DENY

dining:orders:void

STAYS

stays:* view-level permissions

MAINTENANCE

maintenance:* staff-level permissions

BAR

no access
```

---

# 6. Effective Permission Resolution

The resolver is the heart of the system.

It must accept a membership/business context and produce the current effective authorization set.

```text
GetEffectivePermissionsAsync(staffMembershipId)

1. Load membership
2. Confirm business + active status
3. If owner → global business-owner bypass (if policy permits)
4. Load enabled operation access
5. Load permissions for each operation role
6. Load business-level admin permissions where applicable
7. Load staff custom overrides
8. Ignore expired overrides
9. Apply DENY
10. Apply ALLOW
11. Apply scope rules
12. Produce EffectivePermissionSet
13. Attach authorization version
14. Cache result
```

`GetEffectivePermissionsAsync` resolves *what* the member can do, business-wide — it does not know *which branch* is being requested. Branch access is checked separately, on the specific request, because it depends on the resource being touched, not on the membership alone. See `HasBranchAccessAsync` below and §7 for where this fits in the request flow.

```text
HasBranchAccessAsync(staffMembershipId, branchId)

1. Load membership
2. If owner → TRUE (bypass)
3. staff_branch_access row exists for (staffMembershipId, branchId)?
       → YES → TRUE
       → NO  → FALSE
```

## Final Precedence

```text
Business owner bypass (if enabled)
        ↓
Membership must be active
        ↓
Branch access must be granted (owner bypasses; skipped if request is not branch-scoped)
        ↓
Operation access must be enabled
        ↓
Role permissions provide base ALLOW
        ↓
Explicit DENY removes permission
        ↓
Explicit ALLOW adds permission
        ↓
Resource/domain rules can still reject the action
```

For conflicting overrides at the same scope, the database must prevent duplicates.

Do not rely on arbitrary row order.

---

# 7. Complete Authorization Request Flow

```text
HTTP Request
    ↓
Authentication middleware
    ↓
JWT valid?
    ├── NO → 401
    └── YES
          ↓
Resolve current Business Context
          ↓
Load Staff Membership
    ├── missing → 403
    └── suspended/inactive → 401/403 according to API contract
          ↓
Request scoped to a branch?
    ├── NO  → skip branch check
    └── YES → HasBranchAccessAsync(membership, branchId)
                  ├── owner → bypass
                  ├── granted → continue
                  └── not granted → 403
          ↓
ASP.NET Core Authorization Policy
          ↓
PermissionAuthorizationHandler
          ↓
L1 Memory Cache
    ├── HIT → use permission set
    └── MISS
          ↓
        Redis
    ├── HIT → hydrate L1
    └── MISS → PostgreSQL → resolve → Redis → L1
          ↓
Required permission present?
    ├── NO → 403
    └── YES
          ↓
MediatR command/handler
          ↓
Domain/resource checks
          ↓
Execute transaction
```

ASP.NET Core supports policy-based authorization through requirements and handlers, and policies can be applied declaratively to endpoints. 

---

# 8. ASP.NET Core Implementation

## 8.1 Permission Constants

```csharp
public static class DiningPermissions
{
    public const string ViewOrders = "dining:orders:view";
    public const string CreateOrder = "dining:orders:create";
    public const string HandleOrder = "dining:orders:handle";
    public const string VoidOrder = "dining:orders:void";
    public const string RefundOrder = "dining:orders:refund";
}
```

---

## 8.2 Declarative Endpoint Protection

```csharp
[RequirePermission(DiningPermissions.VoidOrder)]
[HttpPost("{id}/void")]
public async Task<IActionResult> Void(Guid id)
{
    await mediator.Send(new VoidOrderCommand(id));

    return NoContent();
}
```

The custom attribute should translate to a dynamic policy such as:

```text
permission:dining:orders:void
```

ASP.NET Core's policy system evaluates requirements through authorization handlers. 

---

## 8.3 Authorization Handler

```csharp
protected override async Task HandleRequirementAsync(
    AuthorizationHandlerContext context,
    PermissionRequirement requirement)
{
    var userId = context.User.GetUserId();
    var businessId = context.User.GetBusinessId();

    if (userId is null || businessId is null)
        return;

    var membership =
        await membershipService.GetActiveMembershipAsync(
            userId,
            businessId);

    if (membership is null)
        return;

    if (membership.BusinessRole == BusinessRole.Owner)
    {
        context.Succeed(requirement);
        return;
    }

    var branchId = context.User.GetBranchId(); // from route/resource, may be null

    if (branchId is not null)
    {
        var hasBranchAccess =
            await membershipService.HasBranchAccessAsync(
                membership.Id,
                branchId.Value);

        if (!hasBranchAccess)
            return; // 403 — branch not granted to this staff member
    }

    var allowed =
        await authorizationService.HasPermissionAsync(
            membership.Id,
            requirement.Permission);

    if (allowed)
        context.Succeed(requirement);
}
```

---

## 8.4 Authorization Service Contract

```csharp
public interface IAuthorizationService
{
    Task<bool> HasPermissionAsync(
        Guid staffMembershipId,
        string permission,
        CancellationToken cancellationToken = default);

    Task<EffectivePermissionSet> GetEffectivePermissionsAsync(
        Guid staffMembershipId,
        CancellationToken cancellationToken = default);
}
```

`HasBranchAccessAsync` / `GetAccessibleBranchesAsync` (called by the handler as `membershipService.HasBranchAccessAsync(...)` above) live on the membership service contract, not here — branch grants are membership data, not permission data:

```csharp
public interface IMembershipService
{
    Task<StaffMembershipDto> GetActiveMembershipAsync(
        Guid userId, Guid businessId);

    Task<bool> HasBranchAccessAsync(
        Guid staffMembershipId,
        Guid branchId);

    Task<IReadOnlyList<Guid>> GetAccessibleBranchesAsync(
        Guid staffMembershipId);
}
```

`GetAccessibleBranchesAsync` powers the branch picker/filter shown in the portal ("Jaffna Branch (HQ)" / "Colombo Branch" with green "Access Granted" badges) — the frontend never computes this itself.

---

## 8.5 Effective Permission Set

```csharp
public sealed class EffectivePermissionSet
{
    public required Guid StaffMembershipId { get; init; }

    public required Guid BusinessId { get; init; }

    public required long Version { get; init; }

    public required IReadOnlySet<string> Allowed { get; init; }

    public required IReadOnlySet<string> Denied { get; init; }
}
```

---

# 9. Caching and Invalidation

## 9.1 Cache Keys

```text
L1:
authz:{businessId}:{userId}

L2:
authz:{businessId}:{userId}
```

Optional:

```text
authz-version:{businessId}:{userId}
```

Branch access (separate cache, invalidated independently since it changes far less often than permissions):

```text
branch-access:{businessId}:{userId}   → ["branch_id_1", "branch_id_2", ...]
```

Always include business context.

Never use:

```text
authz:{userId}
```

alone when the same user can belong to multiple businesses.

---

## 9.2 Cache Value

```json
{
  "membershipId": "M001",
  "businessId": "B001",
  "version": 19,
  "allowed": [
    "dining:orders:view",
    "dining:orders:create",
    "dining:orders:handle",
    "dining:orders:refund"
  ],
  "denied": [
    "dining:orders:void"
  ],
  "expiresAt": "2026-09-02T12:00:00Z"
}
```

---

## 9.3 Invalidation

```text
Admin action
    ↓
PostgreSQL transaction
    ├── change role/permission/override
    ├── increment authorization_states.version
    └── insert audit record
    ↓
COMMIT
    ↓
publish AuthorizationChanged
    ↓
Redis deletes L2 cache
    ↓
all API nodes receive invalidation
    ↓
each node deletes L1 cache
    ↓
next request rebuilds authorization
```

Redis Pub/Sub is not durable.

For larger scale, publish a transactional outbox event and deliver it through a durable broker.

Versioning remains a safety mechanism.

---

# 10. Important Write Transactions

## 10.1 Assign Role to Employee

```sql
BEGIN;

UPDATE staff_operation_access
SET role_id = :new_role,
    updated_at = now()
WHERE staff_membership_id = :membership
  AND operation_type = :operation;

UPDATE authorization_states
SET version = version + 1,
    updated_at = now()
WHERE staff_membership_id = :membership;

INSERT INTO authorization_audits (...);

INSERT INTO authorization_outbox (...);

COMMIT;
```

---

## 10.2 Grant Individual Permission

```sql
BEGIN;

INSERT INTO staff_custom_permissions (
    staff_membership_id,
    permission_id,
    effect,
    reason,
    created_by_user_id,
    created_at,
    updated_at
)
VALUES (
    :membership,
    :permission,
    'ALLOW',
    :reason,
    :actor,
    now(),
    now()
)
ON CONFLICT (...)
DO UPDATE SET
    effect = EXCLUDED.effect,
    reason = EXCLUDED.reason,
    updated_at = now();

UPDATE authorization_states
SET version = version + 1,
    updated_at = now()
WHERE staff_membership_id = :membership;

INSERT INTO authorization_audits (...);

INSERT INTO authorization_outbox (...);

COMMIT;
```

---

# 11. EF Core Modeling Rules

EF Core maps relational relationships through foreign keys.

The database should remain the final source of referential integrity, not only EF navigation configuration. 

```csharp
modelBuilder.Entity<StaffMembership>()
    .HasIndex(x => new
    {
        x.UserId,
        x.BusinessId
    })
    .IsUnique();

modelBuilder.Entity<StaffOperationAccess>()
    .HasIndex(x => new
    {
        x.StaffMembershipId,
        x.OperationType
    })
    .IsUnique();

modelBuilder.Entity<StaffBranchAccess>()
    .HasIndex(x => new
    {
        x.StaffMembershipId,
        x.BranchId
    })
    .IsUnique();

modelBuilder.Entity<RolePermission>()
    .HasKey(x => new
    {
        x.RoleId,
        x.PermissionId
    });

modelBuilder.Entity<Permission>()
    .HasIndex(x => x.Code)
    .IsUnique();

modelBuilder.Entity<StaffCustomPermission>()
    .HasIndex(x => new
    {
        x.StaffMembershipId,
        x.PermissionId,
        x.ScopeType,
        x.ScopeId
    })
    .IsUnique();
```

---

# 12. PostgreSQL DDL Baseline

## `staff_memberships`

```sql
CREATE TABLE staff_memberships (
    id uuid PRIMARY KEY,

    user_id uuid NOT NULL
        REFERENCES users(id),

    business_id uuid NOT NULL
        REFERENCES businesses(id),

    business_role varchar(20) NOT NULL
        CHECK (business_role IN ('owner', 'admin', 'member')),

    status varchar(20) NOT NULL
        CHECK (status IN ('active', 'inactive', 'suspended')),

    pin_hash varchar(255),

    joined_at timestamptz NOT NULL,
    created_at timestamptz NOT NULL,
    updated_at timestamptz NOT NULL,

    row_version bigint NOT NULL DEFAULT 1,

    CONSTRAINT uq_staff_membership
        UNIQUE(user_id, business_id)
);

CREATE INDEX ix_staff_memberships_business_status
    ON staff_memberships(business_id, status);
```

## `staff_branch_access`

```sql
CREATE TABLE staff_branch_access (
    id uuid PRIMARY KEY,

    staff_membership_id uuid NOT NULL
        REFERENCES staff_memberships(id)
        ON DELETE CASCADE,

    branch_id uuid NOT NULL
        REFERENCES branches(id),

    granted_by_user_id uuid NOT NULL
        REFERENCES users(id),

    granted_at timestamptz NOT NULL,

    CONSTRAINT uq_staff_branch_access
        UNIQUE(staff_membership_id, branch_id)
);

CREATE INDEX ix_staff_branch_access_membership
    ON staff_branch_access(staff_membership_id);
```

`branches` is owned and defined by the Business module (see its `branches` table). Membership only stores the FK and never queries Business tables directly.

## `permissions`

```sql
CREATE TABLE permissions (
    id uuid PRIMARY KEY,

    code varchar(150) NOT NULL UNIQUE,

    module varchar(50) NOT NULL,
    resource varchar(80) NOT NULL,
    action varchar(80) NOT NULL,

    name varchar(150) NOT NULL,
    description varchar(500),

    is_active boolean NOT NULL DEFAULT true,

    created_at timestamptz NOT NULL
);
```

## `role_permissions`

```sql
CREATE TABLE role_permissions (
    role_id uuid NOT NULL
        REFERENCES roles(id)
        ON DELETE CASCADE,

    permission_id uuid NOT NULL
        REFERENCES permissions(id),

    created_at timestamptz NOT NULL,

    PRIMARY KEY(role_id, permission_id)
);
```

---

# 13. Multi-Business / Tenant Isolation

The application must never rely on:

- A subdomain
- A route value
- A frontend-selected business ID
- A frontend-selected branch ID

as the security decision.

The server resolves the business context and verifies the authenticated user's membership. The same applies to branch context — a URL like `onenex.ai/rio-jaffna` *identifies* which business is being requested, but the URL alone never grants access; membership (and, if the resource is branch-scoped, `staff_branch_access`) must be verified server-side on every request.

## Example Request

```text
grandhotel.onenex.com/api/dining/orders/123
```

### Flow

```text
1. Resolve grandhotel → Business B001

2. Authenticate user U001

3. Find staff_memberships(U001, B001)

4. Verify status = active

5. Order O123 belongs to Branch B_JAFFNA →
   Verify staff_branch_access(M001, B_JAFFNA) OR owner bypass

6. Authorize requested permission

7. Query order with BusinessId = B001 AND BranchId = B_JAFFNA

8. Execute domain rules
```

For PostgreSQL, Row-Level Security can provide a second database-level tenant isolation layer. PostgreSQL policies can restrict rows returned or modified and use `USING / WITH CHECK` expressions. 

### Recommendation

Introduce RLS only after the application tenant context is stable and tested.

Do not make RLS configuration the only protection against cross-tenant access.

---

# 14. Frontend Permission Discovery

```http
GET /api/me/capabilities
```

### Response

```json
{
  "businessId": "B001",
  "version": 19,
  "businessRole": "member",
  "operations": {
    "dining": "manager",
    "stays": "viewer",
    "maintenance": "staff"
  },
  "permissions": [
    "dining:orders:view",
    "dining:orders:create",
    "dining:orders:handle",
    "dining:orders:refund"
  ],
  "branches": [
    { "branchId": "BR_JAFFNA", "name": "Jaffna Branch", "isHeadquarters": true },
    { "branchId": "BR_COLOMBO", "name": "Colombo Branch", "isHeadquarters": false }
  ]
}
```

`branches` lists only the branches this membership can access (owner bypass returns every active branch on the business). The frontend uses this both to hide/show UI and to render the branch switcher/filter inside the business portal.

The frontend uses this to hide/show UI.

Every protected backend operation still performs authorization — including the branch check, which is re-verified per request, not trusted from this cached response.

---

# 15. Owner, Admin and Member Decisions

| **Case** | **Decision** |
|---|---|
| **Owner** | Business owner can bypass normal operation permission checks only where the product explicitly defines owner supremacy. Still require active membership and tenant/resource checks. |
| **Admin** | Admin does NOT automatically receive all operation permissions. Business administration permissions are separate. |
| **Member** | Operation roles determine access. |
| **Admin changing dining config** | Allowed only if business policy explicitly grants business admins configuration access; otherwise require dining config permission. |
| **Branch access (owner)** | Bypasses `staff_branch_access` entirely — every current and future branch. |
| **Branch access (admin/member)** | Must have an explicit `staff_branch_access` row for the requested branch. Not derived from `business_role` or any operation role. |
| **Suspended member** | No normal business access; reject before permission evaluation. |
| **Removed member** | Membership deleted/deactivated and cache invalidated immediately (including branch-access cache). |

---

# 16. Seeded Operation Roles

| **Role** | **Typical Permissions** |
|---|---|
| `full` | All permissions in operation |
| `manager` | View, create, edit, void, approve, reports, configs, subject to operation definition |
| `supervisor` | View, create, approve; no void/config by default |
| `staff` | View and create only |
| `viewer` | Read-only permissions; reports may be included according to product definition |

Do not assume that role names are universal business semantics.

The seeded role-permission mappings are the actual authority.

---

# 17. Initial Permission Catalog

| **Module** | **Permission Examples** |
|---|---|
| **Dining** | `dining:orders:view`, `dining:orders:create`, `dining:orders:handle`, `dining:orders:void`, `dining:orders:approve`, `dining:orders:refund`, `dining:menu:edit`, `dining:reports:view`, `dining:configs:manage` |
| **Stays** | `stays:bookings:view`, `stays:bookings:create`, `stays:bookings:checkin`, `stays:bookings:checkout`, `stays:folio:view`, `stays:folio:void`, `stays:reports:view`, `stays:configs:manage` |
| **Maintenance** | `maintenance:tasks:view`, `maintenance:tasks:create`, `maintenance:tasks:assign`, `maintenance:tasks:complete`, `maintenance:reports:view` |
| **CRM** | `crm:customers:view`, `crm:customers:create`, `crm:customers:update`, `crm:customers:delete`, `crm:loyalty:manage` |
| **Payments** | `payments:transactions:view`, `payments:refund:create`, `payments:reconciliation:manage` |
| **Business** | `business:staff:view`, `business:staff:invite`, `business:staff:remove`, `business:roles:manage`, `business:settings:manage`, `business:branches:view`, `business:branches:manage` |

---

# 18. CQRS / MediatR Integration

```text
HTTP
  ↓
Controller
  ↓
MediatR
  ↓
AuthorizationBehavior
  ↓
ValidationBehavior
  ↓
Handler
  ↓
Domain
  ↓
Repository / DbContext
```

Use endpoint/policy authorization for the HTTP boundary and a MediatR authorization behavior for commands that may also be invoked from non-HTTP entry points.

This is defense in depth, not duplicate business logic.

```csharp
public sealed record HandleOrderCommand(Guid OrderId)
{
    public const string RequiredPermission =
        DiningPermissions.HandleOrder;
}
```

---

# 19. Security Rules

## Never

- Never trust a frontend-supplied business ID without resolving and validating membership.
- Never trust a frontend-supplied branch ID without resolving and validating `staff_branch_access` (or owner bypass).
- Never use the frontend permission list as an authorization source.
- Never put the complete dynamic permission set in the JWT.
- Never query another module's RBAC tables directly.
- Never use role-name string comparisons inside operation modules.
- Never store raw invitation tokens or PINs.
- Never assume operation-role access implies branch access, or vice versa — they are independent checks.

## Always

- Always audit role and permission changes.
- Always invalidate authorization cache after an authorization change.
- Always include business context in authorization cache keys.
- Always verify branch access before executing a query scoped to a specific branch's data.
- Use parameterized queries / EF Core rather than dynamic SQL built from permission strings.
- Do not let an inactive/suspended membership continue through the normal authorization path.
- Keep domain/resource validation after permission authorization.
- Remember that permission alone does not make an operation valid.

---

# 20. Module Boundary

```text
Modules/
├── Membership/
│   ├── Domain/
│   ├── Application/
│   ├── Infrastructure/
│   └── API/
│
├── Dining/
├── Stays/
├── Maintenance/
├── CRM/
├── Payments/
└── Notifications/
```

Operation modules should know permission constants such as:

```text
dining:orders:void
```

But they should **not** know:

```text
staff_memberships
role_permissions
Redis authorization keys
authorization database internals
```

---

# 21. Implementation Order

Implement the system in the following order:

1. Create `permissions` table and seed the initial permission catalog from module constants.
2. Create `roles` and `role_permissions`; seed the five operation roles and business administration permissions.
3. Create `staff_memberships` and enforce `UNIQUE(user_id, business_id)`.
4. Create `staff_operation_access` and implement operation enable/disable.
5. Create `staff_branch_access` and implement branch grant/revoke, including auto-grant-all-current-branches on invite.
6. Create `staff_custom_permissions` with ALLOW/DENY, expiration and reserved scope fields.
7. Create `staff_invitations` and implement single-use hashed invitation tokens.
8. Create `authorization_states` and increment its version on every authorization mutation.
9. Create `authorization_audits` and write an audit record in the same transaction as the authorization change.
10. Implement `EffectivePermissionResolver`.
11. Implement `HasBranchAccessAsync` / `GetAccessibleBranchesAsync`.
12. Implement L2 Redis permission + branch-access cache.
13. Implement L1 `IMemoryCache`.
14. Implement cache invalidation after committed authorization/branch-access changes.
15. Implement ASP.NET Core dynamic permission policies and authorization handler, including the branch-access check.
16. Add MediatR `AuthorizationBehavior`.
17. Add `GET /api/me/capabilities` for frontend UX (including accessible branches).
18. Add tenant/business context middleware and active-membership validation.
19. Add integration tests for cross-business and cross-branch access.
20. Add observability:
    - Authorization cache hit/miss
    - Denied requests
    - Invalidation events
    - Resolver latency
    - Branch-access denials
21. Only after the application flow is stable, evaluate PostgreSQL RLS for defense in depth.

---

# 22. Mandatory Test Cases

| **Test** | **Expected** |
|---|---|
| User has no membership in Business A | 403 |
| User has membership in Business A but active membership in B only | 403 for A |
| Suspended member calls protected API | Rejected before normal permission evaluation |
| Member has Dining manager | Manager permissions allowed |
| Same member has Stays viewer | Only viewer permissions in Stays |
| Business admin without Dining role tries void order | 403 unless explicit business policy grants it |
| ALLOW override adds missing permission | Permission becomes allowed |
| DENY override removes role permission | Permission becomes denied |
| Expired ALLOW override | Ignored |
| Role permission changes | Authorization version increments and cache invalidates |
| Two businesses, same user, different permissions | Cache and authorization remain isolated |
| Cross-business order ID supplied | Domain/tenant query rejects access |
| Owner has no staff_branch_access rows | Still allowed on every branch (bypass) |
| Admin/member has no staff_branch_access row for requested branch | 403, even with full operation-role permissions |
| Admin/member has staff_branch_access for Branch A only, requests Branch B resource | 403 for B, allowed for A |
| New branch created after staff onboarded | Existing non-owner staff have no access until explicitly granted |
| Staff invited while 3 branches active | 3 staff_branch_access rows auto-created |
| Owner revokes one staff_branch_access row | That branch immediately inaccessible to that staff member; others unaffected |
| Redis unavailable | Defined fail-safe strategy; no unauthorized fallback |
| Invalid JWT | 401 |
| Authenticated but unauthorized | 403 |

---

# 23. Final Implementation Decisions / Open Items Closed

| **Topic** | **Final Decision** |
|---|---|
| **Permission naming** | `module:resource:action` |
| **System role ownership** | `business_id NULL` |
| **Custom role ownership** | `business_id` set to owning business |
| **Business role** | Fixed enum `owner/admin/member` |
| **Operation role** | Per operation through `staff_operation_access` |
| **No operation access** | `access_status = disabled`; do not create a fake `none` role |
| **Branch model** | Branch = location under one tenant (Business module `branches`), not a separate business |
| **Branch access** | Explicit per-branch grant via `staff_branch_access`; owner bypasses; independent of business role and operation role |
| **Branch access default** | Auto-granted for every branch active at invite time; new branches require explicit grant |
| **Override effect** | Constrained string CHECK `ALLOW/DENY` |
| **Override expiry** | Supported now |
| **Scope** | Columns reserved now; scope enforcement introduced when product requires it |
| **Cache key** | `authz:{businessId}:{userId}` |
| **Cache strategy** | L1 memory + L2 Redis |
| **Staleness control** | Invalidation + version + TTL |
| **Audit** | Required for security-sensitive authorization mutations |
| **Durable events** | Outbox recommended as system grows |
| **JWT permissions** | Not included |
| **Frontend authorization** | UX only; backend remains authoritative |
| **HTTP authorization** | ASP.NET Core policy/handler |
| **Command authorization** | MediatR behavior as defense in depth |
| **Tenant isolation** | Membership validation + business-scoped queries; RLS optional defense in depth |

---

# 24. End-to-End Example: Handle Order

Abi opens:

```text
grandhotel.onenex.com
```

### JWT

This is the **business-scoped session token** (JWT_2), minted by Identity after Abi selected Grand Hotel in the Owner Portal — see the Identity module's "Business Context & Portal Access" flow. The original login token (JWT_1) never carries a business_id.

```text
sub = U001
business_id = B001
```

### Request

```http
POST /api/dining/orders/O123/handle
```

Order O123 belongs to Branch `BR_JAFFNA` (resolved by the Dining module from the order record, not from the request).

### Authorization Flow

```text
1. JWT validation
   → PASS

2. Resolve B001
   → Grand Hotel

3. Membership U001/B001
   → M001
   → active

4. Load Order O123 → belongs to Branch BR_JAFFNA

5. Branch access check
   → HasBranchAccessAsync(M001, BR_JAFFNA)
   → owner? no → check staff_branch_access
   → granted → PASS

6. Required permission
   → dining:orders:handle

7. L1 cache
   → MISS

8. Redis
   → HIT

9. Effective permissions contain
   dining:orders:handle
   → PASS

10. MediatR HandleOrderCommand
    → PASS

11. Load Order O123
    WHERE business_id = B001

12. Domain rule
    → order exists
    → belongs to B001
    → state is handleable

13. Transaction
    → COMMIT

14. Response
    → 200 / 204
```

If Abi instead lacked `staff_branch_access` for `BR_JAFFNA` (e.g., only granted Colombo), the request stops at step 5 and returns `403 Forbidden` — before the permission check even runs.

If John has only Dining staff permissions and lacks:

```text
dining:orders:handle
```

the request stops at authorization and returns:

```text
403 Forbidden
```

The operation handler should not contain special-case code for John.

---

# 25. Frontend UX Rules

- Fetch capabilities after login/business-context establishment.
- Cache the capability response client-side for UI rendering.
- Use permissions to show/hide buttons, menus and screens.
- Do not assume hidden UI means security.
- Handle a backend `403` gracefully because permissions may have changed since the capabilities call.
- Refresh capabilities after a business switch or authorization version change.
- There is no in-portal business switcher by design — switching businesses means returning to the Owner Portal / login screen and re-selecting (see Identity module). A branch filter/switcher *within* the current business portal is fine, but every branch-scoped request is still re-checked server-side regardless of what the UI shows as "selected."

---

# 26. Observability

| **Metric / Log** | **Purpose** |
|---|---|
| `authz.check.count` | Authorization volume |
| `authz.denied.count` | Security and UX monitoring |
| `authz.cache.l1.hit` | Local cache effectiveness |
| `authz.cache.redis.hit` | Distributed cache effectiveness |
| `authz.resolver.duration` | Database/resolution performance |
| `authz.invalidation.count` | Cache invalidation health |
| `authz.version.mismatch` | Stale-cache detection |
| Authorization audit events | Forensic/security review |

---

# 27. Research and Design Basis

The supplied OneNex documents are the primary design basis.

External verification was used only to validate the ASP.NET Core and PostgreSQL implementation choices.

Microsoft documents describe authorization as distinct from authentication and support policy-based requirements/handlers; this supports the selected dynamic permission policy architecture. 

PostgreSQL documentation confirms that Row-Level Security can restrict rows for `SELECT / INSERT / UPDATE / DELETE` using policies and `USING / WITH CHECK` expressions, supporting its optional use as tenant-isolation defense in depth. 

EF Core documentation confirms relational relationships are represented through foreign keys; the final design therefore treats the database FK constraints and unique indexes as part of the authorization data model rather than relying solely on application navigation properties. 

---

# 28. Production Readiness Checklist

- [ ] All FK constraints created and tested.
- [ ] All uniqueness constraints created.
- [ ] Permission catalog seeded and immutable to ordinary business users.
- [ ] System roles seeded and immutable.
- [ ] Custom role lifecycle implemented.
- [ ] Membership lifecycle implemented.
- [ ] Operation access lifecycle implemented.
- [ ] Overrides support ALLOW/DENY/expiry.
- [ ] Authorization version increments transactionally.
- [ ] Audit record is written in the same transaction.
- [ ] Redis invalidation happens only after successful database commit.
- [ ] L1 cache invalidation is distributed to all API instances.
- [ ] JWT contains identity/context but not the dynamic permission list.
- [ ] All protected endpoints have a permission policy.
- [ ] MediatR behavior protects command entry points where appropriate.
- [ ] Cross-business integration tests pass.
- [ ] Cross-branch integration tests pass (non-owner staff cannot see/act on an ungranted branch).
- [ ] Suspended/removed users are rejected immediately.
- [ ] Frontend capability endpoint is implemented.
- [ ] Metrics and security logs are available.
- [ ] Backup/restore includes authorization tables.
- [ ] RLS decision documented and tested before enabling in production.

---

# 29. Final Architecture Summary

```text
┌──────────────────────┐
│       Identity       │
│    User / JWT / Auth │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│   Staff Membership   │
│       Business       │
└──────────┬───────────┘
           │
     ┌─────┬─────┐
     ▼     ▼     ▼
Business  Branch  Operation Access
Role      Access  Dining/Stays/etc.
owner/    (owner
admin/    bypass;
member    explicit
          grant)
     │     │     │
     │     │     ▼
     │     │   Role
     │     │     │
     │     │     ▼
     │     │  Role Permissions
     │     │     │
     │     │     ▼
     │     │  Staff Overrides
     │     │     │
     └─────┴──┬──┘
              ▼
   Effective Permission Set
   (+ branch access verdict)
              │
        ┌─────┴─────┐
        ▼           ▼
    L1 Memory     Redis
        │           │
        └─────┬─────┘
              ▼
    Authorization Policy
   (permission AND branch)
              │
              ▼
      MediatR / Handler
              │
              ▼
      Domain / RLS Rules
              │
              ▼
       Database Action
```

---

# FINAL DESIGN PRINCIPLE

```text
Identity
    ≠
Membership
    ≠
Role
    ≠
Permission
    ≠
Domain Rule
```

OneNex should keep these concerns separate.

```text
Identity
→ Who are you?

Membership
→ Which business can you operate?

Business Role
→ What business-level administration can you perform?

Branch Access
→ Which physical location's data can you see or act on?

Operation Role
→ What can you do inside this operation?

Permission
→ What specific capability do you have?

Override
→ Is there an employee-specific exception?

Authorization
→ Are you allowed to perform this action?

Domain Rule
→ Is this specific action valid right now?

Tenant Isolation
→ Does the resource actually belong to the business you are operating?
```

This separation is the foundation of the OneNex authorization architecture.