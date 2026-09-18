# The Calendar Adapter — what it does, and how to test it

This is one document with two halves. The first explains what the product is and how it
behaves, for anyone who needs to understand it without reading code. The second is a test
plan built on that explanation.

If you are testing, **read Part 1 first**. Most of this product's behaviour is deliberate
restraint — it refuses to do things it could easily do — and without knowing which refusals
are designed, you will file them as bugs. There is also a list of things that are genuinely
broken right now, in Part 5. Read that before filing anything.

Other documents: [README.md](README.md) is for developers, [DEPLOYMENT.md](DEPLOYMENT.md) is
for whoever runs the servers.

---

# Part 1 — What this is

## The problem

A company already has meeting rooms in Microsoft 365 or Google Workspace. Their staff book
those rooms the way they always have: from Outlook, from Google Calendar, from the invite
they were already writing.

That company also has ZenSpace pods, and ZenSpace has its own bookings, its own displays on
the wall, its own check-in.

Without this adapter those are two separate worlds. Someone books "Board room 2" in Outlook,
walks up to it, and the ZenSpace display says the room is free — because as far as ZenSpace
knows, it is. Someone else books it in ZenSpace and the room shows busy in Outlook to nobody.

**The adapter keeps the two in step, both ways, continuously.** Nobody changes how they
book anything.

## The vocabulary

These words mean specific things throughout the product and throughout this document.

| Word | Meaning |
|---|---|
| **ZenCore** | The core ZenSpace platform — organisations, spaces, bookings, sign-in. The adapter talks to it over HTTP. |
| **pod** | A bookable meeting space in ZenCore. What ZenSpace sells. |
| **room** | A bookable resource in the *customer's* calendar — a Google resource calendar, an Exchange room mailbox. |
| **connection** | One customer's link to one calendar provider. An organisation can hold a Google connection and a Microsoft one at the same time. |
| **mapping** | "This pod **is** that room." One pod to one room, both directions, never more. |
| **booking link** | The adapter's private record that a particular ZenCore booking and a particular calendar event are the same meeting. |
| **origin** | On every booking link: which side the meeting started from. `calendar` or `zencore`. This is the most important field in the system — see below. |
| **discovery** | Reading the customer's directory to find out what rooms they have. |
| **reconciliation** | Comparing both sides and repairing what can be repaired safely. |

## What it actually does

Once a pod is mapped to a room:

- **Someone books the room in Outlook or Google Calendar** → within seconds a matching booking
  appears in ZenCore against that pod. The display on the wall shows it. Check-in works.
- **Someone books the pod in ZenSpace** → an event appears in the room's calendar, so the room
  shows busy to everyone in the company.
- **Either side moves the meeting** → the other side moves.
- **Either side cancels** → the other side is freed.
- **Nobody turns up** → if the customer has set a check-in window on that pod, the room is
  released on both sides.

## What it refuses to do, on purpose

Every one of these will look like a missing feature if you do not know it is deliberate.

- **It never deletes a calendar event that the customer created.** Not on failure, not on
  disconnect, not on cancellation, not ever. Those are real meetings in real people's diaries.
  Events the *adapter* created are its own and may be deleted — which is exactly why every
  booking link records its `origin`.
- **It never creates, renames or deletes anything in the customer's directory.** It reads
  rooms; it does not manage them.
- **A release affects one occurrence, never a series.** Nobody came to this Tuesday's
  stand-up. That says nothing about next Tuesday's.
- **It never cancels a meeting that already happened.** When a whole recurring series is
  cancelled, the provider reports every occurrence as cancelled — including last month's.
  Acting on those would rewrite history and can un-bill a booking.
- **One pod maps to one room, both ways.** Anything else is rejected rather than guessed at.
- **It refuses to guess a mapping.** Suggestions decline to answer when two rooms score within
  a whisker of each other, and a number mismatch ("Pod 3" vs "Pod 8") scores zero however
  similar the names look. A wrong mapping sends real bookings to the wrong room.
- **It reads almost nothing about a meeting.** Times, and the organiser's email address —
  because ZenCore requires one. Never the title, never the description, never the attendee
  list. Mirrored bookings therefore carry a generated title (see below), not the real one.
- **Logs contain ids and short reasons only.** No meeting titles, no attendees, no credentials.

## What a mirrored booking looks like in ZenCore

When the adapter copies a calendar meeting into ZenCore, the booking it creates has:

- **title** — `Booked in the room calendar`. Generated, always. The adapter never reads the
  real title, so there is none to copy. *This is not a bug.*
- **attendees** — empty, deliberately.
- **total_amount / deposit_amount** — `0`. A meeting somebody put in their own calendar was
  not sold by ZenSpace.
- **status** — `confirmed`, **payment_status** — `paid`. Nothing is awaiting approval and
  nothing is owed. Left at ZenCore's defaults it would read "unconfirmed, unpaid", and a
  pending booking is not one a display or a check-in flow treats as real.
- **booking_source** — `external`, so a report can tell a mirrored meeting from a paid one.
- **organizer_email** — the person who created the meeting.
- **external_calendar_id** — the calendar event id.

## The shape of the thing

One service, one container, one database. It serves both the API and the admin UI that
administrators use, on the same origin.

The admin UI has six screens (a seventh, Diagnostics, exists only outside production):

| Screen | What it is for |
|---|---|
| **Overview** | Is everything set up, and is anything wrong. The first screen after sign-in. |
| **Connections** | Connect or disconnect a calendar. Enter the ZenCore API key. Register a customer's own OAuth application. |
| **Discovered rooms** | Every room found in the customer's directory, and whether each is linked. |
| **Space mapping** | Link pods to rooms — one at a time, or in bulk with suggestions. |
| **Sync health** | Per-pod: is this pod syncing, in which directions, and what happened recently. |
| **Directory** | Who in the connected company can put a meeting in these rooms. Read live; nothing stored. |

---

# Part 2 — The setup path, which is also the main test path

A customer goes through this in order. Each step depends on the one before, so this doubles
as the order to test in.

**1. Sign in.** Work email with a one-time code, or Continue with Google. Microsoft sign-in
is deliberately not offered — ZenSpace accounts are email or Google. (A customer can still
connect a Microsoft 365 *calendar* once inside; the two are unrelated.)

**2. Choose an organisation.** Only shown when the account belongs to more than one. Each
card says whether that organisation already has a calendar connected.

**3. Connections → enter the ZenCore API key.** The organisation's own API key, which the
adapter uses to act on its behalf. It is proved before it is stored — a key that does not
work is rejected on the spot rather than accepted and failing quietly later. Without it,
mapping a pod only registers half the plumbing.

**4. Connections → connect a calendar.** Sign in as an administrator of the customer's
Google Workspace or Microsoft 365 and grant access.

**5. Verification runs automatically.** The card says "Checking what we can do with your
calendar" for a few seconds. The adapter **writes a real test booking into one room and
removes it again**, because a granted permission is not proof that a booking will stick.
Until that write succeeds the card says *"We can see your rooms but cannot book them."*

**6. Discovered rooms fills in.** Discovery runs as part of connecting, so there should be
rooms immediately rather than after an overnight sweep.

**7. Space mapping → link pods to rooms.** Apply suggestions, or link by hand. Each link
registers two subscriptions: one so the calendar tells us about changes, one so ZenCore does.

**8. Sync health → confirm.** Every mapped pod should read **Syncing both ways**.

---

# Part 3 — Before you can test anything

These are the things that make testing fail before the product is even involved. Confirm all
of them first.

## You need

- A **ZenSpace account** that is a member of a test organisation, with pods in it.
- A **ZenCore API key for that organisation**, carrying all seven scopes below, spelled
  exactly:

  `create:webhooks`, `read:webhooks`, `delete:webhooks`,
  `read:bookings`, `create:bookings`, `update:bookings`,
  `read:meetingSpaces`

  Spelling matters — `read:meeting-spaces` is not `read:meetingSpaces`.

  The key must also have a **valid creating user**. A key whose `created_by` user has been
  deleted passes every check the adapter can make and then fails only when a pod is mapped,
  as a registration error. If keys are accepted but subscriptions never register, this is why.

- A **test calendar tenant** — a Google Workspace or Microsoft 365 you can administer — with
  at least two rooms defined as bookable resources, and at least one ordinary user account to
  book from.

## Google, specifically

**Domain verification is the longest pole and nothing works without it.** The customer must
add `calendar-adapter.zenspace.io` under *Domain verification* in their own Google Cloud
project. Without it Google will not create a push channel at all, so the calendar will never
tell the adapter anything. Bookings made in Google will simply not appear, with no error.

The Directory screen additionally needs `admin.directory.user.readonly`. A calendar connected
before that permission existed syncs perfectly and fails only on that one screen.

