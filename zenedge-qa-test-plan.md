# Zenspace Meeting Room Display — QA Test Plan

This document is the QA test plan for `zs-meeting-room-display` (the React/Vite admin frontend for SpaceOS Automation). It is meant to be used directly by QA engineers as a structured checklist for release validation and regression testing.

For end-to-end user-flow narratives, see [USER_FLOW_AND_TESTING.md](USER_FLOW_AND_TESTING.md). This document focuses on **what to test, how to test it, and how to report results**.

---

## 1. Purpose & Scope

### In scope
- Authenticated admin console: organizations, physical spaces, meeting room displays, IoT gateways, IoT devices (locks/wifi), org API keys.
- Public magic-link guest experience (`/magic/:token`).
- Realtime updates over Socket.IO (display status, webhook freshness, gateway verification, device status).
- ZenCore REST integration (auth, CRUD, list views).
- Cross-cutting UX: list-table contract, breadcrumbs, toasts, modals, organization scoping, sidebar.

### Out of scope
- ZenEdge backend internals (see `zs-mrd-backend` test plans).
- ZenCore platform tests.
- Device firmware / display-client app testing.
- Load / performance benchmarking (separate effort).

### Release exit criteria
- All P0/P1 test cases pass.
- No open P0/P1 defects.
- Regression checklist (Section 11) signed off.
- Smoke pass on dev **and** prod build (`npm run build && npm run preview`).

---

## 2. Test Environments

| Environment | Frontend URL | `VITE_BACKEND_URL` | `VITE_WEBSOCKET_SERVICE_URL` |
|---|---|---|---|
| Dev | local `npm run dev` | `https://mrd-api-dev.zenspaceiot.com/api` | `https://websocket-service.zenspace.io` |
| Staging | TBD | dev API | staging websocket |
| Prod | TBD | `https://api-automation-spaceos.zenspace.io/api` | `https://websocket-service.zenspace.io` |

**Build commands**
- `npm run dev` — dev server (Vite, HMR).
- `npm run build:dev` — production build pointing at dev API.
- `npm run build` — production build pointing at prod API.
- `npm run preview` — serve a built bundle locally (use to validate SPA rewrites).
- `npm run lint` — ESLint pass; must be clean before release.

**Browsers (must pass)**
- Chrome (latest)
- Edge (latest)
- Safari (latest, including macOS + iOS for magic-link page)
- Firefox (latest)

**Device coverage**
- Desktop 1920×1080 and 1366×768.
- Tablet (iPad portrait + landscape) — magic-link page especially.
- Mobile (375×667 minimum) — magic-link page only; admin console is desktop-first.

---

## 3. Prerequisites & Test Data

Before starting a full pass, QA must have:

| Item | Why needed |
|---|---|
| Test user email with OTP access | Auth flow |
| At least one **super_admin** test user | Validates organization card action menu |
| At least one non–super_admin user | Validates that the menu is hidden |
| 2+ test organizations the user belongs to | Org switcher, scoping |
| A physical space with at least one virtual-pod mapping | Detail page, mapping CRUD |
| A meeting room display (verified + unverified) | Display lifecycle, link/unlink |
| A meeting space ID + ZenSpace API key | Link-display flow |
| An IoT gateway (verified + unverified) | Gateway lifecycle |
| At least one lock and one wifi device attached to a gateway | Device CRUD + actions |
| A booking that triggers webhooks against a mapped physical space | Webhook + booking-access flow |
| A valid magic-link URL (`/magic/:token`) for an active booking | Public guest flow |
| An org with an existing API key (and capability to create more) | Org API keys |

Record all test-data IDs in the test run sheet so failed cases can be reproduced.

---

## 4. Test Case Priority Levels

| Priority | Meaning |
|---|---|
| **P0** | Blocker — release cannot ship if this fails. Includes auth, org switching, magic-link page, display verification. |
| **P1** | Critical — must pass; only deferred with explicit product sign-off. Includes CRUD on every entity, list filtering, webhook log access. |
| **P2** | Important — defects logged; release may proceed with workaround. UI polish, non-critical empty states, sort order. |
| **P3** | Minor — log and fix in next iteration. Cosmetic, copy, spacing. |

---

## 5. Feature Test Cases — Authentication

### TC-AUTH-001 (P0) — Successful OTP login
**Steps**: Navigate to `/auth` → enter valid email → click **Send magic code** → enter received OTP → click **Verify & continue**.
**Expected**: Redirect to `/organization`; `auth_token` present in `localStorage`.

