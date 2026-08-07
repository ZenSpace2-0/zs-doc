# Google Calendar Sync Bridge — Overview

*A single plain-language overview: what we're trying to achieve, how it works, the options for where calendars live, and the known risks. For deep technical detail see the companion docs: `zenspace-calendar-sync-bridge-FULL.md` (master planning doc) and `approach-b-implementation-flow.md` (Approach B with API references).*

---

# 1. What We're Trying to Achieve

## The goal (one sentence)

Let ZenSpace customers accept real bookings for their spaces through third-party platforms that have no direct API — while ZenCore stays the single source of truth and no double-bookings ever slip through.

## The situation today

- **ZenCore** is ZenSpace's booking system — the official record of every space and every booking.
- Customers want their spaces to also be bookable on outside marketplaces: **Deskpass, Upflex, LiquidSpace**.
- Those platforms **don't offer a direct API** to integrate with. The only channel they all speak is **Google Calendar**.
- So today, at best, those platforms can show availability as a read-only display — they can't actually take a booking that ZenCore will honor.

## What we want instead

- Those third-party platforms should function as **real booking channels**, not just "here's what's free" windows.
- A booking made on Upflex (or Deskpass, or LiquidSpace) should flow into ZenCore and become a genuine, honored booking.
- A booking made in ZenCore should show up on those platforms as unavailable, so nobody else grabs the same slot.
- This should work **both directions**, automatically, in near-real-time.

## The non-negotiables (what "success" must protect)

- **ZenCore stays the boss.** It remains the single source of truth. Nothing becomes a real booking until ZenCore validates and confirms it against its own rules (business hours, existing bookings, blocked-out times).
- **Zero double-bookings.** If two people try to grab the same space at nearly the same moment through different channels, exactly one wins — cleanly and predictably.
- **Invisible to the end booker.** This is plumbing. The person booking on Upflex never knows ZenCore exists.
- **Every customer's data stays isolated** from every other customer's — a core platform principle.

## What good looks like (success criteria)

- No double-bookings caused by sync timing or missed events.
- Fast sync — a booking appearing on a calendar is confirmed or rejected by ZenCore within seconds.
- A new customer can be turned on with little or no engineering effort.
- Third-party bookings show up in ZenCore's booking list and revenue reporting exactly like native bookings.

## Why it's worth doing (business value)

- Turns dead read-only integrations into live revenue channels for customers.
- Lets customers plug into new platforms without custom engineering each time.
- A differentiating, sellable feature that makes ZenCore more valuable.

## What this is *not* (scope boundaries)

- Not touching ZenEdge (the IoT/hardware enforcement product) — this lives entirely in ZenCore's integration layer.
- Not building direct integrations to each platform — Google Calendar is the shared bridge on purpose.
- Not replacing ZenCore's booking rules — it reuses them, so bridged bookings obey the same rules as native ones.

---

# 2. How It Works — Visual Flow

Two things to picture: **where the sync logic lives** (an open architecture choice) and **how a booking actually moves** (the same either way).

## 2.1 The architecture choice (still open)

Both options do the **same work** — push bookings out to Google, receive Google's webhook, pull changes back in, validate against ZenCore. They differ only in **where that sync logic lives.**

### Option 1 — Sync built into ZenCore

ZenCore itself does everything: pushes bookings out to Google, receives Google's webhook, pulls changes back in. The Google-sync logic lives inside ZenCore. Simpler, fewer moving parts. The risk: all the noisy Google plumbing (webhook renewals, retries, outages, platform quirks) sits inside your core booking system.

```
┌──────────────────────────┐        ┌───────────────────┐        ┌────────────────────┐
│        ZenCore           │  push  │  Google Calendar  │  read  │     Platforms      │
│  booking system          │ ─────► │ one calendar per  │ ◄───── │  Deskpass, Upflex, │
│  + Google sync built in  │        │      space        │        │    LiquidSpace     │
│                          │ ◄───── │                   │ ─────► │                    │
└──────────────────────────┘webhook └───────────────────┘ write  └────────────────────┘
             │
      (no separate service — sync logic lives inside core booking)
```

### Option 2 — A separate bridge service

Same operations happen, but a dedicated sync service sits in the middle. It handles all the Google plumbing (webhooks, renewals, pulling changes, platform quirks) and — importantly — it doesn't touch ZenCore's database directly. It calls ZenCore's booking API to validate and confirm every booking, so ZenCore's rules stay the single authority. If Google has an outage or the plumbing misbehaves, it degrades sync but never drags down core booking.

