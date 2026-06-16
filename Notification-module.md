# Notification Module

Admin surface for creating and managing **notification rules** — declarative
rules that fire one or more messages (App push / Email / SMS) to a set of
recipients when a booking or space event occurs.

- **Location:** `src/app/(modules)/(authenticated-modules)/(dashboard)/notifications/`
- **Route:** `/notifications/:id` (also reachable via the rewrite alias
  `/meeting-spaces/notifications/:id`)
- **Backend:** `/notifications` REST endpoints via `zenspaceApi`

---

## 1. Concept

A **notification rule** answers three questions:

| Question | Field(s) | Example |
|---|---|---|
| **When** does it fire? | `events[]` | 15 min before booking starts |
| **Who** receives it? | `recipients[]` | everyone with access to this space |
| **Where** does it apply? | scope (`meeting_space_id` / `space_group_id` / `organization_id`) | this meeting room |

Each rule carries a `name`, an `is_active` flag, and bookkeeping fields
(`created_by`, `updated_at`, etc.). At least one scope is **required**.

### Two authoring modes

The module exposes a single public `<NotificationForm/>` that internally routes
to one of two editors:

- **Simple mode** (`notification-form-simple.tsx`) — the default. The admin
  picks a trigger preset and a scope; that's it. Channels and message templates
  are **system-managed** (owned by per-member preferences, not the rule). On
  submit, the form sends only `name + scope + events + is_active`; the backend
  defaults `mediums` to all three channels and auto-builds one `space_users`
  recipient per medium.
- **Advanced mode** (`notification-form-advanced.tsx`) — a multi-step wizard for
  rules that need explicit recipients (literal email/phone/FCM token, specific
  users, role filters), channel restrictions, custom message templates, or
  multiple/custom events.

`NotificationForm` auto-selects the mode for an existing rule via the
`isSimpleRule()` heuristic (a rule is "simple" only if it has ≤1 event, all
recipients are `space_users`, all three mediums are covered, and there are no
per-rule body-template overrides). The user can manually switch Simple →
Advanced; the override resets on close. Pass `forceAdvanced` to always open the
wizard.

---

## 2. Data model

Defined in [model-interfaces/interfaces.ts](../src/app/(modules)/(authenticated-modules)/(dashboard)/notifications/model-interfaces/interfaces.ts), with default-filling classes in [model.ts](../src/app/(modules)/(authenticated-modules)/(dashboard)/notifications/model-interfaces/model.ts).

### Core enums

```ts
NotificationMedium       = 'sms' | 'email' | 'fcm'        // fcm = in-app/push
NotificationRecipientType = 'literal' | 'user' | 'org_users' | 'space_users'
```

- `literal` — a raw address/token typed in (`value` required, validated per medium).
- `user` — a specific user by UUID (`value` required).
- `org_users` — everyone in the organization.
- `space_users` — everyone with access to the scoped meeting space.

### `INotificationGet` (read shape)

| Field | Type | Notes |
|---|---|---|
| `id` | `string` | |
| `name` | `string` | |
| `description` | `string?` | |
| `mediums` | `NotificationMedium[]` | channels the rule sends on |
| `recipients` | `INotificationRecipient[]` | per-channel recipient slots |
| `is_active` | `boolean` | |
| `events` | `INotificationEvent[]` | trigger rows |
| `meeting_space_id` / `space_group_id` / `organization_id` | `string?` | scope (≥1 required) |
| `created_by` / `updated_by` | `string?` | |
| `created_at` / `updated_at` | `string` | ISO |
| `meeting_space` / `space_group` / `organization` / `created_by_user` / `updated_by_user` | expanded relations | optional embeds |

### `INotificationEvent`

```ts
{ event: string; duration: number; enabled: boolean }
```

`duration` is a **minute offset relative to event time**: **negative = before**,
**positive = after**, `0` = at the moment. Validation clamps it to `[-60, +60]`
in the advanced event schema; the simple form allows `0..1440` minutes before.

### `INotificationRecipient`

```ts
{
  type: NotificationRecipientType;
  value?: string | null;          // address/token/user-id for literal|user
  medium: NotificationMedium;
  body_template: string | { title?, body, data? };
  filters?: { roles?: string[] } | null;
}
```

### Write shapes

- `INotificationPost` — `mediums` and `recipients` are **optional** (simple mode
  omits them; backend fills defaults).
- `INotificationPatch = Partial<INotificationPost>`.

---

## 3. API services

[actions/services.ts](../src/app/(modules)/(authenticated-modules)/(dashboard)/notifications/actions/services.ts) — thin wrappers over `zenspaceApi`. All toast on error and rethrow, **except** dry-run.

| Function | Method / Endpoint | Notes |
|---|---|---|
| `getNotificationByIdService(id)` | `GET /notifications/:id` | |
| `createNotificationService(payload)` | `POST /notifications` | |
| `updateNotificationService(id, payload)` | `PUT /notifications/:id` | accepts partial |
| `deleteNotificationService(id)` | `DELETE /notifications/:id` | |
| `dryRunNotificationService(payload)` | `POST /notifications/dry-run` | **does not toast** — surfaces errors to caller for inline display |

### Dry-run (live recipient preview)

`POST /notifications/dry-run` resolves a recipient spec against a scope and
returns counts + sample members **without sending anything**:

```ts
INotificationDryRunResponse = {
  by_recipient: [{ recipient_index, type, medium, recipient_count, samples[] }],
  total_recipients: number,   // authoritative count
}
```