## Microsoft, specifically

- **Consent is two legs.** Signing in is leg one. Leg two is a separate approval screen that
  grants the tenant-wide permissions. A connection sitting at *"Waiting for you to finish
  granting access"* has done the first and not the second.
- **The permissions must be APPLICATION permissions, not Delegated**, added to the app
  registration in Entra and then approved with **Grant admin consent**. Accepting the sign-in
  screen does not do this. If this is missed, the card says *"Your IT admin has not approved
  our permissions yet"* — and reconnecting will not help, however many times you try it.
- The Directory screen needs `User.Read.All`, which must be added to the app registration
  before re-consenting.

## Timing — how long to wait before calling something broken

Most sync is driven by a push from the provider and happens in seconds. Everything else is on
a schedule:

| What | When |
|---|---|
| Light reconciliation (last 2 days) | every 15 minutes |
| Full reconciliation (60 days) | 04:00 daily |
| No-show release sweep | every 5 minutes |
| Subscription renewal | every 15 minutes |
| Connection re-check (can we still read *and* write) | hourly, at :05 |
| Directory re-read | 02:30 daily |
| Stranded-write sweep and pruning | hourly, at :20 |
| A revoked ZenSpace session stops working here | within 60 seconds |

Two manual triggers save you the wait: **"Check both sides now"** on Sync health forces a
reconciliation, and **re-read directory** on Discovered rooms forces a discovery.

**Google coalesces notifications and sometimes sends none at all.** This is normal and
expected. It is exactly why reconciliation exists. If a change has not appeared after a few
seconds, press "Check both sides now" before concluding anything.

The sync window is **90 days ahead and 24 hours back**. A meeting four months out is outside
it and will not sync until the window rolls forward. Test with dates inside it.

---

# Part 4 — The test plan

Each test says what to do, what should happen, and — where it is not obvious — why it matters,
so you can tell a real failure from a surprise.

## A. Sign-in and organisation scope

| # | Do this | Expect |
|---|---|---|
| A1 | Sign in with a work email and the emailed code | Signed in |
| A2 | Sign in with Continue with Google | Signed in |
| A3 | Sign in as an account in two organisations | The organisation picker appears, each card saying whether a calendar is connected |
| A4 | Sign in as an account in one organisation | Straight through to Overview, no picker |
| A5 | Sign out of ZenSpace elsewhere, then use the adapter | Within about a minute it treats you as signed out |
| A6 | While ZenCore is unreachable, load any screen | It says ZenSpace could not be reached — **not** "you are signed out". Sending someone to re-authenticate against something that is down is the failure being avoided |
| A7 | Refresh the page on a deep link such as `/sync-health` | The same screen loads, not a 404 |

**Scope:** an administrator must never be able to see or touch another organisation's data by
changing anything in the browser. Which organisation you are acting as comes from your
sign-in, never from a value the page sends. If you find any way to break that, it is the most
serious class of bug in this product.

## B. Connecting a calendar

| # | Do this | Expect |
|---|---|---|
| B1 | Save a valid ZenCore API key | Accepted, and it reports how many pods it can see |
| B2 | Save a key with a scope missing | Rejected immediately, naming the scopes needed. It is proved before it is stored |
| B3 | Save another organisation's key | Rejected |
| B4 | Connect Google, granting everything | Goes to *Checking what we can do…*, then **Connected and working** |
| B5 | Connect Microsoft, both consent legs | Same, ending at **Connected and working** |
| B6 | Start Microsoft consent and abandon it after leg one | Card reads *"Waiting for you to finish granting access"* — not "failed" |
| B7 | Connect a calendar where the rooms are read-only to us | *"We can see your rooms but cannot book them"* |
| B8 | Connect a tenant with no rooms defined | *"Connected, but there are no rooms to link yet"* |
| B9 | Connect Microsoft without granting application permissions | *"Your IT admin has not approved our permissions yet"*, with instructions naming Entra |
| B10 | Try to connect a second Google connection for the same organisation | Refused — one live connection per provider per organisation |
| B11 | After connecting, check the room calendar used for the write probe | A test booking was created and removed. Nothing left behind |
| B12 | Abandon an OAuth flow, wait 15 minutes, then return to the callback URL | Refused. The consent state is single-use and expires |

## C. Room discovery

