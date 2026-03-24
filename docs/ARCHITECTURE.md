# Todoist “logical day” boundary — architecture notes

**Version:** 1.0  
**Who this is for:** The client, whoever builds this, and anyone maintaining it later.

The short version: we want completions and recurring-task behavior to line up with a **day that ends at 10:00 AM** (local time), not midnight—**without** moving everyone’s due times to 10 AM. That last part matters because you’ve already seen scripts that “fixed” rollover by shifting tasks; we’re explicitly not doing that.

---

## What we’re actually solving

Todoist thinks in **calendar dates** and **timestamps**. That’s fine for most people. For someone whose “Monday” often runs past midnight, finishing something at 1 AM can still *feel* like Monday—but Todoist has already ticked over to Tuesday. Recurring tasks are especially painful here: complete one at the wrong moment, and the **next** occurrence can look like it **skipped** a day, or your rhythm and Todoist’s idea of “today” stop matching.

You also tried an approach that **moved due times** to delay rollover. This project avoids that entirely. Scheduled times stay where you put them; we only intervene where the **API** lets us fix how completions and recurrence line up.

---

## Vocabulary (so we’re not talking past each other)

**Logical day** — Think of the day as running from **10:00 AM** one calendar day to **10:00 AM** the next, in *one* chosen timezone (e.g. `America/New_York`). Everything uses that same zone; we’re not configuring this per project or per task.

**Grace window** — After midnight but before 10 AM, you’re still in the tail end of the *previous* logical day. Completions in that window are the ones we care about.

**Wall-clock due time** — What you actually see on the task (“every day at 9 PM”, etc.). We preserve that.

---

## What’s in scope — and what isn’t

We’re aiming to:

- Notice when tasks (at least **recurring** ones) are **completed** between **midnight and 10 AM** local time.
- Stop recurrence from **skipping** because Todoist treated that completion as “the wrong day.”
- Keep **one** rule for the whole account: same boundary, same timezone.
- Run **without** daily babysitting: webhooks when something happens, plus a scheduled sweep so nothing falls through the cracks.

We’re **not** promising to redesign Todoist’s UI, and we’re **not** (unless you add it later) setting different boundaries per project. We’re also **not** guaranteeing that **Karma / streaks** will match a fantasy version of Todoist that natively supports a 10 AM cutoff—Todoist owns that math. We’ll be clear after a small API test what’s fixable in data vs. what stays cosmetic in the app (more on that below).

---

## Recommended shape of the solution

Todoist doesn’t offer a “my day ends at 10 AM” toggle. So this ends up being a **small service that stays on 24/7**: Todoist sends it events (webhooks), it checks whether our rules apply, and it talks back to Todoist through the **REST** and **Sync** APIs when something needs correcting.

```
┌─────────────────┐     HTTPS      ┌──────────────────────────┐
│  Todoist        │ ──────────────►│  Our integration service   │
│  (webhooks +    │   (signed)     │  · verify webhooks        │
│   APIs)         │                │  · remember what we fixed  │
└─────────────────┘                │  · boundary + recurrence  │
        ▲                          │    logic                   │
        │ you complete tasks       │  · call Sync / REST        │
        │ on phone / desktop       └─────────────┬──────────────┘
        │                                        │
        └────────────────────────────────────────┘
                    updates only when rules say so
```

**What sits inside that box**

- A **webhook endpoint** that Todoist can hit when tasks change (including completed). We verify signatures the way Todoist documents.
- A **worker** that actually processes events, with retries when the API hiccups.
- Somewhere **persistent** to store idempotency keys (so the same webhook doesn’t apply the same fix twice), optional sync cursors, and a light audit trail if we need to debug “why did this task move?”
- A **scheduler**: something like every 5–15 minutes, plus an extra pass shortly after **10 AM** local, to catch anything webhooks missed—phone offline, flaky delivery, that sort of thing.
- **Secrets** (API token or OAuth, webhook secret) living in the host’s secret store, not in the repo.

### Why cloud-first

The brief calls out that the client can’t always open a laptop or run a script. A **hosted** worker doesn’t care whether a PC is asleep; it still gets the webhook and can fix things in the background. A **local-only** script is possible as a backup (see later), but it’s inherently flakier for this use case.

### Which Todoist APIs

We’d lean on **webhooks** for speed, **REST v2** for straightforward reads, and the **Sync API** when we need to update recurrence precisely (`item_update`, `item_update_date_complete`, and friends).

One detail worth a focused test: **`item_complete`** lets you pass an optional **`date_completed`** (UTC, RFC3339). That’s documented for the “plain” completion flow. **Recurring** completions go through a different command (`item_update_date_complete`), and the docs don’t obviously promise the same backdating. So before we bet the farm on one strategy, we spend **a few hours on a test account** with your real kinds of recurrence (“every day”, “weekdays”, weird natural language) and see what Todoist actually returns. That might mean we mostly **patch the next due date** when it’s wrong, rather than replaying completions—but we’ll know instead of guessing.

---

## How the logic works

### Figuring out “which logical day is it?”

We fix a timezone and a **10:00 AM** boundary. For any moment in time:

- Convert to local time in that zone.
- If the local clock is **10:00 AM or later**, the logical date is **today’s** calendar date.
- If it’s **before** 10 AM, we still count you as on **yesterday’s** logical date.

So that 1:30 AM completion lands on the same logical day as “yesterday evening”—which is the behavior you’re asking for.

### Tasks that are *due* between midnight and 10 AM

Those **keep their due time**. A task due at 2 AM stays due at 2 AM; we’re not sliding the whole schedule.

What we **can’t** do is redraw Todoist’s built-in **Today** view to use our 10 AM boundary—that’s their product. What we **can** do is keep **recurrence** (and, where the API allows, **completion timestamps**) consistent with the logical day so the *next* instance isn’t wrong.

