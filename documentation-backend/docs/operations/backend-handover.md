# Backend Handover Runbook

Everything in the backend that needs a human, written for someone who did not build it.

!!! info "Status of this page"
    **Verified against production on 25 August 2026.** Resource shapes, schedules, the
    auto-apply cadence and its execution history were read from GCP, not inferred from code.
    Anything still unverified is listed under [Open questions](#open-questions) — that
    section is part of the document, not an appendix.

    Where a value is marked `code default` it was read from the source and **not** confirmed
    against the live revision.

---

## Start here

| You are… | Go to |
| --- | --- |
| On call, something looks broken | [Triage ladder](#triage-ladder), then the matching [playbook](#playbooks) |
| Asked "why didn't this operator's prices update?" | [Playbook: one operator's prices did not move](#p1) |
| Deciding whether you're allowed to do something | [What you may do without me](#authority) |
| Looking for background, not an incident | [What is deployed](#what-is-deployed) |
| Wondering who owns a thing | [Escalation](#escalation) |

---

## What you may do without me {: #authority }

The point of this section is that nobody waits on me for something that is already decided.

| Action | Clearance | Why |
| --- | --- | --- |
| Flip a single-vendor kill switch (`BOKUN_AUTO_APPLY_DISABLED`) | **Go ahead** | Isolates one degraded vendor; the rest of the batch keeps running |
| Roll back the API service to the previous revision | **Go ahead** | Cloud Run keeps previous revisions live; it is one command and reversible |
| Re-run a digest or snapshot via its `/admin/**/run` route | **Go ahead** | Same payload as the cron, no side effect beyond re-sending |
| Lower `AUTO_APPLY_TASKS_COUNT` or vendor concurrency | **Go ahead** | Always safe in that direction — it protects the database |
| Re-trigger the auto-apply batch by hand | **Go ahead** | Idempotent per slot, and the manual lock protects hand-pushed prices |
| Disable the auto-apply batch entirely | **Ask first** | Prices stop moving for every operator, silently |
| Widen `SELF_COMPARE_ALLOWED_CHANNELS` | **Ask first** | This is how self-referential pricing loops start |
| Run a Prisma migration against production | **Ask first** | The live schema has drifted from `migrations` |
| Change a vendor's pricing-rule semantics | **Ask first** | Product decision, not an operational one |
| Contact a vendor's support on our behalf | **Ask first** | Unless it is a straight "is your API down" question |
| Turn off `AUTO_APPLY_RESPECT_OWNER_TOGGLE` | **Never** | It pushes prices for operators who explicitly opted out |

---

## Triage ladder {: #triage-ladder }

The failure mode nobody watches is silence. A cron that stops firing and a batch nobody
triggers both look exactly like a quiet day. Steps 1 and 2 exist because of that.

1. **Read the price-alerts Slack channel.** Every batch posts a health digest when it ends:
   per-vendor applied/failed, p50/p95 push duration, top error classes, queue depth, and a
   `GREEN` / `YELLOW` / `RED` verdict. **No digest at all is the louder signal** — the batch
   never ran, or Redis was down when it finished.
2. **Confirm the batch is still firing.** It should run 8× a day, roughly every 3 hours. One
   command, [below](#verify-the-batch). If the last execution is more than ~4 hours old,
   go to [playbook P2](#p2).
3. **Confirm the API service has a warm instance.** `minScale: 1` is load-bearing — every
   `@Cron` runs there and nowhere else.
4. **Then Sentry, then Cloud Logging.** The Job's image skips the sourcemap upload, so its
   traces are unsymbolicated: you get the error, not the line. For batch work, filter logs on
   the job name and the `traceId` the `202` returned.
5. **Check `feature_trace`** for the user-visible answer — every meaningful action writes a
   row with input, response, status, duration and business steps. Fastest way to answer *"did
   this operator's push actually happen"* without reading logs.
6. **Only then reach for a kill switch.** Prefer disabling one vendor over stopping the batch.

---

## Playbooks {: #playbooks }

### P1 — One operator's prices did not move {: #p1 }

**Symptom.** An operator, or someone in CS, says prices are not updating.

1. Check `feature_trace` for that subscription. If there is a row with a failure, you have
   your answer and the vendor's error is in it.
2. If the push **succeeded** on our side, the wait is probably the channel's. Direct is
   instant, Viator is seconds, **GetYourGuide is about an hour and up to 24**. Say so — this
   removes a whole category of report.
3. If there is no row at all, the recommendation was never applied. Check in order: is the
   compset owner's auto-apply toggle on (`AUTO_APPLY_RESPECT_OWNER_TOGGLE`); was the slot
   under an `AutoApplyManualLock` from a recent hand-push; did the OTA-source guard skip it
   because the price came from an OTA and the target is Direct.
4. Bokun specifically: **roughly 1 change in 7 never lands**, cause unrecorded. If it is
   Bokun and there is no error, this is the known gap, not a new bug.

### P2 — The batch has not run {: #p2 }

**Symptom.** No health digest, or the last execution is hours old.

The batch is triggered by a **Python caller inside GCP that is not in this repository** —
verified as `python-requests/2.34.2` from Google Cloud address space, POSTing every ~3 hours
and receiving `202` every time. Nothing in the backend schedules it.

1. Verify when it last ran → [command](#verify-the-batch).
2. If executions stopped but the service is healthy, **the caller stopped**, not us. This is
   a data-team question — the backend cannot restart it.
3. You can trigger one by hand in the meantime: `POST /ventrata/recommendations/auto-apply`
   with the `compsetIds` you want. It returns `202` plus an execution name.
4. There is **no alert** on a batch that never starts. If you fix nothing else during my
   leave, this is the one worth adding.

### P3 — The batch runs but takes hours, or two run at once {: #p3 }

This is a known live problem, measured over 29 executions on 21–25 August 2026:

| Metric | Value |
| --- | --- |
| Cadence | every ~3 h, 8× per day, always `202` |
| Median duration | ≈ 88 min |
| Fastest / slowest | 50 min / **6 h 43 min** |
| Exceeded the 4 h task ceiling and retried | **2 of 29** |
| Overlapped the following execution | **6 of 29** |

Cause: the Job is running **unsharded**. Its spec is `taskCount: 1` with `parallelism: 5`,
so one task does all the work, even though the code supports hash-sharding a batch across N
tasks via `CLOUD_RUN_TASK_INDEX` / `_COUNT`.

**Consequence.** Long runs cross into the next trigger, so two batches push prices
concurrently, and a run past 4 h is killed and retried from the start.

**Mitigation.** Raise the effective task count so the batch shards (the caller may pass it,
or set `AUTO_APPLY_TASKS_COUNT` on the Service). Watch pool-wait time in the digest when you
do — the Job talks to Postgres unpooled, so more tasks means more raw connections.

### P4 — Bokun is failing {: #p4 }

Bokun degrades more than the others: **12.1 s median push, 17× Ventrata.**

A three-stage breaker already limits the damage: per-experience quarantine at 4 consecutive
5xx → global half-open cooldown of 30 s once cumulative 5xx crosses 10 → hard stop after 3
failed probes. You will see `[Bokun CIRCUIT]` lines in the logs.

If the breaker is tripping repeatedly, set `BOKUN_AUTO_APPLY_DISABLED=true`. The rest of the
batch keeps running. Re-enable when their gateway recovers.

### P5 — A schedule stopped firing {: #p5 }

Every `@Cron` returns early when `RUN_WORKER === 'true'` and therefore runs **only** in the
API service, kept alive by `minScale: 1`.

If schedules went quiet, check the live revision for `minScale: 0` or a `RUN_WORKER=true`
that should not be there. There is no missing-heartbeat alert — a stopped cron produces no
error, no Sentry event, nothing.

### P6 — Notifications and digests stopped, API is fine {: #p6 }

`main.ts` wraps the Redis connection in try/catch: it logs the failure and **continues
booting without a queue**. The API keeps serving normally while every queued side effect —
emails, Slack posts, the end-of-batch digest — silently stops being produced.

Bull Board is commented out in `main.ts`, so there is no queue UI. Inspect Redis directly.

---

## What is deployed {: #what-is-deployed }

Verified 25 August 2026. All built from this repository. Service and Worker share one image
and differ only by `RUN_WORKER`; the Job is a second image from `Dockerfile.job` whose `CMD`
is the Nest application context, not the HTTP server.

`walkwaysaasbackend` — Cloud Run Service, public
:   NestJS + Fastify API. Three containers per revision: Cloud SQL Auth Proxy →
    pgbouncer `:6432` → Nest `:3000`. **All `@Cron` handlers fire here.**
    ✅ Verified: minScale 1, maxScale 5, 4 vCPU / 4 Gi.

`walkwaysaasbackend-worker` — Cloud Run Service, private
:   Same image as the API, selected by `RUN_WORKER=true`. Drains the BullMQ
    `notifications` queue: emails, Slack posts, the end-of-batch health digest.

`walkway-auto-apply` — Cloud Run **Job**
:   Runs the auto-apply batch off the user-facing service, so it survives what
    would otherwise kill it: OOM kills, a revision rollover mid-push, the HTTP
    timeout ceiling. ✅ Verified: `taskCount: 1`, `parallelism: 5`, 4 h task
    timeout, max-retries 1.

`walkwaysaasbackend-dev` — Cloud Run Service
:   The dev environment. Read the warning below before assuming a refresh ran
    against production.

!!! warning "Dev and prod share the scheduler pool"
    Many of the runtime-created per-compset refresh jobs point at
    **`walkwaysaasbackend-dev`**, not prod. When a refresh "did nothing" in production,
    check which host its scheduler job actually targets before debugging the backend.

### Connections

Runtime queries go through pgbouncer in transaction mode
(`connection_limit=1&pgbouncer=true`, pool 76, 500 max clients). Migrations and admin
scripts use `DATABASE_DIRECT_URL` to bypass the pooler.

!!! warning "The Job has no proxy sidecar"
    The Cloud Run Job reaches Postgres directly and unpooled. If pool-wait time climbs in the
    digest, lower the task count before anything else.

### Deploys

Cloud Build deploys all three on push to `main`; the trigger lives in the GCP console as
inline YAML, **not in the repo**, so it cannot be reviewed in a PR. The Job image is tagged
with the commit SHA — at time of writing it matched the `main` tip, which is how you confirm
the batch is running current code.

---

## Scheduled work

Three different mechanisms schedule things. Confusing them costs hours.

### 1. In-app `@Cron` handlers

Fire in the **API service only**. Times below are UTC; handlers declared without a
`timeZone` run in the container's zone, which is UTC on Cloud Run.

| Handler | Expression | Declared zone | ≈ UTC | Does | Gate |
| --- | --- | --- | --- | --- | --- |
| Support inbox poll | `*/2 * * * *` | container | continuous | IMAP scan → support issues, first-response times | `SUPPORT_INBOX_ENABLED` |
| Expire recommendations | `0 * * * *` | container | hourly | Marks lapsed recommendations expired | — |
| Prune price history | `0 2 * * *` | container | 02:00 | Deletes history older than 90 days | — |
| Min/max signal detection | `0 3 * * *` | container | 03:00 | ENG-1130 floor/ceiling signals | — |
| Auto-Pilot weekly digest | `0 9 * * 1` | Europe/Paris | 07:00 Mon | Slack digest | — |
| Auto-Pilot follow-up | `0 10 * * *` | Europe/Paris | 08:00 | Chases accounts that activated but never pushed | — |
| Operator health scorecard | `0 10 * * 1` | Europe/Paris | 08:00 Mon | Portfolio score → Slack | — |
| Daily report | `0 9 * * *` | container | 09:00 | Yesterday's recommendation activity | — |
| Adoption funnel | `0 11 * * 1` | Europe/Paris | 09:00 Mon | Weekly funnel snapshot | — |
| HTA price snapshot | `0 9 * * *` | America/New_York | 13:00 | ENG-613 daily price email | recipients hardcoded in config |

!!! note "DST and collisions"
    The Paris and New York handlers shift by one hour in the UTC column at the end of
    October. Two handlers already collide at 08:00 UTC on Mondays.

### 2. External Cloud Scheduler jobs that call us

Not in this repo. They will keep firing whatever happens to the codebase.

| Job | Schedule | Hits |
| --- | --- | --- |
| `saas-notifications` | `0 13 * * 1-7` Europe/Paris | `POST /notifications/trigger-daily-notifications` |
| *(the auto-apply trigger)* | every ~3 h — **not a Cloud Scheduler job**, see [P2](#p2) | `POST /ventrata/recommendations/auto-apply` |

### 3. Per-compset refresh jobs, created by the app at runtime

The application creates one Cloud Scheduler job per compset on-demand refresh setup, calling
back into itself with an OIDC token, at whatever hour and timezone the operator picked.
There are on the order of a hundred of them. They are **not in this repo** — list them in the
console, never expect to find them in code. Endpoints hit:
`/api/compset/on-demand-trigger` and `/api/compset/refresh`.

---

## Auto-apply: the batch that moves money

!!! danger "Nothing in this repository schedules it"
    An external Python caller inside GCP POSTs to `/ventrata/recommendations/auto-apply`
    every ~3 hours. The Service then dispatches a Cloud Run Job execution using its own
    runtime service account. If that caller stops, **no prices move and no alert fires.**

1. **External POST** — `POST /ventrata/recommendations/auto-apply` with `compsetIds`. The only
   entry point. It lives on the Ventrata controller for historical reasons and dispatches to
   every vendor.
2. **Dispatch to the Job** — with `AUTO_APPLY_VIA_CLOUD_RUN_JOB=true` the Service fires a
   `walkway-auto-apply` execution through the Cloud Run Admin API and returns **202 with the
   execution name**. Payload rides as a `PAYLOAD_JSON` env override alongside `TRACE_ID`.
3. **Shard** — *in principle.* Each task hashes the shared compset list down to its own shard
   using `CLOUD_RUN_TASK_INDEX` / `_COUNT`. **In production this is not happening**:
   `taskCount: 1`. See [P3](#p3).
4. **Push, vendor by vendor** — Ventrata first, then Bokun, Xola and Prioticket as parallel
   sub-batches keyed on `products.channel`. Guards per recommendation: the compset-owner
   toggle, the OTA-source guard that stops OTA-sourced prices reaching Direct, and
   `AutoApplyManualLock`, which protects any slot a human just pushed by hand.
5. **Record** — every push lands in the vendor's `*_price_changes` table with its duration,
   the vendor's HTTP time, and `pushOrigin` = `AUTO` here (`MANUAL` for API, bulk, undo,
   legacy).
6. **Digest** — a BullMQ job aggregates and posts to Slack. Needs Redis up.

### Verify the batch {: #verify-the-batch }

```bash
gcloud run jobs executions list --job=walkway-auto-apply \
  --project=ww-da-ingestion --region=us-central1 --limit=10 \
  --format="table(metadata.name,status.startTime,status.completionTime)"
```

Healthy looks like: a new execution every ~3 h, each finishing in well under 3 h, none
overlapping the next.

---

## The five vendor rails

Latencies are our own push, measured on production over a trailing 90-day window. What the
channel adds afterwards is operator-reported, not instrumented.

### Ventrata — 693 ms median push

Auth
:   `Authorization: Bearer <apiToken>`, one key per subscription, with
    `VENTRATA_TOKEN` as fallback. OCTO plus REST.

What bites
:   Also hosts the unified auto-apply entry point for *every* vendor — the route
    name is historical, not a scoping mistake. Full price-change history and an
    undo system live here too.

Knobs
:   `VENTRATA_AUTO_APPLY_CONCURRENCY`, `VENTRATA_PRICE_PUSH_TIMEOUT_MS`,
    `VENTRATA_GET_AVAILABILITY_TIMEOUT_MS`, `VENTRATA_PREFETCH_CONCURRENCY`,
    `VENTRATA_VERIFY_AFTER_PUSH`

Onboarding
:   One credential, still collected and typed in by hand. No app store exists, so
    self-serve entry is the only lever.

### Bokun — 12.1 s median push, 17× Ventrata

Auth
:   HMAC-signed REST v2, plus App Store OAuth. Three separate credentials when
    collected manually.

What bites
:   **Roughly 1 change in 7 never lands**, and the cause is unrecorded. A
    three-stage breaker limits the damage: per-experience quarantine at 4
    consecutive 5xx, then a global half-open cooldown of 30 s once cumulative
    5xx crosses 10, then a hard stop after 3 failed probes. Look for
    `[Bokun CIRCUIT]` in the logs. See [P4](#p4).

Knobs
:   `BOKUN_AUTO_APPLY_DISABLED`, `BOKUN_PRICE_PUSH_DRY_RUN`,
    `BOKUN_CIRCUIT_5XX_THRESHOLD`, `BOKUN_PER_EXPERIENCE_5XX_THRESHOLD`,
    `BOKUN_GLOBAL_COOLDOWN_MS`, `BOKUN_MAX_COOLDOWN_ROUNDS`, `BOKUN_SKIP_NOOP`,
    `BOKUN_DRIFT_THRESHOLD_PCT`

Onboarding
:   The app-store install flow is **already written and needs wiring** — one
    click would replace all three credentials.

### Xola — 5.9 s median push

Auth
:   `x-api-key` header, **not** Bearer. Resolution order: `x-api-token` request
    header → `subscription.xolaApiKey` → `XOLA_DEFAULT_API_KEY` env.

What bites
:   If a push ever authenticates as the wrong operator, that resolution order is
    why — an inherited request header wins over the subscription's own key.
    Booking-priced products are handled separately in guardrails and push
    (ENG-2423).

Knobs
:   `XOLA_RECOMMENDED_PRICE_ROUNDING_INCREMENT_MAJOR`,
    `XOLA_RECOMMENDED_PRICE_ROUNDING_MODE`

Onboarding
:   Every new supplier has to be registered **by Xola's own team**. No status
    endpoint, no agreed turnaround — an operator can be signed and unreachable
    with nothing on our side saying so.

### Prioticket — low volume

Auth
:   OCTO, same adapter shape as the others.

What bites
:   Nothing specific. Smallest rail, behaves like Ventrata. Covered by
    `prioticket.service.push-origin.spec.ts` and the dispatcher tests.

Knobs
:   None specific.

Onboarding
:   Manual.

### Viator — read path, not a push rail

Auth
:   Partner JWT verified against Viator's JWKS: `VIATOR_JWKS_URL`,
    `VIATOR_JWT_SECRET`, `VIATOR_JWT_PUBLIC_KEY`.

What bites
:   Easy to assume it is a fifth price-push rail. It is not — linkout, partner
    campaigns and a waitlist. Campaign sends go through
    `scripts/send-campaign-emails.ts`.

Knobs
:   None specific.

Onboarding
:   Partner-side, handled per deal.

## Kill switches and knobs

Environment variables on the Cloud Run revision. Changing one is a revision update,
not an image rebuild, so **rollback is the previous revision**. Defaults below are read
from source; the live revision was not readable at the time of writing (see
[Open questions](#open-questions)).

#### Emergency switches

The four you would actually reach for during an incident.

`BOKUN_AUTO_APPLY_DISABLED` — default `false`
:   **Single-vendor kill switch.** First thing to flip when Bokun's gateway is
    degraded. The Ventrata, Xola and Prioticket sub-batches carry on.

`RUN_WORKER` — `false` on the API, `true` on the worker
:   Selects Service or Worker behaviour. **Never set it true on the API
    service**: every cron handler self-skips when it is, and all ten schedules
    stop with no error and no alert.

`AUTO_APPLY_TASKS_COUNT` — default `5`
:   Parallel Job tasks. Production currently runs `taskCount: 1`, which is why
    batches overlap — see [P3](#p3). Raising it shards the work; lowering it
    protects the database, which the Job reaches unpooled.

`PRICE_PUSH_HEALTH_DIGEST_ENABLED`
:   The end-of-batch Slack digest. **Leave this on.** It is the only proof the
    batch completed, and its absence is your only alarm.

#### Tuning knobs

`AUTO_APPLY_VIA_CLOUD_RUN_JOB` — default `false`
:   Routes the batch to the Cloud Run Job. Given executions exist, this *is* on
    in production. Off means the batch runs on the user-facing service, where a
    revision rollover kills it mid-push.

`AUTO_APPLY_JOB_NAME` — default `walkway-auto-apply`
:   Dispatch target. Change only if the Job is renamed.

`AUTO_APPLY_RESPECT_OWNER_TOGGLE`
:   Honours the compset owner's auto-apply toggle (ENG-2431). **Do not turn
    off** — it pushes for operators who opted out.

`MANUAL_PRICE_LOCK_DEFAULT_HOURS` / `_MAX_HOURS`
:   How long a hand-pushed slot is protected from the batch, and the ceiling a
    user may request.

`SELF_COMPARE_ALLOWED_CHANNELS`
:   Channels allowed to compare against themselves. Widening this is how
    self-referential pricing loops start.

`BOKUN_PRICE_PUSH_DRY_RUN` — default `false`
:   Computes and logs the pushes without sending them. Pair with
    `BOKUN_PRICE_PUSH_DRY_RUN_LOG_DIR` when reproducing locally.

`BOKUN_CIRCUIT_5XX_THRESHOLD` — default `10`
:   Cumulative 5xx before the global cooldown arms. `0` disables the global
    breaker and keeps per-experience quarantine only.

`SUPPORT_INBOX_ENABLED` — default `false`
:   The 2-minute IMAP poll. Inert until the mailbox credentials are confirmed in
    the target environment.

`VENDOR_HTTP_MAX_SOCKETS`
:   Global outbound socket ceiling across every vendor HTTP call.

## Dependencies

| Dependency | What it carries | If it breaks |
| --- | --- | --- |
| Cloud SQL Postgres | Prisma 5 schema + SQL migrations. Runtime pooled, migrations direct. | Everything stops |
| Redis Memorystore | BullMQ queue + caching | API fine, all queued side effects vanish — [P6](#p6) |
| `walkway-saas-mlservice` | Compset discovery, product grading | Compset features degrade |
| BigQuery | Analytics reads | Reports only |
| Mage KPI marts | Everything the Data Hub (ENG-791) reads | **Reports go stale with no error** |
| Elasticsearch | Product/compset search indexing | Search degrades |
| Stripe | Subscriptions, billing, `POST /stripe/webhook` | Billing only |
| Slack | Digests, alerts, per-vendor debug feeds | **You lose your main signal** |
| Gmail SMTP / IMAP | Outbound email; the support-inbox poll | Email + support ingestion |
| Google OAuth | Sign-in | Nobody can log in |
| Cloud Scheduler | Per-compset refreshes + `saas-notifications` | Refreshes stop |
| Sentry | Errors for Service and Job | You debug blind |
| Vendor APIs | The five rails | Bokun is the one that degrades |

---

## Escalation {: #escalation }

!!! danger "Fill this in before you leave"
    This is the one table I cannot generate. The Linear workspace has two teams — **Tech
    (`ENG`)** and **Walkway (`WAL`)** — and the roster below is everyone active. Assign a
    real owner per area, with a Slack handle and a timezone, and this page becomes something
    people can actually lean on at 3 a.m.

| Area | Owner | Backup | Slack / TZ |
| --- | --- | --- | --- |
| Backend, price push | *to confirm* | | |
| Auto-apply trigger (the Python caller) | *data team — confirm who* | | |
| Data / Mage / BigQuery | *to confirm* | | |
| GCP, infra, billing | *to confirm* | | |
| Vendor relationships (Bokun, Xola) | *to confirm* | | |
| Product decisions on pricing behaviour | *to confirm* | | |

Active workspace members at handover: alegria, anurag, Brayan Ortiz, Brent Boone, bruno,
Emmanuel Gautier, gabriel, garikoitz, juan, laura, terry, Thiago Hernandez, vinicius, zlata.

---

## Landmines

!!! danger "Credentials are committed in plaintext"
    The Cloud Run manifest in this repository carries live third-party and database
    credentials as literal values, and they are in git history. Anyone with repo read access
    has them. They belong in Secret Manager — but **rotation matters more than the move**,
    because history cannot be un-shared. Treat as open until rotated.

!!! danger "The batch overlaps itself and sometimes exceeds its ceiling"
    Unsharded, 2 of 29 sampled runs blew the 4 h task timeout and were retried; 6 of 29
    overlapped the next execution. See [P3](#p3).

!!! danger "No alert exists for the two silent failures"
    A batch that never starts, and a cron that stops firing, both produce zero signal. The
    absence of a Slack digest is the only symptom of either.

!!! warning "Redis failure degrades silently"
    See [P6](#p6).

!!! warning "The live schema drifted from migrations"
    A recent commit is `chore(prisma): sync the schema with the live DB` — the database had
    moved ahead of `prisma/migrations`. Diff before the next migration, or a fresh
    environment will not reproduce prod.

!!! note "Dev and prod scheduler jobs are interleaved"
    Per-compset refresh jobs target both `walkwaysaasbackend` and `walkwaysaasbackend-dev`.
    Check the host before debugging.

!!! note "Repo hygiene"
    `dist/` is committed; `stripe-cli.zip` (8.7 MB), `token.txt` and `storageState.json` sit
    in the repo root. The last two are worth reading before assuming they are inert.

!!! note "Two clones, one remote"
    Backend work belongs in the `walkway_backend_last` clone; `walkway_saas_backend` is a
    stale second clone. The backoffice has the same duplication.

---

## Open questions {: #open-questions }

Honest list of what this page does **not** establish. Each one is a small task, not a mystery.

1. **What is the Python caller that triggers auto-apply?** Confirmed as `python-requests`
   from GCP address space every ~3 h, but not attributed to a named service or pipeline.
   Ask the data team and write the answer here.
2. **Live values of the feature flags.** The table above is code defaults; reading the
   deployed revision's environment was not permitted from my session. Confirm at least
   `AUTO_APPLY_VIA_CLOUD_RUN_JOB`, `AUTO_APPLY_TASKS_COUNT`, `AUTO_APPLY_RESPECT_OWNER_TOGGLE`
   and `PRICE_PUSH_HEALTH_DIGEST_ENABLED`.
3. **Why is `taskCount` 1?** Deliberate, or drift? The answer decides whether [P3](#p3) is a
   one-line fix.
4. **Why does 1 Bokun change in 7 never land?** The cause is genuinely unrecorded. Recording
   the failure reason is what turns this from folklore into a fix.
5. **Escalation owners** — see [above](#escalation).

---

## Commands

### Roll back the API service

```bash
gcloud run revisions list --service=walkwaysaasbackend \
  --project=ww-da-ingestion --region=us-central1

gcloud run services update-traffic walkwaysaasbackend \
  --project=ww-da-ingestion --region=us-central1 \
  --to-revisions=<revision-name>=100
```

### Flip a kill switch without a build

```bash
gcloud run services update walkwaysaasbackend \
  --project=ww-da-ingestion --region=us-central1 \
  --update-env-vars=BOKUN_AUTO_APPLY_DISABLED=true
```

### Logs

```bash
# the Job
gcloud logging read \
  'resource.type=cloud_run_job AND resource.labels.job_name=walkway-auto-apply' \
  --project=ww-da-ingestion --limit=200 --freshness=1d

# the API service
gcloud logging read \
  'resource.type=cloud_run_revision AND resource.labels.service_name=walkwaysaasbackend' \
  --project=ww-da-ingestion --limit=200 --freshness=1h

# who called auto-apply, and did it get a 202
gcloud logging read \
  'resource.type=cloud_run_revision AND resource.labels.service_name=walkwaysaasbackend
   AND httpRequest.requestUrl:"recommendations/auto-apply"' \
  --project=ww-da-ingestion --limit=10 --freshness=24h \
  --format="value(timestamp,httpRequest.status,httpRequest.userAgent)"
```

### Re-fire a digest or snapshot

```text
POST /admin/hta-price-snapshot/run
POST /admin/auto-pilot-digest/run
POST /admin/operator-health-score/run
POST /admin/adoption-funnel/run
POST /admin/signals/run
```

### Locally

```bash
npm install && npx prisma generate
npm run start:dev              # API, hot reload
npm run start:worker           # BullMQ worker, separate process
npm run test:push-origin       # the contract tests that matter for vendor work
npx prisma migrate deploy      # uses DATABASE_DIRECT_URL
```

Swagger is at `/api`.

---

## In flight at handover

| Branch | What it is | State |
| --- | --- | --- |
| `feat/eng-2464-price-recommendation-impact-columns` | Impact columns on `price_recommendations`, plus the schema-sync commit. The branch I was on. | in progress |
| `fix/backport-2431-2346-2472-onto-dev` | Backport of the owner gate, waitlist calendar source, multipart fix onto `dev`. | open |
| `fix/eng-2472-register-fastify-multipart` | Register `@fastify/multipart` — upload routes 500 without it. | open |
| `feat/eng-2423-relanding-on-main` | Xola booking-priced products, re-landing after PR #462 was reverted in #463. **Read the revert first.** | needs care |
| `feat/eng-2346-waitlist-calendar-pending-source` | Pending source on the waitlist calendar. | open |
| `fix/eng-2431-auto-apply-gate-compset-owner` | Gates auto-apply on the compset owner's toggle. | open |

Conventions: Conventional Commits, one ENG ticket per branch, never edit a merged migration —
always a follow-up. Every new vendor flow ships with a `*.push-origin.spec.ts` plus dispatcher
coverage in `ventrata.service.spec.ts`.

---

## Keeping this page true

A runbook nobody updates is worse than none, because people trust it.

- The **verified** claims carry a date. If that date is more than a quarter old, re-run the
  commands in [Verify the batch](#verify-the-batch) and the revision describe.
- When you answer one of the [open questions](#open-questions), delete it from that list and
  fold the answer into the body.
- When you add a feature flag, add it to the kill-switch table in the same PR. That is the
  cheapest moment it will ever be.

**Further reading:** `README.md` (architecture, local setup), `docs/` (per-feature and
per-vendor depth), `docs/AUTO_APPLY_CLOUD_RUN_JOB.md` (the Job's IAM setup).
