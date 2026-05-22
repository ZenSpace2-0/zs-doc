# ZenSpace Admin — QA Test Plan

A full-app QA reference for `zs-admin` (with linked checks for the `zs-booking` end-user app where flows touch each other). Use this as a release-gate checklist; pick the relevant module sections for targeted regression.

---

## 1. Scope

**In scope**
- `zs-admin` dashboard: authentication, organization, groups, meeting spaces, bookings (incl. cancel/refund), pricing, vouchers, devices, members, roles/users, settings, integrations, notifications/webhooks, profile.
- Cross-app touchpoints with `zs-booking` (end-user) when an admin action changes a customer-facing surface (booking confirmation, cancellation, pricing).
- Permission gating (role-based feature visibility).

**Out of scope (separate test plans)**
- `zs-backend` API contract tests.
- `zs-meeting-room-display` and `airtame` device firmware.
- Stripe webhook simulation beyond what the admin UI surfaces.

---

## 2. Test Environment

| Item | Value |
|---|---|
| Admin app URL | `https://admin.<env>.zenspace.io` |
| Booking app URL | `https://book.<env>.zenspace.io` |
| Backend API | `https://api.<env>.zenspace.io` |
| Environments | `dev`, `staging`, `prod-readonly` for smoke |
| Browsers | Chrome (latest), Safari (latest), Firefox (latest), Edge (latest) |
| Devices | Desktop 1440×900, Laptop 1280×800, iPad 1024×768, iPhone 14 (390×844) |
| Realtime | WebSocket connectivity must be verified (booking detail uses live updates) |
| Stripe | Test mode keys configured on the org used for testing |

**Test users**
- `admin@qa.zenspace.io` — Owner permissions
- `manager@qa.zenspace.io` — Manager role (no destructive perms)
- `viewer@qa.zenspace.io` — Read-only
- `external-customer@qa.zenspace.io` — End-user for booking flow

**Seed data per env**
- 1 organization with Stripe connected
- 2 groups (one active, one with availability window in the past)
- 4+ meeting spaces (mix of `zenspace`/`hybrid`/`third_party` operating modes, with and without deposits, free and paid)
- Cancellation policy: `before_hours` org default + at least one space-level override
- 2 amenities, 1 voucher (active), 1 dynamic-pricing rule

---

## 3. Pre-flight Smoke (run before any module testing)

- [ ] Sign in succeeds for all three role users.
- [ ] Org switcher loads all orgs the user belongs to; switching orgs reloads scoped data.
- [ ] Dashboard landing renders with no console errors.
- [ ] Network tab: no 5xx; auth token attached to API calls.
- [ ] Realtime socket connects (check `useBookings`/booking-realtime hook in DevTools).

---

## 4. Module Test Cases

Format: each table row is one scenario. **Priority**: `P0` = release blocker, `P1` = important, `P2` = nice-to-have. Capture screenshots for any failure.

### 4.1 Authentication & Org Selection

| ID | Scenario | Steps | Expected | Pri |
|---|---|---|---|---|
| AUTH-01 | Valid sign-in | Enter valid creds → submit | Redirect to `/organization`; token in storage | P0 |
| AUTH-02 | Invalid creds | Wrong password | Inline error; no redirect; no token | P0 |
| AUTH-03 | Session expiry | Idle until token expires → click any nav | Redirect to sign-in; original URL preserved as `?next=` | P1 |
| AUTH-04 | Multi-org user | Sign in as user in 2+ orgs | Org list shown; selecting one enters dashboard scoped to that org | P0 |
| AUTH-05 | Single-org user | Sign in as user in 1 org | Auto-enter that org's dashboard | P1 |
| AUTH-06 | Sign out | Profile menu → sign out | Token cleared; redirect to sign-in; back button does NOT restore session | P0 |
| AUTH-07 | Direct deep link while signed out | Paste `/bookings/<id>` URL when signed out | Redirect to sign-in; after sign-in lands on the deep-linked URL | P1 |

### 4.2 Organization

| ID | Scenario | Steps | Expected | Pri |
|---|---|---|---|---|
| ORG-01 | Create org | New org → fill required fields → save | Org appears in list; auto-enter new org | P0 |
| ORG-02 | Edit org details | Open org settings → change name, timezone, currency | Saved values reflected on reload; timezone affects booking time displays | P0 |
| ORG-03 | Stripe connect | Settings → Stripe → connect account | Stripe OAuth flow completes; status badge shows "Connected" | P0 |
| ORG-04 | Number-of-spaces limit | Set `number_of_spaces` low → try to create another meeting space | Toast error: "max limit reached"; meeting space NOT created | P1 |
| ORG-05 | Org timezone propagation | Change org timezone → reload booking detail | Booking times re-render in new timezone | P1 |