If you also want **non-recurring** tasks to count toward the previous logical day for stats, say so—that’s optional scope and might use `date_completed` where it applies.

### Avoiding “skipped” recurring days (the main loop)

The approach we’d implement is: **look at what Todoist did, decide if it’s wrong, fix it if needed**—not “preemptively reschedule everything.”

After a completion (from a webhook or from reconciliation), we read the task’s new state. We compute what the **next due** *should* be from the recurrence string and the **logical** completion date, then compare. If Todoist advanced one step too far (or not in the way we expect), we issue a targeted **`item_update`** via Sync so the **next occurrence** is right, while keeping the **time of day** and wording of the recurrence as intact as the API allows.

We’d dedupe work with something like a key per `(task, completion event)` so duplicate webhooks don’t double-apply. If the same task might get edits from two places at once, we’d serialize updates **per task** so we don’t race ourselves.

### If patching the due date isn’t enough

There’s a heavier fallback: **uncomplete**, then complete again through Sync in a way that matches recurring semantics—maybe with an explicit completion timestamp **if** the API supports it for that path. That can cause a flash of wrong state on devices and extra notifications, so we’d only go there if the spike shows we have to.

---

## Making it dependable day to day

Webhooks give us quick reactions when you check something off on your phone. Scheduled passes catch the cases where webhooks never arrived. Idempotency stops duplicate fixes. A simple **health** URL plus an uptime check means we know if the service is down; optional **email or Slack** alerts help when the same correction keeps failing.

Under normal conditions, you’re looking at corrections within **a few minutes** of a completion. Anything that slipped through the night should get picked up in reconciliation **not long after 10 AM** local.

---

## Where the edges are (we’d rather say this now than surprise you later)

- **Today / home screens** may still “feel” midnight-based. We’re fixing **task data** where we can, not replacing Todoist’s whole UI.
- **Karma and streaks** use Todoist’s internal rules. If those ignore the kind of backdating we can do on recurring items, the numbers might not perfectly match your logical day. We’ll validate early and set expectations with you.
- **Shared projects** need a clear rule: do we only adjust tasks on *your* account, or collaborators too?
- Phones sometimes stay offline; **sync delay** happens. That’s why reconciliation exists.
- **APIs change.** Part of maintenance is glancing at Todoist’s developer notes when things break or before big upgrades.

---

## Security and privacy (briefly)

Least-privilege tokens, webhook verification, and logs that favor **IDs** over dumping full task titles unless you need them for support. If GDPR or data residency matters, we pick hosting accordingly.

---

## Deliverables (mapped to the original brief)

- **Running service** — deployed worker, webhook wired up, scheduler, small database or KV for dedupe.
- **Setup guides** — For a hosted setup, “install on your computer” often isn’t the story: it’s connect Todoist once, paste the webhook URL into the developer app, set your timezone. If we ship a local option too, we document **Task Scheduler** on Windows and **launchd** on Mac in plain language with screenshots.
- **This kind of doc** — architecture, setup, limitations, troubleshooting.

---

## Local-only option (if you really want it)

Possible, but weaker: a small daemon on your machine, maybe **ngrok** for HTTPS so Todoist can reach you, or polling instead of webhooks (slower and noisier). Autostart via Task Scheduler / Launch Agents. The catch is sleep mode and missed webhooks—we’d lean harder on reconciliation when the machine wakes up. Cloud is still the recommendation for reliability and for **zero** daily interaction.

---

## Keeping it healthy over time

Every few months, skim **Todoist’s API / webhook docs** for breaking changes. After any weird incident, we might tighten reconciliation or add a metric. Keep the runtime and HTTP libraries patched like any small production service.

---

## What we’d look for in whoever builds this

Production experience with **REST APIs**, **webhooks**, and **background jobs** that retry safely. **Timezones and DST** trip everyone up once—we want someone who’s been burned before and tests edge cases. Deep Todoist experience helps but isn’t mandatory if the first week includes a honest **spike** on real recurrence patterns before the heavy coding.

---

## Effort ballpark

Numbers are for **one experienced engineer**; fixed-price quotes usually tuck in a little buffer for weird recurrence edge cases.

| Phase | Hours (rough range) |
|-------|---------------------|
| Spike: what Todoist actually does for recurring completions / timestamps | 4–10 |
| Service skeleton: auth, webhook, secrets, hosting | 12–20 |
| Logical-day logic + recurrence correction + automated tests | 24–48 |
| Reconciliation, monitoring, hardening idempotency | 8–16 |
| Friendly setup docs + internal runbook | 6–12 |
| Extra polish: shared tasks, DST surprises, failure modes | 8–20 |
| **Total** | **about 62–126** |

Calendar time is often **around 3–6 weeks**, depending on feedback speed and test access. **Rate / fixed fee** is for the bidder to fill in.

---

## Questions we’d ask before locking scope

- Should **one-off** (non-recurring) tasks follow the same logical-day completion rules, or only recurring?
- Is **Karma / streak alignment** a must-have, or is **correct next occurrence** the real win?
- **Solo account** vs tasks shared with others?
- Hosting preference (region) and how you want to be **notified** if the automation is stuck?
- Do you travel across timezones a lot, or is **one** home timezone enough?

---

## References

- [Todoist Developer](https://developer.todoist.com) — REST v2, Sync API, webhooks.
- Sync commands we’ll likely touch: `item_complete` (and its optional `date_completed`), `item_update_date_complete`, `item_uncomplete`, `item_update`, `item_close`.

---

This write-up is our best picture of the approach and the tradeoffs. The spike on a real account is where we turn “should work” into “we’ve seen Todoist do X, so we do Y.” Final behavior always has to match what the live API actually returns.