### TC-AUTH-002 (P1) — Invalid email format
**Steps**: Enter `not-an-email` → click Send.
**Expected**: Inline Zod validation error; no network request fired.

### TC-AUTH-003 (P1) — OTP request failure
**Steps**: Enter a non-existent email → Send.
**Expected**: Toast error; stays on email step.

### TC-AUTH-004 (P1) — Wrong OTP
**Steps**: Submit a 6-digit wrong OTP.
**Expected**: Toast error; stays on OTP step; can retry.

### TC-AUTH-005 (P1) — Resend / change email
**Steps**: After receiving OTP, click **Use a different email**.
**Expected**: Returns to email step; OTP state cleared.

### TC-AUTH-006 (P0) — Already authenticated bypass
**Steps**: While logged in, navigate to `/auth`.
**Expected**: Redirect to `/organization`.

### TC-AUTH-007 (P0) — 401 token clear
**Steps**: Manually corrupt `auth_token` in DevTools → reload a protected page.
**Expected**: Token removed; redirect to `/auth`.

### TC-AUTH-008 (P1) — Logout
**Steps**: Use sidebar logout (Team Switcher menu).
**Expected**: Token cleared; redirect to `/auth`.

---

## 6. Feature Test Cases — Organizations

### TC-ORG-001 (P0) — List render
**Steps**: Land on `/organization` post-login.
**Expected**: All orgs the user belongs to are shown as cards with name, email, address, website, phone, space count, status, role, created date.

### TC-ORG-002 (P0) — Select organization
**Steps**: Click a card.
**Expected**: Navigates to `/physical-spaces?orgId=<id>`; sidebar shows org name; subsequent API calls include the org context.

### TC-ORG-003 (P1) — super_admin action menu visibility
**Steps**: Compare same view as super_admin and as regular user.
**Expected**: Three-dot menu visible only for super_admin.

### TC-ORG-004 (P1) — Switch organization
**Steps**: Sidebar Team Switcher → **Switch Organization**.
**Expected**: Returns to `/organization`. Selecting another org rehydrates context cleanly (no stale lists from previous org).

### TC-ORG-005 (P1) — Missing `orgId` query param
**Steps**: Navigate to `/physical-spaces` (no `?orgId=`).
**Expected**: User is held by `OrganizationProvider` until org context resolves (or routed back to `/organization` if unresolvable). No silent error.

---

## 7. Feature Test Cases — Physical Spaces

### TC-PS-001 (P1) — List + pagination
**Steps**: Open `/physical-spaces`. Page through results.
**Expected**: `ListViewTable` honors `page`, `limit`, `sortBy`, `sortOrder`, `search` query params. URL reflects state.

### TC-PS-002 (P1) — Create physical space
**Steps**: Click **Create** → fill `name` (required, ≤200 chars) → submit.
**Expected**: Toast success; new row appears (`hardRefresh` fires); modal closes.

### TC-PS-003 (P1) — Edit physical space
**Steps**: Open detail → edit name/description.
**Expected**: Persists; detail reflects change.

### TC-PS-004 (P2) — Validation
**Steps**: Submit with empty name or name >200 chars.
**Expected**: Zod inline error; submit disabled / blocked.

### TC-PS-005 (P1) — Create virtual-pod mapping
**Steps**: On detail page, add mapping with `virtual_meeting_space_id`, `start_at`, `end_at`.
**Expected**: Validation rejects `end_at` ≤ `start_at`. Mapping appears in list.

### TC-PS-006 (P1) — Mapping overlap behavior
**Steps**: Create a mapping that overlaps an existing one.
**Expected**: Server-controlled behavior surfaced via toast — either rejected or auto-adjusted depending on `allow_overlap_adjust`. No silent failure.

### TC-PS-007 (P1) — Delete mapping
**Steps**: Delete an existing mapping.
**Expected**: Confirmation prompt; list refreshes on success.

### TC-PS-008 (P1) — Webhook log access from detail
**Steps**: Click a webhook log row.
**Expected**: Navigates to `/physical-spaces/webhook-logs/:logId`; payload + status visible.

### TC-PS-009 (P1) — Credential attempt history
**Steps**: Open credential attempt entry.
**Expected**: Navigates to `/physical-spaces/credential-logs/:requestId` (or equivalent route); details rendered.

