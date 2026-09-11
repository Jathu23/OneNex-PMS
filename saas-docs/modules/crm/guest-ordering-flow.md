# Guest Ordering Flow — QR → Order, Fully Anonymous

> Status: DRAFT — proposed design, needs team review before implementation.
> Implements `identity-decisions.md` D2's "Place order via QR" row. Spans
> three modules: the ordering/Dining module (owns `orders`), CRM (owns
> `business_customers`), and Identity (owns the signing keypair the guest
> credentials reuse). Every schema change this design needs is listed in
> §7; every endpoint in §8; the end-to-end walkthrough is §13.

---

## 1. Decision

**Guest QR ordering captures no contact information at all.** Not name,
not phone, not email.

### Why ordering can go further than booking

Online room booking still needs a phone number — without one there's no
way to handle a no-show, a cancellation, or check-in identity matching.
That requirement is unchanged (`identity-decisions.md` D2's booking row).
Food ordering and dining are a different shape, and only *they* get the
zero-contact guest flow:

```
Booking → has real-world logistics that need a contact:
  - a room needs a name for check-in / ID matching
  - a no-show or cancellation needs someone to reach
  - the booking module already assumes a contactable guest (D2's
    booking row, guest_action_tokens idea for cancel/modify)

Ordering (QR, in-person) → has none of that:
  - the guest is physically present at the table/counter already
  - fulfillment doesn't require reaching them afterward — the food
    either gets delivered to the table/counter or it doesn't, in the
    same few minutes
  - there is no "no-show" concept for an order already being cooked
```

So dropping contact capture is safe specifically *because* the guest is
present and the interaction is short-lived — it would not be a safe
default for booking, and this doc does not propose extending it there.

---

## 2. What This Means for the Data Model

No `business_customers` row is created when a QR order is placed.
`LookupOrCreateAsync` is **not called** on that path — there is no
phone/email to look up or create against, and `chk_has_contact`
(`customer-module-design.md` §4.1) means a row couldn't be created even
if we wanted one.

```
Order recorded in DB:  ✅ always — full detail, independent of this doc
CRM link (business_customers):  ✅ but only once contact is actually
                                    given — via the optional receipt
                                    opt-in (§10), or when an account
                                    claims the orders (§12). Never at
                                    order-placement time.
Repeat-visit recognition:  ✅ possible, anonymously, with no login and no
                              contact ever given — the Guest Device ID
                              (§12) lets a returning guest's past orders
                              at this business surface again automatically
Claim-to-account later:  ✅ possible — auto-linked the moment the guest
                             logs in (§12's device-based claim), with no
                             manual "claim" step for the guest to take.
                             This is D6's *outcome* (orders end up
                             attached to the account) reached by a
                             weaker-trust mechanism than D6's own
                             verified-phone/email match — see §12
```

**When a customer record does get created.** If the guest later wants an
emailed invoice/receipt, that is the moment a record has to exist — you
cannot email an invoice to nobody. Two distinct things can get created,
and they are not the same:

| | Created when | What it is |
|---|---|---|
| `business_customers` row (business-specific) | Guest gives phone/email via receipt opt-in (§10) | A CRM record at *that one business*. No login, no password. `user_id` stays NULL — it's a guest profile |
| `ApplicationUser` (global OneNex account) | Guest registers through Identity's normal flow | A real login, usable across all of OneNex. Only this enables §12's auto-claim, and D6's verified-match claim |

A guest can end up with the first and never the second (opt in for a
receipt, never register), or the second and never the first (register
first, then order). Neither implies the other.

---

## 3. Three Guest Credentials — None of Them the Identity JWT

OneNex has exactly **one** JWT structure (`identity-module-design.md` →
"JWT Design"), issued only to a real `ApplicationUser` (`sub` =
`users.id`). An anonymous guest has no `ApplicationUser`, and — at order
time — no `business_customers` row either. So none of the credentials
below can be a variant of the Identity JWT; there is no `sub` to put in
one.

| | Identity JWT | Order Access Token (§4) | Confirmation code (§11) | Guest Device ID (§12) |
|---|---|---|---|---|
| Ties to | `users.id` | One `orders.id` | One `orders.id` | One browser/device, at one business |
| Spans | A login session | A single order | A single order | Many orders, across visits |
| Issued by | Identity, at `/auth/login` | Ordering module, at order creation | Ordering module, at order creation | The client itself — nothing is issued |
| Signed | Yes (RS256) | Yes (RS256, own claim shape) | No — a short random string | No — a random UUID |
| Strength | Verified identity | Unguessable (122-bit class) | Weak — short, brute-forceable without limits | Unguessable (122-bit), but unverified |
| Grants | Full account access | Status read + receipt opt-in on that order | Status read only, on that order | Order history for that device + claim eligibility |

Note the correction to an earlier draft of this doc, which claimed "there
is no session-spanning credential in this design." That is true of the
**Order Access Token** specifically — it is per-order and never reused.
It is *not* true of the design as a whole: the Guest Device ID (§12) is
deliberately session- and visit-spanning. That is the entire reason it
exists.

---

## 4. Order Access Token (OAT) — Shape

```
{
  "typ": "guest_order_access",   ← discriminator; rejected by any endpoint
                                     expecting the Identity JWT, and vice
                                     versa
  "order_id": "...",             ← the ONE order this token can act on
  "business_id": "...",          ← defense in depth alongside order_id
  "jti": "...",
  "iat": ..., "exp": ...          ← short-lived, see below
}
```

Signed with the same RS256 keypair Identity already manages — no new
signing infrastructure, just a second signed claim shape (the Identity
JWT being the first), validated by its own code path.

### Issuance

At order creation, and only then:

```
POST /api/businesses/{businessId}/orders   (no auth — see §6 for the
                                              abuse trade-off this accepts)
  X-Guest-Device-Id: <guestDeviceId>       ← optional (§12); absent is fine
  { branchId, tableNumber?, items: [...] }  ← no name, no phone, no email

Server (ordering module):
  1. Validate items/pricing against the business's menu (its own concern)
  2. INSERT order — full detail, with:
       business_customer_id = NULL      ← column exists, just unset (§7)
       guest_device_id      = <header, or NULL if not sent>
       confirmation_code    = <generated, §11>
  3. Issue an Order Access Token scoped to this one order_id
  4. Return { order, orderAccessToken, confirmationCode }
```

### Client storage & use

The client holds the OAT for as long as it cares to check status — in
memory or `sessionStorage` for that order's receipt screen. It is not
worth persisting across a browser restart: it expires quickly, and the
Guest Device ID (§12) is the thing that survives instead.

```
GET /api/businesses/{businessId}/orders/{orderId}
  Authorization: Bearer <OAT>
  → server checks token.order_id == path {orderId}
       AND token.business_id == path {businessId}
  → returns that order only if both match
```

### Expiry

Short — long enough to cover one order's realistic lifecycle (placed →
preparing → ready → served), e.g. **1–2 hours**. There is no refresh
endpoint. Two separate fallbacks exist for after it expires, covering
two different failure modes:

- Still on the same device → the Guest Device ID (§12) still works, and
  has no expiry.
- Different device, or storage cleared → the confirmation code (§11),
  which is why it exists despite the GDI also existing.

### Placing a second order in the same visit

The OAT does not link them — each order gets its own, and the first
grants nothing toward the second. But the orders are **not** unlinked in
general: if the client sends the same `X-Guest-Device-Id` (§12), both
rows carry the same `guest_device_id`, and `GET /orders/by-device`
returns them together. That is the intended grouping mechanism for "what
has this guest ordered."

What the device ID deliberately does *not* do is drive billing. If a
business needs to bill several orders together at one table, that happens
at the **table** level (`branchId`/`tableNumber`, on every order) — one
physical table is often several people with several devices, and one
device can move tables. Table-level consolidation is a POS/ordering-module
concern; this doc doesn't define it.

---

## 5. What the OAT Authorizes — And Nothing More

Scope note: this section is about the OAT specifically. The confirmation
code's limits are in §11, the Guest Device ID's in §12.

```
✓ GET  /api/businesses/{businessId}/orders/{orderId}
    → that one order, nothing else
✓ POST /api/businesses/{businessId}/orders/{orderId}/receipt-optin
    → attach contact info to exactly that order (§10)

✗ Any other order_id, even at the same business
✗ /api/customers/me/*  → meaningless here; there is no account, no
    business_customers row, nothing to be "me"
✗ Any staff/RBAC-gated endpoint (crm:*, membership:*, etc.)
    → rejected on typ alone, before any permission check runs
✗ Placing a second order
    → order creation takes no auth at all (§4); a fresh OAT is issued
      per order, the old one grants nothing toward a new one
✗ Reading other orders from the same device
    → that's the Guest Device ID's job (§12), not the OAT's
```

