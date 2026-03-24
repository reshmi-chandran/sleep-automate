# Todoist “logical day” boundary — architecture notes

**Version:** 1.2  
**Who this is for:** The client, whoever builds this, and anyone maintaining it later.

The short version: we want completions and recurring-task behavior to line up with a **day that ends at 10:00 AM** (local time), not midnight—**without** moving everyone’s due times to 10 AM. That last part matters because you’ve already seen scripts that “fixed” rollover by shifting tasks; we’re explicitly not doing that.

---

## What we’re actually solving

Todoist thinks in **calendar dates** and **timestamps**. That’s fine for most people. For someone whose “Monday” often runs past midnight, finishing something at 1 AM can still *feel* like Monday—but Todoist has already ticked over to Tuesday. Recurring tasks are especially painful here: complete one at the wrong moment, and the **next** occurrence can look like it **skipped** a day, or your rhythm and Todoist’s idea of “today” stop matching.

You also tried an approach that **moved due times** to delay rollover. This project avoids that entirely. Scheduled times stay where you put them; we only intervene where the **API** lets us fix how completions and recurrence line up.

---

## Vocabulary (so we’re not talking past each other)

**Logical day** — Think of the day as running from **10:00 AM** one calendar day to **10:00 AM** the next, in *one* chosen timezone (for example America/New_York). Everything uses that same zone; we’re not configuring this per project or per task.

**Grace window** — After midnight but before 10 AM, you’re still in the tail end of the *previous* logical day. Completions in that window are the ones we care about.

**Wall-clock due time** — What you actually see on the task (“every day at 9 PM”, etc.). We preserve that.

---

## What’s in scope — and what isn’t

We’re aiming to:

- Notice when tasks (at least **recurring** ones) are **completed** between **midnight and 10 AM** local time.
- Stop recurrence from **skipping** because Todoist treated that completion as “the wrong day.”
- Keep **one** rule for the whole account: same boundary, same timezone.
- Run **without** daily babysitting: webhooks when something happens, plus a scheduled sweep so nothing falls through the cracks.

We’re **not** promising to redesign Todoist’s UI, and we’re not (unless you add it later) setting different boundaries per project. We’re also **not** guaranteeing that **Karma / streaks** will match a fantasy version of Todoist that natively supports a 10 AM cutoff—Todoist owns that math. We’ll be clear after a small API test what’s fixable in data vs. what stays cosmetic in the app.

---

## Technical solution (what gets built)

This section is the engineering view: **what** the system does and **how** the pieces fit together. There are no implementation snippets here—just architecture and behavior.

### 1. System pattern

Todoist has no first-party “day boundary” setting. The technical approach is an **integration service**: always reachable on the public internet over HTTPS, registered as a Todoist **application** so it can receive **webhooks** and call Todoist’s **REST** and **Sync** APIs with a token tied to the user’s account.

**End-to-end flow in words:** You complete or update a task in any official Todoist client. Todoist’s servers persist the change and, for subscribed events, **POST** a signed payload to our service. Our service validates the request, enqueues work, fetches fresh task state if needed, runs **logical-day and recurrence rules**, and issues **mutating API calls** only when a correction is required. A separate **scheduled job** repeats a lighter version of the same checks so missed webhooks or delayed sync do not leave bad state behind.

### 2. Major components

| Component | Role |
|-----------|------|
| **HTTPS webhook endpoint** | Accepts Todoist callbacks, rejects bad signatures or replays, returns quickly (often by pushing work to a queue rather than doing heavy work inline). |
| **Queue + worker** | Processes events with retries, backoff, and per-task ordering so two updates for the same task do not interleave incorrectly. |
| **Rule engine** | Pure logic: given completion time (UTC), configured timezone, boundary time, task metadata (recurrence string, previous due, new due), decides whether a correction is needed and what the target state should be. |
| **Todoist API client** | Wraps authenticated REST reads and Sync command batches; handles rate limits and transient errors. |
| **Persistent store** | Idempotency keys, optional webhook event IDs, minimal audit rows (task id, action taken, timestamp), and configuration (timezone, boundary, feature flags). |
| **Scheduler** | Triggers periodic reconciliation and a **post–10 AM** sweep in local time. |
| **Observability** | Structured logs, health check URL for uptime monitors, optional alerts when corrections fail repeatedly. |

