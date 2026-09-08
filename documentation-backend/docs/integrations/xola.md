# Xola

Median push **5.9 s**. Native Xola REST, `x-api-key` auth.

!!! abstract "The one thing to understand"
    On Ventrata you set a price on a slot. On Xola **there is no such thing as a slot price**.
    A price override is expressed as a *purchase rule* — a conditional pricing instruction —
    and for that rule to apply to one date and time, a *schedule* must exist on the
    experience, and the rule must be *linked* to the experience.

    So a single price push can create three objects on the vendor side: a **schedule**, a
    **purchase rule**, and a **purchase rule link**. If any is missing or stale, the price
    silently falls back to the experience's catalog price. Most Xola incidents are one of
    those three being wrong.

---

## Authentication

Header is `x-api-key`, **not** `Authorization: Bearer`. Getting this wrong returns a 401 that
reads like an expired token.

The key is resolved in a strict order, and the first hit wins:

1. `x-api-token` **request header**, if present
2. `subscription.xolaApiKey`
3. the user's stored key (a propagated copy of the subscription's)
4. `XOLA_DEFAULT_API_KEY` from the environment

!!! warning "This order is the cause of wrong-operator pushes"
    A request header beats the subscription's own key. If a push ever applies to the wrong
    operator's experience, an inherited or hard-coded `x-api-token` is the first thing to
    check. `logXolaApiKeyResolutionDebug` logs which source won for every resolution — that
    log line is the fastest way to settle it.

---

## The objects involved

`Experience`
:   The product. Xola ids are Mongo ObjectIds (24 hex chars). Walkway sanitises inbound ids
    with `sanitizeXolaNativeExperienceOrProductId` because callers sometimes pass a Walkway
    product id instead.

`Catalog item` → `template`
:   A demographic ticket type (Adult, Child…). Templates are matched to Walkway unit types by
    normalising and fuzzy-matching `template.code` and `item.sku` against a set of aliases —
    exact match, then substring in either direction. A price is pushed **per template**.

`Schedule`
:   A calendar entry on the experience that says *these dates, these times*. Walkway creates
    one per pushed slot when needed. Without a schedule the price cannot be scoped to a slot.

`Purchase rule`
:   The pricing instruction itself: a filter (which bookings does this apply to) plus actions
    (what price to set). Lives at `/api/purchaseRules`.

`Purchase rule link`
:   Attaches a rule to an experience. **Without the link the rule exists but never fires at
    checkout.**

`Seller`
:   Required in the rule payload. Read from the experience response; when Xola omits it or
    the call fails, Walkway falls back to `products.supplier_id` for the `xola` channel.

---

## The push, step by step

`XolaService.updateExperiencePriceWithHistory` is the entry point for both manual and batch
pushes.

1. **GET the experience** — seller and catalog templates.
2. **Resolve template prices** — apply pricing rules (e.g. Child = 50 % of Adult), then
   rounding and the min/max clamp.
3. **Ensure a schedule id.** Reuse the one in the request body; else reuse the one recorded on
   the last history row for the same slot; else `POST /experiences/:id/schedules` — but only
   when both `date` and `arrivalTime` are set.
4. **Build the purchase rule payload** — shape depends on whether a schedule id exists, see
   below.
5. **Create or update the rule.** `PUT` when reusing a rule id from the request, from history,
   or from a price-signature match; `POST` otherwise.
6. **`POST /purchaseRuleLinks`** — only when step 5 created a new rule.
7. **Persist `xolaPriceChange`** — the history row undo reads from.
8. **Update the local `price` table.**

### Endpoints

| Action | HTTP | Path |
| --- | --- | --- |
| Read experience | `GET` | `/experiences/{id}` |
| Create schedule | `POST` | `/experiences/{id}/schedules` |
| Delete schedule | `DELETE` | `/experiences/{id}/schedules/{scheduleId}` |
| Create purchase rule | `POST` | `/purchaseRules` |
| Update purchase rule | `PUT` | `/purchaseRules/{id}` |
| Read purchase rule | `GET` | `/purchaseRules/{id}` |
| Link rule to experience | `POST` | `/purchaseRuleLinks` |

