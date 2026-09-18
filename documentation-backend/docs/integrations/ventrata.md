# Ventrata

Median push **693 ms** — the fastest and most reliable rail, and the reference implementation
the others are modelled on. OCTO plus REST, `Authorization: Bearer <apiToken>`.

!!! abstract "The one thing to understand"
    **Ventrata does not return the price you pushed.** Push `retail = 3900` and the
    availability response comes back with `3900 + tax`. Every consumer of that response has to
    deduct `includedTaxes` to recover the net price. Skipping that step inflates prices in our
    own database on every push.

Deep reference in the backend repo: **`src/ventrata/PRICE_PUSH_FLOW.md`** (653 lines, covers the
frontend acceptance flow too).

---

## A note on the route names

`POST /ventrata/recommendations/auto-apply` is the entry point for **every** rail, and
`VentrataService.autoApplyBatchPricePush` is the dispatcher for all four. This is historical,
not architectural — Ventrata was first. Do not read scoping into the path.

---

## Authentication

`Authorization: Bearer <apiToken>`, one key per subscription, with `VENTRATA_TOKEN` as an
environment fallback. Keys are validated before being stored, so a bad key fails at
configuration rather than at push time — the one rail where that is true.

---

## The push

The target price is sent **as-is**: no discount logic, no gross-up.

`unitPricing[].retail`
:   The target price per unit. Pricing rules (Child = 50 % of Adult, and so on) are resolved
    *before* the push, so what goes over the wire is already the final per-unit figure.

### Parity modes

| Mode | Behaviour |
| --- | --- |
| `FULL_PARITY` | Push to every source in turn — `TERMINAL`, `CHECKOUT`, `DASHBOARD`, `CONNECT`, `CONCIERGE`, `KIOSK` — with the same value |
| `SPLIT` | Push only to the named source, e.g. `CHECKOUT` for direct, `CONNECT` for OTAs |

`FULL_PARITY` means one logical push is six sequential HTTP calls. It is still the fastest
rail. See `docs/PRICE_PARITY_MODE.md`.

---

## The tax round-trip

After pushing, the **actual** availability response is fetched and used to upsert
`public.price`. The response is tax-inclusive:

```json
{ "unitPricing": [ {
    "unitType": "ADULT",
    "original": 4480,
    "retail":   4255,
    "includedTaxes": [ { "name": "State/Local Sales Tax", "original": 374, "retail": 355 } ]
} ] }
```

Taxes are deducted from both figures to recover net prices:

```
totalTaxOriginal = sum(includedTaxes[].original)
totalTaxRetail   = sum(includedTaxes[].retail)

originalPrice    = (original - totalTaxOriginal) / 100
discountedPrice  = (retail   - totalTaxRetail)   / 100
hasOffer         = original != retail
```

Not every product has taxes; when `includedTaxes` is empty the deduction is zero and amounts
pass through unchanged.

!!! warning "`has_offer` tracks discounts, not taxes"
    It is `true` only when a discount is actually running — `original != retail` in the
    response. A taxed product with no discount has `has_offer = false` and two identical price
    columns. Reading `has_offer` as "this product has some price complexity" is wrong.

### When the post-push fetch fails

Fall back to the pushed value from `updateDto.unitPricing[0].retail`: both `original_price` and
`discounted_price` are set to it, and `has_offer` defaults to `false` because no discount
information is available. The price is right; the discount metadata is simply unknown.

---

## Which unit gets priced {: #unit-selection }

!!! danger "A product can expose two units of the same type"
    World of Illusion exposes two units typed `ADULT`. Ventrata reported in September 2026
    that Walkway was pricing the wrong one, and they were right.

Read live from their API, stable across dates:

```
[0] unit_fe1f5f86-…  ADULT  2125   the standard Adult
[6] unit_3311ddfe-…  ADULT     0   "Carer / Companion (ID Required)"
```

The push built a `unitType → unitId` map like this:

```js
unitIdByUnitType = new Map(
  liveUnits.filter((u) => !!u.unitId).map((u) => [u.unitType, u.unitId]),
);
```

`new Map(entries)` keeps the **last** value for a repeated key, so `ADULT` resolved to the
Carer id — on every push, for every date, not intermittently. The map is now built by
iterating and skipping a type already resolved: **first wins**. When a product exposes
several units of one type, a warning names them; the extras keep their current price, and
deciding what to do about that is a product call, not a code fix.

