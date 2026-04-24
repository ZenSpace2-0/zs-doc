# Adapters — Operator Guide

A practical guide to adding and managing device adapters in SpaceOS. If you set up meeting rooms with smart locks, Wi-Fi hotspots, fans, or lights, this is for you.

No code in here. If you need the technical reference, see [FE_ADAPTERS.md](./FE_ADAPTERS.md).

---

## What is an adapter?

An **adapter** is a small bridge that lets SpaceOS talk to one smart device in a meeting room — a door lock, a Wi-Fi hotspot, a fan, a light, or anything similar. Think of it as a translator: the device speaks its own language, the adapter translates it into something SpaceOS understands, and vice versa.

Once you add an adapter, three things start working automatically:

1. **The device appears on the guest's magic-link page** when a booking starts, so guests can unlock doors, see Wi-Fi credentials, or control the room.
2. **SpaceOS monitors its health**, so you'll know if the device goes offline.
3. **You can send commands to it** from the admin console — lock a door, generate a Wi-Fi voucher, etc.

One adapter = one device. If a room has a door lock and a Wi-Fi hotspot, you add two adapters.

---

## When would I add an adapter?

Typical scenarios:

- You installed a new smart lock on a meeting-room door and want guests to unlock it from their booking link.
- You have a UniFi access point and want bookings to automatically issue Wi-Fi vouchers.
- You added a smart fan or light that should appear on the guest's page.
- An integrator delivered a custom device with its own web-based control UI that should live alongside your other rooms.

If the device doesn't have a URL and an API key, it's not ready to become an adapter yet — check with your integrator.

---

## Before you start

You'll need two pieces of information from whoever installed the device (or the vendor portal):

| What                                                                                       | Example                                                       |
| ------------------------------------------------------------------------------------------ | ------------------------------------------------------------- |
| **Endpoint URL** — the full web address for that specific device, including its unique ID. | `https://space-os.tailc029f9.ts.net?device_id=ada-58b6b21b-…` |
| **API key** — a secret token that proves SpaceOS is allowed to talk to the device.         | `sk-live-…`                                                   |

The URL **must** contain the `?device_id=…` part. If you only have a bare URL, go back to your integrator and ask them to supply the full one — there's no separate field for the device ID in the form.

You also need to already have the **physical space** set up in SpaceOS. Adapters live inside a space; they don't exist on their own.

---

## Step-by-step: adding an adapter

1. Open the physical space the device belongs to.
2. Switch to the **Adapters** tab.
3. Click **Add Adapter** (top right of the list).
4. In the dialog:
   - Paste the full **Adapter Endpoint URL**.
   - Paste the **API Key**.
   - Click **Preview Capabilities**.
5. SpaceOS now asks the device what it can do. After a moment, you'll see a summary screen with:
   - **Device Type** — lock, wifi, fan, light, etc.
   - **Criticality** — how important this device is (CRITICAL, STANDARD, OPTIONAL).
   - **Protocol** — the adapter's version.
   - **Webhook Capable** — whether the device pushes live events.
   - **HTML Embed** — whether guests will see a live tile.
   - **Actions** it supports (e.g. unlock, status, generate voucher).
6. If everything looks right, click **Confirm & Onboard**. The adapter is now registered and will start showing up in the list and on guest pages.

If something is off — wrong device type, missing actions — click **Back**, fix the URL or API key, and try again.

---

## The adapters list

Once you have at least one adapter, the list looks like a table by default. Each row is one adapter.

| Column            | What it shows                                                                                                                                         |
| ----------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Name**          | The friendly name the device reports (e.g. "Door 9728"). A colored badge next to it shows the criticality. Below, the device type (lock, wifi, fan…). |
| **Brand · Model** | Who makes the device (e.g. _UniFi Access · UA-Door-door_). Blank for very old adapters.                                                               |
| **Endpoint**      | Just the host part of the URL, for a quick sanity check. Hover to see the full URL.                                                                   |
| **Status**        | `Active` (green), `Disabled` (grey), or `Error` (red). A small warning icon appears when recent health checks have been failing.                      |
| **Health**        | When the device was last pinged ("Last pinged 2 min ago") and how many recent failures there have been.                                               |
| **Actions**       | A visibility switch and a kebab menu (the three dots) with everything else. See below.                                                                |

