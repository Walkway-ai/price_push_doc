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
