# Image specifications

This document is the single source of truth for image requirements in Zen Admin forms for:

- Meeting spaces (thumbnail, gallery images, floor image)
- Groups (banner image, header logo, footer logo)

Validation behavior below refers to current client-side checks in create/edit forms.

---

## Shared limits

| Rule | Value |
|------|--------|
| Max file size | **5 MB** per file |
| Typical formats (UI copy) | **PNG**, **JPG** (uploaders generally use `accept="image/*"`) |

---

## Meeting space images

**Source of truth:**  
`src/app/(modules)/(authenticated-modules)/(dashboard)/meeting-spaces/components/meeting-space-create-form.tsx`  
`MEETING_SPACE_GALLERY_IMAGE_MIN_SIDE`, `MEETING_SPACE_GALLERY_IMAGE_RECOMMENDED_SIDE`, `validateMeetingSpaceGalleryImageFile`

### Thumbnail image

- **Aspect:** **1:1** (square), cropped by uploader (`ThumbnailUpload` with `aspect={1}`)
- **Recommended size:** **1200 × 1200** px
- **Minimum suggested source:** **800 × 800** px
- **Purpose:** Primary visual preview in square contexts

### Gallery images (Images)

| Rule | Value |
|------|--------|
| Aspect ratio | **Square (1:1)** |
| Minimum | **800 × 800** px (each side >= 800) |
| Recommended | **1200 × 1200** px |
| Max count | **10** images |

**Validation:** New file uploads are checked in the browser. Non-square files or files below minimum edge length are rejected with inline error + toast. Existing URL values loaded in edit mode are not dimension-revalidated by this client-side check.

### Floor image / floor plan

- **Aspect:** No fixed ratio
- **Max size:** **5 MB**
- **Purpose:** Optional floor plan or diagram

---

## Group images

**Source of truth:**  
`src/app/(modules)/(authenticated-modules)/(dashboard)/groups/components/group-form.tsx`  
Banner: inline `1200 × 256` validation in banner uploader handler  
Logos: `GROUP_LOGO_IMAGE_MIN_SIDE`, `GROUP_LOGO_IMAGE_MAX_SIDE`, `GROUP_LOGO_IMAGE_RECOMMENDED_SIDE`, `validateGroupLogoImageFile`

### Banner image

| Rule | Value |
|------|--------|
| Exact dimensions | **1200 × 256** px (required) |
| Max count | **3** images |

**Validation:** Each new uploaded banner image must match exact dimensions.

### Header logo and footer logo

| Rule | Value |
|------|--------|
| Aspect ratio | **Square (1:1)** |
| Allowed size range | **256–512** px per side (inclusive) |
| Recommended | **512 × 512** px |

**Validation:** New file uploads are validated in browser. Invalid files are rejected (form error + toast), and uploader is reset so invalid preview does not stick.

---

## Keeping this doc in sync

Update this document whenever image constants, uploader limits, validation behavior, or form copy changes in:

- `meeting-space-create-form.tsx`
- `group-form.tsx`
