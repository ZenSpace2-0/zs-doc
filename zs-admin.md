# zs-admin Frontend Skill Document

## 1. Title and Purpose

This document is consolidated AI context for the `zs-admin` frontend codebase.

It is designed for coding agents such as Claude, Cursor, and GPT so they can:

- understand the app architecture quickly
- make safe feature changes
- preserve critical behavior across admin and booking flows
- navigate the repo without re-discovering core patterns every time

This repository contains two major frontend surfaces in one codebase:

- the protected admin application
- the public booking application

Terminology used throughout this document:

- **ZenCore** = the backend/API platform the frontend talks to
- **ZenEdge** = the MRD/device platform and related physical/device operations

Important architectural reality:

- the frontend talks **directly to ZenCore**
- ZenCore **proxies or mediates** most ZenEdge-facing behavior
- there is **no separate direct browser client** for ZenEdge found in this repo

## 2. Agent Ingestion Order

Read the codebase in this order before making meaningful frontend changes.

### A. App bootstrap

1. `src/app/layout.tsx`
2. `src/providers/auth-provider.tsx`
3. `src/app/(modules)/(authenticated-modules)/layout.tsx`
4. `src/providers/authenticated-modules-provider.tsx`
5. `src/app/(modules)/(authenticated-modules)/(dashboard)/layout.tsx`
6. `src/providers/dashboard-data-provider.tsx`
7. `src/providers/redux-provider.tsx`
8. `src/providers/stripe-provider.tsx`

### B. Router definitions and URL behavior

1. `next.config.ts`
2. `src/middleware.ts`
3. `src/app/(modules)/`
4. `src/common/global-components/sidebar-navbar/app-sidebar.tsx`
5. `src/common/global-components/url-pathname/url-pathname.tsx`
6. `src/common/helpers/helper-fn.ts` for `appendOrgIdToUrl`

### C. State management

1. `src/redux/store.ts`
2. `src/hooks/redux-hook.ts`
3. `src/common/reducers/dashboard-reducers.ts`
4. `src/common/reducers/common-reducers.ts`
5. feature `reducers/reducers.ts` files under `src/app/(modules)/(authenticated-modules)/...`

### D. API client layer

1. `src/common/helpers/api.ts`
2. feature `actions/services.ts`
3. feature `actions/actions.ts`
4. `src/common/interface/interface.ts`

### E. Feature modules

Suggested reading order:

1. `src/app/(modules)/(authenticated-modules)/(dashboard)/groups/`
2. `src/app/(modules)/(authenticated-modules)/(dashboard)/meeting-spaces/`
3. `src/app/(modules)/(authenticated-modules)/(dashboard)/bookings/`
4. `src/app/(modules)/(authenticated-modules)/(dashboard)/dynamic-pricing/`
5. `src/app/(modules)/(authenticated-modules)/(dashboard)/vouchers/`
6. `src/app/(modules)/(authenticated-modules)/(dashboard)/devices/`
7. `src/app/(modules)/(authenticated-modules)/(dashboard)/device-logs/`
8. `src/app/(modules)/(authenticated-modules)/(dashboard)/user-device-access/`
9. `src/app/(modules)/(authenticated-modules)/(dashboard)/integrations/`
10. `src/app/(modules)/(authenticated-modules)/(dashboard)/settings/`

### F. Shared UI and design system

1. `src/components/ui/`
2. `src/common/global-components/`
3. `src/components/ui/calendar.tsx`
4. `src/common/global-components/date-selector/`
5. `src/components/ui/sidebar.tsx`

### G. Environment and config

1. `package.json`
2. `tsconfig.json`
3. `eslint.config.mjs`
4. `postcss.config.mjs`
5. `.env.local`
6. `.env.development`
7. `.env.production`

### H. Tests / e2e / verification

1. `package.json` scripts
2. `src/app/(modules)/(authenticated-modules)/(dashboard)/testing/`
3. any `*.test.*` / `*.spec.*` if added later

Current reality:

- there is **no conventional automated unit/integration/e2e test suite** found in this repo
- the `testing/` dashboard module is an **app feature**, not a frontend test harness

## 3. Frontend Architecture Overview

### Framework and runtime

- Next.js `16.0.1`
- React `19.2.0`
- TypeScript `5`
- Tailwind CSS `4`
- App Router with route groups

### Rendering strategy

Primary rendering model:

- client-heavy App Router application
- most business pages are `use client`
- auth, Redux, Stripe, and realtime behavior are client-driven

Effective strategy by surface:

- **Admin app**: mostly CSR after client bootstrap
- **Booking app**: also mostly CSR, though routed through Next App Router
- **Route-level code splitting**: provided by App Router page boundaries
- **Suspense**: used around some layouts/providers, especially dashboard bootstrap

There is no evidence of significant ISR/SSG content strategy in the business application.

### Module boundaries and folder conventions

Primary structure:

```text
src/
  app/
    (modules)/
      auth/
      booking-app/
      airtame/
      (authenticated-modules)/
        (for-organization)/
        (dashboard)/
  common/
  components/
  hooks/
  providers/
  redux/
```

Common feature convention:

```text
feature/
  (views)/
  actions/
  components/
  reducers/
  schema/
  model-interfaces/
  helpers/
```

Meaning:

- `(views)/` = route pages
- `actions/services.ts` = ZenCore request layer
- `actions/actions.ts` = orchestration + Redux dispatch
- `reducers/reducers.ts` = slice
- `schema/` = RHF + Zod validation
- `model-interfaces/` = response/request typing and models
- `components/` = feature UI
- `helpers/` = transforms/config/utilities

### How frontend talks to ZenCore and ZenEdge

#### ZenCore communication

The frontend talks to ZenCore through:

- HTTP via `src/common/helpers/api.ts`
- WebSocket via `src/hooks/use-realtime.ts`

ZenCore HTTP base URL:

- `NEXT_PUBLIC_BACKEND_URL`

ZenCore auth:

- `Authorization: Bearer <auth_token>` when token exists in `localStorage`

ZenCore request pattern:

- `zenspaceApi.request(url, { method, data })`
- JSON by default
- `FormData` supported for uploads
- `401` clears token and redirects to `/auth?next=...`

ZenCore websocket base:

- `NEXT_PUBLIC_WEBSOCKET_SERVICE_URL`
- fallback derived from `NEXT_PUBLIC_BACKEND_URL` by replacing `http` with `ws`

#### ZenEdge communication

There is no direct browser-to-ZenEdge client found in the codebase.

ZenEdge-related operations are handled through ZenCore endpoints, including:

- MRD session creation
- physical space listing and mappings
- device and device-log views
- user device access
- device failure policy

So the frontend’s practical architecture is:

- **ZenCore = direct integration point**
- **ZenEdge = mediated through ZenCore**

## 4. Route / Page / Screen Inventory (Complete)

### URL behavior notes

- Route groups like `(modules)` and `(dashboard)` do not appear in URLs
- `/booking/*` is exposed through `next.config.ts` rewrites to `/booking-app/*`
- `/organization/roles/*` rewrites to `/system-roles/*`
- `/organization/edit/:id` rewrites to `/organization/:id`

### Route inventory

