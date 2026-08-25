# Market Intelligence access flag

## Summary

A dedicated user flag **`marketIntelligenceAccess`** was added so that Market Intelligence (MI) access can be granted independently of the **Viator Partner** flag.

- **Before:** Only users with `viatorPartner: true` could access Market Intelligence. Impersonating a non–Viator-partner user (e.g. productoes@ecotuktuk.com) blocked MI even when that client had premium + MI.
- **After:** MI access is driven by **`marketIntelligenceAccess`**. Viator Partner can stay for Viator-specific behavior (e.g. support email, product list bypass); the frontend should use **`marketIntelligenceAccess`** to gate MI.

---

## Backend changes

### 1. Schema (Prisma)

- **Model:** `User`
- **New field:** `marketIntelligenceAccess Boolean? @default(false)`

**Migration:** Run after pulling:

```bash
npx prisma generate
npx prisma migrate dev --name add_market_intelligence_access
```

### 2. API

| Method | Endpoint | Description |
|--------|----------|-------------|
| **GET** | `/api/users/:id/market-intelligence-access` | Returns `{ marketIntelligenceAccess: boolean }` |
| **PATCH** | `/api/users/:id/market-intelligence-access` | Body: `{ "marketIntelligenceAccess": true }` — sets MI access for that user |

Same auth/authorization as other user admin endpoints (e.g. viator-partner).

### 3. User flags map

**GET** `/api/users/flags` (or equivalent that returns `getAllUserFlags()`) now includes **`marketIntelligenceAccess`** per user:

```json
{
  "<userId>": {
    "onDemand": false,
    "viatorPartner": false,
    "marketIntelligenceAccess": true,
    "directSell": false,
    "pricePush": true,
    "canAutoPricePush": false
  }
}
```

Use the **impersonated user’s** (or current user’s) `id` and read `marketIntelligenceAccess` to decide MI access.

---

## Frontend recommendations

1. **Gate Market Intelligence** on **`marketIntelligenceAccess`** (and optionally keep `viatorPartner` for other UI, e.g. “Viator partner” badge or support link).
2. **Impersonation:** When impersonating user B, use **B’s** `marketIntelligenceAccess` from the flags map (key = B’s user id), not the impersonating admin’s.
3. **Back office:** To give a client (e.g. productoes@ecotuktuk.com) access to MI without making them a Viator partner:
   - Resolve the user id (e.g. from users list or by email).
   - Call **PATCH** `/api/users/:id/market-intelligence-access` with `{ "marketIntelligenceAccess": true }`.

---

## Back office / Alegria

- For **premium clients with Market Intelligence** who are **not** Viator partners: set **`marketIntelligenceAccess`** to `true` for that user via the PATCH endpoint above (or a back-office UI that calls it).
- **Viator Partner** can remain `true` for actual Viator partners (MI + Viator-specific behavior); for non-Viator clients with MI, only **`marketIntelligenceAccess`** needs to be `true`.
