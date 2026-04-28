# Zenspace Meeting Room Display — User Flow & Testing Guide

This document describes the application's user flows and provides a structured testing guide for QA and for understanding how the app works. It reflects the current state of the app: a **Physical Spaces**-led admin console with realtime-driven onboarding (displays, IoT gateways, adapters), organization-scoped API keys, and a public magic-link booking-access experience.

---

## 1. Application Overview

**Zenspace Meeting Room Display** is a React (Vite + TypeScript) web app for managing meeting room displays, physical spaces, IoT gateways, and adapters under Zenspace organizations. Users sign in with email OTP, pick an organization, then primarily manage:

- **Physical Spaces** — physical locations (pod/floor/zone) and their virtual-pod mapping timeline. The detail page is the main hub for devices/adapters, webhook logs, and credential attempt history.
- **Meeting Room Displays** — displays that show room status (create, verify, link/unlink to a ZenSpace meeting space, configure, monitor webhooks/screenshots).
- **IoT Gateway** — gateways that connect IoT devices (create, verify, manage).
- **API Keys** — organization-scoped API keys with per-permission flags. Plaintext shown once at create/rotate.

The app uses **React Router**, **Redux Toolkit**, and **realtime updates** via Socket.IO (display status, webhooks, IoT verification, adapter status).

The app also exposes a **public magic-link page** at `/magic/:token` (Magic Link) that lets a guest see their booking and interact with attached devices via embedded adapter iframes. It does not require login.

---

## 2. Route Map

| Route                                         | Layout           | Description                                                                                    |
| --------------------------------------------- | ---------------- | ---------------------------------------------------------------------------------------------- |
| `/auth`                                       | Public           | Login (email + OTP)                                                                            |
| `/organization`                               | Organization     | List organizations; select one to enter the app                                                |
| `/physical-spaces`                            | Protected        | List physical spaces (requires `?orgId=<id>`)                                                  |
| `/physical-spaces/:id`                        | Protected        | Physical space detail — mappings, virtual pod, devices/adapters, webhook logs, attempt history |
| `/physical-spaces/webhook-logs/:logId`        | Protected        | Physical-space webhook log detail                                                              |
| `/physical-spaces/credential-logs/:requestId` | Protected        | Booking-access credential attempt log detail                                                   |
| `/meeting-room-displays`                      | Protected        | List meeting room displays                                                                     |
| `/meeting-room-displays/:id`                  | Protected        | Display detail (config, webhooks, screenshots, etc.)                                           |
| `/meeting-room-displays/webhook-logs/:logId`  | Protected        | Display webhook log detail                                                                     |
| `/IoT-Gateway`                                | Protected        | List IoT gateways                                                                              |
| `/iot-gateways/:id`                           | Protected        | IoT gateway detail                                                                             |
| `/api-keys`                                   | Protected        | List organization API keys (create / rotate / revoke)                                          |
| `/magic/:token`                               | Public (no auth) | Magic Link — guest booking access page                                                         |
| `*` (any other)                               | —                | Redirects to `/auth`                                                                           |

**Layouts and guards:**

- **PublicRoute** (`/auth`) — if already authenticated, redirects to `/organization`.
- **OrganizationProtectedRoute** (`/organization`) — auth required; renders `OrganizationLayout`.
- **ProtectedRoute** (every protected app route) — auth required; renders `ProtectedLayout` which wraps content in `OrganizationProvider` and the main `AppSidebar`. The provider reads `orgId` from the query string and loads org context into Redux before rendering the page.
- The `/magic/:token` route is plain (no guard), and the page fetches its data directly via `joinBackendUrl()` rather than through the authenticated API client.

**Sidebar (AppSidebar) entries (top to bottom):**

- Physical Spaces (`/physical-spaces`)
- IoT Gateway (`/IoT-Gateway`, also matches `/iot-gateways/:id`)
- Meeting Room Displays (`/meeting-room-displays`)
- API Keys (`/api-keys`)

---

## 3. User Flows (Step-by-Step)

### 3.1 Authentication Flow (Login)

**Entry:** User opens app → default redirect to `/auth`.

1. **Login page** (`/auth`)
   - User sees "Sign in to Meeting Room Display" and email input.
   - User enters work email → clicks **"Send magic code"**.
2. **OTP request**
   - App calls `requestOTPSevice(email)`.
   - On success: UI switches to OTP step; message "We've sent a verification code to **email**".
   - User can click **"Use a different email"** to go back to email step.