---

## 6. Security Trade-off This Accepts

Dropping contact capture removes the one piece of friction that
(incidentally) made order spam mildly costly to an attacker. With **zero**
input required to create an order, `POST /orders` is a fully open,
unauthenticated write endpoint — the single highest-risk endpoint in this
whole design, and it must be treated that way explicitly, not left as an
afterthought:

- **Hard per-IP and per-table/branch rate limiting** on order creation —
  tighter than any other public endpoint in the system, because there is
  no account or verified phone to fall back on for abuse detection.
- **`guest_device_id` is an additional abuse signal** (§12) — a single
  device firing dozens of orders is suspicious in a way the endpoint
  otherwise can't see. It is client-supplied and therefore trivially
  rotated by a determined attacker, so it may *raise* suspicion but must
  never be the only control; IP/branch limits stay mandatory.
- Consider requiring the QR-encoded URL to include a short, business-
  rotated table/session code (printed on the physical QR, rotated
  periodically by staff) rather than a bare `branch_id` — this doesn't
  identify the *guest*, but it makes it harder for a script to
  mass-generate orders without ever having scanned a real code. Not
  decided here; open question (§14).
- Kitchen/POS display should treat a burst of orders from one
  table/branch in a short window as suspicious, independent of anything
  these credentials prove.
- This is the direct, accepted cost of the decision in §1 — going in
  eyes-open, not something to quietly patch over later.

---

## 7. Schema Changes Required

No new table. Everything this design needs lives on the `orders` row the
ordering module already has to write — but it is **three** columns, not
one, and an earlier draft of this doc only ever named one of them:

```sql
ALTER TABLE orders
  -- NULL for a fully anonymous order. Set later by the receipt opt-in
  -- (§10) or the device claim (§12). The column always exists; §4's
  -- flow simply leaves it unset.
  ADD COLUMN business_customer_id uuid NULL REFERENCES business_customers(id),

  -- The guest's device correlation tag (§12). Not a foreign key — there
  -- is no table of devices, deliberately. NULL if the client never sent
  -- one.
  ADD COLUMN guest_device_id uuid NULL,

  -- Human-facing recovery code (§11). Must be stored to be looked up.
  ADD COLUMN confirmation_code varchar(10) NULL;

-- Device history lookup: GET /orders/by-device (§12)
CREATE INDEX ix_orders_business_device
  ON orders(business_id, guest_device_id)
  WHERE guest_device_id IS NOT NULL;

-- Confirmation-code lookup (§11). Unique per branch among orders still
-- in their lookup window — codes are short and MUST be reusable once an
-- order ages out, or a branch exhausts the namespace in a day.
-- Status vocabulary is the Dining module's own lifecycle:
-- placed → preparing → ready → served, plus cancelled/voided.
CREATE UNIQUE INDEX uq_orders_active_confirmation_code
  ON orders(branch_id, confirmation_code)
  WHERE confirmation_code IS NOT NULL
    AND status NOT IN ('served', 'cancelled', 'voided');
```

Why still "no new table":

- There is no CRM state to create at order time (§2) — that only happens
  through `LookupOrCreateAsync` in §10/§12, which writes
  `business_customers`, a table CRM already owns.
