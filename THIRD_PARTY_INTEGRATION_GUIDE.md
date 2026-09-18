# ZenSpace Third-Party Integration — Developer Guide

This guide is for external developers building a bridge between ZenSpace and
another calendar or booking system (Google Calendar, Outlook, Apple Calendar,
or a custom system).

It covers authentication, connection setup, the full booking API, webhook
delivery, error handling, and an end-to-end walkthrough. If you follow the
steps in order you will have a working integration.

> **Scope.** This guide is the contract between ZenSpace and your bridge.
> It does not cover how to build the bridge itself (polling Google, OAuth
> flows, etc.) — for that, see
> [OPERATING_MODE_THIRD_PARTY.md](./OPERATING_MODE_THIRD_PARTY.md).
>
> **Status.** Everything documented here is implemented today unless a
> section is explicitly labelled **Roadmap**.

---

## 1. Concepts

A minimal vocabulary you need before writing any code.

| Term | Meaning |
|---|---|
| **Organization** | A ZenSpace tenant (one customer company). You integrate against one organization at a time. |
| **Space group** | A logical grouping of meeting spaces (e.g. a building or floor). Carries timezone and address. |
| **Meeting space** | A reservable room. Each space has an `operating_mode`: `zenspace`, `hybrid`, or `third_party`. Your bridge integrates with spaces in `hybrid` or `third_party` mode (§6.8, §13.1). |
| **Operating mode `hybrid`** | ZenSpace **and** your platform both sell the room. It stays publicly bookable in the ZenSpace booking app, and your API bookings block that availability. The mode most partners want. |
| **Operating mode `third_party`** | The external calendar (your system) owns the booking lifecycle exclusively. ZenSpace mirrors it for reporting, access credentials, and device integrations, and removes the room from its own public surfaces. |
| **Connection** | A registered integration between your system and one meeting space, for one mode (e.g. "Google Calendar"). Has a lifecycle: `pending → approved → suspended`. |
| **API key** | The credential your bridge uses. Issued by an organization admin, scoped to the organization. Sent as `x-api-key` header. |
| **Booking** | A reservation on a meeting space for a time window. This is what your bridge pushes into ZenSpace on every calendar event. |
| **Physical-space mapping** | Optional link from a ZenSpace meeting space to a physical door/lock/wifi device. When present, ZenSpace can auto-generate access credentials at T-15 before a booking starts. |

**Rule of thumb.** A `third_party` space is invisible to ZenSpace's public
booking app — every booking on it must be pushed through the API by your bridge.
A `hybrid` space stays publicly bookable, so both sides sell the same inventory:
push your bookings promptly and re-check availability right before you commit.

---

## 2. Quick start (end-to-end, 5 minutes)

The fastest way to see the flow before reading the reference material.

```bash
# 1. Submit a connection request (use the API key the admin gave you)
curl -X POST https://api-spaceos.zenspace.io/api/v1/third-party/connection-request \
  -H "Content-Type: application/json" \
  -H "x-api-key: zsk_live_XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX" \
  -d '{
    "organization_id": "123e4567-e89b-12d3-a456-426614174000",
    "space_id": "123e4567-e89b-12d3-a456-426614174111",
    "third_party_mode": "Google Calendar",
    "detail": { "name": "Conference Room A", "capacity": 10, "timezone": "America/Los_Angeles" },
    "webhook_url": "https://your-bridge.example.com/webhooks/zenspace"
  }'
# → returns { data: { id: "<connection_id>", status: "pending", ... } }

# 2. Wait for the org admin to approve the connection in the ZenSpace admin UI.
#    When they do, ZenSpace POSTs to your webhook_url with status="approved".

# 3. Create a booking (once approved)
curl -X POST https://api-spaceos.zenspace.io/api/v1/bookings \
  -H "Content-Type: application/json" \
  -H "x-api-key: zsk_live_XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX" \
  -d '{
    "organization_id": "...",
    "space_id": "...",
    "space_group_id": "...",
    "title": "Team Sync",
    "start_time": "2026-04-20T14:00:00Z",
    "end_time":   "2026-04-20T15:00:00Z",
    "attendees": [],
    "booked_from": "cvent",
    "total_amount": 0,
    "deposit_amount": 0,
    "metadata": { "external_event_id": "google-event-abc123" }
  }'
# → returns { data: { id: "<booking_id>", status: "confirmed", payment_status: "paid", ... } }
```

That's it. The rest of this guide covers the details, edge cases, and
hardening.

---

## 3. Base URL and API conventions

### Base URL

| Environment | URL |
|---|---|
| Production | `https://api-spaceos.zenspace.io` |
| Dev / sandbox | Ask your ZenSpace contact. The contract is identical. |

All paths are prefixed with `/api/v1` (NestJS URI versioning).

> **Availability version note (changed since this guide's first release).** There is no
> longer an `/api/v2/` namespace. The availability surface that used to live on `v2` was
> **promoted to `/api/v1/`** and is now canonical; the older, buggy v1 handler was demoted
> to `/api/v0/` and is **retired**. If you integrated against `/api/v2/...`, drop the `2`
> and use `/api/v1/...` — the contract is otherwise unchanged. Never point a new
> integration at `/api/v0/`.

### Response envelope

Every response uses this shape:

```json
{
  "status": 200,
  "success": true,
  "message": "Human-readable summary",
  "data": { /* or [] for lists */ },
  "timestamp": "2026-04-13T12:00:00.000Z",
  "meta": { /* present on paginated responses */ }
}
```

On error `success` is `false` and `message` carries the reason. HTTP status
codes follow standard semantics (see §9).

### Datetime contract

- All times are **ISO-8601 in UTC** with a trailing `Z`, e.g.
  `2026-04-20T14:00:00Z`.
- Times with an offset (e.g. `…+02:00`) are accepted on input but will be
  converted to UTC before storage.
- Responses always return UTC.
- For display, use the meeting space's organization timezone (returned as
  `timezone` on organization/space responses). Conversion is the bridge's
  responsibility.

---

## 4. Authentication

### Getting an API key

API keys are issued by an **organization admin** via the ZenSpace admin
panel at **https://admin-spaceos.zenspace.io/**.

Ask the admin you are integrating with to:

1. Log in to the admin panel and pick the target organization.
2. Open **Settings → API Keys → New API Key**.
3. Fill the form:
   - **Name** — descriptive (e.g. "Google Calendar bridge — production").
   - **Key type** — `live` for production, `test` for sandbox.
   - **Permissions (scopes)** — at minimum `create:bookings`.
     For a typical calendar bridge: `create:bookings`, `read:bookings`,
     `update:bookings`, `read:meetingSpaces`, `read:spaceGroups`.
     ⚠️ **Read the scope-format warning below before filling this in.**
   - **IP allowlist** (optional) — restrict the key to your bridge's
     egress IPs.
   - **Expiry** (optional) — set a rotation deadline.
4. Click **Create**. The plain key value is shown **once** at this
   moment — the admin must copy it immediately.
5. Share the plain key with you **out-of-band** (encrypted channel,
   password manager, etc.). ZenSpace only stores the SHA-256 hash; the
   plain value is irrecoverable after this screen.

Keys look like `zsk_live_XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX` or
`zsk_test_...`.

> **Admin-panel alternative — API**: the admin can also create a key
> programmatically via `POST /api/v1/organizations/{organizationId}/api-keys`
> (JWT auth required). The response includes the plain key **once**.

#### ⚠️ Scope format — the most common way a partner key silently breaks

Scope strings are **`action:subject`, camelCase after the colon**:

| ✅ Correct | ❌ Wrong — does not exist |
|---|---|
| `create:bookings` | `bookings:write`, `write:bookings` |
| `read:bookings` | `bookings:read` |
| `read:meetingSpaces` | `meeting_spaces:read`, `read:meeting-spaces` |
| `read:spaceGroups` | `space_groups:read` |
| `read:thirdPartyConnections` | `third-party:read` |

The actions are **`create` / `read` / `update` / `delete`** — **there is no `write:*`**.
Earlier revisions of this guide recommended `bookings:write`, which is wrong in both
halves; if your key was created from that advice, it needs replacing.

**Why this bites so hard:** a key minted with a bad scope string **authenticates
fine** — and then returns **403 on every call, forever**, with nothing at creation
time hinting why. Permission matching is a literal string comparison with no
normalisation and no aliasing, so `read:meeting-spaces` and `read:meetingSpaces` are
simply different strings. Newer ZenSpace builds reject an unknown scope at key-creation
time with a 400 naming it and suggesting the closest match, but keys minted before that
are still out there failing silently.

> **Fastest unblock for a mis-scoped key:** an **empty `scopes[]`** (or omitting the
> field entirely) means *"inherit the organization's full permissions"* and is the
> documented way to issue an unrestricted key. That is **not** the same as a key with
> scopes — it is the escape hatch, not the default. Prefer explicit scopes for
> production keys.

### Sending the API key

Send the key on every request via the `x-api-key` header. The following
forms are accepted, in order of preference:

1. `x-api-key: zsk_live_...` — preferred
2. `x-org-api-key: zsk_live_...`
3. `Authorization: Bearer zsk_live_...`

```http
POST /api/v1/bookings HTTP/1.1
Host: api-spaceos.zenspace.io
Content-Type: application/json
x-api-key: zsk_live_XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
```

### Key rotation

Rotate the key periodically (90 days is a reasonable default). To rotate:

1. Ask the admin to issue a new key.
2. Deploy the new key to your bridge.
3. Verify traffic succeeds.
4. Ask the admin to revoke the old key.