3. **OTP verification**
   - User enters 6-digit OTP → **"Verify & continue"**.
   - App calls `verifyOTPSevice(session_id, otp)`.
   - On success: `checkAuth()` runs, then redirect to **`/organization`**.
4. **Already logged in**
   - If `isAuthenticated` and user lands on `/auth`, they are redirected to `/organization`.

**Auth contract:** token is stored in `localStorage.auth_token`. A 401 from any authenticated request clears the token and pushes the user back to `/auth`.

**Testing focus:** Invalid email, OTP request failure, wrong/expired OTP, "Use a different email", and redirect when already authenticated.

---

### 3.2 Organization Selection Flow

**Entry:** After login, user is on `/organization`.

1. **Organization list**
   - Page title: "Select an organization".
   - Data from `/auth/organizations`; rendered as cards.
   - Each card shows: name, email, address, website, phone, number of spaces, status (Active/Inactive/Suspended), role, created at.
   - Card action menu (three-dot) is only visible to `super_admin` users.
2. **Select organization**
   - Clicking a card calls `navigate('/physical-spaces?orgId=<org.id>')` — Physical Spaces is now the landing module after org selection.
3. **Organization context**
   - `OrganizationProvider` (used inside `ProtectedLayout`) reads `orgId` from URL.
   - It fetches org via `getOrganizationByIdAction(orgId)` and stores it in Redux (`organization`).
   - All protected pages use this `organization` for API `org_id`, breadcrumbs, timezone formatting, etc.
4. **Switch organization**
   - From any protected page: sidebar **Team Switcher** (avatar + org name) → **"Switch Organization"** → navigates to `/organization` so user can pick another org.

**Testing focus:** No orgs (empty state copy), API failure, missing/invalid `orgId` on a protected route, and "Switch Organization" from different pages.

---

### 3.3 Physical Spaces Flow (primary module)

**Entry:** `/physical-spaces?orgId=<id>` — selected directly from the organization card.

1. **Physical Spaces list**
   - Fetches from `/physical-spaces?org_id=<org_id>`.
   - Cards/table columns: Name + description, Status (Active/Inactive), Created at.
   - Actions: **Create Space**, per row **View** (primary), **Edit**, **Delete**.
2. **Create / edit space**
   - **Create Space** opens `PhysicalSpaceForm` (modal). Required: `name` (≤200). Optional: description.
   - **Edit** opens the same form pre-filled.
3. **Open space detail**
   - Click row/card or **View** → `/physical-spaces/:id?orgId=...` (URL is built via `appendOrgIdToUrl`).
4. **Physical space detail** (`/physical-spaces/:id`)
   - **Hero metrics:** Status, Devices count, Active Pod (current virtual mapping), Last Updated.
   - **Pill nav sections:**
     - **Mappings** — timeline of virtual-pod links with `start_at` → `end_at` and status (`active`, `scheduled`, `expired`). Has both a calendar card view (`MappingCalendar`) and a table view. Each row supports **Unlink** (with confirm).
     - **Virtual Pod** (only shown if a mapping is currently active) — details about the linked virtual meeting space: status timeline (previous/current/next), space metadata (group, capacity, availability window, amenities), and current booking organizer/start/end if any.
     - **Devices** (labelled "Devices", powered by `AdapterList`) — third-party adapter devices linked to this physical space, with realtime status from `useRealtimeAdapters`. Onboard new adapters via `AdapterOnboardForm`.
     - **Webhook Logs** — list of physical-space webhook logs (`/webhooks/physical-space/logs?physical_space_id=<id>`). Click a row to open `/physical-spaces/webhook-logs/:logId`. Realtime updates via `useRealtimeWebhooks`.
     - **Attempt History** — append-only booking-access credential attempt log (`CredentialLogsList`). Click a row to open `/physical-spaces/credential-logs/:requestId`.

**Testing focus:** create/edit/delete space, mapping calendar + table, unlink mapping, virtual-pod state, adapter list (realtime status), webhook log opening + realtime, credential attempt log opening.

---

### 3.4 Meeting Room Displays Flow

**Entry:** Sidebar → "Meeting Room Displays" → `/meeting-room-displays` (with org context).

