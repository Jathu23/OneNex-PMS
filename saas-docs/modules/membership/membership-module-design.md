# Membership Module — Design Document

> Draft — Core module. Reviewed and finalized during architecture planning.
> This module answers: "Who can access which business, with what role, and what are they allowed to do?"
>
> **Note:** [`Custom_RBAC.md`](./Custom_RBAC.md) in this same folder is the authoritative, final implementation baseline for this module — it merges this document with a second RBAC design and adds caching, versioning, audit and the branch-access dimension described below. Where the two disagree, `Custom_RBAC.md` wins. This document has been updated to stay consistent with it on branch access.

---

## Design Philosophy

**Simple by Default. Powerful when needed.**

```
Layer 1 (Default — 80% of businesses):
  Staff invite → pick one role → all operations auto-set → done

Layer 2 (Operation-specific — 15%):
  Expand "Customize" → set different role per operation

Layer 3 (Power users — 5%):
  Custom roles + per-staff permission overrides
```

UI shows complexity only when owner asks for it.

---

## Module Responsibility

- Staff invitation flow
- Business-level role (owner / admin / member)
- Operation-level role per operation
- Permission resolution (role permissions + custom overrides)
- Staff status management (active / inactive / suspended)
- PIN management for POS / terminal login

**Does NOT own:**
- User accounts, JWT → Identity module
- Business registration → Business module
- Actual operation work → Operation modules
- Sending emails → Notification service

---

## Three Independent Access Dimensions

### Dimension 1: `business_role`

Governs business management only. Fixed enum — 3 values. System-defined. Owners cannot create new types.

```
owner  → full access everywhere (hardcoded bypass — no permission check needed)
admin  → manage staff, roles, settings
         does NOT grant any operation access
member → no business management. operation roles govern everything.
```

`admin` permissions (seeded at startup, immutable):
```
business:staff:view
business:staff:invite
business:staff:remove
business:roles:manage
business:settings:manage
```

### Dimension 2: Operation Role

Governs what staff can do inside each operation. Assigned per operation independently.

```
full        → all permissions in that operation
manager     → manage + create + edit + void + approve + reports + configs
supervisor  → create + approve (no void, no reports, no configs)
staff       → create + view only
viewer      → view only
```

### Dimension 3: `branch_access`

Governs **which physical location's** data staff can see or act on — completely independent of `business_role` and Operation Role. Matches the "Branch Access Control" flow from the product mockups: a Manager can be granted Jaffna and Colombo but denied Kandy, regardless of their Dining role being `manager` everywhere.

```
owner  → implicit access to every branch (bypass, no staff_branch_access rows needed)
admin/member → access only to branches with an explicit staff_branch_access row
```

Default on invite: one `staff_branch_access` row per branch **currently active** on the business (same auto-populate pattern as operation access). Owner can then customize — revoke specific branches per staff member.

**These three dimensions are fully independent.**
`business_role = admin` does NOT grant operation access, and does NOT grant branch access.
Operation reports, void, configs — all come from operation role only.
Which branch's data is visible — comes from branch access only.

---

## Permission Check Logic

```
Request arrives

Step 1: business_role = owner?
        → YES → allow immediately (bypass all checks)

Step 2: Suspension check (Redis)
        → suspended? → 401 immediately

Step 3: Branch access check (only if the request targets a branch-scoped resource)
        → staff_branch_access row exists for this branch?
        → NO → 403 (no access to this branch)

Step 4: Operation role check
        → staff_operation_access row exists for this operation?
        → NO → 403 (no access to this operation)

Step 5: Permission check (Redis cache)
        → "dining:orders:void" in user's permission list?
        → YES → proceed
        → NO  → 403
```

---

## Entities

### `permissions` — master registry

All permission codes. Defined in code as constants. Seeded at startup. Never modified by users.

Format: `{module}:{resource}:{action}`

```
dining:orders:view
dining:orders:create
dining:orders:void
dining:orders:approve
dining:menu:edit
dining:tables:manage
dining:reports:view
dining:configs:manage

stays:bookings:view
stays:bookings:create
stays:checkin:process
stays:checkout:process
stays:folio:view
stays:folio:void
stays:folio:approve
stays:reports:view
stays:configs:manage

business:staff:view
business:staff:invite
business:staff:remove
business:roles:manage
business:settings:manage
```

Each module owns its permission constants. Membership module only stores them.