Keys can be revoked individually without disrupting other integrations.

### Auth failure modes

| Condition | HTTP | `message` |
|---|---|---|
| Key not found / hash mismatch | 401 | `Invalid or expired API key` |
| Key revoked (`is_active=false`) | 401 | `Invalid or expired API key` |
| Key expired (`expires_at` past) | 401 | `Invalid or expired API key` |
| Request IP not in `allowed_ips` | 401 | `Invalid or expired API key` |
| No key sent on a key-required route | 401 | route-specific |

> `POST /api/v1/third-party/connection-request` accepts the key as **optional
> but recommended**. All booking endpoints treat the key as
> optional-but-recommended too — without it you lose auditability and
> IP-whitelist protection. **Always send the key in production.**

---

## 5. Connecting to a meeting space

Before you can push bookings to a meeting space, you must have an
**approved connection** for that space in `third_party` mode. This is a
one-time setup per space.

### 5.1 Step-by-step

1. **Org admin creates the meeting space** (or picks an existing one) and
   sets its `operating_mode` to `third_party`. This is done inside
   ZenSpace — you do not do this.

2. **Org admin issues an API key** for your bridge. They share the plain
   key with you out-of-band.

3. **Your bridge POSTs a connection request** (see 5.2). This creates a
   connection in `pending` state.

4. **The admin approves the connection** in the ZenSpace admin UI. ZenSpace
   POSTs a notification to your `webhook_url` when this happens.

5. **Your bridge starts pushing bookings** once the approval webhook
   arrives.

### 5.2 `POST /api/v1/third-party/connection-request`

The only endpoint external integrators call to register. All other
connection-management endpoints (`approve`, `decline`, `suspend`, list,
update, delete) are admin-only.

```http
POST /api/v1/third-party/connection-request
Content-Type: application/json
x-api-key: zsk_live_XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
```

```json
{
  "organization_id": "123e4567-e89b-12d3-a456-426614174000",
  "space_id": "123e4567-e89b-12d3-a456-426614174111",
  "third_party_mode": "Google Calendar",
  "detail": {
    "name": "Conference Room A",
    "capacity": 10,
    "timezone": "America/Los_Angeles",
    "address": "123 Market St, San Francisco, CA",
    "pricing": { "base_price": 25.00 },
    "business_hours": {
      "monday":    { "open": "09:00", "close": "18:00" },
      "tuesday":   { "open": "09:00", "close": "18:00" },
      "wednesday": { "open": "09:00", "close": "18:00" },
      "thursday":  { "open": "09:00", "close": "18:00" },
      "friday":    { "open": "09:00", "close": "17:00" },
      "saturday":  null,
      "sunday":    null
    }
  },
  "webhook_url": "https://your-bridge.example.com/webhooks/zenspace"
}
```

#### Field reference

| Field | Type | Required | Notes |
|---|---|---|---|
| `organization_id` | UUID | yes | The organization that owns the space. |
| `space_id` | UUID | yes | The target meeting space. Must belong to the organization, must already have a space group assigned. |
| `third_party_mode` | enum | yes | One of: `Google Calendar`, `Outlook Calendar`, `Microsoft Outlook`, `Apple Calendar`, `Other`. Literal strings with spaces. |
| `detail` | object | yes | Snapshot of your view of the space. See below. |
| `detail.name` | string | no | Displayed if the ZenSpace space has no name. |
| `detail.capacity` | number | no | Seating capacity. |
| `detail.timezone` | IANA string | no | e.g. `America/Los_Angeles`. |
| `detail.address` | string | no | Street address. |
| `detail.pricing.base_price` | number | no | Only `base_price` is read during sync. |
| `detail.business_hours` | object | no | Map of weekday → `{ open, close }` or `null`. Day names are case-insensitive. Time format `HH:MM` or `HH:MM:SS`. |
| `webhook_url` | URL | strongly recommended | Where ZenSpace POSTs status-change and booking-event notifications. **Without this you receive nothing.** |
| `api_key_id` | UUID | no | Auto-derived from your `x-api-key` header. Normally omit. |

#### Successful response

```http
HTTP/1.1 201 Created
Content-Type: application/json
```

```json
{
  "status": 201,
  "success": true,
  "message": "Connection request created successfully. Waiting for approval.",
  "data": {
    "id": "c7e8f9a0-1234-5678-9abc-def012345678",
    "organization_id": "123e4567-e89b-12d3-a456-426614174000",
    "space_group_id": "...",
    "meeting_space_id": "123e4567-e89b-12d3-a456-426614174111",
    "third_party_mode": "Google Calendar",
    "detail": { "...": "..." },
    "status": "pending",
    "webhook_url": "https://your-bridge.example.com/webhooks/zenspace",
    "api_key_id": "...",
    "created_at": "2026-04-13T12:00:00.000Z",
    "updated_at": "2026-04-13T12:00:00.000Z"
  }
}
```

**Persist `data.id` — that's your `connection_id`, which appears in every
webhook payload ZenSpace sends you.**

### 5.3 Approval behavior

When the admin approves your request, ZenSpace does a **one-way, non-destructive sync**
from your `detail` payload onto the meeting space. Only empty fields on the
ZenSpace side are populated, to avoid overwriting admin-configured data:

| `detail` field | Synced when the ZenSpace side is... |
|---|---|
| `name` | null or empty |
| `capacity` | null/undefined/0 |
| `pricing.base_price` | null or undefined |
| `business_hours` | no existing rows |

The full `detail` object is always stored under
`meeting_space.metadata.third_party_detail` for reference.

**Send your best data up front.** You do not get a second chance to
overwrite unless the admin manually clears the field first.

#### Business hours — precedence with admin-side configuration

Admins can configure business hours on the space in two other ways:

1. **Inline on the meeting-space API** — `POST /api/v1/meeting-spaces` or
   `PUT /api/v1/meeting-spaces/:id` now accepts a top-level `business_hours`
   section (day-keyed, same shape as your `detail.business_hours`). REPLACE
   semantics — writes fresh rows and soft-deletes any existing ones.
2. **Per-row via `POST /api/v1/business-hours`** — one entry at a time, supports
   scheduled `effective_from` / `effective_until` for future changes.

Interaction with your `detail.business_hours` on connection approval:

- If the admin has already configured hours via (1) or (2) **before** you
  submit the connection request — or between your request and the approval —
  `detail.business_hours` is **not applied** at approval time. Your payload
  is still stored under `meeting_space.metadata.third_party_detail`, but the
  admin must reconcile manually if they want your schedule instead.
- If the space has **no** existing rows at approval time, `detail.business_hours`
  is applied normally. Same day-keyed format, `null` for closed, omitted day
  for "inherit from group".
- After approval, the admin can **overwrite** the synced rows at any time via
  method (1) (REPLACE) or adjust individual days via method (2). You have no
  way to push fresh hours back through the connection — submit a new request
  if you need to reset.

**Practical rule of thumb.** If the external calendar is the source of
truth for the schedule, include `detail.business_hours` and ask the admin
not to configure BH before approving. If the admin already has a schedule
in mind, omit `detail.business_hours` and let them own it.

See [docs/MEETING_SPACE_OPERATING_MODES.md §6.1](./MEETING_SPACE_OPERATING_MODES.md)
for the full precedence matrix and admin-side walkthrough.

### 5.4 Connection lifecycle

```
                 ┌─────────────┐
                 │   pending   │  <- POST /third-party/connection-request
                 └──────┬──────┘
            approve     │     decline
        ┌───────────────┴───────────────┐
        ▼                               ▼
  ┌───────────┐                   ┌───────────┐
  │ approved  │                   │ declined  │  (terminal)
  └─────┬─────┘                   └───────────┘
        │ suspend
        ▼
  ┌───────────┐
  │ suspended │
  └───────────┘
```

- `pending` → `approved`: booking events start flowing. Metadata sync runs.
- `pending` → `declined`: terminal. To try again, submit a new request.
- `approved` → `suspended`: booking events stop. Admin can re-approve later.
- `declined` bookings cannot be reversed.
- Only **one approved connection** per `(space_id, third_party_mode)` pair.

---

## 6. Booking API

All endpoints accept the API key. `organization_id`, `space_id`, and
`space_group_id` are always required on create.

### 6.1 Create a booking — `POST /api/v1/bookings`

Use this on every new event in your calendar.

```http
POST /api/v1/bookings
Content-Type: application/json
x-api-key: zsk_live_...
```

```json
{
  "organization_id": "123e4567-...",
  "space_id": "123e4567-...",
  "space_group_id": "123e4567-...",
  "title": "Team Sync",
  "description": "Weekly engineering sync",
  "start_time": "2026-04-20T14:00:00Z",
  "end_time":   "2026-04-20T15:00:00Z",
  "attendees": [
    { "name": "Alice", "email": "alice@example.com" },
    { "name": "Bob",   "email": "bob@example.com" }
  ],
  "total_amount": 0,
  "deposit_amount": 0,
  "organizer_email": "alice@example.com",
  "organizer_name": "Alice Smith",
  "organizer_phone": "+14155551212",
  "organizer_company": "Acme Corp",
  "booking_source": "external",
  "metadata": {
    "external_event_id": "google-event-abc123"
  }
}
```

#### Field reference (abridged — see Swagger for the full DTO)