| Browser path | Filesystem page | Owning module | Access level | Primary data dependencies | Loading / error / empty behavior |
|---|---|---|---|---|---|
| `/auth` | `src/app/(modules)/auth/page.tsx` | Auth | Public | `requestOTPSevice`, `verifyOTPSevice` | Form loading, OTP validation errors, redirect on success |
| `/organization` | `src/app/(modules)/(authenticated-modules)/(for-organization)/organization/(views)/page.tsx` | Organization | Protected | org list APIs, list view | List loading, searchable list, empty org state |
| `/organization/create` | `src/app/(modules)/(authenticated-modules)/(for-organization)/organization/(views)/create/page.tsx` | Organization | Protected | org create services | Form submit loading, inline validation, no empty state |
| `/organization/edit/:id` -> `/organization/:id` | `src/app/(modules)/(authenticated-modules)/(for-organization)/organization/(views)/[id]/page.tsx` | Organization | Protected | org fetch/update services | Loads existing org, form validation, error on failed fetch |
| `/organization/roles` -> `/system-roles` | `src/app/(modules)/(authenticated-modules)/(for-organization)/system-roles/(views)/page.tsx` | System roles | Protected, effectively super-admin UX | role list services | List loading, empty state if no roles |
| `/organization/roles/create` -> `/system-roles/create` | `src/app/(modules)/(authenticated-modules)/(for-organization)/system-roles/(views)/create/page.tsx` | System roles | Protected, effectively super-admin UX | role form services | Form validation, submit loading |
| `/organization/roles/:id` -> `/system-roles/:id` | `src/app/(modules)/(authenticated-modules)/(for-organization)/system-roles/(views)/[id]/page.tsx` | System roles | Protected, effectively super-admin UX | role detail service | Detail loading, error if missing role |
| `/organization/roles/edit/:id` -> `/system-roles/edit/:id` | `src/app/(modules)/(authenticated-modules)/(for-organization)/system-roles/(views)/edit/[id]/page.tsx` | System roles | Protected, effectively super-admin UX | `getRoleByIdService`, update service | Form loading + validation |
| `/amenities` | `src/app/(modules)/(authenticated-modules)/(for-organization)/amenities/(views)/page.tsx` | Amenities | Protected, super-admin-style nav exposure | amenities services, list view | Table loading, empty state, CRUD toasts |
| `/dashboard` | `src/app/(modules)/(authenticated-modules)/(dashboard)/dashboard/(views)/page.tsx` | Dashboard home | Protected | `getDashboardAction` | Dashboard loading cards, org bootstrap dependency |
| `/profile` | `src/app/(modules)/(authenticated-modules)/(dashboard)/profile/(views)/page.tsx` | Profile | Protected | `useAuth()` | Lightweight profile rendering, minimal empty state |
| `/groups` | `src/app/(modules)/(authenticated-modules)/(dashboard)/groups/(views)/page.tsx` | Groups | Protected + `read:spaceGroups` | group list APIs, ListViewTable | Table loading, search/filter, empty state |
| `/groups/create` | `src/app/(modules)/(authenticated-modules)/(dashboard)/groups/(views)/create/page.tsx` | Groups | Protected + `create:spaceGroups` | group form services | Form loading, validation, create flow |
| `/groups/:id` | `src/app/(modules)/(authenticated-modules)/(dashboard)/groups/(views)/[id]/page.tsx` | Groups | Protected + `read:spaceGroups` | group detail, group meeting spaces, related lists | Detail loading, nested tables, permission-gated actions |
| `/groups/:id/edit` | `src/app/(modules)/(authenticated-modules)/(dashboard)/groups/(views)/[id]/edit/page.tsx` | Groups | Protected + `update:spaceGroups` | group fetch/update services | Form load, validation, update flow |
| `/groups/:id/calendar` | `src/app/(modules)/(authenticated-modules)/(dashboard)/groups/(views)/[id]/calendar/page.tsx` | Groups calendar | Protected | group booking/availability actions | Calendar loading, filtered booking states |
| `/meeting-spaces/create` | `src/app/(modules)/(authenticated-modules)/(dashboard)/meeting-spaces/(views)/create/page.tsx` | Meeting spaces | Protected + `create:meetingSpaces` | meeting space create services, amenities, groups | Very large form, upload and validation states |
| `/meeting-spaces/:msId` | `src/app/(modules)/(authenticated-modules)/(dashboard)/meeting-spaces/(views)/[msId]/page.tsx` | Meeting spaces | Protected + multiple tab-level permissions | meeting space detail, bookings, devices, notifications, webhooks, physical mapping | Heavy detail loading, tab-level permission fallbacks, empty states in nested sections |
| `/meeting-spaces/:msId/edit` | `src/app/(modules)/(authenticated-modules)/(dashboard)/meeting-spaces/(views)/[msId]/edit/page.tsx` | Meeting spaces | Protected + `update:meetingSpaces` | meeting space fetch/update services | Large edit form loading, validation, upload states |
| `/bookings` | `src/app/(modules)/(authenticated-modules)/(dashboard)/bookings/(views)/page.tsx` | Bookings | Protected + `read:bookings` | bookings list services | Table loading, summary cards, filter bar, empty state |
| `/bookings/:id` | `src/app/(modules)/(authenticated-modules)/(dashboard)/bookings/(views)/[id]/page.tsx` | Bookings | Protected + `read:bookings` | `getBookingByIdService` | Detail loading, error fallback if booking missing |
| `/devices/:id` | `src/app/(modules)/(authenticated-modules)/(dashboard)/devices/(views)/[id]/page.tsx` | Devices | Protected | device logs by device | Detail loading, log empty state |
| `/device-logs` | `src/app/(modules)/(authenticated-modules)/(dashboard)/device-logs/(views)/page.tsx` | Device logs | Protected | org-scoped device logs | Table loading, filters, empty state |
| `/user-device-access` | `src/app/(modules)/(authenticated-modules)/(dashboard)/user-device-access/(views)/page.tsx` | User device access | Protected | `/v1/user-device-access/me` list | List/card loading, empty state |
| `/vouchers` | `src/app/(modules)/(authenticated-modules)/(dashboard)/vouchers/(views)/page.tsx` | Vouchers | Protected + `read:vouchers` | voucher list services | Table loading, empty state, permission-gated create |
| `/vouchers/create` | `src/app/(modules)/(authenticated-modules)/(dashboard)/vouchers/(views)/create/page.tsx` | Vouchers | Protected + `create:vouchers` | voucher form services | Form validation and submit loading |
| `/vouchers/:id` | `src/app/(modules)/(authenticated-modules)/(dashboard)/vouchers/(views)/[id]/page.tsx` | Vouchers | Protected + `read:vouchers` | voucher detail service | Detail loading, not-found style error cases |
| `/vouchers/:id/edit` | `src/app/(modules)/(authenticated-modules)/(dashboard)/vouchers/(views)/[id]/edit/page.tsx` | Vouchers | Protected + `update:vouchers` | voucher fetch/update services | Form load + validation |
| `/integrations` | `src/app/(modules)/(authenticated-modules)/(dashboard)/integrations/(views)/page.tsx` | Integrations | Protected + `read:thirdPartyConnections` | third-party connection list | Card/list loading, empty state, modal actions |
| `/roles` | `src/app/(modules)/(authenticated-modules)/(dashboard)/roles/(views)/page.tsx` | Roles | Protected + `read:roles` | org role list services | List loading, empty state, permission-gated create |
| `/roles/create` | `src/app/(modules)/(authenticated-modules)/(dashboard)/roles/(views)/create/page.tsx` | Roles | Protected + `create:roles` | role form services | Form validation and submit loading |
| `/roles/:id` | `src/app/(modules)/(authenticated-modules)/(dashboard)/roles/(views)/[id]/page.tsx` | Roles | Protected + `read:roles` | role detail service | Detail loading |
| `/roles/edit/:id` | `src/app/(modules)/(authenticated-modules)/(dashboard)/roles/(views)/edit/[id]/page.tsx` | Roles | Protected + `update:roles` | role fetch/update services | Form load, system-role guard, validation |
| `/user` | `src/app/(modules)/(authenticated-modules)/(dashboard)/user/(views)/page.tsx` | Users | Protected + `read:orgUsers` | user list services | Table loading, empty state |
| `/user/create` | `src/app/(modules)/(authenticated-modules)/(dashboard)/user/(views)/create/page.tsx` | Users | Protected + `create:orgUsers` | user form services, roles async select | Form validation, submit loading |
| `/user/:id` | `src/app/(modules)/(authenticated-modules)/(dashboard)/user/(views)/[id]/page.tsx` | Users | Protected + `read:orgUsers` | user detail service | Detail loading, read-only state |
| `/user/:id/edit` | `src/app/(modules)/(authenticated-modules)/(dashboard)/user/(views)/[id]/edit/page.tsx` | Users | Protected + `update:orgUsers` | user fetch/update services | Form load + validation |
| `/testing` | `src/app/(modules)/(authenticated-modules)/(dashboard)/testing/(views)/page.tsx` | Testing feature | Protected | internal testing/load-test list | List loading, empty state |
| `/testing/create` | `src/app/(modules)/(authenticated-modules)/(dashboard)/testing/(views)/create/page.tsx` | Testing feature | Protected | create testing job services | Form validation and submit loading |
| `/testing/:id` | `src/app/(modules)/(authenticated-modules)/(dashboard)/testing/(views)/[id]/page.tsx` | Testing feature | Protected | testing detail services | Detail loading, report fetch states |
| `/settings` | `src/app/(modules)/(authenticated-modules)/(dashboard)/settings/(views)/page.tsx` | Settings hub | Protected | settings shell content | Hub loading minimal; page-level permission guard currently commented out |
| `/settings/stripe` | `src/app/(modules)/(authenticated-modules)/(dashboard)/settings/stripe/page.tsx` | Stripe settings | Protected + `read:payments` | org Stripe config, Stripe provider | Settings load + save states |
| `/settings/organization` | `src/app/(modules)/(authenticated-modules)/(dashboard)/settings/organization/page.tsx` | Org settings | Protected | current org fetch | Loading card/form state |
| `/settings/organization/:id` | `src/app/(modules)/(authenticated-modules)/(dashboard)/settings/organization/[id]/page.tsx` | Org settings | Protected | org fetch/update services | Form load, validation |
| `/settings/api-keys` | `src/app/(modules)/(authenticated-modules)/(dashboard)/settings/api-keys/(views)/page.tsx` | API keys | Protected + `read:apiKeys` | API key list services | Table loading, empty state |
| `/settings/api-keys/create` | `src/app/(modules)/(authenticated-modules)/(dashboard)/settings/api-keys/(views)/create/page.tsx` | API keys | Protected + `create:apiKeys` | API key create service | Form validation, secret display flow |
| `/settings/api-keys/:id` | `src/app/(modules)/(authenticated-modules)/(dashboard)/settings/api-keys/(views)/[id]/page.tsx` | API keys | Protected + `read:apiKeys` | API key detail service | Detail loading |
| `/settings/api-keys/edit/:id` | `src/app/(modules)/(authenticated-modules)/(dashboard)/settings/api-keys/(views)/edit/[id]/page.tsx` | API keys | Protected + `update:apiKeys` | API key fetch/update services | Form load + validation |
| `/settings/device-failure-policy` | `src/app/(modules)/(authenticated-modules)/(dashboard)/settings/device-failure-policy/(views)/page.tsx` | Device failure policy | Protected | failure-policy services | Loading and save state |
| `/notifications/:id` | `src/app/(modules)/(authenticated-modules)/(dashboard)/notifications/(views)/[id]/page.tsx` | Notifications | Protected + `read:notifications` | notification detail service | Detail loading |
| `/meeting-spaces/notifications/:id` | rewrite alias -> above | Notifications | Protected + `read:notifications` | same as above | same as above |
| `/webhooks/:id` | `src/app/(modules)/(authenticated-modules)/(dashboard)/webhooks/(views)/[id]/page.tsx` | Webhooks | Protected + `read:webhooks` | webhook detail service | Detail loading |
| `/meeting-spaces/webhooks/:id` | rewrite alias -> above | Webhooks | Protected + `read:webhooks` | same as above | same as above |
| `/booking/:orgSlug/...` | `src/app/(modules)/booking-app/(views)/[...slugs]/page.tsx` | Public booking app | Public | org/group availability actions | Public loading, availability empty states, selection fallbacks |
| `/booking/:orgSlug/:groupSlug/:spaceSlug` | `src/app/(modules)/booking-app/(views)/[orgSlug]/[groupSlug]/[spaceSlug]/page.tsx` | Public booking app | Public | org/group/meeting space actions, availability slots, pricing | Loading, unavailable-state card, inline booking validation |
| `/booking/payment` | `src/app/(modules)/booking-app/(views)/payment/page.tsx` | Public booking app | Public | booking detail + Stripe payment retry | Payment loading, success/failure status |
| `/booking/details/:id` | `src/app/(modules)/booking-app/(views)/details/[id]/page.tsx` | Public booking app | Public | booking detail service | Confirmation/detail loading and missing-booking fallback |
| `/airtame` | `src/app/(modules)/airtame/(views)/page.tsx` | Airtame | Public | meeting space by id + realtime | Loading, empty-ish informational states |

