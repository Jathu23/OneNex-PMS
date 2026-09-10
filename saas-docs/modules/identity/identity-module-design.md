# Identity Module — Design Document

> Status: DRAFT — Needs team review before implementation.
> Built on: ASP.NET Core Identity

---

## Responsibility

**One line:** WHO you are. Globally. Nothing else.

Manages the global identity of every human who touches OneNex.
Same registration flow for owner, staff, and customer.
Knows nothing about businesses, roles, or permissions — those belong to other modules.

---

## Technology: ASP.NET Core Identity

### Why ASP.NET Core Identity

```
→ Battle-tested Microsoft library
→ Password hashing built-in (no custom crypto)
→ Token generation built-in (email verify, password reset)
→ Brute force protection built-in (lockout)
→ OAuth/external login ready (Phase 2)
→ Industry standard — every .NET developer knows it
```

### What We USE

```
✓ IdentityUser         → extended as ApplicationUser
✓ UserManager<T>       → register, find, update, delete, token operations
✓ Password hashing     → PBKDF2 (built-in, secure)
✓ Email confirm tokens → UserManager.GenerateEmailConfirmationTokenAsync()
✓ Password reset tokens→ UserManager.GeneratePasswordResetTokenAsync()
✓ Account lockout      → auto-lock after N failed login attempts
✓ Claims               → attach user data to JWT
✓ IPasswordHasher<T>   → used internally by UserManager
```

### What We SKIP

```
✗ IdentityRole + RoleManager → Membership module handles roles
✗ Cookie authentication      → JWT only for request authentication. One
                                narrow exception: an HttpOnly cookie
                                transports the refresh-token *value* on
                                web (see "Web vs Mobile Token Delivery")
                                — no ASP.NET Core cookie-auth middleware,
                                no session cookie the framework
                                authenticates requests against.
✗ SignInManager              → too cookie-focused, use UserManager directly
✗ IdentityDbContext default  → we use our own ApplicationDbContext
```

### What We CUSTOMIZE

```
~ ApplicationUser    → extend IdentityUser (add name, phone, status)
~ JWT service        → custom token generation (Identity doesn't do JWT)
~ Refresh tokens     → custom table (Identity doesn't manage these)
~ Phone uniqueness   → custom validator (Identity doesn't enforce global uniqueness)
```

---

## Entities

---

### Entity 1: ApplicationUser (extends IdentityUser)

```
ASP.NET Core Identity provides IdentityUser with:
  Id, Email, NormalizedEmail, EmailConfirmed
  PasswordHash, SecurityStamp
  PhoneNumber, PhoneNumberConfirmed
  TwoFactorEnabled, LockoutEnd, LockoutEnabled, AccessFailedCount

We EXTEND with:
  Name        VARCHAR(100)   NOT NULL
  Status      ENUM           (active / suspended / deleted)
  CreatedAt   TIMESTAMP
  UpdatedAt   TIMESTAMP
```

**Database table: AspNetUsers (Identity default)**

```sql
-- Identity creates this automatically. We add extra columns:
ALTER TABLE AspNetUsers ADD
  Name        NVARCHAR(100) NOT NULL DEFAULT '',
  Status      NVARCHAR(20)  NOT NULL DEFAULT 'active',
  CreatedAt   TIMESTAMP     NOT NULL DEFAULT NOW(),
  UpdatedAt   TIMESTAMP     NOT NULL DEFAULT NOW();
```

**What IdentityUser already has (important fields):**
```
Id                    → our user_id (GUID)
Email                 → unique (Identity enforces)
EmailConfirmed        → email verification status
PhoneNumber           → we add uniqueness validation
PasswordHash          → hashed by Identity (PBKDF2)
LockoutEnd            → when lockout expires(lockout feature is a security mechanism that temporarily prevents a user from logging in after too many failed login attempts.)
LockoutEnabled        → lockout feature on/off
AccessFailedCount     → failed login count
SecurityStamp         → changes when password/security info changes
                        (used to invalidate existing tokens)
```

---

### Entity 2: RefreshTokens (Custom — Identity doesn't provide)

Each row is one **authentication session** (see Session Management below) — `device_info` is descriptive metadata about that session, not an identity key.

```sql
refresh_tokens:
  id              UUID          PK   DEFAULT gen_random_uuid()
  user_id         UUID          NOT NULL → AspNetUsers.Id
  family_id       UUID          NOT NULL  (constant across a rotation chain —
                                            same value from login through every
                                            refresh; see Refresh Token Reuse
                                            Detection below)
  token_hash      VARCHAR(500)  NOT NULL  (hashed — never store plain)
  expires_at      TIMESTAMP     NOT NULL  (30 days from issue)
  revoked_at      TIMESTAMP     NULLABLE  (null = still valid)
  revoked_reason  VARCHAR(100)  NULLABLE  (logout / rotation / suspicious / reuse_detected)
  device_info     VARCHAR(200)  NULLABLE  (browser, OS — for display)
  ip_address      VARCHAR(45)   NULLABLE  (IPv6 max length)
  created_at      TIMESTAMP     NOT NULL  DEFAULT NOW()

INDEX: user_id, token_hash, family_id
```

**Why hashed:**
```
Refresh token stolen from DB → cannot be used directly.
Plain token only in client. Hash only in DB.
Same pattern as password storage.
```

**Refresh Token Reuse Detection:**
```
Incoming refresh token hashes to a row where revoked_reason = 'rotation'
  → this exact token was already rotated away once before — someone is
    replaying a token that should no longer exist (stolen after rotation,
    or a client retry bug racing a legitimate refresh)
  → treat as compromise, not a normal invalid-token error:
      UPDATE refresh_tokens SET revoked_at = NOW(), revoked_reason = 'reuse_detected'
      WHERE family_id = <family_id> AND revoked_at IS NULL
  → every token in the family dies — not just the reused one — forcing
    re-login on every device sharing that session
  → write IdentitySecurityAudits event RefreshTokenReuseDetected (see below)

A single stolen-and-later-used-once token only ever costs the attacker one
rotation before the legitimate client's next refresh call trips this check
(whichever side refreshes second finds its token already 'rotation'-revoked).
```

---

### Fast Session Invalidation (SecurityStamp + Redis)