Hosting can be any managed platform that supports HTTPS, background workers, cron, and a small database (for example container platforms with cron, or serverless with a queue and scheduled functions). The important part is **always-on** availability, not a specific vendor.

### 3. Authentication and Todoist app setup

The service needs a **long-lived credential** to read tasks and apply updates. In practice that means either **OAuth** (Todoist app, user authorizes once, refresh tokens stored encrypted) or a **personal API token** (simpler for a single user, fewer moving parts, but the user must rotate it if leaked).

Separately, a **Todoist developer application** is configured with the **webhook URL** and **secret** used to verify incoming requests. One-time setup: create the app in Todoist’s developer portal, note client id/secret if using OAuth, register the production webhook URL, and complete authorization so the runtime token is available to the worker.

### 4. Webhooks: what we listen for

Application webhooks can notify on task lifecycle changes (exact event names depend on Todoist’s current webhook schema). For this project the critical signal is **task completed** (and possibly **updated** if we need to detect recurrence changes caused by the client). The worker filters: only tasks that are **recurring** (or in scope per product decision) and whose **completion timestamp** falls in the grace window when converted to the configured timezone trigger the full correction path.

**Why not webhooks alone:** Delivery can fail, clients can be offline, or Todoist may batch sync. So webhooks are the **fast path**; they are not the only path.

### 5. REST vs Sync API (when to use which)

**REST v2** is appropriate for **reading** active tasks, projects, and labels—simple GETs with pagination if the account is large.

**Sync API** is appropriate when we must **mutate** items the way Todoist’s own clients do: completing, uncompleting, updating due dates and recurrence, and sending **command batches** with client-generated UUIDs so retries are safe.

The technical solution relies on **Sync** for corrections to recurring items because recurrence advancement is modeled there explicitly. REST alone is usually insufficient for nuanced recurrence fixes.

**Relevant Sync concepts (names are API identifiers, not code):** plain completion supports an optional **completed-at** timestamp in UTC. Recurring completion uses a dedicated **update-date-and-complete** style command whose documented arguments focus on the new **due** object and forward flag rather than backdating completion—hence the implementation **spike** to see whether structural fixes (adjusting the next **due**) are the primary tool versus replaying completion.

### 6. Core processing pipeline (technical)

For each qualifying event:

1. **Normalize time:** Convert the completion instant from UTC into the configured **IANA timezone**, accounting for DST rules in that zone.
2. **Classify:** If local time is on or after the boundary (10:00), **no** logical-day correction for the grace-window rule. If local time is before 10:00, treat the completion as belonging to the **previous calendar day’s** logical day (see the logic section below).
3. **Fetch state:** Load the task’s current **due** structure, recurrence **string**, and related fields after Todoist has applied the user’s completion.
4. **Compute expected next occurrence:** From the recurrence semantics and the **logical** completion date (not the raw calendar date of the UTC timestamp alone), derive what the **next due** should be. This may require parsing or delegating to a recurrence library that understands Todoist’s patterns; edge cases include every N days, weekdays only, and month-end rules.
5. **Diff:** Compare expected vs actual **due** (date, time, is_recurring flags as applicable).
6. **Act:** If within tolerance they match, exit. If not, send a Sync **item update** (or the command the spike validates) to set the **due** to the expected value while **preserving** the user-visible time-of-day and recurrence language wherever the API allows.
7. **Record:** Write an idempotency record so the same webhook or a retry does not reapply the same mutation.

**Concurrency:** A single logical **lock per task id** (or queue partition) avoids two workers correcting the same task after rapid edits.

**Idempotency:** Keys derived from webhook delivery id, or a hash of task id + completion time + previous due snapshot.

### 7. Reconciliation job (scheduled)

On a fixed interval (for example five to fifteen minutes) and again shortly after **10:00 AM** local:

- Fetch a candidate set of **active recurring** tasks (or recently completed items via Todoist’s completed-task endpoints if the API and rate limits allow a narrower window).
- For tasks completed since the last checkpoint, re-run the same **classify → compute → diff → act** pipeline.
- This catches webhook gaps and **offline completion** that syncs late.

The post–10 AM run specifically targets “anything that happened overnight while the user was asleep” so the account is consistent before the new logical day is fully underway.

### 8. Fallback strategy (only if needed)

If **item update** cannot reconcile a bad recurrence state, the spike may show that **uncomplete** followed by a recurrence-aware **complete** command is required. That path has UX side effects (brief incorrect UI state, notifications). The technical solution treats it as **opt-in after evidence**, not the default.

### 9. Configuration (deploy-time and runtime)

Typical parameters stored in environment or secure config:

- IANA timezone string for the user.
- Boundary hour and minute (default 10:00).
- Whether non-recurring tasks participate in completion backdating.
- Todoist credentials and webhook secret.
- Optional: list of project ids to include or exclude if scope is narrowed.
- Alert webhook URL or email for operator notifications.

### 10. Non-functional requirements

- **Rate limits:** Todoist enforces per-token limits; the client batches Sync commands and backs off on HTTP 429.
- **Timeouts:** Webhook handler responds fast; long work happens asynchronously.
- **Security:** TLS only, verify webhook signature, secrets not in logs, least-privilege token scopes.
- **Reliability:** At-least-once delivery semantics mean idempotency is mandatory.

### 11. Testing approach (technical, not code)

- **Unit tests** for the rule engine: fixed instants crossing DST, boundary at 10:00, completions at 09:59 vs 10:00.
- **Integration tests** against a **sandbox or personal test project** with synthetic recurring tasks.
- **Dry-run mode** (optional): log “would have updated task X to due Y” without calling Sync, for safe validation in production shadow period.

### 12. Data we persist (logical model)

Nothing heavy is required—this is not a second copy of Todoist.

- **Processed events:** enough to prove “we already handled this webhook or this completion” (idempotency key, optional raw event id, received-at time, outcome: applied / skipped / failed).
- **Reconciliation cursor:** last successful sweep timestamp or sync cursor so the next job does not rescan the entire account every time.
- **Optional audit trail:** task id, old due snapshot, new due applied, reason code (grace-window correction, reconciliation, manual replay), operator-visible for support.
- **Secrets:** stored only in the host’s secret manager (tokens, webhook secret), never in application tables unencrypted.

If the platform supports it, **short TTL** on detailed audit rows keeps privacy and cost down while keeping recent history for debugging.

### 13. Webhook endpoint behavior

- **Verify first:** reject requests that fail signature or timestamp checks (per Todoist’s current webhook security guidance).
- **Respond quickly:** return HTTP success once the payload is validated and **durably queued** (written to a queue table or message broker). Avoid doing Sync calls inside the HTTP request thread.
- **Payload expectations:** map Todoist’s event shape to an internal “completion candidate” record (task id, user id if multi-tenant, completion time if present, event type). If the webhook only carries an id, the worker **always** follows with a REST or Sync read to load full task state before deciding.
- **Ordering:** if multiple events arrive for one task, the queue’s **per-task serialization** preserves last-writer sanity.

### 14. Sync API: batches, retries, and consistency

- **Command UUIDs:** each Sync command carries a client-generated unique id so that **safe retries** do not duplicate work on Todoist’s side when our worker crashes after a successful server write but before we ack the message.
- **Batches:** group independent updates in one request when rate limits and payload size allow; keep **one task’s dependent steps** in order (read → decide → update).
- **Sync token:** after reads that return a sync token, the service may keep it for efficient incremental sync during reconciliation; exact use depends on whether we standardize on REST reads, Sync reads, or a hybrid.
- **Conflict handling:** if an update fails because the task changed again (deleted, edited), the worker **re-fetches** and either recomputes or drops the job with a logged reason instead of looping forever.

### 15. When we deliberately do nothing

The pipeline should **exit early** (with a clear internal reason) when:

- Completion local time is **on or after** the 10:00 boundary—normal Todoist behavior is left alone.
- The task is **not recurring** and product scope says “recurring only.”
- The task lives in an **excluded** project (if that option exists).
- The completion was performed by another user on a **shared** task and policy says “owner only” or “no auto-fix for collaborators.”
- Todoist reports the **next due** already matches expected within agreed tolerance (see below).
- The task is **deleted** or no longer active before the worker runs.

Documenting these branches helps testing and avoids “mystery skips” in logs.

### 16. Comparing “expected” vs “actual” due dates

- **Tolerance:** decide whether comparison is **date-only** (calendar day) or **date-and-time** in local zone, depending on how Todoist returns durations and floating times. The spike should pin this down per field shape in API responses.
- **Floating vs fixed time:** Todoist distinguishes tasks with explicit times from date-only tasks; the rule engine must not “fix” a task by accidentally adding a time component or stripping one.
- **Recurrence string preservation:** after an update, the user-visible recurrence phrase should remain the same unless Todoist’s API forces a canonical form; if the API rewrites the string, that is an acceptable cosmetic change as long as semantics match.

### 17. Subtasks, parents, and light use of hierarchy

The brief notes subtasks are **mostly reference**, not the main recurrence surface. Technically:

- **Recurring parent:** corrections target the **parent item’s** due and recurrence; child items are untouched unless product scope expands.
- **Recurring subtask (rare):** if it exists, the same pipeline applies using that item’s id; no special case unless we discover Todoist nests completion events differently—in that case the webhook mapping step is adjusted once.

### 18. Reconciliation: concrete sourcing strategy

Webhook path is clear; scheduled reconciliation needs a **defined** data source so engineering is not vague:

- **Preferred:** use Todoist’s **completed-items-by-time-range** (or equivalent) REST endpoints to list items completed since the last cursor, filtered to recurring, then run the same classify → compute → diff → act path. Respect API **lookback limits** (Todoist documents maximum ranges for some completed queries).
- **Fallback:** periodic full scan of **active recurring** tasks is possible for small accounts but does not detect “wrong next due” unless we also have completion context; usually the completed feed plus occasional spot-check is enough.
- **Checkpointing:** store “last processed completion window end” in UTC to avoid gaps and double-processing.

### 19. Failure modes and dead letters

- **Transient errors** (network, 5xx, rate limit): retry with exponential backoff and a max attempt count.
- **Permanent errors** (task not found, permission denied): mark job failed, log structured reason, **do not** infinite retry.
- **Repeated failure** on the same task: after N attempts, move to a **dead-letter** queue or table and **alert** the operator so a human can inspect Todoist and logs.
- **Service outage:** when the worker comes back, reconciliation **backfills** using the completion cursor; webhooks that were dropped entirely are still recoverable if the completion falls inside the API’s queryable window.

### 20. Time correctness and clocks

- All **business rules** use the configured IANA timezone library (region database), not fixed UTC offsets, so **DST transitions** are handled on the completion instant.
- Webhook **server timestamps** are trusted only after validation; completion time should come from **Todoist’s payload or authoritative task read**, not from “when we received the HTTP request,” unless the docs guarantee equivalence.
- Document one **policy** for the boundary instant itself: e.g. 10:00:00.000 inclusive on the “new logical day” side, 09:59:59.999 still grace window—tests lock this in.

### 21. Deployment and rollout (technical)

- **Stages:** dev (test token) → staging or personal project → production with **dry-run** or logging-only mode for a few days → full apply.
- **Secrets rotation:** procedure to update API token or webhook secret without downtime (dual-read short window or blue-green deploy).
- **Multi-tenant (future):** if more than one user ever shares one deployment, isolate rows by account id and use **per-user** OAuth tokens; webhook routing may use path prefix or signed tenant id in setup.

### 22. Technical solution — completion checklist

Use this as the “done” definition for the engineering work:

| Area | Done when |
|------|-----------|
| Ingress | HTTPS webhook verified, fast ack, durable queue. |
| Worker | Retries, per-task ordering, idempotency enforced. |
| Rules | Logical day + grace window correct across DST; documented boundary inclusive/exclusive behavior. |
| Todoist integration | Read path and Sync update path proven on real recurring patterns from the spike. |
| Reconciliation | Scheduled jobs run; cursor/backfill recovers missed webhooks. |
| Safety | No-op paths defined; conflicts re-fetch or bail cleanly. |
| Ops | Health check, logs, optional alerts, runbook for token rotation and dead letters. |
| Quality | Unit tests on time logic; integration tests on a throwaway project; dry-run option if agreed. |

That closes the **technical solution** section: enough detail to implement and review without prescribing programming language or deployment vendor.

---

## Recommended shape of the solution (summary)

Todoist doesn’t offer a “my day ends at 10 AM” toggle, so we ship a **small service that stays on 24/7**: webhooks for speed, **Sync API** for precise fixes, **scheduled reconciliation** for reliability, and a **persistent idempotency** layer so retries are safe.

**Why cloud-first:** The brief calls out that the client can’t always open a laptop or run a script. A hosted worker doesn’t care whether a PC is asleep. A local-only daemon remains possible as a backup but is weaker for reliability.

**Spike before locking strategy:** Spend a few hours on a test account with your real recurrence patterns to see whether **backdated completion** is available for recurring items or whether **next-due correction** is the primary mechanism.

---

## How the logic works (logical day and recurrence)

### Figuring out “which logical day is it?”

We fix a timezone and a **10:00 AM** boundary. For any moment in time:

- Convert to local time in that zone.
- If the local clock is **10:00 AM or later**, the logical date is **today’s** calendar date.
- If it’s **before** 10 AM, we still count you as on **yesterday’s** logical date.

So that 1:30 AM completion lands on the same logical day as “yesterday evening”—which is the behavior you’re asking for.

### Tasks that are *due* between midnight and 10 AM

Those **keep their due time**. A task due at 2 AM stays due at 2 AM; we’re not sliding the whole schedule.

What we **can’t** do is redraw Todoist’s built-in **Today** view to use our 10 AM boundary—that’s their product. What we **can** do is keep **recurrence** (and, where the API allows, **completion timestamps**) consistent with the logical day so the *next* instance isn’t wrong.

If you also want **non-recurring** tasks to count toward the previous logical day for stats, say so—that’s optional scope and may use the plain completion path’s optional **completed-at** field where Todoist honors it.

### Avoiding “skipped” recurring days (the main loop)

The approach we’d implement is: **look at what Todoist did, decide if it’s wrong, fix it if needed**—not “preemptively reschedule everything.”

After a completion (from a webhook or from reconciliation), we read the task’s new state. We compute what the **next due** *should* be from the recurrence string and the **logical** completion date, then compare. If Todoist advanced one step too far (or not in the way we expect), we issue a targeted **Sync item update** so the **next occurrence** is right, while keeping the **time of day** and wording of the recurrence as intact as the API allows.

We’d dedupe work with a stable key per task and completion event so duplicate webhooks don’t double-apply. If the same task might get edits from two places at once, we’d serialize updates **per task** so we don’t race ourselves.

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
- **Setup guides** — For a hosted setup, “install on your computer” often isn’t the story: connect Todoist once, register the webhook URL in the developer app, set timezone. If we ship a local option too, we document **Task Scheduler** on Windows and **launchd** on Mac in plain language with screenshots.
- **This kind of doc** — architecture, setup, limitations, troubleshooting.

---

## Local-only option (if you really want it)

Possible, but weaker: a small daemon on your machine, maybe a tunneling service for HTTPS so Todoist can reach you, or polling instead of webhooks (slower and noisier). Autostart via Task Scheduler / Launch Agents. The catch is sleep mode and missed webhooks—we’d lean harder on reconciliation when the machine wakes up. Cloud is still the recommendation for reliability and for **zero** daily interaction.

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

---

This write-up is our best picture of the approach and the tradeoffs. The spike on a real account is where we turn “should work” into “we’ve seen Todoist do X, so we do Y.” Final behavior always has to match what the live API actually returns.
