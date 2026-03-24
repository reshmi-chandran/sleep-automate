# Proposal: Todoist “logical day” boundary (10:00 AM)

**Version:** 1.0  
**Author:** Reshmi Chandran  

This document describes the **technical approach**, **deliverables**, and **scope boundaries** for the Todoist logical-day integration. It is written for the client and maintainers as a single reference—architecture and behavior only, no source code.

---

## Requirements summary

The goal is for completions and recurring-task behavior to align with a **day that ends at 10:00 AM** in the user’s local timezone, not midnight—**without** moving due times to 10 AM as a hack. Approaches that “fixed” rollover by **shifting task times** are **explicitly out of scope**.

Todoist’s product is built around **calendar dates** and **timestamps**. When a routine often runs past midnight, finishing something at 1 AM can still belong to **that** “Monday,” even though Todoist has moved to Tuesday. Recurring tasks are most affected: the **next** occurrence can look like it **skipped** a day, or the user’s rhythm stops matching what the app shows.

This integration uses Todoist’s **APIs** to keep recurrence consistent **within what Todoist exposes**, while preserving user-set scheduled times.

---

## Terminology

**Logical day** — Configured as running from **10:00 AM** one calendar day to **10:00 AM** the next, in **one** selected timezone (for example America/New_York). The rule applies account-wide unless scope is later narrowed (e.g. by project).

**Grace window** — Between **midnight and 10 AM** local time, the user is still in the tail end of the *previous* logical day. Completions in this window drive correction logic.

**Wall-clock due time** — The time shown on the task (“every day at 9 PM,” etc.). **Preserved by design**; the solution does not slide the whole schedule to 10 AM.

---

## In scope and out of scope

**In scope:**

- Detect when **recurring** tasks are **completed** between **midnight and 10 AM** local (per configured timezone).
- Reduce recurrence **skipping** caused by Todoist treating those completions as the “wrong” calendar day.
- One **global** day-boundary rule for the automation (not per-task configuration unless added as extra scope).
- Operation **without daily manual steps** for the user: **webhooks** when something happens, plus a **scheduled sweep** so missed events do not leave bad state.

**Out of scope unless explicitly added:**

- Redesigning Todoist’s UI.
- Per-project day boundaries (possible later; not the default).
- A guarantee that **Karma / streaks** will behave exactly as if Todoist natively supported a 10 AM cutoff—Todoist owns that logic. Short API validation clarifies **what can be corrected in task data** vs **what may still look “off” in the product UI**.

---

## Technical solution — what gets built

This section is the engineering view: **what** runs and **how** it behaves. There are no code listings here—only architecture and behavior.

### 1. Overall pattern

Todoist doesn’t offer a “day ends at 10 AM” setting. The deliverable is an **integration service**: reachable on the public internet over **HTTPS**, registered as a Todoist **application** so it can receive **webhooks** and call Todoist’s **REST** and **Sync** APIs using credentials tied to the **user’s** account.

**Flow in plain language:** The user completes or updates a task in any official Todoist client. Todoist saves the change and, for subscribed events, sends a **signed** webhook to the integration service. The service validates the call, queues work, loads fresh task state when needed, applies **logical-day and recurrence rules**, and sends **updates to Todoist** only when a correction is required. A **scheduled job** repeats a lighter version of the same checks so a missed webhook or delayed sync does not leave a wrong next due date in place.

### 2. Main components

| Component | Role |
|-----------|------|
| **HTTPS webhook endpoint** | Receives Todoist callbacks; verifies signatures; responds quickly (usually by enqueueing work, not doing heavy Sync calls in the HTTP thread). |
| **Queue + worker** | Processes jobs with retries, backoff, and **per-task ordering** so rapid edits don’t interleave incorrectly. |
| **Rule engine** | Pure logic: given completion time (UTC), configured timezone, boundary time, and task metadata (recurrence, dues), decides if a correction is needed and what the target state should be. |
| **Todoist API client** | Handles authenticated REST reads and Sync command batches, rate limits, and transient errors. |
| **Persistent store** | Idempotency keys, optional event ids, minimal audit rows, and configuration (timezone, boundary, flags). |
| **Scheduler** | Periodically reconciles, plus a run shortly **after 10:00 AM** in the configured timezone. |
| **Observability** | Logs, a health URL for uptime checks, optional alerts if corrections keep failing. |

