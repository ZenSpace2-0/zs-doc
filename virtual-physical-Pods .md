## Virtual Pods with External MRD Physical Pods – Design Notes

### 1. High‑Level Idea (What You Described)

- **MRD owns physical pods**
  - The **system of record for physical rooms/pods is MRD**, not `zs_cloud`.
  - All **physical details** live in MRD:
    - Devices (locks, Wi‑Fi, sensors, etc.).
    - Physical attributes (location, capacity, hardware capabilities).
    - Any low‑level device / access control configuration.

- **ZS Cloud owns “our pods” (logical products)**
  - In `zs_cloud`, we keep our existing concept of **meeting spaces / pods** as **logical / commercial entities**:
    - What the customer sees on the storefront.
    - Pricing, merchandising, naming, imagery, cancellation policy, etc.
  - These “our pods” **do not directly own physical devices**; instead, they are **mapped to MRD physical pods**.

- **Mapping model**
  - We maintain a **mapping from ZS pods (our `meeting_spaces`) to MRD physical pods**.
  - One MRD physical pod can be **mapped to multiple ZS pods**, as long as we can avoid overlapping bookings.
  - Availability and assignment are computed so that:
    - We **never double‑book** a MRD physical pod.
    - We can still expose different ZS pods (SKUs) that share the same real room.

- **Booking flow**
  - When a customer books “our pod” (ZS pod):
    - We **route the booking to MRD** against the chosen MRD physical pod.
    - MRD continues to manage devices and low‑level access.
    - ZS keeps a record that this booking corresponds to:
      - Our logical pod (ZS meeting space / product).
      - A specific MRD physical pod for the given time window.

### 2. Conceptual Model

- **MRD Physical Pod**
  - Source of truth for:
    - Real‑world room identity.
    - Devices / access control.
    - Physical‑level availability and occupancy.
  - Identified by an **MRD pod ID** (external reference stored in ZS).

- **ZS Cloud Pod (ZS Meeting Space)**
  - Logical product / SKU managed in `zs_cloud`.
  - Has:
    - Name, slug, description, images.
    - Pricing and commercial rules.
    - Organizational / group associations (what org/location sees it).
  - Does **not** directly hold devices; it only references MRD physical pods.

- **ZS↔MRD Mapping**
  - A **link between a ZS pod and one or more MRD physical pods**.
  - Constraints on the mapping:
    - One MRD physical pod can serve **multiple ZS pods**.
    - For any given time window, **only one booking can occupy a specific MRD physical pod**, regardless of which ZS pod was used.

### 3. Availability & Overlap Rules

- **Core rule you mentioned**
  - We can map **one MRD physical pod to multiple ZS pods** **as long as**:
    - Their **booked time windows do not overlap** on that MRD pod.
  - In other words:
    - ZS can offer many “virtual SKUs” pointing at the same MRD room.
    - But when we actually book, we must ensure **MRD availability** for that physical pod at the requested start/end time.

- **How we check availability**
  - Step 1: User selects a **ZS pod** and a **time window**.
  - Step 2: ZS checks which **MRD physical pods** are mapped to that ZS pod.
  - Step 3: For each mapped MRD pod:
    - Ask MRD (or local cached availability) whether that physical pod is free for the requested window.
  - Step 4:
    - If at least one MRD physical pod is free:
      - The ZS pod is considered **available**.
      - We pick one MRD pod and proceed with booking.
    - If none are free:
      - The ZS pod is **unavailable** for that time.

### 4. Booking Flow (End‑to‑End)

- **1. Customer selects a ZS pod**
  - Frontend only knows about **ZS pods** (our meeting spaces).
  - User chooses a pod, date, and start/end time.

- **2. ZS resolves to MRD physical pod**
  - Given the ZS pod:
    - Look up all **mapped MRD physical pods**.
    - Filter to those **physically available** in MRD for that time.
    - Choose one according to a strategy (first‑fit, least‑used, etc.).

- **3. Create booking**
  - In ZS:
    - We create a booking record tied to the **ZS pod**.
    - We also store the **MRD physical pod ID** and time window.
  - In MRD:
    - We either:
      - Create a booking directly via MRD’s API, or
      - Reserve the time slot via MRD’s availability/booking interface.
    - MRD enforces **no overlap** on its physical pod and handles devices.

- **4. Post‑booking behavior**
  - All device / access / lock behavior is still driven by MRD using its own rules.
  - ZS uses the MRD booking reference to:
    - Show status in dashboards.
    - Display correct physical room info in emails / UI.

### 5. What I’m Confident I Understand from Your Message

- **Clear points**
  - You want to **keep the existing “virtual vs physical” doc**, but add a **second variant** where:
    - **MRD is the only place where physical pods/devices live.**
    - ZS pods are purely logical and mapped to MRD pods.
  - It must be possible to:
    - Map **one MRD physical pod to multiple ZS pods**.
    - Avoid overlapping bookings on that MRD pod across all those ZS pods.
  - When a user books a ZS pod:
    - The actual **booking and enforcement happens at MRD** on the physical pod.
    - ZS tracks which of our pods was booked and which MRD physical pod was used.

- **Assumptions that might need your confirmation**
  - **Availability source**:
    - I assumed MRD exposes an API for availability / booking which we will call from ZS.
  - **Data ownership**:
    - I assumed **all device configuration and low‑level access policies stay in MRD**, and ZS won’t mirror device tables locally.
  - **Conflict resolution**:
    - I assumed MRD is the **single point of truth** for detecting and preventing double bookings on a physical pod.

If this matches what you had in mind, next we can turn this into a more formal spec (schema changes, APIs, and flows) and align it with the existing `VIRTUAL_MEETING_SPACES_DESIGN.md` so both variants are clearly documented.