- The OAT and the Guest Device ID are both stateless as credentials:
  the OAT is a signature + claim check with no DB round-trip, and the
  GDI is validated only by string equality against the column above.
  Neither needs a session/token table, so neither gets one.
- The confirmation code is the one piece that *must* be persisted — you
  cannot resolve a code you never stored — which is exactly why it's a
  column above rather than another stateless token.

(An earlier draft justified "no session table" with "there is nothing to
anchor a session to." That reasoning is now wrong — `guest_device_id` is
precisely such an anchor. The conclusion still holds, but for the
different reason above: an indexed column answers the same question a
table would, without a second identity-shaped entity shadowing
`business_customers`.)

---

## 8. API Shape

| Method | Endpoint | Auth | Purpose |
|---|---|---|---|
| GET | `/api/businesses/{businessId}/menu` | none | Anonymous menu view (D2, unchanged) |
| POST | `/api/businesses/{businessId}/orders` | none (optional `X-Guest-Device-Id`) | Place an order — no contact info in the body at all. Returns `{ order, orderAccessToken, confirmationCode }`. Rate-limited hard (§6) |
| GET | `/api/businesses/{businessId}/orders/{orderId}` | Order Access Token | Poll status — scoped to exactly that order (§5) |
| POST | `/api/businesses/{businessId}/orders/{orderId}/receipt-optin` | Order Access Token **or** Guest Device ID | Optionally attach `{fullName, phone?, email?}` — triggers `LookupOrCreateAsync` and links the order (§10) |
| GET | `/api/businesses/{businessId}/orders/lookup?code=...&branchId=...` | none (see §11 constraints) | Status-only fallback when both the OAT and the device are gone — weaker, rate-limited, expires with the order |
| GET | `/api/businesses/{businessId}/orders/by-device` | Guest Device ID (header) | Full order history at this business for this device — no account, no login (§12) |
| POST | `/api/businesses/{businessId}/customers/claim-by-device` | JWT (no business_id) | Auto-links every unclaimed device-tagged order to the logged-in account — CRM-owned; resolves identity then publishes `GuestDeviceClaimedEvent` for the ordering module to apply (§12) |

---

## 9. Relationship to Canonical Decisions

- **`identity-decisions.md` D2** — this doc is the "how" for D2's updated
  "Place order via QR" row (fully anonymous). D2's booking row is
  unaffected — see §1 for why booking still needs a contact.
- **`customer-module-design.md` §6.1, §9, §4.1** — unchanged, and reused
  as-is. The *ordering* path deliberately does not call
  `LookupOrCreateAsync`; §10's receipt opt-in and §12's claim both do
  call it (and `AttachToBusinessAsync`) with exactly the semantics those
  sections already define. This doc adds no new CRM linking rules.
- **`identity-module-design.md` "JWT Design"** — unaffected; this doc
  reuses only the signing keypair, never the claim shape or issuance
  flow, and the OAT is explicitly not a variant of it (§3).
- **`onenex-backend-module-boundaries.md` "Module Communication Rules"**
  — §12's `claim-by-device` follows that doc's rule that a module may
  never write another module's tables directly: CRM resolves identity and
  publishes `GuestDeviceClaimedEvent`; the ordering module applies it to
  its own `orders`. See §12.

---

## 10. Recovering CRM Value — Optional Receipt Opt-In

A guest can *choose* to give contact info after ordering, to get a
receipt and (as a side effect) become a real, recognizable, claimable
`business_customers` guest — without that ever being a requirement to
place the order.

```
POST /api/businesses/{businessId}/orders/{orderId}/receipt-optin
  Authorization: Bearer <OAT>        ← either this…
  X-Guest-Device-Id: <guestDeviceId> ← …or this (see "which credential", below)
  { fullName, phone?, email? }       ← at least one of phone/email,
                                         same rule as chk_has_contact

Server (ordering module — this endpoint is under /orders/, which it owns):
  1. Authorize: OAT matching this order_id + business_id, OR a device id
     matching this order's guest_device_id
  2. ICustomerService.LookupOrCreateAsync(businessId, phone, email,
       fullName, branchId, capturedByUserId: null)
       ← exactly the call customer-module-design.md §6.1/§9 defines;
         CRM publishes CustomerCapturedEvent from inside it — the
         ordering module does not publish CRM's events itself
  3. UPDATE orders SET business_customer_id = <result> WHERE id = orderId
       ← its own table, using an id resolved through CRM's public
         interface: no boundary violation
```

