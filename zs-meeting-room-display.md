# zs-meeting-room-display Frontend Skill Document

## 1) Title and purpose
This file is the consolidated frontend AI context document for `zs-meeting-room-display`.

Its purpose is to give coding agents a production-grade understanding of the repository so they can make safe, fast, contract-aware changes with minimal re-discovery. It is intended to function as long-term skill/context material for Claude, Cursor, GPT, or similar agents working on this frontend.

This document is based on the code currently in this repository. It focuses on actual implementation details, not generic frontend theory.

Terminology used throughout:

- `ZenCore`: the backend/API platform reached through `VITE_BACKEND_URL`
- `ZenEdge`: the MRD/device realtime platform reached through `VITE_WEBSOCKET_SERVICE_URL`

---

## 2) Agent ingestion order
Read the repository in this order before making non-trivial changes.

| Order | Area | Why it matters | Concrete paths |
|---|---|---|---|
| 1 | App bootstrap | Establishes runtime shell, providers, router mount, global CSS, and toaster | `index.html`, `src/main.tsx`, `src/index.css` |
| 2 | Router definitions | Source of truth for all routes, guards, and page ownership | `src/App.tsx` |
| 3 | Layouts and route guards | Explains public vs protected app shells and organization scoping | `src/common/global-components/protected-layout.tsx`, `src/common/global-components/organization-layout.tsx`, `src/providers/auth-provider.tsx`, `src/providers/organization-provider.tsx` |
| 4 | State management | Defines global Redux slices, refresh invalidation, and typed hooks | `src/redux/store.ts`, `src/redux/common-reducers.ts`, `src/redux/hooks/redux-hook.ts` |
| 5 | API client layer | Explains auth headers, error behavior, base URL joining, and request envelope assumptions | `src/common/helpers/api.ts`, `src/common/interface/interface.ts`, `src/common/actions/services.ts` |
| 6 | Feature modules | Domain behavior lives here; inspect the specific module you are changing | `src/modules/auth/`, `src/modules/public/magic-link/`, `src/modules/authenticated/organization/`, `src/modules/authenticated/physical-spaces/`, `src/modules/authenticated/meeting-room-displays/`, `src/modules/authenticated/iot-gateway/`, `src/modules/authenticated/spaces/`, `src/modules/authenticated/locks/`, `src/modules/authenticated/wifi/`, `src/modules/authenticated/org-api-keys/` |
| 7 | Shared UI and design system | Shared shell, table scaffolding, modals, form wrappers, and primitives are here | `src/common/global-components/`, `src/components/ui/`, `components.json`, `src/lib/utils.ts` |
| 8 | Realtime and cross-module refresh | Critical for display, webhook, gateway, and device status updates | `src/hooks/useRealtimeWebhooks.ts`, `src/hooks/useRealtimeDisplayStatus.ts`, `src/hooks/useRealtimeIotVerification.ts`, `src/hooks/useDeviceStatusUpdates.ts` |
| 9 | Env and config | Build behavior, aliases, TS strictness, envs, and lint expectations | `vite.config.ts`, `tsconfig.json`, `tsconfig.app.json`, `eslint.config.js`, `.env.development`, `.env.production` |
| 10 | Tests and e2e reality check | There is no automated test suite in this repo; use docs/manual guidance | `docs/USER_FLOW_AND_TESTING.md`, `docs/PHYSICAL_SPACE_SETUP_GUIDE.md` |

Suggested minimum read set for any feature change:

1. `src/App.tsx`
2. relevant route file in `src/modules/.../views/`
3. relevant `actions/services.ts`
4. relevant reducer and actions files
5. related shared UI wrapper in `src/common/global-components/`

---

## 3) Frontend architecture overview

### Framework and runtime

| Concern | Implementation |
|---|---|
| Build/runtime | Vite 7 |
| UI framework | React 19 |
| Language | TypeScript |
| Router | `react-router-dom` with `BrowserRouter` |
| Global state | Redux Toolkit |
| Forms | `react-hook-form` |
| Validation | Zod |
| Styling | Tailwind CSS v4 with CSS variables |
| Component primitives | Radix + shadcn-style local components |
| Icons | `lucide-react` |
| Realtime | `socket.io-client` |
| Notifications | Sonner |

### Rendering strategy
The app is a client-rendered SPA.

| Concern | Status |
|---|---|
| CSR | Yes |
| SSR | No |
| SSG | No |
| ISR | No |
| Next.js app/router features | No |

The app mounts in `src/main.tsx` and uses `BrowserRouter`, so deployment must support SPA rewrites.

### Module boundaries and folder conventions
Top-level module split:

- `src/modules/auth/`: OTP login flow
- `src/modules/public/`: unauthenticated public experiences, currently the magic-link screen
- `src/modules/authenticated/`: internal product/admin/operator features

Common feature structure inside `src/modules/authenticated/<feature>/`:

- `views/`: top-level screens
- `components/`: feature-local UI
- `actions/services.ts`: ZenCore API calls
- `actions/actions.ts`: thunk-like dispatch orchestration
- `reducers/reducers.ts`: Redux slice where used
- `schema/schema.ts`: Zod validation
- `model-interfaces/` or `interface-model/`: type/model wrappers
- `helpers/` or `hooks/`: feature-specific utilities

This is a convention, not a perfect rule. Some features are lighter-weight than others.

### How the frontend talks to ZenCore

| Concern | Implementation |
|---|---|
| Base URL | `import.meta.env.VITE_BACKEND_URL` |
| Main client | `zenspaceApi.request()` in `src/common/helpers/api.ts` |
| Auth | `Authorization: Bearer <auth_token>` from `localStorage` |
| Error model | Returns parsed JSON for non-OK responses; does not consistently throw on HTTP 4xx/5xx |
| Special handling | On 401, removes `auth_token` and redirects to `/auth` |
| Shared response envelope | `ApiResponse<T>` from `src/common/interface/interface.ts` |

The public magic-link page is a notable exception. It uses raw `fetch` with `joinBackendUrl()` rather than `zenspaceApi`.

### How the frontend talks to ZenEdge

| Concern | Implementation |
|---|---|
| Base URL | `import.meta.env.VITE_WEBSOCKET_SERVICE_URL` |
| Transport | Socket.IO |
| Namespace/path | `/realtime` |
| Auth | handshake `auth: { token }` using `localStorage.auth_token` |
| Primary consumers | display status, webhook updates, IoT gateway verification, device status updates |

### Important architectural shape
The app is best understood as:

1. a routed admin/operator SPA shell
2. with auth and organization context layered on top
3. using Redux for authenticated domain state
4. using ZenCore REST for data and commands
5. using ZenEdge realtime events for freshness and invalidation
6. plus one public token-based magic-link screen with separate fetch logic

---

## 4) Route/page/screen inventory (complete)
Source of truth: `src/App.tsx`

Route guards:

- `PublicRoute`: unauthenticated route; redirects authenticated users to `/organization`
- `ProtectedRoute`: auth required; renders `ProtectedLayout`
- `OrganizationProtectedRoute`: auth required; renders `OrganizationLayout`

