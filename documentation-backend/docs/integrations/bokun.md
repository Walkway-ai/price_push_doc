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
| Reorder schedules (2.1.19) | `POST` | `/restapi/v2.0/pricing/schedules/reorder` |
| Read daily prices (2.1.19) | `GET` | `/restapi/v2.0/experience/{id}/dailyPricing?from&to` |
| **Write daily prices** (2.1.19) | `POST` | `/restapi/v2.0/experience/{id}/dailyPricing` |
| Delete a daily price (2.1.19) | `DELETE` | `/restapi/v2.0/experience/{id}/dailyPricing?travelDate&rateId&priceCatalogId` |

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

## Daily pricing and schedule priority (REST 2.1.19) {: #daily-pricing }

Bokun shipped REST v2 spec **2.1.19** in September 2026. Nothing was deprecated; two things
were added, and both matter to the push. They live on the branch `feat/bokun-daily-pricing`
(backend) and are **off by default**.

### Daily pricing: one rule per date

A product can be switched, on Bokun's side, from price schedules to *daily pricing*. In that
mode a price is a flat rule keyed by travel date:

```json
{ "travelDate": "2026-10-01", "rate": { "id": 2331221 }, "priceCatalogId": 1,
  "pricingCategoryId": 10, "amount": "42.00", "currency": "EUR" }
```

`POST /experience/{id}/dailyPricing` with `{ "items": [...] }` writes the rules sent and
**leaves every other rule alone** (max 512 rules per product). Compare with the `PUT` above:
no schedule to create for the date, no body that must carry every base and schedule rule the
experience ever had, no risk of deactivating a rule by omission. This is the shape a Walkway
push has always wanted.

Two constraints:

- A product still on schedules answers **409** `Experience#… is not configured to use daily
  pricing`. The switch is done by the operator in the Bokun UI (product > Pricing), not by us.
- `GET` accepts a `from`/`to` window of at most two months.

How the backend uses it (`updateBasePriceWithHistory`, `src/bokun/bokun-daily-pricing.ts`):

1. **Mode** is decided once per push, before any schedule would be created.
   `BOKUN_DAILY_PRICING_MODE` is `off` (default, nothing changes), `on` (every date push goes
   through daily pricing) or `auto` (one `GET dailyPricing` per experience, cached six hours:
   200 means daily, 409 means schedules). `BOKUN_DAILY_PRICING_EXPERIENCES` is a
   comma-separated allowlist that forces daily mode for those experiences whatever the global
   switch says. That allowlist is the intended rollout tool: one experience, then one operator.
2. In daily mode the push builds `pushes × catalogs` items for the one travel date, reads the
   current rules for that date, and either skips (same no-op rule as the `PUT` path,
   `BOKUN_SKIP_NOOP`) or issues one `POST`. No price schedule is created or cached, and the
   history row has `bokunScheduleId = null`.
3. **Undo** works. The history row stores a snapshot under `revertPricingBody` marked
   `__bokunDailyPricing`: the rules that existed for the touched tuples and the rules pushed.
   Undo `POST`s the previous rules back and `DELETE`s the tuples that had none, so the date
   falls back to the base price exactly as before. A no-op push stores nothing to revert and
   undo says so.
4. Dry run (`BOKUN_PRICE_PUSH_DRY_RUN`) logs the items it would `POST`.

!!! note "What does not change in daily mode"
    Rate resolution, currency and catalog filtering, pricing-rule expansion (Child = 80 % of
    Adult), the drift guard, the circuit breaker, the price-table mirror and the history row are
    all the same code. Only the write and the undo differ.

### Schedule priority: the top one wins

When two price schedules cover the same date, Bokun applies **the first in the list** (help
center: "the price schedule at the top of the list overrides the ones below it"). A new
schedule lands at the bottom. Every Walkway single-day schedule created under an operator's
seasonal schedule was therefore ignored on that date, and the push still reported success:
the `PUT` was accepted, the rule existed, the operator's rule simply outranked it.

`POST /pricing/schedules/reorder` with `{ "priceScheduleIds": [...] }` sets the order. The
backend now calls it right after creating a schedule (`getOrCreateScheduleForDate`): every
schedule id known in `bokun_supplier_price_schedules` for the subscription goes first, the
operator's schedules keep their relative order. The call is non-fatal and logged as
`[REST Price Push] Reordered …`; `BOKUN_REORDER_SCHEDULES=false` turns it off.

The backlog (schedules created before this landed) is fixed once per subscription:

```
npx ts-node -r tsconfig-paths/register scripts/bokun-reorder-schedules.ts --subscription <id>
npx ts-node -r tsconfig-paths/register scripts/bokun-reorder-schedules.ts --all
```

### Rollout plan

1. Merge with the defaults (`BOKUN_DAILY_PRICING_MODE=off`, reorder on). Watch for
   `Reordered` lines and for any `Schedule reorder failed` warning.
2. Switch the test product **1174595** (`pricepush2@walkway.ai`) to daily pricing in the Bokun
   UI, set `BOKUN_DAILY_PRICING_EXPERIENCES=1174595`, push one date, check the `GET`, undo,
   check again.
3. Widen the allowlist operator by operator, only for products the operator has switched.
   `auto` is for later, once several operators are on daily pricing.

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
| `BOKUN_DAILY_PRICING_MODE` | `off` | `on` or `auto` switches date pushes to `POST dailyPricing` (see [above](#daily-pricing)) |
| `BOKUN_DAILY_PRICING_EXPERIENCES` | — | Comma-separated experience ids forced into daily mode |
| `BOKUN_REORDER_SCHEDULES` | on | Move Walkway schedules to the top after creating one |

---

## Known edge cases

Documented with reproduction detail in section 9 of `docs/BOKUN_PRICE_PUSH.md`:

- a schedule is created but does not cover the target date, so the price silently falls back
- a schedule covers the date but sits below an operator's seasonal schedule, so the operator's
  price wins (fixed by the [reorder](#daily-pricing); run the backlog script on old accounts)
- `applyPricingRules` returns an unexpected amount
- rate id resolution fails and `rateId` comes back `null`
- OCTO availability and the `PRICING` component disagree

Diagnostic scripts live in `scripts/` — `inspect-bokun-experience.ts`,
`inspect-bokun-schedule.ts`, `inspect-bokun-octo-availability.ts`,
`audit-bokun-schedule-cache.ts`.

---

## Onboarding: the custom app {: #custom-app }

Historically three separate credentials collected by hand. As of September 2026 a new
operator **pastes nothing**: they give their Bokun short name, approve Walkway on Bokun's own
consent screen, and the callback writes the credentials onto the subscription.

### The two entry points, and why only one works

```
FROM WALKWAY  (the only complete path)
  /connect → "Your Bokun address" → Continue to Bokun
    → GET /bokun/oauth/start        authenticated, canEditSubscription
    → state row created WITH subscriptionId + userId
    → {domain}.bokun.io/appstore/oauth/authorize   ← operator approves
    → GET /bokun/oauth/callback     state redeemed, subscription known
    → credentials written → /connect?bokun=connected

FROM BOKUN'S APP STORE
  Bokun → GET /bokun/install        no session at all
    → state row WITHOUT subscriptionId
    → … install saved, subscriptionId = null, NO credentials written
    → /connect?bokun=unclaimed
```

Both build the identical authorize URL, so Bokun cannot tell them apart; the difference is
entirely on our side — whether the pending state carries a subscription.

!!! warning "An app-store install cannot be attributed"
    Nothing in that request says which Walkway account the vendor is. The July 2026 install
    sat with `subscriptionId: null` for two months for exactly this reason. **The claim
    screen that would finish the binding does not exist yet.** Until it does, direct new
    operators to start from `/connect`.

    Never infer the account from products, currency or company name. Bokun does report
    `appInstalledByUserEmail`, which is stored to *propose* a subscription — it says who
    installed on Bokun's side, not who may speak for a Walkway account.

### What Bokun actually returns

Read off a real install on 2026-09-08 (`walkway-pizza`, all six scopes), logged as a shape
with every value withheld:

```
{ access_token: <string:32>,
  appInstalledByUserEmail: <string:27>,
  appInstalledByUserFirstName: <string:2>, appInstalledByUserLastName: <string:7>,
  legacyApiCredentials: { accessKey: <string:32>, secretKey: <string:32> },
  pricing: { chargeType: <string:15>, pricePlan: object },
  scope: <string:90>, vendor_id: <string:18> }
```

!!! danger "The field is `legacyApiCredentials`"
    Not `restApiCredentials`, which is what the code comment claimed. Reading the documented
    name yields `undefined`, stores blank credentials, and fails much later at push time,
    far from the cause.

**There is no OCTO token.** The two keys are the shape of the Access/Secret pair operators
paste today, so they map straight onto `bokunApiKey` / `bokunSecretKey` and the REST v2
price push works untouched. `bokunOctoToken` is left alone — existing customers keep theirs.

That matters for one internal caller: `getUnitTypesByExperienceId` still reads OCTO via
`getBokunOctoTokenOrThrow`. It is **the only OCTO dependency in the pricing chain** and needs
a REST v2 equivalent before an OAuth-only operator can be fully configured. The other four
OCTO callers (`getProducts`, `getProductById`, `getAvailability`, `getBookings`) are
controller surface taking an `x-api-token` header, outside the push path.

### App credentials — one custom app per operator {: #bokun-apps }

Bokun will not give Walkway a public App Store app, and a custom app can only be installed
on the vendor account that created it. So every operator has **their own** custom app, and
onboarding N operators means N client id/secret pairs. Since September 2026 those live in the
`bokun_apps` table, not in the environment:

| Column | Meaning |
| --- | --- |
| `domain` | The vendor's Bokun short name (`walkway-pizza` in `walkway-pizza.bokun.io`), unique. `null` = fallback app used for any vendor without a row. |
| `clientId` / `clientSecretEnc` | The app's credentials. The secret is sealed with AES-256-GCM under **`BOKUN_APP_SECRETS_KEY`** (`src/common/utils/secret-box.ts`) and is never returned by any endpoint or written to a log — the back-office sees a 4-character hint. |
| `scopes`, `active`, `label`, `notes` | Housekeeping. Deactivate rather than delete: installs and pending states point at the row. |

`BOKUN_APP_CLIENT_ID` / `BOKUN_APP_CLIENT_SECRET` still work as the **last fallback** (the
pre-registry app), so the installs made before the table keep their meaning.

**Which app signs what** (`BokunAppsService`):

```
/bokun/oauth/start?domain=X   → app for X, else fallback row, else env pair; appId written on the state
/bokun/oauth/callback         → state peeked (not consumed) → its appId → HMAC verified with THAT secret
                                → only then the state is consumed and the code exchanged with the same app
/bokun/install (cold, from Bokun) → domain's app, else every active app tried against the HMAC, else env
```

!!! warning "Secrets need the key, and the key needs to exist before the first row"
    Without `BOKUN_APP_SECRETS_KEY` the registry refuses to store or read anything
    (`BadRequestException`), on purpose: the alternative is a plaintext client secret in
    Postgres. Generate it once (`openssl rand -base64 32`), keep it in Secret Manager, and
    never rotate it without re-sealing every row (`scripts/register-bokun-app.ts --list`
    shows `(unreadable)` for rows sealed under another key).

**Onboarding an operator, step by step**

1. The operator creates a custom app in their Bokun account (Settings → Apps) with these two
   URLs **character for character** and all six scopes — without `LEGACY_API` no credentials
   come back and the callback lands on `?bokun=failed&reason=no-credentials`:

    ```
    Install URL   {BACKEND}/bokun/install
    Redirect URL  {BACKEND}/bokun/oauth/callback
    ```

    `GET api/admin/bokun-apps/registration-urls` returns them filled in.

2. They send us the app's client id and secret. We register the row:
   `POST api/admin/bokun-apps { label, domain, clientId, clientSecret }` from the back-office, or

    ```
    DATABASE_URL=<prod> BOKUN_APP_SECRETS_KEY=<key> BOKUN_NEW_APP_SECRET=<secret> \
      npx ts-node -r dotenv/config scripts/register-bokun-app.ts \
      --label "<Operator> custom app" --domain <short name> --client-id <id> --client-secret-env BOKUN_NEW_APP_SECRET
    ```

3. The operator opens `/connect`, types their short name, approves on Bokun's consent screen.
   The callback writes `bokunApiKey` / `bokunSecretKey` onto the subscription. Done — they
   pasted nothing.

Schema for `bokun_apps` and the `appId` columns: **`prisma/manual/bokun-apps.sql`**, applied
by hand before the deploy (same `db push` convention as the OAuth tables).

`BOKUN_OAUTH_REDIRECT_BASE_URL` must point at the **backend**. If unset it falls back to
`FRONTEND_URL` and the redirect URI stops matching what Bokun has registered.

### State lives in the database

`bokun_oauth_states`, not memory. It was a `Map` on the service instance; production runs
1–5 Cloud Run instances, so the install could be served by one and the callback by another —
`Invalid or expired state` with nothing actually wrong — and any deploy between the two steps
did the same. Redemption is a conditional `updateMany`, so two concurrent callbacks cannot
both bind. TTL 15 minutes.

Schema for this and for `bokun_oauth_installs` lives in
**`prisma/manual/bokun-oauth-state-and-install-binding.sql`**. The deploy image runs
`prisma generate` only — no `migrate deploy` — so **run the SQL before deploying code that
reads these columns**. It is idempotent.