### Route gaps and important notes

| URL | Status | Notes |
|---|---|---|
| `/` | redirect | middleware redirects to `/organization` |
| `/meeting-spaces` | no page file found | sidebar treats it as related active URL, but there is no list/index page |
| `/booking` | rewrite exists but route may be fragile | internal booking root is backed by `[...slugs]`, which is a non-optional catch-all; verify actual runtime behavior before changing |

## 5. Component Architecture (Complete)

### Feature components vs shared components

#### Feature components

Location:

- `src/app/(modules)/(authenticated-modules)/**/components/`
- `src/app/(modules)/booking-app/components/`

Responsibilities:

- page-specific or module-specific UI
- tables, forms, tabs, and domain widgets
- orchestration around feature services and Redux data

Examples:

- `src/app/(modules)/(authenticated-modules)/(dashboard)/meeting-spaces/components/`
- `src/app/(modules)/(authenticated-modules)/(dashboard)/bookings/components/`
- `src/app/(modules)/booking-app/components/booking-form.tsx`

#### Shared components

Base design primitives:

- `src/components/ui/`

App-level reusable composites:

- `src/common/global-components/`

Important shared areas:

- `sidebar-navbar/`
- `permission/`
- `modal/`
- `list-view/`
- `date-selector/`
- `time-dragger/`
- `google-places-autocomplete/`
- `phone-input/`

Guideline:

- prefer `src/common/global-components/` first for app-level interaction patterns
- prefer `src/components/ui/` for low-level primitives
- avoid one-off components when an app-wide variant already exists

### Layout system and shell/navigation

Root shells:

- root app layout: `src/app/layout.tsx`
- authenticated layout: `src/app/(modules)/(authenticated-modules)/layout.tsx`
- dashboard shell: `src/app/(modules)/(authenticated-modules)/(dashboard)/layout.tsx`
- organization shell: `src/app/(modules)/(authenticated-modules)/(for-organization)/layout.tsx`
- booking app shell: `src/app/(modules)/booking-app/layout.tsx`

Dashboard navigation:

- `src/common/global-components/sidebar-navbar/app-sidebar.tsx`
- `src/common/global-components/sidebar-navbar/nav-main.tsx`
- `src/common/global-components/sidebar-navbar/nav-user.tsx`
- `src/common/global-components/sidebar-navbar/team-switcher.tsx`

Breadcrumb/navigation context:

- `src/common/global-components/url-pathname/url-pathname.tsx`

### Design system, theme, and tokens

Design system:

- shadcn-style components under `src/components/ui/`
- utility composition through `src/lib/utils.ts`

Styling:

- Tailwind CSS 4
- utility-first classes used directly in components

Theme status:

- `next-themes` exists as a dependency
- some theme-aware code exists
- root layout does **not** currently install a visible `ThemeProvider`
- assume theme support is partial/incomplete unless confirmed in the task

### Form strategy and validation approach

Primary form stack:

- React Hook Form
- Zod
- `zodResolver`
- shadcn `Form`, `FormField`, `FormItem`, `FormMessage`

Where schemas live:

- feature-local `schema/` directories

Patterns:

- edit/create forms usually live in feature `components/` and are reused by create/edit routes
- async selects often use shared async paginate components
- some inputs still use raw HTML inputs where appropriate

## 6. State Management and Data Flow

### Global state stores

Primary store:

- Redux Toolkit in `src/redux/store.ts`

Registered slices:

- `common`
- `user`
- `dashboard`
- `roles`
- `groups`
- `amenities`
- `apiKeys`
- `meetingSpaces`
- `webhooks`
- `notifications`
- `bookings`
- `devices`
- `vouchers`

Typed hooks:

- `src/hooks/redux-hook.ts`

### Server state and caching

There is **no React Query or SWR** in this codebase.

Current data-loading model:

- imperative service calls
- Redux slice hydration through action files
- page/component-level local loading states

Caching behavior:

- mostly Redux-held resource state
- repeated page fetches and manual refetching
- no generic stale-while-revalidate layer

### Mutation patterns and invalidation rules

Common mutation pattern:

1. UI submits through feature form
2. feature `actions/services.ts` calls ZenCore
3. page or thunk dispatches slice updates or triggers refetches
4. toasts and/or inline errors present outcome

Invalidation/refetch style:

- manual refetch after create/update/delete
- list views re-fetch based on pagination/search/filter changes
- `hardRefresh` in `common` slice is used as a coarse refresh trigger in some areas

### Derived state and cross-module synchronization

Examples:

- dashboard shell depends on `dashboard.organization`
- permissions come from `dashboard.organizationPermissions`
- Stripe provider depends on `dashboard.organization`
- booking app reuses admin actions/models for groups, spaces, bookings, pricing

Important implication:

- modules are not cleanly isolated by surface
- changing shared models/services can affect both admin and public booking flows

### Realtime updates

Path:

- `src/hooks/use-realtime.ts`

Transport:

- Socket.IO websocket

Observed update domains:

- meeting space status
- third-party connections
- bookings
- device approvals/status/alerts

Pattern:

- realtime events often trigger Redux refetches
- not all events update normalized store state directly

Known caveats:

- some `socket.off()` cleanup names do not match the subscribed event names
- some hooks may have stale closure or dependency-array issues

## 7. API Contract Map (FE Perspective)

## 7.1 Shared API behavior

| Item | Current behavior |
|---|---|
| Primary client | `src/common/helpers/api.ts` |
| ZenCore base URL | `NEXT_PUBLIC_BACKEND_URL` |
| Auth | `Authorization: Bearer <auth_token>` from `localStorage` when available |
| Body handling | JSON by default; `FormData` supported |
| Success handling | returns parsed JSON envelope |
| Failure handling | returns parsed error JSON when possible |
| 401 behavior | clears `auth_token`, redirects to `/auth?next=...` |
| Automatic retry | none found in generic client |
| Public pages | may still send token if user is logged in elsewhere in the browser |

### Response envelope pattern

Common response shape is represented by `ApiResponse<T>` and generally includes:

- `status`
- `success`
- `message`
- `data`
- `timestamp`
- optional `meta`

## 7.2 ZenCore API map

All HTTP requests below go to ZenCore.

### Auth and session

| Method | Endpoint | Source | Request shape | Response shape | Auth | Error / retry |
|---|---|---|---|---|---|---|
| `GET` | `/auth/profile` | `src/app/(modules)/auth/actions/service.ts` | none | current user profile | Bearer | `401` clears token; no retry |
| `POST` | `/auth/otp/request` | same | `{ identifier, type: 'email' }` | OTP session payload | Public | inline/toast handling by auth flow; no retry |
| `POST` | `/auth/otp/verify` | same | `{ session_id, code }` | auth token + session result | Public | on success stores `auth_token`; no retry |
| `POST` | `/auth/logout` | same | none | logout result | Bearer | no automatic retry |
| `GET` | `/auth/permissions/{orgId}` | org actions/services | none | org permission payload | Bearer | no automatic retry |

### Organizations and org permissions

| Method | Endpoint | Source | Request shape | Response shape | Auth | Error / retry |
|---|---|---|---|---|---|---|
| `POST` | `/organizations` | organization services | org create payload | org object | Bearer | form validation + toasts; no retry |
| `GET` | `/organizations/{orgId}` | same | none | org detail | Bearer | used by org settings/detail |
| `GET` | `/organizations/by-slug/{orgSlug}` | same | none | org detail | Bearer or public depending page usage | no retry |
| `PUT` | `/organizations/{orgId}` | same | partial org update | updated org | Bearer | form errors/toasts |
| `DELETE` | `/organizations/{orgId}` | same | none | delete result | Bearer | destructive action; no retry |
| `GET` | `/organizations/check-slug/{orgSlug}` | same | none | availability result | Bearer | field-level validation use |
| `POST` | `/organization-user-mapping/accept-invitation/{orgId}` | same | invite acceptance payload | mapping result | Bearer | no retry |
| `GET` | `/organizations/{orgId}/dashboard` | dashboard services | none | dashboard summary | Bearer | dashboard loading state |

### Users and roles

| Method | Endpoint | Source | Request shape | Response shape | Auth | Error / retry |
|---|---|---|---|---|---|---|
| `POST` | `/users` | user services | user create payload | user object | Bearer | form validation + toasts |
| `GET` | `/users/{id}` | user services | none | user detail | Bearer | detail loading / no retry |
| `PUT` | `/users/{userId}` | user services | user update payload | updated user | Bearer | form validation |
| `DELETE` | `/users/{userId}` | user services | none | delete result | Bearer | destructive action |
| `DELETE` | `/users/multiple` | user services | `{ ids }` | bulk delete result | Bearer | destructive bulk action |
| `POST` | `/roles` | role services | role payload | role object | Bearer | form validation |
| `GET` | `/roles/{roleId}` | role services | none | role detail | Bearer | no retry |
| `PUT` | `/roles/{roleId}` | role services | update payload | updated role | Bearer | form validation |
| `DELETE` | `/roles/{roleId}` | role services | none | delete result | Bearer | destructive action |
| `GET` | `/roles?...` | list + async selects | query params | paginated roles | Bearer | list loading / no retry |