---

## Schedules

Created with `repeat: 'custom'` and an explicit `dates` array — never a recurring pattern.

```json
{
  "name": "Override date 2026-08-16 @ 945",
  "type": "available",
  "repeat": "custom",
  "dates": ["2026-08-16"],
  "departure": "fixed",
  "times": [945],
  "priceDelta": 0,
  "priceOverride": true,
  "allowedPrivacies": ["public", "private"]
}
```

`times` is an **integer in HHMM form**, not a string: `945` is 09:45, `1430` is 14:30. The
helpers `arrivalTimeToStartTime` and `arrivalTimeIntegerToHHmm` convert both ways. The
stricter one returns `null` for garbage input so callers skip creating a manual lock rather
than writing a `00:00` row that matches no recommendation slot.

The `name` follows the convention `Override date <date> @ <time>`, or `Override date <date>`
when there is no time. **That string is load-bear** — the cleanup routine identifies
Walkway-created schedules by it.

### Automatic cleanup

`purgeOutdatedOverrideSchedulesForExperience` deletes Walkway's own past override schedules,
identified by the name convention plus `repeat: custom` plus a date entirely in the past.
Left unchecked these accumulate on every experience, and Xola's own UI becomes unusable.

### Purging retired slots {: #purge-stale }

Past-dated cleanup is not enough: the overrides that cause trouble are the **future** ones
sitting on slots the operator has withdrawn. The code fix stops new ones being created; it
does not remove what is already there.

```
POST /xola/experiences/{id}/schedules/purge-stale-overrides
X-API-KEY: <key>          Body: {"dryRun": true}
```

Dry run by default. It re-reads the experience, keeps only schedules named `Override date …`
carrying exactly one date and one time, and deletes those whose slot `slotOffering` says is
no longer offered. Prices still correct survive. Deletes run **sequentially** — a burst
against a live storefront is the load that makes the checkout widget slower.

!!! warning "It reads the native REST experience, not OCTO"
    An earlier version called `getExperienceById`, which hits `/octo/products/:id`. That
    representation carries no `schedules`, so the route found zero overrides, deleted
    nothing and returned `200`. Both the push and the cleanup must read
    `getExperienceFromXolaApi` or the cleanup cannot see what the push created.

The blunt `purge-by-name` route deletes **every** Walkway override including the correct
ones. On Empire that would have removed 1,776 schedules instead of 170. Prefer the stale
route unless you actually mean to unwind the whole integration.

`DELETE` can return **423 Locked**, which Xola uses when a resource is locked, when there are
active or future bookings tied to the schedule, or per seller policy. Walkway logs it and
moves on — it is not a push failure. A 404 is treated as already-deleted.

---

## The operator's schedule wins {: #operator-schedule }

!!! danger "Creating a schedule puts a slot on sale"
    A Walkway override is `type: 'available'`. Creating one for a time slot the operator
    has retired **puts that slot back on sale**, and customers book a departure that will
    not run. This is not theoretical: Empire Tours reported exactly that in September 2026
    (ENG-2574).

The original check only looked at schedules with `repeat === 'custom'`, which are Walkway's
own overrides:

```js
// Do not map weekly schedules for dynamic slot pricing
if (repeat !== 'custom') return false;
```

So the question being asked was *"do I already have an override here?"*, never *"does the
operator still offer this slot?"*. When an operator retired a slot, the next push found no
override, created one, and reopened it. Worse, an override created last month then counted
as evidence that the slot was offered — the check confirmed its own mistake.

`slotOffering(schedules, date, arrivalTime)` in `src/xola/xola-slot-offering.ts` now asks the
real question, judged on the operator's schedules **only** — ours are excluded from the
evidence on purpose. It returns one of:

| `reason` | Meaning |
| --- | --- |
| `offered` | An operator `available` schedule covers this slot. Push proceeds. |
| `no-operator-schedule-covers-slot` | No operator schedule covers it — retired, or past the end of their season. |
| `blacked-out-by-operator` | An operator `unavailable` schedule covers it. A blackout beats any availability on the same slot. |

