# Billing: who gets invoiced for what (ENG-2593)

Not price push, but it lives in the same backend and the same people get paged for it.

## `billing_accounts` — one default Stripe customer per subscription

`BillingAccount` (`subscriptionId` unique, `stripeCustomerId`, `billingEmail`,
`defaultBillingCurrency`, `collectionMethod`, `splitInvoices`, `notes`). Backfilled on
2026-09-14 by `scripts/backfill-billing-accounts-from-stripe.ts` (31 rows): a Stripe customer
is matched to a subscription through, in order, a `revenue_based_subscriptions` link,
`User.stripeCustomerId`, an exact email, then the corporate e-mail domain (public providers
excluded) when it resolves to a single ACTIVE subscription. Duplicate Stripe customers are
ranked (linked > live Stripe subscription > most recent). Existing `stripeCustomerId`s are
never overwritten. Six ambiguous and nine unmatched customers were handed to finance; the
script accepts a `--decisions` file for them.

## `billing_stripe_customers` — several Stripe customers per subscription

One customer per subscription does not describe the accounts. Boost Portugal bills the two
Sintra products to `cus_Ukq0w5YwmDv5wY` and the other five to `cus_UHkZeVIOSC3sNj`; Empire is
one customer but three invoices (Chicago, Washington DC, New York — `invoice_group` on the
entitlements already covers that); Tours For Today wants the variable fee on a different
payment method than the flat fee (not covered yet).

`BillingStripeCustomer` (branch `feat/billing-stripe-customers`, applied in production on
2026-09-15): `stripe_customer_id` **unique**, `subscription_id` **nullable and not unique**,
`product_ids text[]`, `billing_email`, `legal_name`, `label`, `status`, `notes`.
`billing_accounts.stripe_customer_id` stays the default for products no row claims. The
ledger's attribution (`resolveSubscriptionId`) consults this table first; a row with a null
subscription is a deliberate "no subscription", not a fall-through.

Boost Portugal is seeded (`scripts/seed-billing-stripe-customers-boost.ts`, idempotent
upsert, refuses a product listed under two customers). **The invoice run still receives the
Stripe customer from the pipeline**: until the pipeline reads `product_ids` (or a resolver
endpoint exists), the Sintra lines still go to the default customer.

## Stripe webhook (ENG-2597)

`POST /stripe/webhook` verifies the signature with `STRIPE_WEBHOOK_SECRET` (read at service
construction, raw body via `rawBody`), answers 400 unsigned, and posts payment failures to
`SLACK_BILLING_ALERTS_WEBHOOK_URL`. Verified end to end in the **sandbox** on 2026-09-14
(13 deliveries, 201, `invoice.payment_failed` processed, Slack 200). The **live** destination
still has to be created in the Walkway Stripe account with the same URL and its `whsec_`
set on `walkwaysaasbackend`; a sandbox secret was pasted in a chat during the test and
should be rotated.
