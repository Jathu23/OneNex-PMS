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
✗ Cookie authentication      → JWT only, no cookies for API
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
LockoutEnd            → when lockout expires
LockoutEnabled        → lockout feature on/off
AccessFailedCount     → failed login count
SecurityStamp         → changes when password/security info changes
                        (used to invalidate existing tokens)
```

---

### Entity 2: RefreshTokens (Custom — Identity doesn't provide)

```sql
refresh_tokens:
  id              UUID          PK   DEFAULT gen_random_uuid()
  user_id         UUID          NOT NULL → AspNetUsers.Id
  token_hash      VARCHAR(500)  NOT NULL  (hashed — never store plain)
  expires_at      TIMESTAMP     NOT NULL  (30 days from issue)
  revoked_at      TIMESTAMP     NULLABLE  (null = still valid)
  revoked_reason  VARCHAR(100)  NULLABLE  (logout / rotation / suspicious)
  device_info     VARCHAR(200)  NULLABLE  (browser, OS — for display)
  ip_address      VARCHAR(45)   NULLABLE  (IPv6 max length)
  created_at      TIMESTAMP     NOT NULL  DEFAULT NOW()

INDEX: user_id, token_hash
```

**Why hashed:**
```
Refresh token stolen from DB → cannot be used directly.
Plain token only in client. Hash only in DB.
Same pattern as password storage.
```

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
     (data anonymized, record kept for audit)
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
✓ Issue JWT_1 { user_id, email } on success
✓ Issue refresh token → store hash in refresh_tokens table
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
✓ Send email via INotificationService
```

### Refresh Token
```
✓ JWT_1 expires: 15 minutes (short-lived)
✓ Refresh token expires: 30 days
✓ Refresh token rotation: every refresh → issue new token + revoke old
✓ One refresh token per device (device_info tracks this)
✓ Logout → revoke refresh token (JWT naturally expires in 15 min)
✓ Suspicious activity → revoke ALL refresh tokens for user
✓ SecurityStamp change → all JWTs invalid (password changed, account suspended)
```

### Account Suspension / Deletion
```
Suspension (admin action):
✓ status → 'suspended'
✓ SecurityStamp regenerated → all existing JWTs invalid immediately
✓ New login blocked

Deletion (GDPR soft delete):
✓ status → 'deleted'
✓ Email → anonymized (deleted_uuid@deleted.onenex.com)
✓ Phone → null
✓ Name → 'Deleted User'
✓ Record kept for audit trail (financial records need user reference)
✓ New login blocked
```

---

## JWT Design

```
JWT_1 (Access Token — global identity, issued at /auth/login):
{
  "sub": "user_id",
  "email": "arun@gmail.com",
  "jti": "unique_token_id",     ← for revocation
  "iat": 1234567890,
  "exp": 1234568790             ← 15 minutes
}

Signing: RS256 (asymmetric)
  → Private key signs (server only)
  → Public key verifies (can share with other services)
  → More secure than HS256 (symmetric)
```

JWT_1 deliberately carries **no business context**. A user can belong to many businesses (`onenex.ai/rio-jaffna`, `onenex.ai/bella-salon`, ...); embedding one business_id in the login token would be wrong the moment they need a second business. See "Business Context & Portal Access" below for JWT_2, the business-scoped token minted after a business is selected.

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

**Request:**
```json
{
  "email": "arun@gmail.com",
  "password": "SecurePass@123",
  "deviceInfo": "Chrome on Windows"
}
```