| # | Do this | Expect |
|---|---|---|
| C1 | Open Discovered rooms straight after connecting | Rooms appear within moments — the read is queued as part of connecting, so it may need one refresh. What must **not** happen is an empty screen until the overnight sweep |
| C2 | Add a room in the customer's admin console, re-read the directory | The new room appears |
| C3 | Remove a room from the directory, re-read | The room is **still listed**, marked inactive — not deleted. "This room used to exist" answers a support question that a missing row cannot |
| C4 | Check building names on a Google directory | Real building names, not UUIDs |
| C5 | Connect a Google tenant with no buildings defined | Rooms still discovered. A cosmetic lookup failing must never lose the rooms |

## D. Linking and unlinking

| # | Do this | Expect |
|---|---|---|
| D1 | Link one pod to one room | Linked. Sync health shows it |
| D2 | Link the same pod to a second room | Rejected: *"This pod is already linked to a different room. Unlink it first."* |
| D3 | Link a second pod to the same room | Rejected, the other way round |
| D4 | Press Apply twice quickly | No duplicate, no wall of red — the second is "already linked" |
| D5 | Link a room that has since become inactive | Rejected, naming the room |
| D6 | Bulk-apply across many pods | Serialised, with a per-row outcome. Some rows failing does not abandon the rest |
| D7 | Re-run the same bulk apply | It completes what was missing and duplicates nothing |
| D8 | Ask for suggestions where names match well | Sensible suggestions |
| D9 | Ask where "Pod 3" and "Pod 8" are the only candidates | **No suggestion.** A number mismatch scores zero |
| D10 | Unlink a pod | Both subscriptions stop. The message says bookings already in the calendar are left as they are |
| D11 | Check the room's calendar after D10 | **Every event is still there.** Unlinking stops syncing; it does not tidy up |
| D12 | Re-link the same pod to the same room | Works. It must not be blocked by anything left over from the unlink |
| D13 | After D12, watch the meetings already in that room | They do **not** get duplicated as second bookings |

## E. Calendar → ZenSpace (inbound)

Book from an ordinary user account, inviting the room. Not from the admin console.

| # | Do this | Expect |
|---|---|---|
| E1 | Book the room for tomorrow | A booking appears against the pod within seconds — title `Booked in the room calendar`, organiser your address, source `external`, amount 0 |
| E2 | Check the organiser on a Google booking | The **person who booked**, not the room's own mailbox. On Google the room is technically the organiser of anything booked into it, so reading the wrong field attributes every booking to the room |
| E3 | Move that meeting an hour later | The booking moves. **See Part 5 — this is currently broken** |
| E4 | Delete the meeting | The booking is cancelled and the pod goes free |
| E5 | Book, then delete before anything syncs | Nothing is left behind |
| E6 | Book something that already happened yesterday | Inside the 24-hour look-back it syncs; older is outside the window |
| E7 | Book for four months out | Outside the 90-day window. Not a bug |
| E8 | Book the same room from two accounts at once | Whatever the calendar allows is mirrored faithfully |
| E9 | Check the service logs after all of the above | **No meeting titles, no attendee addresses, no credentials.** Ids and reasons only |

## F. ZenSpace → calendar (outbound)

| # | Do this | Expect |
|---|---|---|
| F1 | Create a booking on a mapped pod in ZenCore | An event appears in the room's calendar at the right time |
| F2 | Check the timezone stamped on that event | UTC. Not the timezone of whichever administrator happened to consent |
| F3 | Move the ZenCore booking | The event moves |
| F4 | Cancel the ZenCore booking | The event is deleted — it was ours to delete |
| F5 | Watch for a loop | The event the adapter just wrote must **not** come back as a second booking. Two independent guards exist for this; if a meeting ever duplicates itself, stop and report it immediately |
| F6 | Create a booking on an *unmapped* pod | Nothing written anywhere. Not an error |
| F7 | Create a booking while the connection cannot write | Recorded as failed, visible on Sync health |

## G. Cancellation, which is where the subtle bugs live

| # | Do this | Expect |
|---|---|---|
| G1 | Cancel in ZenCore a booking that **came from the calendar** | The ZenCore booking is cancelled and the pod goes free. **Their meeting stays in the diary, untouched.** |
| G2 | Wait a day after G1 (or force reconciliation) | The booking does **not** come back. A cancellation that silently undoes itself overnight is the bug this guards |
| G3 | After G1, move the meeting in the calendar | The cancelled booking is **not** revived |
| G4 | Delete from the calendar an event the adapter itself created from a ZenCore booking | The ZenCore booking is cancelled. This is somebody cancelling that meeting, not an echo |
| G5 | Cancel the same meeting twice, or cancel and then force reconciliation | No error. The end state is what matters, not who got there first |
| G6 | Cancel one occurrence of a recurring meeting | Only that occurrence's booking is cancelled |
| G7 | Cancel a whole recurring series that has been running for months | **Future occurrences only.** Past ones are left alone |

