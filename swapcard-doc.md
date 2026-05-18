# Swapcard ↔ ZenSpace Bridge

A small web service that turns **Swapcard meeting webhooks** into **ZenSpace
bookings**, and lets operators create meetings the other direction from
inside the admin UI. Map each Swapcard event Location to one ZenSpace
meeting space; once mapped, every meeting created or updated in Swapcard is
automatically reflected as a booking in ZenSpace (a.k.a. ZenCore), and the
operator can also originate meetings from the bridge itself.

For production-readiness guidance see [PRODUCTION.md](PRODUCTION.md).
For AWS-specific deploys see [AWS_PRODUCTION.md](AWS_PRODUCTION.md).

---

## 1. What it does

```
┌───────────────┐  webhook   ┌───────────────┐  REST    ┌──────────────┐
│   Swapcard    │ ─────────► │    Bridge     │ ───────► │   ZenCore    │
│  (event mtg)  │  HMAC sig  │  (this app)   │ x-api-key│ (bookings)   │
└───────────────┘    ▲       └───────────────┘          └──────────────┘
                     │              ▲ │
            GraphQL  │              │ │ (ZenCore connection-status webhook)
            mutation │              └─┘
            (operator-
             initiated)
```

- **Inbound (the dominant flow):** receives `meeting_create` /
  `meeting_update` webhooks from Swapcard, verifies the HMAC signature, and
  creates the matching ZenSpace booking.
- **Outbound (the reverse flow):** operator clicks "+ Create meeting" on a
  mapping row, picks a slot + 2 participants, and the bridge calls Swapcard's
  `createMeeting` GraphQL mutation and then ZenCore's REST `/bookings` —
  Swapcard's resulting webhook is then an idempotent no-op.
- **Outbound REST:** calls ZenCore REST APIs (`/api/v1/bookings`,
  `/api/v1/third-party/connection-request`, etc.) using the operator's
  ZenSpace API key.