### TC-PS-010 (P0) — Realtime webhook freshness
**Steps**: Trigger a webhook against this physical space externally.
**Expected**: Webhook log section auto-refreshes on `mrd:physical-space.webhook.processed` event; no manual reload needed.

---

## 8. Feature Test Cases — Meeting Room Displays

### TC-MRD-001 (P0) — Create display
**Steps**: `/meeting-room-displays` → **Create** → fill required `physical_space_id` → submit.
**Expected**: Verification code surfaced; display appears in list as unverified.

### TC-MRD-002 (P0) — Verify display
**Steps**: Use the verification code in the device-client emulator (or have backend mark verified).
**Expected**: Status flips to verified in realtime via `mrd:display.status.updated`; UI updates without reload.

### TC-MRD-003 (P0) — Link to ZenSpace meeting space
**Steps**: Detail → **Link** → enter `zenspace_api_key` + valid UUID `meeting_space_id`.
**Expected**: Linked status shown; webhook config registered.

### TC-MRD-004 (P1) — Invalid `meeting_space_id`
**Steps**: Submit a non-UUID value.
**Expected**: Zod rejects; submit blocked.

### TC-MRD-005 (P1) — Unlink from ZenSpace
**Steps**: Detail → **Unlink ZS**.
**Expected**: Link status clears; confirmation required.

### TC-MRD-006 (P1) — Check link availability
**Steps**: Try to link a display to a `meeting_space_id` already linked elsewhere.
**Expected**: Backend availability check rejects with clear error.

### TC-MRD-007 (P1) — Update display config
**Steps**: Detail → edit `booking_url` (required, must be valid URL).
**Expected**: Zod blocks invalid URL; valid URL saves and propagates.

### TC-MRD-008 (P2) — Upload image / logo
**Steps**: Upload via multipart endpoints.
**Expected**: New image visible immediately; broken/oversized files surfaced gracefully.

### TC-MRD-009 (P1) — Regenerate device token
**Steps**: Click regenerate token.
**Expected**: Confirmation; old token invalidated; new token shown once.

### TC-MRD-010 (P1) — Request screenshot
**Steps**: Click **Request screenshot** on detail.
**Expected**: New screenshot appears in screenshot list when uploaded (realtime).

### TC-MRD-011 (P1) — Webhook log detail
**Steps**: Click a webhook log row.
**Expected**: Navigates to `/meeting-room-displays/webhook-logs/:logId`; full payload + status visible.

### TC-MRD-012 (P1) — Notifications + stats
**Steps**: View notifications panel; check stats counts.
**Expected**: Pagination + filters work; counts match list.

### TC-MRD-013 (P1) — Unlink device
**Steps**: Click **Unlink device**.
**Expected**: Display reverts to unverified state.

### TC-MRD-014 (P1) — Delete display
**Steps**: Delete from list or detail.
**Expected**: Confirmation modal; row removed; associated logs no longer accessible from UI.

### TC-MRD-015 (P2) — Show booking URL toggle
**Steps**: Toggle the "show booking URL" switch on detail.
**Expected**: Setting persists; affects display config payload as documented in commit `430eea0`.

---

## 9. Feature Test Cases — IoT Gateways

### TC-GW-001 (P1) — Create gateway
**Steps**: `/IoT-Gateway` → **Create** → fill `name` (required, ≤100) + optional fields.
**Expected**: Verification code surfaced; gateway appears as unverified.

### TC-GW-002 (P1) — Invalid IPv4 in `unifi_local_ip`
**Steps**: Enter `999.999.999.999`.
**Expected**: Zod rejects.

### TC-GW-003 (P0) — Verify gateway (realtime)
**Steps**: Have the gateway connect with verification code.
**Expected**: Status flips via `mrd:gateway.status.updated`; UI updates without reload.

### TC-GW-004 (P1) — Update gateway fields
**Steps**: Edit name / adapter type / Tailscale config.
**Expected**: Saves; detail reflects.

### TC-GW-005 (P1) — Network / health / locks panels
**Steps**: Open each panel on detail.
**Expected**: Data renders or shows graceful empty state if backend returns no data.

### TC-GW-006 (P1) — Delete gateway
**Steps**: Delete from list.
**Expected**: Confirmation; cascade behavior on attached devices follows backend rules; no orphan UI.

---

## 10. Feature Test Cases — IoT Devices (Locks & Wifi)

### TC-DEV-001 (P1) — Create lock
**Steps**: `/locks` → **Create** → fill `gateway_id`, `external_id`, `display_name`, `physical_space_id`, enable ≥1 capability.
**Expected**: Created; appears in list.