There is no true route-level role gating in `src/App.tsx`. Access is primarily guest vs authenticated, plus organization context availability.

| Path | Screen / primary file | Owning module | Access level | Primary data dependencies | Loading / error / empty states |
|---|---|---|---|---|---|
| `/auth` | `src/modules/auth/views/login.tsx` | `auth` | Public | `requestOTPSevice()`, `verifyOTPSevice()`, `getAppVersionService()`, `useAuth().checkAuth()` | Global `Loader` while auth bootstraps; inline OTP/email step errors; toast-based failures |
| `/organization` | `src/modules/authenticated/organization/views/list.tsx` | `authenticated/organization` | Protected | `ListViewTable` against `/auth/organizations?` | Shared list skeletons; empty organizations state via `ListViewTable`; organization selection cards |
| `/physical-spaces` | `src/modules/authenticated/physical-spaces/views/list.tsx` | `authenticated/physical-spaces` | Protected + organization-scoped | `ListViewTable` against `/physical-spaces?org_id=...&`; create/delete via physical-space services | Shared list skeletons; empty state from `ListViewTable`; toast on mutation failures |
| `/physical-spaces/:id` | `src/modules/authenticated/physical-spaces/views/detail.tsx` | `authenticated/physical-spaces` | Protected + organization-scoped | `getPhysicalSpaceByIdAction()`, `getPhysicalSpaceDevicesAction()`, mappings list, embedded webhook log list, realtime webhook refresh | `Loader` while fetching; `NotFound` when entity missing; card-level empty states for booking/status/device areas |
| `/physical-spaces/webhook-logs/:logId` | `src/modules/authenticated/physical-spaces/components/physical-space-webhook-detail.tsx` | `authenticated/physical-spaces` | Protected + organization-scoped | `getWebhookLogsByPhysicalSpaceIdAction(logId)` | `Loader`, `NotFound`, tabbed payload/header/response detail |
| `/meeting-room-displays` | `src/modules/authenticated/meeting-room-displays/views/list.tsx` | `authenticated/meeting-room-displays` | Protected + organization-scoped | `ListViewTable` against `/displays?org_id=...&`, `useRealtimeDisplayStatus()`, display CRUD and link/unlink services | Shared list skeletons and empty state; verification/link flows surface toasts and modal states |
| `/meeting-room-displays/:id` | `src/modules/authenticated/meeting-room-displays/views/detail.tsx` | `authenticated/meeting-room-displays` | Protected + organization-scoped | `geMeetingRoomDisplayByIdAction()`, image/logo upload services, screenshot request, notifications stats, embedded webhook list, realtime webhook hook | `Loader`, `NotFound`, tab-level `Empty` states for status/device/webhook sections |
| `/meeting-room-displays/webhook-logs/:logId` | `src/modules/authenticated/webhook-logs/views/detail.tsx` | `authenticated/webhook-logs` | Protected + organization-scoped | `geWebhookLogByIdAction(logId)` | `Loader`, `NotFound`, full webhook payload/booking extraction UI |
| `/IoT-Gateway` | `src/modules/authenticated/iot-gateway/views/list.tsx` | `authenticated/iot-gateway` | Protected + organization-scoped | `ListViewTable` against `/iot-gateways?org_id=...&`, realtime verification hook, gateway CRUD services | Shared list skeletons and empty state; onboarding heartbeat/waiting indicators |
| `/iot-gateways/:id` | `src/modules/authenticated/iot-gateway/views/detail.tsx` | `authenticated/iot-gateway` | Protected + organization-scoped | `getIoTGatewayByIdAction()`, realtime verification | `Loader`, `NotFound`, status banners and onboarding diagnostics |
| `/spaces` | `src/modules/authenticated/spaces/views/lists.tsx` | `authenticated/spaces` | Protected + organization-scoped | `ListViewTable` against `/meeting-spaces?organization_id=...&`, inline filter using `/space-groups?` | Shared list skeletons; empty state via `ListViewTable` |
| `/spaces/:id` | `src/modules/authenticated/spaces/views/detail.tsx` | `authenticated/spaces` | Protected + organization-scoped | `getSpaceLinkedDisplaysAction()`, `useDeviceStatusUpdates()` | `Loader`; status tab `Empty` state when live device state is absent |
| `/locks` | `src/modules/authenticated/locks/views/lists.tsx` | `authenticated/locks` | Protected + organization-scoped | `ListViewTable` against `/iot-devices?org_id=...&device_type=LOCK&`, lock form, generic IoT device CRUD | Shared list skeletons; empty state via `ListViewTable` |
| `/wifi` | `src/modules/authenticated/wifi/views/lists.tsx` | `authenticated/wifi` | Protected + organization-scoped | `ListViewTable` against `/iot-devices?org_id=...&device_type=WIFI_HOTSPOT&`, wifi form, generic IoT device CRUD | Shared list skeletons; empty state via `ListViewTable` |
| `/api-keys` | `src/modules/authenticated/org-api-keys/views/list.tsx` | `authenticated/org-api-keys` | Protected + organization-scoped | `ListViewTable` against `/org-api-keys?org_id=...&`, create/revoke/rotate API key services | Shared list skeletons; empty state via `ListViewTable`; secret modal after create/rotate |
| `/magic/:token` | `src/modules/public/magic-link/views/page.tsx` | `public/magic-link` | Public token-based access | direct `fetch` calls to `/magic-links/*` endpoints, no Redux | full-page `loading` / `ready` / `error` states; device-level empty state; inline toast bar |
| `*` | router fallback in `src/App.tsx` | app shell | n/a | none | hard redirect to `/auth` |

### Internal or hidden screens

| Screen type | Notes |
|---|---|
| Hidden from main sidebar | `/spaces` and `/spaces/:id` are real routes, but `Spaces` is commented out in `src/common/global-components/sidebar-navbar/app-sidebar.tsx` |
| Deep-link only | webhook log detail routes are reachable from embedded log lists, not top-level nav |
| Internal/admin-ish UI | organization three-dot menu on cards is only rendered when `user?.super_admin` is true in `src/modules/authenticated/organization/components/organization-card.tsx` |

### Loading/error shell behavior across routes

| Layer | Behavior |
|---|---|
| Auth bootstrap | `AuthProvider` sets `isLoading`; route guards show `Loader` until resolved |
| Protected app with missing org context | `OrganizationProvider` shows an “Organization Not Found” empty state with retry/back navigation |
| List screens | `ListViewTable` handles loading skeletons, fetch errors by falling back to empty data, and empty-state cards |
| Detail screens | Most detail pages use explicit `Loader` + `NotFound` patterns |

---

## 5) Component architecture (complete)

### Feature components vs shared components

| Layer | Location | Purpose |
|---|---|---|
| Feature-local components | `src/modules/**/components/` | domain-specific cards, forms, detail sections, tables, and modals |
| Shared app components | `src/common/global-components/` | app shell, list scaffolding, modals, typography, path display, filters, file uploads, custom dropdowns |
| Shared design-system primitives | `src/components/ui/` | button, input, dialog, sidebar, tabs, table, empty state, badge, scroll-area, form wrappers, etc. |

### Layout system and shell/navigation