You can switch to a card/grid view using the toolbar toggle on the right. Both views show the same information and controls — pick whichever you prefer.

---

## Day-to-day controls

Everything you can do to one adapter lives in the **Actions** column.

### Guest visibility switch

The small toggle next to the kebab.

- **On** (default): the device tile shows up on the guest's magic-link page.
- **Off**: the device keeps working normally — it's still polled, actions still work — but guests just don't see the tile.

Use this when you want to temporarily hide a device (say, for a renovation or a private event) without turning it off completely.

**When you turn it off**, SpaceOS asks for confirmation. Turning it back on is instant.

**When the adapter is disabled** (see below), the visibility switch is greyed out. You need to re-enable the adapter first.

### The kebab menu (the three dots)

Click it on any row. You'll see:

- **Test connection** — sends a ping to the device. A toast message tells you whether it responded and how fast. Use this when you suspect something's wrong.
- **Preview UI** — opens a popup showing exactly what a guest (or admin, or member) would see when they load that device's tile. Great for sanity-checking a new install or debugging display issues. Greyed out if the device doesn't support an embedded UI.
- **Dispatch action** — opens a small sub-menu of commands the device supports, split into two groups:
  - **Admin** — commands only operators should run (lock a door, view the audit log, change access rules).
  - **Guest-equivalent** — commands the guest can normally run themselves (unlock, status). Running them from here is useful when you need to operate the device on a guest's behalf.
- **Disable / Re-enable** — turns the adapter off (or back on). See below.
- **Delete** — permanently removes the adapter and its stored credentials. Cannot be undone; asks for confirmation.

While any of these are running, the kebab icon turns into a small spinner.

### Running an action

When you click a command in **Dispatch action**, one of two things happens:

**For simple commands** (unlock, lock, status, audit_log…):

A confirmation dialog appears: _"Send 'unlock' to Door 9728?"_ Click **Send**. A toast tells you whether the device accepted the command and what it replied.

**For commands that need extra input** (generate voucher, grant access…):

A form opens with the fields the device needs — time limits, guest counts, data caps, etc. The form is generated from what the device itself advertises, so what you see depends on the specific adapter. Fill in what you need and click **Dispatch**.

### Previewing the UI

Click **Preview UI** in the kebab. A popup opens with:

- The device's live tile, rendered exactly as a guest would see it.
- A **Role** selector at the top:
  - **Guest** — what a guest with a booking sees.
  - **Admin** — what an operator sees (may include extra controls).
- **Custom window** checkbox — set a specific booking start and end time to preview how the tile looks for that window. Useful for testing future reservations.
- **Refresh** button — re-fetches the tile with the current settings.

If the preview fails, the popup shows a message with a **Retry** button.

### Disabling an adapter

Click **Disable** in the kebab, confirm the dialog. From that point:

- SpaceOS stops polling the device.
- Incoming webhook events from the device are rejected.
- The device tile is hidden from guests (regardless of the visibility switch).
- Actions sent to the device will fail.

The adapter stays registered — you haven't deleted it. To bring it back, click **Re-enable** (same kebab spot). No confirmation needed to re-enable.

Use **Disable** when you want to pause a device for maintenance, a vendor swap, or a suspected security issue.

### Deleting an adapter

Click **Delete**, confirm the dialog. The adapter and its stored API key are permanently removed. If you later want that device back, you have to onboard it again from scratch.

---

## Status cheat sheet

| You see…                            | It means…                                                                        | What to do                                                                                                                   |
| ----------------------------------- | -------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------- |
| **Active** (green badge)            | Everything's working.                                                            | Nothing.                                                                                                                     |
| **Active** + orange warning icon    | Recent pings have been failing, but the device is still answering overall.       | Click **Test connection** to see the exact error. If it keeps happening, check the device's network or the vendor dashboard. |
| **Error** (red badge)               | Too many recent ping failures — SpaceOS has flagged the device as offline.       | Use **Test connection** and the vendor portal to investigate. The guest still sees the tile but labeled as unavailable.      |
| **Disabled** (grey badge)           | You (or someone else) explicitly turned the adapter off.                         | Re-enable it from the kebab when you're ready.                                                                               |
| **Hidden** small badge next to name | The visibility switch is off — adapter is running but guests won't see the tile. | Flip the switch back on when you want guests to see it again.                                                                |

