# Kiosk Display Groups — Admin & Kiosk Flow

## Overview

A **Kiosk Display Group** slices a Space Group into capacity-bounded buckets so the kiosk display can render at most ~4 meeting spaces per page (the kiosk's status table overflows beyond that). Admins create one or more display groups inside a space group, assign meeting spaces to them, and then point a physical kiosk at a URL filtered to a single display group.

```
Space Group:  "Main Office"
├── Display Group "Floor 2"   (max 4)
│   ├── Pod 1
│   ├── Pod 2
│   └── Pod 3
├── Display Group "Lobby"     (max 4)
│   ├── Pod 8
│   └── Pod 9
└── Pod 5, Pod 6, Pod 7        (unassigned — fall into the kiosk's default page)
```

This doc covers the admin CRUD flow and the kiosk consumer flow as wired in `zs-admin`. Backend reference: [`docs/kiosk-display-groups-design.md`](kiosk-display-groups-design.md).

---

## 1. URL routing

| Surface | Route | Notes |
|---|---|---|
| Admin section | `/groups/:id` | Renders inside the existing Group detail page as a card just below "Meeting Spaces". |
| Public kiosk (booking-app shell) | `/booking/:orgSlug/:groupSlug/kiosk-display?display_group=:slug` | Original path, still supported via existing rewrite. |
| Public kiosk (short form) | `/:orgSlug/:groupSlug/kiosk-display?display_group=:slug` | Rewritten to the booking-app route via [next.config.ts](../next.config.ts). |
| Public kiosk (no Redux variant) | `/:orgSlug/:groupSlug/:kioskDisplayId` | Lives under `(public)/` route group; no Redux provider, uses [`usePublicMeetingSpaceStatus`](../src/hooks/use-realtime.ts). |

**`display_group` param semantics**:
- Omitted → kiosk renders all spaces in the group (no filter)
- Present → kiosk resolves the slug via `GET /kiosk-display-groups/by-slug/:slug?space_group_id=…` and filters `spaces_summary` + `spaces_availability` to rows where `kiosk_display_group_id` matches the resolved id.

The slug-not-found path surfaces a clear "Display group not found in this space group" error so a typo'd kiosk URL fails loudly instead of silently rendering the unfiltered group.

---

## 2. File map

```
src/app/(modules)/(authenticated-modules)/(dashboard)/groups/components/kiosk-display-groups/
├── interfaces.ts                       # Types (IKioskDisplayGroup, …WithCount, error envelope)
├── services.ts                         # API calls (zenspaceApi.request wrappers)
├── kiosk-display-groups-section.tsx    # Main section: ListViewTable + actions + modals
├── display-group-form-modal.tsx        # Create / Edit modal
├── assign-spaces-modal.tsx             # Bulk-assign + per-row Unassign for selected group
└── (Manage Spaces modal — colocated inside section file)

src/app/(modules)/(authenticated-modules)/(dashboard)/groups/(views)/[id]/page.tsx
└── Renders <KioskDisplayGroupsSection orgSlug groupSlug spaceGroupId /> right after the Meeting Spaces card.

src/app/(modules)/booking-app/(views)/[orgSlug]/[groupSlug]/kiosk-display/page.tsx
src/app/(modules)/(public)/[orgSlug]/[groupSlug]/[kioskDisplayId]/page.tsx
└── Kiosk consumer pages — read display_group slug, resolve, filter availability data.

src/hooks/use-realtime.ts
├── useMeetingSpaceStatus              # admin variant (uses Redux)
└── usePublicMeetingSpaceStatus        # kiosk variant (callback-only, no Redux)

src/app/(modules)/(authenticated-modules)/(dashboard)/roles/helpers/config.ts
└── kioskDisplayGroups permissions exposed to role management UI.

next.config.ts
└── Rewrite: /:orgSlug/:groupSlug/kiosk-display → /booking-app/:orgSlug/:groupSlug/kiosk-display
```

---

## 3. API surface

All endpoints are JWT-gated except `by-slug` which is public (the kiosk uses it without auth).

| Method | Path | Service | Notes |
|---|---|---|---|
| `GET` | `/kiosk-display-groups?space_group_id=…&page=1&limit=50&sortBy=display_order&sortOrder=ASC` | `listKioskDisplayGroupsService` | Used implicitly by `ListViewTable`. Each row includes `current_count`. |
| `GET` | `/kiosk-display-groups/:id` | `getKioskDisplayGroupService` | Single fetch with `current_count`. |
| `GET` | `/kiosk-display-groups/by-slug/:slug?space_group_id=…` | `getKioskDisplayGroupBySlugService` | **Public.** Used by kiosk to resolve slug → id. |
| `POST` | `/kiosk-display-groups` | `createKioskDisplayGroupService` | Body: `space_group_id`, `name`, `slug`, `max_spaces?`, `display_order?`, `description?`, `is_active?`. |
| `PUT` | `/kiosk-display-groups/:id` | `updateKioskDisplayGroupService` | Partial update. **`max_spaces` cannot drop below `current_count`** → 409. |
| `DELETE` | `/kiosk-display-groups/:id?force=true` | `deleteKioskDisplayGroupService` | Soft-refuses (409) if group has spaces. Pass `force=true` to null out FKs and proceed. |
| `POST` | `/kiosk-display-groups/:id/spaces` | `assignMeetingSpacesService` | Body: `{ meeting_space_ids: string[] }`. Atomic — over-cap → 409. |
| `DELETE` | `/kiosk-display-groups/:id/spaces/:meetingSpaceId` | `unassignMeetingSpaceService` | Sets `meeting_space.kiosk_display_group_id = null`. |

### Error codes

| Code | When | UX in zs-admin |
|---|---|---|
| `KIOSK_DISPLAY_GROUP_AT_CAPACITY` | Bulk-assign would overflow, or update would lower `max_spaces` below `current_count`. | Toast + inline `max_spaces` field error with current/cap. Submit button disables when over-cap pre-flight. |
| `KIOSK_DISPLAY_GROUP_HAS_SPACES` | Delete on a group with assigned spaces, no `force=true`. | Two-step alert: first refusal, second prompt confirms `force=true`. |
| Slug 409 | `slug` already exists within the parent space group. | Inline form error on the slug field. |

---

## 4. Permissions

Added to [`roles/helpers/config.ts`](../src/app/(modules)/(authenticated-modules)/(dashboard)/roles/helpers/config.ts):

```ts
kioskDisplayGroups: ['create', 'read', 'update', 'delete']
```

Constants exposed:
- `PERMISSIONS.READ_KIOSK_DISPLAY_GROUPS`
- `PERMISSIONS.CREATE_KIOSK_DISPLAY_GROUPS`
- `PERMISSIONS.UPDATE_KIOSK_DISPLAY_GROUPS`
- `PERMISSIONS.DELETE_KIOSK_DISPLAY_GROUPS`

The section returns `null` (renders nothing) when the user lacks `READ_*`. CRUD buttons gate on the corresponding `CREATE_/UPDATE_/DELETE_*` permission via `PermissionGuard` or inline `canUpdate`/`canDelete` checks.

> **Backend coordination**: Make sure these four `*:kioskDisplayGroups` permissions are seeded on at least one role before relying on the UI for the first time, otherwise nothing renders.

---

## 5. Admin section anatomy

### `KioskDisplayGroupsSection`

- **Inputs**: `spaceGroupId`, `orgSlug`, `groupSlug`.
- **Reads**: `dashboard.organization` for permissions; `common.hardRefresh` to invalidate the table after CRUD.
- **Writes**: dispatches `setHardRefresh(!hardRefresh)` after every successful CRUD action so the inner `ListViewTable` re-fetches.

The table is rendered with `ListViewTable<IKioskDisplayGroupWithCount>` and these columns: **Name (+ description)**, **Slug** (mono), **Capacity** (`current/max` badge, red when full), **Order**, **Active** badge, **Actions**.

### Actions per row

| Action | Where | Behavior |
|---|---|---|
| **Open kiosk** | Primary button on the row | Opens `/<orgSlug>/<groupSlug>/kiosk-display?display_group=<group.slug>` in a new tab. Disabled when org/group slug missing. |
| **Add spaces / Manage spaces** | Top of dropdown | Opens `AssignSpacesModal` when the group has free slots; opens `ManageSpacesModal` when full. Label switches based on `current_count >= max_spaces`. |
| **Edit** | Dropdown | Opens `DisplayGroupFormModal` pre-populated with the row. |
| **Delete** | Dropdown | Opens `AlertModal` with a two-step force-delete fallback. |

Row click also opens the manage modal — same target as the dropdown's "Manage spaces" item.

### `DisplayGroupFormModal`

Create + Edit form. Auto-derives `slug` from `name` until the user types in the slug field. Edit mode locks `max_spaces` `min` to the current `current_count` so users can't try to shrink a full group.

### `AssignSpacesModal`

Bulk-assign UI built on `ListViewTable<AssignableSpace>` filtered by `space_group_id` (the parent space group). The table shows ALL meeting spaces in the parent so admins can:
- Pick unassigned spaces (default selection target)
- Move spaces from another display group (assignments overwrite)
- See "Already in this group" rows that contribute zero new selections

A separate **Unassign** button column appears for rows already in the target group, so the same modal handles both directions.

Pre-flight guard: `newSelections.length > remaining` disables the submit button and surfaces an inline warning. The server still enforces capacity atomically — a 409 falls back to a toast.

### `ManageSpacesModal`

Read-only-ish view for a single display group. Built on `ListViewTable<MeetingSpaceLite>` filtered by both `space_group_id` and `kiosk_display_group_id` so it returns only that group's spaces. Each row has an **Unassign** button. Header shows the live `current/max` badge and an "Add spaces" CTA that hands off to `AssignSpacesModal` (closing this modal).

---

## 6. Kiosk consumer flow

```
Bootstrap (sequential):
  1. getOrganizationBySlugService(orgSlug)
  2. getGroupBySlugService(groupSlug, organization.id)
  3. if displayGroupSlug:
       getKioskDisplayGroupBySlugService(displayGroupSlug, group.id)
  4. fetchAvailability(group.id)        # GET /space-groups/:id/availability?include_time_slots=true

Render loop:
  - filteredAvailability = filter spaces_summary & spaces_availability by displayGroup.id
  - spaceRows = compute SpaceRow[] from filteredAvailability + bookings + unavailability
  - upcomingChips = group_time_slots filtered by wall-clock now-in-tz
  - 1 Hz tick re-evaluates active booking / chip filter

Live updates:
  - usePublicMeetingSpaceStatus subscribes to /realtime namespace for the filtered space ids
  - On 'zscloud:meeting.status_update' → re-fetch availability
```

The **availability response carries `kiosk_display_group_id` and `kiosk_display_group_name` per row** (added via the [admin model classes](../src/app/(modules)/(authenticated-modules)/(dashboard)/groups/model-interfaces/model.ts) and the [meeting-space availability model](../src/app/(modules)/(authenticated-modules)/(dashboard)/meeting-spaces/model-interfaces/model.ts)), so the filter happens entirely client-side without changing the existing availability endpoint.

---

## 7. Common recipes

### Add a display group

1. Open the Group detail page → "Display Groups" section → **+ Add display group**.
2. Form: name → slug auto-derives → optional description / max_spaces / display_order / is_active.
3. Server returns `KIOSK_DISPLAY_GROUP_AT_CAPACITY` only if you tried to lower `max_spaces` below `current_count` (impossible on create).
4. Slug uniqueness is per-space-group — same slug can repeat across different space groups.

### Move spaces between display groups

A space can be in **at most one** display group. To move from A → B:

1. Open B's "Add spaces" → pick the spaces from A → submit.
2. Backend overwrites the FK in place — no separate unassign step needed.
3. Both A's `current_count` and B's `current_count` will refresh after the next list fetch (triggered by the section's `setHardRefresh`).

### Delete a display group with spaces

1. **Delete** → server returns `KIOSK_DISPLAY_GROUP_HAS_SPACES`.
2. The alert modal flips into a force-confirm state explaining how many spaces will be unassigned.
3. Confirm → the section calls `deleteKioskDisplayGroupService(id, force=true)` which nulls FKs + soft-deletes.

### Test the kiosk for one display group

Click the row's **Open kiosk** button — it opens `/<orgSlug>/<groupSlug>/kiosk-display?display_group=<slug>` in a new tab. The header shows `Group Name · Display Group Name` so the kiosk identity is visible.

For physical kiosks, save that exact URL (or the `(public)` short form) on the device.

---

## 8. Out of scope (v2 candidates)

These intentionally weren't built in v1:

- **Per-org default `max_spaces`** — currently hard-coded to 4 in the backend. A future setting on the Organization entity would let large venues bump it.
- **Display-group-level branding** — the parent space group's branding always applies on the kiosk.
- **Server-driven rotation timing** — the kiosk currently rotates every 15 s (URL: `?rotate_ms=`). v2 may move this onto the display group itself.
- **Drag-handle reordering** — `display_order` is currently editable through the form only; no `@dnd-kit/core` integration yet.
- **`kiosk_display_group_id` on the meeting-space form** — admins can also set it from the meeting-space "Spaces" tab in v2 (guide §7.1). Today, assignment lives only in the display group section.

See [`docs/kiosk-display-groups-design.md` §10](kiosk-display-groups-design.md) for the full backend roadmap.