1. **Displays list**
   - Fetches from `/displays?org_id=<org_id>`.
   - Columns: Name, Verified status (Not verified / Verified / Linked), Created at, Actions.
   - Actions: **Create**; per row: **View**, **Link to ZenSpace** (verified + not linked), **Unlink from ZenSpace** (linked), **Edit**, **Delete**.
2. **Create display**
   - **Create** → opens `MeetingRoomDisplayForm` (modal). Required: `physical_space_id`. Submit → `createMeetingRoomDisplayService` → verification step (code shown, to enter on the physical device).
3. **Verify display**
   - User completes verification on the physical device.
   - `useRealtimeDisplayStatus({ autoRefresh: true })` listens for `mrd:display.status.updated` and auto-closes the verification modal once `is_verified && !zenspace_meeting_space_id`. The list refreshes automatically.
4. **Link to ZenSpace**
   - From list: **Link to ZenSpace** opens `LinkZenSpaceForm`.
   - User selects a meeting space (async paginated select). If the space already has a display, the form offers a "Replace existing link" option.
   - Required: `zenspace_api_key`, `meeting_space_id` (UUID). Submit → `linkZenSpaceService` → list refreshes; row shows "Linked".
5. **Unlink from ZenSpace**
   - **Unlink from ZenSpace** → `UnlinkZenSpaceModal` → confirm → unlink → list refresh.
6. **View display detail** (`/meeting-room-displays/:id`)
   - Tabs / sections: Overview, configuration, webhooks, screenshots, notifications.
   - Actions: edit display, upload image/logo, link/unlink ZenSpace, regenerate token, request screenshot, open webhook log detail (`/meeting-room-displays/webhook-logs/:logId`).
   - Realtime: `useRealtimeWebhooks({ displayId })` keeps webhook lists fresh.
7. **Edit display** — opens `MeetingRoomDisplayForm` pre-filled with `editData`.
8. **Delete display** — confirmation modal → `deleteMeetingRoomDisplayService` → list refresh.

**Form invariants:**

- `booking_url` (display config) must be a valid URL.
- `meeting_space_id` (link form) must be a UUID.

**Testing focus:** Create → realtime verify → link → unlink, edit, delete, screenshots/notifications/webhook logs from detail page.

---

### 3.5 IoT Gateway Flow

**Entry:** Sidebar → "IoT Gateway" → `/IoT-Gateway`.

1. **Gateways list**
   - Fetches gateways for current org.
   - Onboarding state derived via `resolveGatewayOnboardingState`.
   - Actions: **Create**, **Edit**, **Delete**, **View** (navigate to detail).
2. **Create gateway**
   - **Create** → `IoTGatewayForm`. Required: `name` (≤100). Optional: `org_id`, `tailscale_config_id`, `adapter_type`. Submit → create → verification step (similar to displays).
   - `useRealtimeIotVerification` listens for `mrd:gateway.status.updated`. When the just-created gateway verifies, the verification modal closes and the list hard-refreshes.
3. **View gateway detail** (`/iot-gateways/:id`)
   - Shows gateway identity, network info, health, locks, registered locks (admin endpoints).
4. **Edit / delete**
   - **Edit** opens form with existing data; on save, list refreshes.
   - **Delete** → confirm → delete → refresh.

**Testing focus:** Create, realtime verification (modal auto-close), edit, delete, gateway detail loading.

---

### 3.6 API Keys Flow

**Entry:** Sidebar → "API Keys" → `/api-keys`.

1. **Keys list**
   - Fetches from `/org-api-keys?org_id=<org_id>`.
   - Columns: Name, Permissions (R/W/U/D summary), Status (Active/Inactive), Expiration, Last Used, Created, Revoked At, Actions.
2. **Create key**
   - **Create** opens `OrgApiKeyForm`. Required: `name` (≤200), `permissions` object. Optional: `expires_at`.
   - On success the list shows a **one-time secret modal** with the plaintext key. The key is **never** retrievable again — closing the modal discards the plaintext from memory.
3. **Rotate key**
   - **Rotate** (primary action button) opens a rotate form. On success: old key is revoked, new plaintext key is shown via the secret modal under a different title/copy ("The old key is now revoked…").
4. **Revoke key**
   - **Revoke** → confirm via `AlertModal` → `revokeOrgApiKeyService` → list refresh. Only available while the key is active.

**Critical UX guardrail:** plaintext API keys are visible exactly once after create/rotate, and only inside `OrgApiKeySecretModal`. Do not expose them anywhere else.