### Amenities

| Method | Endpoint | Source | Request shape | Response shape | Auth | Error / retry |
|---|---|---|---|---|---|---|
| `POST` | `/amenities` | amenities services | amenity payload | amenity object | Bearer | form validation |
| `GET` | `/amenities/{id}` | same | none | amenity detail | Bearer | no retry |
| `PUT` | `/amenities/{id}` | same | update payload | updated amenity | Bearer | form validation |
| `DELETE` | `/amenities/{id}` | same | none | delete result | Bearer | destructive action |
| `DELETE` | `/amenities/multiple` | same | `{ ids }` | bulk delete | Bearer | destructive bulk action |
| `GET` | `/amenities?...` | list / async select | query params | paginated amenities | Bearer | list loading / no retry |

### Groups (space groups)

| Method | Endpoint | Source | Request shape | Response shape | Auth | Error / retry |
|---|---|---|---|---|---|---|
| `POST` | `/space-groups` | groups services | group create payload | group object | Bearer | form validation |
| `POST` | `/space-groups/copy` | same | copy payload | copied group | Bearer | no retry |
| `PUT` | `/space-groups/{groupId}` | same | update payload | updated group | Bearer | form validation |
| `GET` | `/space-groups/{groupId}` | same | none | group detail | Bearer | detail loading |
| `GET` | `/space-groups/by-slug/{slug}?organization_id=` | same | none | group detail | public or bearer depending caller | booking/public flow dependency |
| `GET` | `/space-groups/check-slug/{slug}?organization_id=` | same | none | slug availability | Bearer | field validation |
| `DELETE` | `/space-groups/{groupId}` | same | none | delete result | Bearer | destructive action |
| `DELETE` | `/space-groups/multiple` | same | `{ ids }` | bulk delete | Bearer | destructive bulk action |
| `GET` | `/space-groups/{groupId}/availability?...` | same | `start_date`, `end_date`, `date`, flags | availability data | public or bearer depending page | no automatic retry |
| `GET` | `/space-groups/{groupId}/available-spaces?...` | same | date/time params | available spaces list | Public booking or bearer | no automatic retry |

### Meeting spaces

| Method | Endpoint | Source | Request shape | Response shape | Auth | Error / retry |
|---|---|---|---|---|---|---|
| `POST` | `/meeting-spaces` | meeting-space services | create payload | meeting space object | Bearer | large form validation |
| `POST` | `/meeting-spaces/copy` | same | copy payload | copied space | Bearer | no retry |
| `GET` | `/meeting-spaces/{id}` | same | none | detail object | Bearer or public if called through public page variant by slug path elsewhere | no retry |
| `GET` | `/meeting-spaces/by-slug/{slug}?organization_id=` | same | none | detail object | public booking + admin | no retry |
| `GET` | `/meeting-spaces/by-code/{code}` | same | none | detail object | Bearer | no retry |
| `GET` | `/meeting-spaces/check-slug/{slug}?organization_id=&exclude_id?` | same | none | availability result | Bearer | field validation |
| `PUT` | `/meeting-spaces/{id}` | same | update payload | updated space | Bearer | large form validation |
| `PUT` | `/meeting-spaces/{id}/active-status` | same | `{ is_active }` | updated flag | Bearer | no retry |
| `DELETE` | `/meeting-spaces/{id}` | same | none | delete result | Bearer | destructive action |
| `DELETE` | `/meeting-spaces/multiple` | same | `{ ids }` | bulk delete | Bearer | destructive bulk action |
| `GET` | `/meeting-spaces/{spaceId}/availability?...` | same | date/query flags | availability object/time slots | admin and booking-adjacent flows | no auto retry |
| `GET` | `/meeting-space-status/timeline/{spaceId}?...` | same | paging/time range | status timeline | Bearer | no auto retry |

### Dynamic pricing

| Method | Endpoint | Source | Request shape | Response shape | Auth | Error / retry |
|---|---|---|---|---|---|---|
| `POST` | `/meeting-spaces/{meetingSpaceId}/pricing-rules` | dynamic pricing services | pricing rule payload | created rule | Bearer | form validation/toasts |
| `GET` | `/meeting-spaces/{meetingSpaceId}/pricing-rules/{ruleId}` | same | none | rule detail | Bearer | edit flow loading |
| `PUT` | `/meeting-spaces/{meetingSpaceId}/pricing-rules/{ruleId}` | same | update payload | updated rule | Bearer | form validation |
| `DELETE` | `/meeting-spaces/{meetingSpaceId}/pricing-rules/{ruleId}` | same | none | delete result | Bearer | destructive action |
| `POST` | `/pricing/calculate` | same | pricing calculation payload | price breakdown | public booking + admin | no retry, UI recalculates on change |

### Business hours and unavailability

| Method | Endpoint | Source | Request shape | Response shape | Auth | Error / retry |
|---|---|---|---|---|---|---|
| `POST` | `/business-hours` | business-hours services | schedule payload | created config | Bearer | form validation |
| `GET` | `/business-hours?space_id=` | same | query | hours list | Bearer | list loading |
| `PUT` | `/business-hours/{id}` | same | update payload | updated config | Bearer | no retry |
| `DELETE` | `/business-hours/{id}` | same | none | delete result | Bearer | destructive action |
| `DELETE` | `/business-hours/multiple` | same | `{ ids }` | bulk delete | Bearer | destructive bulk action |
| `POST` | `/space-unavailability` | unavailability services | unavailability payload | created record | Bearer | form validation |
| `GET` | `/space-unavailability?...` | same | query | list | Bearer | list loading / empty states |
| `PUT` | `/space-unavailability/{id}` | same | update payload | updated record | Bearer | no retry |
| `DELETE` | `/space-unavailability/{id}` | same | none | delete result | Bearer | destructive action |

### Bookings and payment

| Method | Endpoint | Source | Request shape | Response shape | Auth | Error / retry |
|---|---|---|---|---|---|---|
| `POST` | `/bookings` | bookings services | booking payload | booking object | public booking or bearer | booking form handles inline + toast validation |
| `GET` | `/bookings/{id}` | bookings services | none | booking detail | public detail or bearer | detail loading |
| `POST` | `/bookings/{id}/cancel` | bookings services | `{ reason, cancellation_fee, refund_percentage? }` | cancelled booking result | Bearer | modal-driven validation, no retry |
| `POST` | `/bookings/{bookingId}/retry-payment` | booking payment services | `{ payment_intent_id }` | retry payment client secret | public payment flow | payment-specific error messaging |

### Vouchers

