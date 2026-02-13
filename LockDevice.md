## Lock Booking & Control Flow: ZenSpace Cloud → MRD → Raspi/Gateway → Device

### Overview

This document defines the **end-to-end lock flow** across:

- **ZenSpace Cloud** → calls MRD lock APIs  
- **MRD** → calls Raspi/gateway APIs  
- **Raspi/gateway** → talks to physical lock / UniFi  

The goal is for all lock operations to be **fully synced**:

> **ZenSpace Cloud → MRD → Raspi/Gateway → Device**

This doc is **API-contract only** (endpoints, request/response payloads), no implementation details.

---

## 1. Create / Update Lock Booking

### 1.1 ZenSpace Cloud → MRD

- **Endpoint (MRD)**: `POST /api/v1/lock/booking`
- **Purpose**: Create or update a booking for a meeting space and provision lock (and optionally WiFi) access.

**Request body** (ZenSpace → MRD):

- `booking_id` (string, required): Booking ID in ZenSpace
- `meeting_space_id` (string, required): ZenSpace meeting space ID
- `start_time` (ISO 8601, required): Booking start time
- `end_time` (ISO 8601, required): Booking end time
- `guest` (object, optional): Guest info
  - `name` (string, optional): Guest name
  - `email` (string, optional): Guest email
  - `phone` (string, optional): Guest phone
- `booking_details` (object, optional): Booking metadata
  - `title` (string, optional): Booking title
  - `space_name` (string, optional): Space name
  - `location` (string, optional): Location name
  - `org_name` (string, optional): Organization name

> **Note**: WiFi / voucher configuration is **not** part of this payload.  
> If/when WiFi is modeled as a separate device type, it will have **its own APIs and payloads**, independent of the lock booking payload.

**Example request**:

```json
{
  "booking_id": "booking-uuid-123",
  "meeting_space_id": "zs-space-456",
  "start_time": "2026-01-27T10:00:00.000Z",
  "end_time": "2026-01-27T12:00:00.000Z",
  "guest": {
    "name": "Jane Doe",
    "email": "jane@example.com"
  },
  "booking_details": {
    "title": "Team Standup",
    "space_name": "Conference Room A"
  }
}
```

**MRD behavior (conceptual)**:

1. Resolve **primary LOCK device** for `meeting_space_id` (from `iot_devices`).
2. Use the booking window **as provided**:
   - `validFrom = start_time`
   - `validUntil = end_time`
3. Look up the **gateway** for that lock:
   - Use `gateway_id` on the device
   - From gateway, derive `baseUrl`:
     - Prefer `tailscale_url` (e.g. `https://zencam-1.tail.../`)
     - Fallback `https://{unifi_local_ip}` if configured
4. Call Raspi/gateway booking API with the payload below.

---

### 1.2 MRD → Raspi/Gateway

- **Endpoint (Raspi)**: `POST {baseUrl}/api/v1/bookings`
- **Purpose**: Create or update booking with lock PIN and optional WiFi voucher at the gateway.

**Key mapping (MRD → Raspi)**:

- `id` ← `booking_id` from ZenSpace
- `door_id` ← **external_id** of the LOCK device (`iot_devices.external_id`, UniFi door id)
- `booking_start_time` ← buffered `validFrom` (ISO)
- `booking_end_time` ← buffered `validUntil` (ISO)
- `user_id` ← `guest_name` (or a dedicated ID if available)
- `user_email` ← `guest_email`
- `status` ← `"active"` for new bookings
- `wifi_voucher` ← `wifi_voucher` from ZenSpace payload
- `voucher_name`, `seating_capacity`, `voucher_max_speed_kbps` ← passed through when present
- `site_id` ← optional (configured per gateway or omitted if managed by Raspi)

**Request body** (MRD → Raspi):

- `id` (string, required): Booking ID from cloud/booking system
- `door_id` (string, required): Door device ID (UniFi door id)
- `booking_start_time` (ISO 8601, required): Valid access start time
- `booking_end_time` (ISO 8601, required): Valid access end time
- `user_id` (string, optional): User/guest identifier
- `user_email` (string, optional): User email
- `status` (string, optional, default `active`): `active`, `completed`, `cancelled`
- `wifi_voucher` (boolean, required): Whether to create a WiFi voucher
- `voucher_name` (string, optional): WiFi voucher name
- `seating_capacity` (number, optional): Max WiFi connections
- `voucher_max_speed_kbps` (number, optional): WiFi speed cap
- `site_id` (string, optional): UniFi site ID (required when `wifi_voucher` is `true` if Raspi needs it)
- `lock_pin` (string, required): Lock PIN for door access. **Currently MRD always sends this field but may send an empty string when it expects Raspi to generate its own PIN.**