---

### `roles` — role definitions

```sql
id, business_id (null=system / not null=custom), name, is_system, created_at
UNIQUE(business_id, name)
```

**System roles** (`business_id = null`) — seeded once at application startup. Shared across all businesses. Immutable.

**Custom roles** (`business_id = X`) — created by business owner. Scoped to that business only.

When a new business is created: no new role rows. System roles are shared globally.

---

### `role_permissions` — permissions per role

```sql
id, role_id → roles.id, permission → permissions.code
UNIQUE(role_id, permission)
```

System role default sets (seeded, immutable):

```
full:        all permissions
manager:     view, create, void, approve, edit, manage, reports, configs
supervisor:  view, create, approve
staff:       view, create
viewer:      view, reports (read-only)
```

Custom role permissions → owner picks from `permissions` table. Change affects all staff with that role.

---

### `staff_memberships` — user ↔ business link

```sql
id, user_id → users.id, business_id → businesses.id,
business_role VARCHAR(20),   -- owner | admin | member
pin_hash,
status VARCHAR(20),          -- active | inactive | suspended
invited_at, joined_at
UNIQUE(user_id, business_id)
```

One row per user per business. `business_role` is a fixed enum — not from `roles` table.

---

### `staff_operation_access` — per-operation role

```sql
id, staff_membership_id → staff_memberships.id,
operation_type VARCHAR(20),  -- dining | stays | bar | wellness | events | retail
role_id → roles.id           -- system role or custom role
UNIQUE(staff_membership_id, operation_type)
```

Default invite flow: one role selected → system auto-creates one row per enabled operation.

```
Owner invites Kamal as "Manager":
  Dining  → manager role
  Stays   → manager role
  Bar     → manager role
  [auto-populated for all enabled operations]
```

Owner customizes: Kamal's Stays → viewer, Bar → no access.

---

### `staff_branch_access` — per-branch grant

```sql
id, staff_membership_id → staff_memberships.id,
branch_id → branches.id (Business module),
granted_by_user_id → users.id,
granted_at
UNIQUE(staff_membership_id, branch_id)
```

Default invite flow: one row per branch **active at invite time** — same pattern as operation access auto-populate.

```
Owner invites Kamal as "Manager":
  Jaffna Branch (HQ) → auto-granted
  Colombo Branch     → auto-granted
  Kandy Branch       → auto-granted
  [auto-populated for all currently active branches]

Owner customizes: revokes Kamal's Kandy Branch access.
```

`business_role = owner` bypasses this table entirely — no rows are needed for an owner to see every branch. A new branch created later is **not** retroactively granted to existing non-owner staff; it requires an explicit grant (mirrors the "new operation" open question below).

---

### `staff_custom_permissions` — per-staff overrides

```sql
id, staff_membership_id → staff_memberships.id,
permission → permissions.code,
is_granted BOOLEAN,   -- true=grant, false=revoke
reason VARCHAR(255)
UNIQUE(staff_membership_id, permission)
```

Scoped to ONE staff member. Other staff with same role — unaffected.

**Resolution order:**
```
1. Role permissions (base set)
2. Apply is_granted=false  → revoke specific permissions
3. Apply is_granted=true   → grant additional permissions
```

Use case: 99% covered by operation roles. Custom overrides = edge cases only.

---

### `staff_invitations` — invitation flow

```sql
id, business_id, invited_by → users.id,
email, default_role_id → roles.id,
token_hash, expires_at,
status VARCHAR(20),   -- pending | accepted | expired | cancelled
accepted_at, created_at
```

Flow:
```
Owner invites email → invitation row created → StaffInvitedEvent
                                                     ↓
                                          Notification: email sent

Staff clicks link:
  Has account?  → log in → accept → staff_memberships row created
  No account?   → register (Identity) → accept → staff_memberships row created

Token: single-use, 72hr expiry, hashed in DB
```

---

## Permission Enforcement — ASP.NET Core Policy-based

### Why Policy-based (not custom action filter)?

```
Custom action filter:
  ✓ Simple to write
  ✗ Outside framework pipeline
  ✗ 401 vs 403 manually handle பண்ணணும்
  ✗ Hard to combine with other auth requirements

ASP.NET Core Policy:
  ✓ Framework pipeline — middleware level check
  ✓ 401 vs 403 automatically correct
  ✓ Testable via IAuthorizationService
  ✓ Combines with other requirements naturally
```