| Method | Endpoint | Source | Request shape | Response shape | Auth | Error / retry |
|---|---|---|---|---|---|---|
| `GET` | `/v1/vouchers` | voucher services | query params | voucher list | Bearer | list loading / empty state |
| `POST` | `/v1/vouchers` | same | voucher payload | created voucher | Bearer | form validation |
| `GET` | `/v1/vouchers/{id}` | same | none | detail | Bearer | detail loading |
| `PUT` | `/v1/vouchers/{id}` | same | update payload | updated voucher | Bearer | form validation |
| `DELETE` | `/v1/vouchers/{id}` | same | none | delete result | Bearer | destructive action |
| `POST` | `/v1/vouchers/validate` | same | validate payload | validation result + discount data | booking/public flow | no retry, inline voucher feedback |

### Integrations, notifications, webhooks

| Method | Endpoint | Source | Request shape | Response shape | Auth | Error / retry |
|---|---|---|---|---|---|---|
| `GET` | `/third-party?...` | integrations services/list | query | connection list | Bearer | list loading |
| `POST` | `/third-party/{id}/approve` | integrations services | `{ reason }` | updated connection | Bearer | modal action, no retry |
| `POST` | `/third-party/{id}/decline` | same | `{ reason }` | updated connection | Bearer | modal action |
| `POST` | `/third-party/{id}/suspend` | same | `{ reason }` | updated connection | Bearer | modal action |
| CRUD | `/notifications`, `/notifications/{id}` | notification services | notification form payload | notification objects | Bearer | form validation, list/detail loading |
| CRUD | `/webhooks`, `/webhooks/{id}` | webhook services | webhook payload | webhook objects | Bearer | form validation |
| `GET` | `/event-types?...` | notification/webhook forms | query | event types list | Bearer | async-select loading |

### API keys, uploads, version, internal testing

| Method | Endpoint | Source | Request shape | Response shape | Auth | Error / retry |
|---|---|---|---|---|---|---|
| CRUD | `/organizations/{organizationId}/api-keys...` | API key services | API key payloads | API key detail/list items | Bearer | form validation + destructive actions |
| `POST`/`PUT` | `/upload/entity` | common services | `FormData` upload | document/upload result | Bearer | upload error surfaced to caller |
| `DELETE` | `/upload/entity/{docId}` | common services | none | delete result | Bearer | no retry |
| `GET` | `/version` | common services | none | version metadata | Bearer or public if endpoint allows | no retry |
| `POST` | `/api/testing/load-test` | testing services | load-test config payload | test job result | Bearer | form validation |
| `GET` | `/api/testing/load-test/{id}` | same | none | detail | Bearer | detail loading |
| `GET` | `/api/testing/load-test/{id}/report?format=json` | same | none | report JSON | Bearer | no retry |
| `GET` | `/api/testing/load-test/{id}/report?format=csv|pdf` | same | none | blob download | Bearer | manual download handling |

## 7.3 ZenEdge-related contract map (proxied via ZenCore)

These FE flows are conceptually ZenEdge-facing, but the browser still uses ZenCore endpoints.

### Physical spaces / MRD sessions

| Method | Endpoint | Source | Request shape | Response shape | Auth | Error / retry |
|---|---|---|---|---|---|---|
| `POST` | `/mrd-sessions` | `meeting-spaces/actions/physical-space-services.ts` | `{ api_key, organization_id }` | `{ mrd_session_id, expires_at }` | Bearer | inline field errors used in UI |
| `GET` | `/physical-spaces?mrd_session_id=` | same | query | physical spaces list | Bearer | list loading / no retry |
| `POST` | `/physical-spaces/{physicalSpaceId}/mappings` | same | mapping payload | created mapping | Bearer | inline conflict alerts and validation |
| `DELETE` | `/physical-spaces/{physicalSpaceId}/mappings/{mappingId}?organization_id=` | same | optional override body | delete/unmap result | Bearer | guarded unlink modal + conflict handling |

### Devices / access / logs / failure policy

| Method | Endpoint | Source | Request shape | Response shape | Auth | Error / retry |
|---|---|---|---|---|---|---|
| `GET` | `/v1/admin/devices?...` | devices services | query | devices list | Bearer | list loading / empty states |
| `PUT` | `/v1/admin/devices/{deviceId}/approve` | devices services | approval payload | updated device | Bearer | action toast/modal feedback |
| `POST` | `/v1/devices/action` | devices services | `{ meeting_space_id, device_type, action, reason? }` | action result | Bearer | no automatic retry |
| `GET` | `/v1/admin/devices/logs?...` | device log services/pages | query | logs list | Bearer | list loading |
| `GET` | `/v1/user-device-access/me?organization_id=` | user-device-access pages | query | credential/access records | Bearer | list/card empty states |
| `GET` | `/v1/admin/device-failure-policies/choices` | failure-policy services | none | choices | Bearer | settings loading |
| `GET/PUT` | `/organizations/{organizationId}/device-failure-policy` | failure-policy services | settings payload | policy | Bearer | settings save flow |
| `GET/PUT` | `/meeting-spaces/{meetingSpaceId}/device-failure-policy` | failure-policy services | settings payload | policy | Bearer | settings save flow |

## 7.4 Shared list and select patterns

| Pattern | Path | Behavior |
|---|---|---|
| List views | `src/common/global-components/list-view/list-view.tsx` | GET with `page`, `limit`, `sortBy`, `sortOrder`, `search`, plus filters |
| Async paginate select | `src/common/global-components/select/select-async-paginate.tsx` | GET with `search`, `page`, `limit=10`; maps `result.data` to options |

## 7.5 WebSocket / realtime contract map

ZenCore websocket namespace:

- `/realtime`

Observed event names:

- `zscloud:meeting.status_update`
- `zscloud:connection.update`
- `zscloud:booking.update`
- `zscloud:booking.bulk_result`
- `zscloud:device.pending_approval`
- `zscloud:device.status_changed`
- `zscloud:device.alert`

Auth:

- Socket auth sends `localStorage.auth_token`

Retry:

- relies on Socket.IO behavior; no bespoke reconnect orchestration found

## 8. User Journeys / Logical Flows (End-to-End)

### Auth / login / session bootstrap

1. User opens `/auth`
2. User requests OTP with email
3. User verifies OTP
4. FE stores `auth_token` in `localStorage`
5. `AuthProvider` revalidates session using `/auth/profile`
6. Authenticated route trees become accessible
7. If a request later receives `401`, the API client clears token and redirects back to auth

### Organization selection and dashboard bootstrap

1. User reaches `/organization`
2. User selects or creates an organization
3. Dashboard navigation carries `?orgId=...`
4. `DashboardDataProvider` reads `orgId`
5. FE loads organization + permissions into Redux
6. Dashboard pages render only after org bootstrap succeeds

Critical implication:

- dashboard pages assume org context is present
- links that drop `orgId` can break data loading

### Booking create flow

1. End user opens booking page under `/booking/...`
2. FE loads organization/group/space context from slugs
3. FE loads availability
4. User selects date/time
5. FE calculates pricing
6. User enters organizer details
7. Voucher may be validated
8. FE creates booking
9. FE enters payment flow if needed
10. User reaches booking details/confirmation

### Booking update / cancel flow

Admin path:

1. Operator opens `/bookings`
2. Searches and filters to target booking
3. Opens detail page
4. Uses booking cancellation modal if needed
5. Sends cancellation reason / fee / refund payload
6. Reviews resulting status

### Physical mapping / operations flow

Space-level admin path:

1. Operator opens `/meeting-spaces/:msId`
2. Opens physical-space tab
3. Creates MRD session through ZenCore
4. Loads physical spaces from ZenCore-proxied ZenEdge
5. Selects physical space and mapping window
6. Creates mapping
7. Handles conflicts or unlink requirements if they exist