From here the guest behaves exactly like any other guest
`business_customers` row: recognizable on their next visit by phone, and
eligible for D6's verified claim flow the same as a staff-captured
walk-in.

**Which credential authorizes it.** Either the OAT or the Guest Device
ID is accepted, and this matters more than it looks. If only the OAT
were accepted, opt-in would silently expire 1–2 hours after the order
(§4) — so a guest who decides the next day that they want an invoice
would have no route at all. The GDI has no expiry, so accepting it keeps
"can I get an invoice for last night's order?" answerable. Both are
122-bit unguessable values held only by the ordering client, so they
carry equivalent trust here; neither is weak enough to worry about, and
the confirmation code (§11) is deliberately *not* accepted, being the
one credential that is.

**Why this stays a separate step, not a field on `POST /orders`:**
folding an optional contact field into order creation would put a "want
to give your info?" prompt in the critical path of ordering — exactly
the friction §1 rejected. As a follow-up call, the guest sees their order
confirmed first, with receipt capture as a clearly skippable next action.

---

## 11. Lost-Token Recovery — Confirmation Code

Order creation also returns a short, human-facing **confirmation code**,
stored as `orders.confirmation_code` (§7). It covers the one case the
other two credentials can't: the guest no longer has the device that
placed the order (storage cleared, different phone, borrowed someone
else's device to order). If they still have the device, §12's Guest
Device ID already answers this with no code needed.

```
POST /orders response:
  {
    order: {...},
    orderAccessToken: "<OAT — short-lived, this browser>",
    confirmationCode: "482-916"   ← short, printable/displayable,
                                     independent of both other credentials
  }

GET /api/businesses/{businessId}/orders/lookup?code=482-916&branchId=...
  → no auth (this IS the path for someone with no token and no device) —
    but see the constraints below; this is not a free-for-all lookup
```

This is a **deliberately weaker** credential than the other two, and must
be constrained accordingly:

- **Read-only, status only** — never order contents/pricing/items, only
  e.g. `"preparing" / "ready" / "served"`. Limits what a successful guess
  actually exposes.
- **Requires `branchId` alongside the code** — narrows the namespace, and
  matches the uniqueness index in §7, which is scoped the same way.
- **Expires with the order's lifecycle** — once an order is terminal
  (served/cancelled/voided) past a short grace window, the code stops
  resolving, and §7's partial unique index releases it for reuse.
- **Rate-limited at least as strictly as `POST /orders`** (§6) — a
  6-digit-class code has real brute-force exposure otherwise; per-IP
  throttling *and* the branch scoping are both required, not either/or.
- **Never accepted for any write** — not the receipt opt-in (§10), not
  the device claim (§12). Reads only. It's the one guest credential
  weak enough that this restriction is load-bearing.

---

## 12. Repeat Visits and Later Account-Linking — Guest Device ID

The scenario: the same person orders three times as a fully anonymous
guest, then later registers a OneNex account (or logs into an existing
one) and should get those three orders attached to it. With zero
information captured, there is nothing about *the person* to match on.
What *can* be correlated is "the same browser/device placed these
orders" — weaker than D6's verified match, but enough, and by decision
(GO10) it is applied automatically rather than offered for confirmation.

### Guest Device ID (GDI) — not a token, not signed, not identity

```
Client, on first ORDER at a business (deliberately not on menu view —
see below):
  guestDeviceId = crypto.randomUUID()      ← 122 bits, generated
                                               entirely client-side
  → stored in localStorage, scoped per business:
      guest_device_<businessId>
  → sent on every subsequent order at THAT business, and on the
    by-device / receipt-optin calls:
      X-Guest-Device-Id: <guestDeviceId>
```

**Generated at first order, not at first menu view.** An earlier draft
left this open ("either works"); it shouldn't be. Generating on menu view
would write client storage for every passer-by who only ever looks at the
menu — worse for privacy, and it makes the menu page non-trivially
stateful for no gain. Generating at first order means the ID exists
exactly when there is finally something to correlate.

This is deliberately **not** a fourth signed token alongside the Identity
JWT (§3), the OAT (§4), and the confirmation code (§11) — it needs no
server-side signing or issuance at all. Its only job is to be a stable,
unguessable, opaque string the same browser sends back every time,
functionally identical to a shopping-cart cookie. The server validates it
by string equality against `orders.guest_device_id`; there is no
cryptographic property to check.

**Per-business, not global:** scoped to one `business_id`, mirroring
`customer-module-design.md` §8.2/§8.4's tenant isolation — a device
recognized at Grand Hotel must not let anyone correlate that device's
activity at City Apartments. A global device ID would quietly rebuild the
cross-business tracking the CRM module goes out of its way to avoid for
named accounts; it shouldn't come back through the anonymous door.

### It is "who made this order" — with a real caveat

`orders.guest_device_id` (§7) is, in effect, the answer to "who made this
order" for an anonymous order — the closest one that exists when there's
no name, phone, email, or account behind it. But it is **not** a
`user_id` under a different name:

| | `user_id` / `business_customer_id` | `guest_device_id` |
|---|---|---|
| Proven how | Password/OTP (Identity), or a captured phone/email | Nothing — the client just asserts it |
| Guaranteed present | Yes, once an account/profile exists | No — cleared storage, a private tab, or a client that never sends it → NULL |
| Forgeable/replayable | No (signed session, or verified contact) | Yes, in principle, if it ever leaked — a bare string, not cryptographically bound to anything |
| Safe to show staff as "the customer" | Yes | No — no name/contact behind it; never present it as an identity, only use it internally for correlation |

For the things this doc uses it for — order history, receipt opt-in
authorization, the claim below — treating it as the order's author is
exactly correct. It must never be promoted past that (into a receipt, a
refund decision, or anything staff-facing implying a verified customer).
That gap isn't a missing feature; it's the permanent consequence of §1.
A `business_customers` row remains the only thing in this system allowed
to mean "a specific person" — the GDI never becomes one, it only ever
feeds *into* creating one.

### What it enables before any account exists

```
GET /api/businesses/{businessId}/orders/by-device
  X-Guest-Device-Id: <guestDeviceId>
  → every order at this business carrying that device id
    (index: ix_orders_business_device, §7)

  Same trust model as the OAT: possession of a 122-bit random value is
  the credential. Unlike the confirmation code (§11), it isn't
  brute-forceable in practice, so it can safely return full order
  detail — the caller already knows what they ordered.
```

This is useful before anyone thinks about accounts: not just "your
orders tonight" surviving a page reload, but "your orders at this
business" surviving across visits — this week, last month — for as long
as `localStorage` holds the ID, built from something the guest never had
to type. This is what makes §2's "repeat-visit recognition" a genuine ✅.

### The claim step — auto-linked, by decision

```
POST /api/businesses/{businessId}/customers/claim-by-device
  Authorization: <Identity JWT, no business_id>   ← must be logged in
                                                      (or just registered);
                                                      same auth state as
                                                      /customers/me/*
  { guestDeviceId }

Server (CRM module — endpoint is under /customers/, and CRM does NOT own
the orders table, hence step 2 being an event rather than a write):
  1. AttachToBusinessAsync(businessId, userId)
       ← customer-module-design.md §6.2/§9, unchanged — resolves or
         creates the linked business_customers row for this user
  2. Publish GuestDeviceClaimedEvent
       { businessId, guestDeviceId, businessCustomerId }
  3. Return the resolved business_customers profile

Ordering module (subscribes to GuestDeviceClaimedEvent):
  UPDATE orders
     SET business_customer_id = event.businessCustomerId
   WHERE business_id      = event.businessId
     AND guest_device_id  = event.guestDeviceId
     AND business_customer_id IS NULL          ← first claimer wins per
     AND created_at > now() - interval '90 days'  order; already-linked
                                                  rows are never stolen
```

The `IS NULL` guard means claiming is **first-claimer-wins per order**:
on a shared device, whoever logs in first takes the orders placed up to
that point, and a later person takes only the ones placed after. That's
the shared-device trade-off working as intended, not an edge case to fix.

The 90-day window is the recency limit — an ID that sat in storage for a
year across a device resale is a much weaker signal than one from last
week. Exact window is a product call, not a technical constraint.

**Why the event split.** `onenex-backend-module-boundaries.md`'s "Module
Communication Rules" are explicit that a module may never write another
module's tables directly, only via a shared interface (sync) or a domain
event (async). CRM owns `business_customers`; the ordering module owns
`orders`. So CRM resolves identity and publishes; the ordering module
reacts on its own table. (§10's opt-in doesn't need an event because it
runs *inside* the ordering module already, and only reads CRM through
`ICustomerService` — the boundary is respected in both directions.)

No per-order selection, no confirmation prompt. This can fire
automatically the first time a logged-in session sees a stored
`guest_device_<businessId>` for a business it's interacting with (e.g.
piggybacked onto the normal `AttachToBusinessAsync` call), with no
separate "claim" UI at all. The `guest_device_id` column is **kept**
after claiming, not cleared — it's the audit trail explaining why those
orders are attached to that account.