**Testing focus:** create, copy plaintext from modal, ensure plaintext is no longer accessible after closing, rotate, revoke, expired-key display formatting.

---

### 3.7 Public Magic Link Flow (`/magic/:token`)

**Entry:** Guest opens `/magic/:token` directly (no login required).

1. **Bootstrap**
   - Page calls `getMagicLinkContext(token)` (raw `fetch` against `VITE_BACKEND_URL` via `joinBackendUrl`, not the authenticated `zenspaceApi`).
   - Loading state: animated shield with "Verifying your access...".
2. **Ready state**
   - Banner shows booking space name and a formatted booking window (`start_time` → `end_time` in `space_timezone`).
   - Devices are grouped into sections (locks, wifi, fan, AC, lights, sensors, cameras, other).
   - Each device card may render an **adapter iframe** (`adapter_embed_html`) for in-page interaction, or an error state if `adapter_embed_error` is present. Iframes use `sandbox="allow-scripts"`.
3. **Error state**
   - Invalid/expired token, network error, or backend rejection → error icon, server message, **Try again** button (re-runs bootstrap).
4. **Empty state**
   - If `context.devices` is empty: "No devices are currently available for this booking."

**Critical guardrails:**

- This page must remain reachable without an authenticated session.
- Do not log token, embed HTML, or any guest credentials to the console or analytics.
- Do not loosen the iframe sandbox.

**Testing focus:** valid token (booking + devices), expired/invalid token, server error → retry, no devices, locked vs unlocked timezone formatting, iframe rendering and adapter error fallback.

---

### 3.8 Webhook Logs Flow

There are two webhook log surfaces:

- **Display webhook logs**
  - Source: `/displays/:id/webhook-logs` and the global `/webhooks/logs`.
  - Detail route: `/meeting-room-displays/webhook-logs/:logId`.
  - Realtime: `useRealtimeWebhooks` consumes `mrd:webhook.received`.
- **Physical-space webhook logs**
  - Source: `/webhooks/physical-space/logs?physical_space_id=<id>`.
  - Detail route: `/physical-spaces/webhook-logs/:logId`.
  - Realtime: `mrd:physical-space.webhook.processed`.

Both detail pages show full payload and metadata.

**Testing focus:** open a log from each surface, deep-link to a `logId`, realtime list updates without page refresh.

---

### 3.9 Booking Access Credential Logs Flow

**Entry:** Physical Space detail → **Attempt History** section → click a row.

1. **Credential attempt list** is rendered by `CredentialLogsList` inside the Attempt History section.
2. Clicking a row navigates to `/physical-spaces/credential-logs/:requestId`.
3. **Detail page** shows status, type (LOCK/WIFI), timing metrics, and any error state for the credential generation attempt — for diagnosing per-type retry orchestration backed by `booking_access_credential_state` on the backend.

**Testing focus:** open detail from Attempt History, deep-link a `requestId`, status/type formatting via `formatCredentialStatus` / `formatCredentialType`.

---

### 3.10 Logout Flow

**Entry:** Any authenticated page (organization or protected).

1. User opens **sidebar footer** → **NavUser** (avatar + name).
2. Dropdown: user info + **"Log out"**.
3. On click: `localStorage.removeItem('auth_token')`, `logoutUserService()`, `checkAuth()`, `navigate('/auth')`.
4. User lands on the login page; protected routes are no longer accessible.

**Testing focus:** logout from organization page vs from a protected page; after logout, hitting `/organization` or `/physical-spaces` redirects to `/auth`.

---

## 4. Navigation Summary

```
/auth
  └─ (success) → /organization

/organization
  └─ (click org card) → /physical-spaces?orgId=<id>

Protected (orgId in URL):
  /physical-spaces?orgId=<id>                          → list
  /physical-spaces/:id                                 → detail (mappings, virtual pod, devices/adapters,
                                                          webhook logs, attempt history)
  /physical-spaces/webhook-logs/:logId                 → physical-space webhook log detail
  /physical-spaces/credential-logs/:requestId          → credential attempt log detail

  /meeting-room-displays                               → list (Create, Link/Unlink, Edit, Delete, View)
  /meeting-room-displays/:id                           → detail (config, webhooks, screenshots, ...)
  /meeting-room-displays/webhook-logs/:logId           → display webhook log detail

  /IoT-Gateway                                         → gateways list
  /iot-gateways/:id                                    → gateway detail

  /api-keys                                            → API key list (Create, Rotate, Revoke;
                                                          plaintext shown once)

Public (no auth):
  /magic/:token                                        → guest booking access page

Sidebar "Switch Organization"                          → /organization
NavUser "Log out"                                      → /auth
```

