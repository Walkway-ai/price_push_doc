# Time-slot exclusions on the auto-apply (ENG-2584)

## Why

World of Illusion runs promotional departures at fixed times whose price must not move: the
09:00–12:00 window and the 19:00–20:30 window on their Ventrata product. Auto-pilot has no
notion of "this slot is off limits"; the only lever was to switch the whole product off.

## What it does

`price_push_slot_exclusions` (Prisma `PricePushSlotExclusion`) holds, per subscription and
optionally per product (`productGradeChannelCode`), a start time, an end time and the
weekdays it applies to. Every auto-apply rail — Ventrata, Xola, Bokun, Prioticket — runs the
pure matcher `src/price-push/slot-exclusion.ts` over its recommendations **before** pushing;
a matching slot is skipped, counted in the `slot-excluded` bucket and recorded as
`Skipped: Time-slot exclusion (ENG-2584)` so the history explains the gap.

Manual pushes are not affected: an operator may still push a promotional slot by hand.

## Managing exclusions

`api/price-push/slot-exclusions` (JWT + `canEditPricingRules`): list, create, update, delete,
scoped to the caller's subscription. Seed by hand with
`scripts/seed-price-push-slot-exclusions.ts` when onboarding an account.

Schema applied with `prisma db push` after a `db pull` (no migration file); the SQL is kept in
`prisma/manual/eng-2584-price-push-slot-exclusions.sql` for review.

!!! warning "The module must import `SubscriptionsModule`"
    The first deploy (PR #568) failed to boot: *"Nest can't resolve dependencies of the
    SubscriptionPermissionGuard"*. `SlotExclusionModule` imports `PrismaModule` and
    `SubscriptionsModule` since the hotfix PR #569. Any new module that uses the guard needs
    the same two imports.

## State

World of Illusion is seeded in production: two rows on subscription `bde2237f…` for pgcc
`89af1e69-9a18-43d4-b4bd-a4b4b0b8f7b4DEFAULTVentrata` (09:00–12:00 and 19:00–20:30). Their
`ProductGradeSetup.autoApply` was still `false` at the time of writing; re-enabling it is the
operator-facing step that makes the exclusions matter.
