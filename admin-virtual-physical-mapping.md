# Physical Space to Virtual Space Mapping Guide <!-- omit in toc -->

This document explains how a dashboard user connects a physical space from MRD to a Zenspace virtual meeting space.

- [1. What this flow does](#1-what-this-flow-does)
- [2. Where to access it](#2-where-to-access-it)
- [3. Before you start](#3-before-you-start)
- [4. Connect a physical space](#4-connect-a-physical-space)
- [5. If a physical space is already linked](#5-if-a-physical-space-is-already-linked)
- [6. Date range rules](#6-date-range-rules)
- [7. What happens behind the scenes](#7-what-happens-behind-the-scenes)
- [8. Troubleshooting](#8-troubleshooting)

---

## 1. What this flow does

This flow lets an operator map:

- a Zenspace virtual meeting space
- to an MRD physical space
- for a selected active date range

The dashboard does not permanently store the raw MRD API key in the browser. Instead, the UI creates a short-lived MRD session and uses that to load physical spaces and create the mapping.

---

## 2. Where to access it

Open the dashboard and go to:

1. **Meeting Spaces**
2. Open a specific meeting space detail page
3. Open the **Physical Space** tab

This tab is implemented in:

- `src/app/(modules)/(authenticated-modules)/(dashboard)/meeting-spaces/components/tabs/physical-space-tab.tsx`

---

## 3. Before you start

You need:

- access to the target meeting space in the dashboard
- a valid MRD API key
- a meeting space that is not already linked to another physical space

The screen also uses:

- `organization.id` from the dashboard Redux store
- the meeting space availability window from `available_from` and `available_until`

---

## 4. Connect a physical space

Follow this flow in the UI:

1. Open the **Physical Space** tab on a meeting space detail page.
2. Click **Connect Now**.
3. In the modal, enter the **MRD API key**.
4. Click **Connect**.
5. After a successful connection, the dashboard loads available MRD physical spaces.
6. Select the physical space you want to use.
7. Choose the mapping period using the date range picker.
8. Click **Map Virtual Space**.

After success:

- the mapping is created in the backend
- the physical-space list refreshes
- the meeting space can appear as linked on the next detail refresh

---

## 5. If a physical space is already linked

If the meeting space detail API already returns a populated `physical_space` object with a `physical_space_id`, the tab does **not** allow creating a new mapping immediately.

Instead, the UI shows:

- the current physical space name
- the current physical space ID
- the mapping ID
- an **Unlink First** action

This prevents users from accidentally creating a second active link for the same virtual meeting space.

To replace an existing link:

1. Click **Unlink First**
2. Confirm removal
3. Wait for the unlink to complete
4. Connect again and create a new mapping

---

## 6. Date range rules

The mapping period uses the shared date range selector component.

Rules:

- the selected date range must be valid
- the range must stay inside the meeting space availability window
- the start date cannot be before `available_from`
- the end date cannot be after `available_until`

The selected range is submitted as:

- `start_at`: start of the selected start day
- `end_at`: end of the selected end day

---

## 7. What happens behind the scenes

The UI currently performs this sequence:

1. Create a short-lived MRD session:
   - `POST /api/mrd-sessions`
2. Load physical spaces using that session:
   - `GET /api/physical-spaces?mrd_session_id=...`
3. Create the mapping:
   - `POST /api/physical-spaces/:id/mappings`
4. Remove a mapping when unlinking:
   - `DELETE /api/physical-spaces/:id/mappings/:mappingId`

Important implementation details:

- the raw MRD API key is entered only in the modal
- the raw key is not persisted in local storage
- only the short-lived MRD session reference is stored locally
- the organization ID comes from Redux, not from manual user input

Related files:

- `src/app/(modules)/(authenticated-modules)/(dashboard)/meeting-spaces/components/tabs/physical-space-tab.tsx`
- `src/app/(modules)/(authenticated-modules)/(dashboard)/meeting-spaces/actions/physical-space-services.ts`
- `src/app/(modules)/(authenticated-modules)/(dashboard)/meeting-spaces/model-interfaces/physical-space-interfaces.ts`
- `src/app/(modules)/(authenticated-modules)/(dashboard)/meeting-spaces/model-interfaces/interfaces.ts`

---

## 8. Troubleshooting

### No connect button appears

Check whether the meeting space already has a linked `physical_space`. If it does, the UI will show **Unlink First** instead of the connect flow.

### The connect modal shows an inline error under the API key field

This usually means:

- the MRD API key is missing
- the MRD API key is invalid
- the backend rejected the session creation request

### No physical spaces load after connecting

Possible reasons:

- the MRD session expired
- the MRD API key does not have access to physical spaces
- the backend could not fetch spaces from MRD

Try reconnecting to create a fresh session.

### The selected date range is rejected

Check that the selected mapping range falls fully between:

- `available_from`
- `available_until`

### Mapping is blocked even after reconnecting

If the meeting space still returns a linked `physical_space`, unlink that mapping first before trying to create a new one.
