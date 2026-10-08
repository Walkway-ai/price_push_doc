# Peek (Peek Pro) — integration study

Not integrated yet. This page is the read of Peek's partner surface done on 2026-10-08,
before any code: what the price-push model is, how it maps onto ours, and what has to be
confirmed with Peek before building. Source: the official TypeScript SDK
[`@peektravel/app-utilities`](https://peek-travel.github.io/app-utilities/html/peek.html)
(there is no public Swagger; the SDK wraps Peek's "app installations" API). Everything below
is what that SDK exposes, nothing more.

!!! abstract "The one thing to understand"
    Peek prices by **overrides** on a **pricing engine**: one upsert per activity × date range,
    per ticket, optionally filtered by **start time**. Per-slot pricing is native, unit prices
    are native, undo is a clear. Of the four rails it is the closest to what a Walkway push is.
    The catch is the integration model: Peek is an **app platform** — the operator installs our
    app, we get an install id and an API URL by webhook — not an API key pasted into the SaaS.

---

## The entity model

| Peek entity | What it is | Key |
| --- | --- | --- |
| **Install** | Our app installed on one operator's Peek account | `installId` (JWT subject), `apiUrl` |
| **Product** | An activity, a rental or an add-on | `productId`, `type` (`ACTIVITY` / `RENTAL` / `ADD-ON`), `currency` |
| **Ticket** (resource option) | A bookable sub-option of a product — Adult, Child, a rental size… | `ProductTicket.id` = `resourceOptionId` |
| **Timeslot** | A departure on a date, open/closed, with capacity per ticket | `timeslotId` (`<productId>|…`), `from`/`end` ISO |
| **Pricing engine** | A named container for overrides, optionally scoped to activities | `engineId` |
| **Override** | For an engine × activity × date range: per-ticket prices (fixed or %) gated by filters | `order`, `resourceOptions[]`, `filters[]` |
| **Channel** | A reseller channel, with a `pricingModel` label | `channelId` |

Peek entity → Walkway entity: product = `product_id`, ticket = option (`product_grade_channel_code`
is a product × ticket), slot = date + start time, channel = read-only (no per-channel price to
write, see below).

---

## How a price is pushed

Four SDK calls, all on `PricingService`:

| Step | Call | Notes |
| --- | --- | --- |
| Once per install | `createEngine({ name: "Walkway", activityIds? })` | Store the returned `id`. Empty `activityIds` = every activity. |
| Push | `upsertOverrides({ engineId, dateRange, activities: [{ activityId, overrides }] })` | `dateRange` is a PostgreSQL inclusive range, `[2026-10-29,2026-10-29]` for one day. |
| Undo | `clearOverrides({ engineId, dateRange, activityIds })` | "upsert with empty overrides". **Never clear by omitting an activity**: an activity sent with `overrides: []` is cleared. |
| Rename / rescope | `updateEngine`, `deleteEngine` | `deleteEngine` is idempotent. |

One override entry:

```json
{
  "order": 0,
  "resourceOptions": [
    { "id": "<ticket id>", "mode": "fixed", "price": { "amount": "118.00", "currency": "MXN" } },
    { "id": "<child ticket id>", "mode": "percentage", "percentageAdjustment": "-15" }
  ],
  "filters": [
    { "startTimeRange": "[10:00:00,10:00:00]" },
    { "spotsTaken": { "minSpots": 0, "maxSpots": 5 } }
  ]
}
```

- **Per ticket**: `resourceOptions[].id` is the ticket. A push carries every unit it means to
  price, like Ventrata's `unitPricing`.
- **Per slot**: the `startTimeRange` filter (`[HH:MM:SS,HH:MM:SS]`, inclusive) scopes the entry
  to the departure. Native, unlike Bókun (one rate per departure needed) and Xola (rule +
  timeslot link).
- **Per occupancy**: `spotsTaken` gates on seats already sold (zero-indexed). Not something we
  push today; worth knowing for yield rules later.
- `mode: "percentage"` applies a delta to the base price (`> -100`); `mode: "fixed"` is the
  absolute price Walkway computes. We push fixed.
- `order` is precedence when several entries match (lower first). The SDK says nothing about
  what happens when two entries at the same `order` overlap — **confirm with Peek**.
- The response returns `activityContexts[]`: the resolved overrides per activity × date as
  stored. That is the read-back verification for free, the thing Xola made us build (ENG-2707).

Amounts are decimal strings (`"118.00"`), currency ISO 4217, never numbers.

---

## What can be read

| Need | Call | What comes back |
| --- | --- | --- |
| Catalogue | `ProductService.getAllProducts()` | Activities + add-ons, each with `tickets[]` (`id`, `name`, `minPrice`, `maxPrice` across the range) |
| Slots of a day | `TimeslotService.getForDay(productId, date)` | Timeslots, open/closed |
| Availability | `AvailabilityService.getAvailabilityTimes({ activityId, date, resourceOptionQuantities })` | Slots with `from`/`end`, `status`, capacity and `taken` per ticket |
| Resellers | `ResellerService.getAllChannels()` | Channels with a `pricingModel` label |

**No "current price of this slot" read.** `minPrice`/`maxPrice` are a range over the
product, and availability carries capacity, not price. The price in force on a slot has to be
derived from the base price plus the resolved overrides (`activityContexts` after an upsert,
or whatever Peek exposes for a read — **confirm**). This matters for the `price` table mirror
and for the old-price snapshot every undo relies on.

**No per-channel price to write.** Channels are read-only labels. There is no equivalent of
Ventrata's CHECKOUT / CONNECT split: one price, applied to everything Peek sells. A product
mapped under `SPLIT` parity has no reseller leg here.

---

## Authentication and the install flow

Peek apps, not API keys:

1. The operator installs the Walkway app from Peek. Peek sends an **install webhook** with
   the `installId` and the install's `apiUrl` (e.g.
   `https://apps.peek.com/installations-api/<app>`); both are persisted per subscription.
2. Every call is signed with a JWT: `jwtSecret` = the app's secret, `issuer` = the app id,
   subject = `installId`. The SDK mints and caches it (`tokenTtlSeconds`, refresh leeway 60 s).
3. Requests Peek sends us carry `x-peek-auth`; `verifyPeekAuthToken()` checks signature,
   expiry, issuer `app_registry_v2`.
4. Gateway modes: `v1` (backoffice gateway, needs `gatewayKey` as `pk-api-key`) and `v2`
   (installations API). New integrations go `v2`.

Rate limiting: the SDK retries HTTP 429 with backoff (`retryDelaysMs`, default 1 s / 2 s /
4 s). No documented quota.

Same shape as the Bókun custom app (ENG-2654, see [Bokun — custom app](bokun.md#custom-app)):
an onboarding screen that sends the operator to install the app, a webhook endpoint that
stores the install, and credentials that are per install rather than per user.

---

## Mapping onto the Walkway push

| Walkway | Peek |
| --- | --- |
| `product_id` | `productId` (activity) |
| option / `product_grade_channel_code` | ticket `id` (`resourceOptionId`) — one PGCC per product × ticket |
| slot `(experience_date, start_time)` | `dateRange = [date,date]` + `filters: [{ startTimeRange: "[HH:MM:SS,HH:MM:SS]" }]` |
| unit prices | `resourceOptions[]`, `mode: "fixed"` |
| push | `upsertOverrides` on the Walkway engine |
| undo | `clearOverrides` for that date and activity — **this clears every ticket's override on the date**, not one unit; keep the previous override set in the history row and re-upsert it, as the Bókun daily-pricing undo does |
| source CHECKOUT / CONNECT | one price, no channel dimension |
| credentials | `installId` + `apiUrl` per subscription; app secret in env |

Open points to settle with Peek before building:

1. **Read the price in force** on a slot (base + overrides), for the old-price snapshot and
   the `price` table mirror.
2. **Overlapping overrides**: two entries matching the same ticket and time, same `order`.
   Also whether an operator's own engines take precedence over ours (ours should lose, like
   Bókun's operator-schedule guard).
3. **Sandbox**: an operator test account where `createEngine` / `upsertOverrides` can run.
4. **Rentals**: whether overrides apply the same way to `RENTAL` products.
5. **Volume**: any cap on overrides per engine or per date (Bókun's 512 daily rules bit us).

---

## What this would look like in the backend

Same shell as the other three rails, smaller:

- `src/peek/`: `peek-apps.service.ts` (install webhook, credentials per subscription),
  `peek.service.ts` (`updateSlotPriceWithHistory` → `upsertOverrides`, undo →
  re-upsert the snapshot or `clearOverrides`), `peek_price_changes` + units table, a
  `pushOrigin`, the manual lock, the first-push report hook.
- Catalogue sync from `getAllProducts()`: one `Product` row per product × ticket, channel `peek`.
- Pricing mode on the option: fixed only. No parity split, no daily-max.

Nothing here is started; this page is the brief.
