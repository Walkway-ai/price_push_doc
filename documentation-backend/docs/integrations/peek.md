# Peek (Peek Pro)

Phase 1 is in the backend (`src/peek`, PR #637, live in prod since 2026-10-08) and the push
was validated end to end on Peek's sandbox on 2026-10-09: install webhook received, catalogue
read, one override pushed on a start time, price in force read back at the new amount, undo,
price back to base. This page is the rail as it is, plus the portal setup that cost a day to
find and the points still open with Peek. Source for the API: the official TypeScript SDK
[`@peektravel/app-utilities`](https://peek-travel.github.io/app-utilities/html/peek.html) and
the developer portal docs at `apps.peek.com/portal/docs` (login required).

!!! abstract "The one thing to understand"
    Peek prices by **overrides** on a **pricing engine**: one upsert per activity × date range,
    per ticket, optionally filtered by **start time**. Per-slot pricing is native, unit prices
    are native, undo is a re-upsert. Of the four rails it is the closest to what a Walkway push
    is. The integration model is the unusual part: Peek is an **app platform** — the operator
    installs our app, the registry tells us by webhook which install and which API URL — not
    an API key pasted into the SaaS.

---

## What the sandbox run proved

Walkway Test Account (Peek stage, 1 product "Walking Tour", ticket Adult, 40 USD), slot
2026-10-23 16:00, read through `availabilityTimes { price }`:

| Step | Call | Price in force |
| --- | --- | --- |
| before | — | 40.00 USD |
| push | `createEngine` + `upsertOverrides` (fixed 44.00, `startTimeRange [16:00:00,16:00:00]`) | **44.00 USD** |
| undo | `clearOverrides` + `deleteEngine` | 40.00 USD |

So: the override takes effect immediately on the price Peek sells at, the start-time filter
scopes it to the one departure, and the upsert response (`activityContexts`) echoes exactly
what was stored. Nothing to confirm there any more.

---

## The entity model

| Peek entity | What it is | Key |
| --- | --- | --- |
| **App** | Walkway as registered in Peek's registry. One per environment: `walkway` (production), `walkway-test-sandbox` (sandbox) | app slug = `PEEK_APP_ID`, one `shared_secret_key` |
| **Install** | Our app installed on one operator's Peek account | `installId` (JWT subject), `apiUrl` |
| **Product** | An activity, a rental or an add-on | `productId`, `type` (`ACTIVITY` / `RENTAL` / `ADD-ON`), `currency` |
| **Ticket** (resource option) | A bookable sub-option of a product — Adult, Child, a rental size… | `ProductTicket.id` = `resourceOptionId` |
| **Timeslot** | A departure on a date, open/closed, with capacity per ticket | `timeslotId` (`<productId>|…`), `from`/`end` ISO |
| **Pricing engine** | A named container for overrides, optionally scoped to activities | `engineId` (`cpe_…`) |
| **Override** | For an engine × activity × date range: per-ticket prices (fixed or %) gated by filters | `order`, `resourceOptions[]`, `filters[]` |
| **Channel** | A reseller channel, with a `pricingModel` label | `channelId` |

Peek entity → Walkway entity: product = `product_id`, ticket = option (`product_grade_channel_code`
is `<productId><ticketId>Peek`), slot = date + start time, channel = read-only (no per-channel
price to write, see below).

---

## How a price is pushed

Four SDK calls, all on `PricingService`:

| Step | Call | Notes |
| --- | --- | --- |
| Once per install | `createEngine({ name: "Walkway", activityIds? })` | Id persisted on `peek_installs.engine_id`. Empty `activityIds` = every activity. |
| Push | `upsertOverrides({ engineId, dateRange, activities: [{ activityId, overrides }] })` | `dateRange` is a PostgreSQL inclusive range, `[2026-10-23,2026-10-23]` for one day. |
| Undo | re-upsert the previous list, or `clearOverrides({ engineId, dateRange, activityIds })` | "upsert with empty overrides". **Never clear by omitting an activity**: an activity sent with `overrides: []` is cleared. |
| Rename / rescope | `updateEngine`, `deleteEngine` | `deleteEngine` is idempotent. |

One override entry, as stored by the sandbox run:

```json
{
  "order": 0,
  "resourceOptions": [
    { "id": "c3fe552a-…", "mode": "fixed", "price": { "amount": "44.00", "currency": "USD" } }
  ],
  "filters": [
    { "startTimeRange": "[16:00:00,16:00:00]" }
  ]
}
```

- **Per ticket**: `resourceOptions[].id` is the ticket. A push carries every unit it means to
  price, like Ventrata's `unitPricing`.
- **Per slot**: the `startTimeRange` filter (`[HH:MM:SS,HH:MM:SS]`, inclusive) scopes the entry
  to the departure. Native, unlike Bókun (one rate per departure needed) and Xola (rule +
  timeslot link).
- **Per occupancy**: `spotsTaken: { minSpots, maxSpots }` gates on seats already sold. Not
  pushed today; worth knowing for yield rules later.
- `mode: "percentage"` applies a delta to the base price (`> -100`); `mode: "fixed"` is the
  absolute price Walkway computes. We push fixed.
- `order` is precedence when several entries match (lower first). Walkway writes slot entries
  first and a whole-day entry last.
- The response returns `activityContexts[]` with the resolved overrides per activity × date as
  stored: the read-back verification for free. The push is FAILED if the stored context does
  not carry the slot with every pushed amount.

Amounts are decimal strings (`"44.00"`), currency ISO 4217, never numbers.

The backend keeps the override list in force per activity × date on the last APPLIED history
row (`applied_overrides`): Walkway is the only writer on its own engine, so merging a slot
into that list (`mergeSlotOverride`) and re-upserting the whole day is exact. Undo writes the
list minus the slot, plus the slot's previous entry if it had one.

---

## What can be read

| Need | Call | What comes back |
| --- | --- | --- |
| Catalogue | `ProductService.getAllProducts()` | Activities + add-ons, each with `tickets[]` (`id`, `name`, `minPrice`, `maxPrice` across the range) |
| Slots of a day | `TimeslotService.getForDay(productId, date)` | Timeslots, open/closed, capacity, booking count |
| Availability | `AvailabilityService.getAvailabilityTimes({ activityId, date, resourceOptionQuantities })` | Slots with `from`/`end`, `status`, capacity and `taken` per ticket |
| **Price in force on a slot** | raw GraphQL `availabilityTimes(activityId, date, resourceOptionQuantities) { time price { amount currency } }` | The price Peek sells at, base + resolved overrides. The SDK's typed wrapper drops the `price` field; call the proxy directly. |
| Overrides stored | raw GraphQL `pricingOverridesActivityContextsPaginated(filter: { activityIds, dateRange, engineIds })` | The same `activityContexts` the upsert returns, readable any time |
| Resellers | `ResellerService.getAllChannels()` | Channels with a `pricingModel` label |

`availabilityTimes.price` is what the first page of this study said did not exist. It does,
on the backoffice GraphQL behind the registry proxy (schema found by introspection; the SDK
simply does not select it). It is the right source for the old-price snapshot before a push,
the read-back after one, and the `price` table mirror — all three are "not in phase 1" today.

**No per-channel price to write.** Channels are read-only labels. There is no equivalent of
Ventrata's CHECKOUT / CONNECT split: one price, applied to everything Peek sells. A product
mapped under `SPLIT` parity has no reseller leg here.

---

## Authentication and the install flow

One secret does both directions: the app's `shared_secret_key` (`PEEK_APP_SECRET`), HS256.

**Inbound — Peek → us.** Every install notification, webhook and hook carries
`X-Peek-Auth: Bearer <jwt>` with `iss: app_registry_v2`, `sub: <install id>`,
`display_version`, a 60 s `exp`, and `user` when an operator triggered it. The backend
verifies with `parseInstallWebhook(token, body, secret)` and acts on the token's `sub`, never
on the body's `install_id` alone. The install payload:

```json
{
  "install_id": "79394784-…",
  "status": "installed",
  "display_version": "1.0.4",
  "modified_by": { "email": "…", "name": "…" },
  "api": { "url": "https://apps.peek.com/installations-api/walkway-test-sandbox" },
  "account": { "id": "0f78d15b-…", "platform": "peek", "name": "Walkway Test Account",
               "is_test": true, "timezone": "America/Los_Angeles" }
}
```

`status` is `installed` or `uninstalled`; treat uninstall as deactivation, the same
`install_id` can come back. Peek retries a non-2xx six times with backoff and then
**uninstalls the app** — the endpoint must answer 2xx fast and be idempotent on `install_id`.

**Outbound — us → Peek.** The SDK mints `sub = installId`, `iss = PEEK_APP_ID`, short `exp`,
signed with the secret, and sends it as `X-Peek-Auth` (not `Authorization`) to
`<apiUrl>/peek-backoffice-api-v1/` — the registry re-signs the request for the platform
environment, so we never hold a Peek credential. 401 = wrong secret or an install the registry
does not know; `400 invalid_app_id_in_url` = wrong app slug in the URL; `400 Extendable not
installed` = the manifest on the installed version lacks `peek_backoffice_api@v1`.

Rate limiting: the SDK retries HTTP 429 with backoff (`retryDelaysMs`, default 1 s / 2 s /
4 s). No documented quota. Every call is logged in the portal (Logs → API Calls).

Same shape as the Bókun custom app (ENG-2654, see [Bokun — custom app](bokun.md#custom-app)):
an onboarding screen that sends the operator to install the app, a webhook endpoint that
stores the install, and credentials that are per install rather than per user.

---

## Portal setup — what has to be true for the webhook to arrive

Peek's developer portal (`apps.peek.com/portal`) edits an **environment draft**; nothing is
live until **Deploy…** on the environment tile. Four things, learnt the hard way on
2026-10-09 (four reinstalls that never reached us):

1. **Base URL** of the environment = the backend origin (sandbox → the backend running the
   sandbox credentials; production → `https://walkwaysaasbackend-….run.app`). Saving writes
   the draft only.
2. **Manifest**: the install webhook is an extendable that has to be declared, it is not sent
   to the Base URL by itself. Import on the environment tile (whole manifest):

    ```json
    {
      "global": [
        { "slug": "app_registry_webhook@v1",
          "configuration": { "url": "/api/peek/webhooks/install" } }
      ],
      "peek": [
        { "slug": "peek_backoffice_api@v1", "configuration": {} }
      ],
      "acme": null,
      "cng": null
    }
    ```

    `url` is relative to the Base URL. `peek_backoffice_api@v1` takes optional
    `queries` / `mutations` allowlists enforced by the platform; `{}` was enough for reads
    and the pricing mutations on the sandbox.
3. **Deploy** the draft (new version, existing installs upgrade automatically).
4. The account installs that version. Each uninstall/reinstall creates a **new install id**;
   the portal's "Tenant Ref ID" is the account id, not the install id (the install id is in
   the install page URL and in the webhook). Peek's platform env for the sandbox is
   `peek-stage`, but the `apiUrl` is the production registry host for both environments:
   `https://apps.peek.com/installations-api/<app slug>`. The SDK's own default
   (`app-registry.peeklabs.com`) does not answer; always use the `apiUrl` from the webhook.

Backend environment: `PEEK_APP_ID` (app slug, JWT issuer), `PEEK_APP_SECRET`, `PEEK_ENABLED`.
One deployment = one Peek environment; the sandbox credentials are on the prod backend today
(the dev backend does not carry the Peek code), to be swapped for the production app's when
the first real operator installs.

---

## Mapping onto the Walkway push

| Walkway | Peek |
| --- | --- |
| `product_id` | `productId` (activity) |
| option / `product_grade_channel_code` | ticket `id` (`resourceOptionId`) — `<productId><ticketId>Peek` |
| slot `(experience_date, start_time)` | `dateRange = [date,date]` + `filters: [{ startTimeRange: "[HH:MM:SS,HH:MM:SS]" }]` |
| unit prices | `resourceOptions[]`, `mode: "fixed"` |
| push | `POST api/peek/price-push` → `upsertOverrides` on the Walkway engine |
| undo | `POST api/peek/price-push/:id/revert` → re-upsert the previous list for the day |
| source CHECKOUT / CONNECT | one price, no channel dimension |
| credentials | `installId` + `apiUrl` per subscription (`peek_installs`); app secret in env |

Routes: `POST api/peek/webhooks/install` (public, token-verified; a middleware routes any
POST carrying `x-peek-auth` there), `GET api/peek/install`, `GET api/peek/products`,
`POST api/peek/products/sync`, `POST api/peek/price-push`, `POST …/:id/revert`,
`GET …/history`, back office `GET api/admin/peek/installs` and
`PATCH …/:installId/link` (install ↔ subscription when the installing email matches nothing).

Still open with Peek:

1. **Overlapping overrides**: two entries matching the same ticket and time at the same
   `order`, and whether an operator's own engines take precedence over ours (ours should
   lose, like Bókun's operator-schedule guard).
2. **Rentals**: whether overrides apply the same way to `RENTAL` products.
3. **Volume**: any cap on overrides per engine or per date (Bókun's 512 daily rules bit us).

Next in the backend: read `availabilityTimes.price` before and after each push (old-price
snapshot, read-back, `price` mirror), auto-pilot dispatch, the first-push report hook.