The implementation is not tied to a single cloud vendor; requirements are **HTTPS**, **background workers**, **cron**, and a small database or queue store.

### 3. Authentication and one-time Todoist setup

The service needs a **long-lived credential** to read and update tasks. That is typically either **OAuth** (one-time user authorization; refresh tokens stored encrypted) or a **personal API token** (simpler for a single user; rotate if exposed).

A **Todoist developer application** is registered with the **webhook URL** and **secret** Todoist uses to sign requests. One-time user authorization is required; setup steps are included in the delivery documentation.

### 4. Webhooks

Todoist’s webhook schema names the event types; the important one here is **task completed** (and sometimes **updated**, if required for edge cases). The worker only runs the full correction path for **recurring** tasks (unless scope is expanded) whose **completion time**, interpreted in the **configured** timezone, falls in the **grace window**.

**Why not webhooks alone:** delivery can fail, devices can be offline, sync can lag. Webhooks are the **fast path**; the scheduler is the **safety net**.

### 5. REST vs Sync API

**REST v2** is used for straightforward reads (tasks, projects, etc.).

The **Sync API** handles mutations the way Todoist’s own clients do—especially **recurrence fixes**—including **command batches** with client-generated UUIDs so **retries are safe**.

**Important detail:** plain **complete** supports an optional **completed-at** time in UTC. **Recurring** completion uses a different Sync path whose documented fields focus on the new **due** rather than backdating completion. A **short validation pass on a test account** using real recurrence patterns confirms whether the primary fix is **adjusting the next due** vs. a heavier replay path, so production behavior is evidence-based.

### 6. Core processing pipeline

For each qualifying event:

1. **Normalize time** — Convert completion instant to the configured **IANA** timezone (DST-aware).
2. **Classify** — If local time is **on or after** 10:00, standard Todoist behavior is left unchanged for this rule. If **before** 10:00, the completion is treated as belonging to the **previous calendar day’s** logical day.
3. **Fetch state** — Load the task’s **due** and recurrence fields after Todoist has applied the user’s completion.
4. **Compute expected next occurrence** — From recurrence semantics and the **logical** completion date—**not** the raw calendar date of the timestamp alone.
5. **Compare** — Expected vs actual **due** (within tolerances fixed during API validation).
6. **Correct** — If needed, issue a Sync **item update** (or the command the validation pass confirms) so the **next occurrence** is right, preserving time-of-day and recurrence meaning wherever the API allows.
7. **Record** — Write idempotency data so duplicate webhooks don’t double-apply.

**Concurrency:** one logical **queue lane per task id** avoids races when the user edits quickly.

### 7. Scheduled reconciliation

On a fixed interval (for example every 5–15 minutes) and again shortly after **10:00 AM** in the configured local timezone:

- Pull recently **completed** recurring items (via Todoist’s **completed-by-time-range** style endpoints when suitable), or another strategy fixed during implementation.
- Re-run the same **classify → compute → compare → correct** path.
- **Checkpoint** in UTC to avoid gaps or endless reprocessing.

The post–10 AM run specifically cleans up **overnight** activity before the new logical day is in full swing.

### 8. Fallback (only if validation says it is required)

If a vanilla **due update** cannot fix bad recurrence state, **uncomplete** + recurrence-aware **complete** may be required. That can cause brief UI flicker and notifications, so it is **Plan B**, not the default.

### 9. Configuration

Typical settings in secure configuration:

- **IANA** timezone and **boundary** time (default 10:00).
- Whether **non-recurring** tasks participate (optional add-on).
- Todoist credentials and webhook secret.
- Optional project include/exclude lists.
- Optional alert destination for repeated failures.

### 10. Non-functional requirements

