# MRD / External Lock Provider – API Reference

> **Last updated:** 2026-02-12
>
> This document describes every HTTP request exchanged between the **ZenSpace Backend** and the **MRD (or any external lock provider)**. There are two directions of communication:
>
> | Direction | Description |
> |---|---|
> | **MRD → ZenSpace** | Device registration / deregistration |
> | **ZenSpace → MRD** | Booking access, cancellation, manual lock/unlock |

---

## Table of Contents

1. [Authentication](#1-authentication)
2. [MRD → ZenSpace APIs](#2-mrd--zenspace-apis)
   - 2.1 [Register Device](#21-register-device)
   - 2.2 [Deregister Device](#22-deregister-device)
3. [ZenSpace → MRD APIs](#3-zenspace--mrd-apis)
   - 3.1 [Create Booking Access](#31-create-booking-access)
   - 3.2 [Cancel Booking Access](#32-cancel-booking-access)
   - 3.3 [Manual Lock/Unlock Action](#33-manual-lockunlock-action)
4. [Error Handling & Logging](#4-error-handling--logging)
5. [Flow Diagrams](#5-flow-diagrams)

---

## 1. Authentication & Headers

### MRD → ZenSpace (Device Registration / Deregistration)

These endpoints are hosted by ZenSpace. The caller (MRD or admin) must send:

| Header | Value | Required | Description |
|---|---|---|---|
| `Content-Type` | `application/json` | ✅ | Request body is always JSON |
| `x-api-key` | `<zenspace_api_key>` | ✅ (one of) | ZenSpace-issued API key **OR** |
| `Authorization` | `Bearer <jwt_token>` | ✅ (one of) | Admin JWT token |

Either `x-api-key` or `Authorization` is accepted. The endpoint validates that at least one is present.

### ZenSpace → MRD (Lock Operations) – Internal Implementation

These are outbound calls made by ZenSpace to MRD. The headers are set **internally** by `LockIntegrationService.request()` using Node.js native `http`/`https` module (no Axios). Every outbound call includes:

| Header | Value | Set By | Description |
|---|---|---|---|
| `Content-Type` | `application/json` | Auto | Body is always `JSON.stringify(payload)` |
| `Content-Length` | `<byte_length>` | Auto | Computed via `Buffer.byteLength(payload)` – ensures correct transfer encoding |
| `x-api-key` | `<device.api_key>` | Auto | Read from `meeting_space_devices.api_key` (provided by MRD during device registration) |

#### Internal HTTP client details

```
File: src/modules/meeting-space-devices/lock-integration.service.ts

- HTTP library:   Node.js native `http` / `https` (NOT Axios, NOT HttpService)
- Protocol:       Auto-detected from device.api_url (http:// or https://)
- Port:           Parsed from URL, defaults to 443 (https) or 80 (http)
- Body encoding:  JSON.stringify(body) → written via req.write(payload)
- Response parse: Attempts JSON.parse on response body; falls back to raw string
- Timeout:        No explicit timeout (uses Node.js defaults)
- Retry:          No automatic retry – failures are logged and handled per operation
```

> **Note:** MRD endpoints must accept `application/json` content type and the `x-api-key` header for authentication.

---

## 2. MRD → ZenSpace APIs

These are endpoints hosted by **ZenSpace** that the **MRD** (or admin) calls.

### 2.1 Register Device

Registers a lock device for a meeting space. Device is created in `pending_approval` state and requires admin approval before it is used for lock operations.

```
POST /api/v1/devices/register
```

#### Request Headers

```
Content-Type: application/json
x-api-key: <zenspace_api_key>
```

#### Request Body

```json
{
  "meeting_space_id": "276d70b2-8155-49ff-9c9a-ce4d564adad1",
  "device_type": "LOCK",
  "device_name": "Main Entrance Lock",
  "api_url": "https://mrd-api-dev.zenspaceiot.com",
  "api_key": "mrd_api_key_xxx"
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `meeting_space_id` | UUID | ✅ | The meeting space to attach this lock device to |
| `device_type` | string | ✅ | Device type (currently `"LOCK"`) |
| `device_name` | string | ✅ | Human-readable name (e.g. `"Main Entrance Lock"`) |
| `api_url` | URL | ✅ | MRD base URL. ZenSpace will append paths like `/api/v1/lock/booking` to this. **No trailing slash.** |
| `api_key` | string | ✅ | API key that ZenSpace will send as `x-api-key` when calling MRD APIs |

#### Response (201 Created)

```json
{
  "status": "success",
  "message": "Device registered. Awaiting admin approval.",
  "data": {
    "device_id": "d1a2b3c4-...",
    "meeting_space_id": "276d70b2-8155-49ff-9c9a-ce4d564adad1",
    "meeting_space_name": "Pod A",
    "organization_id": "org-uuid",
    "organization_name": "Acme Corp",
    "space_group_id": "group-uuid",
    "space_group_name": "Downtown Office",
    "device_type": "LOCK",
    "device_name": "Main Entrance Lock",
    "status": "pending_approval",
    "approved": false,
    "created_at": "2026-02-12T10:00:00.000Z"
  }
}
```

> **Note:** `organization_id` and `space_group_id` are auto-populated from the meeting space. Only one LOCK device is allowed per meeting space (unique constraint on `meeting_space_id + device_type`).

---

### 2.2 Deregister Device

Disables an active lock device for a meeting space. The device remains in the database but is marked as `disabled` and will no longer be used for lock operations.

```
POST /api/v1/devices/deregister
```

#### Request Headers

```
Content-Type: application/json
x-api-key: <zenspace_api_key>
```

#### Request Body

```json
{
  "meeting_space_id": "276d70b2-8155-49ff-9c9a-ce4d564adad1",
  "device_type": "LOCK"
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `meeting_space_id` | UUID | ✅ | The meeting space whose device should be disabled |
| `device_type` | string | ✅ | Device type to deregister (e.g. `"LOCK"`) |

#### Response (200 OK)

```json
{
  "status": "success",
  "message": "Device deregistered (disabled) successfully",
  "data": {
    "device_id": "d1a2b3c4-...",
    "meeting_space_id": "276d70b2-8155-49ff-9c9a-ce4d564adad1",
    "meeting_space_name": "Pod A",
    "organization_id": "org-uuid",
    "organization_name": "Acme Corp",
    "space_group_id": "group-uuid",
    "space_group_name": "Downtown Office",
    "device_type": "LOCK",
    "device_name": "Main Entrance Lock",
    "status": "disabled",
    "approved": false,
    "created_at": "2026-02-12T10:00:00.000Z"
  }
}
```

---

## 3. ZenSpace → MRD APIs

These are endpoints hosted by **MRD** that **ZenSpace** calls. The base URL comes from `device.api_url` (provided during registration).

> **URL construction:** `device.api_url` + path
>
> Example: If `api_url = "https://mrd-api-dev.zenspaceiot.com"`, then:
> - Booking access → `https://mrd-api-dev.zenspaceiot.com/api/v1/lock/booking`
> - Booking cancel → `https://mrd-api-dev.zenspaceiot.com/api/v1/lock/booking/cancel`
> - Lock action → `https://mrd-api-dev.zenspaceiot.com/api/v1/lock/action`

---

### 3.1 Create Booking Access

**Called automatically** when a booking becomes **paid** (via Stripe webhook or zero-amount booking). Creates a time-bound access code / PIN for the guest.

```
POST {device.api_url}/api/v1/lock/booking
```

#### Request Headers (sent by ZenSpace internally)

```
Content-Type: application/json
Content-Length: <computed_byte_length>
x-api-key: <device.api_key>
```

#### Request Body (sent by ZenSpace)

```json
{
  "booking_id": "b7e1f2a3-4c5d-6e7f-8a9b-0c1d2e3f4a5b",
  "meeting_space_id": "276d70b2-8155-49ff-9c9a-ce4d564adad1",
  "start_time": "2026-02-12T11:30:00.000Z",
  "end_time": "2026-02-12T12:00:00.000Z",
  "guest": {
    "name": "John Doe",
    "email": "john@example.com"
  },
  "booking_details": {
    "title": "Team Standup",
    "space_name": "Pod A"
  }
}
```

| Field | Type | Description |
|---|---|---|
| `booking_id` | UUID | ZenSpace booking ID |
| `meeting_space_id` | UUID | Meeting space ID |
| `start_time` | ISO 8601 | Booking start time (UTC) |
| `end_time` | ISO 8601 | Booking end time (UTC) |
| `guest.name` | string \| null | Organizer / guest name |
| `guest.email` | string | Organizer / guest email |
| `booking_details.title` | string | Booking title |
| `booking_details.space_name` | string \| undefined | Meeting space name (may be undefined if relation not loaded) |

#### Expected Response (200 OK) from MRD

```json
{
  "access_code": "1234",
  "valid_from": "2026-02-12T11:25:00.000Z",
  "valid_until": "2026-02-12T12:05:00.000Z",
  "html_template": "<p>Your unlock PIN is <strong>1234</strong>. Valid from 11:25 AM to 12:05 PM.</p>"
}
```

| Field | Type | Description |
|---|---|---|
| `access_code` | string | PIN / access code for the lock |
| `valid_from` | ISO 8601 | When the code becomes active (can be earlier than booking start for early entry) |
| `valid_until` | ISO 8601 | When the code expires (can be later than booking end for buffer) |
| `html_template` | string (HTML) | HTML snippet injected into the booking confirmation email under "Access Instructions" |

> **What ZenSpace does with the response:**
> 1. Stores `access_code`, `valid_from`, `valid_until`, `html_template` in `booking.metadata.lock_access`
> 2. Injects `html_template` into the **booking confirmation email** sent to the guest
> 3. If the MRD API returns a non-2xx status, the email is still sent with a fallback message: *"Currently your unlock PIN is not generated, but we will notify you once it's generated."*

#### Error Response (4xx / 5xx)

If MRD returns a non-2xx status:

```json
{
  "error": "Device not found for this meeting space",
  "message": "No lock device registered for the provided meeting_space_id"
}
```

> ZenSpace logs the error in `device_api_logs` (operation: `lock.booking.create`) and continues with the email flow.

---

### 3.2 Cancel Booking Access

**Called automatically** when a paid booking is **cancelled**. Revokes the previously issued access code.

```
PUT {device.api_url}/api/v1/lock/booking/cancel
```

#### Request Headers (sent by ZenSpace internally)

```
Content-Type: application/json
Content-Length: <computed_byte_length>
x-api-key: <device.api_key>
```

#### Request Body (sent by ZenSpace)

```json
{
  "booking_id": "b7e1f2a3-4c5d-6e7f-8a9b-0c1d2e3f4a5b",
  "meeting_space_id": "276d70b2-8155-49ff-9c9a-ce4d564adad1"
}
```

| Field | Type | Description |
|---|---|---|
| `booking_id` | UUID | ZenSpace booking ID (same as sent in create) |
| `meeting_space_id` | UUID | Meeting space ID |

#### Expected Response (200 OK) from MRD

```json
{
  "success": true,
  "message": "Booking access revoked"
}
```

> **Best-effort:** If cancellation fails on MRD side, ZenSpace logs the error and continues with the booking cancellation. The booking will still be cancelled in ZenSpace.

---

### 3.3 Manual Lock/Unlock Action

**Called by admin** from the ZenSpace dashboard. Sends a manual lock or unlock command to MRD.

```
POST {device.api_url}/api/v1/lock/action
```

#### Request Headers (sent by ZenSpace internally)

```
Content-Type: application/json
Content-Length: <computed_byte_length>
x-api-key: <device.api_key>
```

#### Request Body (sent by ZenSpace)

```json
{
  "meeting_space_id": "276d70b2-8155-49ff-9c9a-ce4d564adad1",
  "action": "unlock",
  "performed_by": {
    "user_id": "admin-user-uuid",
    "user_type": "admin",
    "user_name": "Jane Smith"
  },
  "reason": "Guest locked out, manual override"
}
```

| Field | Type | Description |
|---|---|---|
| `meeting_space_id` | UUID | Meeting space ID |
| `action` | `"lock"` \| `"unlock"` | Action to perform |
| `performed_by.user_id` | UUID | Admin user who triggered the action |
| `performed_by.user_type` | string | Always `"admin"` for now |
| `performed_by.user_name` | string | Admin user's display name |
| `reason` | string \| undefined | Optional reason for the action |

#### Expected Response (200 OK) from MRD

```json
{
  "success": true,
  "action": "unlock",
  "message": "Lock unlocked successfully",
  "device_status": "unlocked"
}
```

> **Note:** Unlike booking access APIs, lock action errors **are propagated** to the admin caller so they know the action failed.

---

## 4. Error Handling & Logging

### Logging

All ZenSpace → MRD API calls are logged in the `device_api_logs` table:

| Column | Description |
|---|---|
| `device_id` | Which device was used |
| `operation` | `lock.booking.create`, `lock.booking.cancel`, `lock.action` |
| `endpoint` | Full URL called |
| `method` | `POST` or `PUT` |
| `response_status` | HTTP status code from MRD |
| `response_time_ms` | Round-trip time in milliseconds |
| `error_message` | Error details (null on success) |
| `created_at` | Timestamp |

### Error Behavior by Operation

| Operation | On MRD Error | Booking/Email Impact |
|---|---|---|
| `lock.booking.create` | Logged, **not thrown** | Booking is confirmed. Email sent with fallback text: *"Currently your unlock PIN is not generated, but we will notify you once it's generated."* |
| `lock.booking.cancel` | Logged, **not thrown** | Booking cancellation proceeds normally |
| `lock.action` | Logged, **thrown to admin** | Admin sees error in API response |

---

## 5. Flow Diagrams

### Device Registration & Approval

```
MRD                           ZenSpace Backend                    Admin Dashboard
 │                                  │                                  │
 │  POST /api/v1/devices/register   │                                  │
 │ ───────────────────────────────►  │                                  │
 │                                  │  WebSocket: device_pending_approval
 │                                  │ ────────────────────────────────► │
 │  ◄─── 201 { status: pending }    │                                  │
 │                                  │                                  │
 │                                  │  PUT /api/v1/admin/devices/:id/approve
 │                                  │ ◄──────────────────────────────── │
 │                                  │                                  │
 │                                  │  WebSocket: device_approved       │
 │                                  │ ────────────────────────────────► │
 │                                  │  ─── 200 { status: active }  ──► │
```

### Booking → Lock Access Flow

```
Guest / Stripe               ZenSpace Backend                    MRD Lock Provider
 │                                  │                                  │
 │  Payment succeeds (webhook)      │                                  │
 │ ───────────────────────────────►  │                                  │
 │                                  │  POST /api/v1/lock/booking       │
 │                                  │ ────────────────────────────────► │
 │                                  │                                  │
 │                                  │  ◄── 200 { access_code, html }   │
 │                                  │                                  │
 │  ◄── Confirmation email          │                                  │
 │      (with lock HTML or          │                                  │
 │       fallback text)             │                                  │
```

### Booking Cancellation → Lock Revoke

```
Admin / User                  ZenSpace Backend                    MRD Lock Provider
 │                                  │                                  │
 │  Cancel booking                  │                                  │
 │ ───────────────────────────────►  │                                  │
 │                                  │  PUT /api/v1/lock/booking/cancel │
 │                                  │ ────────────────────────────────► │
 │                                  │                                  │
 │                                  │  ◄── 200 { success: true }       │
 │  ◄── Booking cancelled           │                                  │
```

### Manual Lock/Unlock

```
Admin Dashboard               ZenSpace Backend                    MRD Lock Provider
 │                                  │                                  │
 │  POST /api/v1/lock/action        │                                  │
 │  { action: "unlock" }            │                                  │
 │ ───────────────────────────────►  │                                  │
 │                                  │  POST /api/v1/lock/action        │
 │                                  │ ────────────────────────────────► │
 │                                  │                                  │
 │                                  │  ◄── 200 { success: true }       │
 │  ◄── 200 { status, response }    │                                  │
```

---

## Quick Reference: All Endpoints

| # | Direction | Method | Path | Auth | When Called |
|---|---|---|---|---|---|
| 1 | MRD → ZS | `POST` | `/api/v1/devices/register` | API key or JWT | MRD registers a new lock device |
| 2 | MRD → ZS | `POST` | `/api/v1/devices/deregister` | API key or JWT | MRD disables a lock device |
| 3 | ZS → MRD | `POST` | `/api/v1/lock/booking` | `x-api-key` | Booking becomes paid |
| 4 | ZS → MRD | `PUT` | `/api/v1/lock/booking/cancel` | `x-api-key` | Booking is cancelled |
| 5 | ZS → MRD | `POST` | `/api/v1/lock/action` | `x-api-key` | Admin triggers manual lock/unlock |
| 6 | ZS (admin) | `GET` | `/api/v1/admin/devices` | JWT | Admin lists devices |
| 7 | ZS (admin) | `PUT` | `/api/v1/admin/devices/:id/approve` | JWT | Admin approves a device |