**Response (200):**
```json
{
  "accessToken": "jwt_token",
  "refreshToken": "refresh_token",
  "expiresIn": 900
}
```

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
6. Generate JWT_1 (access token)
7. Generate refresh token → hash → store in refresh_tokens
8. Reset AccessFailedCount
9. Return tokens
```

---

### POST /auth/refresh-token

**Request:**
```json
{
  "refreshToken": "refresh_token"
}
```

**Response (200):**
```json
{
  "accessToken": "new_jwt_token",
  "refreshToken": "new_refresh_token",
  "expiresIn": 900
}
```

**Errors:**
```
401 → Invalid refresh token
401 → Expired refresh token
401 → Revoked refresh token
```

**Token rotation:**
```
1. Hash incoming token → find in refresh_tokens
2. Validate (not expired, not revoked)
3. Revoke old token (revoked_at = NOW(), reason = 'rotation')
4. Issue new JWT_1
5. Issue new refresh token → store
6. Return new tokens
```

---

### POST /auth/logout

**Request:** (JWT in Authorization header)
```json
{
  "refreshToken": "refresh_token"
}
```

**Response (200):**
```json
{
  "message": "Logged out successfully."
}
```

**What happens:**
```
1. Find refresh token by hash
2. Set revoked_at = NOW(), reason = 'logout'
3. JWT naturally expires (15 min) — no blacklist needed
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
3. All existing JWTs now invalid (SecurityStamp changed)
4. All refresh tokens revoked (security — password changed)
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

### Why a Second Token

A OneNex login is business-agnostic — the same account can be staff at Rio Restaurant, Bella Salon and Dzine Retail, each with a different role and branch scope. JWT_1 (above) only proves *who* logged in. Every business-scoped API call needs to know *which business* the request is operating in, and that context must be **verified against membership on every request** — never trusted from a client-selected value (see Membership module → Multi-Business / Tenant Isolation).

The solution is a second, short-lived token minted only after the user has picked a business:

```
JWT_2 (Business-Scoped Access Token — issued at /auth/select-business):
{
  "sub": "user_id",
  "email": "arun@gmail.com",
  "business_id": "biz_uuid",
  "jti": "unique_token_id",
  "iat": 1234567890,
  "exp": 1234568790             ← 15 minutes, same lifetime as JWT_1
}
```

JWT_2 still carries **no permission list and no branch_id** — permissions are resolved live from Membership on every request (never cached in the token, so a permission change takes effect on the next call, not on next login), and branch is a per-request/per-resource concern checked against `staff_branch_access`, not a session-wide claim (a portal may let a user look at multiple branches' worth of data in one session, filtered by what they're allowed to see — it is not a single fixed value like business_id).

### The Three-Step Flow

```
Step 1 — Owner Portal Login (OneNex/Login)
  POST /auth/login → JWT_1 + refresh token
  GET  /auth/me/memberships (JWT_1) → list of every business this user
       belongs to, with business_role + accessible branches
       (delegates to Membership.GetMyMemberships — see Membership module)

Step 2 — Select Business & Branch
  User picks a business (e.g. "Rio Restaurant") from the Owner Portal list
  POST /auth/select-business { businessId } (JWT_1) → JWT_2
       Identity verifies the membership exists and is active
       (calls IMembershipService.IsActiveMember — 401/403 if not)

Step 3 — Business Portal (onenex.ai/{business-slug})
  All API calls now carry JWT_2
  Business module resolves the slug → businessId (GET /api/businesses/resolve/{slug})
       to confirm the URL matches the token's business_id
  Every branch-scoped request is still checked against staff_branch_access
       independently — JWT_2 proves "this business", not "every branch in it"
```

### No In-Portal Business Switcher — By Design

Once inside a Business Portal, there is no control to switch to a different business. Switching means returning to the Owner Portal (or logging in again) and repeating Step 2. This is intentional isolation, not a missing feature: it keeps a business-scoped session from ever silently carrying data across tenants. A branch switcher/filter *within* the current business portal is fine (see Membership module) — that is a different business's worth of isolation from a different branch's worth of isolation.

### New APIs

#### GET /auth/me/memberships

**Request:** (JWT_1 in Authorization header)

**Response (200):**
```json
{
  "memberships": [
    {
      "businessId": "uuid",
      "businessName": "Rio Restaurant",
      "slug": "rio-jaffna",
      "businessRole": "manager",
      "branches": [
        { "branchId": "uuid", "name": "Jaffna Branch", "isHeadquarters": true },
        { "branchId": "uuid", "name": "Colombo Branch", "isHeadquarters": false }
      ]
    },
    {
      "businessId": "uuid",
      "businessName": "Bella Salon",
      "slug": "bella-salon",
      "businessRole": "staff",
      "branches": [
        { "branchId": "uuid", "name": "Jaffna Branch", "isHeadquarters": true }
      ]
    }
  ]
}
```

