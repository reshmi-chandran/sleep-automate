# Proposal: Todoist “logical day” boundary (10:00 AM)

**Version:** 1.5  
**Prepared for:** [Client name — fill in]  
**Prepared by:** [Your name — fill in]  

This document describes **the technical approach I propose**, what **you** can expect in delivery, and **what stays out of scope** for the engagement. I’m sharing it so you have a clear picture before we lock details—without diving into source code.

---

## What you asked for (in my words)

You want completions and recurring-task behavior to line up with a **day that ends at 10:00 AM** in your local timezone, not midnight—**without** moving all your due times to 10 AM. You’ve already seen approaches that “fixed” rollover by shifting task times; **that is not what I’m building.**

Todoist’s product is built around **calendar dates** and **timestamps**. If your routine often runs past midnight, finishing something at 1 AM can still belong to **your** “Monday,” even though Todoist has moved to Tuesday. Recurring tasks suffer most: the **next** occurrence can look like it **skipped** a day, or your rhythm stops matching what the app shows.

My job is to use Todoist’s **APIs** to keep recurrence sane **within the limits of what Todoist exposes**, while leaving your scheduled times where you set them.

---

## Terms I’ll use in this document

**Logical day** — I’ll treat your day as running from **10:00 AM** one calendar day to **10:00 AM** the next, in **one** timezone you choose (for example America/New_York). This rule applies account-wide unless you and I later agree to narrow scope (e.g. by project).

**Grace window** — Between **midnight and 10 AM** local time, you’re still in the tail end of the *previous* logical day. Those are the completions I focus on for correction logic.

**Wall-clock due time** — The time shown on the task (“every day at 9 PM,” etc.). **I intend to preserve this**; I’m not proposing to slide your whole schedule to 10 AM.

---

## Scope I’m committing to (and what I’m not)

**In scope:**

- Detect when **recurring** tasks are **completed** between **midnight and 10 AM** local (per your configured timezone).
- Reduce recurrence **skipping** caused by Todoist treating those completions as the “wrong” calendar day.
- One **global** day-boundary rule for the automation (not per-task configuration unless we add it as extra scope).
- Operation **without daily manual steps** on your side: **webhooks** when something happens, plus a **scheduled sweep** so missed events don’t leave bad state.

**Out of scope unless we add it explicitly:**

- Redesigning Todoist’s UI.
- Per-project day boundaries (possible later; not the default).
- A guarantee that **Karma / streaks** will behave exactly as if Todoist natively supported a 10 AM cutoff—Todoist owns that logic. After a short API validation on your patterns, I’ll tell you **what I can fix in task data** vs **what may still look “off” in the app**.

---

## Technical solution — what I will build

This section is the engineering view: **what** runs and **how** it behaves. There are no code listings here—only architecture and behavior.

### 1. Overall pattern

Todoist doesn’t offer a “day ends at 10 AM” setting. So I’ll deliver an **integration service**: reachable on the public internet over **HTTPS**, registered as a Todoist **application** so it can receive **webhooks** and call Todoist’s **REST** and **Sync** APIs using credentials tied to **your** account.

**Flow in plain language:** You complete or update a task in any official Todoist client. Todoist saves the change and, for the events we subscribe to, sends a **signed** webhook to **my service**. The service validates the call, queues work, loads fresh task state when needed, applies **logical-day and recurrence rules**, and sends **updates to Todoist** only when a correction is required. A **scheduled job** repeats a lighter version of the same checks so a missed webhook or delayed sync doesn’t strand you with a wrong next due date.

### 2. Main components

| Component | Role |
|-----------|------|
| **HTTPS webhook endpoint** | Receives Todoist callbacks; verifies signatures; responds quickly (usually by enqueueing work, not doing heavy Sync calls in the HTTP thread). |
| **Queue + worker** | Processes jobs with retries, backoff, and **per-task ordering** so rapid edits don’t interleave incorrectly. |
| **Rule engine** | Pure logic: given completion time (UTC), your timezone, boundary time, and task metadata (recurrence, dues), decides if a correction is needed and what the target state should be. |
| **Todoist API client** | Handles authenticated REST reads and Sync command batches, rate limits, and transient errors. |
| **Persistent store** | Idempotency keys, optional event ids, minimal audit rows, and configuration (timezone, boundary, flags). |
| **Scheduler** | Periodically reconciles, plus a run shortly **after 10:00 AM** in your timezone. |
| **Observability** | Logs, a health URL for uptime checks, optional alerts if corrections keep failing. |