### Booking access credential display / email-trigger UI

What exists in the FE:

- user/device access views
- device lists and device logs
- booking/device-related operational UI

What was **not clearly found** as a dedicated frontend flow:

- a separate end-user credential email trigger workflow specific to bookings

Guidance:

- if asked to implement credential-delivery UX, inspect `user-device-access`, device-related views, and booking detail pages first

### Failure handling and user messaging paths

Common user-facing patterns:

- inline field validation for forms
- destructive action confirmation modals
- toast notifications for generic success/failure
- alert banners for some complex API conflicts
- empty states in list views and detail sections
- explicit “Booking Unavailable” state in public booking pages

## 9. Permissions and Guardrails

### Route guards

Global login gate:

- `src/providers/authenticated-modules-provider.tsx`

Permission gate component:

- `src/common/global-components/permission/permission-guard.tsx`

Permission resolution hook:

- `src/hooks/use-organization-permissions.ts`

Rule:

- `isSuperAdmin` bypasses normal permission checks

### Permission constants

Path:

- `src/app/(modules)/(authenticated-modules)/(dashboard)/roles/helpers/config.ts`

Examples:

- `read:spaceGroups`
- `create:meetingSpaces`
- `read:bookings`
- `read:vouchers`
- `read:thirdPartyConnections`
- `read:webhooks`
- `read:notifications`

### UI-level access control

The app uses a combination of:

- route-page `PermissionGuard`
- inline `hasPermission(...)` checks
- navigation hiding for privileged surfaces

Examples:

- create buttons hidden if permission missing
- table actions omitted if permission missing
- tabs and sections hidden in meeting-space detail views
- super-admin-only org shell navigation for system roles and amenities

### Feature flags

No general feature-flag platform such as LaunchDarkly or Unleash was found.

The practical substitutes are:

- permission-based visibility
- organization config toggles
- Stripe enabled/disabled state
- active/inactive resource flags

### Security-sensitive UX patterns

- auth token stored in `localStorage`
- 401 handling centralized in `src/common/helpers/api.ts`
- API key secret display flows under settings
- booking customer/contact information and user records contain PII
- public booking pages should avoid assuming admin-only context

## 10. Validation and Error UX Catalog

### Major form validation patterns

| Area | Validation approach | Notes |
|---|---|---|
| Auth OTP | RHF + Zod + inline field validation | email and OTP flow |
| Organization forms | RHF + Zod | multi-step and standard create/edit patterns |
| Group forms | RHF + Zod | address/timezone/media forms |
| Meeting space form | RHF + Zod + large custom helpers | high-complexity form with uploads, pricing, rules |
| Booking form | RHF + Zod + custom phone validation + duration constraints | public booking critical path |
| Voucher form | RHF + Zod | eligibility, discount, limits |
| Dynamic pricing form | RHF + Zod + numeric input sanitization | fixed/multiplier/adjustment rules |
| API key form | RHF + Zod | security-sensitive create/update |

### API error normalization and user feedback

Common patterns:

- services return backend response objects instead of throwing on all failures
- UI typically checks `success` and `message`
- `toast.error(...)` is common in service or action layers
- more advanced flows use inline field messages or alert blocks

Examples:

- booking/public forms use inline field messages
- physical mapping uses inline API conflict messaging for overlapping mappings
- destructive admin operations use confirmation modals plus toast feedback

### Offline / timeout / retry UX

What exists:

- local loading indicators
- manual retry by user action
- websocket reconnect handled by Socket.IO defaults

What does **not** appear to exist as a broad platform feature:

- centralized offline banner
- automatic retry policy for general HTTP requests
- React Query style request retry/invalidation framework

### Fallback states and recovery actions

Observed fallback patterns:

- `Loading` components
- empty state cards/components
- “Organization Not Found” shell state in `DashboardDataProvider`
- unavailable-state card for booking pages
- list-view empty states
- retry buttons in some provider or detail failure states

## 11. Environment, Config, and Deployment

### Environment variables inventory

| Variable | Required | Purpose | Notes |
|---|---|---|---|
| `NEXT_PUBLIC_BACKEND_URL` | Yes | ZenCore HTTP base URL | central FE API dependency |
| `NEXT_PUBLIC_WEBSOCKET_SERVICE_URL` | Optional | explicit ZenCore websocket URL | otherwise derived from backend URL |
| `NEXT_PUBLIC_BOOKING_APP_URL` | Optional | embed/public booking URL generation | used in group booking embed |
| `NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY_TEST` | Optional but practically required for payment flows | Stripe fallback key | used when org config key unavailable |
| `NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY_LIVE` | Optional but practically required for payment flows | Stripe fallback key | production/live path |
| `NEXT_PUBLIC_GOOGLE_PLACE_API` | Optional | Google Places integration | place autocomplete |
| `NEXT_PUBLIC_GOOGLE_MAPS_API_KEY` | Optional fallback | Google Maps fallback key | used if place API var absent |

### Per-environment files

Files present:

- `.env.local`
- `.env.development`
- `.env.production`

Important note:

- `.gitignore` ignores `.env.development` and `.env.production`
- if those are tracked elsewhere historically, treat as a secrets/process issue to review manually

### Build / run / lint commands

| Command | Purpose |
|---|---|
| `npm run dev` | local development using `.env.development` |
| `npm run prod` | runs `next dev` with `.env.production` |
| `npm run dev:local` | local development using `.env.local` |
| `npm run build` | production build |
| `npm run start` | production server |
| `npm run lint` | ESLint |

### CI/CD and deployment assumptions

What is visible:

- standard Next build/start pattern
- `.vercel` ignored in `.gitignore`

What was **not found**:

- Dockerfile
- GitHub Actions workflows
- explicit CI config
- vercel.json

Inference:

- deployment is likely a conventional Next deployment with env-injected runtime config
- Vercel is plausible but not guaranteed from code alone

## 12. Testing Coverage Map

### Tooling status

| Area | Status |
|---|---|
| Unit tests | none found |
| Integration tests | none found |
| E2E tests | none found |
| Storybook | not found |
| MSW/mocks | not found |
| Jest/Vitest | not found |
| Playwright/Cypress config | not found |

### Important clarification

The `testing/` dashboard module is an application feature for test/load-related workflows, not a frontend automated test harness.

### Coverage map by domain

| Module | Automated coverage status | Notes |
|---|---|---|
| Auth | Weak / absent | central token/session behavior needs caution |
| Dashboard bootstrap | Weak / absent | `orgId` bootstrap critical but untested |
| Groups | Weak / absent | list/detail/forms rely on manual verification |
| Meeting spaces | Weak / absent | largest and riskiest form area |
| Booking app | Weak / absent | public critical path, pricing + payment sensitive |
| Dynamic pricing | Weak / absent | rules and numeric input behavior mostly manual |
| Vouchers | Weak / absent | business-rule heavy |
| Devices / logs / access | Weak / absent | operational side effects, realtime dependencies |
| Realtime hooks | Weak / absent | event cleanup and subscriptions fragile |

### Test data / mocks / fixtures

No standardized mock/fixture strategy was found.

Agents should assume:

- verification is currently mostly manual
- service calls hit real backend environments in normal dev usage