### Approach

`[RequirePermission]` attribute → extends `AuthorizeAttribute` → internally uses framework policy system.

```csharp
// Developer writes this (clean, type-safe):
[RequirePermission(DiningPermissions.VoidOrder)]

// Framework sees this (powerful):
[Authorize(Policy = "permission:dining:orders:void")]
```

Constants used — no magic strings. Typo = compile error, not runtime error.

### Flow

```
[RequirePermission(DiningPermissions.VoidOrder)]
  → AuthorizeAttribute: Policy = "permission:dining:orders:void"
  → PermissionPolicyProvider: dynamically creates policy
  → PermissionAuthorizationHandler: does the actual check
      → owner? bypass
      → suspended? fail
      → request targets a branch? → Redis: branch-access:{userId}:{businessId}
          not in list? → fail (403)
      → Redis: permissions:{userId}:{businessId}
          HIT  → check list
          MISS → DB load → cache → check list
      → permission in list? succeed / fail
```

---

## Redis Cache Strategy

```
Cache key:    permissions:{userId}:{businessId}
Cache value:  ["dining:orders:view", "dining:orders:create", ...]
TTL:          5 minutes (auto-expiry safety net)

Cache key:    branch-access:{userId}:{businessId}
Cache value:  ["branch_id_1", "branch_id_2", ...]
TTL:          5 minutes (same safety net; invalidated on grant/revoke)

Suspension:   suspended:{userId}:{businessId} = "1"  TTL: 24h
```

**On permission change:**
```
PermissionsChangedEvent published
  → Redis DEL permissions:{userId}:{businessId}
  → next request: cache MISS → fresh DB load → re-cached
  → change takes effect on next API call
```

**On suspension:**
```
StaffSuspendedEvent published
  → Redis SET suspended:{userId}:{businessId}
  → next request: suspension check hits Redis → 401 immediately
```

**Performance:**
```
Redis hit  → ~0.3ms overhead per request (99% of requests)
Cache miss → ~5ms (DB query) — only on first request or after change
```

---

## Access Matrix

| Action | Requires |
|---|---|
| Invite / remove staff | business_role: admin, owner |
| Change business settings | business_role: admin, owner |
| Manage subscription | business_role: owner only |
| Manage custom roles | business_role: admin, owner |
| Create / close a branch | business_role: admin, owner (Business module `business.branch.manage`) |
| Grant / revoke a staff member's branch access | business_role: admin, owner |
| See or act on a branch's data | branch_access: explicit grant for that branch — OR business_role: owner (bypass) |
| Manage menu | dining operation role: manager, full |
| Take orders | dining operation role: staff and above |
| Void orders | dining operation role: manager, full |
| View dining reports | dining operation role: manager, full, viewer |
| Change dining configs | dining: full — OR business_role: admin, owner |
| Check-in / check-out | stays operation role: staff and above |
| View stays reports | stays operation role: manager, full, viewer |
| Change stays configs | stays: full — OR business_role: admin, owner |

---

## Entity Relationships

```
permissions (master list — code constants, seeded at startup)

roles
  └── role_permissions

users + businesses
  └── staff_memberships          UNIQUE(user_id, business_id)
        ├── business_role         owner | admin | member
        ├── staff_branch_access    (per branch grant → branches.id, Business module)
        ├── staff_operation_access  (per operation → role_id)
        └── staff_custom_permissions (per-staff overrides)

staff_invitations → accepted → staff_memberships created
```

---

## Domain Events

**Consumed:**
```
BusinessCreatedEvent    → create owner staff_membership (business_role = owner)
                          all current operations → full access
                          (owner needs no staff_branch_access rows — bypass)

BranchCreatedEvent      → no automatic staff_branch_access rows for non-owner staff;
                          explicit grant required (Business module event, §26)
```

**Published:**
```
StaffInvitedEvent       → Notification: send invitation email
StaffJoinedEvent        → audit
StaffSuspendedEvent     → Redis: set suspension flag immediately
StaffRemovedEvent       → Redis: delete permissions + branch-access cache
PermissionsChangedEvent → Redis: delete permissions cache
BranchAccessChangedEvent → Redis: delete branch-access cache
```

---

## Module Boundary (Shared.Contracts)