```
┌──────────────┐  validate   ┌───────────────┐  push out  ┌──────────────────┐  read   ┌──────────┐
│   ZenCore    │  + confirm  │  Sync service │ ─────────► │ Google Calendar  │ ◄────── │ Platforms│
│  source of   │ ◄────────── │  (the bridge) │            │  one per space   │         │ Deskpass │
│    truth     │ ─────────►  │               │ ◄───────── │                  │ ──────► │  Upflex  │
└──────────────┘  booking    └───────────────┘ webhook/   └──────────────────┘  write  └──────────┘
                  events            │            pull
                         (handles all Google plumbing + failures,
                          never touches ZenCore's DB directly)
```

**The difference in one line:** it's not *what* happens (both do the same push-out / webhook-in / validate steps) — it's *where the sync logic lives*. Inside ZenCore (simpler, but plumbing risk touches core) versus a separate process (more robust and cleaner boundary, but one more thing to run).

## 2.2 The booking flow (same either way)

Whichever architecture is chosen, a booking moves the same way. There are two directions, and they are **not** symmetrical.

### Inbound — outside platform → ZenCore (the careful direction)

```
   1. Someone books on Upflex / Deskpass
                 │
                 ▼
   2. Written as an event on Google Calendar
                 │
                 ▼
   3. Google pings the bridge: "something changed"   ◄── note: the ping is empty,
                 │                                        it does NOT say what changed
                 ▼
   4. Bridge pulls the new event (asks Google what changed)
                 │
                 ▼
   5. Bridge asks ZenCore: "is this slot actually bookable?"
                 │                    (checks business hours, existing
                 │                     bookings, blocked times — under a lock)
        ┌────────┴────────┐
        ▼                 ▼
   6a. YES →          6b. NO →
   saved as a         event removed
   real booking       from calendar,
                      conflict logged
```

The important beats: steps 3–4 — Google only says "something changed," so the bridge has to go *pull* the actual event. And steps 5–6 — nothing is trusted; every incoming booking is re-checked against ZenCore, and either becomes real (6a) or gets removed and logged (6b). **That validation step is what prevents double-bookings.**

### Outbound — ZenCore → platforms (the easy direction)

```
   1. Booking made      2. ZenCore tells      3. Written to         4. Platforms see
      in ZenCore    ─►     the bridge      ─►    Google Calendar ─►    slot as taken
```

This direction is simpler because there's **nothing to validate** — ZenCore already confirmed the booking, so the bridge just publishes it outward as a calendar event, and the platforms pick it up as unavailable.

### The one asymmetry that matters

- **Incoming bookings are guilty until proven innocent** — they must pass ZenCore's check before becoming real.
- **Outgoing bookings are already trusted** — ZenCore approved them, so they're just published.

That asymmetry is the heart of how double-bookings get prevented: the only way a booking becomes "real" is by passing through ZenCore's single authoritative check, no matter which channel it came from.

---

# 3. Where the Calendars Live — Three Approaches

There are **three ways** to set up where the Google Calendars live. (One calendar is needed per space.) This is a separate decision from the architecture choice above.

## Approach A — Customer owns the calendars (per-customer OAuth)
Each customer connects **their own** Google account. Their space calendars stay in their own Google Workspace.

**Pros**
- Each customer's data stays separate and safe.
- If one customer's account is hacked, others are not affected.
- No speed problems — each customer has their own Google usage limits.
- Works well for big or strict customers who care about data rules.

**Cons**
- Harder to set up — the customer's IT team usually has to approve it.
- Doesn't work for customers who don't use Google (many hotels use Microsoft).
- Only useful if most customers actually have Google.

## Approach B — ZenSpace owns the calendars (ZenSpace-hosted)
ZenSpace uses **one** Google account and creates all customer calendars inside it. Customers don't need Google at all.

**Pros**
- Very easy setup — nothing needed from the customer.
- Fully automatic — calendars are created instantly.
- Only one system to build and manage.

**Cons**
- ZenSpace now holds everyone's booking data in one place (more legal responsibility).
- If that one account is hacked, **all** customers are affected.
- All customers share the same Google usage limit — a busy customer can slow down others.
- Cannot follow country-specific data rules (e.g. Europe) — data location is fixed.
- Keeping each customer's data separate now depends fully on ZenSpace's own code.