## 13. Performance, Accessibility, i18n

### Performance-sensitive areas

Most sensitive surfaces:

- `meeting-space-create-form.tsx`
- `meeting-spaces/[msId]/page.tsx`
- booking app pages under `src/app/(modules)/booking-app/`
- realtime-heavy device and status views
- dynamic pricing and booking price recalculation flows

### Code splitting / lazy loading

What exists:

- App Router page-level splitting
- Suspense in major shell/provider boundaries

What was not found as a pervasive pattern:

- widespread explicit lazy imports for feature components
- React Query-style prefetch/dehydrate architecture

### Accessibility

Strengths:

- Radix/shadcn base components
- labels/messages in many RHF forms
- button and dialog primitives generally accessible by default

Known gaps or caution areas:

- heavy use of custom calendar/time/drag controls
- carousel-based media areas
- no automated accessibility test coverage found
- some UI branches rely heavily on visual cues and toasts

### Internationalization

No dedicated i18n framework or localization file structure was found.

Implication:

- app copy is effectively hard-coded English
- any i18n work would be a new architectural addition

## 14. Critical Invariants / Non-Negotiable Rules

These behaviors must be preserved unless the task explicitly changes them.

1. **Do not break `/booking` rewrites.**
   The public booking app is exposed through `next.config.ts` rewrites and must remain reachable under `/booking/...`.

2. **Do not break dashboard `?orgId=` context.**
   The dashboard shell depends on URL query-based org bootstrap via `DashboardDataProvider`.

3. **Do not bypass centralized auth token handling.**
   `src/common/helpers/api.ts` owns token header injection and 401 redirect behavior.

4. **Do not assume direct ZenEdge browser access.**
   ZenEdge-related operations are currently mediated by ZenCore.

5. **Do not rename public routes or rewrite targets casually.**
   Friendly URLs such as `/booking/...` and `/organization/roles/...` are part of the app contract.

6. **Preserve date-only semantics carefully.**
   Availability, booking, pricing, and mapping flows are highly sensitive to timezone and date normalization behavior.

7. **Do not assume feature isolation between admin and booking app.**
   Public booking flows reuse admin-side models and services.

8. **Preserve permission-gated UI behavior.**
   Hidden actions and `PermissionGuard` usage are part of the security UX contract even if backend also enforces authorization.

9. **Do not remove Stripe provider assumptions without tracing booking/payment pages.**
   Payment screens and some dashboard settings depend on provider-level setup.

10. **Treat README as secondary truth when it conflicts with code.**
   Source code and current docs in `docs/` are more reliable.

### Anti-regression checklist

- dashboard pages still work with `orgId`
- booking pages still load via `/booking/...`
- 401 still clears token and redirects correctly
- permission-hidden actions remain hidden
- form validation messages still surface properly
- date and timezone logic still produces correct selected/disabled dates
- booking pricing still recalculates correctly
- physical mapping still handles ZenEdge conflicts through ZenCore responses

## 15. AI Execution Guidance

### Safe change workflow for agents

Use this order for most tasks:

1. Identify the route/page
2. Find the owning feature module
3. Read the relevant page component
4. Read the feature components used by that page
5. Read the corresponding `actions/services.ts`
6. Read Redux slice(s) if the UI depends on global state
7. Read schema/validation if forms are involved
8. Update tests if tests exist, otherwise document manual smoke verification

Recommended change path:

- route/page -> feature component -> state/service layer -> validation -> verification

### Mandatory checks before finalizing

Always attempt or recommend these checks:

1. `typecheck`
   Suggested: `npx tsc --noEmit`

2. `lint`
   Run: `npm run lint`

3. `tests / e2e smoke`
   Current reality:
   - no standard automated frontend suite found
   - perform targeted manual smoke verification for affected flows

4. `build`
   Run: `npm run build`

If any step cannot be run, say so explicitly in the final handoff.

### Hard rules for agents

- do not rename public routes/contracts without migration notes
- do not silently break query-string org context
- do not change auth token storage/401 behavior casually
- do not change shared date handling without checking booking + admin flows
- do not assume unused dependencies describe actual runtime architecture

## 16. Canonical References

### Core architecture

- `package.json`
- `next.config.ts`
- `src/app/layout.tsx`
- `src/middleware.ts`
- `src/providers/auth-provider.tsx`
- `src/app/(modules)/(authenticated-modules)/layout.tsx`
- `src/app/(modules)/(authenticated-modules)/(dashboard)/layout.tsx`
- `src/providers/dashboard-data-provider.tsx`
- `src/providers/redux-provider.tsx`
- `src/providers/stripe-provider.tsx`

### State

- `src/redux/store.ts`
- `src/hooks/redux-hook.ts`
- `src/common/reducers/dashboard-reducers.ts`
- `src/common/reducers/common-reducers.ts`

### Auth and API

- `src/common/helpers/api.ts`
- `src/app/(modules)/auth/actions/service.ts`
- feature `actions/services.ts`

### Permissions

- `src/common/global-components/permission/permission-guard.tsx`
- `src/hooks/use-organization-permissions.ts`
- `src/app/(modules)/(authenticated-modules)/(dashboard)/roles/helpers/config.ts`

### Navigation and shared UI

- `src/common/global-components/sidebar-navbar/app-sidebar.tsx`
- `src/common/global-components/url-pathname/url-pathname.tsx`
- `src/components/ui/`
- `src/common/global-components/`

### Major domains

- `src/app/(modules)/(authenticated-modules)/(dashboard)/groups/`
- `src/app/(modules)/(authenticated-modules)/(dashboard)/meeting-spaces/`
- `src/app/(modules)/(authenticated-modules)/(dashboard)/bookings/`
- `src/app/(modules)/(authenticated-modules)/(dashboard)/dynamic-pricing/`
- `src/app/(modules)/(authenticated-modules)/(dashboard)/vouchers/`
- `src/app/(modules)/(authenticated-modules)/(dashboard)/devices/`
- `src/app/(modules)/(authenticated-modules)/(dashboard)/device-logs/`
- `src/app/(modules)/(authenticated-modules)/(dashboard)/user-device-access/`
- `src/app/(modules)/(authenticated-modules)/(dashboard)/integrations/`
- `src/app/(modules)/(authenticated-modules)/(dashboard)/settings/`

### Public booking app

- `src/app/(modules)/booking-app/`
- `src/app/(modules)/booking-app/(views)/[...slugs]/page.tsx`
- `src/app/(modules)/booking-app/(views)/[orgSlug]/[groupSlug]/[spaceSlug]/page.tsx`
- `src/app/(modules)/booking-app/components/booking-form.tsx`
- `src/app/(modules)/booking-app/(views)/payment/page.tsx`

### Realtime

- `src/hooks/use-realtime.ts`

### Existing docs

- `docs/APP_DOCUMENTATION.md`
- `docs/TECHNICAL_ARCHITECTURE.md`
- `docs/PHYSICAL_SPACE_MAPPING_GUIDE.md`
- `docs/booking-iframe-embed.md`
- `docs/MEETING_SPACE_AVAILABILITY_IMPLEMENTATION_PLAN.md`
- `docs/GROUP_AVAILABILITY_IMPLEMENTATION_PLAN.md`
- `docs/VOUCHER_IMPLEMENTATION_PLAN.md`
- `docs/IMAGE_SPECIFICATIONS.md`