### 4.3 Groups

| ID | Scenario | Steps | Expected | Pri |
|---|---|---|---|---|
| GRP-01 | Create group — happy path | `/groups/create` → fill all steps → submit | Toast success; **redirect to `/groups/{newId}` (detail page)** | P0 |
| GRP-02 | Create — back button after submit | Complete GRP-01 → press browser back | Should land on the page BEFORE the create form (not on the create form itself) | P0 |
| GRP-03 | Edit group | Open group → Edit → change field → submit | Toast success; **redirect to `/groups/{id}` (detail page)** | P0 |
| GRP-04 | Edit — back button after submit | Complete GRP-03 → press browser back | Should land on the page BEFORE the edit form (not the edit form) | P0 |
| GRP-05 | Cancel button on form | Open create or edit → click Cancel | Returns to `/groups`; no save toast | P1 |
| GRP-06 | Availability window | Set `available_from` / `available_until` → save → try to create a meeting space outside this range | Meeting space save rejected with date-range error | P1 |
| GRP-07 | Group calendar | Open `/groups/{id}/calendar` | Renders monthly view; bookings across child spaces shown | P1 |
| GRP-08 | Kiosk display groups | Add a display group → assign meeting spaces | Modal saves; assigned spaces appear in the kiosk group | P2 |
| GRP-09 | Embed snippet | Group → Embed on your website | Copyable iframe snippet; pasting it elsewhere renders the group's public booking page | P2 |
| GRP-10 | Duplicate group | Group card menu → Duplicate | New group created with `-copy` suffix; meeting spaces NOT copied unless opted in | P2 |
| GRP-11 | Delete group | Group menu → Delete (must have permission) | Confirmation modal; on confirm, group + its spaces are removed | P1 |
| GRP-12 | Permission gating | Sign in as viewer | Create/Edit/Delete buttons NOT visible | P0 |

### 4.4 Meeting Spaces

| ID | Scenario | Steps | Expected | Pri |
|---|---|---|---|---|
| MS-01 | Create — zenspace mode (full form) | Open group → Add meeting space → step through all 6 steps → review → submit | Space created; **redirect to `/meeting-spaces/{id}`**; back from there should not land on the create form | P0 |
| MS-02 | Create — third_party mode | Choose `third_party` operating mode on step 1 | All other fields hide; submit succeeds with just operating mode + third_party_id | P0 |
| MS-03 | Create — back button after submit | Complete MS-01 → press browser back | Lands on whatever page preceded the create form (not the create form) | P0 |
| MS-04 | Edit space — full update | Open `/meeting-spaces/{id}/edit` → modify → submit | Toast success; **redirect to `/meeting-spaces/{id}`**; back does NOT return to edit | P0 |
| MS-05 | Slug uniqueness | Type a slug already taken | Inline "Taken" indicator; submit blocked | P0 |
| MS-06 | Slug auto-generation | Type a name in create mode | Slug auto-fills (kebab-case); in edit mode it only re-fills when "Regenerate" is checked | P1 |
| MS-07 | Capacity required in zenspace mode | Clear capacity → try to submit | Required field error fires | P1 |
| MS-08 | Min < max booking duration | Set `min_booking_duration` higher than `max_booking_duration` after a previous selection | `max_booking_duration` is auto-reset; user must reselect | P1 |
| MS-09 | Cleaning toggle | Toggle `cleaning_required` off | `maintenance_duration` snaps to 0 | P2 |
| MS-10 | Deposit toggle | Toggle `deposit_required` off | `security_deposit` and `deposit_refundable` clear | P2 |
| MS-11 | Effective date within group window | Set effective_from before group `available_from` | Submit rejected with explicit error referencing group window | P1 |
| MS-12 | Image gallery — non-square upload | Upload a 1000×800 image to gallery | Reject with size requirements message | P2 |
| MS-13 | Image gallery — valid square | Upload 1200×1200 PNG | Accepted; appears in preview | P2 |
| MS-14 | Floor plan upload | Upload PDF or image as floor plan | Saved; visible on detail page | P2 |
| MS-15 | Calendar tab | Open `/meeting-spaces/{id}/calendar` | Bookings shown for this space only; can navigate weeks/days | P1 |
| MS-16 | Devices tab | Devices tab → attach device | Device appears; status badge correct | P1 |
| MS-17 | Pricing rules | Add a pricing rule on the space | Rule listed; preview reflects modified price for booking flow | P1 |
| MS-18 | Duplicate meeting space | Duplicate modal → submit | New space with `-copy` slug; all settings carried over | P2 |
| MS-19 | Status timeline | Change status to `under_maintenance` | Timeline shows the transition + timestamp | P2 |
| MS-20 | Permission gating | Viewer cannot see Edit/Delete; Manager cannot Delete | Buttons hidden per role | P0 |

