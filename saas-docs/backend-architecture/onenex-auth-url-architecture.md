# OneNex — Auth & URL Architecture

> How staff authentication, business context, and URL routing works across all channels.

---

## Core Concept — Two Separate Concerns

```
WHO you are  →  Authentication (email + password)
WHERE you are →  Context (which business, which operation)
```

These are always handled separately.

---

## URL Architecture

```
app.onenex.com              → Owner dashboard (all businesses)
{slug}.onenex.com           → Business staff web access
staff.theirown.com          → Custom domain (premium tier — DNS CNAME)
```

---

## Context Source by Channel

| Channel | Context Source | Example |
|---|---|---|
| Browser | Subdomain | `grandhotel.onenex.com` |
| Mobile App | QR Code + JWT | Scan once, always set |
| POS Terminal | Device registration | One-time setup |
| Custom Domain | DNS CNAME mapping | `staff.grandhotel.com` |

---

## Browser — Subdomain Pattern

### How it works

```
Staff goes to grandhotel.onenex.com
      ↓
Server reads subdomain → "grandhotel" → resolves to Business A
      ↓
Login form shows: "Grand Hotel Colombo"
      ↓
Staff enters email + password
      ↓
Server issues JWT:
  { user_id: 123, business_id: 456, role: "front-desk-agent" }
      ↓
Every API request → JWT verified → business context always known
No "which business?" asked. Ever.
```

### Cookie scoping
```
Cookie domain: .onenex.com (parent)
→ Works across all subdomains
→ Staff can have multiple business tabs open without re-login
```

### Security rule
```
Subdomain = context hint (UX)
JWT validation + membership check = actual security (always enforced server side)
Never trust subdomain alone.
```

---

## Mobile App — QR Code + JWT

```
First time setup:
Owner → OneNex → generates business QR code
Staff opens app → scans QR → business context stored securely in device

Daily use:
App opens → business context already set → login with PIN or biometric
JWT issued with tenant claim → all requests carry context
```

---

## POS Terminal / Kiosk — Device Registration

```
One-time setup:
Owner registers terminal in OneNex settings
→ Terminal receives device token
→ Terminal forever knows: "I am Business A, POS Station 2"

Daily use:
Staff walks up → types short PIN → logged in
Context from device registration. Zero friction.
```

---

## V1 Simple → V2 Proper Upgrade Path

### V1 (Start Simple)

```
Single domain: app.onenex.com

Login → 1 business → directly in
Login → 2+ businesses → picker shows → select → in
```

### V2 (Upgrade Later)

```
Subdomain: {slug}.onenex.com
QR code mobile setup
Device registration for terminals
Custom domain support
```

### What makes upgrade smooth — 3 MUST-HAVES from Day 1

#### 1. JWT from day 1 (no server-side sessions)

```
Login → issue JWT: { user_id, business_id, role }
Every API request → reads business_id from JWT

V2 upgrade: subdomain resolves business_id → same JWT structure
API changes: ZERO
```

#### 2. Business slug field — add from day 1

```
Business table:
  id:   456
  name: "Grand Hotel Colombo"
  slug: "grandhotel"   ← required from day 1

Later: grandhotel.onenex.com → slug → business_id 456
Without slug: subdomains impossible to retrofit
```

#### 3. Tenant resolution in one place only

```
AuthMiddleware → resolves business_id → attaches to request

V1: reads from JWT claim (user selected at login)
V2: reads subdomain → maps to business_id → embeds in JWT

One middleware change. Everything else: untouched.
```

### Exact changes from V1 → V2

```
1. Wildcard DNS: *.onenex.com → server
2. Middleware: read subdomain → resolve to business_id
3. Login page: pre-fill business from subdomain (no picker needed)
4. Redirect existing users to their subdomain URL

API layer:       ZERO changes
Database:        ZERO changes (slug already exists)
Business logic:  ZERO changes
```

---

## Anti-Patterns — Avoid These in V1

| Anti-pattern | Why dangerous |
|---|---|
| Server-side sessions | Must rewrite to JWT for subdomain upgrade — painful |
| No slug field | Subdomains need slug — impossible to retrofit cleanly |
| Tenant logic scattered in code | Cannot change context source in one place |
| Hardcoded `app.onenex.com` in code | Hunt and replace everywhere during upgrade |

---

## Summary

```
V1: app.onenex.com + JWT + business_slug + single auth middleware
                         ↓  (routing change only)
V2: *.onenex.com + same JWT + same slug + middleware reads subdomain
```

> Get the foundation right. The upgrade is just a routing change.
