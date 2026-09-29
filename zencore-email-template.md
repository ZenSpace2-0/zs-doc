# Email Branding — User Guide & Test Plan

Last updated: 2026-09-29

- [Overview](#overview)
- [Part 1 — User guide](#part-1--user-guide)
- [Part 2 — Tester guide](#part-2--tester-guide)

---

## Overview

Email branding lets organization admins change how booking emails look. It works at two levels: the whole **organization**, and each **space group**. With nothing set, emails look exactly as they do today.

It applies to these emails: confirmation, cancellation, moved, refund, attendee and access.

Every field is looked up in this order. The first level that has a value wins.

1. **Space group** — set on the group's Email branding page.
2. **Organization** — set on Settings → Email Branding.
3. **Built-in default** — what emails use today.

An empty field always means "inherit from the level above". Clear a field on a group and it falls back to the organization's value. Clear it on the organization and it falls back to the default.

---

## Part 1 — User guide

### Who can use it

Only organization admins (users with the `update:organizations` permission) and super-admins. Managers and other roles don't see the Email Branding tile or button, and the server rejects their changes.

### Where to find it

| Level | How to get there | What you change |
| --- | --- | --- |
| Organization | **Settings** → **Email Branding** tile | The look for every group in the organization |
| Space group | **Groups** → open a group → **Email branding** button in the header, next to **Edit** | Overrides for that group only |

### Reading the screen

The form is on the left and a live preview is on the right. Each field has a small badge that says where its value comes from:

| Badge | Meaning |
| --- | --- |
| **Default** | Nothing is set, so the built-in default is used. |
| **From organization** | Group screen only. The group uses the organization's value. |
| **Set on organization** | Organization screen. The organization has its own value. |
| **Set on this group** | Group screen. The group overrides the organization. |

- An empty field shows the inherited value in grey as its placeholder.
- Click **Reset** beside a field to clear it and inherit again.
- On/off settings have three choices: **Inherit / On / Off** on the group screen, and **Default / On / Off** on the organization screen. When Inherit or Default is picked, the text beside it says what the setting resolves to, for example "Resolves to on".

### The sections

| Section | Fields | Notes |
| --- | --- | --- |
| **Branding** | Logo, Header colour, Accent strip, Buttons and links, Host label, Host name | Logo: PNG or JPG, about 200 KB, shown 32 px high. Colours: use the picker or type a hex value like `#1d4ed8`. Host name defaults to the organization name. |
| **Wording** | Confirmation headline, Space word | The headline is used in the confirmation email only. The space word (for example Pod, Room, Desk) also sets the default row labels. |
| **Buttons** | Cancel booking button, Make another booking button | Turn each button on or off. |
| **Booking details** | Space, When, Event and Location rows | Show or hide each row, and rename its label. The Location row defaults to "Find the {space word}". |
| **Notes** | Arrival note (on/off + text), Extra note (on/off + text) | Plain text. Line breaks are kept. A group can turn the extra note off to hide the organization's note. |
| **Contact & footer** | Support email, Footer tagline (on/off), Tagline text | Support email defaults to `support@zenspace.io`. |

### Built-in defaults

| Field | Default |
| --- | --- |
| Header colour | `#0b1437` |
| Accent strip | `#22c55e` |
| Buttons and links | `#1d4ed8` |
| Host label | Hosted at |
| Host name | The organization name |
| Confirmation headline | Your booking is confirmed |
| Space word | Space |
| Row labels | {space word} · When · Event · Find the {space word, lowercase} |
| Arrival note | Please arrive a few minutes before {start_time} {timezone} so you can start on time. |
| Extra note | None |
| Support email | support@zenspace.io |
| Tagline | Calm. Precise. Empowering. |
| All buttons, rows and notes | Shown |

### Placeholders in notes

Notes can include the placeholders below, which are filled in for each booking. Click a chip under the text box to insert one where the cursor is. Any other `{word}` is rejected.

| Placeholder | Filled with |
| --- | --- |
| `{space_name}` | The booked space's name |
| `{space_word}` | Your space word, as typed |
| `{space_word_lower}` | Your space word, in lowercase |
| `{date}` | Booking date |
| `{start_time}` | Start time |
| `{end_time}` | End time |
| `{timezone}` | The space's timezone |
| `{organization_name}` | Organization name |
| `{group_name}` | Space group name |

### Preview

The preview updates as you type and uses sample booking data. Switch between **Confirmation** and **Cancellation** at the top of the preview. It's approximate: the real email may differ slightly in spacing and fonts.

### Saving and resetting

1. Make your changes. "unsaved changes" appears beside the buttons.
2. Click **Save changes**. Any field with an error is highlighted in red and must be fixed first.
3. To start over, click **Reset all** and confirm.
   - On a group, this removes every group override, so the group follows the organization again.
   - On the organization, this returns every organization field to its default. Group overrides are kept.

If you try to leave or reload the page with unsaved changes, the browser asks you to confirm.

### What you can't customise

WiFi details, payment amounts and the calendar invite are set by the system.

---

## Part 2 — Tester guide

### Setup

**Accounts**

| Account | Permission | Used for |
| --- | --- | --- |
| Org admin | Has `update:organizations` | Most tests |
| Manager | No `update:organizations` | Permission tests |
| Super-admin | Super-admin flag | Cross-org test |

**Data**

- One organization with two space groups, **Group A** and **Group B**, and at least one bookable space in each.
- An inbox you can read, to receive booking emails.
- Test files: a PNG under 200 KB, a JPG under 200 KB, a PNG over 200 KB, and a GIF or SVG.
- Optional: Postman or curl with an admin token and a manager token, for the API tests.

**Start clean:** click **Reset all** on the organization and on both groups before starting.

### Test cases

Record each result as Pass / Fail / Blocked, with a note or screenshot for every failure.

#### Permissions

| ID | Steps | Expected | Result |
| --- | --- | --- | --- |
| P1 | Log in as org admin. Open **Settings**, then open **Group A**. | The **Email Branding** tile is on Settings. The **Email branding** button is in the group header. | |
| P2 | Log in as manager. Open **Settings** and **Group A**. | No tile and no button. | |
| P3 | As manager, open `/settings/email-branding` and `/groups/<group-slug>/email-branding` directly. | A "not authorized" page appears. No form. | |
| P4 | Using the manager token, send `PUT /space-groups/<id>/email-branding` with `{"header_color":"#ff0000"}`. | 403. Nothing changes. | |
| P5 | Log in as super-admin. Switch to another organization and save one field. | Saves successfully. | |

#### Inheritance

| ID | Steps | Expected | Result |
| --- | --- | --- | --- |
| I1 | On a clean organization, open the org Email Branding page. | Every badge says **Default**. Placeholders match the defaults table in Part 1. Host name shows the organization name. | |
| I2 | On the org page, set Header colour `#123456` and Support email `help@test.com`. Save. Open Group A's page. | Both fields show **From organization**, with the org values as grey placeholders. | |
| I3 | On Group A, set Header colour `#aa0000`. Save. | Group A's badge is **Set on this group** and its preview header is red. Group B still shows `#123456` **From organization**. | |
| I4 | On Group A, click **Reset** on Header colour. | The field empties at once. The badge changes to **From organization** and the placeholder shows `#123456`. After Save and a reload, the change is still there. | |
| I5 | On the org page, set Space word `Pod`. Save. | Row label placeholders read "Pod" and "Find the pod", on the org page and on the group pages. The preview matches. | |
| I6 | On the org page, set Cancel booking button to **Off**. Save. Open Group A. | Group A shows **Inherit**, "Resolves to off". Setting **On** on Group A shows the button in Group A's preview. | |
| I7 | On the org page, set an Extra note. Save. On Group A, set Extra note to **Off**. Save. | Group A's preview and emails have no extra note. Group B still shows it. | |
| I8 | On the org page, hide the Event row. On Group A, set it to **Show** and rename it "Conference". | Group A shows a "Conference" row. Group B has no Event row. | |

#### Validation

| ID | Steps | Expected | Result |
| --- | --- | --- | --- |
| V1 | Type `blue`, then `#12`, into Header colour. | The error "Use a hex colour like #1d4ed8." appears. **Save changes** is disabled. | |
| V2 | Type `#abc`. Then use the colour picker. | `#abc` is accepted. The picker fills the hex box. | |
| V3 | Paste `http://example.com/logo.png` into the logo URL. | The error "Logo must be an https URL." appears. | |
| V4 | Type `not-an-email` into Support email. | The error "Enter a valid email address." appears. | |
| V5 | Try to type more than the limit into each text field: headline 80, space word 30, host label 40, host name 100, row label 40, tagline 80, arrival note 500, extra note 1000. | Input stops at the limit. The counter shows, for example, `80/80`. | |
| V6 | Type `Hi {first_name}` into the Arrival note text. | An error names `{first_name}`. Save is disabled. | |
| V7 | Place the cursor mid-sentence in a note and click the `{date}` chip. | `{date}` is inserted at the cursor, and the cursor lands after it. | |
| V8 | With the admin token, send a PUT with an unknown key, for example `{"foo":"bar"}`. | 400 with a readable message. The same error made from the UI would appear as a toast or next to the field. | |

#### Logo upload

| ID | Steps | Expected | Result |
| --- | --- | --- | --- |
| U1 | Click **Upload** and choose the PNG under 200 KB. | The URL field fills with an `https://` link. The logo shows in the tile and in the preview header. | |
| U2 | Upload the JPG under 200 KB. | Works the same as U1. | |
| U3 | Upload the PNG over 200 KB. | Rejected with "Logo is N KB — keep it to about 200 KB." | |
| U4 | Upload the GIF or SVG. | Rejected with "Logo must be a PNG or JPG." | |
| U5 | Click the bin icon next to **Replace**. | The logo is cleared, and the field shows the inherited logo or "No logo". | |

#### Preview

| ID | Steps | Expected | Result |
| --- | --- | --- | --- |
| PV1 | Change colours, headline, rows and notes one at a time. | The preview updates as you type. | |
| PV2 | Put every placeholder in the Arrival note. | The preview shows sample values (for example "10:00 AM PDT"), with no raw `{tokens}` left. | |
| PV3 | Switch the preview to **Cancellation**. | The headline changes. The Cancel button and the notes are hidden. The button reads "Book again". | |
| PV4 | Look for a door PIN block. | There is none. | |
| PV5 | Type an invalid colour. | The preview keeps the last valid inherited colour and doesn't turn black. | |

#### Save and reset

| ID | Steps | Expected | Result |
| --- | --- | --- | --- |
| S1 | Open a page with no changes. Then change one field. | **Save changes** is disabled until something changes. Then "unsaved changes" appears and Save is enabled. | |
| S2 | Save. | A success toast appears. The badges update. After a reload, the values are still there. | |
| S3 | Change a field and reload the page without saving. | The browser asks you to confirm leaving. | |
| S4 | On Group A, click **Reset all** and confirm. | Every field shows **Inherit** or **From organization**/**Default**. The org values are untouched. | |
| S5 | On the org page, click **Reset all** and confirm. Then run `GET /organizations/<id>/email-branding`. | Every badge is **Default**. The API returns every `sources` value as `"default"`. Group overrides still exist. | |
| S6 | Open a level that has never been saved. | **Reset all** is disabled. | |

#### Email delivery (end to end)

| ID | Steps | Expected | Result |
| --- | --- | --- | --- |
| E1 | Set org and Group A values across every section. Book a space in Group A. | The confirmation email shows the Group A overrides, with the org values everywhere else: logo, colours, host line, headline, rows, notes with real values, buttons, support email and tagline. | |
| E2 | Cancel that booking. | The cancellation email uses the same branding. | |
| E3 | Book a space in Group B. | The email uses only the org values. | |
| E4 | Trigger a moved, refund, attendee and access email. | Each one uses the branding. | |
| E5 | Reset all on the org and both groups. Book again. | The email looks exactly as it did before this feature. | |
| E6 | Put a note with line breaks and check the email. | The line breaks are kept. | |

#### UI and edge cases

| ID | Steps | Expected | Result |
| --- | --- | --- | --- |
| X1 | Switch the admin app to dark mode. | The form is readable. The preview stays light, as emails are. | |
| X2 | Narrow the window to phone width. | The preview moves under the form. There is no sideways scroll. Badges and **Reset** links wrap instead of overlapping. | |
| X3 | Set a very long host name (100 characters). | It is truncated in the preview header, not overflowing. | |
| X4 | With unsaved changes, switch to another organization. | The page loads the new organization's values, not the old form. | |

### Reporting a bug

Include the test ID, which account you used, which page (organization or group, and the group name), the steps, what you expected and what happened, and a screenshot. For email bugs, attach the received email or its HTML source.