- Respect Todoist **rate limits** (batching, backoff on HTTP 429).
- **Fast** webhook responses; heavy work **off** the request thread.
- **TLS**, webhook **signature verification**, secrets **not** logged.
- **Idempotent** processing (at-least-once delivery from providers is normal).

### 11. Testing (intent only—no source in this document)

- **Unit tests** on the rule engine (DST boundaries, 09:59 vs 10:00).
- **Integration tests** against a **test project** with synthetic recurring tasks.
- Optional **dry-run** mode that logs what would change without applying it, for a low-risk rollout.

### 12. Data stored

This is **not** a full copy of Todoist—only what the integration needs:

- **Idempotency** records for processed work.
- **Reconciliation cursor** / checkpoint.
- Optional **short-lived audit** rows for support.
- **Secrets** only in the host’s secret manager.

### 13. Webhook handler behavior

Verify signatures first; **acknowledge** after the job is **durably queued**; if the payload is minimal, **always** re-read the task from Todoist before deciding.

### 14. Sync retries and consistency

- **UUID per Sync command** so safe retries don’t double-apply on Todoist.
- **Ordered** steps per task (read → decide → write).
- On conflict (task deleted/edited), **re-fetch** and either recompute or exit with a clear log reason.

### 15. When the service intentionally does nothing

The service exits early when:

- Completion is **on or after** 10:00 local (no grace-window correction).
- Task isn’t **recurring** (if that is the agreed scope).
- Project is **excluded** (if that option is used).
- **Shared-task policy** says not to alter collaborator completions (if defined).
- Todoist’s state already **matches** expected within tolerance.
- Task is **gone** before the worker runs.

These branches are documented so logs remain interpretable.

### 16. Comparing dues

API validation pins down whether comparisons are **date-only** vs **date-and-time**, and how **floating** vs **fixed** tasks behave in payloads—so a “fix” does not accidentally change time semantics.

### 17. Subtasks

Subtasks are **mostly reference** in the stated use case. **Recurring parents** are the default correction target; rare recurring subtasks use the same pipeline on that item’s id unless Todoist’s events require a one-time mapping tweak.

### 18. Failure handling

Retries for transient errors; **stop** retrying on permanent errors; **dead-letter / alert** if the same task fails repeatedly; **backfill** after outages using the reconciliation cursor.

### 19. Time policy

Rules use a real timezone database (not a fixed UTC offset). Completion time comes from **Todoist’s data**, not from webhook receipt time, unless the official docs state they are equivalent. Tests document whether **10:00:00** is inclusive on the **new** logical day.

### 20. Rollout

Typical sequence: development → **test project** → production, optionally **log-only** for a few days, then live corrections. Delivery documentation includes **token rotation** so secrets can be replaced without service drama.

### 21. Definition of done (for this build)

| Area | Done when |
|------|-----------|
| Ingress | Webhook verified, fast ack, durable queue. |
| Worker | Retries, per-task ordering, idempotency. |
| Rules | Correct logical day + grace handling across DST; boundary policy documented. |
| Todoist | Read + update paths proven on **real** recurrence patterns from API validation. |
| Reconciliation | Scheduled runs + checkpoint restore missed webhooks. |
| Safety | Conflict handling; clean no-op paths. |
| Ops | Health check, logs, optional alerts, short runbook. |
| Quality | Tests on time logic + integration on a throwaway project; optional dry-run mode. |

---

## Summary of the approach

The design is a **small, always-on service**: **webhooks** for speed, **Sync API** for precise recurrence fixes, **scheduled reconciliation** for reliability, and **idempotency** so retries are safe.

**Cloud hosting** is recommended: it avoids depending on a PC being on or on scripts run by hand. A **local-only** fallback is possible but **less reliable** for webhooks and machine sleep.

**API validation:** a short test-account pass using **real** recurrence patterns makes the correction strategy evidence-based, not theoretical.

---

## How the rules interpret the logical day

- Before **10 AM** local → logical date is still **yesterday’s** calendar date for boundary purposes.
- **On or after 10 AM** → logical date is **today’s** calendar date.

So a **1:30 AM** completion aligns with **yesterday evening** for how the **next** recurring instance is computed.

