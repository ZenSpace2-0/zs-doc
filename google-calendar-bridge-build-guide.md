# Building the Google Calendar Bridge with ZenCore

*A straightforward build guide: how the bridge connects ZenCore to third-party booking platforms (Deskpass, Upflex, LiquidSpace) using Google Calendar as the two-way link. ZenCore stays the single source of truth; the bridge keeps Google Calendar and ZenCore in sync in both directions.*

---

## The picture

```
┌──────────────┐   events    ┌───────────────┐  push out  ┌──────────────────┐  read   ┌──────────┐
│   ZenCore    │ ──────────► │  Sync bridge  │ ─────────► │ Google Calendar  │ ◄────── │ Platforms│
│  source of   │             │               │            │  one per space   │         │ Deskpass │
│    truth     │ ◄────────── │               │ ◄───────── │                  │ ──────► │  Upflex  │
└──────────────┘  validate   └───────────────┘  webhook   └──────────────────┘  write  └──────────┘
                  + confirm
```

- **ZenCore → bridge → Google:** when a booking is made in ZenCore, the bridge writes it to that space's Google Calendar, so the platforms see the slot as taken.
- **Google → bridge → ZenCore:** when someone books on a platform, it lands on the Google Calendar, the bridge picks it up and confirms it in ZenCore.

---

## Step 1 — Set up authentication

The bridge authenticates to ZenCore with a **per-organization API key**.

- Each customer (organization) issues an API key: `POST /api/v1/organizations/{organizationId}/api-keys`.
- The bridge sends it on every call as a header: `x-api-key: zsk_live_…`.
- Always include `organization_id` explicitly in requests so every call is clearly scoped to the right customer.

On the Google side, the bridge uses a Google account (per the chosen hosting model) with the Calendar API enabled and an HTTPS endpoint ready to receive notifications.

---

## Step 2 — Map spaces to calendars

For each customer, the bridge discovers their bookable spaces and creates one Google Calendar per space.

1. **List the spaces:** `GET /api/v1/meeting-spaces?organization_id=<uuid>&limit=100`
   Each space comes back with its id, name, `space_group.timezone`, and booking rules (min/max duration, advance-booking window, booking increment).
2. **Create a Google Calendar** for each space (`calendars.insert`), naming it after the space and setting its timezone to the space's group timezone.
3. **Store the link** — save `space_id ↔ google_calendar_id` in the bridge's own binding table. This mapping is the bridge's source of truth for which calendar belongs to which space.
4. **Share the calendar** with the third-party platform so it can read availability and write bookings.

---

## Step 3 — Seed initial availability

So the platforms show correct availability from day one, the bridge loads each space's existing bookings into its calendar.

1. **Read current bookings** for the space:
   `GET /api/v1/meeting-spaces/:id/availability?start_date=&end_date=`
   (returns existing bookings and blocked/unavailable times for the range).
2. **Write each one to the Google Calendar** as an event (`events.insert`), tagging it with the ZenCore booking id so the bridge recognizes its own events later.
3. **Take a baseline sync token** from Google (`events.list` returns a `nextSyncToken`) — the bookmark for future incremental reads.

---

## Step 4 — Subscribe to ZenCore booking events (ZenCore → Google)

The bridge listens for booking changes in ZenCore and mirrors them to Google Calendar.

1. **Subscribe** to ZenCore's booking event stream for the organization — events fire on `booking.created`, `booking.updated`, `booking.cancelled`, and `booking.deleted`. Each event carries the booking's org, space, space group, timezone, times, status, and origin.
2. **On each event, update the calendar:**
   - `created` → `events.insert` on the space's calendar.
   - `updated` → update the matching event.
   - `cancelled` / `deleted` → remove the event.
3. **Skip echoes** — if the event is for a booking the bridge itself created (via its origin marker / stored id), it's already on the calendar, so ignore it.

*(A "changed since" query on the bookings list is also available as a catch-up/reconciliation path: `GET /api/v1/bookings?organization_id=&changed_since=…`.)*

---

## Step 5 — Receive platform bookings (Google → ZenCore)

When someone books on a platform, it appears on the Google Calendar. The bridge validates and confirms it in ZenCore.

1. **Google notifies the bridge** that the calendar changed (via a watch channel registered with `events.watch`). The notification just says "something changed."
2. **The bridge pulls what changed** using its stored sync token (`events.list?syncToken=…`), which returns only the new/edited/removed events plus a fresh token.
3. **For each new external event:**
   - If it's one of the bridge's own events (tagged), skip it.
   - Otherwise, it's a platform booking. **Pre-check it against the space's rules** (duration limits and advance-booking window from Step 2), then **create it in ZenCore**:
     `POST /api/v1/bookings` with `organization_id`, `space_id`, start/end times (UTC), `booking_source: "external"`, and the calendar event id in `external_calendar_id` (so repeats are automatically de-duplicated).
   - **Confirmed** → ZenCore returns the booking; mark the calendar event confirmed.
   - **Rejected** → read the error code, remove the event from the calendar, and log it.

---

## Step 6 — Keep the connection alive

Two lightweight background jobs keep sync healthy:

- **Renew Google watch channels** before they expire (they last ~7 days) so inbound notifications never stop.
- **Periodic reconciliation** — occasionally do a full read of each space's bookings (`availability` endpoint) and compare against the calendar, to catch anything missed during an outage. This also picks up deletions.

---

## Handling recurring bookings

ZenCore treats each booking as an individual record. So the bridge expands a recurring calendar event into individual instances and creates one ZenCore booking per instance, stamping them all with a shared series id (`recurring_series_id`) so they can be managed together.

---

## The ZenCore APIs the bridge uses

| Purpose | Call |
|---|---|
| Issue / use auth | `POST /api/v1/organizations/{id}/api-keys`, then `x-api-key` header |
| List a customer's spaces + rules + timezone | `GET /api/v1/meeting-spaces?organization_id=` |
| Read existing bookings / availability for a range | `GET /api/v1/meeting-spaces/:id/availability?start_date=&end_date=` |
| Create a booking from a platform | `POST /api/v1/bookings` (with `external_calendar_id`, `booking_source: "external"`) |
| Catch-up / reconcile changed bookings | `GET /api/v1/bookings?organization_id=&changed_since=` |
| Subscribe to booking events | ZenCore booking event subscription (`booking.created / updated / cancelled / deleted`) |

## The Google Calendar APIs the bridge uses

| Purpose | Call |
|---|---|
| Create a calendar per space | `calendars.insert` |
| Give the platform access | `acl.insert` |
| Watch for platform bookings | `events.watch` |
| Baseline read + first sync token | `events.list` |
| Pull only what changed | `events.list?syncToken=` |
| Write a ZenCore booking out | `events.insert` |
| Remove a cancelled/rejected event | `events.delete` |

---

## In short

1. **Authenticate** per organization with an API key.
2. **Map** each ZenCore space to its own Google Calendar.
3. **Seed** existing bookings so availability is correct from the start.
4. **Mirror ZenCore → Google:** subscribe to booking events, write them to the calendar.
5. **Mirror Google → ZenCore:** catch platform bookings, validate and confirm them in ZenCore.
6. **Maintain:** renew watch channels and reconcile periodically.

ZenCore stays authoritative throughout — every booking, whichever channel it came from, becomes real only once ZenCore confirms it.