| Field | Type | Required | Notes |
|---|---|---|---|
| `organization_id` | UUID | yes | |
| `space_id` | UUID | yes | Must be in `third_party` mode with an approved connection. |
| `space_group_id` | UUID | yes | Must match the space's group. |
| `title` | string | yes | Max 255 chars. |
| `description` | string | no | Free-form. |
| `start_time` | ISO-8601 UTC | yes | |
| `end_time` | ISO-8601 UTC | yes | Must be after `start_time`. |
| `attendees` | array | yes | Pass `[]` if none. Each entry is free-form JSON; `{ name, email }` is the convention. |
| `total_amount` | number ≥ 0 | yes | Pass `0` if payment is handled outside ZenSpace (typical for `third_party`). |
| `deposit_amount` | number ≥ 0 | yes | Pass `0` for `third_party`. |
| `organizer_email` | email | **optional in `third_party` mode**; required in `zenspace`/`hybrid` | When omitted for `third_party`, ZenSpace sends no organizer emails. Always required if a voucher is applied. |
| `organizer_name` | string | no | |
| `organizer_phone` | string | no | |
| `organizer_company` | string | no | |
| `booking_source` | enum | no | Recommended: `external`. Values: `web`, `mobile`, `api`, `admin`, `walk_in`, `external`. |
| `user_id` | UUID | no | Only if the organizer is a registered ZenSpace user. Leave null for external users. |
| `metadata.external_event_id` | string | strongly recommended | Your own event ID (e.g. Google event ID). Used for dedupe — see §8. |
| `voucher_id` / `voucher_code` | UUID / string | no | If provided, `organizer_email` is required. |
| `is_magic_link` | boolean | no (default `false`) | When `true`, ZenSpace synchronously issues a booking-access magic link during create and includes it in the response. Requires `x-api-key` on the request and an active physical-space mapping on the space. **Fail-closed**: if the link cannot be issued, the booking is NOT created. See §6.1.1. |

#### Successful response

```http
HTTP/1.1 201 Created
```

```json
{
  "status": 201,
  "success": true,
  "message": "Booking created successfully",
  "data": {
    "id": "b1234567-89ab-cdef-0123-456789abcdef",
    "organization_id": "...",
    "space_id": "...",
    "space_group_id": "...",
    "title": "Team Sync",
    "start_time": "2026-04-20T14:00:00.000Z",
    "end_time":   "2026-04-20T15:00:00.000Z",
    "status": "confirmed",
    "payment_status": "paid",
    "booking_source": "external",
    "organizer_email": "alice@example.com",
    "metadata": { "external_event_id": "google-event-abc123" },
    "created_at": "2026-04-13T12:00:00.000Z",
    "updated_at": "2026-04-13T12:00:00.000Z"
  }
}
```

#### Key behaviors for `third_party` mode

- **Booking is immediately `confirmed` and `paid`.** No Stripe payment
  intent is created. Rationale: payment is handled by your system.
- **Organizer emails are opt-in per meeting space** via the
  `email_notification` flag on the space (defaults to `false`). When
  `false` (default), ZenSpace sends no confirmation, cancellation, or
  booking-access emails — your system owns communication. When an admin
  sets it to `true`, emails are sent to `organizer_email` if present.
- **Scheduler forwarding and booking-access credential generation proceed
  normally.** Credentials (magic link / lock PIN / wifi voucher) are
  always generated at T-15 minutes if a physical-space mapping exists;
  only the organizer-email delivery is gated by `email_notification`.
- **No webhook is fired back to you** for this creation (it came from you).
  See §7 for when you *do* receive booking webhooks.

#### Precondition errors

| Condition | HTTP | `message` |
|---|---|---|
| `third_party` space, no connection exists | 400 | `Cannot create booking: No third-party connection found...` |
| `third_party` space, no approved connection | 400 | `Cannot create booking: No approved third-party connection found...` |
| `third_party` space, any pending/suspended connection exists (even alongside an approved one) | 400 | `Cannot create booking: Third-party connection(s) are not in approved state...` |
| `space_id` missing / invalid | 400 | varies |
| Time overlap with another non-cancelled booking | 400 | `Time slot already booked` |
| Voucher applied without `organizer_email` | 400 | `organizer_email is required when applying a voucher` |
| `organizer_email` missing on a non-`third_party` space | 400 | `organizer_email is required` |

**Connection requirements differ by mode.** On a **`third_party`** space an approved
connection is mandatory and *every* connection on the space must be approved. On a
**`hybrid`** space a connection is **not required at all** — and if connections do
exist in a non-approved state, an API-key request is still allowed through (only
browser/JWT callers are blocked). So a `hybrid` partner integration can start booking
as soon as the space is in the right mode and you hold a valid key.

### 6.1.1 Requesting the magic link at booking-create time — `is_magic_link`

Set `is_magic_link: true` on the create payload when you want the
booking-access magic link back in the HTTP response, so your bridge can
embed it into the calendar event description at the moment of creation
(not at T-15 when the organizer email / `booking.credentials_ready`
webhook fires — see §7.5 for that later-stage delivery).

**Preconditions:**

| Requirement | Why |
|---|---|
| `x-api-key` header on the request | The magic link URL is in the response body; API key gates exposure and throttles the synchronous ZenEdge call that gets triggered per invocation. |
| The meeting space has an active physical-space mapping covering the booking window | Without a mapping, ZenSpace's edge infrastructure cannot issue a magic link. |
| Endpoint must be `POST /api/v1/bookings` (not `/bulk`) | See below. |

**Fail-closed semantics.** If the link cannot be issued for any reason,
the booking is **not created**. You get a 4xx / 5xx response with a
structured body telling you how to recover. There is no "booking exists
but magic_link is null" state — it's either (201 + link) or (error + no
booking).

**Request example:**

```http
POST /api/v1/bookings
Content-Type: application/json
x-api-key: zsk_live_...
```

```json
{
  "organization_id": "123e4567-...",
  "space_id": "123e4567-...",
  "space_group_id": "123e4567-...",
  "title": "Team Sync",
  "start_time": "2026-04-20T14:00:00Z",
  "end_time":   "2026-04-20T15:00:00Z",
  "attendees": [],
  "total_amount": 0,
  "deposit_amount": 0,
  "organizer_email": "alice@example.com",
  "metadata": { "external_event_id": "google-event-abc123" },
  "is_magic_link": true
}
```

**Successful response:**

```http
HTTP/1.1 201 Created
```

```json
{
  "status": 201,
  "success": true,
  "message": "Booking created successfully",
  "data": {
    "id": "b1234567-89ab-cdef-0123-456789abcdef",
    "organization_id": "...",
    "space_id": "...",
    "title": "Team Sync",
    "start_time": "2026-04-20T14:00:00.000Z",
    "end_time":   "2026-04-20T15:00:00.000Z",
    "status": "confirmed",
    "payment_status": "paid",
    "...": "...other standard booking fields...",

    "magic_link": {
      "url": "https://edge.zenspace.io/ma/abc123..."
    },
    "booking_access_credentials": {
      "status": "success",
      "request_id": "zenedge_req_abc123",
      "overall_status": "complete_success",
      "credentials": {
        "lock": { "template_html": "<div>Unlock code: 1234...</div>" },
        "wifi": { "template_html": "<div>WiFi: zenspace-guest ...</div>" }
      }
    }
  }
}
```

- `magic_link.url` — always present on 201 responses when the flag was set.
  Treat as secret; it's time-bound to the booking window.
- `booking_access_credentials.credentials.lock` / `.wifi` — either of
  these may be `null` if the space doesn't have that credential type
  configured. A partial success where `magic_link.url` is populated but
  only one of lock/wifi is returned still counts as success.

**Failure response.** When the link cannot be issued, you get a non-2xx
response with a structured body:

```json
{
  "status": 422,
  "success": false,
  "error_code": "MAGIC_LINK_NO_MAPPING",
  "category": "permanent",
  "retry_guidance": "retry_without_flag",
  "message": "Cannot issue magic link: no active physical-space mapping exists for this meeting space. Retry without is_magic_link to create the booking without credentials, or ask an admin to add a mapping.",
  "request_id": null
}
```

Switch on `error_code` (not `message`, which may reword). Each code has
a stable `category` and `retry_guidance` that together tell you what to
do next:

| `error_code` | HTTP | `category` | `retry_guidance` | What to do |
|---|---|---|---|---|
| `MAGIC_LINK_NO_MAPPING` | 422 | permanent | `retry_without_flag` | Retry the same payload without `is_magic_link`. Admin needs to configure a mapping. |
| `MAGIC_LINK_ZENEDGE_BUSINESS_FAILURE` | 422 | permanent | `retry_without_flag` | ZenSpace's edge said this booking can't get a link. Retry without the flag if you need the booking anyway. |
| `MAGIC_LINK_ZENEDGE_TIMEOUT` | 504 | transient | `retry_later` | Back off and resend the same payload (including `is_magic_link`) after ~60 s. |
| `MAGIC_LINK_ZENEDGE_TRANSPORT_FAILURE` | 502 | transient | `retry_later` | Same as timeout. |
| `MAGIC_LINK_AUTH_KEY_MISSING` | 500 | permanent | `abort` | Server-side misconfiguration. Alert a human; do not retry. |

**Recommended integrator flow (pseudo-code):**

```
response = POST /api/v1/bookings  payload_with_is_magic_link=true

if response.status == 201:
    embed response.data.magic_link.url into calendar event
    done
else if response.body.error_code starts with "MAGIC_LINK_":
    switch response.body.retry_guidance:
      "retry_without_flag":
        // permanent — create booking plainly (no link)
        response = POST /api/v1/bookings  same_payload_without_is_magic_link
        // proceed; calendar event has no link, that's OK
      "retry_later":
        // transient — enqueue with backoff
        enqueue(payload, delay=60s)
      "abort":
        alert_admin(response.body.error_code)
else:
    // other error (validation, auth, etc.) — handle per §9
```