- **Stateful:** stores mapping rows + synced bookings in a local SQLite
  database (file path is configurable; on Fly it lives on a persistent
  volume that's auto-created from `fly.toml`).
- **Admin UI:** a small web app (cookie-session login, mapping form, mapping
  list, bookings list, in-app meeting creator) for setting things up and
  observing sync status.

---

## 2. Repository layout

```
swapcard-bridge/
├── DOCS.md                  ← this file
├── README.md                ← short quick-start
├── PRODUCTION.md            ← production-readiness plan (platform-agnostic)
├── AWS_PRODUCTION.md        ← AWS-specific deploy guide (Lightsail)
├── Dockerfile               ← node:20-slim + better-sqlite3 build deps
├── docker-compose.yml       ← for single-host deploys (Lightsail/EC2/etc.)
├── fly.toml                 ← Fly.io app config (volume auto-creates via initial_size)
├── litestream.yml           ← optional SQLite → S3 replication (off by default)
├── .gitattributes           ← pins *.sh to LF so scripts run inside Linux containers
├── package.json
├── migrations/              ← idempotent SQL migrations (numbered)
├── scripts/
│   ├── entrypoint.sh        ← container entrypoint (wraps node in litestream when enabled)
│   ├── fix-fly-volume.sh    ← one-shot recovery for the "DB wipes on deploy" incident
│   └── introspect-swapcard.js  ← schema dump for local exploration
└── src/
    ├── server.js            ← Fastify bootstrap + auth gate hook + /healthz
    ├── db.js                ← SQLite (better-sqlite3) + repos
    ├── lib/
    │   ├── auth.js          ← cookie-session helpers + hardcoded creds
    │   ├── hmac.js          ← Swapcard HMAC-SHA256 verifier
    │   ├── timezone.js      ← "lazy Z" → real UTC conversion for slot times
    │   ├── swapcard.js      ← GraphQL client + queries + mutations
    │   └── zencore.js       ← REST client + error class + space helpers + OTP auth
    ├── routes/
    │   ├── auth.js          ← /login, /logout, OTP endpoints
    │   ├── pages.js         ← /, /load, /save, /mappings, /bookings, /create-meeting, ...
    │   └── webhooks.js      ← /webhooks/swapcard, /webhooks/zencore
    ├── views/               ← EJS templates (layout, home, map, event_detail, ...)
    └── public/
        └── styles.css       ← design system + components
```

---

## 3. How a sync flows end-to-end

### 3a. One-time setup (admin)

1. Operator opens the bridge admin UI (`/`) and signs in.
2. Pastes ZenSpace API key + Swapcard API key + Swapcard event ID. Submits.
3. Bridge calls Swapcard's GraphQL `event(id)` query and renders one row per
   Location.
4. For each Location the operator pastes a ZenSpace meeting space UUID and
   clicks **Connect**. The bridge:
   - Calls `GET /api/v1/meeting-spaces/{id}` on ZenCore to derive
     `organization_id` and `space_group_id` from the space itself (so the
     operator doesn't have to type them).
   - Calls `POST /api/v1/third-party/connection-request` to create the
     connection between Swapcard org and that ZenSpace space.
   - Generates a fresh HMAC secret, saves the mapping row, and shows the
     secret in a copy-modal.
5. Operator pastes the URL `https://<bridge-host>/webhooks/swapcard` plus
   the HMAC secret into Swapcard Studio → Integrations → Webhooks, subscribed
   to **Meetings: Created & Updated**.
6. A ZenSpace admin approves the third-party connection on the ZenCore side.
   (See §6 on how the bridge learns about the approval.)

### 3b. Per-meeting sync

```
Swapcard meeting created/updated
   ↓
POST /webhooks/swapcard (raw body, X-Signature-256 header)
   ↓
verify HMAC against every candidate mapping for the event
   ↓
extract place.id from payload  → find specific mapping for that Location
   ↓
(optional) GET meeting via Swapcard GraphQL → enrich participant names/emails
   ↓
POST /api/v1/bookings on ZenCore  (org_id, space_id, slot, attendees, ...)
   ↓
upsert synced_booking row, mark connection 'approved'
```

Cancellations work symmetrically: when the meeting status becomes
`DECLINED/CANCELED/EXPIRED`, the bridge calls `POST
/api/v1/bookings/{id}/cancel` on ZenCore.

### 3c. ZenCore booking → Swapcard meeting (mirror)

When a booking is created directly inside ZenSpace (not from this bridge),
ZenCore can call the bridge so it shows up as a Swapcard meeting too. The
operator clicks **Register ZenCore webhook** once on the event-detail
page; the bridge then handles every subsequent `booking.created` event
delivered to `/webhooks/zencore`.

```
ZenCore booking created
   ↓
POST /webhooks/zencore  (event=booking.created, booking={…})
   ↓
detect booking event (not a connection-status update)
   ↓
idempotency guard: synced_booking has this zenspace_booking_id? → skip
   ↓
find mapping by zenspace_space_id
   ↓
GraphQL: meetingSlotsAvailable(eventId)
   ↓
match the booking's UTC start/end against (lazyZ → real UTC) slot times
   ↓
GraphQL: eventPerson(filters: [{ emails: [...] }])  → resolve Relay IDs
   ↓
GraphQL: createMeeting(input: { eventId, slotId, placeId, participants })
   ↓
upsert synced_booking — the Swapcard webhook for the new meeting becomes
                         an idempotent no-op (status='active' is set)
```

Important quirks:

- **Webhook registration is per-organization, not per-event.** Clicking
  Register on any event registers the same URL globally; subsequent
  clicks return `already_existed: true`.
- **Attendee mapping is best-effort.** Emails on the ZenCore attendee list
  are looked up via `eventPerson(filters: [{ emails: [...] }])`. Each
  matched email becomes a Swapcard participant. The first matched attendee
  is flagged `isOrganizer: true`.
- **No matched EventPersons → mirror aborts.** A synced_booking row is
  recorded with `status='error'` and `last_error` explaining which emails
  did not match. Add the missing attendees to the Swapcard event first.
- **No matching slot → mirror aborts** with the same error pattern.
- **Cancellations are no-ops.** Swapcard's GraphQL has no
  `cancelMeeting`/`deleteMeeting` mutation in the schema this app talks
  to, so a cancelled ZenCore booking is logged and ignored — the Swapcard
  meeting stays in place.
- **Loop prevention.** If a booking arrives whose `zenspace_booking_id`
  already exists in `synced_booking` (meaning we created it ourselves
  via the webhook or `/create-meeting` flow), it's skipped.

### 3d. Operator-initiated meeting (reverse direction)

The same plumbing also runs backwards — useful when the operator wants to
book a Swapcard meeting from the bridge admin UI rather than from Swapcard
Studio. On any `approved`/`connected` mapping row, the **+ Create meeting**
button opens a modal asking for date, start time, end time (both stepped
in 15-minute increments), and two participant Relay-IDs.

```
operator clicks "+ Create meeting" → modal
   ↓
POST /mappings/:eventId/:locationId/create-meeting
   ↓
validate inputs (15-min boundary, end > start, distinct participants)
   ↓
GraphQL: meetingSlotsAvailable(eventId)
   ↓
find a slot whose beginsAt/endsAt exactly match the requested window
   ↓
GraphQL mutation: createMeeting(input: { eventId, slotId, placeId,
                                         participants: [{id,isOrganizer:true},{id}] })
   ↓
POST /api/v1/bookings on ZenCore (org_id, space_id, slot, attendees from
                                  the createMeeting response, ...)
   ↓
upsert synced_booking → the Swapcard webhook that fires for this same
                         meeting becomes an idempotent no-op
```

Important quirks of this path:

- **Timezone:** date + start + end are interpreted in the *event's* IANA
  timezone (the same timezone the `meetingSlotsAvailable` strings are in),
  not the operator's browser locale.
- **Slot matching:** `beginsAt`/`endsAt` are compared as strings after
  normalising the optional `T`/space separator and trailing `Z`. Matching
  is exact — no fuzzy windowing.
- **Organizer flag:** participant 1 in the modal is always sent as
  `isOrganizer: true`. ZenCore's first attendee gets `is_organizer: true`
  in the booking, so the same person carries through both systems.
- **Failure modes:**
  - *No matching slot:* 422, modal shows the available slots on that date.
  - *ZenCore rejects (suspended/declined/pending):* the local mapping's
    `connection_status` is corrected to reflect ZenCore's reality and an
    actionable hint is returned. The orphan Swapcard meeting stays in
    Swapcard (no public delete mutation in our schema) — the operator can
    either delete it from Swapcard Studio or pick a different slot on retry.
  - *Time overlap in ZenCore:* the operator picked a slot the ZenSpace
    side already has a booking for. Pick another slot.

---

## 4. Environment & configuration

All configuration is via env vars. See `.env.example` for the canonical list.

| Variable          | Default                      | Notes                                                           |
|-------------------|------------------------------|-----------------------------------------------------------------|
| `PORT`            | `8080`                       | HTTP port. Fly maps 8080 internally.                            |
| `NODE_ENV`        | `production`                 | Toggles pino-pretty logger in dev.                              |
| `DATABASE_PATH`   | `./bridge.db`                | Must point at a persistent location in Docker/Fly (`/data/...`).|
| `PUBLIC_URL`      | `http://localhost:3000`      | Used to construct the Swapcard webhook URL shown to operators.  |
| `LOG_LEVEL`       | `info`                       | pino level.                                                     |
| `ADMIN_EMAIL`     | `admin@zenspace.io`          | Login email. Override in production.                            |
| `ADMIN_PASSWORD`  | `bridge2026`                 | Login password. Override in production.                         |
| `SESSION_SECRET`  | (insecure placeholder)       | HMAC secret for the session cookie. **Must override in prod.**  |

Set production secrets on Fly:

```bash
fly secrets set \
  ADMIN_EMAIL="ops@yourorg.com" \
  ADMIN_PASSWORD="<long random>" \
  SESSION_SECRET="$(openssl rand -hex 32)" \
  PUBLIC_URL="https://swapcard-zencore.fly.dev" \
  -a swapcard-bridge
```

---

## 5. Data model

Two tables, defined in [migrations/001_init.sql](migrations/001_init.sql):

### `mapping`
One row per (Swapcard event, Swapcard location) pair. Holds **all** secrets
needed to handle a webhook for that pair: the per-mapping HMAC secret used to
verify Swapcard webhooks, and the ZenSpace API key + IDs used to create the
booking.

| Column                    | Notes                                                       |
|---------------------------|-------------------------------------------------------------|
| `swapcard_event_id`       | Swapcard Relay-style ID (e.g. `RXZlbnRfNDQ1OTY3Mw==`).      |
| `swapcard_location_id`    | Swapcard place ID (the `place.id` in the webhook payload).  |
| `swapcard_location_name`  | Human-readable name for the admin UI.                       |
| `swapcard_api_key`        | Swapcard event-admin API key (for GraphQL participant fetch).|
| `swapcard_webhook_secret` | HMAC-SHA-256 secret. Generated by the bridge.               |
| `zenspace_api_key`        | ZenSpace API key (`x-api-key` for ZenCore REST).            |
| `zenspace_org_id`         | Resolved from `getMeetingSpace` — not entered by the user.  |
| `zenspace_group_id`       | Resolved from `getMeetingSpace` — not entered by the user.  |
| `zenspace_space_id`       | Meeting space UUID the operator pasted.                     |
| `connection_status`       | `saved` / `pending` / `approved` / `connected` / `error`.   |
| `connection_id`           | ZenCore third-party connection id (when known).             |
| `last_error`              | Last error message from ZenCore, if any.                    |

Primary key is `(swapcard_event_id, swapcard_location_id)`.

### `synced_booking`
One row per Swapcard meeting that's been synced. Used for idempotency,
cancellation, and the **Bookings** admin page.

| Column                 | Notes                                              |
|------------------------|----------------------------------------------------|
| `swapcard_meeting_id`  | Primary key. From the webhook payload.             |
| `zenspace_booking_id`  | What ZenCore returned from `POST /bookings`.       |
| `start_time`/`end_time`| ISO-8601 strings.                                  |
| `participants`         | Comma-separated names (display only).              |
| `status`               | `active` / `cancelled` / `error`.                  |

---

## 6. The bridge↔ZenCore connection lifecycle

ZenCore enforces that bookings are only allowed for **approved** third-party
connections. The bridge cannot poll ZenCore for connection status (no public
GET endpoint), so it learns about approval in three ways:

1. **ZenCore approval webhook** — `POST /webhooks/zencore` updates
   `connection_status` when ZenCore notifies us. Best-effort: relies on
   ZenCore actually firing it with the expected payload shape.
2. **Implicit detection** — the first successful `createBooking` proves the
   connection is approved (otherwise it would have rejected with the
   precondition error). The webhook handler flips
   `connection_status='approved'` on success, and back to `'pending'` if it
   gets `isConnectionPrecondition`.
3. **Manual override** — operator can mark a row approved from the admin
   UI (button shown when status isn't already approved). Useful when
   waiting for the first booking and wanting the badge to reflect reality.

### Connection-already-exists handling

When the operator deletes a mapping locally and tries to reconnect (same
space or different space) the ZenCore-side connection still exists, and the
new connection-request returns 400 *"third-party connection already
exists"*. The bridge surfaces this in an **error modal** that explicitly
tells the operator to remove the existing connection from ZenCore admin
before retrying. (Earlier versions tried to silently treat it as success;
this turned out to be confusing — the operator now decides what to do.)

---

## 7. HTTP routes

### Public (no auth)
| Method | Path                      | Purpose                                  |
|--------|---------------------------|------------------------------------------|
| GET    | `/login`                  | Login page.                              |
| POST   | `/login`                  | Submit credentials, set session cookie.  |
| POST   | `/logout`                 | Clear session, redirect to `/login`.     |
| GET    | `/healthz`                | DB-backed health probe (200 / 503).      |
| POST   | `/webhooks/swapcard`      | Inbound Swapcard webhook (HMAC-verified).|
| POST   | `/webhooks/zencore`       | Inbound ZenCore connection-status hook.  |
| GET    | `/static/*`               | CSS, fonts.                              |
| GET    | `/favicon.ico`            | 204.                                     |

### Admin (require valid session cookie; see §8)
| Method | Path                                                       | Purpose                                                       |
|--------|------------------------------------------------------------|---------------------------------------------------------------|
| GET    | `/`                                                        | Events list (landing).                                        |
| GET    | `/events/new`                                              | Add-event form (creds + Swapcard event ID).                   |
| GET    | `/events/:eventId`                                         | Event detail — mappings + bookings + webhook info.            |
| POST   | `/events/:eventId/reload`                                  | Re-open the mapping editor for a known event.                 |
| POST   | `/events/:eventId/forget`                                  | Remove from the global event list (mappings stay).            |
| POST   | `/load`                                                    | Load Swapcard event, render `/map`.                           |
| POST   | `/save`                                                    | Test / Save / Connect a single mapping.                       |
| GET    | `/mappings`                                                | All saved mappings.                                           |
| POST   | `/mappings/:eventId/:locationId/disconnect`                | Revoke the ZenCore connection for one mapping.                |
| POST   | `/mappings/:eventId/:locationId/create-meeting`            | Operator-initiated meeting creator (see §3c).                 |
| POST   | `/mappings/delete-all`                                     | Remove every local mapping row.                               |
| GET    | `/bookings`                                                | All synced bookings.                                          |
| POST   | `/bookings/:meetingId/cancel`                              | Cancel one booking (ZenCore + local).                         |
| POST   | `/admin/event-webhook-secret`                              | Manually record a Swapcard webhook secret for an event.       |
| POST   | `/admin/zencore-webhook/register`                          | Register the bridge as a ZenCore booking-webhook recipient.   |
| POST   | `/admin/clear-database`                                    | Wipe every mapping + booking + event-webhook row.             |

---

## 8. Authentication

The admin UI is gated by a small cookie-session implementation in
[src/lib/auth.js](src/lib/auth.js).

- **Credentials** are hardcoded (overridable via env vars, see §4).
- **Token format:** `<base64url(JSON{exp,email})>.<hmac-sha256-hex>`. A
  single dot separator with `lastIndexOf` for splitting — works correctly
  even when the email itself contains dots.
- **Cookie attributes:** `HttpOnly`, `SameSite=Lax`. `Secure` is added
  conditionally — only when the request came over HTTPS (detected via
  `x-forwarded-proto` for proxies, `req.protocol` direct, or
  `req.socket.encrypted`). This is critical: browsers silently drop
  `Secure` cookies on plain HTTP, which broke localhost dev in earlier
  versions.
- **TTL:** 7 days.
- **Auth gate** is registered as an `onRequest` hook in
  [src/server.js](src/server.js). Public prefixes: `/webhooks/`, `/static/`,
  `/login`, `/logout`, `/healthz`, `/favicon.ico`. Everything else
  redirects to `/login?next=<original-url>` when unauthenticated.

Webhooks are **not** protected by cookie auth; they authenticate by HMAC.

---

## 9. Webhook security

`POST /webhooks/swapcard` verifies an HMAC signature on every request:

- Each mapping row owns its own webhook secret (auto-generated, 32 bytes
  hex). The secret is **shown once in a modal** after Connect; the
  operator pastes it into Swapcard Studio.
- On receipt, the bridge looks up *every* mapping for the event ID, then
  tries each mapping's secret against the signature header. The first
  match wins. (This way one bridge can safely host multiple events with
  different secrets.)
- The verifier accepts the signature in **hex, base64, or base64url**, with
  or without an `sha256=` prefix. Matched in constant time
  via `crypto.timingSafeEqual`.
- The raw request body is captured at the body-parser level (see the
  `addContentTypeParser` block in [src/server.js](src/server.js)) so the
  signature is computed against the exact bytes Swapcard sent.

`POST /webhooks/zencore` is **not** signed. Per the original ZenCore
integration guide, it relies on URL-secrecy. For production, switch to a
path secret or ask ZenCore for a signing scheme.

---

## 10. Webhook payload handling

Swapcard webhook payloads use `place` (not `location`), `begins_at`/`ends_at`
(snake_case), and `context.event_id` for the event reference. The handler is
robust to several payload shapes — see the field-fallback chains at the top
of [src/routes/webhooks.js](src/routes/webhooks.js).

Important quirks:
- **Event ID format**: Swapcard webhooks always send the canonical Relay
  base64 ID (e.g. `RXZlbnRfNDQ1OTY3Mw==`), even if the operator typed the
  short numeric form (`4459673`). `/load` normalizes saves to the canonical
  form (using `event.id` from the GraphQL response). For legacy rows saved
  under the short form, the webhook handler decodes the canonical ID,
  looks up by short form, and rewrites the row to canonical so the next
  webhook hits the fast path.
- **Participant enrichment**: webhook payloads contain participant IDs but
  no names or emails. The handler best-effort fetches `meeting.participants`
  via Swapcard GraphQL to get the name + email; if that fails (schema
  mismatch, auth, etc.) it falls back to creating the booking with empty
  attendee names, which is still better than failing.
- **Action gating**: any `meeting_*` action is processed; the handler then
  uses the *fetched meeting status* to decide whether to create or cancel
  the booking. This is more reliable than trying to enumerate Swapcard
  action names.

---

## 11. Running locally

```bash
npm install
npm run migrate    # ensures bridge.db exists + schema applied (idempotent)
npm run dev        # node --watch src/server.js
```

Visit `http://localhost:8080` and sign in with the default credentials
(`admin@zenspace.io` / `bridge2026`).

To exercise the webhook path locally without a real Swapcard, you'll need a
public tunnel (e.g. `cloudflared tunnel`, `ngrok`) — Swapcard cannot reach
your laptop directly. Set `PUBLIC_URL` to the tunnel URL so the admin UI
shows the right webhook URL to paste into Studio.

---

## 12. Deploying to Fly.io

The repo ships with a working `fly.toml`:

```toml
app = "swapcard-bridge"
primary_region = "bom"

[env]
  PORT = "8080"
  NODE_ENV = "production"
  DATABASE_PATH = "/data/bridge.db"

[[mounts]]
  source = "data"
  destination = "/data"
  initial_size = "1"  # Fly auto-creates the 1 GB volume on first deploy
                      # so a missing-volume misconfig can't silently land
                      # the DB on ephemeral container storage.
```

Standard deploy:

```bash
fly deploy -a swapcard-bridge
fly scale count 1 -a swapcard-bridge   # SQLite is single-writer, pin to one machine
```

If you scale to >1 machine, you'd need to either move off SQLite or accept
that each machine has its own independent volume — local-disk SQLite has
no multi-writer story. For this app the right answer is "stay at 1
machine"; see [PRODUCTION.md](PRODUCTION.md) §3-4.

> **History note:** This app was originally named `swapcard-zencore` in
> region `sjc`, then renamed to `swapcard-bridge` in `bom`. Fly volumes are
> scoped to `(app, region)` and don't migrate, so the rename effectively
> orphaned the old volume — and without `initial_size`, the new app booted
> with an empty `/data` on the container's writable layer, wiping the DB on
> every deploy. `scripts/fix-fly-volume.sh` automates the recovery of the
> old data plus the redeploy. With `initial_size` in place this failure
> mode is structurally impossible going forward.

---

## 13. Common operational issues

| Symptom (in logs / UI)                                                   | Cause                                                                                                                                  | Fix                                                                                                                                                                                                                  |
|--------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `webhook for unknown event`, `totalRows: 0`                              | DB is empty (volume not mounted, or never saved a mapping).                                                                            | Run `scripts/fix-fly-volume.sh`. With `initial_size` in `fly.toml`, future deploys auto-create the volume. See §12.                                                                                                  |
| `webhook for unknown event`, `totalRows > 0`                             | Mapping saved under a different event-ID format.                                                                                       | Auto-heals on first webhook (legacy fallback). Otherwise resave the mapping.                                                                                                                                         |
| `webhook signature did not verify`                                       | Webhook secret in Swapcard Studio doesn't match bridge's row.                                                                          | Reopen the connect modal and paste the bridge's secret into Studio.                                                                                                                                                  |
| `meeting_fetch_failed` (Swapcard GraphQL 4xx/5xx)                        | Swapcard API key invalid or schema mismatch.                                                                                           | Verify the API key works with the Swapcard developer-admin endpoint.                                                                                                                                                 |
| `booking blocked — connection not approved yet`                          | ZenCore connection still pending approval.                                                                                             | Approve the connection in ZenCore admin. Status flips to `approved` on next booking.                                                                                                                                 |
| Connect returns red "error" badge                                        | ZenCore says "third-party connection already exists".                                                                                  | Remove the existing connection in ZenCore admin, then click Connect again.                                                                                                                                           |
| Create-meeting modal: `No slot matches that window`                      | Requested date/start/end doesn't match any entry in `meetingSlotsAvailable`.                                                           | Use one of the same-day slots the modal lists in the error.                                                                                                                                                          |
| Create-meeting modal: `ZenCore connection is suspended` (`2 suspended`)  | Multiple stale third-party connections accumulated on ZenCore's side from previous Connect/Disconnect cycles. ZenCore enforces "ALL approved." | In ZenCore admin → meeting space → third-party connections, remove every suspended record. Then click Disconnect on the bridge row and Connect again. Mapping badge now auto-flips to `suspended` so you see this.|
| Create-meeting succeeds in Swapcard but ZenCore booking fails            | ZenCore rejected after the Swapcard mutation already ran — orphan meeting now occupies the slot.                                       | Either delete the orphan meeting in Swapcard Studio, or pick a different slot when retrying. There's no `cancelMeeting` mutation in the schema this app uses.                                                       |
| `/healthz` returns 503                                                   | SQLite file is missing/unreadable — usually the volume detached.                                                                       | Check `fly volumes list`. See §12 history note for the rename incident.                                                                                                                                              |
| ZenCore booking arrives but no Swapcard meeting appears                  | `/webhooks/zencore` couldn't match an EventPerson for any attendee — or the booking's slot/space isn't mapped.                          | Check the latest synced_booking row for that `zenspace_booking_id` — `last_error` will say `no_matching_event_persons` (add the attendees to the Swapcard event) or `no_matching_swapcard_slot` (book a real slot). |
| "Register ZenCore webhook" returns `webhook_registration_failed`         | ZenCore's webhook-registration endpoint isn't at `POST /api/v1/webhooks` or isn't exposed.                                              | The error response includes a `hint` with the manual URL — paste it into ZenCore admin instead, same pattern as the connection-status webhook (DOCS §9).                                                            |

The bridge logs aggressively (raw webhook body, signature prefixes, candidate
HMAC digests, ZenCore error bodies, etc.) — `fly logs -a swapcard-bridge`
should make root-causing easy.

---

## 14. Things this app does NOT do

- Multi-tenant isolation between admin users — there's a single hardcoded
  login. If you need multiple operators, swap in a real auth provider.
- Update an already-synced booking when the Swapcard meeting time changes.
  Swapcard sends `meeting_update`, but the handler currently skips when a
  booking already exists with `status='active'`. Adding update support
  means calling ZenCore's `PUT /api/v1/bookings/{id}` with the new slot.
- Bidirectional sync. Bookings cancelled in ZenCore are not propagated back
  to Swapcard.
- Rollback of an operator-initiated meeting when ZenCore rejects the
  booking. Swapcard's GraphQL schema this app talks to has no
  `cancelMeeting`/`deleteMeeting` mutation, so an orphan meeting is left
  in Swapcard for the operator to remove manually. See §13.
- Polling for ZenCore connection status — no public GET API. See §6.
  (The create-meeting flow updates `connection_status` reactively when a
  booking fails, but the bridge can't proactively poll.)
- High-availability storage. SQLite on a single Fly volume is fine for tens
  of events but not designed for multi-region failover. Litestream
  (see [PRODUCTION.md](PRODUCTION.md) §5) is the recommended lightweight
  durability story.