The guard runs **whether or not a schedule id was already resolved**. Guarding only the
create path would have missed the reported failure entirely: for a slot Walkway had already
overridden, `resolveExistingScheduleIdFromExperienceData` finds that override — it matches on
`repeat === 'custom'`, which is ours — so the push sailed through and put a fresh price on a
withdrawn slot.

Kill switch: `XOLA_RESPECT_OPERATOR_SCHEDULE=false` restores the old behaviour. Default on.
It exists because the check depends on parsing operator schedules correctly, and an
experience whose schedules we misparse stops being priced entirely — a loud failure, but one
someone may need to unblock without a deploy.

### Hours are stated two different ways {: #time-ranges }

!!! warning "The most expensive mistake in this file"
    A schedule's hours live in `times` **or** in `timeRanges`. Reading only `times` treats a
    `timeRanges` schedule as covering the whole day.

```json
{
  "name": "Mon 8.30am Start time (until fixed)",
  "days": [1], "repeat": "weekly", "type": "unavailable",
  "timeRanges": [{ "startTime": 800, "endTime": 845 }]
}
```

That schedule has no `times` at all. Because an empty `times` means "all day" — which is how
a genuine day-wide blackout such as Thanksgiving is expressed — an 8:00–8:45 blackout was
read as closing all of Monday, including the 9am departure the operator sells through a
separate `Monday: 9am - No End Date` schedule.

Two things went wrong as a result: the push guard refused to price Monday 9am, and the
cleanup judged it retired. **26 override schedules were deleted that should have been kept.**
The slots stayed on sale — the operator's own schedule covered them — but lost their Walkway
price until the next push.

`scheduleCoversSlot` now reads both. Only a schedule stating **neither** covers the whole
day. Range bounds are inclusive, so a slot exactly on the edge counts as inside; for a
blackout that errs toward not selling, which is the direction the rest of the module takes.

### Blackouts are also a workaround {: #blackouts }

Operators use `type: 'unavailable'` schedules to close dates. Empire also adopted them as a
**workaround for the bug above** — "I have instructed the Empire team to use blackouts to
retire time slots" — so a blackout dated around September 2026 on their account may be a
patch rather than an intention. Once the fix is live they should be able to remove them.

When reading a blackout, check the name against `timeRanges`: an operator who means 8:30
may write a schedule that closes the day, and only the name says otherwise.

---

## Purchase rules — two shapes

Which shape gets built depends entirely on whether a schedule id is available.

### Schedule-scoped (the correct one)

Used whenever a schedule id exists. Filter is `in_product_schedule_filter`, actions use
`shortcut.absolute_per_purchase`. This is the shape Xola themselves recommend for slot
pricing.

### Legacy date/time/source

The fallback when no schedule id could be resolved: an `and_filter` over `arrivalDate`,
`arrivalTime` and `source`, with modifiers.

!!! warning "The legacy shape does not reliably reach checkout"
    For timed inventory it may simply not apply. The service emits an explicit warning when it
    falls back to this shape:

    > ⚠️ [Xola] No scheduleId: using legacy date/time/source purchase rule filters. For timed
    > inventory, create a schedule on Xola and pass scheduleId so pricing applies to the slot.

    Seeing that line in the logs means the push probably had no effect on the price a guest
    sees, even though it returned success. **Treat it as a failure even when the status says
    otherwise.**

### The link payload

```json
{
  "name": "Schedule Price Variations",
  "seller":   { "id": "<sellerId>" },
  "entities": [{ "id": "<experienceId>", "object": "experience" }],
  "tags":     [{ "id": "walkway" }],
  "purchaseRules": [
    { "purchaseRule": { "id": "<ruleId>" }, "sequence": 7500001 }
  ]
}
```

The `walkway` tag is how our rules are told apart from the operator's own. `sequence`
determines precedence against other rules — the high value puts ours late, so it wins.

---