| Area | File(s) | Responsibility |
|---|---|---|
| Public/auth shell | route-local in `src/modules/auth/views/login.tsx` | standalone login experience |
| Organization shell | `src/common/global-components/organization-layout.tsx` | sidebar + top header for org selection |
| Protected shell | `src/common/global-components/protected-layout.tsx` | main authenticated app shell; wraps `OrganizationProvider` |
| Main navigation | `src/common/global-components/sidebar-navbar/app-sidebar.tsx` | Physical Spaces, IoT Gateway, Meeting Room Displays, Locks, Wifi, API Keys |
| Org navigation | `src/common/global-components/sidebar-navbar/organization-sidebar.tsx` | Organizations-only sidebar |
| User menu/logout | `src/common/global-components/sidebar-navbar/nav-user.tsx` | current user display and logout action |

### Reusable primitives and patterns
High reuse components and patterns:

- `ListViewTable` in `src/common/global-components/list-view/list-view.tsx`
- `AsyncPaginateSelect` in `src/common/global-components/select/select-async-paginate.tsx`
- modal wrappers under `src/common/global-components/modal/`
- `PageHeader`
- `Empty` and `NotFound`
- sidebar primitives under `src/components/ui/sidebar.tsx`

### Design system, theme, and tokens
Design system setup:

- shadcn-style configuration in `components.json`
- CSS variables and Tailwind theme tokens in `src/index.css`
- aliases defined for `@/components`, `@/lib`, `@/hooks`

Observed token strategy:

- semantic CSS variables such as `--background`, `--foreground`, `--primary`, `--border`, `--sidebar-*`
- light and dark variable sets defined in `src/index.css`
- component classes rely heavily on semantic tokens like `bg-background`, `text-foreground`, `border-border`

Theme observations:

- dark-mode variables exist
- `next-themes` is installed, but the repo does not expose a broad theme-management architecture; theme usage is light and mostly indirect

### Form strategy and validation approach

| Concern | Implementation |
|---|---|
| Form library | `react-hook-form` |
| Validation | Zod with `zodResolver` |
| Field wrappers | local form primitives in `src/components/ui/form.tsx` |
| Async selects | `AsyncPaginateSelect` |
| Submission UX | inline button loading + Sonner toasts |
| Validation timing | varies, but often `mode: 'onChange'` for complex forms |

Primary forms:

- login and OTP
- physical space form
- physical space mapping form
- meeting room display create form
- link display to ZenCore/ZenSpace meeting space form
- meeting room display config form
- IoT gateway form
- lock form
- wifi form
- org API key create/rotate form

---

## 6) State management and data flow

### Global state stores
Redux Toolkit is the main global state solution.

Store: `src/redux/store.ts`

Registered slices:

- `common`
- `meetingRoomDisplay`
- `webhookLogs`
- `organization`
- `IoTGateway`
- `spaces`
- `locks`
- `wifi`
- `physicalSpace`

### Context state

| Context | File | Responsibility |
|---|---|---|
| Auth context | `src/providers/auth-provider.tsx` | current user, auth bootstrap, `checkAuth()` |
| Organization context | `src/providers/organization-provider.tsx` | resolves `orgId` from query string, loads org into Redux, blocks children until ready |

### Server state and caching
There is no React Query, SWR, or other dedicated server-state cache in this repo.

Current strategy:

- explicit service calls for fetch/mutate
- Redux slices for selected authenticated domain entities
- component state for page-local server results
- list fetching through reusable `ListViewTable`
- global invalidation through `common.hardRefresh`

### Mutation patterns and invalidation rules
Most mutations follow this pattern:

1. call a ZenCore service function
2. check `res?.success`
3. show `toast.success()` or `toast.error()`
4. either refetch the current entity or dispatch `triggerHardRefresh()`

Important invalidation mechanism:

- `src/redux/common-reducers.ts` stores `hardRefresh` as `boolean | number`
- `triggerHardRefresh()` sets a timestamp so dependent screens refetch
- `ListViewTable` refetches whenever `hardRefresh` changes

### Derived state and cross-module synchronization

| Mechanism | Example |
|---|---|
| Redux slice hydration from model wrappers | `new PhysicalSpaceGetModel(res.data)` before dispatch |
| Org context from URL search params | `OrganizationProvider` reads `?orgId=` |
| Local state layered on API state | magic-link page keeps device control state, copy state, visibility toggles, etc. |
| Cross-module refresh | realtime hooks dispatch feature refetches and `triggerHardRefresh()` |

### Async orchestration style
This codebase mostly uses hand-written thunk-like action creators rather than `createAsyncThunk`.

Examples:

- `src/modules/authenticated/physical-spaces/actions/actions.ts`
- `src/modules/authenticated/meeting-room-displays/actions/actions.ts`
- `src/modules/authenticated/organization/actions/actions.ts`

### Event bus / websocket / realtime updates
ZenEdge realtime hooks:

| Hook | Purpose | Core event(s) |
|---|---|---|
| `src/hooks/useRealtimeWebhooks.ts` | webhook freshness for displays and physical spaces | `mrd:webhook.received`, `mrd:physical-space.webhook.processed` |
| `src/hooks/useRealtimeDisplayStatus.ts` | display verification/online status | `mrd:display.status.updated` |
| `src/hooks/useRealtimeIotVerification.ts` | gateway verification/server status | `mrd:gateway.status.updated` |
| `src/hooks/useDeviceStatusUpdates.ts` | door/device live state in space detail | `mrd:device.status.updated` |

These hooks generally:

- require authenticated user + local token
- connect to `${VITE_WEBSOCKET_SERVICE_URL}/realtime`
- emit `join` payloads for relevant rooms
- refresh ZenCore-backed data after ZenEdge events arrive

---

## 7) API contract map (FE perspective)
This section is from the frontend’s point of view only.

### Shared contract assumptions

| Concern | Contract |
|---|---|
| Base response envelope | `ApiResponse<T>` with `status`, `success`, `message`, `data`, `timestamp`, optional `meta` |
| List shape | `ApiResponse<T[]>` plus `meta.total`, `meta.page`, `meta.limit`, `meta.totalPages` |
| Auth | Bearer token for authenticated ZenCore requests |
| HTTP retries | none at the shared client layer |
| HTTP error normalization | non-OK responses are parsed and returned; call sites often inspect `res?.success` |

### 7A. ZenCore: auth/session and app bootstrap

| Method | Path | Request shape | Response shape | Auth | Error / retry behavior | Source |
|---|---|---|---|---|---|---|
| GET | `/version` | none | `ApiResponse<{ version: string; updatedAt: string }>` | No token assumption in code | bubbles if network fails | `src/common/actions/services.ts` |
| GET | `/auth/me` | none | `ApiResponse<{ user: IUserGet }>` | Bearer | non-OK response returned by `zenspaceApi`; 401 clears token and redirects | `src/modules/auth/actions/service.ts` |
| POST | `/auth/otp/request` | `{ identifier: string; type: 'email' }` | `ApiResponse<ReqOTPResponse>` | No bearer required | toast on thrown/network failure; caller checks `success` | `src/modules/auth/actions/service.ts` |
| POST | `/auth/otp/verify` | `{ session_id: string; code: string }` | `ApiResponse<AuthResponse>` | No bearer required | on success stores `access_token` in `localStorage`; toast on thrown/network failure | `src/modules/auth/actions/service.ts` |
| POST | `/auth/logout` | none | parsed JSON response | Bearer if token still present | called after token removal in UI logout flow | `src/modules/auth/actions/service.ts`, `src/common/global-components/sidebar-navbar/nav-user.tsx` |