### 4.5 Business Hours & Unavailability

| ID | Scenario | Steps | Expected | Pri |
|---|---|---|---|---|
| BH-01 | Create business hours | Add Mon-Fri 9–17 to a meeting space | Saved; visible in booking app slot generation | P0 |
| BH-02 | Overlapping ranges | Add 9-12 and 11-15 same day | Validation error or merge behavior per spec | P1 |
| BH-03 | Bulk apply to days | Set 9-17 then bulk-apply to weekdays | All weekdays receive the range | P2 |
| UN-01 | Add unavailability block | Space → unavailability → add block (date+time) | Booking app cannot book those slots | P0 |
| UN-02 | Recurring unavailability | Add weekly recurring block | Generates blocks for the next N weeks | P1 |
| UN-03 | Delete block | Remove a future block | Slots free up in booking app on refresh | P1 |

### 4.6 Dynamic Pricing

| ID | Scenario | Steps | Expected | Pri |
|---|---|---|---|---|
| DP-01 | Create rule — multiplier | "Peak hours" 18-21 → ×1.5 | Saved; booking quote for 19:00 shows 1.5× base | P0 |
| DP-02 | Create rule — fixed adjustment | Weekend +$20 | Quote on Saturday reflects +20 | P1 |
| DP-03 | Date range rule | New Year's Eve override | Applies only on configured dates | P1 |
| DP-04 | Rule priority | Two overlapping rules | Highest-priority rule wins per spec | P1 |
| DP-05 | Delete rule | Remove a rule | Booking quote reverts immediately on next quote call | P2 |

### 4.7 Vouchers

| ID | Scenario | Steps | Expected | Pri |
|---|---|---|---|---|
| VO-01 | Create % voucher | 10% off, no limits | Code appears in list; usable in booking app | P0 |
| VO-02 | Create fixed-amount voucher | $25 off | Reduces final amount by exactly $25 (floored at 0) | P0 |
| VO-03 | Usage limit | Set max_uses = 2 → apply twice in booking → try third | Third use rejected | P1 |
| VO-04 | Expiry | Set expiry in the past | Voucher rejected as expired in booking app | P1 |
| VO-05 | Space restriction | Restrict to Space A → try on Space B | Rejected with scoped-error message | P1 |
| VO-06 | Remove voucher highlight | Open a voucher used in a booking → look at booking detail | Highlight/badge shows voucher applied; remove action behaves per recent UX change | P2 |

### 4.8 Bookings

| ID | Scenario | Steps | Expected | Pri |
|---|---|---|---|---|
| BK-01 | Bookings list — filters | Filter by status, date range, space, customer | List updates; URL params reflect filters | P0 |
| BK-02 | Open booking detail | Click a row | `/bookings/{id}` opens with full info, status badge, payment summary | P0 |
| BK-03 | Realtime update | Open booking detail; from another tab change status (or simulate event) | Detail re-renders without manual refresh | P1 |
| BK-04 | Payment status polling | Open booking with `pending` payment | Polled every 10–30s depending on socket state | P2 |
| BK-05 | Issue refund (non-cancel) | Detail → Issue refund → enter amount | Refund processed; payment status updates; refund row appears in history | P0 |
| BK-06 | Cancel booking — happy path | Detail → Cancel booking → confirm with policy refund | Booking status → cancelled; refund issued per policy; toast success | P0 |
| BK-07 | Cancel — custom override | Cancel modal → switch to Custom override → set amount | Override applied; refund matches admin entry; capped at preview's `additional_total_refund` | P0 |
| BK-08 | Cancel — bypass mode | Admin uses bypass to mark cancelled with no refund | Status changes; no refund triggered | P1 |
| BK-09 | **Cancel — `cancellation_allowed === false`** | Open Cancel modal on a booking the BE blocks from cancelling | **Destructive warning shown inside modal; "Cancel Booking" submit button is DISABLED** | P0 |
| BK-10 | Cancel — preview fetch failure | Throttle network or kill preview endpoint | Modal shows "you can still cancel using a custom override" fallback; submit still works in custom mode | P1 |
| BK-11 | Cancel — already refunded | Open Cancel modal on a booking whose payment is fully refunded | Preview is skipped; "Mark as cancelled — no new refund" button text shown | P1 |
| BK-12 | Voucher booking — refund math | Cancel a booking that used a voucher | Refund uses `total_amount` (post-voucher), NOT sticker price | P0 |
| BK-13 | Deposit refund toggle | Booking with deposit → cancel → choose "force keep deposit" | Deposit retained; only booking-side refund issued | P1 |
| BK-14 | Permission gating | Viewer opens booking detail | Cancel + Refund buttons hidden | P0 |
| BK-15 | List bulk-cancel | Select multiple bookings → bulk cancel | Each one cancelled; failures reported per row | P2 |