`samples` is capped server-side (~5 per slot), so per-member breakdowns are
approximate; trust `total_recipients` for the headline number.

### Delivery tracking

`GET /notifications/deliveries` (query: `INotificationDeliveryQuery`) returns
`INotificationDelivery[]` — an audit log of what was actually sent, with
`status: 'sent' | 'failed' | 'skipped'`, `error`, `body_excerpt`, and timestamps.
Rendered by the deliveries drawer.

---

## 4. Helpers

| File | Exports | Purpose |
|---|---|---|
| [helpers/triggers.ts](../src/app/(modules)/(authenticated-modules)/(dashboard)/notifications/helpers/triggers.ts) | `TRIGGER_OPTIONS`, `compileTriggerToEvents()`, `deriveTriggerFromEvents()` | Map between admin-friendly trigger presets and raw `INotificationEvent` rows. |
| [helpers/describe-events.ts](../src/app/(modules)/(authenticated-modules)/(dashboard)/notifications/helpers/describe-events.ts) | `describeEvent()`, `describeEvents()` | Render an event row as a phrase, e.g. "15 min before booking starts". |
| [helpers/member-reach.ts](../src/app/(modules)/(authenticated-modules)/(dashboard)/notifications/helpers/member-reach.ts) | `pivotDryRunByMember()`, `formatMemberChannels()`, `formatMemberLabel()` | Pivot a dry-run response from "by channel" to "by member" for the reach preview. |
| [helpers/use-notification-dry-run.ts](../src/app/(modules)/(authenticated-modules)/(dashboard)/notifications/helpers/use-notification-dry-run.ts) | `useNotificationDryRun()` | Debounced (250ms) live dry-run hook; stays silent on malformed input. |

### Trigger presets → event codes

| Preset (`TriggerKind`) | Compiles to | Offset input? |
|---|---|---|
| `before_start` | `booking.start`, `duration = -|n|` | yes (default 15) |
| `on_start` | `booking.start`, `0` | no |
| `before_end` | `booking.end`, `-|n|` | yes (default 5) |
| `on_end` | `booking.end`, `0` | no |
| `booking_created` | `booking.created`, `0` | no |
| `booking_updated` | `booking.updated`, `0` | no |
| `booking_cancelled` | `booking.cancelled`, `0` | no |
| `space_state_changed` | `space.stateChanged`, `0` | no |
| `custom` | any event code from the catalog | yes |

`deriveTriggerFromEvents()` is the best-effort inverse used when hydrating the
simple form for editing; rules it can't cleanly map (multiple events, etc.) get
`kind: 'custom'` with `needsAdvanced: true` so the UI can offer the advanced editor.

---

## 5. Components

| File | Role |
|---|---|
| `notification-form.tsx` | Public entry point; routes to simple/advanced. |
| `notification-form-simple.tsx` | One-step dialog (preset + scope). |
| `notification-form-advanced.tsx` | Multi-step wizard (recipients, channels, templates). |
| `notifications-list.tsx` | List of rules. |
| `notification-detail.tsx` | Single-rule detail view (loaded by the `[id]` route). |
| `notification-card.tsx` / `notification-card-skeleton.tsx` | Rule cards + loading state. |
| `notification-rule-summary-card.tsx` | Compact rule summary (uses `describeEvents`). |
| `notification-deliveries-drawer.tsx` | Sheet showing the delivery audit log for a rule. |
| `meeting-space-notifications-panel.tsx` | Merged Members + Notifications tab on a meeting space; shows live "→ N of M will receive" via a panel-level dry-run. |

---

## 6. State

[reducers/reducers.ts](../src/app/(modules)/(authenticated-modules)/(dashboard)/notifications/reducers/reducers.ts) — a small Redux slice (`notifications`) holding the single
currently-viewed `notification`. `getNotificationByIdAction(id)` (in
[actions/actions.ts](../src/app/(modules)/(authenticated-modules)/(dashboard)/notifications/actions/actions.ts)) fetches a rule and dispatches `setNotificationById`,
wrapping the response in `NotificationGetModel` for default-filling. List data is
fetched directly by the list components (not held in this slice).

---

## 7. Validation

[schema/schema.ts](../src/app/(modules)/(authenticated-modules)/(dashboard)/notifications/schema/schema.ts) (Zod):

- **`notificationFormSchema`** — advanced mode. Enforces: ≥1 medium, ≥1 recipient,
  ≥1 event, ≥1 scope; each recipient's `medium` must be one of the selected
  `mediums`; per-medium `value` validation (email format, international phone via
  `phoneNumberRefine`, FCM token regex, user UUID).
- **`notificationFormSimpleSchema`** — simple mode. Requires `name`, a
  `triggerKind`, an `offsetMinutes` in `0..1440`, a `customEventCode` when
  `triggerKind === 'custom'`, and ≥1 scope. Omits mediums/recipients/templates by
  design.

---

## 8. Permissions

Guarded via `PermissionGuard` (see [roles/helpers/config.ts](../src/app/(modules)/(authenticated-modules)/(dashboard)/roles/helpers/config.ts)):

| Permission | Key |
|---|---|
| Read | `read:notifications` |
| Create | `create:notifications` |
| Update | `update:notifications` |
| Delete | `delete:notifications` |

The `[id]` detail route renders a `NotFound`/unauthorized fallback when the user
lacks `READ_NOTIFICATIONS`.