## H. Recurrence

| # | Do this | Expect |
|---|---|---|
| H1 | Create a weekly recurring meeting in the room | One booking **per occurrence** inside the window, not one booking for the series |
| H2 | Move a single occurrence | Only that one moves |
| H3 | Let the window roll forward | Later occurrences appear |
| H4 | Check a series spanning a daylight-saving change | Every occurrence is at the right local time. An hour's drift after the clocks change is the failure being guarded |

## I. No-show release

Requires a **check-in window** set on the pod in ZenCore, in minutes.

| # | Do this | Expect |
|---|---|---|
| I1 | Pod with **no** check-in window, nobody arrives | **Nothing is released, ever.** No policy is not permission to act |
| I2 | Pod with a 10-minute window, still inside it | Nothing yet |
| I3 | Window elapsed, nobody checked in, event created by ZenSpace | Booking marked no-show, our event deleted. **See Part 5 — currently broken** |
| I4 | Same, but the meeting came from the customer's calendar | The room **declines** the meeting rather than deleting it. The meeting stays in everyone's diary; the room is freed |
| I5 | Someone checks in a moment before the sweep | Nothing is released |
| I6 | A recurring meeting where nobody came this week | **This occurrence only.** Next week must be untouched |

## J. Disconnect and reconnect — the "does removing an account break anything" tests

| # | Do this | Expect |
|---|---|---|
| J1 | Disconnect a calendar with pods mapped and bookings in flight | Succeeds |
| J2 | Look at the customer's calendar afterwards | **Every event is still there, including ones the adapter created.** A disconnect is not a statement that those meetings were cancelled |
| J3 | Look at Space mapping and Discovered rooms afterwards | Both empty for that connection. Mappings and rooms do not survive a disconnect |
| J4 | Look at Connections afterwards | The connection is shown as disconnected, not vanished — it is part of the audit trail |
| J5 | Reconnect the same calendar | Works. Rooms are discovered again |
| J6 | Count the rooms after reconnecting | Each room appears **once**. Not once per connection the customer has ever made |
| J7 | Re-link a pod that was mapped before the disconnect | Works. It must not be blocked by a leftover mapping |
| J8 | Disconnect when the customer has already removed our access at their end | Still succeeds. A customer who revoked us must still be able to finish here |
| J9 | Disconnect and reconnect twice in a row | No duplicates, no stuck state |
| J10 | With two connections (Google **and** Microsoft), disconnect one | The other is completely unaffected |
| J11 | Change the ZenCore API key to a different valid one | Existing mappings keep working |
| J12 | Remove a customer's own OAuth application while a connection is live | Refused — the refresh token was granted to that client id |

## K. Health, diagnosis and reconciliation

| # | Do this | Expect |
|---|---|---|
| K1 | Open Sync health with everything working | Every pod **Syncing both ways** |
| K2 | Let a subscription lapse on one side | That pod shows which direction is deaf — the two are reported separately, never merged |
| K3 | Press "Check both sides now" | A reconciliation runs and reports what it compared |
| K4 | Create a booking directly in the calendar while the adapter is stopped, then start it and reconcile | The missing booking is created |
| K5 | The reverse — a ZenCore booking whose calendar event is missing | Reported for a person to decide, **not** silently deleted. Guessing wrong in that direction removes real meetings |
| K6 | Read the Recent activity table | Plain sentences, not event codes |
| K7 | Check `/health` | Reports the database and the live queue connection |

---

# Part 5 — Known issues. Read this before filing a bug.

## 1. Moving a meeting does not update the booking (tests E3, F3)

**ZenSpace itself rejects booking updates from this adapter.** ZenCore's
`PUT /bookings/:id` requires a signed-in user's identity, and the adapter authenticates with
an organisation API key, which carries none. Creating and cancelling bookings both accept the
same key; only updating does not.

**What you will see:**

- A meeting moved in the customer's calendar keeps its old time in ZenSpace, and the pod stays
  held at the old time.
- **No-show release never frees a room** (test I3), because marking a booking as a no-show is
  also an update.