I’m not locked to one cloud vendor; what matters is **HTTPS**, **background workers**, **cron**, and a small database or queue store.

### 3. Authentication and one-time Todoist setup

The service needs a **long-lived credential** to read and update your tasks. That’s typically either **OAuth** (you authorize once; I store refresh tokens securely) or a **personal API token** (simpler for a single user; you’d rotate it if it were ever exposed).

Separately, I’ll register a **Todoist developer application** with the **webhook URL** and **secret** Todoist uses to sign requests. You’ll need to complete authorization once; I’ll document the exact clicks.

### 4. Webhooks

Todoist’s webhook schema names the event types; the important one here is **task completed** (and sometimes **updated**, if we need it for edge cases). The worker only runs the full correction path for **recurring** tasks (unless we expand scope) whose **completion time**, interpreted in **your** timezone, falls in the **grace window**.

**Why not webhooks alone:** delivery can fail, devices can be offline, sync can lag. Webhooks are the **fast path**; the scheduler is the **safety net**.

### 5. REST vs Sync API

I’ll use **REST v2** for straightforward reads (tasks, projects, etc.).

I’ll use the **Sync API** for mutations that match how Todoist’s own clients behave—especially **recurrence fixes**—including **command batches** with client-generated UUIDs so **retries are safe**.

**Important detail:** plain **complete** supports an optional **completed-at** time in UTC. **Recurring** completion uses a different Sync path whose documented fields focus on the new **due** rather than backdating completion. I’ll run a **short spike on a test account** with **your** recurrence patterns to confirm whether the primary fix is **adjusting the next due** vs. a heavier replay path—so I don’t guess in production.

### 6. Core processing pipeline

For each qualifying event:

1. **Normalize time** — Convert completion instant to your **IANA** timezone (DST-aware).
2. **Classify** — If local time is **on or after** 10:00, I leave standard Todoist behavior alone for this rule. If **before** 10:00, I treat the completion as belonging to the **previous calendar day’s** logical day.
3. **Fetch state** — Load the task’s **due** and recurrence fields after Todoist has applied your completion.
4. **Compute expected next occurrence** — From recurrence semantics and the **logical** completion date—**not** the raw calendar date of the timestamp alone.
5. **Compare** — Expected vs actual **due** (within tolerances we define during the spike).
6. **Correct** — If needed, issue a Sync **item update** (or the command the spike validates) so the **next occurrence** is right, preserving time-of-day and recurrence meaning wherever the API allows.
7. **Record** — Write idempotency data so duplicate webhooks don’t double-apply.

**Concurrency:** one logical **queue lane per task id** avoids races when you edit quickly.

### 7. Scheduled reconciliation

On a fixed interval (for example every 5–15 minutes) and again shortly after **10:00 AM** local:

- Pull recently **completed** recurring items (via Todoist’s **completed-by-time-range** style endpoints when suitable), or another strategy we lock in during implementation.
- Re-run the same **classify → compute → compare → correct** path.
- **Checkpoint** in UTC so I don’t leave gaps or reprocess endlessly.

The post–10 AM run is specifically to clean up **overnight** activity while you’re not on a laptop.

### 8. Fallback (only if the spike says we need it)

If a vanilla **due update** can’t fix bad recurrence state, we may need **uncomplete** + recurrence-aware **complete**. That can cause brief UI flicker and notifications, so I’ll treat it as **Plan B**, not the default.

### 9. Configuration

Typical settings I’ll support in secure config:

- Your **IANA** timezone and **boundary** time (default 10:00).
- Whether **non-recurring** tasks participate (if you want that add-on).
- Your Todoist credentials and webhook secret.
- Optional project include/exclude lists.
- Optional alert destination if something keeps failing.

### 10. Non-functional requirements I’ll meet