Powers the Owner Portal's business list/switcher. Delegates entirely to `IMembershipService.GetMyMemberships` — Identity does not know about roles or branches itself.

---

#### POST /auth/select-business

**Request:** (JWT_1 in Authorization header)
```json
{
  "businessId": "uuid"
}
```

**Response (200):**
```json
{
  "accessToken": "jwt_2_token",
  "expiresIn": 900
}
```

**Errors:**
```
403 → No active membership for this business (never reveal whether the business exists to a non-member)
401 → JWT_1 invalid/expired
```

**What happens internally:**
```
1. Validate JWT_1
2. IMembershipService.IsActiveMember(userId, businessId) → 403 if false
3. Generate JWT_2 { sub, email, business_id }
4. Return JWT_2 (no new refresh token — refresh still rotates against JWT_1's session)
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
✓ SecurityStamp → invalidates all tokens on password change
✓ Soft delete → GDPR compliant
✓ Rate limiting → resend verification max 3/hour
✓ HTTPS only → all endpoints
✓ JWT_2 → business context re-verified against membership on every /auth/select-business call, never trusted from JWT_1 or client input
✓ JWT_2 → carries no permission list and no branch_id; both re-checked live per request
✓ SecurityStamp change (password reset/suspension) → invalidates JWT_2s too, since they derive from the same user identity
```

---

## Module Project Structure

```
OneNex.Identity/
├── Domain/
│   ├── Entities/
│   │   ├── ApplicationUser.cs        ← extends IdentityUser
│   │   └── RefreshToken.cs
│   └── Enums/
│       └── UserStatus.cs
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
│   │       └── SelectBusinessHandler.cs   ← mints JWT_2
│   └── Interfaces/
│       └── IJwtService.cs
│
├── Infrastructure/
│   ├── Services/
│   │   └── JwtService.cs             ← JWT generation + validation
│   ├── Repositories/
│   │   └── RefreshTokenRepository.cs
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
  device_info   → "Chrome on Windows", "iPhone Safari"
  ip_address    → where login happened
  created_at    → when session started
  revoked_at    → null = still active

"Active sessions" = all refresh_tokens WHERE revoked_at IS NULL

V1:     Logout all devices
Phase 2: View sessions list + logout specific device
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
3. Valid   → sets User context → controller executes
4. Invalid → 401 returned, controller never reached
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

PROTECTED (JWT_1 required):
  GET  /auth/me
  GET  /auth/me/memberships
  POST /auth/select-business
  POST /auth/logout
  POST /auth/logout-all
  GET  /auth/sessions          (Phase 2)
  DELETE /auth/sessions/{id}   (Phase 2)

PROTECTED (JWT_2 required — business-scoped):
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
✓ Refresh token rotation   → reuse of old token = suspicious activity
✓ SecurityStamp            → password change invalidates all tokens
✓ API keys hashed          → stolen DB → keys useless
```

---

## Open Questions (Discuss Before Implementation)

- JWT signing: RS256 (asymmetric) or HS256 (symmetric)? RS256 recommended.
- Refresh token expiry: 30 days? Or shorter (7 days) for security?
- Lockout: 5 attempts / 15 min — or different thresholds?
- Phone number: required at registration or optional?
- Social login (Google): V1 or Phase 2?
- Two-factor auth: V1 or Phase 2?
- Multiple devices: one refresh token per device or unlimited?
- SecurityStamp validation per request: V1 or Phase 2?
- API keys: store in Identity module or Business module?
- JWT_2 lifetime: same 15 minutes as JWT_1, or longer since re-selecting a business is cheap and frequent (e.g. multi-business owners bouncing between portals)?
- JWT_2 refresh: does `/auth/refresh-token` need a business-scoped variant, or does a expired JWT_2 always fall back to re-running `/auth/select-business` with the still-valid JWT_1?
- Should `/auth/me/memberships` be cached client-side with a short TTL to avoid a round trip on every Owner Portal visit, given membership/role changes are infrequent?