### TC-DEV-002 (P2) — Capability validation
**Steps**: Submit with zero capabilities enabled.
**Expected**: Zod rejects.

### TC-DEV-003 (P1) — Assign / unassign physical space
**Steps**: Use assign/unassign actions.
**Expected**: Reflected on both device and physical space detail pages.

### TC-DEV-004 (P0) — Lock / unlock action
**Steps**: Click lock then unlock.
**Expected**: Toast feedback; status updates via `mrd:device.status.updated` realtime event.

### TC-DEV-005 (P1) — Unlock with custom window
**Steps**: Send unlock with `unlock_window_ms` value.
**Expected**: Backend honors; status reverts after window.

### TC-DEV-006 (P1) — Wifi device CRUD
**Steps**: Repeat TC-DEV-001..003 against `/wifi`.
**Expected**: Same behavior; rule parity with locks.

### TC-DEV-007 (P1) — Delete device
**Steps**: Delete; confirm.
**Expected**: Row removed; assignments cleared.

---

## 11. Feature Test Cases — Org API Keys

### TC-KEY-001 (P0) — Create API key
**Steps**: `/api-keys` → **Create** → name (≤200) + permission flags → submit.
**Expected**: Modal shows **plaintext key once**; copy button works; warning that key won't be shown again is visible.

### TC-KEY-002 (P0) — Plaintext exposure boundary
**Steps**: Close create modal, then reopen the key from the list.
**Expected**: Plaintext is NOT shown again anywhere in the UI (logs, list, detail).

### TC-KEY-003 (P1) — Rotate API key
**Steps**: Click **Rotate**.
**Expected**: New plaintext shown once; old key invalidated server-side.

### TC-KEY-004 (P1) — Revoke API key
**Steps**: Click **Revoke**.
**Expected**: Key marked revoked; cannot be used anymore.

### TC-KEY-005 (P2) — Permission flags
**Steps**: Create with various permission combinations.
**Expected**: Saved correctly; visible in list/detail.

---

## 12. Feature Test Cases — Magic Link (Public)

### TC-ML-001 (P0) — Valid token render
**Steps**: Open `/magic/<valid_token>` in incognito (no auth).
**Expected**: Booking details + attached device adapter iframes render. No redirect to `/auth`.

### TC-ML-002 (P0) — Invalid / expired token
**Steps**: Open `/magic/garbage`.
**Expected**: Friendly error state; no crash; no silent auth redirect.

### TC-ML-003 (P1) — Device action via iframe
**Steps**: Use an embedded lock adapter to unlock.
**Expected**: Action executes; status updates surfaced.

### TC-ML-004 (P1) — Mobile rendering
**Steps**: Open page at 375×667 viewport.
**Expected**: Layout usable; no horizontal scroll; iframe scales.

### TC-ML-005 (P2) — Timezone display
**Steps**: Verify booking time displayed in `display_timezone`.
**Expected**: Matches expected zone, not browser local.

---

## 13. Cross-Cutting Test Cases

### TC-X-001 (P1) — `ListViewTable` query contract
**Test on every list page** (`/physical-spaces`, `/meeting-room-displays`, `/IoT-Gateway`, `/locks`, `/wifi`, `/api-keys`).
**Expected**: `page`, `limit`, `sortBy`, `sortOrder`, `search`, and filter params reflect in URL and survive reload.

### TC-X-002 (P1) — `hardRefresh` invalidation
**Steps**: Perform a create/update/delete on any list.
**Expected**: List refetches without manual reload.

### TC-X-003 (P0) — Realtime reconnect
**Steps**: Disconnect network for 10s, reconnect.
**Expected**: Socket.IO reconnects; relevant data refreshes.

### TC-X-004 (P1) — Breadcrumb consistency
**Steps**: Inspect breadcrumb on every detail page.
**Expected**: Title matches page heading (casing/spelling) — regression target per commit `f1e91e0`.

### TC-X-005 (P2) — Toast behavior
**Steps**: Trigger success and error toasts.
**Expected**: Sonner toasts appear with correct severity; auto-dismiss.

### TC-X-006 (P1) — 401 propagation across realtime + REST
**Steps**: Force a 401 mid-session.
**Expected**: REST handler clears token, websocket handshake fails cleanly, user routed to `/auth`.