- Respect Todoist **rate limits** (batching, backoff on HTTP 429).
- **Fast** webhook responses; heavy work **off** the request thread.
- **TLS**, webhook **signature verification**, secrets **not** logged.
- **Idempotent** processing (at-least-once delivery from providers is normal).

### 11. Testing (no code here—just intent)

- **Unit tests** on the rule engine (DST boundaries, 09:59 vs 10:00).
- **Integration tests** against a **test project** with synthetic recurring tasks.
- Optional **dry-run** mode that logs what would change without applying it, if you want a quiet rollout.

### 12. Data stored

This is **not** a full copy of Todoist—only what the integration needs:

- **Idempotency** records (what I already processed).
- **Reconciliation cursor** / checkpoint.
- Optional **short-lived audit** rows for support.
- **Secrets** only in the host’s secret manager.

### 13. Webhook handler behavior

Verify signatures first; **ack** after the job is **durably queued**; if the payload is thin, **always** re-read the task from Todoist before deciding.

### 14. Sync retries and consistency

- **UUID per Sync command** so safe retries don’t double-apply on Todoist.
- **Ordered** steps per task (read → decide → write).
- On conflict (task deleted/edited), **re-fetch** and either recompute or exit with a clear log reason.

### 15. When I intentionally do nothing

The service exits early when:

- Completion is **on or after** 10:00 local (no grace-window correction).
- Task isn’t **recurring** (if that’s the agreed scope).
- Project is **excluded** (if we use that option).
- **Shared-task policy** says not to touch collaborator completions (if we define that).
- Todoist’s state already **matches** expected within tolerance.
- Task is **gone** before the worker runs.

I’ll document these paths so logs stay understandable.

### 16. Comparing dues

During the spike I’ll pin down whether comparisons are **date-only** vs **date-and-time**, and how **floating** vs **fixed** tasks behave in API payloads—so I don’t “fix” a task by accidentally changing its time semantics.

### 17. Subtasks

You noted subtasks are **mostly reference**. I’ll target **recurring parents** by default; rare recurring subtasks use the same pipeline on that item’s id unless Todoist’s events require a one-time mapping tweak.

### 18. Failure handling

Retries for transient errors; **stop** retrying on permanent errors; **dead-letter / alert** if the same task fails repeatedly; **backfill** after outages using the reconciliation cursor.

### 19. Time policy

Rules use a real timezone database (not a fixed UTC offset). Completion time comes from **Todoist’s data**, not “when my server happened to receive the webhook,” unless their docs say those are equivalent. I’ll lock whether **10:00:00** is inclusive on the **new** logical day and test it.

### 20. Rollout

I plan: dev → your test project → production, optionally **log-only** for a few days, then live corrections. I’ll document **token rotation** so we don’t need a drama if a secret is replaced.

### 21. Definition of done (for this build)

| Area | Done when |
|------|-----------|
| Ingress | Webhook verified, fast ack, durable queue. |
| Worker | Retries, per-task ordering, idempotency. |
| Rules | Correct logical day + grace handling across DST; boundary policy documented. |
| Todoist | Read + update paths proven on **your** recurrence patterns from the spike. |
| Reconciliation | Scheduled runs + checkpoint restore missed webhooks. |
| Safety | Conflict handling; clean no-op paths. |
| Ops | Health check, logs, optional alerts, short runbook. |
| Quality | Tests on time logic + integration on a throwaway project; dry-run if you want it. |

---

## Summary of the approach

I’ll run a **small, always-on service**: **webhooks** for speed, **Sync API** for precise recurrence fixes, **scheduled reconciliation** for reliability, and **idempotency** so retries are safe.

**Why I’m recommending cloud hosting:** it matches your need to avoid depending on a PC being on or scripts being run by hand. A **local-only** fallback is possible but **less reliable** for webhooks and sleep mode.

**Spike:** I’ll spend focused time on a test account using **your** real recurrence styles so the correction strategy is evidence-based, not theoretical.

---

## How the rules interpret “your” day

- Before **10 AM** local → logical date is still **yesterday’s** calendar date for boundary purposes.
- **On or after 10 AM** → logical date is **today’s** calendar date.

So a **1:30 AM** completion counts with **yesterday evening** in terms of how I compute the **next** recurring instance.