### 4.9 Booking — End-user (zs-booking) interplay

| ID | Scenario | Steps | Expected | Pri |
|---|---|---|---|---|
| BKE-01 | Create booking end-to-end | End-user picks space → slot → pays → confirmation | Admin sees the booking in `/bookings` within seconds | P0 |
| BKE-02 | End-user cancel — `cancellation_allowed === false` | End-user opens cancel modal on a BE-blocked booking | **Destructive warning inside modal; submit DISABLED** | P0 |
| BKE-03 | End-user cancel — happy path | End-user cancels | Booking status updates; refund processed; admin detail reflects change in real time | P0 |
| BKE-04 | Booking confirmation email | Complete a booking | Customer receives confirmation; admin notification fires if configured | P1 |
| BKE-05 | iframe embed | Embed group page via snippet from GRP-09 | Booking flow works inside the iframe | P2 |

### 4.10 Devices, Logs, Access

| ID | Scenario | Steps | Expected | Pri |
|---|---|---|---|---|
| DEV-01 | Add device | Devices module → add → assign to space | Device appears in space's Devices tab | P1 |
| DEV-02 | Device log feed | Trigger an event (device check-in) | Log entry appears with timestamp + payload | P1 |
| DEV-03 | User-device access | Grant a user access to a space's device | Access record visible; revocation works | P1 |
| DEV-04 | Push tokens | Mobile app registers → check tokens list | Token visible; can revoke | P2 |
| DEV-05 | Booking access retry policy | Settings → policy → configure retry attempts | Failed access attempts retried per config; logged | P1 |
| DEV-06 | Device failure policy | Settings → failure policy → set fallback behavior | When a device is unreachable, configured fallback fires | P1 |

### 4.11 Members, Roles, Users

| ID | Scenario | Steps | Expected | Pri |
|---|---|---|---|---|
| MEM-01 | Add meeting-space member | Open space members → add user | User listed with role | P1 |
| MEM-02 | Remove member | Remove a member | Removed from list; loses access to space-scoped features | P1 |
| ROL-01 | Create custom role | Roles → new role → set permissions | Role available in user assignment | P1 |
| ROL-02 | Edit permissions | Toggle a permission on an existing role | Users with role see UI changes immediately on next page load | P1 |
| USR-01 | Invite user | User search and create → enter email → assign role | Invitation sent; user appears as "pending" | P0 |
| USR-02 | Resend invite | Pending user → resend | New invite email sent | P2 |
| USR-03 | Disable user | Toggle active → off | User can no longer sign in | P0 |

### 4.12 Amenities

| ID | Scenario | Steps | Expected | Pri |
|---|---|---|---|---|
| AME-01 | Create amenity | Amenities → new → name + icon + type | Amenity listed; selectable in meeting space form | P1 |
| AME-02 | Inline create from MS form | MS form → amenities select → type new name → Create | Modal opens; on save, amenity added + selected on the form | P1 |
| AME-03 | Edit amenity | Update icon or type | Reflected in MS form selector immediately (cache invalidated) | P2 |
| AME-04 | Delete amenity | Delete an amenity in use by spaces | Confirmation; on delete, amenity removed from spaces or warning shown per spec | P2 |

### 4.13 Settings