```csharp
public interface IMembershipService
{
    Task<bool>         HasPermission(Guid userId, Guid businessId, string permission);
    Task<bool>         IsActiveMember(Guid userId, Guid businessId);
    Task<string>       GetBusinessRole(Guid userId, Guid businessId);
    Task<List<string>> GetPermissions(Guid userId, Guid businessId);
    Task<bool>         HasBranchAccess(Guid userId, Guid businessId, Guid branchId);
    Task<List<Guid>>   GetAccessibleBranches(Guid userId, Guid businessId);
    Task<List<MembershipSummaryDto>> GetMyMemberships(Guid userId);
}
```

`GetMyMemberships` returns every business (with role + accessible branches) the user belongs to — it powers the Owner Portal's business/branch picker (see Identity module's Business Context & Portal Access flow). It is the one query in this interface that is not scoped to a single business, by design — it is how a user discovers which businesses they can even select.

No other module queries membership tables directly.

---

## V1 Scope

| Table | V1 |
|---|---|
| `permissions` | Build — seeded from module constants |
| `roles` | Build — 5 system operation roles seeded |
| `role_permissions` | Build — default sets seeded |
| `staff_memberships` | Build |
| `staff_branch_access` | Build |
| `staff_operation_access` | Build |
| `staff_custom_permissions` | Build |
| `staff_invitations` | Build |

---

## Project Structure

```
Modules/Membership/
├── Domain/
│   ├── Entities/
│   │   ├── Role.cs
│   │   ├── RolePermission.cs
│   │   ├── StaffMembership.cs
│   │   ├── StaffBranchAccess.cs
│   │   ├── StaffOperationAccess.cs
│   │   ├── StaffCustomPermission.cs
│   │   └── StaffInvitation.cs
│   └── Events/
│       ├── StaffInvitedEvent.cs
│       ├── StaffJoinedEvent.cs
│       ├── StaffSuspendedEvent.cs
│       ├── StaffRemovedEvent.cs
│       └── PermissionsChangedEvent.cs
│
├── Application/
│   └── Features/
│       ├── Staff/
│       │   ├── Commands/ InviteStaff, AcceptInvitation,
│       │   │             ChangeStaffRole, SuspendStaff, RemoveStaff
│       │   └── Queries/  GetStaffList, GetStaffMembership
│       ├── OperationAccess/
│       │   ├── Commands/ SetOperationAccess
│       │   └── Queries/  GetOperationAccess
│       ├── BranchAccess/
│       │   ├── Commands/ GrantBranchAccess, RevokeBranchAccess
│       │   └── Queries/  GetBranchAccess, GetMyMemberships
│       ├── Permissions/
│       │   ├── Commands/ SetCustomPermissions
│       │   └── Queries/  GetEffectivePermissions
│       ├── Roles/
│       │   ├── Commands/ CreateCustomRole, UpdateCustomRolePermissions,
│       │   │             DeleteCustomRole
│       │   └── Queries/  GetRoles
│       └── Pin/
│           └── Commands/ SetPin, RemovePin
│
├── Infrastructure/
│   ├── Repositories/
│   │   ├── StaffMembershipRepository.cs
│   │   └── RoleRepository.cs
│   ├── Authorization/
│   │   ├── RequirePermissionAttribute.cs
│   │   ├── PermissionPolicyProvider.cs
│   │   ├── PermissionRequirement.cs
│   │   └── PermissionAuthorizationHandler.cs
│   ├── Caching/
│   │   └── PermissionCacheService.cs
│   ├── Seeds/
│   │   ├── PermissionSeeder.cs
│   │   └── SystemRoleSeeder.cs
│   └── MembershipDbContext.cs
│
└── API/
    └── Controllers/
        ├── StaffController.cs
        ├── RolesController.cs
        ├── PermissionsController.cs
        └── InvitationsController.cs
```

---

## Open Questions

- New operation enabled after staff already onboarded → auto-assign default role or require explicit assignment?
- New branch created after staff already onboarded → confirmed: explicit assignment required, no auto-grant (see `staff_branch_access` above) — revisit only if product later wants an "auto-grant new branches" toggle per staff member.
- Custom role deletion → blocked if staff assigned. Reassign first or soft-delete?
- Invitation resend → new token or extend expiry?
- PIN: length, numeric only, lockout after N failed attempts?
- Branch-access cache TTL / invalidation — same Redis key pattern as permissions, or a longer TTL since branch grants change less often?
- Redis TTL: 5 min currently — adjust based on load testing?