**Tasks due between midnight and 10 AM** keep that due time. The integration **cannot** change how Todoist renders the official **Today** screen; it **can** align task **data** (next due / completion metadata where the API allows).

**Non-recurring tasks** — inclusion for **completion attribution** is **optional scope** (confirm under Scope confirmations).

**Main correction loop:** observe what Todoist did after completion → compute what **should** have happened for the **logical** day → if wrong, apply a **minimal** Sync update. **Webhook deduplication** and **per-task serialization** prevent double application and races.

---

## Expected reliability

Under normal conditions, corrections within **a few minutes** of a completion. Overnight gaps should be cleaned up **not long after 10 AM** local via reconciliation. Optional **email or Slack** when failures repeat.

---

## Limitations (explicit)

- Todoist **Today / home** may still **feel** midnight-based; the integration corrects **data** where the API allows.
- **Karma / streaks** may not perfectly match the logical day; early validation sets achievable expectations.
- **Shared projects:** policy should state whether only the **account owner’s** completions are adjusted.
- **Offline devices** cause delay until sync—reconciliation covers that gap.
- **APIs evolve**—ongoing maintenance (if engaged) includes watching Todoist developer notes.

---

## Security and privacy

Least-privilege access, verified webhooks, minimal sensitive data in logs (prefer **IDs**). **GDPR** or **data residency** requirements inform hosting choice.

---

## Deliverables

- **Working service** — deployed integration, webhook configured, scheduler, small store for dedupe/checkpoints.
- **Setup guides** — step-by-step for **Windows** and **Mac** where relevant; for a hosted setup, that usually means **authorize Todoist once**, paste the webhook URL, set timezone—not “run a script every morning.”
- **Documentation** — approach (like this), setup, limitations, troubleshooting.

---

## Local-only option

A daemon on the user’s machine plus HTTPS tunneling, or polling instead of webhooks. Feasible but **weaker** when the machine sleeps; reconciliation after wake carries more load. **Cloud hosting remains the recommended default** for accessibility and reliability.

---

## Ongoing maintenance (optional)

Periodic review of Todoist API / webhook changes, dependency updates, and tuning after incidents. Scoped separately from the MVP.

---

## Effort and timeline

**Target: 40 hours** for an **MVP**: single account, **in-use** recurrence patterns, hosted **happy path**, without large add-ons (deep shared-project policy, multi-timezone travel logic, exhaustive edge-case polish). A second phase can cover deferred items.

The **40 hours** include a **Todoist test account** and hands-on API validation, time-boxed within that budget.

| Phase | Hours (approx.) |
|-------|-----------------|
| API validation: recurring completion / Sync behavior on a test account | 4 |
| Service skeleton: auth, webhook, secrets, minimal hosting | 8 |
| Logical-day logic + recurrence correction + focused tests | 18 |
| Reconciliation, basic monitoring, idempotency | 5 |
| Setup guide + short runbook | 3 |
| Buffer (DST / failure-mode pass / small fixes) | 2 |
| **Total** | **40** |

**Calendar time:** often **about 1–2 weeks**, depending on engineering availability and how quickly questions or test-project access can be answered.

The **40-hour** line item assumes the scope described in this document (MVP). Work outside that scope is handled as a separate estimate or change request, with **in** / **out** items listed so expectations stay explicit.

---

## Scope confirmations

Confirm with the client / product owner:

- Should **one-off** tasks follow the same completion-day rules, or **recurring only**?
- Is **Karma / streak** alignment a must-have, or is **correct next occurrence** the main success metric?
- **Solo** account vs significant **shared** task use with others?
- Hosting **region** and **alerting** if the service is unhealthy?
- Mostly **one home timezone**, or frequent travel across zones?

---

## References

- [Todoist Developer](https://developer.todoist.com) — REST v2, Sync API, webhooks.

---

This document reflects Todoist’s **published APIs** and must be validated against a **live test account**. **Final behavior must match what the API actually returns**, not what any paper design assumes. If validation contradicts this document, update the approach or scope before implementation proceeds on that basis.