**Example MRD → Raspi request**:

```json
{
  "id": "booking-uuid-123",
  "door_id": "76addcde-34c0-4053-902f-ace2c36b1912",
  "booking_start_time": "2026-01-27T09:45:00.000Z",
  "booking_end_time": "2026-01-27T12:15:00.000Z",
  "user_id": "Jane Doe",
  "user_email": "jane@example.com",
  "status": "active",
  "wifi_voucher": false,
  "voucher_name": "Meeting-WiFi",
  "seating_capacity": 4,
  "voucher_max_speed_kbps": 10240
}
```

**Raspi → MRD success response (new booking)**:

```json
{
  "ok": true,
  "booking": {
    "id": "booking-uuid-123",
    "door_id": "76addcde-34c0-4053-902f-ace2c36b1912",
    "lock_pin": "1234",
    "booking_start_time": "2026-01-27T09:45:00.000Z",
    "booking_end_time": "2026-01-27T12:15:00.000Z",
    "user_id": "Jane Doe",
    "user_email": "jane@example.com",
    "status": "active",
    "created_at": "2026-01-27T09:00:00.000Z",
    "updated_at": "2026-01-27T09:00:00.000Z"
  },
  "created": true,
  "visitor_id": "unifi-visitor-id-abc",
  "voucher_code": "ABCD-1234-EFGH",
  "voucher_id": "64a1b2c3d4e5f6789012345",
  "voucher_name": "Meeting-WiFi",
  "message": "Booking created successfully"
}
```

**MRD → ZenSpace Cloud response**:

MRD maps the Raspi response into the format ZenSpace expects:

- `access_code` (string): PIN / access code from the lock (from `booking.lock_pin`); empty string if Raspi hasn't generated one yet
- `valid_from` (ISO 8601): When the code becomes active (from `booking.booking_start_time`, or `start_time` as fallback)
- `valid_until` (ISO 8601): When the code expires (from `booking.booking_end_time`, or `end_time` as fallback)
- `html_template` (string, HTML): HTML snippet for the booking confirmation email

```json
{
  "access_code": "1234",
  "valid_from": "2026-01-27T10:00:00.000Z",
  "valid_until": "2026-01-27T12:00:00.000Z",
  "html_template": "<p>Your unlock PIN is <strong>1234</strong>. Valid from 10:00 AM to 12:00 PM.</p>"
}
```

> **What ZenSpace does with this response:**
> 1. Stores `access_code`, `valid_from`, `valid_until`, `html_template` in `booking.metadata.lock_access`
> 2. Injects `html_template` into the **booking confirmation email** sent to the guest
> 3. If MRD returns a non-2xx status, the email is still sent with a fallback message

**Error response** (4xx / 5xx):

```json
{
  "error": "Device not found for this meeting space",
  "message": "No lock device registered for the provided meeting_space_id"
}
```

---

## 2. Cancel Lock Booking

### 2.1 ZenSpace Cloud → MRD

- **Endpoint (MRD)**: `PUT /api/v1/lock/booking/cancel`
- **Purpose**: Cancel a booking and revoke lock access.

**Request body** (ZenSpace → MRD):

- `booking_id` (string, required)
- `meeting_space_id` (string, required)

**Example**:

```json
{
  "booking_id": "booking-uuid-123",
  "meeting_space_id": "zs-space-456"
}
```

**MRD behavior (conceptual)**:

1. Resolve primary LOCK device for `meeting_space_id`.
2. Resolve related gateway and its `baseUrl`.
3. Forward cancellation to Raspi via its booking cancel API.

---

### 2.2 MRD → Raspi/Gateway

- **Endpoint (Raspi)**: `DELETE {baseUrl}/api/v1/bookings/{id}/cancel`

**Path parameter**:

- `id`: booking ID (e.g. `"booking-uuid-123"`)

**Request**: no body required.

**Raspi → MRD success response**:

```json
{
  "ok": true,
  "booking": {
    "id": "booking-uuid-123",
    "door_id": "76addcde-34c0-4053-902f-ace2c36b1912",
    "status": "cancelled"
  },
  "message": "Booking cancelled successfully"
}
```

**MRD → ZenSpace Cloud response**:

```json
{
  "success": true,
  "message": "Booking access revoked"
}
```

On failure:

```json
{
  "success": false,
  "message": "Failed to cancel booking on gateway"
}
```

> **Best-effort:** If cancellation fails on MRD side, ZenSpace logs the error and continues with the booking cancellation.

---

## 3. Manual Lock / Unlock Action

### 3.1 ZenSpace Cloud → MRD

