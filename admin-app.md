# Zenspace App Guide

## What This App Does

Zenspace helps teams manage spaces that can be booked by internal users or external customers.

The app has two sides:

- The admin side, where operators set up and manage the business
- The booking side, where customers or end users browse and reserve spaces

In simple terms, the app supports this full journey:

1. Set up an organization
2. Add groups or locations
3. Create meeting spaces inside those groups
4. Configure pricing, availability, devices, vouchers, and rules
5. Let customers book spaces through the booking app — by booking directly, browsing availability, or submitting a request for admin approval
6. Manage bookings, booking requests, and device access from the admin dashboard

The platform spans two products:

- **ZenCore Space OS** — booking, calendar, pricing, and availability management (most of this guide).
- **ZenEdge Operations** — the hardware/edge side: physical door access, adapter devices, keypad PINs, and per-user door links (see [Device access & door control](#16-device-access--door-control-zenedge)).

## Who Uses It

### Admins and Operations Teams

They use the dashboard to:

- create and organize groups
- add meeting spaces
- manage bookings (single or in bulk via a cart)
- review and approve booking requests
- set prices and discounts, including full-day pricing
- connect devices, control door access, and manage per-user access
- configure rules, notifications, and settings

### Customers or End Users

They use the booking experience to:

- browse available spaces (book directly, view availability only, or request a space)
- pick a date and time (or a full day)
- review price
- complete payment
- see booking confirmation

## URL and navigation notes

- Visiting `/` redirects to `/organization` (organization selection and workspace entry).
- Public booking URLs are exposed under **`/booking/...`**; Next.js rewrites those paths to the internal `booking-app` routes (same behavior, user-facing URL stays `/booking/...`).
- The public booking app has **three browse modes**, distinguished by a path segment:
  - **Book** (default) — `/booking/{orgSlug}/{groupSlug}/` : browse and book directly.
  - **View availability** — `/booking/view/{orgSlug}/{groupSlug}` : read-only availability, no booking.
  - **Request spaces (SRF)** — `/booking/request/{orgSlug}/{groupSlug}` : submit a request for admin approval instead of paying.
- A full-screen **kiosk display** board lives at `/booking/{orgSlug}/{groupSlug}/kiosk-display` (optionally `?display_group={slug}` to show one display group).
- Some organization-role URLs are friendly aliases: for example, **`/organization/roles`** is rewritten to **`/system-roles`** (actual path in the address bar may show `/system-roles`).
- Organization editing uses **`/organization/edit/:id`**, which is rewritten to **`/organization/:id`**.
- Deep links from a meeting space can open **`/notifications/:id`** and **`/webhooks/:id`**; the same screens are also reachable via rewrite aliases **`/meeting-spaces/notifications/:id`** and **`/meeting-spaces/webhooks/:id`**.
- There is **no** standalone **`/meeting-spaces`** list page in the app router; meeting spaces are created and opened from **Groups** (and the sidebar treats group-related URLs as the active section).

## Time, timezone, and clock behavior

- **All time pickers and time labels display in 12-hour format** (e.g. `1:30 PM`). Internally, times are still stored and sent to the backend in 24-hour `HH:mm` form — only the display is converted, so backend payloads and duration/overlap logic are unaffected.
- Times that belong to a space or group (business hours, unavailability windows, kiosk boards, full-day windows) are rendered in **the space's / organization's timezone**, not the admin's local browser timezone.
- A live **timezone clock** appears on the **group detail** location card (the group's timezone) and in the **sidebar footer** (the active organization's timezone). It ticks every second and shows the current local date and time.

## Main user flows

### 1. Sign In and Enter the Workspace

The first step is signing in.

After authentication, the user enters the organization workspace flow. If the user belongs to multiple organizations, they can choose which workspace to enter. Access and refresh tokens are held in a centralized auth store, and the sign-in flow supports safe post-login redirects back to the destination the user was headed to.

Typical flow:

1. Open the app
2. Sign in
3. Select an organization
4. Enter the dashboard

Why this matters:

- Everything in the dashboard is organization-based
- Users only see the data and settings for the organization they are working in

### 2. Create or Select an Organization

Organizations are the top-level business container in the app.

An organization usually represents a company, brand, or operating business using Zenspace.

Inside an organization, teams can manage:

- groups
- spaces
- users
- bookings and booking requests
- pricing
- device access
- settings

Typical flow:

1. Open the organization list
2. Search or choose an organization
3. Or create a new organization
4. Enter that organization's workspace

### 3. Set Up Groups or Locations

Groups represent the next level under an organization.

A group often acts like a location, branch, building, or collection of spaces.

A group usually contains:

- name and identity
- address and timezone
- images and public-facing presentation
- meeting spaces that belong to that group
- optional kiosk display groups (see flow 14)

Typical flow:

1. Open the groups page
2. Create a group
3. Add address and timezone
4. Add images or presentation content
5. Start adding meeting spaces inside that group

From a group's detail page, operators can also:

- open a **Booking Links** dropdown to launch the public booking app in any of its three modes (Book / View availability / Request spaces)
- manage **kiosk display groups** for wayfinding boards
- open the group calendar

Why groups are important:

- Booking and availability often depend on group context
- The booking app can expose a group-level public experience

### 4. Create and Manage Meeting Spaces

Meeting spaces are the main bookable units in the platform.

A meeting space can represent:

- a meeting room
- a desk area
- a studio
- a private office
- another reservable space

Admins can configure:

- name and code
- room details and description
- capacity
- images and thumbnails
- booking duration rules (min/max duration, booking increment)
- availability window
- base (hourly) price and deposit-related values
- **full-day booking** (opt-in) and a separate **full-day price**
- **public surface scopes** — for each public mode (Book / View-only / Request), whether it offers hourly slots, full day, or both
- active or inactive state

Typical flow:

1. Open a group
2. Create a meeting space
3. Fill in basic details
4. Add booking rules (including full-day options if desired)
5. Add price details (hourly base price and, if enabled, full-day price)
6. Publish or activate the space

After a meeting space is created, admins can go deeper into its detail tabs:

- bookings for that space
- dynamic pricing
- devices and **adapter device controls** (door locks, keypads)
- **their own door access** to that space (ZenEdge magic link)
- physical-space mapping
- space unavailability
- calendar, timeline, logs, and integrations

### 5. Manage Availability

Availability controls when a space can be booked.

This usually depends on several layers working together:

- the meeting space availability window
- business hours
- blocked or unavailable ranges (space unavailability)
- current bookings
- pricing or booking rules

Typical flow:

1. Set the allowed booking date window
2. Define business hours
3. Add unavailable periods when needed
4. Review how booking slots appear in the booking app

This helps teams prevent bookings outside of valid operating hours. Unavailability windows are shown in the space's timezone so operators in other regions read them correctly.

### 6. Configure Pricing

Pricing in Zenspace starts with a base (hourly) price on the meeting space, then can be extended with a full-day price and adjusted with dynamic pricing rules.

Pricing can be influenced by:

- the hourly base price
- an optional flat full-day price
- fixed values
- multipliers
- adjustments
- date-based rules
- time-based rules

Typical flow:

1. Set the base meeting space price
2. Optionally enable full-day booking and set a full-day price
3. Open dynamic pricing
4. Create a rule
5. Choose when the rule applies
6. Choose how the rule changes the price
7. Save and test the outcome in the booking flow

This allows teams to charge differently for:

- peak hours
- weekends
- date ranges
- special events
- whole-day reservations

### 7. Create Vouchers and Discounts

Vouchers let admins create promotional or controlled discount campaigns.

They can be used to:

- attract new bookings
- reward selected users
- run limited campaigns
- reduce price for certain conditions

Typical flow:

1. Open vouchers
2. Create a voucher
3. Set the code and discount type
4. Add date limits or usage limits
5. Restrict where or when it can be used
6. Publish the voucher

During booking, valid vouchers can reduce the booking amount before payment.

### 8. Manage Bookings

Bookings are one of the most important operational parts of the app.

The bookings area helps teams:

- review all reservations
- check status and payment info
- inspect customer details
- cancel bookings when needed
- understand what is happening across spaces

The bookings list defaults to a **grouped view**, so a multi-item cart booking (see flow 9) shows as one collapsed row rather than many separate lines. From the list toolbar, admins with the right permission can start a booking two ways:

- **New Booking** — the single-booking flow (one group, one space, one date).
- **Book multiple** — the cart-based flow for assembling many bookings at once (flow 9).

Typical flow:

1. Open the bookings page
2. Filter or search bookings
3. Open a booking
4. Review status, customer, and payment details
5. Cancel or update when needed

This is where day-to-day operators usually spend a lot of time.

### 9. Book Multiple Spaces at Once (Cart Booking)

The cart flow lets an admin/organizer assemble **many bookings in one transaction** and check out with a **single payment**. It opens from the **"Book multiple"** button on the bookings page and runs as a 4-step wizard.

What a cart can hold:

- **Multiple spaces** across different groups (items are grouped by space in the cart).
- **Multiple days** — pick a date range and bulk-add one booking per day.
- **Full-day or time-window** items per space+day. The system prevents conflicting combinations (a space+day can't hold both a full day and a timed slot, and overlapping/duplicate timed items are blocked).

The four steps:

1. **Build cart** — pick a group → space → sub-range of days → booking type. For a **time window**, a bounded time picker (driven by the space's aggregated business hours) shows a per-day availability verdict before you add. For a **full day**, one full-day item is added per day in the range. Every change re-prices the whole cart via a debounced preview that flags per-item status.
2. **Organizer** — enter organizer email (required), optional name/phone, choose **pricing mode** (charge the customer, or **complimentary/comp** with a required reason), and add an admin note.
3. **Payment** — for charged carts, pay the whole cart inline with Stripe using a single client secret. A countdown timer shows how long the slots are held (about 30 minutes); comp/free carts skip this step.
4. **Confirmed** — a success screen lists every booking created, with per-item prices, statuses, and links.

**Conflict handling.** Days that are already booked can still be *added* to the cart on purpose — the conflict is surfaced later. Invalid items (past/closed/maintenance) block checkout and must be removed; conflicts do not block. If checkout hits taken slots, a **conflict resolver** appears: for each existing conflicting booking the admin chooses **"Cancel & refund existing"** (bump it) or **"Keep existing."** A new cart item is only created if *every* booking it overlaps is bumped — otherwise that item is skipped so a kept slot is never double-booked. Nothing is ever mass-cancelled; only explicitly chosen bookings are bumped.

### 10. Review and Approve Booking Requests

When a space's public surface is set to **"Submit a request,"** a visitor submits a request (one or more full-day or time-slot items, plus requester name/email/phone/company and notes) instead of paying. These do **not** appear in the normal bookings list — they land in a dedicated **Booking Requests** queue for admin review.

Typical flow:

1. Open **Booking Requests** (a sibling tab beside Bookings)
2. Filter by status — Pending / Approved / Declined / Expired / All
3. Open a request to review it — the detail view runs a read-only "approve dry-run" that shows any conflicts and whether the request is still approvable
4. **Approve** or **Decline**

What approval does:

- **Approve** creates confirmed bookings **comped to $0** and cancels + refunds any live conflicting bookings.
- **Decline** creates nothing and refunds nothing (an optional reason can be given).

A booking request is a pending, unpaid, admin-reviewed intent submitted from the public side; it becomes a real booking only when approved. Approving is gated by the `approve:bookings` permission.

### 11. Approve Full-Day Bookings

Full-day bookings reserve a space's entire business-day window for a given date rather than a time slot. When submitted they enter a **`pending_approval`** status and appear in the bookings pipeline with an **approval card** that shows:

- the requested date (in the space's timezone)
- a price estimate
- an expiry countdown
- any conflicting bookings that would be bumped

The admin can **Approve** or **Decline** from that card. (This is distinct from the Submit-a-Request queue in flow 10 — full-day approval applies to full-day requests already inside the bookings pipeline.)

### 12. Connect Devices and Operational Access

Some spaces are connected to devices such as locks, access systems, or other hardware.

The app includes areas for:

- devices
- device logs
- user-device access

Typical flow:

1. Add or view devices linked to a space
2. Review device status
3. Inspect logs for operational issues
4. Manage which users or bookings get access

This helps bridge the booking system with the physical world. For live door control and per-user door links, see flow 15.

### 13. Connect Physical Spaces to Virtual Spaces

Some teams use a physical-to-virtual mapping flow where the physical space exists in another system, and Zenspace stores the mapping relationship.

This flow lets admins:

- connect a Zenspace meeting space to a physical space
- define the date window for that connection
- review mapping conflicts
- unlink existing mappings if needed

Typical flow:

1. Open a meeting space
2. Go to the physical space tab
3. Connect to physical spaces
4. Choose a physical space
5. Set the mapping period
6. Save the mapping

There is a dedicated guide for this flow:

- `docs/PHYSICAL_SPACE_MAPPING_GUIDE.md`

### 14. Set Up Kiosk Display Groups

A **display group** slices a space group into capacity-bounded buckets (for example "Floor 2") that a kiosk board can rotate through. Each display group has a name, a slug (unique within the space group), a max number of spaces (1–64), a display order, an active flag, an optional description, and an optional booking URL printed as the kiosk's QR code.

Typical flow:

1. Open a group's detail page
2. Open **Display Groups** from the overflow menu
3. Create a display group and set its capacity and order
4. Assign meeting spaces into it (respecting the max)
5. Open the public kiosk board to verify

The public **kiosk display** is a full-screen wayfinding board at `/booking/{orgSlug}/{groupSlug}/kiosk-display` (add `?display_group={slug}` to show just one group). It shows a live per-space status board (Available / Reserved / Maintenance / Closed / Unavailable / Available-at), "booked by" info, today's schedule timeline, a live header clock in the group's timezone, and a QR code to book. It auto-refreshes in real time.

### 15. Device Access & Door Control (ZenEdge)

ZenEdge adds physical door access on top of bookings. There are three main surfaces:

**Adapter device controls (per meeting space).** On a meeting space's detail page, a **"My Access & Controls"** panel lists the physical access devices on that room (door locks, gateways/keypads) for the signed-in admin. Each device card shows a status badge, the admin's keypad **PIN** slot, and interactive controls that the device declares — momentary actions (Unlock / Lock), toggles, sliders, or dropdowns. Pressing a control sends a live command; a Refresh button re-pulls state. Missing credentials or permissions degrade to a soft "Access unavailable" card rather than an error. The meeting-spaces list also has a compact inline **Unlock** button per row where supported.

**My Door Access (per user).** On the profile/user area, a **"My Door Access"** card shows the signed-in user's own ZenEdge door-access **magic links** — one per physical space, each with a status (issued / revoked / expired), a validity window, and **Open link** / **Copy link** actions. The link's secret token is never displayed. Regular users are auto-provisioned when assigned to a space; super-admins are not (that would grant every door), so they get a **"Generate / refresh my access"** button that provisions links across every organization with doors (idempotent). Links spanning multiple organizations are grouped under org headers.

**Per-space door link.** A meeting space's detail page also has a **"My Access to This Space"** section showing the signed-in admin's own door link scoped to just that room.

**User access validity & auto-renew.** Every user has a door-access validity window — **Access from** and **Access until** — plus an **extension mode**:

- **Auto-renew** (default; for permanent staff) — the backend keeps a rolling window alive, so access renews itself.
- **Manual expiry** (for contractors/visitors) — a hard stop on the "until" date; when it passes the user loses door access until an admin extends it.

The create/edit user form has an "Access Validity" card with a segmented Auto-renew / Manual toggle and two date pickers, enforcing that access starts today or later and that "until" is not before "from." On the user detail page, an **Extend access** button opens a dialog with quick presets (+30 days / +90 days / +1 year) or a custom date. Design detail: `docs/USER_ACCESS_VALIDITY_AUTO_RENEW_FE_PLAN.md`.

### 16. Configure Integrations and Notifications

Zenspace includes operational tools beyond basic bookings.

These include:

- integrations
- webhooks
- notifications
- device failure policy
- API keys

Typical flow:

1. Open the relevant settings or integration page
2. Configure the connection or rule
3. Save the configuration
4. Monitor the resulting behavior

These areas are especially useful for teams integrating Zenspace with external platforms or internal workflows.

### 17. Use the Public Booking App

The booking app is the customer-facing side of the product. It runs in one of three modes depending on the entry URL: **Book** (browse and pay), **View availability** (read-only), or **Request spaces** (submit for admin approval).

A customer usually sees:

- available spaces
- date and time pickers (12-hour display)
- pricing summary
- booking form
- payment step (Book mode) or a request form (Request mode)
- confirmation page

Typical customer flow (Book mode):

1. Open the booking page for a group or space
2. Choose a date
3. Choose a time range or a full day
4. Review price
5. Enter booking details
6. Apply voucher if available
7. Pay
8. See booking confirmation

In **Request** mode the customer submits their selection instead of paying; it appears in the admin **Booking Requests** queue (flow 10) for approval.


This flow is powered by the same admin-managed data:

- groups
- meeting spaces
- pricing rules (including full-day price)
- business hours
- availability
- vouchers
- public surface scopes (which modes and hourly/full-day options are offered)

### 18. Common Day-to-Day Admin Journeys

Here are the most common real-world journeys in the app.

#### Launching a New Space

1. Create or select the organization
2. Create a group or location
3. Add a meeting space
4. Upload images
5. Set booking rules and price (hourly and, if desired, full-day)
6. Configure dynamic pricing if needed
7. Set the public surface modes (Book / View / Request) and their scopes
8. Activate the space
9. Test the public booking page

#### Booking Several Spaces for an Event

1. Open bookings → **Book multiple**
2. Pick a date range and add spaces/days (full-day or time windows)
3. Review the cart preview and clear any invalid items
4. Enter organizer details and choose charge vs complimentary
5. Resolve any conflicts at checkout
6. Pay once for the whole cart
7. Review the confirmation list

#### Handling a Booking Request

1. Open **Booking Requests**
2. Review pending requests and their conflicts
3. Approve (creates comped bookings, bumps conflicts) or Decline

#### Handling a Booking Problem

1. Open bookings
2. Search for the customer or booking
3. Review status and payment info
4. Inspect the meeting space and availability
5. Cancel or adjust the booking if necessary

#### Running a Promotion

1. Create a voucher
2. Define usage rules
3. Limit by date or scope if needed
4. Publish the code
5. Monitor booking usage

#### Troubleshooting Space Operations

1. Open the meeting space
2. Review linked devices and use the **My Access & Controls** panel to test the door
3. Inspect device logs
4. Review booking access, per-user door links, or physical mapping
5. Adjust operational settings

## How the App Is Organized for Users

From a user perspective, the app is easiest to understand in this order:

1. Organization
2. Group
3. Meeting space
4. Availability and pricing
5. Booking experience (direct, cart, or request)
6. Operations, device access, and integrations

That means:

- organizations contain groups
- groups contain meeting spaces (and optional kiosk display groups)
- meeting spaces are what customers book
- bookings depend on availability, price, and rules
- devices, door access, and integrations support real-world operations

---

## Modules and screens (detailed reference)

The sections below map **product modules** to **routes (URLs)** and **what each screen is for**. Permission names (for example `read:spaceGroups`) indicate which capability gates access; exact UI may hide actions when a permission is missing.

### Authentication

| URL | Page purpose |
| --- | --- |
| `/auth` | Sign-in: request and verify OTP, then redirect into the app when successful. Public; no organization context yet. Tokens are stored in a centralized auth store; supports safe post-login redirect back to the intended destination. |

**Typical actions:** enter identifier, receive code, verify, land on organization flow or last destination.

### Organization workspace (before dashboard)

These screens run after login and before (or alongside) entering a specific organization's dashboard. They manage **organizations**, **platform-wide system roles**, and **amenities** used when configuring spaces.

#### Organizations

| URL | Page purpose |
| --- | --- |
| `/organization` | List searchable organizations the user can access; entry point after `/` redirect. |
| `/organization/create` | Create a new organization. |
| `/organization/edit/:id` (rewritten to `/organization/:id`) | View and edit one organization. |

**Typical actions:** pick workspace, create org, update org profile and settings exposed on this form.

#### System roles (platform / organization-alias URLs)

Friendly URLs under `/organization/roles/...` rewrite to `/system-roles/...`.

| URL (user-facing alias → actual path) | Page purpose |
| --- | --- |
| `/organization/roles` → `/system-roles` | List system-level roles. |
| `/organization/roles/create` → `/system-roles/create` | Create a system role. |
| `/organization/roles/:id` → `/system-roles/:id` | View a system role. |
| `/organization/roles/edit/:id` → `/system-roles/edit/:id` | Edit a system role. |

**Typical actions:** define which system roles exist and how they are named or configured (super-admin style workflows).

#### Amenities

| URL | Page purpose |
| --- | --- |
| `/amenities` | Manage the catalog of amenities (icons, labels) used when describing meeting spaces. |

**Typical actions:** add, edit, or remove amenity entries used across the product.

### Dashboard home and profile

| URL | Page purpose |
| --- | --- |
| `/dashboard` | Organization dashboard home: summary metrics and entry cards dependent on loaded org context. |
| `/profile` | Signed-in user profile summary. Also surfaces the **My Door Access** card (own ZenEdge magic links; super-admins get a "Generate / refresh my access" action). |

**Typical actions:** orient in the org, jump to common tasks, view own profile and door access.

### Groups

Groups are locations or logical containers for meeting spaces. Permissions such as `read:spaceGroups`, `create:spaceGroups`, and `update:spaceGroups` apply.

| URL | Page purpose |
| --- | --- |
| `/groups` | Table of all groups for the current organization; search and filters. |
| `/groups/create` | Create a new group (address, timezone, presentation, etc.). |
| `/groups/:id` | Group detail: identity, spaces in this group, live timezone clock, **Booking Links** dropdown (Book / View availability / Request spaces), and related operational links. |
| `/groups/:id/edit` | Edit group fields. |
| `/groups/:id/calendar` | Calendar-oriented view of bookings / availability for this group. |
| `/groups/:id/display-groups` | Manage **kiosk display groups**: create/edit buckets, set capacity and order, assign meeting spaces. Requires `read:kiosk_display_groups`. |

**Typical actions:** create location, open a group, add or open meeting spaces from the group, launch the public booking app in any mode, manage kiosk display groups, adjust group-level presentation, inspect group calendar.

### Meeting spaces

There is **no** `/meeting-spaces` index route. Spaces are reached from **group detail** or direct links. Creating and editing use dedicated routes; everything else is tabs on the detail page.

| URL | Page purpose |
| --- | --- |
| `/meeting-spaces/create` | Wizard-style form: new meeting space (group, capacity, pricing base, full-day options, rules, images, amenities, public surface scopes, etc.). Requires `create:meetingSpaces`. |
| `/meeting-spaces/:msId` | **Detail hub** for one space (`msId` is the space slug in practice). Tabbed UI (see below). Multiple tab areas check fine-grained permissions. |
| `/meeting-spaces/:msId/edit` | Edit core meeting space fields. Requires `update:meetingSpaces`. |

**Meeting space detail tabs / sections** (same URL, tab control):

| Tab / section | What it covers |
| --- | --- |
| **Overview** | Hero imagery, day metrics (bookings, revenue, utilization), status, embedded booking summaries and device log toggle, links to public booking app. |
| **Pricing** | Hourly base price, optional full-day price, and **dynamic pricing** rules (no separate `/dynamic-pricing` route). |
| **Policies** | Booking policies and related configuration, including full-day booking and public surface scopes. |
| **Amenities** | Assigned amenities for this space. |
| **Calendar** | Space-level calendar / availability visualization. |
| **Timeline** | Status timeline and operational history. |
| **My Access & Controls** | ZenEdge adapter devices for this room: status, keypad PIN, and live controls (Unlock/Lock, toggles, sliders, dropdowns). |
| **My Access to This Space** | The signed-in admin's own door magic link scoped to this room (open / copy / validity). |
| **Connections** | Third-party or integration cards tied to the space. |
| **Physical Space** | Map Zenspace meeting space ↔ external physical space over date ranges; conflicts. See `docs/PHYSICAL_SPACE_MAPPING_GUIDE.md`. |
| **Notifications** | Space-scoped notifications list and management entry points. |
| **Webhooks** | Space-scoped webhooks list and management entry points. |

**Typical actions:** open "Booking App" for the public URL, edit space, configure pricing (hourly + full-day) and rules, manage policies and public surface scopes, inspect calendar/timeline, control door devices, open own door link, manage integrations, physical mapping, notifications, and webhooks.

### Bookings

Organization-wide booking operations. Gated by `read:bookings` (and related actions where implemented). Creating admin bookings requires `create:admin_booking`.

| URL | Page purpose |
| --- | --- |
| `/bookings` | List with filters and summary cards; defaults to **grouped view** (a cart shows as one row). Toolbar offers **New Booking** (single) and **Book multiple** (cart). |
| `/bookings/:id` | Single booking detail: customer, payment, status, space context. Full-day requests show a **booking approval card** (Approve / Decline). |

**Cart booking (modal, no dedicated route):** a 4-step wizard — Build cart → Organizer → Payment → Confirmed — for booking many spaces/days in one Stripe transaction, with checkout-time conflict resolution. Backend contract: `docs/CART_BOOKING_FRONTEND_IMPLEMENTATION_PLAN.md`.

**Typical actions:** filter by date or status, open a booking, approve/decline full-day requests, start single or cart bookings, cancel or reconcile from detail.

### Booking requests

The Submit-a-Request (SRF) queue. Requests come from the public booking app's **Request** mode and are reviewed here. Approving is gated by `approve:bookings`.

| URL | Page purpose |
| --- | --- |
| `/booking-requests` | Queue with status tabs (Pending / Approved / Declined / Expired / All); sibling tab beside `/bookings`. |
| `/booking-requests/:id` | Request detail with a read-only approve dry-run (conflicts, approvable/expired flags). Approve creates comped bookings and bumps conflicts; Decline creates nothing. |

**Typical actions:** filter by status, review a request and its conflicts, approve or decline.

### Devices, device logs, and user device access

| URL | Page purpose |
| --- | --- |
| `/devices/:id` | Device detail for one device (identifier in path): logs and status focused on that device. |
| `/device-logs` | Organization-wide device log stream / table with filters. |
| `/user-device-access` | Who has access to which devices (org-scoped "me" / access list patterns in the UI). |

**Typical actions:** troubleshoot a single device, scan org-wide logs, audit access grants. Live per-space door control lives on the meeting-space detail page (**My Access & Controls**).

### Users

| URL | Page purpose |
| --- | --- |
| `/user` | List organization users (`read:orgUsers`). |
| `/user/create` | Invite or create user (`create:orgUsers`), role selection, and **Access Validity** (auto-renew vs manual expiry, from/until dates). |
| `/user/:id` | User detail; shows access-validity mode + window and an **Extend access** action (+30d / +90d / +1y or custom). |
| `/user/:id/edit` | Edit user (`update:orgUsers`), including access validity. |

**Bulk assign to spaces (modal):** add one user to **multiple meeting spaces at once** with per-channel notification preferences (push/email/SMS). Returns partial-success results; the user must already belong to each space's organization.

**Typical actions:** onboard staff, adjust roles, set door-access validity, extend access, bulk-assign to spaces.

### Vouchers

| URL | Page purpose |
| --- | --- |
| `/vouchers` | List vouchers (`read:vouchers`). |
| `/vouchers/create` | Create voucher (`create:vouchers`). |
| `/vouchers/:id` | Voucher detail. |
| `/vouchers/:id/edit` | Edit voucher (`update:vouchers`). |

**Typical actions:** define codes, discount types, validity windows, usage limits, scope; verify behavior in the booking app.

### Integrations

| URL | Page purpose |
| --- | --- |
| `/integrations` | Third-party connections for the org (`read:thirdPartyConnections`); connect or manage integrations via modals/cards. |

**Typical actions:** connect services, review connection health, open related configuration.

### Organization roles (dashboard)

Distinct from **system roles** under `/system-roles`. These are **roles inside the current organization**.

| URL | Page purpose |
| --- | --- |
| `/roles` | List org roles (`read:roles`). |
| `/roles/create` | Create org role (`create:roles`). |
| `/roles/:id` | Role detail. |
| `/roles/edit/:id` | Edit org role (`update:roles`); may guard system-backed roles. |

**Typical actions:** define permission bundles for org users, assign during user management.

### Testing (in-app feature)

The **Testing** sidebar item is an **operational / load-testing style feature** inside the product, not automated frontend tests.

| URL | Page purpose |
| --- | --- |
| `/testing` | List testing jobs or scenarios. |
| `/testing/create` | Configure and start a new testing job. |
| `/testing/:id` | Job detail and results / reports. |

### Settings

| URL | Page purpose |
| --- | --- |
| `/settings` | Settings hub: navigation shell to sub-settings. |
| `/settings/stripe` | Stripe connection and payment settings (`read:payments` / save flows). |
| `/settings/organization` | Current organization settings shortcut. |
| `/settings/organization/:id` | Edit organization by id (fetch/update form). |
| `/settings/api-keys` | List API keys (`read:apiKeys`). |
| `/settings/api-keys/create` | Create key (`create:apiKeys`); often includes one-time secret display. |
| `/settings/api-keys/:id` | Key detail. |
| `/settings/api-keys/edit/:id` | Edit key metadata or rotation flows (`update:apiKeys`). |
| `/settings/device-failure-policy` | Policy for device failures (how the system should behave when devices error). |
| `/settings/scheduled-jobs` | Scheduled/recurring job configuration. |

**Typical actions:** connect Stripe, rotate API keys, adjust org billing identity, set failure-handling policy, configure scheduled jobs.

### Notifications and webhooks (standalone detail pages)

These pages are usually opened from links inside **meeting space** tabs or lists.

| URL | Page purpose |
| --- | --- |
| `/notifications/:id` | Notification detail (`read:notifications`). Alias: `/meeting-spaces/notifications/:id`. |
| `/webhooks/:id` | Webhook detail (`read:webhooks`). Alias: `/meeting-spaces/webhooks/:id`. |

### Public booking app

Rewrites expose **`/booking`** to the booking module. Slugs are **organization**, **group**, and **meeting space** slugs. The path segment selects the mode.

| URL | Page purpose |
| --- | --- |
| `/booking/{orgSlug}/{groupSlug}/` | **Book** mode: browse and book directly. |
| `/booking/view/{orgSlug}/{groupSlug}` | **View availability** mode: read-only, no booking. |
| `/booking/request/{orgSlug}/{groupSlug}` | **Request** (SRF) mode: submit a request for admin approval. |
| `/booking/{orgSlug}/{groupSlug}/{spaceSlug}` | Direct landing on one bookable space: slots (or full day), pricing, booking form. |
| `/booking/{orgSlug}/{groupSlug}/kiosk-display` | Full-screen kiosk wayfinding board (live status, schedule, clock, QR). `?display_group={slug}` scopes to one display group. |
| `/booking/payment` | Payment continuation or retry (Stripe); depends on session/booking context. |
| `/booking/details/:id` | Post-booking confirmation / read-only booking detail for the customer. |

**Typical actions:** pick slot or full day, enter details, apply voucher, pay (Book mode) or submit a request (Request mode), view confirmation. Embed scenarios may use `docs/booking-iframe-embed.md`.

### Airtame

| URL | Page purpose |
| --- | --- |
| `/airtame` | Dedicated surface for Airtame-related display / pairing flows (meeting space context via query or route params as implemented). Public-facing module with its own layout. |

## Related Docs

For deeper feature-specific details, see:

- `docs/PHYSICAL_SPACE_MAPPING_GUIDE.md`
- `docs/CART_BOOKING_FRONTEND_IMPLEMENTATION_PLAN.md`
- `docs/USER_ACCESS_VALIDITY_AUTO_RENEW_FE_PLAN.md`
- `docs/booking-iframe-embed.md`
- `docs/MEETING_SPACE_AVAILABILITY_IMPLEMENTATION_PLAN.md`
- `docs/GROUP_AVAILABILITY_IMPLEMENTATION_PLAN.md`
- `docs/VOUCHER_IMPLEMENTATION_PLAN.md`
- `docs/IMAGE_SPECIFICATIONS.md`
- `docs/TECHNICAL_ARCHITECTURE.md` (technical onboarding)
- `zs-admin.md` at repository root (agent-oriented route and architecture map)

## Summary

Zenspace is a complete space operations and booking platform spanning booking management (ZenCore) and physical door access (ZenEdge).

Admins use it to build and manage the workspace structure, configure pricing (hourly and full-day) and availability, run single or multi-space cart bookings, review booking requests, control door access, connect operational systems, and monitor bookings.

Customers use it to discover spaces and either book and pay directly, view availability, or submit a request for approval.

If someone is new to the product, the easiest way to learn it is:

1. Start with organization and group setup
2. Learn how meeting spaces are configured (including full-day and public surface modes)
3. Understand how pricing and availability affect booking
4. Follow the customer booking flow from start to finish
5. Explore cart booking, booking requests, and full-day approvals
6. Use bookings, device access, kiosk displays, and settings for daily operations

For a **screen-by-screen map**, use the **Modules and screens (detailed reference)** section above.
