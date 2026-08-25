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

`DELETE` can return **423 Locked**, which Xola uses when a resource is locked, when there are
active or future bookings tied to the schedule, or per seller policy. Walkway logs it and
moves on — it is not a push failure. A 404 is treated as already-deleted.

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

---

## Onboarding

Every new supplier must be registered **by Xola's own team**. There is no status endpoint and
no agreed turnaround, so an operator can be signed, configured on our side, and still
unreachable — with nothing in our system saying so. Budget for it, and confirm registration
before promising a go-live date.