## `PUT /purchaseRules/{id}` merges, it does not replace {: #action-merge }

!!! danger "The single most damaging Xola defect to date"
    Xola, confirming the semantics on 2026-09-08:

    > Actions are keyed by id. On every rule update, the actions array you send is merged
    > into the existing rule — it is not a full replacement. Add — omit id. A new action is
    > created and appended. **There is no dedupe.**

Walkway built its actions with **no `id`** and PUT the rule on every push, precisely to
avoid stacking. Every push therefore appended a fresh set instead of replacing one.

On Chicago Gangsters and Ghosts Tours that reached **91,593 actions across 611 rules**, of
which roughly 3,050 have any effect. One rule alone carried 4,386 actions over 5 demographic
templates — 877 pushes' worth, matching its cadence since 2026-05-20 exactly. Xola mitigated
it on their side and asked us to fix the logic.

This is what the operator feels: Aanand at Empire reported **up to a minute of latency in the
checkout widget** and higher cart abandonment. Schedules are not the driver here — measured
on their account, 1,939 schedules against roughly 66,600 live actions.

`reconcileRuleActions(existing, templatePrices)` in `src/xola/xola-rule-actions.ts` now
reconciles instead of appending:

- the **first** attributable action per template keeps its id and is updated in place;
- every other action for that template is sent back with `{ id, _remove: true }`;
- templates with no action yet get a new one;
- actions for templates we are not pricing, and actions we cannot attribute, are **left
  alone** — they may be the operator's own, and deleting someone else's pricing rule to tidy
  ours would be a far worse bug than the one being fixed.

An action is ours when its filter reads
`{ object: 'equals_filter', field: 'template.id', operand: <templateId> }`. Anything else is
not attributable.

`_remove` requires a matching id, and an id Xola does not know returns 400, so only actions
carrying one are ever removed.

**It drains as it runs.** A rule holding 889 actions for one template converges to 1 on its
next push — no separate sweep needed. Which also means: *the bloat only clears if pushes are
running*. Empire's had been stopped since 2026-09-01, so nothing was draining.

---

## Rule reuse by price signature

Creating a rule per push would leave thousands of rules on an experience. Instead Walkway
fingerprints the price set:

```
sha256( sorted([{ templateId, cents }, …]) )
```

Prices are normalised to integer cents and sorted by template id, so the signature is stable
regardless of ordering or float representation. When an existing rule carries the same
signature, Walkway issues a `PUT` and **merges the new schedule id into that rule's existing
schedule list** rather than creating another rule.

Reading the current schedule list back out means walking the filter tree —
`extractInProductScheduleFromRuleFilter` handles both a top-level
`in_product_schedule_filter` and one nested under an `and_filter`.

### Stale schedule recovery

If a schedule was deleted on Xola's side but is still referenced by a rule, the `PUT` fails
with a 400 shaped like:

```json
{ "field": { "filter.filters.0.schedules.items": {
    "reason": "no_schedule_found",
    "message": "Could not find a schedule with id …" } } }
```

`extractNoScheduleFoundIdsFromXolaError` pulls the offending 24-hex ids out of that payload —
from the structured field *or* by regex over the message text — so the rule can be retried
without them. Without this, one deleted schedule would poison a rule permanently.

---

## Booking-priced products (ENG-2423)

Some Xola experiences are priced **once per booking**, not per guest. They still return a
single demographic (`Guests`, `guests-over-21`, `beer-tour`) and express the per-booking
nature only as `priceType: "outing"`.

The authority is **`products.pricing_per` in our database, not the Xola payload**. A
"vendor returned no templates" check never fires for these, so payload-shape detection would
miss them entirely. When a `product_grade_channel_code` is supplied it is preferred for the
lookup, because one Xola experience can map to several Walkway product rows and the pgcc names
exactly one.

---

## Skip and failure buckets

Batch results are canonicalised into buckets for the Slack digest. Order matters — the
auto-apply-off message is long and mentions other keywords, so it is matched first.

