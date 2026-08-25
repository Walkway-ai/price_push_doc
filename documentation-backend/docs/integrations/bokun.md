# Bokun

Median push **12.1 s** — 17× Ventrata — and roughly **1 change in 7 never lands**, cause
unrecorded. The slowest and least reliable rail, and the one with the most machinery built
around it.

!!! abstract "The one thing to understand"
    The `PUT` that writes prices is **neither a full replace nor a pure upsert**. Base rules
    and schedule rules follow *opposite* rules about omission. Get it wrong in the safe-looking
    direction and prices silently revert for every OTA. This cost a post-mortem in May 2026.

Deep reference in the backend repo: **`docs/BOKUN_PRICE_PUSH.md`** (in French, more detail than
this page, including the transformation order and diagnostic scripts).

---

## The entity model

| Bokun entity | What it is | Key |
| --- | --- | --- |
| **Experience** | A product | `experienceId`, numeric |
| **Rate** | A pricing variant of an experience | `rate.id` plus `rate.externalId`, a short label like `TG1` |
| **PricingCategory** | A ticket category — Adult, Child, Senior, Free. Each experience has a base/default one, typically Adult, used as the anchor | `pricingCategoryId` |
| **PriceCatalog** | A distribution channel — Bokun Marketplace, GetYourGuide, Viator… An experience belongs to several | `priceCatalogId` |
| **PriceSchedule** | A calendar window with a name, defining *when* an override applies | `priceScheduleId`, returns `start`, `end`, optionally `dates[]` |
| **ExperiencePriceRule** | One price line: for this experience, on this rate × category × catalog, optionally scoped to a schedule, the price is X | `rate.id`, `pricingCategoryId`, `priceCatalogId`, optional `priceScheduleId`, `amount`, `currency` |

An experience carries a list of `experiencePriceRules` containing two families:

**Base rules** — no `priceScheduleId`. The default price for any date not covered by an
override.

**Schedule rules** — with a `priceScheduleId`. An override for the dates that schedule covers.

The effective price of a slot resolves as:

```
if a schedule covering the date has a rule on the right rate × category × catalog:
    → that schedule rule's price
else:
    → the base rule's price for the same tuple
```

!!! warning "A schedule rule only applies if its schedule actually covers the date"
    Pushing a rule attached to a schedule whose `dates[]` or `start`/`end` window does not
    contain the target date means the rule is **ignored** and the price falls back to base. The
    push reports success. This is the most common cause of "we pushed but nothing changed".

---

## Endpoints

All requests are HMAC-signed. Credentials per subscription via
`getBokunHmacCredentialsOrThrow`, stored on `Subscription` then `User` as fallback.

| Action | HTTP | Path |
| --- | --- | --- |
| Read rates | `GET` | `/restapi/v2.0/experience/{id}/components?componentType=RATES` |
| Read pricing rules | `GET` | `/restapi/v2.0/experience/{id}/components?componentType=PRICING` |
| Read defaults and catalogs | `GET` | `/restapi/v2.0/experience/{id}/components?componentType=ALL` |
| **Write prices** | `PUT` | `/restapi/v2.0/experience/{id}/components?componentType=PRICING` |
| List schedules | `GET` | `/restapi/v2.0/pricing/schedules?pageNo=&pageSize=` |
| Read schedule | `GET` | `/restapi/v2.0/pricing/schedule/{id}` |
| Create schedule | `POST` | `/restapi/v2.0/pricing/schedule` |

Three `GET`s precede every push — rates to resolve `rateId` from `tour_grade_code`, `PRICING`
to build the body, `ALL` for `defaultPricingCategoryId`. That round-trip count is a large part
of the 12-second median.

---

## The `PUT` semantics

Bokun applies a hybrid rule, and the two halves are asymmetric:

**Base rules must all be present.**
:   The body must contain every base rule for every `(rate × pricingCategory × priceCatalog)`
    tuple this push touches. Omit one and Bokun returns a 400: *"Request would deactivate base
    rule(s) without replacing them"*.

**Schedule rules must not be omitted.**
:   Omitting an existing schedule rule **deactivates it**. No error, no warning. The slot falls
    back to the base rule for every OTA.

The stable production pattern:

```
PUT body = [
  ...all existing rules not replaced by this push,
  ...the new rules built for this push,
]
```