### Accepted trade-off: unverified, by decision

D6's claim flow (`customer-module-design.md` §7) only auto-links against
**Identity-verified** phone/email. A device ID proves none of that — it
proves "the same browser sent this string before," which is also true if
a family shares one phone, a work laptop orders lunch for several
colleagues, or a device is resold without clearing storage. Auto-linking
on that signal is a **deliberate product decision, not an oversight**:
shared-device history bleeding into one account is accepted as low-stakes
(an extra order or two in someone's history) in exchange for
zero-friction recognition. It is explicitly weaker than D6's guarantee
and should never be described to a guest as "confirmed" the way a
phone/email claim is.

### How this differs from §10's receipt opt-in

| | Receipt opt-in (§10) | Device-based claim (this section) |
|---|---|---|
| When | Any time the OAT or device ID is still around | Any time later, once an account exists |
| Requires | Guest actively gives phone/email | Nothing — works even if the guest never typed anything |
| Trust level | Identity-verifiable (D6-eligible) | Unverified — auto-linked by decision, shared-device risk accepted |
| Creates | A `business_customers` row immediately | Nothing until the person logs in — then links everything found |
| Scope | The one order it was called on | Every unclaimed order from that device at that business |

Complementary, not competing: a guest who never opts into a receipt still
gets the device fallback, and it fires with no action on their part.

---

## 13. End-to-End Walkthrough

```
FIRST VISIT
  Guest scans QR  ─────────▶ GET /menu                    (no auth, no storage)
  Adds items, submits ─────▶ POST /orders                 (no auth)
                              ├─ orders row: business_customer_id NULL,
                              │  guest_device_id set, confirmation_code set
                              ├─ client generates + stores guest_device_<biz>
                              └─ returns { order, OAT, confirmationCode }
  Checks status   ─────────▶ GET /orders/{id}  (OAT)
  Orders again    ─────────▶ POST /orders      (same device id header)
                              └─ second order, its own OAT, same device id

  Optional, any time:
  Wants a receipt ─────────▶ POST /orders/{id}/receipt-optin  (OAT or device id)
                              ├─ LookupOrCreateAsync → business_customers
                              │  row (guest, user_id NULL)
                              └─ that order's business_customer_id set

  Lost the phone/storage:
  Has the code    ─────────▶ GET /orders/lookup?code=…&branchId=…  (status only)

RETURN VISIT (same device, still anonymous)
  Opens ordering page ─────▶ GET /orders/by-device  (device id)
                              └─ full history at this business — no login

LATER: REGISTERS OR LOGS IN
  Authenticated, device id still in storage:
  Client calls    ─────────▶ POST /customers/claim-by-device  (Identity JWT)
                              ├─ CRM: AttachToBusinessAsync → business_customers
                              │  row (linked, user_id set)
                              ├─ CRM publishes GuestDeviceClaimedEvent
                              └─ ordering module: every unclaimed order from
                                 that device (≤90d) gets business_customer_id

  From here: normal CRM customer. Order history visible via the account
  on any device, D6 claim flow applies to anything else they match.
```

---

## 14. Open Questions

- **QR/table code rotation** (§6): should the QR encode a
  business-rotated short code (defeating naive scripted abuse) alongside
  `business_id`/`branch_id`, or is rate-limiting alone enough for V1?
  Needs a call from whoever owns abuse/fraud posture.
- **Confirmation-code format** (§11): is a 6-digit numeric code (easy to
  read off a receipt printer) an acceptable brute-force surface given the
  other constraints (branch-scoped, status-only, rate-limited, expires
  with the order), or should it be longer/alphanumeric? Same owner as
  the question above.
- **Guest-safe cancel/modify** (inherited from D2): still unresolved. For
  a fully anonymous order it would have to work off the OAT or device id
  (no phone for an OTP link) — likely a short window right after
  placement, before kitchen prep starts. Not designed here.
- **Payment:** out of scope entirely — however it's collected
  (pay-at-counter, card-on-order), it doesn't depend on anything here.
  None of these three credentials is a payment credential.
- **Guest-device ID on native apps:** `localStorage` is the web story;
  an iOS/Android app would use its own keychain/preferences equivalent.
  Not specified here — same value, different storage primitive.

---

## Decision Log

| # | Decision | Status |
|---|---|---|
| GO1 | QR ordering captures **no contact info at all** — deliberately narrower than D2's original "guest capture (name+phone)" language, which still applies to booking | DECIDED (per user direction) |
| GO2 | No `business_customers` row at order time — `LookupOrCreateAsync` is not called on the ordering path (it *is* called by §10/§12) | DECIDED (follows from GO1) |
| GO3 | The **Order Access Token** is per-order and never session-spanning. The design as a whole *does* have a session-spanning credential — the Guest Device ID — which is a separate, weaker, unsigned thing | PROPOSED (corrects an earlier draft that claimed no session-spanning credential existed at all) |
| GO4 | No new table — three nullable columns on `orders` (`business_customer_id`, `guest_device_id`, `confirmation_code`) plus two indexes, §7 | PROPOSED |
| GO5 | Order creation is a fully open, unauthenticated endpoint and must carry the strictest rate-limiting in the system as a result | PROPOSED |
| GO6 | Multi-order billing consolidation at one table is table-number-based (POS/ordering concern), explicitly **not** device-id-based — one table is often several devices, and one device can move tables | PROPOSED |
| GO7 | Receipt opt-in (§10) is one of **two** CRM-value recovery paths (the other being §12's claim) — never a requirement to order. Authorized by OAT **or** device id, so it doesn't die with the OAT's 1–2h expiry | PROPOSED |
| GO8 | The confirmation code (§11) is the weakest credential: status-only, branch-scoped, expiring, read-only — never accepted for any write | PROPOSED |
| GO9 | Retroactive account-linking works via a client-generated, unsigned **Guest Device ID** (§12) — generated at first *order*, scoped per business, stored as a plain column, never an identity in its own right | PROPOSED |
| GO10 | Device-based claiming **auto-links** every unclaimed order from that device, no per-order confirmation — first-claimer-wins per order, 90-day recency window; shared-device bleed-through accepted | DECIDED (per user direction) |
| GO11 | QR code rotation, and confirmation-code length/format, remain open pending abuse/fraud-posture input (§14) | OPEN |
| GO12 | `claim-by-device` resolves identity in CRM, then publishes `GuestDeviceClaimedEvent` for the ordering module to apply to its own `orders` — CRM never writes `orders` directly, per `onenex-backend-module-boundaries.md` | DECIDED (fixes a boundary violation in an earlier draft) |