## Approach C — Mix of both (hosted by default, customer-owned as an upgrade)
ZenSpace hosts calendars for most customers (easy setup), but big or strict customers can choose to connect their own Google account.

**Pros**
- Easy setup for most customers.
- Still has a good answer for big customers who care about data safety.
- Best of both worlds.

**Cons**
- Most work to build — you have to make both systems.
- Setup has to figure out which type each customer is.
- The default (hosted) side still has the same data-responsibility concerns.

## Comparison table

| What matters | A: Customer owns | B: ZenSpace owns | C: Mix of both |
|---|---|---|---|
| **Easy to set up?** | Hard | Easiest | Easy for most |
| **Works without customer having Google?** | No | Yes | Yes |
| **Data kept separate & safe?** | Best | Weakest | Depends on type |
| **If hacked, who is affected?** | One customer | All customers | Default: all / Upgrade: one |
| **Speed / usage limits** | Own limit each | Shared (can slow down) | Shared or own |
| **Follows country data rules?** | Yes | No | Only upgrade side |
| **Legal responsibility on ZenSpace** | Low | High | Medium–High |
| **How much to build** | One system | One system | Two systems (most) |
| **Best for** | Big / strict customers | Small / coworking customers | Mixed customers |

## How to choose

It depends on **who your first customers are**:
- Mostly small or coworking spaces → **Approach B** (easy setup wins).
- Mostly hotels or big companies → **Approach A** (data safety wins).
- A mix of both → **Approach C**.

**Simple way to find out:** check the email addresses of current customers to see how many already use Google vs Microsoft. That tells us which approach fits best.

---

# 4. Known Limitations & Issues

Things that could block or break us — split into three groups: problems we already know about, a hidden risk, and outside things we still need to check.

## 4.1 Problems we already know about

**Google can quietly stop sending us updates.**
Google tells us when someone books on an outside platform, but this "listening connection" expires after a while. If we don't refresh it in time, bookings stop coming in — with no error message. It just goes silent. So we must build an automatic refresh, plus a regular "double-check everything" sweep as backup.

**Google can reset our sync link.**
Sometimes Google invalidates the shortcut we use to get updates. When that happens, we have to re-pull everything fresh. This is normal and we plan for it.

**Google has usage limits.**
Google limits how much we can use its calendar system. We must ask Google to raise our limit before we grow. (This is also why the "ZenSpace owns one account" option is risky — everyone would share one limit.)

**ZenCore connection — already sorted.**
We were worried ZenCore might not give us booking info easily. Good news: it does, per customer, using an API key. No problem here.

## 4.2 The hidden risk (most important one)

**ZenCore reads and writes bookings using slightly different rules.**
Inside ZenCore, the part that *shows* availability and the part that *saves* a booking don't use the exact same logic. For example, they can read time zones differently, and handle opening hours differently.

Today this doesn't cause trouble. But once outside bookings start flowing in, this mismatch could let two bookings land on the same space at the same time — which is the **exact thing this whole project is meant to prevent.**

**What we need to decide:** either fix these small differences inside ZenCore first, or make sure every outside booking goes through ZenCore's "save" path (not its "show" path) so it follows one consistent set of rules. Someone needs to own this decision before we start building.

## 4.3 Outside things we still need to check

**Google Calendar can't "hold" a slot.**
Google has no way to lock a slot for a moment while we decide. That's fine — we handle the locking inside ZenCore instead — but we should confirm this works for all three platforms.

**Limits per Google account.**
Google limits how many calendars and connections one account can handle. This matters a lot if ZenSpace owns one big account for everyone. We need to check the numbers.

**Each platform may write bookings differently.**
Deskpass, Upflex, and LiquidSpace might each fill in calendar events in their own way (different fields, time formats, etc.). We need to look at each one so we can read their bookings correctly.

## 4.4 Bottom line
Nothing here stops us from building. But two things must be handled carefully: keeping Google's "listening connection" alive (so bookings don't silently stop), and fixing the ZenCore read-vs-write rule differences (so we don't accidentally allow double-bookings). The second one is the bigger risk and needs an owner before we start.

---

*End of overview. Sequence: goal → how it works → where calendars live → known limitations. Deep detail lives in `zenspace-calendar-sync-bridge-FULL.md` and `approach-b-implementation-flow.md`.*