### 7B. ZenCore: organization and global list queries

#### Explicit service calls

| Method | Path | Request shape | Response shape | Auth | Error / retry behavior | Source |
|---|---|---|---|---|---|---|
| GET | `/auth/organizations/:orgId` | none | `ApiResponse<IOrganizationGet>` | Bearer | toast on thrown/network failure | `src/modules/authenticated/organization/actions/services.ts` |

#### List/query patterns used by `ListViewTable`
Query shape built in `src/common/global-components/list-view/list-view.tsx`:

- `page`
- `limit`
- `sortBy`
- `sortOrder`
- optional `search`
- arbitrary filter fields appended directly

| Method | Path pattern | Returned shape | Auth | Notes / source screens |
|---|---|---|---|---|
| GET | `/auth/organizations?{params}` | `ApiResponse<IOrganizationGet[]>` | Bearer | organization list |
| GET | `/meeting-spaces?organization_id={orgId}&{params}` | paginated meeting-space list | Bearer | spaces list |
| GET | `/physical-spaces?org_id={orgId}&{params}` | paginated physical-space list | Bearer | physical spaces list |
| GET | `/physical-spaces/{id}/mappings?org_id={orgId}&{params}` | paginated mapping list | Bearer | physical-space detail |
| GET | `/displays?org_id={orgId}&{params}` | paginated MRD list | Bearer | display list |
| GET | `/iot-gateways?org_id={orgId}&{params}` | paginated gateway list | Bearer | gateway list |
| GET | `/iot-devices?org_id={orgId}&device_type=LOCK&{params}` | paginated device list | Bearer | locks list |
| GET | `/iot-devices?org_id={orgId}&device_type=WIFI_HOTSPOT&{params}` | paginated device list | Bearer | wifi list |
| GET | `/org-api-keys?org_id={orgId}&{params}` or `/org-api-keys?{params}` | paginated API key list | Bearer | API keys list |
| GET | `/displays/{displayId}/screenshots?{params}` | paginated screenshot list | Bearer | screenshot list component |
| GET | `/displays/{displayId}/webhook-logs?{params}` | paginated webhook log list | Bearer | display detail |
| GET | `/webhooks/physical-space/logs?physical_space_id={id}&{params}` | paginated webhook log list | Bearer | physical-space detail |
| GET | `/displays/{displayId}/notifications?{params}` | paginated notification list | Bearer | notification list component |
| GET | `/webhooks/logs/export` | download-like response | Bearer | wired as `downloadApi`, but full file download handling is incomplete |

Error behavior for `ListViewTable`:

- catches failures
- logs the error
- sets local table data to `[]`
- shows empty-state UI rather than a dedicated fetch error surface

### 7C. ZenCore: async select and lookup endpoints
Used by `src/common/global-components/select/select-async-paginate.tsx`

Query shape:

- `page`
- `limit=10`
- optional `search`

| Method | Path pattern | Typical purpose | Response expectation | Auth |
|---|---|---|---|---|
| GET | `/iot-gateways?org_id={orgId}&{params}` | gateway select fields | `result.data[]` array | Bearer |
| GET | `/meeting-spaces?organization_id={orgId}&{params}` | meeting space selection | `result.data[]` array | Bearer |
| GET | `/physical-spaces?org_id={orgId}&{params}` | physical-space selection | `result.data[]` array | Bearer |
| GET | `/space-groups?{params}` | spaces inline filter | `result.data[]` array | Bearer |

Error behavior:

- logs to console
- returns empty options
- no explicit retry UI

### 7D. ZenCore: meeting room displays

| Method | Path | Request shape | Response shape | Auth | Error / retry behavior | Source |
|---|---|---|---|---|---|---|
| POST | `/displays` | `IMeetingRoomDisplayPost` | `ApiResponse<IMeetingRoomDisplayGet>` | Bearer | toast on thrown/network failure; callers inspect `success` | `src/modules/authenticated/meeting-room-displays/actions/services.ts` |
| PUT | `/displays/{id}` | `Partial<IMeetingRoomDisplayPost>` | `ApiResponse<IMeetingRoomDisplayGet>` | Bearer | same | same |
| DELETE | `/displays/{id}` | none | `ApiResponse<IMeetingRoomDisplayGet>` | Bearer | same | same |
| GET | `/displays/{id}` | none | `ApiResponse<IMeetingRoomDisplayGet>` | Bearer | same | same |
| GET | `/displays/meeting-space/{meetingSpaceId}/check-link-availability?excludeDisplayId=` | none | `ApiResponse<IMeetingSpaceUnlinkGet>` | Bearer | used before linking to detect replacement flow | same |
| POST | `/displays/{id}/link-zs` | `ILinkZenSpacePost` | `ApiResponse<IMeetingRoomDisplayGet>` | Bearer | link form shows success/error toasts and refreshes current display + hardRefresh | same + `src/modules/authenticated/meeting-room-displays/components/link-zenspace-form.tsx` |
| POST | `/displays/{id}/unlink-zs` | none | `ApiResponse<IMeetingRoomDisplayGet>` | Bearer | unlink modal shows toast and refreshes | same |
| POST | `/displays/{entityId}/image` | `FormData` with `file` | `ApiResponse<any>` | Bearer | upload form shows toast and refetches | same |
| POST | `/displays/{entityId}/logo` | `FormData` with `file` | `ApiResponse<any>` | Bearer | upload form shows toast and refetches | same |

Related display-adjacent ZenCore endpoints:

| Method | Path | Request shape | Response shape | Auth | Source |
|---|---|---|---|---|---|
| GET | `/displays/meeting-spaces/{msId}/devices` | none | `ApiResponse<IMeetingSpaceLinkedDisplaysGet>` | Bearer | `src/modules/authenticated/spaces/actions/services.ts` |
| POST | `/displays/{displayId}/request-screenshot` | none | `ApiResponse<IScreenshotGet>` | Bearer | `src/modules/authenticated/screenshot/actions/services.ts` |
| GET | `/displays/{displayId}/notifications/stats` | none | `ApiResponse<INotificationStats>` | Bearer | `src/modules/authenticated/notifications/actions/services.ts` |

### 7E. ZenCore: physical spaces