Three further defects made the same class of mistake possible elsewhere, fixed alongside
though none caused that report: a case-sensitive `u.unitType === 'adult'` that never matched
Ventrata's upper-case `"ADULT"`; `guessUnitTypeFromId()` ending in `return UnitType.ADULT`,
which answered ADULT for every UUID and made any `find` using it stop on element 0; and a
positional `|| availability.unitPricing[0]` fallback. The fallback path now throws rather
than pricing an unidentified category.

Walkway prices **every unit type a product exposes**, not only ADULT. A €0 unit is not
necessarily a bug — a `FREE`/`OTHER` rule can legitimately set it.

### The local price table lost every push for three days (ENG-2619) {: #no-adult-price }

The fix above had a side effect. After each push the local `price` table is upserted from
the pushed payload (the post-push GET is off in production, `VENTRATA_VERIFY_AFTER_PUSH`
unset). That upsert found the ADULT unit with `guessUnitTypeFromId`, whose ADULT default is
what PR #548 removed — so from 2026-09-07 it found **no** adult unit, and every Ventrata push
wrote `priceUpsertStatus = SKIPPED / no_adult_price`: **401,938 rows across 10 operators in
three days**, while the vendor write itself kept succeeding. The calendar showed the push as
done and the local price never moved.

Fixed in PR #573 (2026-09-14, `src/ventrata/price-table-adult-unit.ts`): the upsert now
names the adult unit from the **pre-push availability**, where Ventrata labels every unit's
type; a single-unit payload is the base price by construction; the id hint is the last
resort, for the few ids that spell their type. Since 2026-09-15 every row is `VERIFIED`.

What the same ticket taught about Extranomical (product `39f74fa7`, Yosemite): our write
does reach Ventrata and is served on both the `CHECKOUT` and `CONNECT` sources. The price
still not showing on Viator is Ventrata → Viator propagation, not us — and two things on
the operator's side: pushes are written as Ventrata pricing rules (`PATCH
/products/{id}/pricing`), so when the operator "removed all rules" our 19 September price
went with them (279.00 → base 229.00); and the promotion *Airbnb - 20% YOS* was still
active on every date checked, served at 183.20 against a 229.00 base. Test protocol for
such a case: one manual push on a date with no promotion, no revert, check the `CONNECT`
source with `scripts/inspect-ventrata-availability.ts --source=CONNECT`, and have the
operator check Viator the same day.

### `ventrata_price_change_units.unitType` is wrong before September 2026 {: #unit-log }

!!! warning "Do not trust this column on historical rows"
    All **16,355,070** rows say `ADULT`.

The row was built from `currentUnitPrice?.type`, but Ventrata sends `unitType`. The read was
`undefined` every time and fell through to `guessUnitTypeFromId`, which answered ADULT for
everything. The one table that could have surfaced the mis-push instead agreed with it: four
pushes, six units each, all labelled ADULT, the correct Adult id absent from every row.

Fixed to read `unitType`, then `type`, then the id hint, with `'UNKNOWN'` as a last resort —
the column is non-nullable, so an unidentifiable unit gets a label a query can find rather
than a guess. **Existing rows were not backfilled.** Any analysis of unit types over history
has to treat pre-fix rows as unlabelled.

---

## Verification and undo

Ventrata is the only rail that can **verify and self-correct**: after pushing it re-reads and,
if the applied price does not match the target, pushes a correction. The history row is written
before the post-push fetch and updated afterwards if a correction happened — so a history row
can legitimately differ from the first thing the vendor reported.

`VENTRATA_VERIFY_AFTER_PUSH` gates it. Undo re-pushes the captured old prices; the mechanism is
documented in `src/ventrata/UNDO_SYSTEM_SUMMARY.md` and
`src/ventrata/PRICE_CHANGE_UNDO_GUIDE.md`.

---

## Tuning

| Variable | Effect |
| --- | --- |
| `VENTRATA_AUTO_APPLY_CONCURRENCY` | In-batch parallelism. Lower this before lowering the Job's task count |
| `VENTRATA_PRICE_PUSH_TIMEOUT_MS` | Per-push HTTP timeout |
| `VENTRATA_GET_AVAILABILITY_TIMEOUT_MS` | Availability read timeout |
| `VENTRATA_PREFETCH_CONCURRENCY` | Parallelism of the availability prefetch |
| `VENTRATA_VERIFY_AFTER_PUSH` | Re-read and correct after pushing |
| `VENTRATA_AUTO_APPLY_DEBUG` / `_VERBOSE` | Batch logging detail |

---

## Onboarding

One credential, still collected and typed in by hand. **No app store exists**, so unlike Bokun
there is no install flow to wire — self-serve entry in our own product is the only lever
available.