---

## What the guest sees

When a guest opens their magic link:

- Your space name and booking time appear at the top.
- Each visible, active adapter shows up as its own tile, grouped by type (Door access, Wi-Fi, Climate, Lighting…).
- Each tile is rendered by the device itself — so the look and the controls come from the vendor, not SpaceOS.
- If a device is offline or failing, the tile says "This device is temporarily unavailable." The guest can still see the other devices.
- Devices with the visibility switch off, or with the adapter disabled, don't appear at all.

The guest never sees any credential text (PINs, vouchers) from SpaceOS — that's rendered inside the device's own tile, and SpaceOS never stores it.

---

## Common scenarios

**I want to hide a device from guests for a day.**
Use the **Guest visibility** switch. Flip it off now, flip it on tomorrow. The device keeps running in the background.

**I want to stop the device entirely — it's being replaced.**
Use **Disable** (or delete it if you're replacing with a different device).

**A guest says their booking page shows "Device unavailable."**
The adapter's status is probably `error`. Click **Test connection** on the row. Share the error with whoever maintains the device.

**I want to see what the guest is seeing right now.**
Click **Preview UI** on the adapter. Choose role = Guest. That's exactly the tile they're looking at.

**I want to unlock a door for a guest who can't find their booking link.**
Click the kebab → **Dispatch action** → under **Guest-equivalent**, pick **unlock**. Confirm and send.

**A device got a new URL after a firmware update.**
There's no "edit URL" option today — delete the old adapter and re-onboard with the new URL. Your integrator should confirm the API key is still valid.

**The device type I see is wrong (e.g. shows "display" instead of "lock").**
The adapter itself is misreporting. Don't onboard it — check with your integrator to fix the device firmware.

**Ping works but actions fail with "Adapter rejected."**
The device answered, but refused the specific command. The error message after the colon is from the device — usually a permission or state issue (e.g., trying to unlock an already-unlocked door). Check the device's own admin portal.

**Actions fail with "Adapter unreachable."**
The device isn't responding at all right now. Network issue on the device side. Wait a moment and retry; if it persists, the device is down.

---

## Glossary

- **Adapter** — the software bridge between SpaceOS and one physical device.
- **Physical space** — the meeting room itself, as SpaceOS knows it. Adapters live inside a physical space.
- **Endpoint URL** — the specific web address (including `?device_id=…`) SpaceOS uses to talk to an adapter.
- **API key** — the secret token the adapter requires before it will accept commands.
- **Magic link** — the short-lived link guests receive with a booking; it opens the guest page showing all the devices in the room.
- **Criticality** — a vendor-supplied label (`CRITICAL`, `STANDARD`, `OPTIONAL`) hinting at how essential the device is. Currently informational only.
- **Action** — a command you can send to the device (unlock, lock, generate voucher, etc.). Each adapter advertises its own list.
- **Health check / ping** — a lightweight request SpaceOS sends to confirm the device is still reachable.
- **Preview** — a read-only, in-console view of the live tile the guest would see.

---

## Where to get help

- For deeper technical questions, see [FE_ADAPTERS.md](./FE_ADAPTERS.md).
- For how the guest page works end-to-end, see [FE_ADAPTER_MAGIC_LINK_SYNC.md](./FE_ADAPTER_MAGIC_LINK_SYNC.md).
- For setting up the physical space itself, see [PHYSICAL_SPACE_SETUP_GUIDE.md](./PHYSICAL_SPACE_SETUP_GUIDE.md).
- For vendor-specific issues (the device isn't responding, the URL changed, the API key is rejected), contact your device integrator or vendor. SpaceOS is a message carrier — the device owns its own configuration.