| Method | Path | Request shape | Response shape | Auth | Error / retry behavior | Source |
|---|---|---|---|---|---|---|
| POST | `/physical-spaces` | `IPhysicalSpacePost` | `ApiResponse<IPhysicalSpaceGet>` | Bearer | toast on thrown/network failure | `src/modules/authenticated/physical-spaces/actions/services.ts` |
| GET | `/physical-spaces/{id}?org_id={orgId}` | none | `ApiResponse<IPhysicalSpaceGet>` | Bearer | same | same |
| PUT | `/physical-spaces/{id}?org_id={orgId}` | `Partial<IPhysicalSpacePost & { is_active?: boolean }>` | `ApiResponse<IPhysicalSpaceGet>` | Bearer | same | same |
| DELETE | `/physical-spaces/{id}?org_id={orgId}` | none | `ApiResponse<IPhysicalSpaceGet>` | Bearer | same | same |
| POST | `/physical-spaces/{physicalSpaceId}/mappings?org_id={orgId}` | `IPhysicalSpaceMappingPost` | `ApiResponse<IPhysicalSpaceMappingGet>` | Bearer | same | same |
| GET | `/physical-spaces/{physicalSpaceId}/mappings?org_id={orgId}` | none | `ApiResponse<IPhysicalSpaceMappingGet[]>` | Bearer | same | same |
| DELETE | `/physical-spaces/{physicalSpaceId}/mappings/{mappingId}?org_id={orgId}` | none | `ApiResponse<IPhysicalSpaceMappingGet>` | Bearer | same | same |
| GET | `/physical-spaces/{id}/devices?org_id={orgId}` | none | `ApiResponse<IDevicesGet>` | Bearer | same | same |
| GET | `/webhooks/physical-space/logs/{logId}` | none | `ApiResponse<IWebhookLogsGet>` | Bearer | same | same |

### 7F. ZenCore: IoT gateways, locks, wifi, and generic devices

#### IoT gateways

| Method | Path | Request shape | Response shape | Auth | Source |
|---|---|---|---|---|---|
| POST | `/iot-gateways` | `IIoTGatewayPost` | `ApiResponse<IIoTGatewayGet>` | Bearer | `src/modules/authenticated/iot-gateway/actions/services.ts` |
| GET | `/iot-gateways/{id}?org_id={orgId}` | none | `ApiResponse<IIoTGatewayGet>` | Bearer | same |
| PUT | `/iot-gateways/{id}?org_id={orgId}` | `Partial<IIoTGatewayPost & { tailscale_url?: string \| null }>` | `ApiResponse<IIoTGatewayGet>` | Bearer | same |
| DELETE | `/iot-gateways/{id}?org_id={orgId}` | none | `ApiResponse<IIoTGatewayGet>` | Bearer | same |

#### Gateway-derived lock/wifi lookups

| Method | Path | Request shape | Response shape | Auth | Source |
|---|---|---|---|---|---|
| GET | `/iot-gateways/{iotId}/locks?org_id={orgId}` | none | `ApiResponse<ILockDeviceGet[]>` | Bearer | `src/modules/authenticated/locks/actions/services.ts` |
| GET | `/iot-gateways/{iotId}/network-info?org_id={orgId}` | none | `ApiResponse<IWifiNetworkInfoGet>` | Bearer | `src/modules/authenticated/wifi/actions/services.ts` |

#### Generic IoT device CRUD

| Method | Path | Request shape | Response shape | Auth | Source |
|---|---|---|---|---|---|
| POST | `/iot-devices` | `IIoTDevicePost` | `ApiResponse<IIoTDevicePost>` | Bearer | `src/modules/authenticated/devices/actions/services.ts` |
| PUT | `/iot-devices/{deviceId}` | `Partial<IIoTDevicePost>` | `ApiResponse<unknown>` | Bearer | same |
| DELETE | `/iot-devices/{deviceId}?org_id={organizationId}` | none | `ApiResponse<unknown>` | Bearer | same |

#### Dynamic per-device action URLs
`src/modules/authenticated/devices/components/device-card.tsx` performs action calls using backend-provided URLs in `display.controls[].actions`.

Behavior:

- strips a leading `/api` from the backend-supplied action URL before calling `zenspaceApi.request()`
- uses `action.method` if provided, otherwise defaults to `POST`
- sends `{ lock: boolean }` for lock toggle flows

This is a dynamic contract owned by ZenCore. Agents should not hardcode new control endpoints here without first verifying backend payload shape.

### 7G. ZenCore: webhook logs and API keys

| Method | Path | Request shape | Response shape | Auth | Source |
|---|---|---|---|---|---|
| GET | `/webhooks/logs/{id}` | none | `ApiResponse<IWebhookLogsGet>` | Bearer | `src/modules/authenticated/webhook-logs/actions/services.ts` |
| POST | `/org-api-keys` | `IOrgApiKeyPost` | `ApiResponse<IOrgApiKeyWithPlaintext>` | Bearer | `src/modules/authenticated/org-api-keys/actions/services.ts` |
| POST | `/org-api-keys/{id}/revoke?org_id={orgId}` | none | `ApiResponse<IOrgApiKeyGet>` | Bearer | same |
| POST | `/org-api-keys/{id}/rotate` | `IOrgApiKeyRotatePayload` | `ApiResponse<IOrgApiKeyWithPlaintext>` | Bearer | same |

### 7H. ZenCore: public magic-link endpoints
These are called directly with `fetch` in `src/modules/public/magic-link/views/page.tsx`, not `zenspaceApi`.

| Method | Path | Request shape | Response shape expected by FE | Auth | Error / retry behavior |
|---|---|---|---|---|---|
| POST | `/magic-links/validate` | `{ token }` | `ApiResponse<MagicLinkData>` | No bearer; token passed in body | `requestJson()` throws on non-OK; top-level page enters `error` state |
| GET | `/magic-links/booking-access-credentials?token={token}` | none | `ApiResponse<GuestAccessPayload>` | No bearer | first attempt in a two-endpoint fallback chain |
| GET | `/magic-links/guest-access?token={token}` | none | `ApiResponse<GuestAccessPayload>` | No bearer | second attempt; if both fail, FE fabricates a permissive null-valued credential payload |
| GET | `/magic-links/devices?token={token}` | none | `ApiResponse<DeviceItem[]>` | No bearer | on failure devices reset to empty |
| GET | `/magic-links/doors/status?token={token}&device_id={id}` | none | `ApiResponse<DoorStatusData>` | No bearer | lock-specific status refresh |
| GET | `/magic-links/wifi/credentials?token={token}&device_id={id}` | none | `ApiResponse<WifiCredentialsData>` | No bearer | best-effort; null on failure |
| POST | `/magic-links/doors/unlock` | `{ token, device_id }` | `ApiResponse<any>` | No bearer | UI shows inline toast on success/failure |

Important caveat:

- `sendControlCommand()` in the magic-link page is currently a placeholder and does not call ZenCore for fan/light/climate controls

### 7I. ZenEdge: realtime/websocket contract map