---

## 5. Realtime / WebSocket Surface

All hooks connect via Socket.IO to `${VITE_WEBSOCKET_SERVICE_URL}/realtime` (or a configured namespace path) using JWT handshake auth with `localStorage.auth_token`.

| Hook                         | Page(s)                               | Join payload                    | Server event                                                   |
| ---------------------------- | ------------------------------------- | ------------------------------- | -------------------------------------------------------------- |
| `useRealtimeDisplayStatus`   | Displays list, display detail         | `{ display_id }` (optional)     | `mrd:display.status.updated`                                   |
| `useRealtimeWebhooks`        | Display detail, physical-space detail | `{ display_id }` or none        | `mrd:webhook.received`, `mrd:physical-space.webhook.processed` |
| `useRealtimeIotVerification` | IoT Gateways list                     | `{ iot_gateway_id }` (optional) | `mrd:gateway.status.updated`                                   |
| `useRealtimeAdapters`        | Physical space detail (Devices)       | `{ physical_space_id }`         | adapter status events                                          |

If the WebSocket service is unreachable, the app should still be usable (manual refresh / `triggerHardRefresh()` covers list refresh on mutations).

---

## 6. Testing Checklist (QA / Manual Testing)

Use this as a structured checklist for release or regression testing.

### 6.1 Authentication

- [ ] Open app → redirects to `/auth`.
- [ ] Submit invalid email → appropriate error.
- [ ] Submit valid email → OTP sent; UI shows OTP step.
- [ ] "Use a different email" → back to email step.
- [ ] Wrong OTP → error; can retry.
- [ ] Correct OTP → redirect to `/organization`.
- [ ] Already logged in: open `/auth` → redirect to `/organization`.

### 6.2 Organization

- [ ] Organization list loads; cards show correct data.
- [ ] Action menu (three-dot) is hidden for non-super-admins.
- [ ] Select organization → navigates to `/physical-spaces?orgId=<id>`; org name shown in sidebar.
- [ ] Missing/invalid `orgId` on protected route → graceful empty/error state.
- [ ] "Switch Organization" from sidebar → `/organization`; selecting a new org updates context everywhere.

### 6.3 Physical Spaces (primary)

- [ ] List loads (cards + table view).
- [ ] Create Space → form validation (name required) → appears in list.
- [ ] Edit Space → form pre-filled → save reflects.
- [ ] Delete Space → confirm modal → row removed.
- [ ] Open detail → hero metrics correct; Status, Devices, Active Pod, Last Updated.
- [ ] Mappings — table + calendar render; status badges (active/scheduled/expired) correct; Unlink confirms and removes mapping.
- [ ] Virtual Pod section appears only when an active mapping exists; status timeline (previous/current/next), booking details when present.
- [ ] Devices (Adapters) — list loads; realtime status updates without refresh; onboard new adapter form submits and the new adapter appears.
- [ ] Webhook Logs — list loads; clicking row opens `/physical-spaces/webhook-logs/:logId`; realtime updates reflect new entries.
- [ ] Attempt History — list loads; clicking row opens `/physical-spaces/credential-logs/:requestId`.

### 6.4 Meeting Room Displays

- [ ] List loads; create opens form; required `physical_space_id` enforced.
- [ ] Verification code displayed; verifying on a real device flips status to "Verified" via realtime (no manual refresh).
- [ ] Link to ZenSpace: meeting space async select; submit shows "Linked"; "Replace existing link" works when target space already has a display.
- [ ] Unlink → list shows "Verified" again.
- [ ] View → detail page tabs and data correct; image/logo upload; regenerate token; request screenshot.
- [ ] Webhook logs accessible from detail; realtime updates.
- [ ] Edit → updates reflect in list/detail.
- [ ] Delete → confirm → removed.

### 6.5 IoT Gateway

- [ ] List loads.
- [ ] Create gateway → verification step appears; realtime verification closes modal and refreshes list.
- [ ] View gateway detail → identity, network, health, locks render.
- [ ] Edit → save → list/detail updated.
- [ ] Delete → confirm → removed.

### 6.6 API Keys

