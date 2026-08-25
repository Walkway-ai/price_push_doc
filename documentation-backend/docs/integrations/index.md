# Integrations — how price push actually works

Four rails carry prices to booking platforms: **Ventrata**, **Bokun**, **Xola** and
**Prioticket**. A fifth integration, **Viator**, is a read and linkout path, not a push rail.

They share an entry point and an audit contract. Everything past that differs, sometimes
profoundly — Bokun's `PUT` is neither a replace nor an upsert, Xola needs two objects created
on the vendor side before a price can exist, Ventrata adds tax to what you pushed and hands
it back. Those differences are the reason this section exists.

---

## The shared contract

Everything below is identical across rails. Read it once, then read your rail's page.

**One entry point.**
:   `POST /ventrata/recommendations/auto-apply` takes a list of `compsetIds` and dispatches
    internally on `products.channel`. The route lives on the Ventrata controller for
    historical reasons; it is not Ventrata-specific. Ventrata's sub-batch runs first, then
    Bokun, Xola and Prioticket in parallel.

**Guards applied per recommendation, before any vendor call.**
:   The compset-owner auto-apply toggle (ENG-2431); the OTA-source guard, which stops a price
    sourced from an OTA being pushed to a Direct channel (ENG-2385); and
    `AutoApplyManualLock`, which protects a slot a human pushed by hand within the lock
    window.

**`pushOrigin` is decided by the backend, never by the caller.**
:   `auto_apply` batches stamp `AUTO`. Everything else — API, bulk-with-rules, undo, legacy —
    stamps `MANUAL`. The client's request body cannot influence it. See
    `src/price-push/pricePushOrigin.ts`.

**Every push writes two rows.**
:   A `feature_trace` row with `featureName: 'PRICE_PUSH'`, the normalised input, the vendor
    response, status, duration and per-phase timings; and a vendor history row used for undo
    and for the health digest. Ventrata, Bokun and Xola each have their own
    `*_price_changes` table; Prioticket writes to the vendor-neutral
    `recommendation_price_pushes` (ENG-920).

**Timings are measured per phase**, not just end to end — `src/price-push/pricePushTimings.ts`
splits vendor HTTP time from database time from pool-wait, which is what makes the health
digest able to say *whose* fault a slow batch is.

---

## What differs, at a glance

| | Ventrata | Bokun | Xola | Prioticket |
| --- | --- | --- | --- | --- |
| Auth | `Bearer` token | HMAC-signed | `x-api-key` | `Bearer <jwt>\|<distributorId>` |
| Protocol | OCTO + REST | REST v2 | Native REST | OCTO + custom webhook |
| Write verb | `POST` availability | `PUT` component | `POST`/`PUT` purchase rule | `PATCH` webhook |
| Objects to create first | none | price schedule | **schedule + purchase rule + link** | none |
| Price you push is what you read back | **no** — tax added on top | yes | yes | yes |
| Median push (90 d) | 693 ms | 12.1 s | 5.9 s | low volume |
| Undo | yes | yes | yes | yes |
| Circuit breaker | no | **yes, three-stage** | no | no |

The two cells worth staring at: **Xola requires three vendor objects to exist** before a price
applies, and **Ventrata does not return what you pushed**. Both are covered on their pages.

---

## Per-rail pages

- [Ventrata](ventrata.md) — the reference implementation, and the tax round-trip
- [Bokun](bokun.md) — the entity model and the `PUT` semantics that cost a post-mortem
- [Xola](xola.md) — schedules, purchase rules, and rule reuse by price signature
- [Prioticket](prioticket.md) — the pipe-delimited auth and the Walkway webhook

---

## Adding a rail

The dispatcher, the guards, the audit contract and the digest are all vendor-neutral, so a
new rail is a service plus a controller plus tests. What is *not* optional:

1. A `*.push-origin.spec.ts` proving `auto_apply` stamps `AUTO` and everything else stamps
   `MANUAL`.
2. Dispatcher coverage in `ventrata.service.spec.ts` — the batch must route your channel.
3. A `feature_trace` write on both success and failure, with the vendor error preserved. A
   push that fails without a trace row is invisible to support.
4. An entry in the comparison table above, and a page next to these four.

`docs/PRICE_PUSH_PARTNER_INTEGRATION.md` in the backend repo states the minimum a partner API
has to offer for a rail to be possible at all.