The JWT is self-contained and normally validated by signature + expiry
alone — no DB or network call needed, which is what makes them scale
horizontally across any number of API nodes. That alone means a suspended
or password-reset account's *existing* tokens stay technically valid for
up to 15 more minutes (until natural expiry) — acceptable for many systems,
but not for "admin suspends an abusive account" or "user resets a
compromised password," both of which need to take effect on the *next
request*, not the next login.

The fix reuses a value Identity already has for free — `AspNetUsers.SecurityStamp`,
which ASP.NET Core Identity already regenerates automatically on password
change — plus the exact Redis pattern the Membership module already uses
for its own immediate-suspension check (`Custom_RBAC.md` §9, "Suspension"):

```
At token issuance (login / refresh / select-business):
  JWT carries claim  sst = AspNetUsers.SecurityStamp (current value)

On every authenticated request:
  Redis GET security-stamp:{userId}
    HIT  → compare to JWT's sst claim
    MISS → DB SELECT SecurityStamp → cache (TTL 5 min, same as Membership's
            permission cache) → compare
  mismatch → 401 immediately (token predates a security-relevant change)
  match    → proceed (no DB hit on the common path)

On password reset / suspend / delete:
  UserManager.UpdateSecurityStampAsync(user)   ← password reset does this
                                                   automatically already;
                                                   suspension/deletion must
                                                   call it explicitly
  Redis DEL security-stamp:{userId}   ← immediately, don't wait for TTL
                                         (same "invalidate now, TTL is only
                                         a safety net" discipline Membership
                                         uses for StaffSuspendedEvent)
```

This makes JWT validation *not* purely stateless anymore — it costs one
Redis GET per request. That is not a new class of dependency: Membership
already requires a Redis GET on every business-scoped request for the
permission/branch-access check, at ~0.3ms on a cache hit. This adds one
more key lookup of the same shape, not a second architecture. Signature
and expiry validation remain fully local (any service with the public key
still does that part with zero network calls) — only the "has this been
revoked since issuance" question needs Redis, and it fails safe (Redis
unavailable → treat as cache miss → fall back to Postgres, same posture
Membership already documents for its own cache).

---

### Entity 3: IdentitySecurityAudits (Custom — Identity-level security log)

Distinct from Membership's `authorization_audits`, which logs role/permission
changes *within* a business. This logs what happens to the *global account*
itself, with no business context at all — the same discipline Membership
already applies to authorization changes, applied here to identity events.

```sql
identity_security_audits:
  id           UUID          PK   DEFAULT gen_random_uuid()
  user_id      UUID          NOT NULL → AspNetUsers.Id
  event_type   VARCHAR(50)   NOT NULL
  ip_address   VARCHAR(45)   NULLABLE
  user_agent   VARCHAR(500)  NULLABLE
  metadata     JSONB         NULLABLE  (event-specific detail, e.g. family_id
                                         on a reuse-detection event)
  created_at   TIMESTAMP     NOT NULL  DEFAULT NOW()

INDEX: user_id, created_at
```

```
Event types:
  LoginSucceeded            LoginFailed
  PasswordChanged           PasswordResetRequested
  EmailVerified             AccountSuspended
  AccountDeleted            RefreshTokenRotated
  RefreshTokenReuseDetected Logout
  LogoutAll                 BusinessSelected   (JWT reissued with business_id)
```

Written in the same transaction as the state change it records — an audit
row that silently failed to commit is worse than no audit table at all,
because it looks like nothing happened.

---

## User Status Flow

```
[Register]
     ↓
[pending_verification]   ← email not yet confirmed
     ↓ POST /auth/verify-email
[active]                 ← normal working state
     ↓                        ↑
[suspended]              ← admin action (abuse, payment failed)
     ↓                        │
[deleted]                ← soft delete (GDPR compliance)
     (data anonymized, record kept for audit) Data anonymized means personal information has been removed or changed so you can’t identify the person it belongs to.
```

---

## Business Rules

### Registration
```
✓ Email must be globally unique (Identity enforces)
✓ Phone must be globally unique (custom validation)
✓ Password minimum: 8 chars, 1 uppercase, 1 number, 1 special char
✓ Email verification required before first login
✓ One account per email — period
  → Same email cannot register as owner AND as staff separately
  → Same account, different roles (via Membership module)
```

### Login
```
✓ Email must be verified before login allowed
✓ Account must be active (not suspended/deleted)
✓ Failed login: increment AccessFailedCount
✓ After 5 failed attempts → account locked for 15 minutes (Identity lockout)
✓ Successful login → reset AccessFailedCount
✓ Issue JWT { sub, sid, sst, business_id=null, ... } on success — see JWT Design
✓ Issue refresh token → store hash in refresh_tokens table, new family_id
```

### Email Verification
```
✓ Token generated by Identity (UserManager.GenerateEmailConfirmationTokenAsync)
✓ Token expires: 24 hours
✓ Token is single-use (Identity marks as used)
✓ Resend allowed: max 3 times per hour (rate limit)
✓ After verify → EmailConfirmed = true → login allowed
```

### Password Reset
```
✓ Token generated by Identity (UserManager.GeneratePasswordResetTokenAsync)
✓ Token expires: 1 hour
✓ Token is single-use
✓ Old password NOT required (forgot password flow)
✓ After reset → SecurityStamp changes → all existing tokens invalidated
  on their next request, not just at next login (see Fast Session
  Invalidation)
✓ Send email via INotificationService
```

### Refresh Token
```
✓ JWT expires: 15 minutes (short-lived)
✓ Refresh token expires: 30 days
✓ Refresh token rotation: every refresh → issue new token + revoke old
✓ One refresh token per authentication session (device_info is descriptive
  metadata about that session — see Session Management — not an identity key)
✓ Reused (already-rotated) refresh token → revoke entire family, force re-login
  (see Refresh Token Reuse Detection under Entity 2)
✓ Logout → revoke refresh token (JWT naturally expires in 15 min)
✓ Suspicious activity → revoke ALL refresh tokens for user
✓ SecurityStamp change → sst check fails on next request for every issued
  JWT (password changed, account suspended) — see Fast Session Invalidation
```