| Bucket | Means |
| --- | --- |
| `auto-apply-off` | The compset owner's toggle is off (ENG-2407) |
| `no-api-key` | No key resolved from any of the four sources |
| `product-not-found` | Not a Xola product, or absent from our tables |
| `compset-not-found` | The compset in the batch payload does not exist |
| `invalid-start-time` | `arrivalTime` missing or unparseable |
| `purchase-rule-fail` | The rule `POST`/`PUT` was rejected |
| `schedule-fail` | Schedule creation or resolution failed |
| `manual-lock` | Slot protected by a recent manual push |
| `no-op-same-price` | Price already at target, nothing sent |
| `auth-error` | 401, 403, or an authentication message |
| `not-found-404` | Vendor 404 |
| `vendor-error` | Any other Xola API error |
| `other` / `unknown` | Unmatched — if this grows, add a bucket |

The digest reports skips and failures **combined** in `skipReasonCounts`, with a
skips-only breakdown alongside. A rising `no-op-same-price` is healthy. A rising
`schedule-fail` or `purchase-rule-fail` is not.

---

## Tuning

`XOLA_DEFAULT_API_KEY`
:   Last-resort key. Convenient in dev, dangerous in prod — it makes a missing subscription
    key look like a working push against whatever account the key belongs to.

`XOLA_RECOMMENDED_PRICE_ROUNDING_INCREMENT_MAJOR` / `_MODE`
:   Rounding applied to the recommended price before it becomes a template price.

`XOLA_RESPECT_OPERATOR_SCHEDULE`
:   Default on. `false` restores pre-ENG-2574 behaviour: Walkway creates an override for any
    slot it has a recommendation for, including ones the operator retired. Only turn it off
    if an experience has stopped being priced entirely because we misparse its schedules,
    and say so in the incident channel — it re-opens the failure Empire escalated.

---

## The Empire incident, September 2026 {: #empire-incident }

Worth reading once, because it exercises everything above and every number here was measured
on their live account rather than estimated.

Aanand reported two things: Walkway reopening retired slots, and up to a minute of checkout
latency. **They are separate defects with separate fixes**, and it is easy to claim the first
fix addresses the second. It does not.

| | Cause | Fix |
| --- | --- | --- |
| Retired slots on sale | The `repeat === 'custom'` check | [Operator schedule guard](#operator-schedule) + [stale purge](#purge-stale) |
| Checkout latency | Actions appended per push | [Action reconciliation](#action-merge), drains as pushes run |

What the purge actually found across their 9 experiences:

```
1,776 Walkway overrides total
  170 no longer offered   →  deleted
1,606 still correct       →  untouched
```

The 170 broke down as 102 future and 68 past, and by reason: 46 `no-operator-schedule-covers-slot`,
124 `blacked-out-by-operator`. Of those 124, **26 were the `timeRanges` misreading** described
above and should not have been deleted.

The genuine ones are unambiguous. On the Minibus tour the operator's own schedules say:

```
until 2026-11-01 : "Thru Nov 1 2026 Daily 6pm & 8pm"  days=[0..6]      times=[1800,2000]
from  2026-11-02 : "Nov 2 2026 - Mar 13 2027 …"       days=[0,2,4,5,6] times=[1800]
```

Winter schedule: no Monday, no Wednesday, no 8pm at all. Walkway's overrides were holding
`Mon 02/11 18h`, `Mon 02/11 20h`, `Tue 03/11 20h`, `Wed 04/11 18h`, `Wed 04/11 20h` on sale.

Two things to carry forward. **Deleting an override does not always take a slot off sale** —
if an operator `available` schedule still covers it, only the Walkway price goes, and the
slot sells at their base rate. And **the operator's own account state is evidence, not
decoration**: Capitol Hill carried 58 operator schedules of which 56 were `unavailable`,
which is what a blackout-based workaround looks like from the outside.

---

## Onboarding

Every new supplier must be registered **by Xola's own team**. There is no status endpoint and
no agreed turnaround, so an operator can be signed, configured on our side, and still
unreachable — with nothing in our system saying so. Budget for it, and confirm registration
before promising a go-live date.
