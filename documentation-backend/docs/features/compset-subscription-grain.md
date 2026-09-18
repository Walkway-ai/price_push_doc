# Compsets at the subscription grain — the soft-release flag

## Why

A compset describes a *product* an operator competes on, and the operator is the
subscription, not the member who clicked "create". Keyed by `userId`, the per-user grain
produced 5,027 redundant compsets across 1,288 (subscription, product) pairs and is the shared
root cause of ENG-1335 (one member's copy leaking a channel), ENG-2393 (a member seeing no
recommendations because they were on someone else's copy) and ENG-2407 (auto-pricing running
off a colleague's copy for eleven days after the owner switched it off).

The regrain (ENG-2552, ENG-2555, ENG-2556, ENG-2558, ENG-2559, ENG-1093) landed on `dev` as
one behaviour for everyone. Before it reaches production it ships behind a **per-subscription
flag**, so one account can run on the new grain while every other account keeps exactly what
`main` does today.

## The flag

`Subscription.compsetAccountGrainEnabled Boolean? @default(false)` — a whitelist, like
`dataHubEnabled`, not an off-switch like `calendarV2Enabled`. SQL in
`prisma/manual/compset-account-grain-flag.sql`, applied with `db push` (no migration file).

| Where it is read | How |
| --- | --- |
| Backend, every gated path | `src/account-grain/account-grain-gate.ts`: `isAccountGrainEnabled(db, subscriptionId)`, `isAccountGrainEnabledForUser(db, userId)` (the subscription the user **owns**, else the earliest one they joined — the same rule `resolveCompsetSubscriptionId` applies, so a compset is never written under one rule and read under another), `accountGrainEnabledMap` for batches. Plain functions over the Prisma client, cached 60 s per process. |
| Front end | JWT claim `compsetAccountGrainEnabled`, stamped on every token path (login, impersonate, signup, Google), resolved server-side in `layout.tsx` and put on the user context. **Read at login only**: a flip lands at the member's next sign-in. |
| Ops override | `ACCOUNT_GRAIN_COMPSETS_FORCE=on\|off` on the backend forces every subscription, for tests and emergencies. |

**Turning it on for one account**: back office → subscription → *Features flag* →
**Compsets at subscription level**, or
`PATCH /api/admin/users/subscription/:id/compset-account-grain {"compsetAccountGrainEnabled": true}`.
Backend effect within a minute; the members' SaaS follows at their next login.

## What the flag gates

Flag **off** = the behaviour `main` has in production, reintroduced line for line next to the
new code. Flag **on** = the regrain.

| Area | Off (main) | On (account grain) |
| --- | --- | --- |
| Push selection (`getPricePushAccounts`) | OWNER/ADMIN members holding `canAutoPricePush` contribute compsets, no `usageFlag` filter, one compset per pgcc ranked (copy owner holds the flag, subscription owner's copy, oldest, id), product-less compsets pass through | every member's **primary** (`SUBSCRIBER`) compsets are candidates; `TEST`, `OFF`, `SECONDARY`, `VIATOR_PARTNER` excluded; `selectPushCompsets` with the account's **governing** pricing config deciding band and toggle; one decision per (pgcc, channel); `decisions[]` on the response |
| Auto-apply owner gate | the compset **user's own** `ProductGradeSetup` row (ENG-2431) | the account's governing row; a mixed batch reads each account its own way |
| Bokun daily-max setup, Xola band and rounding | compset owner's row | governing row |
| Compset writes (`update`, `delete`, `duplicate`, policy, schedules) | any member of the account, behind the permission guard | **OWNER or ADMIN** only; a MEMBER who created a compset cannot edit it |
| New member provisioning | owner's `ProductGradeSetup` rows copied onto the member (+ snapshot) | nothing copied; reads are account-wide |
| Seasonal override create | one row | one row **per member** sharing a `groupId`; update/delete follow the group |
| Duplicate guards on `duplicate`, `create-from-products`, self-comparison | none (main lets a duplicate through) | 409 when the account already covers the product and channel |
| Account-wide reads: `getUserCompsets`, `getCompsetsById`, product/compset status, recommendations v2, AI compset duplicate check, user products, overview product count, auto-pilot alerts, on-demand jobs, mapping SQL | the caller alone | everyone on the account (`resolveReadMemberIds`, `resolveReadAccountContext`) |
| Products caches | `products_cache_version:<userId>` | `products_cache_version:sub:<subscriptionId>` |
| PGS propagation | member rows only | owner side included |
| `getAllUserCompsets` dedupe | recommendation count, then the caller's own copy | a primary beats a non-primary first |
| Front end | manage modal, product page and menus as `main` renders them | every copy listed, account list filter (All / Primary only / Mine / By teammate), "Used / Not used for pricing" badges, owner of each copy, promote / demote |

Deliberately **not** gated, because they close security holes rather than change behaviour:
membership check on the compset policy endpoints (previously ungated), `canManageProducts`
on duplicate, and the on-demand trigger refusing `SECONDARY` (no such row exists without the
demote script).

Already on `main` before the flag, so also ungated: `Compset.subscription_id`, the
account-scoped `getAllUserCompsets`, the read decision in `compset-access.helper.ts`,
`accountAutoApply` / `accountMinPrice` / `accountMaxPrice` on the setup endpoints.

## Data steps, and which ones need a subscription filter

| Script | Effect | Safe for unflagged accounts? |
| --- | --- | --- |
| `backfill-product-grade-setup-subscription.ts` | writes `subscription_id` on PGS rows | yes, additive |
| `collapse-duplicate-product-grade-setups.ts` | sets `retired_into` on exact duplicates | yes, the old path ignores the column |
| `normalize-product-grade-setup.ts` | rewrites every row of a group to the governing values, `autoApply` = OR | changes what unflagged members see; run per account |
| `demote-non-primary-copies.ts` | losing copies → `SECONDARY`, `refreshPolicy = NONE` | **no**: main pushes them with stale data. **Needs a `--subscription` filter before it runs on a flagged account only** |
| `backfill-seasonal-override-groups.ts` | stamps `groupId`, creates ~1,537 mirror rows | **no**: mirrors are independent duplicate windows under the old code. **Needs a `--subscription` filter** |
| `prisma/manual/c5-primary-compset-unique-index.sql` | partial unique index on (subscription, product, channel) for `SUBSCRIBER` | **fleet-wide, incompatible with a soft release**; fails today (1,211 grains with several compsets). After full rollout only |

Also missing before `dev → main`: SQL for `CompsetUsageFlag.SECONDARY`, enum
`CompsetListFilter`, `subscription_members.compset_filter*` and
`pricing_rule_overrides.group_id` (they reached dev through `db push` with no file), and the
`db pull` round-trip that stripped doc comments from `schema.prisma` and renamed the
`(pushedAt, status)` index.

## State on 2026-09-18

- Backend: PR #583 merged into `dev`, deployed on `walkwaysaasbackend-dev`. **Every
  subscription on the dev database is flagged `true`** (3,565 / 3,565) at the product
  owner's request, so dev behaves as before the flag; the soft release is for production.
- Back office: branch `feat/compset-account-grain-toggle` (PR to merge) adds the toggle. The
  deployed back office targets the **production** backend, where the route does not exist yet:
  "Cannot PATCH …/compset-account-grain" means exactly that.
- Front end: branch `feat/compset-account-grain-gate` (PR to merge) gates the UI and fixes two
  `next build` type errors from the products-v2 work (`/api/subscriptions` returned a bare
  string; the products-hub resolver had `return "v2"` hardcoded before the claim read — it now
  honours `productsHubV2Enabled`, on → v2, anything else → v1).
- Production: column absent, nothing changes until `main` has the branch and `db push` ran.

## Test accounts on dev

`5d6066b9` (Walkway test, owner `pricepush2@walkway.ai`): 8 duplicate compset groups across
members, 2 pgccs where `autoApply` diverges between members, 27 seasonal overrides, logins for
OWNER (`pricepush2`), ADMIN (`pierre-alexandre@walkway.ai`) and MEMBER
(`pricepush2.viewer@walkway.ai`). `b4b7b15b` (Boost Portugal): 6 members across all three
roles, 121 duplicate groups, 376 extra copies, heavy push volume — for the scale and for
`scripts/push-set-snapshot.ts` before/after.

## Rollout

1. Merge `dev → main` with the flag `false` everywhere (production unchanged), `db push`.
2. Pilot account: `backfill-product-grade-setup-subscription`, `collapse-duplicate-pgs`,
   `normalize-pgs` (dry run, then apply), `demote-non-primary-copies --subscription`, flag on,
   members re-log, `push-set-snapshot` diff, one week of watching `AUTO_APPLY_DISABLED_BY_OWNER`.
3. Accounts one by one. Then all, the data scripts fleet-wide, the unique index, and a
   cleanup PR removing the legacy paths and the flag.