### TC-X-007 (P2) — Empty states
**Steps**: View every list with zero rows (use fresh test org).
**Expected**: Friendly empty state — no "Loading…" hang, no console errors.

### TC-X-008 (P2) — Hidden routes still accessible by deep link
**Steps**: Navigate directly to `/spaces` and `/spaces/:id`.
**Expected**: Routes render (they exist but are hidden from sidebar by design).

---

## 14. Realtime Event Testing Matrix

| Hook | Event | Trigger to use | Expected UI effect |
|---|---|---|---|
| `useRealtimeDisplayStatus` | `mrd:display.status.updated` | Verify display from device client | Display row/detail flips status badge without reload |
| `useRealtimeWebhooks` | `mrd:webhook.received` | Fire webhook to display | Webhook log list prepends new entry |
| `useRealtimeWebhooks` | `mrd:physical-space.webhook.processed` | Fire physical-space webhook | Physical-space webhook log refreshes |
| `useRealtimeIotVerification` | `mrd:gateway.status.updated` | Verify gateway from device | Gateway status badge updates |
| `useDeviceStatusUpdates` | `mrd:device.status.updated` | Trigger lock/unlock | Device status badge updates on space detail |

For each: also verify the hook disconnects cleanly when navigating away (check `socket.disconnect` in DevTools).

---

## 15. Regression Checklist (run on every release)

- [ ] `npm run lint` — clean
- [ ] `npm run build` — succeeds, no TS errors
- [ ] `npm run preview` — SPA rewrites work on deep links
- [ ] Login → org select → physical spaces lands successfully
- [ ] Logged-in user hitting `/auth` is redirected to `/organization`
- [ ] Unauthenticated user hitting any protected route is redirected to `/auth`
- [ ] All protected pages work with `?orgId=<id>` and break gracefully without it
- [ ] List screens refresh after mutations (no manual reload required)
- [ ] Realtime hooks reconnect after a network blip
- [ ] API key plaintext shown **exactly once**; never reappears
- [ ] Magic-link page renders without authentication
- [ ] Toast notifications appear for both success and error
- [ ] No `console.error` noise during a happy-path login → physical spaces → display detail → log detail
- [ ] Breadcrumbs match page titles (casing + spelling)
- [ ] All buttons that perform destructive actions show a confirmation
- [ ] Forms preserve unsaved state on validation error (don't blank out user input)

---

## 16. Bug Reporting Template

When raising a bug, include the following so it can be triaged and reproduced without follow-up:

```
Title:        [Module] short, specific summary
Priority:     P0 | P1 | P2 | P3
Environment:  Dev | Staging | Prod
URL:          full URL including query params
Browser:      Chrome 120 / Safari 17 / etc.
User:         test user email + super_admin? yes/no
Org ID:       UUID
Test data:    IDs of physical space / display / gateway / device used

Steps to reproduce:
  1. ...
  2. ...
  3. ...

Expected:     what should happen
Actual:       what actually happened

Evidence:
  - Screenshot / screen recording
  - Browser console errors (paste)
  - Network tab: failing request + response body
  - Socket.IO events received (if realtime issue)

Notes:
  - First seen on commit / build #
  - Related test case ID (e.g. TC-MRD-007)
  - Workaround (if any)
```

File bugs in the team tracker against the affected module label.

---

## 17. Test Run Sign-off Template

| Section | Owner | Pass / Fail | Notes |
|---|---|---|---|
| 5. Authentication | | | |
| 6. Organizations | | | |
| 7. Physical Spaces | | | |
| 8. Meeting Room Displays | | | |
| 9. IoT Gateways | | | |
| 10. IoT Devices | | | |
| 11. Org API Keys | | | |
| 12. Magic Link | | | |
| 13. Cross-Cutting | | | |
| 14. Realtime Matrix | | | |
| 15. Regression Checklist | | | |

**Release approved by**: ________________  **Date**: ____________
**Build / commit SHA tested**: ________________

---

## 18. Known Constraints

- This repo has **no automated test suite**. All QA validation is manual.
- The frontend has no role-based route enforcement beyond `super_admin` controlling the organization card menu — do not assume role-based access for other routes.
- Auth token lives in `localStorage` (`auth_token`) — be careful in shared/test machines.
- Magic-link page uses raw `fetch` (not the shared API client), so its error behavior may diverge — test it explicitly.
- `VITE_BACKEND_URL` and `VITE_WEBSOCKET_SERVICE_URL` are the only env vars. Confirm they point to the correct backend before any test run.
