# Prioticket

The smallest rail by volume, and the only one whose write path is a **webhook Prioticket built
for us** rather than a standard endpoint.

Base URL: `https://distributor-api.prioticket.com`

---

## Authentication — the pipe convention

Prioticket is OCTO-compliant for reads, but its auth is unlike any other rail. The operator
identifier is appended to the bearer token after a **pipe character**:

```http
Authorization: Bearer <jwt>|<distributorId>
Content-Type: application/json
User-Agent: Walkway/1.0
```

The JWT is a **Firebase** token. What goes after the pipe depends on which kind of credential
you hold:

- an **admin** token pairs with the operator's `distributorId`
- a **distributor** token pairs with a `supplierId`

The caller decides which, based on the credential type. Passing the wrong one authenticates
fine and then returns empty or wrong data, which is a confusing failure — check the pairing
before suspecting the endpoint.

!!! warning "Tokens expire in about an hour"
    For v1 the operator supplies a **long-lived JWT**, rotated by hand through settings. There
    is no service-account refresh flow yet; it is tracked as a follow-up. If Prioticket pushes
    start failing with 401 across the board, an expired stored token is the first hypothesis,
    and the fix is a credential rotation, not a deploy.

### Credential resolution

Same subscription-level pattern as the other rails, first hit wins:

1. explicit request headers, **only when both** the JWT and the pipe value are present
2. `subscription.prioticketApiToken` + `subscription.prioticketDistributorId`
3. the user's stored copy — the backoffice sets credentials on the subscription and propagates
   them to member users

---

## Endpoints

| Action | HTTP | Path |
| --- | --- | --- |
| Product detail | `GET` | `/v1.0/octo/products/{productId}` |
| Availability | `POST` | `/v1.0/octo/availability` |
| Availability calendar | `POST` | `/v1.0/octo/availability/calendar` |
| **Push pricing** | `PATCH` | `/v1.0/webhook/walkway/availability` |

The first three are standard OCTO. The fourth is custom: Prioticket exposes no OCTO write path
for pricing, so they built a webhook for Walkway. It is a `PATCH`, not a `POST` — and the body
is the **OCTO availability shape carrying a new `unitPricing` array**, which is why it looks
like a read response.

```json
{
  "id":       "<availability id>",
  "productId": "13829",
  "optionId":  "13829",
  "unitPricing": [ … ]
}
```

`productId` and `optionId` are frequently the same value on this vendor. That is normal, not a
copy-paste bug.

---

## The push, step by step

`PrioticketService.updateAvailabilityPriceWithHistory`.

1. **Resolve credentials** in the order above.
2. **Resolve the unit to price.** `resolveAdultUnitId` reads the OCTO product detail, finds the
   option by id — *falling back to the first option* if the id does not match — and inside it
   looks for a unit whose `type` or `reference` uppercases to `ADULT`, **falling back to the
   first unit**. Both fallbacks are silent, and any failure returns `null` rather than
   throwing.
3. **Find the availability slot.** `matchAvailabilitySlot` filters by date prefix on
   `localDateTimeStart`, then matches `T<HH:mm>` within that day. If the date filter yields
   nothing it searches the whole list; if the time does not match it **returns the first
   candidate**.
4. **Apply pricing rules and rounding**, via the shared `PricingCalculatorHelper` — the same
   helper Ventrata uses.
5. **`PATCH` the webhook.**
6. **Write the audit row** inside a Prisma transaction.

!!! danger "Two silent fallbacks in one path"
    Steps 2 and 3 each fall back to "the first one" rather than failing. A slot mismatch can
    therefore push a correct price to the **wrong time**, or an option's price to a different
    option, and report success. Volume is low enough that this has not bitten us visibly — but
    when a Prioticket price looks misapplied, verify which availability id and unit id the
    `feature_trace` row actually recorded before looking anywhere else.

---

## Audit and undo

Prioticket is the one rail that does **not** have its own `*_price_changes` table. It writes to
the vendor-neutral **`recommendation_price_pushes`** (ENG-920), which is the pattern any new
rail should follow.

The write happens **inside a Prisma transaction** wrapping the external call, so the audit row
can never desync from the vendor state — a deliberate difference from the older rails, where
the history write and the HTTP call are separate.

Undo re-pushes the captured **old** unit prices, syncs the local `price` table back, and marks
the row `REVERTED`. It mirrors Ventrata and Bokun, with one precondition: **the original push
must have captured the old unit prices**. Rows written before that capture existed cannot be
reverted automatically.

---

## Operating notes

- No circuit breaker, no dry-run mode, no vendor-specific env flags. If Prioticket degrades,
  it degrades inside the batch alongside everything else — there is no single-vendor kill
  switch equivalent to `BOKUN_AUTO_APPLY_DISABLED`.
- Push-origin behaviour is covered by `prioticket.service.push-origin.spec.ts`.
- Onboarding is manual, coordinated with Prioticket directly.