**Tasks due between midnight and 10 AM** keep that due time. I **cannot** change how Todoist paints the official **Today** screen; I **can** push task **data** (next due / completion metadata where allowed) toward consistency.

**Non-recurring tasks** — tell me if you want them included for **completion attribution**; that’s add-on scope.

**Main correction loop:** observe what Todoist did after you completed a task → compute what **should** have happened for your **logical** day → if wrong, apply a **minimal** Sync update. I’ll dedupe webhook retries and avoid racing myself on the same task id.

---

## Reliability you can expect

Under normal conditions, corrections within **a few minutes** of a completion. Overnight gaps should be cleaned up **not long after 10 AM** local via reconciliation. Optional **email or Slack** if something keeps failing.

---

## Limitations I want explicit in writing

- Todoist **Today / home** may still **feel** midnight-based; I’m correcting **data** where the API allows.
- **Karma / streaks** may not perfectly match your logical day; I’ll validate what’s achievable early.
- **Shared projects:** you and I should agree whether I only adjust **your** completions.
- **Offline devices** cause delay until sync—reconciliation is there for that.
- **APIs evolve**—I’ll monitor Todoist developer notes as part of ongoing care if you engage me for maintenance.

---

## Security and privacy

Least-privilege access, verified webhooks, minimal sensitive data in logs (prefer **IDs**). If **GDPR** or **data residency** matters to you, I’ll factor that into hosting choice.

---

## What I will deliver

- **Working service** — deployed integration, webhook configured, scheduler, small store for dedupe/checkpoints.
- **Setup guides** — step-by-step for **Windows** and **Mac** where relevant; for a hosted setup, that usually means **authorize Todoist once**, paste the webhook URL, set timezone—not “run a script every morning.”
- **Documentation** — approach (like this), setup, limitations, troubleshooting.

---

## Local-only option (if you insist)

A daemon on your machine plus a tunnel for HTTPS, or polling instead of webhooks. Possible, but **weaker** when the machine sleeps. I’d still rely heavily on reconciliation after wake. **Cloud remains my recommendation** for your situation.

---

## Ongoing maintenance (optional engagement)

Periodic check of Todoist API / webhook changes, dependency updates, and tuning after incidents. We can scope this separately.

---

## Effort and timeline (this proposal)

**Target: 40 hours** for an **MVP**: your account, the recurrence patterns you actually use, hosted **happy path**, without large add-ons (deep shared-project policy, multi-timezone travel logic, exhaustive edge-case polish). Phase two can cover anything we defer.

Those 40 hours still include a **real Todoist test account** and hands-on validation—I’m **not** skipping the API spike, just keeping it tight.

| Phase | Hours (approx.) |
|-------|-----------------|
| Spike: recurring completion / Sync behavior on a test account | 4 |
| Service skeleton: auth, webhook, secrets, minimal hosting | 8 |
| Logical-day logic + recurrence correction + focused tests | 18 |
| Reconciliation, basic monitoring, idempotency | 5 |
| Your setup guide + short runbook | 3 |
| Buffer (DST / failure-mode pass / small fixes) | 2 |
| **Total** | **40** |

**Calendar time:** often **about 1–2 weeks**, depending on how dense my days are and how quickly you can answer questions or try a test project.

If we agree on a **fixed price**, I’ll list what is **in** and **out** of those 40 hours so expectations stay aligned.

---

## Questions for you before we finalize scope

- Should **one-off** tasks follow the same completion-day rules, or **recurring only**?
- Is **Karma / streak** alignment a must-have, or is **correct next occurrence** the main success metric?
- **Solo** account vs significant **shared** task use with others?
- Hosting **region** preference and how you’d like **alerts** if the service is unhealthy?
- Mostly **one home timezone**, or frequent travel across zones?

---

## References

- [Todoist Developer](https://developer.todoist.com) — REST v2, Sync API, webhooks.

---

I’m basing this on Todoist’s published APIs and on a validation pass I’ll run against a live test account; **final behavior always has to match what the API actually does**, not what any document assumes. If anything here needs to change once I’ve run the spike, I’ll call that out clearly and adjust scope or estimates with you before building on a shaky assumption.