| ID | Scenario | Steps | Expected | Pri |
|---|---|---|---|---|
| SET-01 | API keys — create | Settings → API keys → new | Key shown ONCE; copy works; key listed (masked) | P1 |
| SET-02 | API keys — revoke | Revoke a key → use it via curl | API returns 401 | P1 |
| SET-03 | Cancellation policy — org default | Set `before_hours` with N hours | New bookings without space override use this policy on cancel preview | P0 |
| SET-04 | Cancellation policy — space override | Set override on a space | Cancel preview for that space uses override (preview `policy.source = 'space'`) | P0 |
| SET-05 | Scheduled jobs | View job history | Jobs listed with last-run timestamp + status | P2 |
| SET-06 | Stripe connect / disconnect | Connect, then disconnect | Status updates; new bookings cannot collect payment after disconnect | P0 |

### 4.14 Integrations, Notifications, Webhooks, Inbox

| ID | Scenario | Steps | Expected | Pri |
|---|---|---|---|---|
| INT-01 | Add integration | Integrations → connect (e.g. Google Calendar) | OAuth completes; status "Connected" | P1 |
| NOT-01 | Notification rule | Create rule → trigger condition | Notification fired and visible in Inbox | P1 |
| WEB-01 | Add webhook | Webhooks → new → URL + events | Test event delivers; signed payload received | P1 |
| WEB-02 | Webhook retry on failure | Make endpoint return 500 → trigger event | Retries per backoff schedule; visible in logs | P2 |
| INB-01 | Read inbox messages | Open Inbox → click message | Marks as read; unread count decrements | P2 |

### 4.15 Profile

| ID | Scenario | Steps | Expected | Pri |
|---|---|---|---|---|
| PRO-01 | Update name | Profile → edit name → save | Reflected in header + audit trail | P2 |
| PRO-02 | Change password | Profile → password → set new | Old password rejected on next sign-in | P1 |
| PRO-03 | Avatar upload | Upload PNG/JPG ≤2MB | Shown in header; rejects oversized files | P2 |

---

## 5. Cross-Cutting Concerns

### 5.1 Navigation & History

The admin app uses Next.js client-side routing. The following must hold:

- [ ] After a successful **create or edit submission**, the form page must NOT remain in history — pressing browser back must skip the form and land on the page before the form. (Implemented via `router.replace`. Verify on: meeting space create/edit, group create/edit. See GRP-02/04, MS-03/04.)
- [ ] Cancel/Back buttons on multi-step forms go to the previous step, not out of the form, except on step 1 where they exit.
- [ ] Deep links restore full state (filters, modal open states should NOT auto-restore — that's the spec).
- [ ] Browser refresh on any page does not crash; loading states render until data arrives.

### 5.2 Permissions

For every module, verify three roles:

| Role | Expected |
|---|---|
| Owner / Admin | All actions available |
| Manager | Read + non-destructive edit; no Delete, no role/permission changes |
| Viewer | Read-only; all action buttons hidden, not just disabled |

Hidden ≠ disabled — buttons should be removed from the DOM for unauthorized users, not greyed out.

### 5.3 Timezone Correctness

- [ ] All booking times rendered using `booking.organization.timezone || organization.timezone || 'UTC'`.
- [ ] Cancellation preview `hours_until_start` matches the org's timezone interpretation.
- [ ] Date pickers respect the selected org timezone (do NOT use the browser's local TZ silently).

### 5.4 Money & Currency

- [ ] All amounts shown with 2 decimal places.
- [ ] Currency symbol matches org currency setting (not hard-coded `$`).
- [ ] Refund math uses post-voucher `total_amount`, not `sticker_amount`.
- [ ] Deposit refunds shown separately from booking-side refunds.

### 5.5 Realtime

- [ ] WebSocket reconnects after network drop.
- [ ] Polling fallback engages within 30s when socket is down.
- [ ] Stale data doesn't persist — booking detail re-fetches on every event for the open booking.

### 5.6 Accessibility (smoke pass)

- [ ] Keyboard nav reaches all primary actions (Tab order is logical).
- [ ] Forms expose validation errors via `role="alert"` (sample 2 forms per module).
- [ ] Modals trap focus; Esc closes.
- [ ] Color contrast ≥4.5:1 for body text.

### 5.7 Error & Empty States

- [ ] Network failure on any list shows a retry CTA, not a blank page.
- [ ] Empty list states show contextual "Create your first X" prompts.
- [ ] Permission-denied for a deep link shows the `NotFound` component (with module-specific icon + message), not a blank screen.

---

## 6. Recently Changed Areas — Targeted Regression

Run these before every release. They cover changes shipped in the current development cycle.

| Area | Test IDs to run | Notes |
|---|---|---|
| Meeting space create/edit redirect | MS-01, MS-03, MS-04 | Verify back button skips form |
| Group create/edit redirect to detail | GRP-01, GRP-02, GRP-03, GRP-04 | Now goes to `/groups/{id}`, not `/groups` |
| Cancellation preview gate | BK-09, BK-10, BKE-02 | Both admin & end-user modals |
| Group-level configuration | GRP-06 + any settings inheritance tests | Recent commit `e5aa0e2` |
| User search & create | USR-01 | Recent commit `181ec8f` |
| Voucher remove highlight | VO-06 | Recent commit `2225669` |

---

## 7. Browser / Device Matrix

| Surface | Chrome | Safari | Firefox | Edge | iPad | iPhone |
|---|---|---|---|---|---|---|
| Sign-in | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| Dashboard | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| Groups & meeting spaces | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| Booking detail + cancel/refund modals | ✓ | ✓ | ✓ | ✓ | ✓ | smoke only |
| Calendar views | ✓ | ✓ | ✓ | ✓ | ✓ | smoke only |
| File upload (images / floor plan) | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| iframe embed (booking) | ✓ | ✓ | ✓ | ✓ | n/a | n/a |

Mobile = primarily admin reviewing data on the go; complex create flows are desktop-first.

---

## 8. Performance & Load (sanity, not formal load test)

- [ ] Bookings list with 1000+ rows: pagination loads <2s per page.
- [ ] Meeting space calendar with 200+ bookings/month: renders <3s.
- [ ] Cancellation preview API: <500ms p95 on a typical booking.
- [ ] Image upload (5MB): completes <10s on stable connection.

---

## 9. Bug Report Template

When filing a bug, include:

```
Title: <short, action-oriented>
Env: dev / staging / prod
Build: <commit sha or release tag>
User role: owner / manager / viewer
Org: <name + id>

Steps to reproduce:
1. ...
2. ...
3. ...

Expected:
<what should happen, referencing Test ID if applicable>

Actual:
<what did happen>

Screenshot / video: <attach>
Console errors: <paste>
Network: <relevant failed requests with status>
```

---

## 10. Sign-off Checklist (per release)

- [ ] All P0 cases passed across required browsers.
- [ ] All P1 cases passed on Chrome at minimum.
- [ ] Recently-changed-areas (§6) re-run on staging.
- [ ] No new console errors on smoke flows.
- [ ] Permission matrix verified for all three roles.
- [ ] Mobile smoke (sign-in, view booking, view meeting space) completed.
- [ ] zs-booking interplay (BKE-*) verified end-to-end.
- [ ] Cancel/refund flows verified with real Stripe test transactions.
- [ ] Release notes drafted and reviewed by product.

---

## Appendix A — Key URLs

| Surface | URL pattern |
|---|---|
| Sign-in | `/auth` |
| Org selection | `/organization` |
| Dashboard | `/dashboard` |
| Groups list | `/groups` |
| Group detail | `/groups/{id}` |
| Group calendar | `/groups/{id}/calendar` |
| Group create | `/groups/create` |
| Group edit | `/groups/{id}/edit` |
| Meeting space create | `/meeting-spaces/create?groupId={id}` |
| Meeting space detail | `/meeting-spaces/{id}` |
| Meeting space edit | `/meeting-spaces/{id}/edit` |
| Meeting space calendar | `/meeting-spaces/{id}/calendar` |
| Bookings list | `/bookings` |
| Booking detail | `/bookings/{id}` |
| Settings — cancellation | `/settings/cancellation-policy` |
| Settings — Stripe | `/settings/stripe` |
| Vouchers | `/vouchers` |
| Devices | `/devices` |
| Roles | `/roles` |

## Appendix B — Useful API Endpoints (for QA verification)

| Purpose | Method + Path |
|---|---|
| Booking by id | `GET /bookings/{id}` |
| Cancellation preview | `GET /bookings/{id}/cancellation-preview` |
| Cancel booking | `POST /bookings/{id}/cancel` |
| Refund (full) | `POST /bookings/{id}/refund/full` |
| Refund (partial) | `POST /bookings/{id}/refund/partial` |
| Meeting space by slug | `GET /meeting-spaces/by-slug/{slug}` |
| Check slug availability | `GET /meeting-spaces/check-slug` |
| Dashboard counts | `GET /dashboard` |

Use Postman / curl with a valid bearer token from a signed-in admin to validate UI state matches API state when triage is needed.