| Hook | Connect target | Join payload | Server event(s) consumed | Side effects | Retry / reconnect |
|---|---|---|---|---|---|
| `useRealtimeWebhooks` | `${VITE_WEBSOCKET_SERVICE_URL}/realtime` | `{ display_id }` optional | `mrd:webhook.received`, `mrd:physical-space.webhook.processed` | refresh display or physical space, dispatch `triggerHardRefresh()`, toast on connection failures | reconnect enabled, max attempts 5 |
| `useRealtimeDisplayStatus` | `${VITE_WEBSOCKET_SERVICE_URL}/realtime` | `{ display_id }` optional | `mrd:display.status.updated` | optional refetch via `getMeetingRoomDisplayByIdService()`, dispatch `triggerHardRefresh()` | reconnect behavior via Socket.IO, custom max attempts tracking for error UI |
| `useRealtimeIotVerification` | `${VITE_WEBSOCKET_SERVICE_URL}${namespacePath}` | `{ iot_gateway_id }` optional | default `mrd:gateway.status.updated` | invoke callback, update gateway onboarding flows | no custom retry cap; toast on connection error |
| `useDeviceStatusUpdates` | `${VITE_WEBSOCKET_SERVICE_URL}/realtime` | `{ meeting_space_id }`, `{ device_ids }` | `mrd:device.status.updated` | dispatch `getSpaceLinkedDisplaysAction(meetingSpaceId)` | reconnect on server disconnect; no explicit retry UI |

### 7J. Integration summary

| Platform | Used for | Base | Transport |
|---|---|---|---|
| ZenCore | REST APIs, auth, CRUD, list screens, public magic-link validation/data | `VITE_BACKEND_URL` | HTTP |
| ZenEdge | realtime status/webhook/device/gateway events | `VITE_WEBSOCKET_SERVICE_URL` | Socket.IO |

---

## 8) User journeys / logical flows (end-to-end)

### Auth / login / session bootstrap
Flow:

1. app loads in `src/main.tsx`
2. `AuthProvider` checks `localStorage.auth_token`
3. if token exists, `GET /auth/me` is called
4. route guards hold on `Loader` while auth resolves
5. unauthenticated users are sent to `/auth`
6. login screen requests OTP by email
7. OTP verification stores `access_token` as `auth_token`
8. `checkAuth()` rehydrates user context and routes to `/organization`

Failure behavior:

- invalid email/OTP yields toasts and form errors
- 401 anywhere through `zenspaceApi` clears token and redirects to `/auth`

### Organization selection
Flow:

1. `/organization` lists available orgs from `/auth/organizations?`
2. selecting a card navigates to `/physical-spaces?orgId={org.id}`
3. `OrganizationProvider` reads `orgId`
4. provider dispatches `getOrganizationByIdAction(orgId)`
5. protected child pages render only after organization exists

Failure behavior:

- missing or bad `orgId` shows “Organization Not Found”
- recovery actions are “Go To Organization Page” and retry

### Booking create / update / cancel flows
There is no first-party booking create/update/cancel UI in this repository.

Observed behavior instead:

- booking data is displayed in space detail, display detail, and webhook log detail
- physical-space mappings influence booking-related availability windows
- lock and wifi device capabilities mention automatic behavior on booking creation/cancellation
- magic-link guest access surfaces booking-derived credentials

Agent rule:

- treat booking state here as ZenCore-provided display/integration data, not a local authoring workflow

### Physical mapping / operations UI flows
Primary flow in `physical-spaces`:

1. create or open a physical space
2. open detail page
3. create mapping to a virtual meeting space via mapping form
4. inspect active pod, booking counts, device inventory, and webhook logs
5. add linked devices such as display, lock, or wifi from physical-space detail

Important files:

- `src/modules/authenticated/physical-spaces/views/detail.tsx`
- `src/modules/authenticated/physical-spaces/components/physical-space-mapping-form.tsx`
- `src/modules/authenticated/physical-spaces/components/mapping-calendar.tsx`

### Display onboarding and linking flow
Flow:

1. create a meeting room display
2. verify display state from device-side process
3. realtime status update arrives through ZenEdge
4. operator links display to a ZenCore meeting space via `link-zs`
5. optional replacement flow checks if the target meeting space is already linked
6. if linked elsewhere, user must confirm unlink/relink flow

Important files:

- `src/modules/authenticated/meeting-room-displays/views/list.tsx`
- `src/modules/authenticated/meeting-room-displays/components/meeting-room-display-form.tsx`
- `src/modules/authenticated/meeting-room-displays/components/link-zenspace-form.tsx`
- `src/modules/authenticated/meeting-room-displays/components/unlink-zenspace-modal.tsx`

### Booking access credential display / email-trigger-related UI
What exists:

- public magic-link page displays door PIN / wifi voucher data when ZenCore exposes it
- lock and wifi forms allow editing email HTML templates used in booking-related communications
- webhook detail pages extract and display booking payload data

What does not exist:

- no direct “send booking email now” operator screen was found
- no booking create/update/cancel UI was found

Relevant files:

- `src/modules/public/magic-link/views/page.tsx`
- `src/modules/authenticated/locks/components/lock-form.tsx`
- `src/modules/authenticated/wifi/components/wifi-form.tsx`
- `src/modules/authenticated/webhook-logs/views/detail.tsx`
- `src/modules/authenticated/physical-spaces/components/physical-space-webhook-detail.tsx`

### Failure handling and user messaging paths

| Pattern | Where it appears |
|---|---|
| Sonner toasts for mutation success/failure | most authenticated forms and actions |
| `Loader` during auth/detail bootstraps | auth route guard, entity detail pages |
| `NotFound` for missing entities | physical-space detail, display webhook detail, gateway detail, webhook detail |
| `Empty` for partial absence of data | lists, status cards, embedded tabs |
| Full-page error state | public magic-link page |

---

## 9) Permissions and guardrails

### Route guards

| Guard type | Behavior | Source |
|---|---|---|
| Public route | blocks authenticated users from `/auth` by redirecting to `/organization` | `src/App.tsx` |
| Protected route | requires authenticated user, otherwise `/auth` | `src/App.tsx` |
| Organization gate | protected pages under `ProtectedLayout` require resolvable `orgId` context | `src/providers/organization-provider.tsx` |

### Role checks and UI-level access control
Observed role/permission behavior is limited.

| Mechanism | Actual behavior |
|---|---|
| Route-level roles | none found |
| `super_admin` | only used to show organization card action menu |
| Org API key permissions | managed at resource level as CRUD flags on keys, not as route guards |
| Hidden actions | some nav items are hidden by commenting them out rather than by permission checks |

### Security-sensitive UX patterns

| Concern | Current behavior |
|---|---|
| Auth token storage | stored in `localStorage` as `auth_token` |
| 401 handling | token cleared and browser redirected to `/auth` |
| API key exposure | plaintext API key shown only once after create/rotate; modal requires user to copy it before confirming close |
| Magic-link access | token passed in route and API query/body; page is intentionally unauthenticated |
| PII exposure | booking organizer email/phone/name can be visible in webhook detail pages and magic-link credential screens |
| Device/credential data | wifi credentials and door PINs can be displayed and copied in the public magic-link flow |

### Practical guardrails for agents

- do not assume route-level role enforcement exists
- do not move credential-bearing flows into logs or console output
- do not change localStorage token semantics casually
- do not broaden plaintext secret exposure beyond current one-time API key modal behavior

---

## 10) Validation and error UX catalog

### Form-level validation rules by major forms