"Replaced" is decided per slot — the tuple `(rateId, pricingCategoryId, priceScheduleId,
priceCatalogId)` — by `ruleIsReplacedByAnyPush`. An existing rule whose tuple a new rule
targets is dropped from the body and superseded; every other existing rule is carried through
verbatim.

!!! danger "Do not optimise the body down"
    Commit `2adcd59` attempted exactly that and violated the schedule-rule half of the
    contract. See the post-mortem of 19 May 2026. If you are about to make this `PUT` send
    less data, read section 11 of `docs/BOKUN_PRICE_PUSH.md` first — it is a checklist written
    for this specific temptation.

---

## Reliability machinery

Bokun degrades often enough to justify three separate mechanisms.

### Circuit breaker, three stages

1. **Per-experience quarantine** — 4 consecutive 5xx on one experience and it is dropped for
   the rest of the batch, so one bad product cannot poison the run. This is what the historical
   11.9 % success-rate batches looked like without it.
2. **Global half-open cooldown** — cumulative 5xx across all experiences crossing
   `BOKUN_CIRCUIT_5XX_THRESHOLD` (default 10) pauses the batch for
   `BOKUN_GLOBAL_COOLDOWN_MS` (default 30 s). The next recommendation acts as a half-open
   probe: success resets the counter, failure pauses again with escalating backoff.
3. **Hard stop** — after `BOKUN_MAX_COOLDOWN_ROUNDS` (default 3) failed probes the circuit
   opens for good and remaining recommendations are skipped. By then Bokun's gateway is
   genuinely down and pushing harder only adds pressure.

Look for `[Bokun CIRCUIT]` in the logs. Set `BOKUN_CIRCUIT_5XX_THRESHOLD=0` to disable the
global stages and keep per-experience quarantine only.

### No-op skip

On by default (`BOKUN_SKIP_NOOP`). A push whose computed price equals the current one is not
sent. Given the round-trip cost this is a significant saving, and a rising no-op count is
healthy, not a problem.

### Dry run

`BOKUN_PRICE_PUSH_DRY_RUN` computes and logs the full `PUT` body without sending it, writing to
`BOKUN_PRICE_PUSH_DRY_RUN_LOG_DIR`. The correct way to validate a payload-shape change before
it touches an operator's prices.

---

## Tuning

| Variable | Default | Effect |
| --- | --- | --- |
| `BOKUN_AUTO_APPLY_DISABLED` | `false` | Single-vendor kill switch; the other rails carry on |
| `BOKUN_SKIP_NOOP` | on | Skip pushes where the price is unchanged |
| `BOKUN_CIRCUIT_5XX_THRESHOLD` | `10` | Cumulative 5xx before the global cooldown arms; `0` disables it |
| `BOKUN_PER_EXPERIENCE_5XX_THRESHOLD` | `4` | Consecutive 5xx before quarantining one experience |
| `BOKUN_GLOBAL_COOLDOWN_MS` | `30000` | Pause length once armed |
| `BOKUN_MAX_COOLDOWN_ROUNDS` | `3` | Failed probes before the hard stop |
| `BOKUN_PRICE_PUSH_DRY_RUN` | `false` | Compute and log, do not send |
| `BOKUN_PRUNE_EXPIRED_SCHEDULES` | opt-in | Delete expired schedules during the push |
| `BOKUN_DRIFT_THRESHOLD_PCT` | — | Guard against an unexpectedly large price move |
| `BOKUN_RATES_FETCH_ATTEMPTS` | — | Retries on the rates `GET` |
| `BOKUN_PUSH_ONLY_NEW_RULES` | — | Narrows what the `PUT` sends. See the warning above |
| `BOKUN_PRICING_PUT_DEBUG` / `_TO_SLACK` | — | Dump the `PUT` body to logs or Slack |

---

## Known edge cases

Documented with reproduction detail in section 9 of `docs/BOKUN_PRICE_PUSH.md`:

- a schedule is created but does not cover the target date, so the price silently falls back
- `applyPricingRules` returns an unexpected amount
- rate id resolution fails and `rateId` comes back `null`
- OCTO availability and the `PRICING` component disagree

Diagnostic scripts live in `scripts/` — `inspect-bokun-experience.ts`,
`inspect-bokun-schedule.ts`, `inspect-bokun-octo-availability.ts`,
`audit-bokun-schedule-cache.ts`.

---

## Onboarding

Three separate credentials when collected by hand. **The app-store install flow is already
written and needs wiring** — one operator click would replace all three. It is the single
highest-leverage onboarding change available on any rail.
