# Walkway SaaS Backend

How prices get from a recommendation to a booking platform, what breaks, and what to do
about it.

!!! tip "First time here? Two pages carry most of the load."
    [**Integrations — how price push actually works**](integrations/index.md) for the shared
    contract and the four rails.
    [**Backend Handover Runbook**](operations/backend-handover.md) for what is deployed, what
    is on fire, and what to do at 2am.

---

## Start from what you are doing

| I want to… | Go to |
| --- | --- |
| Understand the whole path, once | [How price push works](integrations/index.md) |
| Work on a specific booking platform | [Ventrata](integrations/ventrata.md) · [Bokun](integrations/bokun.md) · [Xola](integrations/xola.md) · [Prioticket](integrations/prioticket.md) |
| Fix something that is broken right now | [Triage ladder](operations/backend-handover.md#triage-ladder) |
| Know what is deployed, and where | [What is deployed](operations/backend-handover.md#what-is-deployed) |
| Turn something off in a hurry | [Kill switches](operations/backend-handover.md#playbooks) |
| Know what is half-finished | [Open operational items](operations/backend-handover.md#open-items) |
| Call the API | [API endpoints](price-push/api-endpoints.md) |
| Configure pricing as a user | [Configuration guide](price-push/configuration.md) |

---

## Start from a symptom

| Symptom | Where the answer is |
| --- | --- |
| One operator's prices did not move | [P1](operations/backend-handover.md#p1) |
| The batch has not run at all | [P2](operations/backend-handover.md#p2) |
| The batch takes hours, or two run at once | [P3](operations/backend-handover.md#p3) |
| Bokun is failing or slow | [P4](operations/backend-handover.md#p4) · [circuit breaker](integrations/bokun.md#reliability-machinery) |
| A schedule stopped firing | [P5](operations/backend-handover.md#p5) |
| Notifications stopped, API is fine | [P6](operations/backend-handover.md#p6) |
| A price landed on the wrong ticket category | [Ventrata unit selection](integrations/ventrata.md#unit-selection) |
| We are selling a slot the operator retired | [Xola: the operator's schedule wins](integrations/xola.md#operator-schedule) |
| An operator says checkout got slow | [Xola: `PUT` merges actions](integrations/xola.md#action-merge) |
| A Bokun operator cannot finish connecting | [The custom app](integrations/bokun.md#custom-app) |
| A Bokun product should move to daily pricing | [Daily pricing, verified live](integrations/bokun.md#daily-pricing) |
| Ventrata pushes show `no_adult_price` | [The price table lost every push](integrations/ventrata.md#no-adult-price) |
| A member cannot edit a compset, or sees a colleague's | [Compsets at the subscription grain](features/compset-subscription-grain.md) |
| The push went through but the price is wrong | [Ventrata tax round-trip](integrations/ventrata.md) · [Bokun `PUT` semantics](integrations/bokun.md) |

---

## Read these before changing anything

Three failures that were expensive, are not obvious from the code, and are easy to repeat.

**A vendor's field names are not what the comment says.**
:   Bokun returns `legacyApiCredentials`, not the `restApiCredentials` our own comment
    claimed. Ventrata sends `unitType`, not `type`. Both cost real incidents. Read the
    response, do not trust the note about it.
    [Bokun](integrations/bokun.md#custom-app) · [Ventrata](integrations/ventrata.md#unit-log)

**Creating an object on the vendor side has customer-visible consequences.**
:   A Xola schedule is `type: 'available'`. Creating one to hang a price on puts that slot
    **on sale**, and customers book departures that will not run.
    [Xola](integrations/xola.md#operator-schedule)

**"Update" does not mean replace.**
:   Bokun's `PUT` is neither a full replace nor an upsert, and base rules and schedule rules
    follow opposite rules about omission. Xola's `PUT /purchaseRules/{id}` **merges** the
    actions array with no dedupe — that grew to 91,593 actions on one account.
    [Bokun](integrations/bokun.md) · [Xola](integrations/xola.md#action-merge)

---

## Feature documentation

The Price Push section is written for people configuring and using the product, rather than
for people changing it.

- [Overview](price-push/overview.md) — what price push is
- [How it works](price-push/how-it-works.md) — the architecture
- [Configuration guide](price-push/configuration.md) — setup, step by step
- [Price unit management](price-push/price-units.md) — ADULT, CHILD, SENIOR and the rest
- [Availability management](price-push/availability.md) — dates, times, slots
- [Price change history](price-push/history.md) — the audit trail
- [Undo and revert](price-push/undo.md) — recovering from a bad push
- [Use cases](price-push/use-cases.md) — worked examples
- [API endpoints](price-push/api-endpoints.md) — request and response formats
- [Troubleshooting](price-push/troubleshooting.md) — common problems

### Features

- [Compsets at the subscription grain](features/compset-subscription-grain.md) — the per-subscription soft-release flag, what it gates, how to enable one account
- [Time-slot exclusions](features/price-push-slot-exclusions.md) — keep auto-pilot off a promotional departure (ENG-2584)
- [Billing accounts and Stripe customers](features/billing-accounts.md) — who gets invoiced for what, the webhook
- [Market intelligence access flag](features/market-intelligence-access-flag.md)

---

## Keeping this true

A page that describes behaviour which no longer exists is worse than no page: it is trusted.
When you change a rail, change its page in the same PR, and add the reasoning — the diff
already records what changed, the page has to record **why**, and what it cost to find out.

The [handover runbook](operations/backend-handover.md) carries the operational state: what is
deployed, what is in flight, and what is open. It goes stale fastest, so check its dates
before relying on it.