- **Endpoint (MRD)**: `POST /api/v1/lock/action`
- **Purpose**: Manually lock or unlock locks attached to a meeting space.

**Request body** (ZenSpace → MRD):

- `meeting_space_id` (UUID, required)
- `action` (string, required): `"lock"` or `"unlock"`
- `performed_by` (object, optional): Who performed the action
  - `user_id` (UUID): Admin user ID
  - `user_type` (string): Always `"admin"` for now
  - `user_name` (string): Admin user's display name
- `reason` (string, optional): Reason for the action

**Example**:

```json
{
  "meeting_space_id": "zs-space-456",
  "action": "unlock",
  "performed_by": {
    "user_id": "admin-user-uuid",
    "user_type": "admin",
    "user_name": "Jane Smith"
  },
  "reason": "Guest locked out, manual override"
}
```

**MRD behavior (conceptual)**:

1. Find primary LOCK device where `zenspace_meeting_space_id = meeting_space_id`.
2. Get its gateway and `baseUrl`.
3. Use `external_id` as `door_id` at the gateway.
4. Call Raspi lock/unlock API.
5. Return result to ZenSpace Cloud.

---

### 3.2 MRD → Raspi/Gateway

For each LOCK device:

- **Lock endpoint (Raspi)**: `POST {baseUrl}/api/v1/devices/doors/{door_id}/lock`
- **Unlock endpoint (Raspi)**: `POST {baseUrl}/api/v1/devices/doors/{door_id}/unlock`

**Path parameter**:

- `door_id`: external UniFi door id (`iot_devices.external_id`)

#### Lock Request

No body required (or an empty JSON object).

#### Unlock Request

**Optional request body** (override unlock duration per call):

```json
{
  "unlock_window_ms": 8000
}
```

If `unlock_window_ms` is **not** provided in the request body, the Pi uses:

1. Door-specific setting (`GET /api/v1/devices/doors/:id/unlock-window`)
2. Default setting (if configured)
3. UniFi defaults (no override)

#### Raspi Unlock Response

```json
{
  "ok": true,
  "door_id": "76addcde-34c0-4053-902f-ace2c36b1912",
  "unlocked": true,
  "unlock_window_ms": 8000,
  "unlock_window_seconds": 8,
  "result": {
    "success": true,
    "door_state": "unlocked"
  }
}
```

> **Note:** `unlock_window_seconds` is derived from `unlock_window_ms` using `ceil(ms / 1000)` because UniFi typically accepts seconds. Use `GET /api/v1/devices/doors/:id/status` after unlock to see current `lock_status` and `door_position`.

#### Raspi Lock Response

```json
{
  "ok": true,
  "door_id": "76addcde-34c0-4053-902f-ace2c36b1912",
  "success": true,
  "door_state": "locked"
}
```

**MRD → ZenSpace Cloud response**:

```json
{
  "success": true,
  "action": "unlock",
  "message": "Lock unlocked successfully",
  "device_status": "unlocked"
}
```

On failure:

```json
{
  "success": false,
  "action": "unlock",
  "message": "UNLOCK capability not enabled for this device",
  "device_status": null
}
```

> **Note:** Unlike booking access APIs, lock action errors **are propagated** to the admin caller so they know the action failed.

---

## 4. Enable Locks for a Meeting Space

### 4.1 ZenSpace Cloud → MRD

- **Endpoint (MRD)**: `PUT /api/v1/lock/enabled`
- **Purpose**: Mark locks as enabled (available) for a given meeting space.

**Request body**:

- `meeting_space_id` (string, required)

**Example**:

```json
{
  "meeting_space_id": "zs-space-456"
}
```

**Current MRD behavior (conceptual)**:

- Update `status` of all LOCK devices in MRD DB to `online` where `zenspace_meeting_space_id = meeting_space_id`.

### 4.2 MRD → Raspi/Gateway (future extension)

Today, there is no mandatory Raspi call.  
In a future extension, MRD could call:

- `PUT {baseUrl}/api/v1/devices/doors/{door_id}/enabled` with body:

```json
{
  "enabled": true
}
```

**MRD → ZenSpace Cloud response**:

```json
{
  "success": true,
  "message": "Locks enabled",
  "devices": [
    { "id": "device-uuid", "name": "Main Entrance Lock", "status": "online" }
  ]
}
```

---

## 5. Disable Locks for a Meeting Space

### 5.1 ZenSpace Cloud → MRD

- **Endpoint (MRD)**: `PUT /api/v1/lock/disabled`
- **Purpose**: Mark locks as disabled (not available) for a given meeting space.

**Request body**:

- `meeting_space_id` (string, required)

**Example**:

```json
{
  "meeting_space_id": "zs-space-456"
}
```

**Current MRD behavior (conceptual)**:

- Update `status` of all LOCK devices in MRD DB to `disabled` where `zenspace_meeting_space_id = meeting_space_id`.

### 5.2 MRD → Raspi/Gateway (future extension)

As with enable, MRD could later call:

- `PUT {baseUrl}/api/v1/devices/doors/{door_id}/enabled` with:

```json
{
  "enabled": false
}
```

**MRD → ZenSpace Cloud response**:

```json
{
  "success": true,
  "message": "Locks disabled",
  "devices": [
    { "id": "device-uuid", "name": "Main Entrance Lock", "status": "disabled" }
  ]
}
```

---

## 6. Unlock Window Handling and Disabled Locks

### 6.1 Raspi/Gateway Unlock Window API

- **Endpoint (Raspi)**: `POST {baseUrl}/api/v1/devices/doors/:id/unlock-window`
- **Purpose**: Configure the **unlock window** (how long the door stays unlocked) for a specific door.

**Path parameter**:

- `id`: door device ID (UniFi door id, same as `door_id` / `external_id`)

**Request body**:

- `unlock_window_ms` (number, required): Unlock window duration in milliseconds  
  - Must be `0` or `>= 1000`
  - `0` means **unlock is effectively disabled** at the gateway for this door

**Example request (set 8-second unlock window)**:

```json
{
  "unlock_window_ms": 8000
}
```

**Example request (disable unlock for this door)**:

```json
{
  "unlock_window_ms": 0
}
```

**Example response**:

```json
{
  "ok": true,
  "door_id": "76addcde-34c0-4053-902f-ace2c36b1912",
  "min_ms": 1000,
  "unlock_window_ms": 8000,
  "source": "door"
}
```

### 6.2 When MRD Calls `unlock-window`

MRD should call the Raspi `unlock-window` API in at least these cases:

1. **Lock explicitly disabled in MRD**
   - When a LOCK device is marked as disabled via MRD logic (for example, `PUT /api/v1/lock/disabled` setting `status = 'disabled'` for that device),  
     MRD should call:
     - `POST {baseUrl}/api/v1/devices/doors/{door_id}/unlock-window` with:
       - `unlock_window_ms: 0`
   - This ensures the **gateway level** behavior also treats the door as not unlockable.

2. **UNLOCK capability duration set to 0**
   - When a LOCK device’s capabilities in MRD include:
     - `capability = "UNLOCK"`
     - `enabled = true`
     - `params.duration = 0`
   - MRD should interpret this as **“unlock disabled”** for that door and:
     - Call `POST {baseUrl}/api/v1/devices/doors/{door_id}/unlock-window` with:
       - `unlock_window_ms: 0`
   - This ties the **capability configuration** (`UNLOCK` duration) to the actual gateway behavior.

In both cases, MRD continues to maintain its own `status`/capabilities in `iot_devices`, but also keeps the **gateway’s unlock window** in sync by using `unlock_window_ms = 0` when a lock is logically disabled.

---

## 7. Summary Matrix

### 7.1 ZenSpace Cloud → MRD APIs

- **`POST /api/v1/lock/booking`**  
  - Create or update booking and provision lock (and optional WiFi) access.

- **`PUT /api/v1/lock/booking/cancel`**  
  - Cancel booking and revoke lock access.

- **`POST /api/v1/lock/action`**  
  - Manual lock/unlock for all locks in a meeting space.

- **`PUT /api/v1/lock/enabled`**  
  - Mark locks in a meeting space as enabled.

- **`PUT /api/v1/lock/disabled`**  
  - Mark locks in a meeting space as disabled.

### 7.2 MRD → Raspi/Gateway APIs

- **`POST {baseUrl}/api/v1/bookings`**  
  - Create or update booking + visitor + optional WiFi voucher at gateway.

- **`DELETE {baseUrl}/api/v1/bookings/{id}/cancel`**  
  - Cancel booking and revoke visitor access at gateway.

- **`POST {baseUrl}/api/v1/devices/doors/{door_id}/lock`**  
  - Lock specific door.

- **`POST {baseUrl}/api/v1/devices/doors/{door_id}/unlock`**  
  - Unlock specific door.

- **(Future)** `PUT {baseUrl}/api/v1/devices/doors/{door_id}/enabled`  
  - Enable/disable a door device at gateway level.

- **`POST {baseUrl}/api/v1/devices/doors/{door_id}/unlock-window`**  
  - Set unlock window duration for a door in milliseconds (`0` = unlock disabled, otherwise `>= 1000`).

---

**Last Updated**: 2026-02-12