**How the adapter reports it** — this part is worth checking, because it is new:

- Sync health marks the pod **degraded**, with the words *"New bookings sync, but changes to a
  meeting do not reach ZenSpace"* — even though both subscriptions are alive and green.
- Recent activity shows *"A meeting was moved in the calendar, but ZenSpace would not accept
  the change"* and *"Nobody arrived, but ZenSpace would not release the room, so it was left
  booked."*

**What is deliberately *not* done:** the calendar is never touched to compensate. In the
no-show case the room stays booked on **both** sides, which is honest. Freeing only the
calendar would leave a pod that reads as booked and cannot be booked, which is worse.

**Please do not file the symptoms separately.** They are one fix in ZenCore, not in this
adapter. Once it ships, pending moves apply themselves on the next sync without anyone
retrying them.

## 2. Neither provider has been exercised against a real tenant

Every provider test so far drives a fake built from the documented API shapes. Google and
Microsoft have not been tested against live accounts. **Treat all of Part 4 as unproven
against real providers** — that is precisely what this round of testing is for. Expect
surprises in the areas most likely to differ from documentation: recurrence expansion,
timezone handling, cancelled-event payloads, and how quickly notifications arrive.

## 3. Google push channels need domain verification first

Covered in Part 3. Worth repeating because the failure is completely silent: without it, no
push channel can be created, so nothing in the calendar ever reaches the adapter, and no error
appears anywhere. Bookings made in Google will just not arrive. If inbound sync appears
entirely dead on a Google tenant, check this before anything else.

## 4. Simultaneous bookings for one slot

ZenCore has no lock between checking for a conflict and inserting a booking, so two
simultaneous requests for the same slot can both succeed. The adapter cannot fix this from
outside. If you double-book a pod by racing two requests, that is a ZenCore issue.

---

# Part 6 — Working out what went wrong

Try these in order.

**1. Sync health.** Start here, always. It answers two separate questions per pod — can this
pod hear the calendar, and can it hear ZenSpace — and it never merges them, because each has
a different cause and a different fix. Anything unhealthy is pulled to the top of the screen.

**2. Recent activity**, on the same screen. Written as sentences rather than codes. The ones
worth knowing:

| It says | It means |
|---|---|
| A calendar booking was copied into ZenSpace | Inbound worked |
| A ZenSpace booking was added to the calendar | Outbound worked |
| No action needed — this booking already exists in the calendar | An echo was correctly ignored |
| Left alone — this meeting was not created by ZenSpace | The adapter refused to touch a customer's event. Usually correct |
| A meeting was moved…but ZenSpace would not accept the change | Known issue 1 |
| Nobody arrived, but ZenSpace would not release the room | Known issue 1 |
| Re-read this room from scratch, which is routine | A provider expired our bookmark. Normal |
| A message arrived that we could not verify, and was ignored | A notification failed its authenticity check |

**3. "Check both sides now."** Before concluding that sync is broken, force a reconciliation.
A missed push is normal, especially on Google. If reconciliation repairs it, the plumbing
works and the *notification* was lost — a different problem with a different fix.

**4. The Connections card.** It states plainly what is wrong and what to do about it. Note
that *"We cannot reach your calendar"* and *"Your IT admin has not approved our permissions
yet"* are deliberately different messages, because reconnecting fixes the first and is
useless for the second.

**5. Diagnostics** — development and staging only, never production. Mints a token, shows
what permissions it actually carries, lists the rooms, and queries one mailbox two ways. It is
read-only and cannot change anything.

## When reporting a bug, include

- Which screen, and what the card or chip **said** — the exact words, which are chosen to name
  the cause.
- The provider, and which tenant.
- The pod and the room.
- The times involved, **with timezones**. A great many calendar bugs are timezone bugs.
- Whether it recovered by itself after 15 minutes, or after pressing "Check both sides now".
  This single fact separates "sync is broken" from "a notification was lost", which are
  different problems.
- What the customer's calendar looks like now, and what ZenSpace shows now.

**Report immediately, without waiting, if you see any of these:**

- A meeting the customer created is **deleted** or altered by the adapter.
- A meeting **duplicating itself**, or a booking loop.
- Data from one organisation visible to another.
- A meeting title, attendee list or credential appearing in a log.
- A **past** booking being cancelled or modified.

Those five are the failures this product is built specifically to prevent, and each one is
more serious than anything else on this page.