- [ ] List loads with permissions summary, status, expiration formatting.
- [ ] Create key → secret modal shows plaintext exactly once.
- [ ] Closing the secret modal makes plaintext unrecoverable.
- [ ] Rotate key → new plaintext shown; old key marked revoked in list.
- [ ] Revoke key → status flips to inactive; revoke option no longer offered.
- [ ] Expired keys render in red.

### 6.7 Public Magic Link

- [ ] Valid `/magic/:token` loads booking + devices; sections rendered by category; embed iframes load with `sandbox="allow-scripts"`.
- [ ] Adapter error state rendered when `adapter_embed_error` present.
- [ ] Invalid / expired token → error state with "Try again".
- [ ] No devices → empty-state message.
- [ ] Page works without an authenticated session (incognito).

### 6.8 Webhook Log Surfaces

- [ ] Display webhook log: open from `/meeting-room-displays/:id`; deep link `/meeting-room-displays/webhook-logs/:logId` works.
- [ ] Physical-space webhook log: open from `/physical-spaces/:id`; deep link `/physical-spaces/webhook-logs/:logId` works.
- [ ] Realtime updates push new rows in both surfaces without manual refresh.

### 6.9 Booking-Access Credential Logs

- [ ] Attempt History tab on physical-space detail loads.
- [ ] Click row → `/physical-spaces/credential-logs/:requestId` shows status, type, timings, errors.

### 6.10 Logout & Access Control

- [ ] Log out → token cleared; redirect to `/auth`.
- [ ] After logout, visiting any protected route redirects to `/auth`.
- [ ] Triggering a 401 from any authenticated request also clears the token and redirects to `/auth`.

### 6.11 General

- [ ] Sidebar collapse/expand works; active route is highlighted (including subpaths like `/iot-gateways/:id` mapping to "IoT Gateway").
- [ ] Toasts appear for success/error on create, update, delete (Sonner).
- [ ] Loading skeletons + empty states render correct copy and CTAs.
- [ ] `triggerHardRefresh` causes list views to refetch after mutations.

---

## 7. Key Technical Points (For Understanding)

- **Auth:** `localStorage.auth_token` drives `useAuth()` / `checkAuth()` and protected-route redirects. 401 responses clear the token and route to `/auth`.
- **Organization context:** `OrganizationProvider` reads `?orgId=` and loads org into Redux (`organization` slice). Protected pages read it from there.
- **API client:** `zenspaceApi.request()` (in `src/common/helpers/api.ts`) returns a parsed `ApiResponse<T>`; non-OK HTTP responses are returned (not thrown) — call sites check `res?.success`. The public magic-link page is the exception — it uses raw `fetch` via `joinBackendUrl()`.
- **Redux slices:** `common`, `meetingRoomDisplay`, `webhookLogs`, `organization`, `IoTGateway`, `physicalSpace`, `bookingAccessCredential`. `common.hardRefresh` is the cross-feature invalidation signal — `ListViewTable` refetches when it changes.
- **Realtime:** Socket.IO via `VITE_WEBSOCKET_SERVICE_URL` with JWT handshake on `/realtime`. Relevant hooks listed in §5.
- **List views:** Most lists use `ListViewTable` with a shared URL contract (`page`, `limit`, `sortBy`, `sortOrder`, `search`, plus direct filter params).
- **Plaintext API keys:** shown exactly once in `OrgApiKeySecretModal` after create or rotate. Do not log, store, or surface elsewhere.
- **Magic link:** raw `fetch` to `/magic-links/:token/context`; iframe sandbox is `allow-scripts` only. The route must remain unauthenticated.
- **Backends:** Two — `ZenCore` (REST API at `VITE_BACKEND_URL`) and `ZenEdge` (realtime at `VITE_WEBSOCKET_SERVICE_URL`). See `CLAUDE.md` for terminology and contracts.

---

## 8. Document History

| Date       | Change                                                                                                                                                                                                                                                                                                   |
| ---------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 2025-03-09 | Initial user flow and testing guide.                                                                                                                                                                                                                                                                     |
| 2026-04-28 | Major update: Physical Spaces is now the primary module and post-login landing; documented adapters, API Keys, Magic Link, physical-space webhook + credential attempt log surfaces; updated sidebar; added realtime hook surface and updated testing checklist. Removed Spaces/Locks/WiFi from the doc. |

For more on setup and running the app, see the project **README** and `CLAUDE.md`.