### Account Suspension / Deletion
```
Suspension (admin action):
✓ status → 'suspended'
✓ SecurityStamp regenerated → Redis security-stamp cache invalidated
  immediately → all existing JWTs fail the sst check on their very
  next request, not just at next natural expiry (see Fast Session
  Invalidation)
✓ New login blocked

Deletion (GDPR soft delete — Identity-level erasure only):
✓ status → 'deleted'
✓ Email → anonymized (deleted_uuid@deleted.onenex.com)
✓ Phone → null
✓ Name → 'Deleted User'
✓ Record kept for audit trail (financial records need user reference)
✓ SecurityStamp regenerated → same immediate Redis invalidation as suspension
✓ New login blocked

This anonymizes the GLOBAL identity only — it is not, by itself, a claim
that every business's data about this person is gone. A business that
holds this person as a `business_customers` guest/linked profile (CRM
module) or as a `staff_membership` (Membership module) is a separate data
controller for that relationship and handles its own retention/erasure
under its own rules (see `customer-module-design.md` §8.6, which makes
this exact split explicit for the CRM module). "Soft delete" here means
*the OneNex account is gone* — not that every business's record of this
person is gone too.
```

---

## JWT Design

There is exactly **one** JWT structure. It is issued at `/auth/login` and reissued in place — same `sid`, same claim shape — whenever its context changes (business selection, refresh). The only field that varies across a session's lifetime is `business_id`, which starts absent and gets populated once a business is selected; nothing else about the token's shape changes.

```
JWT (Access Token — issued at /auth/login, reissued at /auth/select-business
and /auth/refresh-token):
{
  "sub": "user_id",
  "sid": "session_id",          ← = refresh_tokens.family_id for this login
                                    session (see Session Management);
                                    constant across every reissuance of this
                                    token — proving a business selection or
                                    refresh is a context upgrade, not a
                                    second login
  "sst": "security_stamp",      ← AspNetUsers.SecurityStamp at issuance,
                                    checked live on every request — see
                                    Fast Session Invalidation
  "business_id": null,          ← absent/null until a business is selected
                                    via /auth/select-business; cleared again
                                    on every /auth/refresh-token — see
                                    "Business Context & Portal Access" below
  "jti": "unique_token_id",     ← per-token id, for audit/correlation only
                                    (revocation is sid/sst-based, not a
                                    per-jti denylist — see Fast Session
                                    Invalidation)
  "iss": "onenex-identity",
  "aud": "onenex-api",
  "iat": 1234567890,
  "exp": 1234568790             ← 15 minutes
}

email deliberately excluded: it's mutable, not needed for authorization,
and it goes stale the moment a user changes it mid-session. Callers that
need it call GET /auth/me rather than trusting a value frozen at issuance.

Signing: RS256 (asymmetric)
  → Private key signs (server only)
  → Public key verifies (can share with other services)
  → More secure than HS256 (symmetric)
```

The JWT deliberately carries **no business context at login**. A user can belong to many businesses (`onenex.ai/rio-jaffna`, `onenex.ai/bella-salon`, ...); embedding one business_id in the token at login would be wrong the moment they need a second business. See "Business Context & Portal Access" below for how and when `business_id` gets added.

---

## Web vs Mobile Token Delivery

Both client types hit the same Identity/Membership/Authorization stack —
this section changes only how the **refresh token** travels between
server and client. It does not change what a refresh token is, how it's
stored server-side, or how it rotates: `family_id` and Reuse Detection
(Entity 2) work identically for both paths. The access token (the JWT,
with or without `business_id`) is unaffected too — it goes in the JSON
response body for both client types, exactly as shown throughout this doc.

### Signal: `X-Client-Type` Header

```
X-Client-Type: web      → refresh token delivered via HttpOnly cookie
X-Client-Type: mobile   → refresh token delivered in the JSON body, as
                           documented in every endpoint below — also the
                           fallback if the header is omitted, so existing
                           mobile clients need no change
```

Sent on `/auth/login`, `/auth/refresh-token`, and `/auth/logout` — the
only three endpoints that ever touch a refresh token directly.

### Web: HttpOnly Cookie for the Refresh Token Only

```
Set-Cookie: onenex_rt=<opaque refresh token>;
            HttpOnly;                 ← unreadable by JS — the entire point
            Secure;                   ← HTTPS only
            SameSite=Strict;          ← see CSRF reasoning below
            Path=/auth                ← sent only to /auth/refresh-token and
                                         /auth/logout, never to business/
                                         operation endpoints
```

The access token still goes in the JSON body on web, exactly like mobile
— this is deliberately narrower than putting both tokens behind cookies.
Web JS holds the access token **in memory only** (a module-level variable
or store — never `localStorage`/`sessionStorage`, which XSS can read) and
attaches it manually via `Authorization: Bearer`, same as mobile. CSRF
protection is therefore only ever a concern for the refresh cookie itself
— every other endpoint stays exactly as CSRF-immune as pure Bearer auth
already was, because they never look at cookies at all.

**Why `SameSite=Strict` closes the CSRF question here without a separate
CSRF token:** this cookie is only ever sent by the OneNex web app's own
background `fetch`/XHR calls to `/auth/refresh-token` or `/auth/logout` —
never as the result of a top-level navigation from an external link, which
is the one case `Strict` behaves differently from `Lax`. A cross-site page
cannot get the browser to attach this cookie to a request at all, so it
cannot trigger a refresh on the victim's behalf.

This matters beyond the usual "CSRF is bad" reasoning: an attacker who
*could* force spurious refresh calls wouldn't even need to read the
response to do damage. Reuse Detection (Entity 2) means a forced extra
rotation racing the legitimate client's own refresh would make that
client's next refresh look like a replayed/stolen token and revoke the
whole session — a denial-of-service angle that costs the attacker nothing
and needs no read access to the response. `SameSite=Strict` prevents the
forced call from happening at all, which is a cleaner fix than trying to
make Reuse Detection tolerant of a race it can't distinguish from a real
attack.

