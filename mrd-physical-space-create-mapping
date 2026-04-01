# Physical Space Setup Guide

This guide explains how to set up a physical space in Meeting Room Display (MRD), assign devices to it, create the virtual pod mapping, and complete the admin-side API-key mapping flow.

---

## 1. Goal

After following this flow, you should have:

- a physical space created in MRD
- one or more devices assigned to that physical space
- a display linked to that physical space
- a virtual pod mapped to the physical space for the required date range
- an organization API key available for the admin-side integration step

---

## 2. Prerequisites

Before starting, make sure you have:

- access to the correct organization in MRD
- an IoT gateway already created if you plan to add Lock or Wi-Fi devices
- the virtual meeting space ID that should be mapped to the physical space
- access to the admin side where API-key based mapping is completed

---

## 3. Create the Physical Space

1. Open MRD and switch to the target organization.
2. Go to `Physical Spaces`.
3. Click `Create`.
4. Enter the required space details such as:
   - name
   - description, if needed
   - active status
5. Save the form.
6. Open the newly created physical space detail page.

The physical space detail page is the main place where you manage:

- mappings
- devices
- active virtual pod details
- webhook logs for linked display devices

---

## 4. Assign Devices to the Physical Space

Open the physical space detail page and go to the `Devices` section.

Use `Add Device` and choose the device type you want to link.

### 4.1 Add a Display

1. Click `Add Device`.
2. Choose `DISPLAY`.
3. The display form uses the current physical space automatically.
4. Create the display.
5. Complete the display verification step if prompted.

Notes:

- A meeting room display is now created against `physical_space_id`.
- The display is installed under the selected physical space, not by manually entering a display name for a meeting space.

### 4.2 Add a Lock

1. Click `Add Device`.
2. Choose `LOCK`.
3. Select the IoT gateway.
4. Select the lock device from the fetched gateway devices.
5. Fill in the device details.
6. Select the `Linked physical space`.
7. Optionally select `Linked meeting space`.
8. Save the lock.

Notes:

- `physical_space_id` is required.
- `zenspace_meeting_space_id` is optional.

### 4.3 Add Wi-Fi

1. Click `Add Device`.
2. Choose `WIFI`.
3. Select the IoT gateway.
4. Review the Wi-Fi network info loaded from the gateway.
5. Fill in the device details.
6. Select the `Linked physical space`.
7. Optionally select `Linked meeting space`.
8. Save the Wi-Fi device.

Notes:

- `physical_space_id` is required.
- `zenspace_meeting_space_id` is optional.

---

## 5. Map the Physical Space to a Virtual Pod

Once the physical space exists, create the mapping window for the virtual pod.

1. Open the physical space detail page.
2. Go to the `Mappings` tab.
3. Click `Add Mapping`.
4. Enter:
   - `Virtual Meeting Space ID`
   - `Start date`
   - `End date`
5. Save the mapping.

What this does:

- links the physical space to a virtual pod for a specific date range
- shows the mapping in the timeline/calendar
- drives the `Active Virtual Pod` section when the mapping is currently active

Important:

- the end date must be later than the start date
- mapping dates are handled using the organization timezone in the UI

---

## 6. Verify the Active Virtual Pod

After creating a valid current mapping, check the physical space detail page:

- `Mappings` shows the date window and timeline
- `Virtual Pod` shows:
  - current pod details
  - status timeline with previous/current/next state
  - booking details when available

This is the quickest way to confirm that the physical space is linked to the expected virtual pod.

---

## 7. Create or Copy the Organization API Key

If the admin-side integration requires an API key:

1. Go to `API Keys` in MRD.
2. Click `Create`.
3. Create an org API key for the required organization.
4. Copy the plaintext API key immediately when shown.

Important:

- the plaintext key is only shown once
- store it securely
- use the minimum permissions needed for the admin integration

---

## 8. Complete the Admin-Side Mapping

The in-app physical space flow in MRD handles:

- physical space creation
- device assignment
- display creation
- date-based virtual pod mapping

If your organization also requires an admin-side API-key mapping step, complete it in the admin system using the copied API key.

Recommended admin-side data to confirm:

- organization
- physical space ID
- physical space name
- display ID, if the display is part of the integration
- virtual meeting space ID
- API key

Suggested admin checklist:

1. Open the admin panel for the same organization.
2. Paste or register the org API key.
3. Select or confirm the target physical space.
4. Link the correct virtual meeting space / pod.
5. If required, link the related meeting room display device.
6. Save the configuration.
7. Return to MRD and verify the result in the physical space detail page.

Note:

- The old in-app ZenSpace linking flow is no longer the primary setup path in MRD.
- Physical-space setup is done in MRD first, while any external API-key mapping is completed in admin.

---

## 9. Validation Checklist

Use this checklist after setup:

- [ ] Physical space was created successfully.
- [ ] Display is assigned to the physical space.
- [ ] Lock is assigned to the physical space, if used.
- [ ] Wi-Fi is assigned to the physical space, if used.
- [ ] Mapping exists for the correct virtual meeting space ID.
- [ ] Mapping dates are correct.
- [ ] `Virtual Pod` tab shows the expected active pod details.
- [ ] Admin-side API-key mapping is saved.
- [ ] Webhook logs appear for the linked display when events are triggered.

---

## 10. Recommended Order

For the cleanest setup, use this order:

1. Create the physical space.
2. Add the display device to the physical space.
3. Add lock and Wi-Fi devices, if needed.
4. Create the virtual pod mapping.
5. Generate or copy the org API key.
6. Complete the admin-side API-key mapping.
7. Verify the active virtual pod and webhook logs.