| Form | File | Key validation rules |
|---|---|---|
| Login email | `src/modules/auth/schema/schema.ts` | email must be valid |
| OTP | same | exactly 6 characters via `min(6)` + `max(6)` |
| Physical space | `src/modules/authenticated/physical-spaces/schema/schema.ts` | `name` required, max 200; optional `description`; optional `is_active` |
| Physical-space mapping | same | `virtual_meeting_space_id`, `start_at`, `end_at` required; end date must be after start date |
| Display create | `src/modules/authenticated/meeting-room-displays/schema/schema.ts` | `physical_space_id` required |
| Link display to meeting space | same | `zenspace_api_key` required; `meeting_space_id` must be UUID; `register_webhook` defaults true |
| Display config | same | `booking_url` required and must be valid URL; several booleans optional |
| IoT gateway create/edit | `src/modules/authenticated/iot-gateway/schema/schema.ts` | `name` required max 100; optional `adapter_type`; optional `unifi_local_ip` must be valid IPv4 if provided |
| Lock create/edit | `src/modules/authenticated/locks/schema/schema.ts` | `gateway_id`, `external_id`, `display_name`, `physical_space_id` required; `display_order >= 0`; at least one capability enabled |
| Wifi create/edit | `src/modules/authenticated/wifi/schema/schema.ts` | mirrors lock rules for wifi hotspot devices; optional HTML templates for booking emails |
| Org API key create | `src/modules/authenticated/org-api-keys/schema/schema.ts` | `name` required max 200; optional valid `expires_at`; `permissions` object required |
| Org API key rotate | same | `name` optional/blank allowed; optional valid `expires_at`; `permissions` object required |

### API error normalization and toast/banner strategy

| Pattern | Behavior |
|---|---|
| Shared API client | parses JSON from failed HTTP responses and returns it |
| Most service wrappers | catch thrown/network errors and `toast.error(message)` |
| Most forms/views | inspect `res?.success`, show success or failure toast |
| List screens | degrade to empty data; no dedicated banner |
| Magic-link page | uses its own `requestJson()` and full-page error state |

### Offline / timeout / retry UX patterns

| Area | Current behavior |
|---|---|
| REST requests | no central retry or offline UX |
| Realtime webhooks | reconnect attempts and toast feedback in some hooks |
| Display status realtime | reconnect/error state tracked; not deeply surfaced in all screens |
| IoT verification realtime | connection error toast, no sophisticated fallback |
| Magic-link page | retry action exists on full-page error state |

### Fallback states and recovery actions

| Area | Recovery action |
|---|---|
| Missing org context | back to `/organization`, retry fetch |
| Missing detail entity | `NotFound` only |
| Empty lists | create CTA or informational empty card |
| Failed magic-link load | retry button on full-page error |
| API key secret modal | user must copy secret before confirming completion |

---

## 11) Environment, config, and deployment

### Env vars inventory

| Variable | Required | Used by | Observed values in local env files | Notes |
|---|---|---|---|---|
| `VITE_BACKEND_URL` | Yes | ZenCore REST requests and magic-link page | `.env.development`: `https://mrd-api-dev.zenspaceiot.com/api` ; `.env.production`: `https://api-automation-spaceos.zenspace.io/api` | browser-exposed because `VITE_*` |
| `VITE_WEBSOCKET_SERVICE_URL` | Yes for realtime | ZenEdge Socket.IO hooks | `.env.development`: `https://websocket-service.zenspace.io` ; `.env.production`: `https://websocket-service.zenspace.io` | browser-exposed because `VITE_*` |

Important config facts:

- `.env*` is gitignored
- there is no `.env.example`
- clones will not automatically know required env vars from tracked files

### Per-environment base URLs and toggles

| Environment | ZenCore base | ZenEdge base |
|---|---|---|
| Development | `https://mrd-api-dev.zenspaceiot.com/api` | `https://websocket-service.zenspace.io` |
| Production | `https://api-automation-spaceos.zenspace.io/api` | `https://websocket-service.zenspace.io` |

Special-case local fallback:

- `src/modules/public/magic-link/views/page.tsx` falls back to `http://localhost:4000/api` when running on localhost Vite ports `517*` and `VITE_BACKEND_URL` is missing

### Build, run, lint, and test commands

| Purpose | Command |
|---|---|
| Dev | `npm run dev` |
| Prod-mode local run | `npm run prod` |
| Production build | `npm run build` |
| Development-env build | `npm run build:dev` |
| Lint | `npm run lint` |
| Preview production build | `npm run preview` |
| Preview development build | `npm run preview:dev` |
| Tests | no script present |

### Config files

| File | Purpose |
|---|---|
| `vite.config.ts` | React plugin, Tailwind plugin, `@` alias |
| `tsconfig.json` | project references and `@/*` path alias |
| `tsconfig.app.json` | app compiler config; strict mode enabled |
| `eslint.config.js` | flat ESLint config for TS + React hooks + Vite refresh |
| `components.json` | shadcn-style component configuration |

### Deployment assumptions and targets
There are no tracked CI/CD manifests or hosting configs in this repo.

Observed state:

- no `.github/workflows/`
- no `Dockerfile`
- no `docker-compose`
- no `vercel.json`
- no `netlify.toml`
- `README.md` is still the default Vite template

Deployment assumptions inferred from code:

- must serve built Vite SPA assets
- must rewrite unknown routes to `index.html`
- must expose ZenCore over HTTPS
- must expose ZenEdge Socket.IO endpoint over HTTPS/WSS

---

## 12) Testing coverage map

### Tooling and scope

| Category | Status |
|---|---|
| Unit tests | none found |
| Integration tests | none found |
| E2E tests | none found |
| Vitest | not configured |
| Jest | not configured |
| Cypress | not configured |
| Playwright | not configured |
| Storybook | not configured |

### Coverage map

| Area | Coverage status | Notes |
|---|---|---|
| Auth flow | manual only | covered in `docs/USER_FLOW_AND_TESTING.md` |
| Organization selection | manual only | same |
| Physical spaces | manual only | no automated coverage |
| Displays | manual only | includes verification/linking and webhook UI, but only documented manually |
| IoT gateway | manual only | realtime verification not automated |
| Locks / Wifi | manual only | no form or mutation tests |
| API keys | manual only | no secret modal automation |
| Magic-link page | manual only | large page with no automated regression suite |

### Test data, mocks, fixtures strategy
No in-repo mock/fixture/testing strategy was found.

Current practical strategy:

- manual QA
- live backend environments via env vars
- documentation-based walkthroughs in `docs/USER_FLOW_AND_TESTING.md`

### Storybook coverage
Storybook is not used in this repo.

Implication for agents:

- any UI change is currently high-regression-risk unless the agent adds focused tests in a future testing setup or performs manual smoke verification

---

## 13) Performance, accessibility, i18n (if applicable)

### Performance-sensitive routes/components

| Area | Why it matters |
|---|---|
| `src/modules/public/magic-link/views/page.tsx` | very large single screen with many states, animations, and multiple network calls |
| `src/modules/authenticated/physical-spaces/views/detail.tsx` | very large detail page with tabs, mappings, device modals, and webhook content |
| `src/modules/authenticated/meeting-room-displays/views/detail.tsx` | large tabbed detail screen with embedded subfeatures |
| `ListViewTable` | reused broadly, so regressions affect most list screens |