**Other behaviors:**

- **Does not mark `credentials_requested`.** The T-15 / T-0 flow still
  fires, so your registered webhook (see §7.5) also receives a
  `booking.credentials_ready` event near booking start. That's by design
  — dedupe on your side using the `x-idempotency-key` header.
- **No retry on the synchronous call.** One ZenEdge attempt with a 5 s
  timeout. The existing T-15 / T-0 retry loop (separate from this path)
  handles retries once the booking is live.
- **Update (`PUT`) ignores the flag.** `is_magic_link` on an update is
  silently dropped. The link from create remains valid for the booking
  window; rescheduled bookings rely on the T-15 webhook for re-delivery.
- **Bulk rejects the flag.** `POST /api/v1/bookings/bulk` returns **400**
  if any item has `is_magic_link: true` — the per-item endpoint is the
  only supported channel.

### 6.2 Update a booking — `PUT /api/v1/bookings/{id}`

```http
PUT /api/v1/bookings/b1234567-89ab-cdef-0123-456789abcdef
Content-Type: application/json
x-api-key: zsk_live_...
```

```json
{
  "start_time": "2026-04-20T15:00:00Z",
  "end_time":   "2026-04-20T16:30:00Z",
  "title": "Team Sync (rescheduled)"
}
```

- Only send fields you want to change.
- Time changes are validated against overlaps and business hours the same
  way as on create.
- Do **not** use update to cancel — use `POST /:id/cancel`.

Response shape matches create (200 OK with the updated booking).

### 6.3 Cancel a booking — `POST /api/v1/bookings/{id}/cancel`

```http
POST /api/v1/bookings/b1234567-89ab-cdef-0123-456789abcdef/cancel
Content-Type: application/json
x-api-key: zsk_live_...
```

```json
{
  "reason": "Cancelled in Google Calendar"
}
```

- Idempotent against already-cancelled bookings: returns 400
  `Booking is already cancelled` if cancelled again.
- Scheduler forwarding is cleaned up automatically.
- Organizer cancellation email is sent if `organizer_email` is set.
- **Refunds:** `third_party` bookings have no ZenSpace payment, so
  calling ZenSpace's refund endpoints will return
  `Cannot refund: booking has no ZenSpace payment record`. Process refunds
  in the originating calendar/payment system.

### 6.4 Get a single booking — `GET /api/v1/bookings/{id}`

```http
GET /api/v1/bookings/b1234567-89ab-cdef-0123-456789abcdef
x-api-key: zsk_live_...
```

Returns the full booking object. Useful for reconciling state after a
webhook or checking whether a booking you created still exists.

### 6.5 Bulk create — `POST /api/v1/bookings/bulk`

For importing an existing calendar during initial sync. Accepts an array
of bookings and processes them sequentially.

```json
{
  "bookings": [
    { "organization_id": "...", "space_id": "...", /* ... */ },
    { "organization_id": "...", "space_id": "...", /* ... */ }
  ]
}
```

Response is a summary:

```json
{
  "data": {
    "summary": { "total": 2, "successful": 2, "failed": 0 },
    "successful": [{ "id": "..." }, { "id": "..." }],
    "failed": []
  }
}
```

**Use sparingly.** Failures per-row don't stop the batch; inspect the
`failed` array.

**Restriction:** `is_magic_link: true` is **not** supported on bulk.
If any item sets it, the whole request is rejected with **400**. Use
the per-item endpoint `POST /api/v1/bookings` for bookings that need a
synchronous magic link. See §6.1.1.

### 6.6 List bookings — `GET /api/v1/bookings`

> ⚠ This endpoint requires **JWT auth** (`read:bookings` permission), not
> an API key. Your bridge cannot call it directly. Reconciliation from the
> bridge side is done via stored `(external_event_id, zenspace_booking_id)`
> mappings and `GET /api/v1/bookings/{id}`.

A bridge-accessible `GET /api/v1/third-party/{connection_id}/bookings` is on
the roadmap (see §12).

### 6.7 Reading availability

If your bridge wants to check a space's availability (to pre-validate
before pushing a booking, or to surface open slots in your UI),
**always use the canonical `/api/v1/` availability routes**. These are the
routes that were previously published as `v2`; they were promoted to `v1`
unchanged. The older handler — the one with known correctness bugs around
per-day business hours — was demoted to `/api/v0/` and is now **retired**.

#### `GET /api/v1/meeting-spaces/{space_id}/availability`