**Bootstrap on page load:** web JS starts every hard reload with no access
token in memory (in-memory storage doesn't survive one, by design). The
app's first call is always `POST /auth/refresh-token` with no body — the
browser attaches `onenex_rt` automatically, the server returns a fresh
access token, and the app proceeds. No valid cookie (never logged in, or
it expired/was revoked) → 401 here → login screen, the same outcome as
mobile's "refresh token expired" path.

### CORS Requirement (Web Only)

Once any request carries a cookie, browsers require explicit credential
handling — not optional configuration, but required for the cookie to be
sent or set cross-origin at all:

```
Frontend fetch:  credentials: 'include'
Server response: Access-Control-Allow-Credentials: true
                 Access-Control-Allow-Origin: <exact origin>   ← never "*"
                     (browsers reject a wildcard origin combined with
                      credentials regardless of server config)
```

### This Is Not Cookie Authentication

"What We SKIP" still holds: no ASP.NET Core cookie-auth middleware, no
`SignInManager`, no session cookie the framework consults to authenticate
a request. `onenex_rt` is read manually, by name, only inside the
`/auth/refresh-token` and `/auth/logout` handlers — a storage vehicle for
the same opaque refresh-token value mobile sends as a JSON field, nothing
more. Every other endpoint's authentication is unchanged: Bearer JWT,
validated by signature/expiry/sst, exactly as documented in API
Authentication below.

---

## APIs

---

### POST /auth/register

**Request:**
```json
{
  "name": "Arun Kumar",
  "email": "arun@gmail.com",
  "phone": "+94771234567",
  "password": "SecurePass@123"
}
```

**Response (201):**
```json
{
  "message": "Registration successful. Please verify your email.",
  "userId": "uuid"
}
```

**Errors:**
```
400 → Email already exists
400 → Phone already exists
400 → Password does not meet requirements
400 → Invalid email format
```

**What happens internally:**
```
1. Validate request (FluentValidation)
2. Check phone uniqueness (custom — Identity doesn't do this)
3. UserManager.CreateAsync(user, password)
4. GenerateEmailConfirmationTokenAsync()
5. INotificationService.SendVerificationEmail(email, token)
6. Return 201
```

---

### POST /auth/verify-email

**Request:**
```json
{
  "userId": "uuid",
  "token": "verification_token"
}
```

**Response (200):**
```json
{
  "message": "Email verified successfully."
}
```

**Errors:**
```
400 → Invalid or expired token
400 → Already verified
404 → User not found
```

---

### POST /auth/resend-verification

**Request:**
```json
{
  "email": "arun@gmail.com"
}
```

**Response (200):**
```json
{
  "message": "Verification email sent."
}
```

**Rules:**
```
→ Max 3 resends per hour (rate limit)
→ If already verified → 400
→ If user not found → still return 200 (security: don't reveal if email exists)
```

---

### POST /auth/login

**Request:** (header `X-Client-Type: web|mobile` — see Web vs Mobile Token Delivery)
```json
{
  "email": "arun@gmail.com",
  "password": "SecurePass@123",
  "deviceInfo": "Chrome on Windows"
}
```

**Response (200) — mobile (or header omitted):**
```json
{
  "accessToken": "jwt_token",
  "refreshToken": "refresh_token",
  "expiresIn": 900
}
```

**Response (200) — web (`X-Client-Type: web`):**
```json
{
  "accessToken": "jwt_token",
  "expiresIn": 900
}
```
Plus `Set-Cookie: onenex_rt=<refresh_token>; HttpOnly; Secure; SameSite=Strict; Path=/auth` — `refreshToken` is omitted from the body; it never reaches web JS.

**Errors:**
```
401 → Invalid credentials (same message for wrong email OR wrong password — security)
401 → Email not verified
401 → Account suspended
423 → Account locked (too many failed attempts) + lockoutEnd timestamp
```

**What happens internally:**
```
1. Find user by email
2. Check status (active?)
3. Check email verified?
4. CheckPasswordAsync (Identity — also handles lockout increment)
5. Check lockout status
6. New family_id (this is a new authentication session)
7. Generate refresh token → hash → store in refresh_tokens (family_id = new)
8. Generate JWT { sub, sid=family_id, sst=current SecurityStamp, business_id=null, ... }
9. Reset AccessFailedCount
10. Write IdentitySecurityAudits: LoginSucceeded (or LoginFailed on any
    rejection above)
11. X-Client-Type: web    → Set-Cookie the refresh token, return accessToken only
    X-Client-Type: mobile
    (or header absent)    → return accessToken + refreshToken in the body
```

---

### POST /auth/refresh-token

**Request — mobile:**
```json
{
  "refreshToken": "refresh_token"
}
```

**Request — web:** no body required; the refresh token is read from the `onenex_rt` cookie, sent automatically by the browser.

**Response (200) — mobile:**
```json
{
  "accessToken": "new_jwt_token",
  "refreshToken": "new_refresh_token",
  "expiresIn": 900
}
```

**Response (200) — web:**
```json
{
  "accessToken": "new_jwt_token",
  "expiresIn": 900
}
```
Plus `Set-Cookie` with the newly-rotated value (same attributes as login).

**Errors:**
```
401 → Invalid refresh token
401 → Expired refresh token
401 → Revoked refresh token
```

**Token rotation:**
```
0. Read the token to rotate: web → onenex_rt cookie; mobile → refreshToken
   body field (see Web vs Mobile Token Delivery)
1. Hash incoming token → find in refresh_tokens
2. Row not found OR expired → 401
3. Row found with revoked_reason = 'rotation' → REUSE DETECTED:
     revoke entire family (see Refresh Token Reuse Detection, Entity 2)
     write IdentitySecurityAudits: RefreshTokenReuseDetected
     → 401 (force re-login, do not issue new tokens)
4. Row found, revoked_reason = 'logout'/'suspicious' → 401 (session ended)
5. Row valid → revoke it (revoked_at = NOW(), reason = 'rotation')
6. Re-check user still active + not locked (mirrors login's checks —
   catches suspension/deletion that happened mid-session)
7. Issue new refresh token → store (same family_id as the token just revoked)
8. Issue new JWT { sub, sid=family_id, sst=current SecurityStamp, business_id=null, ... }
   (business_id is always cleared on refresh — refresh only re-validates the
   account itself, never membership; the client must re-run
   /auth/select-business if it needs business context back — see "Business
   Context & Portal Access")
9. Write IdentitySecurityAudits: RefreshTokenRotated
10. web → Set-Cookie the new refresh token, return accessToken only
    mobile → return accessToken + refreshToken in the body
```

---

### POST /auth/logout

**Request:** (JWT in Authorization header)

Mobile:
```json
{
  "refreshToken": "refresh_token"
}
```

Web: no body required; reads the `onenex_rt` cookie.

**Response (200):**
```json
{
  "message": "Logged out successfully."
}
```

**What happens:**
```
1. Read the token to revoke: web → onenex_rt cookie; mobile → refreshToken
   body field
2. Find refresh token by hash
3. Set revoked_at = NOW(), reason = 'logout'
4. Web → clear the cookie (Set-Cookie onenex_rt=; Max-Age=0; same Path)
5. JWT naturally expires (15 min) — no blacklist needed
```

---

### POST /auth/forgot-password

**Request:**
```json
{
  "email": "arun@gmail.com"
}
```

**Response (200):**
```json
{
  "message": "If this email exists, a reset link has been sent."
}
```

**Security note:**
```
Always return same message — don't reveal if email exists in system.
```

---

### POST /auth/reset-password

**Request:**
```json
{
  "userId": "uuid",
  "token": "reset_token",
  "newPassword": "NewPass@456"
}
```

**Response (200):**
```json
{
  "message": "Password reset successful."
}
```

**What happens internally:**
```
1. UserManager.ResetPasswordAsync(user, token, newPassword)
2. SecurityStamp auto-regenerates (Identity does this)
3. Redis DEL security-stamp:{userId} → all existing JWTs fail the
   sst check on their next request (see Fast Session Invalidation) —
   immediate, not dependent on token expiry
4. All refresh tokens revoked (security — password changed)
5. Write IdentitySecurityAudits: PasswordChanged
```

---

### GET /auth/me

**Request:** (JWT in Authorization header)

**Response (200):**
```json
{
  "userId": "uuid",
  "name": "Arun Kumar",
  "email": "arun@gmail.com",
  "phone": "+94771234567",
  "emailVerified": true,
  "createdAt": "2026-01-15T10:30:00Z"
}
```

---

## Business Context & Portal Access

> Cross-module flow: Identity (who) + Membership (which businesses/branches, what role) + Business (resolve slug → business identity). This is the "One Account. Many Businesses. One Business at a Time." flow from the product design.

### Why business_id Is Populated Later, Not at Login

A OneNex login is business-agnostic — the same account can be staff at Rio Restaurant, Bella Salon and Dzine Retail, each with a different role and branch scope. The JWT issued at login (above) has `business_id` absent. Every business-scoped API call needs to know *which business* the request is operating in, and that context must be **verified against membership on every request** — never trusted from a client-selected value (see Membership module → Multi-Business / Tenant Isolation).

There is no second token type. The same JWT is simply **reissued in place** — same `sid`, same `sst` check, same 15-minute lifetime — with `business_id` now populated, once the user has picked a business:

```
JWT reissued at /auth/select-business — identical shape to the login JWT,
business_id now populated:
{
  "sub": "user_id",
  "sid": "session_id",          ← SAME family_id as the JWT this was
                                    reissued from — proving this is a
                                    context upgrade of the existing
                                    session, not a new login
  "sst": "security_stamp",      ← same live check as at login, see Fast
                                    Session Invalidation
  "business_id": "biz_uuid",    ← now populated
  "jti": "unique_token_id",
  "iss": "onenex-identity",
  "aud": "onenex-api",
  "iat": 1234567890,
  "exp": 1234568790             ← 15 minutes, same lifetime as at login
}
```

Even with `business_id` set, the JWT still carries **no permission list and no branch_id** — permissions are resolved live from Membership on every request (never cached in the token, so a permission change takes effect on the next call, not on next login), and branch is a per-request/per-resource concern checked against `staff_branch_access`, not a session-wide claim (a portal may let a user look at multiple branches' worth of data in one session, filtered by what they're allowed to see — it is not a single fixed value like business_id). Email stays excluded for the same reason as at login.

### Renewing business_id — No Separate Refresh Token

Selecting a business does **not** create its own refresh token or family. There is exactly one refresh-token family per login session (the one created at `/auth/login`), never one per selected business — an owner who bounces between five businesses in one sitting still has a single session, not five. `/auth/refresh-token` always hands back the JWT with `business_id` cleared (see JWT Design); `business_id` only comes back once `/auth/select-business` runs again. When the JWT's `business_id` expires (15 min) mid-session in a Business Portal:

```
1. Client gets 401 on a business-portal call
2. POST /auth/refresh-token (existing session's refresh token) → new JWT,
   business_id cleared
   (this re-validates the account itself is still active/not locked —
   same checks as login)
3. POST /auth/select-business { businessId } (new JWT) → JWT reissued with
   business_id populated again
   (this re-validates IMembershipService.IsActiveMember — catches a
   membership revoked mid-session, not just an account-level problem)
4. Retry the original request
```

Two round trips instead of one, only when `business_id` expires (every 15
minutes at most) — a fine trade for not building a second refresh-token
family type per business, which would otherwise multiply into a
business-scoped-token-per-business plus a refresh-per-business for every
business a multi-business owner touches.

### The Three-Step Flow

```
Step 1 — Owner Portal Login (OneNex/Login)
  POST /auth/login → JWT (business_id absent) + refresh token
  GET  /auth/me/memberships (JWT) → list of every business this user
       belongs to, with business_role + accessible branches
       (delegates to Membership.GetMyMemberships — see Membership module)

Step 2 — Select Business & Branch
  User picks a business (e.g. "Rio Restaurant") from the Owner Portal list
  POST /auth/select-business { businessId } (JWT) → JWT reissued with
       business_id populated
       Identity verifies the membership exists and is active
       (calls IMembershipService.IsActiveMember — 401/403 if not)

Step 3 — Business Portal (onenex.ai/{business-slug})
  All API calls now carry the JWT with business_id set
  Business module resolves the slug → businessId (GET /api/businesses/resolve/{slug})
       to confirm the URL matches the token's business_id
  Every branch-scoped request is still checked against staff_branch_access
       independently — business_id proves "this business", not "every branch in it"
```

### No In-Portal Business Switcher — By Design

Once inside a Business Portal, there is no control to switch to a different business. Switching means returning to the Owner Portal (or logging in again) and repeating Step 2 — or navigating directly to the other business's own URL (see "Direct Business URL Access" below), which re-runs the same `select-business` check under the hood. Either way, the point is that switching is never an in-page action: it is intentional isolation, not a missing feature, and it keeps a business-scoped session from ever silently carrying data across tenants. A branch switcher/filter *within* the current business portal is fine (see Membership module) — that is a different business's worth of isolation from a different branch's worth of isolation.

### Direct Business URL Access — No Central Portal Required

The Three-Step Flow above describes the UI shape for someone choosing among several memberships. It is not the only supported entry point, and it must not be read as requiring a central "Owner Portal" hop for everyone. A staff member who only ever works one business — the common case — may bookmark `onenex.ai/{business-slug}` and land there directly without ever seeing a memberships list. This needs no new endpoint and no session-model change: it is the same JWT issuance, then reissuance with `business_id`, already defined above, triggered by a URL instead of a list click.

```
GET onenex.ai/rio, no JWT with business_id set in memory (fresh page
load — the JWT is memory-only and never survives one, see "Bootstrap on
page load"):

1. Client runs the existing JWT bootstrap first:
     POST /auth/refresh-token (web: cookie-driven, mobile: stored refresh
     token — see Web vs Mobile Token Delivery)
   No valid refresh token (never logged in / expired / revoked)
     → 401 → show a login form on this same page — no redirect to a
        separate portal route
   Valid refresh token
     → a JWT (business_id absent) obtained silently, no form shown

2. If login was required: POST /auth/login → JWT + refresh token,
   exactly as documented elsewhere — identical regardless of which
   business's URL triggered it, because login is business-agnostic
   (see JWT Design)

3. business_id bootstrap — the step the Three-Step Flow doesn't cover,
   because it assumes a businessId already in hand from a list click:
     GET /api/businesses/resolve/rio → businessId
        (the same endpoint Step 3 of the Three-Step Flow already uses to
        confirm a URL against an existing business_id claim — here it
        runs BEFORE business_id is set, to supply the businessId
        select-business needs. One endpoint, two call sites, same
        guarantee: slug → businessId resolution is Business module's job
        either way, never something Identity or the frontend infers on
        its own)
     POST /auth/select-business { businessId } (JWT)
        → IMembershipService.IsActiveMember(userId, businessId) is
          checked exactly as in the list-driven flow — arriving via a
          URL grants no shortcut around this check
        → active member → JWT reissued with business_id set, page
          renders normally
        → not a member  → 403, see below

4. Retry the page's data calls, now carrying the JWT with business_id set
```

**403 on direct entry has different UX from the list-driven 403.** The existing 403 in `/auth/select-business` (see "New APIs" below) was written for a user choosing off their own membership list — reaching it there means something changed mid-session (a membership just got revoked). Direct URL entry hits this far more often and for more mundane reasons: a mistyped slug, a stale bookmark from a since-revoked membership, or someone guessing at a business's URL.

```
403 on direct entry → render "You don't have access to this business" on
  the business's own URL, not a generic error page. The business's
  existence is not secret — its public storefront is meant to be found
  (see customer-module-design.md, Scenario 2) — only staff-level access
  is being denied, so this message is safe to show plainly, unlike the
  list-driven 403's enumeration caution. If the visitor is logged in
  (they must be, to have reached this point), offer a link to their own
  GET /auth/me/memberships so they can find a business they do belong to.
```

No new refresh-token family, no new claim, no schema change: one refresh-token session backs however many businesses a staff member visits this way, each new slug simply re-running step 3 against the existing JWT.

STAFF TYPES
https://onenex.ai/rio
        │
        ▼
Frontend extracts slug
        │
        │
        ▼
Is JWT in memory?
        │
   ┌────┴────┐
   │         │
  YES        NO
   │         │
   │         ▼
   │   POST /auth/refresh-token
   │         │
   │    ┌────┴────┐
   │    │         │
   │   200       401
   │    │         │
   │    │         ▼
   │    │      Show login
   │    │         │
   │    │      POST /auth/login
   │    │         │
   │    └────┬────┘
   │         │
   └─────────┘
        │
        ▼
       JWT
 "I am Arun"
 (business_id: null)
        │
        ▼
GET /api/businesses/resolve/rio
        │
        ▼
businessId = RIO
        │
        ▼
POST /auth/select-business
        │
        ▼
Membership.IsActiveMember(
    Arun,
    Rio
)
        │
   ┌────┴────┐
   │         │
  YES        NO
   │         │
   ▼         ▼
JWT reissued  403
   │         "You don't
   │          have access"
   ▼
JWT =
Arun + business_id: Rio
   │
   ▼
Business Portal
   │
   ▼
GET /api/me/capabilities
   │
   ▼
Membership resolves
roles + permissions
   │
   ▼
Rio APIs
### New APIs

#### GET /auth/me/memberships

**Request:** (JWT in Authorization header)

**Response (200):**
```json
{
  "memberships": [
    {
      "businessId": "uuid",
      "businessName": "Rio Restaurant",
      "slug": "rio-jaffna",
      "businessRole": "member",
      "branches": [
        { "branchId": "uuid", "name": "Jaffna Branch", "isHeadquarters": true },
        { "branchId": "uuid", "name": "Colombo Branch", "isHeadquarters": false }
      ]
    },
    {
      "businessId": "uuid",
      "businessName": "Bella Salon",
      "slug": "bella-salon",
      "businessRole": "owner",
      "branches": [
        { "branchId": "uuid", "name": "Jaffna Branch", "isHeadquarters": true }
      ]
    }
  ]
}
```

`businessRole` is always one of Membership's fixed enum — `owner / admin / member` (`Custom_RBAC.md` §2.2) — never an operation role like "manager" or "staff". Per-operation roles (Dining: manager, Stays: viewer, ...) are a different, richer dimension that only gets resolved *after* a business is selected, via the business-scoped `GET /api/me/capabilities` (Membership module) — this cross-business picker deliberately stays coarse.

Powers the Owner Portal's business list/switcher. Delegates entirely to `IMembershipService.GetMyMemberships` — Identity does not know about roles or branches itself.

---

#### POST /auth/select-business

**Request:** (JWT in Authorization header)
```json
{
  "businessId": "uuid"
}
```

**Response (200):**
```json
{
  "accessToken": "jwt_token",
  "expiresIn": 900
}
```

**Errors:**
```
403 → No active membership for this business (never reveal whether the business exists to a non-member)
401 → JWT invalid/expired
```

**What happens internally:**
```
1. Validate JWT (signature/expiry + live sst check — see Fast Session Invalidation)
2. IMembershipService.IsActiveMember(userId, businessId) → 403 if false
3. Reissue the JWT { sub, sid=<same sid>, sst, business_id, ... } — see
   "Renewing business_id" for why sid is carried over rather than
   starting a new one
4. Write IdentitySecurityAudits: BusinessSelected
5. Return the reissued JWT (no new refresh token — refresh still rotates
   against the same session)
```

---

## Security Checklist

```
✓ Passwords → PBKDF2 hashing (Identity default)
✓ Refresh tokens → stored as hash (never plain)
✓ JWT → RS256 asymmetric signing
✓ JWT lifetime → 15 minutes (short)
✓ Lockout → 5 attempts → 15 min lock
✓ Email enumeration → same response for found/not-found
✓ Token single-use → verify + reset tokens (Identity)
✓ Refresh rotation → new token each use
✓ Refresh reuse detection → replaying an already-rotated token revokes its
  entire family and forces re-login (see Entity 2)
✓ SecurityStamp (sst) → embedded in the JWT at issuance, checked against
  a live Redis-cached value on every request — password change/suspension/
  deletion take effect on the very next request, not just next login (see
  Fast Session Invalidation)
✓ Identity security audit → security-relevant events logged in the same
  transaction as the state change (`identity_security_audits`)
✓ Soft delete → anonymizes the global identity; does not by itself imply
  every business's own record of this person is erased (see Account
  Suspension / Deletion)
✓ Rate limiting → resend verification max 3/hour
✓ HTTPS only → all endpoints
✓ business_id claim → re-verified against membership on every /auth/select-business call, never trusted from an existing token or client input
✓ business_id claim → carries no permission list and no branch_id; both re-checked live per request
✓ Every reissuance of the JWT (with or without business_id) shares the same
  sid/sst — a password reset or suspension invalidates the whole session
  through one single check
✓ Web refresh token → HttpOnly/Secure/SameSite=Strict cookie, scoped to
  `/auth` only — unreadable by JS, never sent to business/operation
  endpoints (see Web vs Mobile Token Delivery)
✓ Web access token → held in memory only, never localStorage/sessionStorage
✓ CORS (web) → explicit origin allowlist + credentials; never a wildcard
  origin once a cookie is involved
```

---

## Module Project Structure

```
OneNex.Identity/
├── Domain/
│   ├── Entities/
│   │   ├── ApplicationUser.cs        ← extends IdentityUser
│   │   ├── RefreshToken.cs           ← carries family_id
│   │   └── IdentitySecurityAudit.cs
│   └── Enums/
│       ├── UserStatus.cs
│       └── ClientType.cs             ← Web / Mobile — from X-Client-Type
│
├── Application/
│   ├── Features/
│   │   ├── Register/
│   │   │   ├── RegisterCommand.cs
│   │   │   ├── RegisterHandler.cs
│   │   │   └── RegisterValidator.cs
│   │   ├── Login/
│   │   │   ├── LoginCommand.cs
│   │   │   ├── LoginHandler.cs
│   │   │   └── LoginValidator.cs
│   │   ├── VerifyEmail/
│   │   ├── ResendVerification/
│   │   ├── ForgotPassword/
│   │   ├── ResetPassword/
│   │   ├── RefreshToken/
│   │   ├── Logout/
│   │   ├── GetMyMemberships/       ← delegates to IMembershipService
│   │   └── SelectBusiness/
│   │       ├── SelectBusinessCommand.cs
│   │       └── SelectBusinessHandler.cs   ← reissues the JWT with business_id set
│   └── Interfaces/
│       └── IJwtService.cs
│
├── Infrastructure/
│   ├── Services/
│   │   ├── JwtService.cs             ← JWT generation + validation
│   │   └── SecurityStampCacheService.cs  ← Redis get/invalidate for sst check
│   ├── Repositories/
│   │   ├── RefreshTokenRepository.cs     ← includes reuse-detection query
│   │   └── IdentitySecurityAuditRepository.cs
│   └── IdentityConfiguration.cs     ← Identity setup (lockout, password rules)
│
└── API/
    └── Controllers/
        └── AuthController.cs
```

---

## Session Management

### Active Sessions (Logout from Devices)

```
refresh_tokens table already stores:
  family_id     → identifies the session across rotations (a "device" in
                    product terms; internally it's a rotation chain, not
                    a literal hardware identity — see Entity 2)
  device_info   → "Chrome on Windows", "iPhone Safari"
  ip_address    → where login happened
  created_at    → when session started
  revoked_at    → null = still active

"Active sessions" = one row per DISTINCT family_id WHERE that family's
current (non-rotated) token has revoked_at IS NULL — a rotated-but-still-
active session has many historical rows in one family; only the latest
matters for the "active sessions" list.

V1:     Logout all devices → revoke every refresh_tokens row for this user,
        across all families
Phase 2: View sessions list (one row per family) + logout specific device
         (revoke that one family only)
```

**APIs:**
```
POST /auth/logout-all          ← V1
  → Revoke ALL refresh tokens for this user
  → User logged out from every device

GET  /auth/sessions            ← Phase 2
  → List all active sessions (device, ip, last used)

DELETE /auth/sessions/{id}     ← Phase 2
  → Logout one specific device
```

---

## API Authentication (How Protected Endpoints Work)

### Human Users — JWT Bearer

```
Every protected API call must include:
Authorization: Bearer <access_token>

ASP.NET Core middleware:
1. Reads Authorization header
2. Validates JWT (signature, expiry, issuer, audience)
3. Redis GET security-stamp:{userId} → compare to token's sst claim
   (see Fast Session Invalidation) — mismatch is treated as invalid
4. Valid   → sets User context → controller executes
5. Invalid → 401 returned, controller never reached
```

**Protected vs Public:**
```
PUBLIC (no token needed):
  POST /auth/register
  POST /auth/login
  POST /auth/verify-email
  POST /auth/forgot-password
  POST /auth/reset-password
  POST /auth/refresh-token

PROTECTED (JWT required):
  GET  /auth/me
  GET  /auth/me/memberships
  POST /auth/select-business
  POST /auth/logout
  POST /auth/logout-all
  GET  /auth/sessions          (Phase 2)
  DELETE /auth/sessions/{id}   (Phase 2)

PROTECTED (JWT required, business_id claim set — business-scoped):
  --- ALL business-portal / operation module endpoints ---
  (Dining, Stays, Bar, Wellness, Events, Retail, Membership staff
   management, Business settings, etc. — anything that operates
   "inside" a specific business)
```

**Token expiry flow:**
```
Access token expires (15 min):
→ 401 on next call
→ Client: POST /auth/refresh-token
→ Get new access token
→ Retry — user never notices

Refresh token expires (30 days):
→ POST /auth/refresh-token → 401
→ Client shows login screen
```

### Machine-to-Machine — API Keys

Some surfaces don't have a human login:
- KDS screen (kitchen display — device registered once)
- POS terminal (device registered once)
- Future: third-party integrations

These use **API Keys** — not JWT.

> **Forward reference, not an Identity-owned schema.** API keys are a
> Business-module concern by definition — a device credential belongs to
> a business, not to a human identity (see "API Keys owned by" below).
> `business-module-design.md` doesn't define this table yet; the shape
> below is what Identity assumes so JWT-vs-API-key auth can be documented
> together in one place. When the Business module doc adds its own
> `api_keys` design, this section should shrink to the boundary statement
> only: *"Identity does not issue or validate API keys — see Business
> module."*

```
api_keys table:
  id              UUID
  business_id     → businesses.id
  name            VARCHAR   ("KDS Kitchen 1", "POS Counter 2")
  key_hash        VARCHAR   (hashed — never store plain)
  key_prefix      VARCHAR   ("onx_live_a1b2c3...")  ← shown in UI
  permissions     JSONB     (what this key can do)
  last_used_at    TIMESTAMP
  expires_at      TIMESTAMP NULLABLE
  status          (active / revoked)
  created_at      TIMESTAMP

Request header:
  X-API-Key: onx_live_a1b2c3d4e5f6...
```

**API Key scope:**
```
V1:     Device registration (KDS, POS terminal)
Phase 3: Public API program for third-party developers
         → OAuth 2.0 Client Credentials (enterprise-grade)
```

**API Keys owned by:** Business Module (business-level concept, not identity)
**API Key validation:** Shared.Contracts → IApiKeyService

---

## Extra Security Layers

```
✓ HTTPS only               → token in transit always encrypted
✓ RS256 signing            → cannot forge token without private key
✓ Short expiry (15 min)    → stolen access token useless quickly
✓ Refresh token rotation   → reuse of an already-rotated token revokes the
                              whole family, not just a 401 (see Entity 2)
✓ SecurityStamp (sst)      → password change/suspension/deletion invalidate
                              every issued JWT — with or without business_id —
                              within one Redis-checked request, not just at
                              next login (see Fast Session Invalidation)
✓ SameSite=Strict (web)    → refresh cookie can't be attached to a cross-site
                              request at all, so a forced/CSRF'd refresh call
                              can't happen — and can't be used to trip Reuse
                              Detection against a legitimate client either
✓ API keys hashed          → stolen DB → keys useless
```

---

## Open Questions (Discuss Before Implementation)

**Resolved by this review pass:**

- ~~business_id claim lifetime~~ → **Decided: same 15 minutes as the rest
  of the JWT.** One expiry constant to reason about; revisit only if the
  extra `/auth/refresh-token` + `/auth/select-business` round trip (see
  "Renewing business_id") proves to be a real UX complaint, not a
  hypothetical one.
- ~~business_id refresh~~ → **Decided: no separate refresh-token family.**
  When the business_id claim expires, the client always falls back to
  refreshing the JWT (which clears business_id) then re-running
  `/auth/select-business` — see "Renewing business_id — No Separate
  Refresh Token" above.
- ~~SecurityStamp validation per request~~ → **Decided: V1, not deferred.**
  Without it, "suspend this account" or "reset this password" doesn't
  actually stop an active session for up to 15 minutes — too long for an
  abuse/compromise response to be credible. See Fast Session Invalidation.
  The cost (one Redis GET per request) matches what Membership already
  pays for its own permission checks.
- ~~Multiple devices: one refresh token per device or unlimited~~ →
  **Decided: unlimited sessions, each its own family_id** (see Entity 2 /
  Session Management) — a cap would need product input on what happens
  when it's hit (block new login? silently evict oldest?), which nothing
  today requires.

**Still open:**

- Password policy: current V1 default (8 chars + 1 upper + 1 number + 1
  special) rejects weak-but-compliant passwords like "Passw0rd!" while
  allowing them technically. A length-first policy (10-12 char minimum +
  reject breached/common passwords via a k-anonymity check, e.g. Have I
  Been Pwned's API) catches real-world weak passwords more reliably.
  Not changed here — it's a product/UX decision (changes registration
  copy and validation messaging), not a purely technical one.
- Two-factor auth: V1 or Phase 2? Either way, no schema migration is
  needed later — `IdentityUser.TwoFactorEnabled` already exists at zero
  cost via ASP.NET Core Identity, and its token storage is handled
  internally by `UserManager`. Deferring to Phase 2 is a scope decision,
  not a technical debt.
- Refresh token expiry: 30 days? Or shorter (7 days) for security?
- Lockout: 5 attempts / 15 min — or different thresholds?
- Phone number: required at registration or optional?
- Social login (Google): V1 or Phase 2?
- API keys: confirmed conceptually owned by Business module (see
  Machine-to-Machine section's forward-reference note) — open item is
  simply *when* that module's doc defines the table, not where it lives.
- Should `/auth/me/memberships` be cached client-side with a short TTL to avoid a round trip on every Owner Portal visit, given membership/role changes are infrequent?