### Code splitting / lazy loading
No route-level or feature-level lazy loading was found.

Observed state:

- `src/App.tsx` imports route modules eagerly
- no `React.lazy()` usage found
- no dynamic route chunking pattern found

### Accessibility
Current strengths:

- Radix/shadcn primitives provide baseline semantics for dialogs, dropdowns, tabs, etc.
- some explicit `aria-label` and `aria-live` usage exists, especially on the magic-link page and copy buttons

Current gaps:

- no explicit accessibility policy or audit docs
- no automated a11y testing
- several complex custom forms and cards likely rely on visual design conventions rather than explicit accessibility validation

### Localization / i18n
No i18n framework or localization resource system was found.

Observed state:

- no `react-intl`
- no `i18next`
- no locale resource folders
- UI strings are hardcoded in English across components

---

## 14) Critical invariants / non-negotiable rules

These constraints should be treated as do-not-break behavior unless the change explicitly includes a migration plan.

1. Authenticated ZenCore requests must continue to use `Authorization: Bearer <auth_token>` from `localStorage`.
2. 401 handling must continue to clear invalid auth state and route the user back to `/auth`.
3. Protected pages under `ProtectedLayout` assume organization context is available through `?orgId=...`.
4. Public magic-link flows must remain usable without authenticated session context.
5. ZenEdge realtime hooks must keep using JWT handshake auth and `/realtime` unless backend migration notes say otherwise.
6. `ListViewTable` query semantics (`page`, `limit`, `sortBy`, `sortOrder`, `search`, direct filter params) are a shared contract across list screens.
7. `zenspaceApi.request()` currently returns parsed failure payloads for non-OK HTTP responses; callers often depend on `res?.success` checks rather than thrown exceptions.
8. Org API keys must preserve one-time plaintext display behavior after create/rotate.
9. Do not rename public routes or backend-facing contracts without migration notes.
10. Do not silently repurpose `VITE_BACKEND_URL` or `VITE_WEBSOCKET_SERVICE_URL`; they are foundational integration contracts.
11. Hidden routes such as `/spaces` still exist and may be relied upon by deep links or unfinished flows.
12. Booking-related UI in this repo is display/integration-oriented, not an authoring system. Do not invent booking mutation behavior without explicit product requirements.

### Anti-regression checklist

- auth still works after refresh
- `/auth` still redirects authenticated users away
- protected routes still redirect unauthenticated users to `/auth`
- organization-scoped pages still work with `?orgId=...`
- list screens still refresh after mutations
- realtime hooks still reconnect and refresh relevant data
- API key secrets remain one-time visible
- magic-link page still works without logged-in session

---

## 15) AI execution guidance

### Safe change workflow for agents

1. Start from the route/page entry point in `src/App.tsx`.
2. Read the owning screen in `src/modules/**/views/`.
3. Read the feature-local components used by that screen.
4. Read the related `actions/services.ts` and `actions/actions.ts`.
5. Read the relevant reducer and model/interface files.
6. Read shared wrappers used by that screen such as `ListViewTable`, `AsyncPaginateSelect`, or layout shell components.
7. Only then implement the change.

### Change heuristics by task type

| Task type | Read first |
|---|---|
| Route/page change | `src/App.tsx`, layout file, target `views/` file |
| Form change | target `components/*form*.tsx`, target `schema/schema.ts`, related service file |
| API integration change | `src/common/helpers/api.ts`, target service file, target interface/model files |
| Realtime change | related hook in `src/hooks/`, related detail/list screen, relevant refresh dispatches |
| Org-scoped behavior | `src/providers/organization-provider.tsx`, organization reducer/actions, URL builders/navigation |
| Public guest experience | `src/modules/public/magic-link/views/page.tsx` |

### Mandatory checks before finalizing
Run or account for all of these before considering a change complete:

| Check | Command / action | Notes |
|---|---|---|
| Typecheck + build | `npm run build` | repository uses `tsc -b && vite build` |
| Lint | `npm run lint` | ESLint config is lightweight but should still be run |
| Tests / e2e smoke | no test script exists | perform at least a manual smoke pass on the affected route(s) |
| Runtime sanity | `npm run dev` or `npm run preview` if needed | especially for route, modal, and realtime changes |

### Required change discipline

- do not rename public routes or contracts without migration notes
- do not change auth token storage or 401 redirect behavior casually
- do not add backend assumptions that conflict with current `ApiResponse<T>` usage
- do not convert ZenCore list contracts into bespoke query shapes unless all consumers are updated
- do not change ZenEdge event names or room join payloads unless backend contract changes are confirmed
- do not expose API keys or guest credentials more broadly than current UX

### When tests are absent
Because this repo has no automated tests, agents should explicitly:

1. note that no test script exists
2. run lint and build
3. smoke the changed route manually where possible
4. call out residual risk in final handoff

---

## 16) Canonical references

### Architecture and shell

- `src/main.tsx`
- `src/App.tsx`
- `src/common/global-components/protected-layout.tsx`
- `src/common/global-components/organization-layout.tsx`
- `src/common/global-components/sidebar-navbar/app-sidebar.tsx`
- `src/common/global-components/sidebar-navbar/nav-user.tsx`
- `src/providers/auth-provider.tsx`
- `src/providers/organization-provider.tsx`

### State and refresh

- `src/redux/store.ts`
- `src/redux/common-reducers.ts`
- `src/redux/hooks/redux-hook.ts`

### Contracts and API access

- `src/common/helpers/api.ts`
- `src/common/interface/interface.ts`
- `src/common/actions/services.ts`
- `src/modules/auth/actions/service.ts`
- `src/modules/authenticated/**/actions/services.ts`

### Realtime / ZenEdge

- `src/hooks/useRealtimeWebhooks.ts`
- `src/hooks/useRealtimeDisplayStatus.ts`
- `src/hooks/useRealtimeIotVerification.ts`
- `src/hooks/useDeviceStatusUpdates.ts`

### Shared UI and design system

- `src/common/global-components/list-view/list-view.tsx`
- `src/common/global-components/select/select-async-paginate.tsx`
- `src/components/ui/`
- `src/index.css`
- `components.json`
- `src/lib/utils.ts`

### Representative feature modules

- `src/modules/auth/`
- `src/modules/public/magic-link/`
- `src/modules/authenticated/organization/`
- `src/modules/authenticated/physical-spaces/`
- `src/modules/authenticated/meeting-room-displays/`
- `src/modules/authenticated/iot-gateway/`
- `src/modules/authenticated/spaces/`
- `src/modules/authenticated/locks/`
- `src/modules/authenticated/wifi/`
- `src/modules/authenticated/org-api-keys/`

### Config and environment

- `package.json`
- `vite.config.ts`
- `tsconfig.json`
- `tsconfig.app.json`
- `eslint.config.js`
- `.env.development`
- `.env.production`

### Docs and manual QA

- `docs/USER_FLOW_AND_TESTING.md`
- `docs/PHYSICAL_SPACE_SETUP_GUIDE.md`

### Important caveat
`README.md` is not a reliable architecture reference for this repo. It is still the default Vite template and should not be treated as source of truth.