The single endpoint you'll usually care about. `third_party` spaces are
**filtered out** of group-level availability endpoints (by design, since
they're not publicly listed), so direct per-space queries are the right
tool.

Two modes (controlled by query params):

- **Single day** — `?date=YYYY-MM-DD` or `?date=<full-ISO>` — returns
  one `AvailabilityResponseDto`.
- **Date range** — `?start_date=YYYY-MM-DD&end_date=YYYY-MM-DD` (max
  90 days) — returns `CalendarAvailabilityResponseDto` with a per-date
  map plus a summary.

```http
GET /api/v1/meeting-spaces/b1234567-.../availability?date=2026-04-22&include_time_slots=true HTTP/1.1
Host: api-spaceos.zenspace.io
x-api-key: zsk_live_...
```

Public endpoint (no auth required), but sending your API key is
recommended for audit trail.

#### What you get back

At minimum, for each day:

- `business_hours` — `{ open_time, close_time, is_closed }` in the
  organization's local time.
- `time_slots[]` — 15-minute slots with `status: "available" | "booked"
  | "unavailable"` (only populated when `include_time_slots=true`).
- `bookings[]` / `unavailability[]` — raw overlap lists (only when
  `include_time_slots` is omitted or false).
- `next_available_time`, `booked_until_end_time`,
  `available_until_end_time` — for "currently free?" UIs.

Full response shape + query-param table + scenarios are in the frontend
migration guide: [availability-v2-frontend-guide.md](./availability-v2-frontend-guide.md).

#### Why not v1?

v1's single-day business-hours lookup has a `NULLS FIRST` SQL ordering
bug that causes group-level business-hours rows to beat per-space
overrides — the opposite of the intended precedence. Per-day BH
(different hours each day of week) was silently wrong. v2 resolves
this in application code with explicit precedence (per-space wins, then
most recent `effective_from`). Same response DTOs in both versions —
dropping the `/v2/` prefix is the only code change you'd need to make.

#### Typical pre-booking check pattern

```
GET /api/v1/meeting-spaces/{id}/availability?date=<booking_date>
     → read business_hours and see if request window fits
     → read bookings[] + unavailability[] to detect overlap
if OK:
  POST /api/v1/bookings
else:
  skip / alert / reschedule
```

This is optional — `POST /api/v1/bookings` will itself reject an invalid
request with a 400 and an actionable `message`. The pre-check is just a
latency / UX optimisation for bridges that want to surface conflicts in
their own UI without round-tripping the booking call.

---

### 6.8 Platform bookings — `booked_from` (you collected the money)

If the guest **paid on your platform** rather than through ZenSpace, say so with the
top-level **`booked_from`** field. Without it, ZenSpace assumes it is responsible for
collecting payment — and that assumption silently loses you the room (see the warning
below).

```jsonc
POST /api/v1/bookings
{
  "meeting_space_id": "…",
  "start_time": "2026-09-10T14:00:00.000Z",
  "end_time":   "2026-09-10T15:00:00.000Z",
  "organizer_email": "jane@example.com",
  "booked_from": "cvent",         // ← your platform, lowercase, free-form
  "total_amount": 45.00           // what the guest actually paid YOU (see below)
}
```

`booked_from` is a free-form string (no enum), so onboarding a new platform needs no
change on our side. It defaults to `zenspace_booking` for anything booked through
ZenSpace's own surfaces, and you can filter on it later:
`GET /api/v1/bookings?booked_from=cvent`.

**It is distinct from `booking_source`** (`web` / `mobile` / `api` / … — the *kind* of
client; every partner booking is `external`, so it cannot tell Cvent from any other partner)
and from `external_calendar_id` (the specific mirrored event, §8).

#### Prerequisite — the space must be in `hybrid` or `third_party` mode

`booked_from` only takes effect on a space whose `operating_mode` is **`hybrid`** or
**`third_party`**. On a `zenspace` space the field is recorded for reporting and
nothing else changes: ZenSpace still creates a Stripe intent, leaves the booking
`PENDING`, and the pending-payment sweep cancels it. Ask the organization admin to
switch the space before you go live.

| Mode | Who owns the booking lifecycle | Visible in the ZenSpace booking app? | Partner bookings via API | Organizer emails |
|---|---|---|---|---|
| `zenspace` | ZenSpace | Yes — publicly bookable | Not eligible for `booked_from` handling | Always sent |
| `hybrid` | **Both** — ZenSpace *and* your platform sell the same room | Yes — publicly bookable | Yes | Always sent |
| `third_party` | Your system only | **No** — excluded from all public availability and browse surfaces | Yes | Opt-in per space via `email_notification` (defaults `false`) |

**Which one do you want?**

- **`hybrid` is what most partners want.** The room keeps selling through ZenSpace's
  own booking app *and* through your platform. Availability stays in sync in both
  directions: a ZenSpace booking blocks your slot, your booking blocks theirs.
  ⚠️ Because both sides sell the same inventory, push every booking promptly and
  read availability immediately before you commit (§6.7) — the gap between your
  availability check and your create is the window where a double-booking can happen.
- **`third_party`** removes the room from ZenSpace's public surfaces entirely, so your
  platform is the only way to book it. Pick this only when the customer genuinely wants
  the room sold exclusively through you. It also flips organizer emails to opt-in, on
  the assumption that your system owns guest communication.
- Both modes give identical `booked_from` behaviour — auto-confirm, no Stripe, no echo.
  The difference is purely who else can sell the room.

**How to switch a space.** The org admin can do it in the ZenSpace admin UI, or via the
API with a key holding `update:meetingSpaces`:

```bash
curl -X PUT "$BASE_URL/api/v1/meeting-spaces/<space_id>" \
  -H "x-api-key: $ZENSPACE_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{ "operating_settings": { "operating_mode": "hybrid" } }'
```

The mode lives under the nested **`operating_settings`** object — a top-level
`operating_mode` is rejected by the validation pipe. Set `"third_party"` instead for
exclusive distribution; on that mode you can also send
`"email_notification": true` in the same object if you want ZenSpace to keep emailing
the organizer. Verify with `GET /api/v1/meeting-spaces/<space_id>` before your first
booking — this is the single most common reason a partner booking comes back `pending`.

#### What it changes — and the three conditions

The special handling applies only when **all three** hold:

1. The request is authenticated with an **API key** (`POST /bookings` is otherwise
   public, so a caller-supplied field alone could not be trusted to skip price checks).
2. The meeting space's `operating_mode` is **`third_party` or `hybrid`**.
3. `booked_from` is **explicitly set** to something other than the default.

When all three hold, ZenSpace:

- **skips price and deposit validation** — you collected the money, so our pricing
  rules are not the authority on what was charged, and your submitted amounts are
  preserved rather than overwritten;
- **creates no Stripe payment intent**;
- writes the booking straight to **`CONFIRMED` + `PAID`**, which still fires scheduler
  forwarding, state transitions and booking-access credentials.

If any condition fails, the value is recorded for reporting and **nothing else
changes** — so adding `booked_from` to an existing integration is safe to roll out
incrementally.

> ⚠️ **Send it, or you may lose the room.** Without `booked_from`, a partner booking is
> created `PENDING` awaiting a ZenSpace payment that will never arrive — and the
> pending-payment sweep cancels it a few minutes later, silently freeing a room your
> platform still shows as sold. The auto-confirm is the point of this field, not a
> convenience.

> ⚠️ **`hybrid` is the mode most partners want**, not `third_party` — see the mode
> comparison above. Historically several platform-booking behaviours were wired to
> `third_party` only; they now handle `hybrid` too, so a `hybrid` space gets exactly
> the same auto-confirm, no-Stripe, no-echo treatment.

#### What amount to send

Whatever you put in `total_amount` is stored as-is and **counts toward the
organization's revenue reporting**, even though ZenSpace never collected it. Post the
real figure if you want channel revenue attributed, or `0` if you want the booking
counted for occupancy only. Pick one convention and keep it — mixing them makes the
org's revenue numbers unreadable.

#### Echo suppression

Platform bookings are excluded from third-party outbound notifications, so ZenSpace
will not echo your own create back to you. Before `booked_from` existed there was no
way to tell an inbound platform booking from a native one on a `hybrid` space, and the
echo made receiving systems mirror an event they already owned.

---

## 7. Webhooks — receiving events from ZenSpace

ZenSpace has **two independent delivery channels** to your bridge. Pick
based on what events you need:

| Channel | Registered via | Events delivered | Retries | Delivery logs |
|---|---|---|---|---|
| **Connection `webhook_url`** | `webhook_url` field on the connection request (§5.2) | Connection lifecycle (§7.3), admin-originated booking changes (§7.4), `booking.credentials_ready` (§7.5) | None for §7.3/§7.4. 3 attempts w/ backoff for §7.5. | None |
| **Generic webhook subscription** | `POST /api/v1/webhooks` (§7.6) | Space lifecycle events today; broader event catalog on the roadmap | Configurable (0–10 attempts) | `GET /webhooks/{id}/deliveries` |

The connection's `webhook_url` is the only way to receive connection-
status and credentials events. The generic subscription is the only way
to receive `space.*` events with retries and delivery logs. Most bridges
end up using both.

### 7.1 Transport semantics

Read this before you build a handler. The current implementation is
intentionally minimal.

- **Method:** `POST`
- **Content-Type:** `application/json`
- **Signature header:** ❌ none. No HMAC, no shared secret. Anyone who
  learns your webhook URL could impersonate ZenSpace. Treat the URL as a
  secret.
- **Auth header:** ❌ none.
- **Retry policy:** ❌ none. Non-2xx responses or timeouts are logged and
  dropped.
- **Delivery log surfaced to you:** ❌ none.
- **Ordering:** not guaranteed.

### 7.2 Best practices on your side

1. **Treat the webhook URL as a secret.** Use a long, unguessable path.
2. **Respond 2xx quickly** (under ~2 seconds). Do heavy work async.
3. **Be idempotent.** Key on `connection_id + event + booking_id` (or
   `status` for connection events).
4. **Log everything.** You have no retry; if you miss an event, you need
   to reconcile against `GET /api/v1/bookings/{id}` manually.
5. **Verify the payload shape.** Don't trust fields to be present.

### 7.3 Connection-status webhooks

Fired when the admin approves, declines, or suspends your connection.

**Approved:**

```json
{
  "status": "approved",
  "connection_id": "c7e8f9a0-1234-5678-9abc-def012345678",
  "meeting_space_id": "123e4567-...",
  "zenspace_webhook": "https://api-spaceos.zenspace.io/api/v1/third-party/webhook/c7e8f9a0-..."
}
```

> **Note.** The `zenspace_webhook` URL is reserved for a future release
> and **currently returns 404**. Treat it as informational.

**Declined:**

```json
{ "status": "declined", "reason": "Room not available for external booking" }
```

**Suspended:**

```json
{ "status": "suspended", "reason": "Connection suspended by administrator" }
```

`reason` on suspend defaults to `Connection suspended by administrator`
if the admin didn't supply one.

### 7.4 Booking-event webhooks

Once your connection is `approved`, ZenSpace POSTs a notification when an
admin modifies a booking **from the ZenSpace side** (not from your
bridge). This lets you sync admin changes back into Google/Outlook.

```json
{
  "event": "updated",
  "booking": {
    "booking_id": "b1234567-89ab-cdef-0123-456789abcdef",
    "meeting_space_id": "123e4567-...",
    "start_time": "2026-04-20T15:00:00.000Z",
    "end_time":   "2026-04-20T16:30:00.000Z",
    "title": "Team Sync (rescheduled)",
    "description": "...",
    "organizer_email": "alice@example.com",
    "organizer_name": "Alice Smith",
    "organizer_phone": "+14155551212",
    "organizer_company": "Acme Corp",
    "status": "confirmed",
    "booking_source": "admin"
  },
  "connection_id": "c7e8f9a0-1234-5678-9abc-def012345678"
}
```

`event` values:

- `"created"` — an admin created a booking on the space from ZenSpace's UI.
- `"updated"` — an admin modified a booking.
- `"cancelled"` — an admin cancelled or deleted a booking.

**Feedback-loop protection.** ZenSpace does **not** send webhooks for
changes your bridge itself made (detected via API-key origin). You will
only receive events for admin-originated changes. This prevents
`bridge → ZenSpace → bridge → Google → bridge → ZenSpace` loops.

Webhooks are only sent while the connection's status is `approved`. If
it's `suspended` or `declined`, events stop. If re-approved, events
resume — but you will not receive backlogs of events that occurred while
suspended.

### 7.5 Booking-access credentials webhook — `booking.credentials_ready`

Fired when ZenSpace has obtained booking-access credentials (magic link,
lock code, WiFi voucher) from the edge infrastructure for a booking on a
meeting space with an approved third-party connection.

**When it fires**

This event mirrors the T-15 / T-0 organizer-email flow that runs for
`zenspace` / `hybrid` spaces with `email_notification=true`. For `third_party`
spaces, it's the channel integrators use to receive access credentials
without relying on ZenSpace email delivery.

- **T-15 minutes** before `booking.start_time` — primary fire.
- **T-0** (exactly at `booking.start_time`) — second fire if the first
  didn't land or if fresh credentials are needed.
- **Retry** — extra fires from ZenSpace's internal retry loop if the
  edge infrastructure reports transient failures. Distinguished by
  `trigger: "retry_attempt"` in the payload.

**Prerequisite.** The meeting space must have an active **physical-space
mapping** to ZenSpace's edge infrastructure. Without a mapping, no
credentials are generated and no webhook fires. This is the same
prerequisite as the organizer-email flow.

**Delivery channel matrix.** This event fires independently of the
organizer-email flow. Depending on the space's `email_notification`
setting and whether your connection has a `webhook_url`:

| `email_notification` | Your `webhook_url` | What happens |
|---|---|---|
| `true` | set | email **and** webhook both fire |
| `true` | not set | email only |
| `false` | set | webhook only (your primary channel for third_party mode) |
| `false` | not set | nothing fires — credentials are generated but not delivered anywhere |

**Payload:**

```json
{
  "event": "booking.credentials_ready",
  "event_time": "2026-04-22T10:45:00.000Z",
  "trigger": "future_booking_t_minus_15",
  "booking": {
    "id": "b1234567-89ab-cdef-0123-456789abcdef",
    "space_id": "123e4567-...",
    "start_time": "2026-04-22T11:00:00.000Z",
    "end_time":   "2026-04-22T12:00:00.000Z",
    "title": "Design review",
    "organizer_email": "asha@example.com"
  },
  "credentials": {
    "magic_link_url": "https://edge.zenspace.io/ma/abc123...",
    "lock": { "template_html": "<div>Unlock code: 1234...</div>" },
    "wifi": { "template_html": "<div>WiFi: zenspace-guest ...</div>" }
  },
  "request_id": "zenedge_req_abc123"
}
```

**Field notes:**

- `trigger` — one of `future_booking_t_minus_15`, `booking.start`,
  `retry_attempt`, or a mode-specific trigger string from the edge
  layer. Use this to distinguish duplicate-looking fires and for logs.
- `credentials.magic_link_url` — the access URL you should put in front
  of your users. Time-bound; treat as secret. May be `null` if the
  edge layer couldn't issue one but issued lock/wifi separately.
- `credentials.lock` / `credentials.wifi` — pre-rendered HTML fragments
  suitable for embedding in an email or popup. Each field is `null` if
  that credential type isn't relevant for the space.
- `request_id` — edge-layer correlation ID. Include in support tickets.

**Transport — same as other webhooks (§7.1) with these additions:**

- **Retry policy:** 3 attempts with backoff (immediate, +1s, +5s). 5s
  timeout per attempt. `4xx` responses are non-retryable; `5xx` and
  timeouts retry. After 3 failed attempts the event is dropped — you do
  **not** get a delivery log. If you miss a T-15 fire but respond to T-0,
  you get a second chance.
- **Headers:**
  - `Content-Type: application/json`
  - `x-event: booking.credentials_ready`
  - `x-idempotency-key: <booking_id>:<trigger>:<connection_id>`
- **Dedupe:** T-15, T-0, and retry fires each carry a distinct
  `x-idempotency-key` (different `trigger` component). If you receive
  the **same** `x-idempotency-key` twice, it's a safe duplicate — same
  booking, same trigger phase — and you can ignore it.

**Best practices on your side:**

1. Store the mapping `(x-idempotency-key → delivered_at)` and skip
   duplicates.
2. Present the `magic_link_url` to the user as early as possible after
   receiving the T-15 event. You have ~15 minutes until the booking
   starts.
3. If you only ever see `trigger: "booking.start"` (T-0) and never
   T-15, check that the edge layer's primary fire is configured —
   raise with ZenSpace support if not.
4. If the space's `email_notification=true` and your webhook is also
   set, you'll receive both channels. Pick one as authoritative and
   treat the other as a belt-and-braces fallback.

### 7.6 Registering richer event subscriptions — `POST /api/v1/webhooks`

This is a **separate channel** to the connection's `webhook_url`. Use it
to receive events that §7.3–7.5 don't cover, with retries and a queryable
delivery log.

**What today's pipeline actually delivers.** The dispatcher only fires
events when a ZenCore service emits them. As of this writing the wired
emitters are:

- `space.update` — meeting-space metadata edited
- `space.delete` — meeting-space deleted
- `space.updateAvailability` — business hours or unavailability changed

The event-types catalog (next subsection) lists many more codes
(`booking.created`, `booking.start`, etc.). Those are reserved but **not
yet emitted by this pipeline** — see [§12](#12-known-gaps--roadmap). Subscribe
to them today and the row is created; you just won't get callbacks until
the bookings module is wired to emit them.

#### Auth

Send your org API key as `x-api-key`. The key must hold the
`create:webhooks` permission. Issue it through the same admin flow you
used for the connection-request key (§4.1) — a single key can carry
`create:bookings` and `create:webhooks` together.

#### Request

```http
POST /api/v1/webhooks HTTP/1.1
Host: api-spaceos.zenspace.io
x-api-key: zsk_live_xxx
Content-Type: application/json

{
  "name": "Google Calendar sync — Boardroom A",
  "webhook_url": "https://your-bridge.example.com/hooks/zenspace",
  "auth_key": "rotate-me-occasionally",
  "is_active": true,
  "retry_count": 3,
  "timeout_ms": 5000,
  "meeting_space_id": "8c2e1d50-9f8a-4f5d-bf3b-2c0a3a9c1e10",
  "events": [
    { "event": "space.update",             "duration": 0, "enabled": true },
    { "event": "space.delete",             "duration": 0, "enabled": true },
    { "event": "space.updateAvailability", "duration": 0, "enabled": true }
  ]
}
```

#### Field reference

| Field | Type | Required | Notes |
|---|---|---|---|
| `name` | string ≤255 | ✅ | Display name in the admin UI |
| `description` | string | optional | |
| `webhook_url` | https URL | ✅ | http is rejected — TLS required |
| `auth_key` | string ≤255 | optional | Sent to your receiver as `x-api-key` so you can verify it's ZenSpace |
| `is_active` | boolean | optional (default `true`) | Master kill switch |
| `retry_count` | int 0–10 | optional (default 3) | Attempts per delivery |
| `timeout_ms` | int 1–60000 | optional (default 5000) | Per-attempt timeout |
| `events` | array | ✅ | At least one element |
| `events[].event` | string | ✅ | Event code; validated against `event_types.event_code` |
| `events[].duration` | int | ✅ | Timing offset in minutes (-5 = 5 min before, 0 = exact, +5 = after). Only meaningful for timed events; use `0` for instantaneous ones |
| `events[].enabled` | boolean | ✅ | Per-event toggle — flip without re-creating |
| `meeting_space_id` | UUID | one of these three | Subscribes to events for this space only |
| `space_group_id` | UUID | required | Subscribes across the group |
| `organization_id` | UUID | | Subscribes across the whole org |

Match the scope to the connection you registered in §5. If your
connection is for one meeting space, scope the webhook the same way —
`organization_id` here would deliver events for spaces you don't own.

#### Discovering available events

```http
GET /api/v1/event-types/active
x-api-key: zsk_live_xxx
```

Returns the active event-type catalog (`event_code`, `display_name`,
`category`). `events[].event` in your subscription must match
`event_code` exactly. The catalog is also embedded inline in the
Swagger description of `POST /webhooks` if you'd rather not make a
second call.

#### Managing the subscription

| Operation | Endpoint |
|---|---|
| Get one | `GET /api/v1/webhooks/{id}` (includes delivery stats) |
| Update events, URL, scope, retry/timeout | `PUT /api/v1/webhooks/{id}` |
| Delete | `DELETE /api/v1/webhooks/{id}` |
| Delivery log (paginated) | `GET /api/v1/webhooks/{id}/deliveries` |
| Single delivery | `GET /api/v1/webhooks/{id}/deliveries/{deliveryId}` |

#### Delivery semantics

- **Method:** `POST` to your `webhook_url`.
- **Headers:** `Content-Type: application/json`, plus `x-api-key: <auth_key>`
  if you set one on the subscription.
- **Body:** event-specific payload — see each emitter's documentation.
  All payloads include the `meeting_space_id` (or relevant scope) so you
  can route by space.
- **Retries:** up to `retry_count` attempts on network error / 5xx /
  non-2xx. 4xx is non-retryable.
- **Ordering:** not guaranteed across events.
- **Idempotency on your side:** ZenSpace does not send an
  `Idempotency-Key` header here today — dedupe on whatever stable id
  the payload carries (booking id, meeting-space id + timestamp, etc.).

#### Lifecycle — independent from the connection

This subscription is **not coupled** to the third-party connection from
§5. Concretely:

- Approving/suspending/disconnecting a connection does **not** touch
  webhook subscriptions you created here.
- When you `POST /third-party/{id}/disconnect`, also
  `DELETE /api/v1/webhooks/{your-id}` — otherwise you'll keep
  receiving events for a space you no longer integrate with.
- Conversely, deleting the subscription does not disconnect the
  connection — bookings against it still work.

---

## 8. Idempotency and dedupe

> **This section changed.** Earlier revisions of this guide said ZenSpace does not
> deduplicate creates and that backend dedupe was on the roadmap. **It now exists.**

`POST /api/v1/bookings` is **idempotent on `external_calendar_id`**, scoped per
organization. If a booking with that `external_calendar_id` already exists for the
org, the request returns the **existing booking with HTTP 200** and creates nothing
(a genuinely new create still returns **201** — both are success).

### Use the top-level field, not metadata

⚠️ The idempotency key is the **top-level `external_calendar_id`** field on the
create body — *not* `metadata.external_event_id`. If your bridge only sets the
metadata key, you get **no server-side protection**. Set both if you like, but
`external_calendar_id` is the one that counts.

```jsonc
POST /api/v1/bookings
{
  "meeting_space_id": "…",
  "start_time": "2026-09-10T14:00:00.000Z",
  "end_time":   "2026-09-10T15:00:00.000Z",
  "external_calendar_id": "google-event-id-abc123",   // ← the idempotency key
  "organizer_email": "jane@example.com"
}
```

Behaviour worth knowing before you rely on it:

- **The lookup runs first**, before the meeting-space fetch and every validation. A
  replay cannot fail on something that changed after the original was accepted
  (space deactivated, hours edited, price drifted).
- **Cancelled bookings match too.** A replay after cancellation does *not* silently
  re-book — you get the cancelled booking back. **Always read `status` on a 200**
  rather than assuming the booking is live.
- **A replay carrying different times returns the stored booking unchanged.** The
  new times are logged, not applied. This is a create endpoint; to reschedule, use
  `PUT /api/v1/bookings/{id}`.
- **Scoped per organization**, not globally — provider event ids are opaque strings
  and two tenants may legitimately use the same one.

### Why this matters for calendar bridges

Without it, a replay fell through to the overlap check and was refused **400 against
the booking the first call had just created**. Bridges re-POST routinely (webhook
redelivery, nightly reconciliation, provider push replay) and commonly read a 4xx as
"ZenSpace rejected this" → they delete the upstream event, destroying a real booking.

### Still worth doing on your side

1. **Persist a mapping** `(external_event_id, zenspace_booking_id)`. Idempotency is a
   safety net, not a substitute for knowing what you created.
2. **Debounce Google webhook bursts** (Google can fire 2–3 times for one change)
   using `syncToken` semantics before pushing to ZenSpace.

### Retrying after failures

- **Network errors / 5xx:** safe to retry with the same body. With
  `external_calendar_id` set, a retry after an unseen success now returns the
  original booking instead of duplicating.
- **400s:** do not retry blindly — read `error.details.category` (§9). `permanent`
  means fix the payload; `transient` may be worth a retry.
- **401s:** check the API key — and check its `scopes` (§4).
- **404s on update/cancel:** the booking may have been deleted. Remove your mapping
  and treat the next source event as a fresh create.

---

## 9. Errors — status codes and handling

| HTTP | Meaning | What to do |
|---|---|---|
| 200 / 201 | Success | — |
| 400 | Validation or precondition failed | Read `message`; do not retry without changing the request. |
| 401 | Auth failed | Check the API key. Do not retry with the same key. |
| 403 | Forbidden (e.g. trying to access another org's data) | Check `organization_id` and `space_id`. |
| 404 | Resource not found | If a known booking returns 404, remove from your dedupe mapping. |
| 409 | Conflict (rare) | Refresh state via `GET /api/v1/bookings/{id}` and re-decide. |
| 429 | Rate limit (not enforced today, but reserved) | Back off exponentially. |
| 5xx | Server error | Retry with backoff: 1s, 2s, 5s, 15s, 1m, give up. Alert a human. |

### Machine-readable refusal codes on `POST /api/v1/bookings`

Booking refusals carry a stable code under **`error.details`**, so you no longer have
to parse `message`:

```json
{
  "status": 400,
  "success": false,
  "message": "Time slot is not available",
  "error": {
    "code": "BAD_REQUEST",
    "details": {
      "error_code": "SLOT_CONFLICT",
      "category": "transient",
      "conflicting_booking_ids": ["b1234567-89ab-cdef-0123-456789abcdef"]
    }
  },
  "data": null,
  "timestamp": "2026-09-10T12:00:00.000Z"
}
```

| `error_code` | `category` | Meaning |
|---|---|---|
| `SLOT_CONFLICT` | transient | Something already occupies the slot (`conflicting_booking_ids` lists them) |
| `OUTSIDE_BUSINESS_HOURS` | permanent | Requested window falls outside the space's hours |
| `SPACE_CLOSED` | permanent | The space is closed that day |
| `SPACE_UNAVAILABLE` | transient | A blackout / unavailability window covers the slot |
| `PRICE_MISMATCH` | permanent | Submitted price disagrees with ZenSpace's calculation |
| `DEPOSIT_MISMATCH` | permanent | Submitted deposit disagrees |
| `INVALID_TIME_RANGE` | permanent | `end_time` not after `start_time`, or a malformed window |
| `CONNECTION_NOT_APPROVED` | transient | No approved third-party connection for this space yet |
| `SPACE_NOT_FOUND` | permanent | Unknown `meeting_space_id` |

**Branch on `category`, not on the HTTP class.** `permanent` → fix the payload and do
not retry; `transient` → the same request may succeed later. The 4xx/5xx split happens
to line up today but is not guaranteed by design.

This addition is **purely additive**: `message` is byte-identical to before and
`error.code` is still status-derived, so existing consumers are unaffected. Extras nest
under `details` because that is the only place the exception filter surfaces them.

### Example error body (non-booking endpoints)

```json
{
  "status": 400,
  "success": false,
  "message": "Cannot create booking: No approved third-party connection found. Connection(s) status: 1 pending. Please wait for connection approval or contact administrator.",
  "data": null,
  "timestamp": "2026-04-13T12:00:00.000Z"
}
```

Log `message` verbatim; where no `error_code` is present it remains the most useful
debugging signal.

---

## 10. End-to-end walkthrough — Google Calendar example

This is the full happy path for a single booking.

```
1. User creates event in Google Calendar
   ┌──────────────────────────────────────┐
   │ "Team Sync"                          │
   │ 2026-04-20  14:00–15:00              │
   │ Room: conf-room-a@resource.calendar.google.com │
   │ Organizer: alice@acme.com            │
   └──────────────────────────────────────┘

2. Google Calendar fires events.watch push to your bridge

3. Bridge:
   a. Resolves room → ZenSpace meeting_space_id via its mapping store
   b. Checks its (google_event_id → zenspace_booking_id) map — not found
   c. Translates event to ZenSpace payload (UTC times, attendees, metadata)
   d. POST /api/v1/bookings  x-api-key: zsk_live_...

4. ZenSpace:
   a. Validates: space is third_party mode ✅
   b. Validates: approved connection exists ✅
   c. Validates: no time overlap ✅
   d. Creates booking → status=confirmed, payment_status=paid
   e. Schedules state-transition jobs (PENDING → IN_PROGRESS → ENDED)
   f. (No outbound webhook — change came from the bridge)

5. Bridge receives 201 Created with booking.id
   Stores mapping: google-event-abc123 ↔ b1234567-...

6. T-15 minutes before start_time:
   If meeting space has a physical-space mapping configured:
   a. ZenSpace generates door/lock PIN, magic link, wifi voucher
   b. ZenSpace (optionally) emails alice@acme.com with the credentials
      (skipped if organizer_email was omitted)

7. T = start_time:
   a. Meeting space state transitions to IN_PROGRESS
   b. ZenCloud WebSocket emits zscloud:meeting.status_update

8. T = end_time:
   a. Meeting space state transitions to ENDED
   b. Scheduler cleans up

9. (Later, if admin cancels the booking from ZenSpace admin UI)
   ZenSpace POSTs to your webhook_url:
   { event: "cancelled", booking: { booking_id: "...", ... },
     connection_id: "..." }
   Bridge receives → deletes the Google Calendar event.
```

Full updates and cancellations follow the same shape:

- **User updates event in Google** → bridge catches → `PUT /api/v1/bookings/{id}`
- **User deletes event in Google** → bridge catches → `POST /api/v1/bookings/{id}/cancel`
- **Admin updates booking in ZenSpace** → webhook `event: "updated"` →
  bridge updates Google event
- **Admin cancels booking in ZenSpace** → webhook `event: "cancelled"` →
  bridge deletes Google event

---

## 11. Go-live checklist

Run through this before flipping your bridge to production.

### Correctness

- [ ] Production API key issued, key_type=`live`, stored in a secrets
      manager (not in source control).
- [ ] Connection approved for every meeting space you will mirror.
- [ ] `webhook_url` verified reachable and returns 2xx.
- [ ] `metadata.external_event_id` set on every create.
- [ ] `(external_event_id → zenspace_booking_id)` persistence is durable
      (survives bridge restart).
- [ ] Timezone conversions verified — create a booking for 2pm local time
      and confirm `start_time` in the response is the correct UTC value.

### Resilience

- [ ] Retry policy implemented on 5xx (exponential backoff, cap).
- [ ] Dead-letter queue or alert for repeated failures.
- [ ] Webhook handler is idempotent and responds under 2s.
- [ ] Webhook URL uses a hard-to-guess path and is HTTPS only.
- [ ] Google `events.watch` channel renewal scheduled (channels expire ~7
      days).

### Security

- [ ] API key is not logged.
- [ ] API key is rotatable (you can roll it without downtime).
- [ ] IP allowlist on the API key if your bridge has a stable egress IP.
- [ ] TLS verified on all outbound calls to ZenSpace.

### Operational

- [ ] Logs include `booking_id`, `external_event_id`, and `connection_id`
      for every request.
- [ ] Metrics: create/update/cancel rates, error rates, webhook
      ingestion rate.
- [ ] Runbook for the top 3 error messages (connection not approved,
      time overlap, invalid API key).

---

## 12. Known gaps / roadmap

Documented honestly so you can plan around them.

| Area | Today | Planned |
|---|---|---|
| Inbound webhook (your bridge → ZenSpace via the `zenspace_webhook` URL in the approval payload) | 404 — not implemented | Will be added; use REST API in the meantime |
| Webhook signing | Plain POST, no HMAC | HMAC with per-connection secret |
| Webhook retries on the connection's `webhook_url` (§7.3, §7.4) | None — fire and forget | Retry with backoff + dead-letter (retries already work for the generic subscription pipeline, §7.6) |
| Webhook delivery log surfaced to integrators | Only for `POST /webhooks` subscriptions (§7.6) — `GET /webhooks/{id}/deliveries` | Log surface for the connection's `webhook_url` channel too |
| Booking events emitted into the generic `POST /webhooks` pipeline | Not wired — only `space.update`, `space.delete`, `space.updateAvailability` fire today. `booking.*` codes are catalogued but no service emits them yet. | Bookings service to emit `booking.created` / `booking.start` / `booking.cancelled` / `booking.end` etc. through `WebhooksService.triggerWebhook`. Until then, use the connection's `webhook_url` (§7.4) for admin-originated booking changes. |
| Backend dedupe on booking creates | **Shipped** — `POST /bookings` is idempotent on the top-level `external_calendar_id`, per organization (§8). Note the key is that field, **not** `metadata.external_event_id`. | Nothing further planned; an `Idempotency-Key` header is not needed |
| Machine-readable refusal codes | **Shipped** — `error.details.error_code` + `category` on booking refusals (§9) | Extend the taxonomy to non-booking endpoints |
| Partner-sold bookings (you collected payment) | **Shipped** — send `booked_from` (§6.8); skips price validation, no Stripe intent, booking confirmed + paid | — |
| Concurrent-create protection | ⚠️ **Gap.** Idempotency covers *replays*, but there is no lock or transaction between the conflict check and the insert, so two **simultaneous** requests for the same slot can both succeed. | Row-level locking / exclusion constraint |
| `GET /api/v1/third-party/{connection_id}/bookings` (bridge-auth list) | None | Planned |
| Rate limits | Not enforced; `429` reserved | Per-key quotas |
| Resubmitting after `declined` | Must create a new request | No change planned |
| Multiple modes on one space | Supported (one approved per mode) | No change planned |

---

## 13. Reference

### 13.1 Operating-mode enum

| Value | Meaning | Eligible for `booked_from` handling (§6.8)? |
|---|---|---|
| `zenspace` | ZenSpace owns the booking lifecycle; the room is publicly bookable in the ZenSpace booking app. | No |
| `hybrid` | The room is sold through **both** ZenSpace's booking app and your platform. Stays on all public availability surfaces. **The mode most partners want.** | Yes |
| `third_party` | Your system owns the lifecycle exclusively. The room is excluded from every ZenSpace public availability/browse surface, and organizer emails become opt-in per space (`email_notification`, default `false`). | Yes |

Set via `PUT /api/v1/meeting-spaces/{id}` with
`{ "operating_settings": { "operating_mode": "hybrid" } }` (permission
`update:meetingSpaces`), or by the org admin in the admin UI.

### 13.2 Connection-status enum

| Value | Meaning |
|---|---|
| `pending` | Awaiting admin action. |
| `approved` | Active. Booking events flow. |
| `declined` | Terminal. Submit a fresh request to try again. |
| `suspended` | Was approved, currently paused. Booking events stop. |

### 13.3 Third-party-mode wire values

| Value | Use for |
|---|---|
| `Google Calendar` | Google Calendar / Workspace |
| `Outlook Calendar` | Outlook.com / personal Outlook |
| `Microsoft Outlook` | Microsoft 365 / Exchange |
| `Apple Calendar` | iCloud Calendar |
| `Other` | Anything else |

### 13.4 Booking-source enum

`web`, `mobile`, `api`, `admin`, `walk_in`, `external`. Use `external`
for bridge-created bookings.

### 13.5 Booking-status and payment-status values

Booking status: `pending`, `confirmed`, `in_progress`, `ended`,
`cancelled`, `no_show`.

Payment status: `pending`, `paid`, `failed`, `cancelled`, `refunded`,
`partial`.

For `third_party` bookings created via API: `status=confirmed`,
`payment_status=paid` at creation time.

### 13.6 Source-of-truth file references

Helpful if you are debugging with a ZenSpace engineer:

- Integration controller: [src/modules/third-party-connections/third-party-connections.controller.ts](../src/modules/third-party-connections/third-party-connections.controller.ts)
- Integration service (lifecycle, sync, webhook send): [src/modules/third-party-connections/third-party-connections.service.ts](../src/modules/third-party-connections/third-party-connections.service.ts)
- Connection entity: [src/entities/third-party-connection.entity.ts](../src/entities/third-party-connection.entity.ts)
- Connection-request DTO: [src/modules/third-party-connections/dto/create-third-party-connection.dto.ts](../src/modules/third-party-connections/dto/create-third-party-connection.dto.ts)
- Booking controller: [src/modules/bookings/bookings.controller.ts](../src/modules/bookings/bookings.controller.ts)
- Booking service: [src/modules/bookings/bookings.service.ts](../src/modules/bookings/bookings.service.ts)
- Create-booking DTO: [src/modules/bookings/dto/create-booking.dto.ts](../src/modules/bookings/dto/create-booking.dto.ts)
- API-key guard: [src/modules/auth/api-key.guard.ts](../src/modules/auth/api-key.guard.ts)
- Space-message controller (§14): [src/modules/notifications/notifications-external.controller.ts](../src/modules/notifications/notifications-external.controller.ts)
- Space-message DTO (§14): [src/modules/notifications/dto/send-space-message.dto.ts](../src/modules/notifications/dto/send-space-message.dto.ts)
- Notifications service (`sendSpaceMessage`): [src/modules/notifications/notifications.service.ts](../src/modules/notifications/notifications.service.ts)

---

## 14. Notifying meeting-space members (ad-hoc messages)

Separate from the booking flow, ZenSpace exposes one endpoint that lets your
system push an arbitrary message to the **members** of a meeting space — the
people an admin has added to that space inside ZenSpace (operations staff,
hosts, room owners, etc.). Use it for operational notices ("maintenance moved
to 4pm", "AV issue reported in this room") rather than guest-facing booking
communication.

This is **not** a booking webhook and **not** tied to a third-party connection.
It is a plain request your system makes *to* ZenSpace.

### 14.1 What it does

- Delivers your `message` verbatim to **every active member** of the meeting
  space, on each channel you request (`email`, `sms`, `fcm`).
- **Honors each member's per-channel preference.** A member who has email
  switched off for that space won't receive the email even if you request
  `email`. So the number of people reached on a channel can be lower than the
  total member count — this is expected, not an error.
- No notification rule and no template are involved; the body is sent as-is.
- Every send attempt is recorded server-side (visible to admins), so missed
  deliveries are auditable.

> **Not the same as `email_notification` in §6.1.** That space-level flag gates
> *organizer* emails for the booking lifecycle. This endpoint targets *space
> members* and obeys their individual membership preferences — a distinct set
> of recipients and toggles.

### 14.2 `POST /api/v1/notifications/space-message`

```http
POST /api/v1/notifications/space-message
Content-Type: application/json
x-org-api-key: zsk_live_XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
```

```json
{
  "meeting_space_id": "123e4567-e89b-12d3-a456-426614174111",
  "mediums": ["email", "fcm"],
  "subject": "Maintenance window rescheduled",
  "message": "The 3pm maintenance for this room has moved to 4pm."
}
```

**Authentication.** This route requires the organization API key in the
`x-org-api-key` header (the `x-api-key` / `Authorization: Bearer zsk_...` forms
from §4 are also accepted). The key's organization **must own** the target
meeting space, or the call is rejected with `403` — an org key cannot message
another org's spaces.

#### Field reference

| Field | Type | Required | Notes |
|---|---|---|---|
| `meeting_space_id` | UUID | yes | Space whose active members receive the message. Must belong to the API key's organization. |
| `mediums` | string[] | yes | Any of `"email"`, `"sms"`, `"fcm"`. At least one, no duplicates. |
| `message` | string | yes | Body, delivered verbatim. Supports `{{space_name}}` / `{{organization_name}}` interpolation; any other unknown `{{placeholder}}` resolves to an empty string. |
| `subject` | string | **required iff `mediums` includes `email`** | Email subject. Also used as the push / in-app title for `fcm`. Ignored for `sms`. When omitted on an sms/fcm-only send, the push title falls back to the space name. |

#### Successful response

```http
HTTP/1.1 201 Created
```

```json
{
  "status": 201,
  "success": true,
  "message": "Space message dispatched",
  "data": {
    "meeting_space_id": "123e4567-e89b-12d3-a456-426614174111",
    "results": [
      { "medium": "email", "recipients_resolved": 4 },
      { "medium": "fcm",   "recipients_resolved": 3 }
    ],
    "total_recipients_resolved": 5
  },
  "timestamp": "2026-04-13T12:00:00.000Z"
}
```

- `results[].recipients_resolved` — members targeted on that channel **after**
  applying their per-channel preferences.
- `total_recipients_resolved` — distinct member count across all requested
  channels.
- The response reports who was *targeted*, not per-user send success. Per-
  recipient delivery outcomes (sent / failed / skipped, with reasons) are
  recorded server-side and visible to ZenSpace admins.

#### `fcm` behavior

An `fcm` send delivers a push to each member's registered devices **and**
writes an in-app inbox entry, so the message is visible on next app open even
if the push itself fails.

#### Errors

| Condition | HTTP | Notes |
|---|---|---|
| Validation failed (missing `message`, empty `mediums`, `subject` missing while `email` requested, bad UUID) | 400 | Read `message`; fix and resend. |
| Missing or invalid `x-org-api-key` | 401 | |
| The key's organization does not own `meeting_space_id` | 403 | |
| `meeting_space_id` not found | 404 | |

#### cURL

```bash
curl -X POST https://api-spaceos.zenspace.io/api/v1/notifications/space-message \
  -H "Content-Type: application/json" \
  -H "x-org-api-key: zsk_live_XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX" \
  -d '{
    "meeting_space_id": "123e4567-e89b-12d3-a456-426614174111",
    "mediums": ["email", "fcm"],
    "subject": "Maintenance window rescheduled",
    "message": "The 3pm maintenance for this room has moved to 4pm."
  }'
```

---

## 15. Getting help

1. **Swagger UI** — live reference for every field, every endpoint:
   **https://api-spaceos.zenspace.io/docs/third-party**
2. **Admin panel** (for API-key and connection approvals):
   **https://admin-spaceos.zenspace.io/**
3. **Implementation patterns** (Pattern A: in-process, B: low-code,
   C: standalone) are in
   [OPERATING_MODE_THIRD_PARTY.md](./OPERATING_MODE_THIRD_PARTY.md).
4. **Bugs or questions:** contact your ZenSpace engineering contact.
   Include `connection_id`, `booking_id`, request body, and full error
   response in any report.
