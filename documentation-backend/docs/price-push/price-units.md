# Price Unit Management

## What are Price Units?

Price Units represent different types of customers who book your tours. Each unit type can have its own price, allowing you to offer different rates for adults, children, seniors, and more.

## Supported Unit Types

### Standard Unit Types

| Unit Type | Description | Typical Age Range | Common Use |
|-----------|-------------|-------------------|------------|
| **ADULT** | Standard adult pricing | 18-64 years | Base price, most common |
| **CHILD** | Discounted child pricing | 3-17 years | Family-friendly tours |
| **SENIOR** | Senior citizen discount | 65+ years | Age-based discount |
| **YOUTH** | Young adult pricing | 13-17 years | Teen-specific pricing |
| **INFANT** | Free or minimal charge | 0-2 years | Often free or low cost |

### How Units Work

Each unit type can have:
- **Different retail prices** - Flexible pricing per customer type
- **Different currencies** - Multi-currency support
- **Individual tracking** - Separate change history per unit
- **Custom rules** - Define relationships between units

## Price Unit Structure

### Basic Unit Object

```json
{
  "unitId": "unit-uuid-123",
  "unitType": "ADULT",
  "oldRetail": 5000,
  "oldCurrency": "EUR",
  "newRetail": 5500,
  "newCurrency": "EUR"
}
```

**Fields Explained:**
- `unitId`: Unique identifier for this pricing unit
- `unitType`: One of: ADULT, CHILD, SENIOR, YOUTH, INFANT
- `oldRetail`: Previous price in cents (5000 = €50.00)
- `oldCurrency`: Currency code (EUR, USD, GBP, etc.)
- `newRetail`: New price in cents (5500 = €55.00)
- `newCurrency`: Currency code

!!! info "Price Format"
    All prices are stored in **cents** (or smallest currency unit) to avoid floating-point errors.  
    Example: €50.00 = 5000 cents

## Managing Multiple Units

### Example: Family Tour Pricing

```json
{
  "productId": "city-tour-full-day",
  "optionId": "morning-departure",
  "availabilityId": "2025-12-25",
  "units": [
    {
      "unitType": "ADULT",
      "oldRetail": 5000,
      "newRetail": 5500,
      "currency": "EUR"
    },
    {
      "unitType": "CHILD",
      "oldRetail": 2500,
      "newRetail": 2750,
      "currency": "EUR"
    },
    {
      "unitType": "SENIOR",
      "oldRetail": 4000,
      "newRetail": 4400,
      "currency": "EUR"
    },
    {
      "unitType": "INFANT",
      "oldRetail": 0,
      "newRetail": 0,
      "currency": "EUR"
    }
  ]
}
```

**Result:**
- Adult: €50.00 → €55.00
- Child: €25.00 → €27.50
- Senior: €40.00 → €44.00
- Infant: Free (€0.00)

## Pricing Rules

### Automatic Unit Pricing

You can set up rules to automatically calculate unit prices based on relationships:

#### Rule Types

1. **Fixed Difference**
   ```json
   {
     "baseUnitType": "ADULT",
     "targetUnitType": "CHILD",
     "ruleType": "FIXED_DIFFERENCE",
     "fixedDifference": -1000  // €10 less than adult
   }
   ```
   **Result:** If Adult = €50, Child = €40 automatically

2. **Percentage**
   ```json
   {
     "baseUnitType": "ADULT",
     "targetUnitType": "CHILD",
     "ruleType": "PERCENTAGE",
     "percentage": 0.5  // 50% of adult price
   }
   ```
   **Result:** If Adult = €50, Child = €25 automatically

3. **Fixed Price**
   ```json
   {
     "baseUnitType": "ADULT",
     "targetUnitType": "INFANT",
     "ruleType": "FIXED_PRICE",
     "fixedPrice": 0  // Always free
   }
   ```
   **Result:** Infant is always €0 regardless of adult price

4. **Free**
   ```json
   {
     "baseUnitType": "ADULT",
     "targetUnitType": "INFANT",
     "ruleType": "FREE"
   }
   ```
   **Result:** Unit is free (shorthand for fixed price = 0)

### Creating Pricing Rules

**API Endpoint:** `POST /api/ventrata/pricing-rules`

```bash
curl -X POST https://api.walkway.com/api/ventrata/pricing-rules \
  -H "Authorization: Bearer TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "subscriptionId": "sub-123",
    "productId": "prod-456",
    "optionId": "opt-789",
    "baseUnitType": "ADULT",
    "targetUnitType": "CHILD",
    "ruleType": "FIXED_DIFFERENCE",
    "fixedDifference": -1000
  }'
```

**Response:**
```json
{
  "id": "rule-abc",
  "subscriptionId": "sub-123",
  "productId": "prod-456",
  "optionId": "opt-789",
  "baseUnitType": "ADULT",
  "targetUnitType": "CHILD",
  "ruleType": "FIXED_DIFFERENCE",
  "fixedDifference": -1000,
  "isActive": true,
  "createdAt": "2025-11-01T10:30:00Z"
}
```

### Rule Priority

When multiple rules exist:

1. **Option-specific rules** (highest priority)
2. **Product-level rules** (applies to all options)
3. **Manual prices** (no rules applied)

## Updating Unit Prices

### Single Unit Update

```bash
POST /api/ventrata/price-changes
```

```json
{
  "subscriptionId": "sub-123",
  "productId": "prod-456",
  "optionId": "opt-789",
  "availabilityId": "2025-12-25",
  "units": [
    {
      "unitId": "unit-adult",
      "unitType": "ADULT",
      "oldRetail": 5000,
      "newRetail": 5500,
      "currency": "EUR"
    }
  ]
}
```

### Bulk Unit Update

Update all units for a product at once:

```json
{
  "subscriptionId": "sub-123",
  "productId": "prod-456",
  "optionId": "opt-789",
  "availabilityId": "2025-12-25",
  "units": [
    {
      "unitType": "ADULT",
      "newRetail": 5500,
      "currency": "EUR"
    },
    {
      "unitType": "CHILD",
      "newRetail": 2750,
      "currency": "EUR"
    },
    {
      "unitType": "SENIOR",
      "newRetail": 4400,
      "currency": "EUR"
    },
    {
      "unitType": "INFANT",
      "newRetail": 0,
      "currency": "EUR"
    }
  ]
}
```

## Unit Pricing Best Practices

### 1. Consistent Ratios

Keep consistent pricing ratios across your products:

```
ADULT: 100% (base)
CHILD: 50% of adult
SENIOR: 80% of adult
INFANT: Free
```

### 2. Use Pricing Rules

Automate unit pricing to:
- ✅ Maintain consistent relationships
- ✅ Save time on bulk updates
- ✅ Reduce human error
- ✅ Ensure pricing logic is applied everywhere

### 3. Validate Before Pushing

Always check unit prices before pushing:

```javascript
// Example validation
function validateUnitPrices(units) {
  const adult = units.find(u => u.unitType === 'ADULT');
  const child = units.find(u => u.unitType === 'CHILD');
  
  // Child should be less than adult
  if (child && child.newRetail >= adult.newRetail) {
    console.warn('Warning: Child price >= Adult price');
  }
  
  // Infant should be free or very low
  const infant = units.find(u => u.unitType === 'INFANT');
  if (infant && infant.newRetail > 500) {
    console.warn('Warning: Infant price seems high');
  }
}
```

### 4. Test in Staging

Always test unit price changes in a staging environment first.

## Currency Handling

### Multi-Currency Support

Each unit can have its own currency:

```json
{
  "units": [
    {
      "unitType": "ADULT",
      "newRetail": 5500,
      "currency": "EUR"
    },
    {
      "unitType": "ADULT",
      "newRetail": 6000,
      "currency": "USD"
    },
    {
      "unitType": "ADULT",
      "newRetail": 5000,
      "currency": "GBP"
    }
  ]
}
```

### Currency Conversion

When updating prices across currencies:

1. Convert to cents/smallest unit
2. Round appropriately
3. Validate conversion rates
4. Document exchange rate used

```javascript
// Example: EUR to USD conversion
const eurPrice = 5000; // €50.00
const exchangeRate = 1.10;
const usdPrice = Math.round(eurPrice * exchangeRate); // 5500 = $55.00
```

## Unit Tracking

Every unit price change is tracked individually:

```json
{
  "priceChangeId": "change-123",
  "units": [
    {
      "id": "unit-change-456",
      "unitType": "ADULT",
      "oldRetail": 5000,
      "newRetail": 5500,
      "oldCurrency": "EUR",
      "newCurrency": "EUR",
      "changePercentage": 10.0
    },
    {
      "id": "unit-change-789",
      "unitType": "CHILD",
      "oldRetail": 2500,
      "newRetail": 2750,
      "oldCurrency": "EUR",
      "newCurrency": "EUR",
      "changePercentage": 10.0
    }
  ]
}
```

## Common Unit Scenarios

### Scenario 1: Seasonal Pricing

Update all units for high season:

```javascript
const SUMMER_MARKUP = 1.20; // 20% increase

units.forEach(unit => {
  unit.newRetail = Math.round(unit.oldRetail * SUMMER_MARKUP);
});
```

### Scenario 2: Special Promotion

Free children for weekend tours:

```javascript
if (isWeekend && unit.unitType === 'CHILD') {
  unit.newRetail = 0;
}
```

### Scenario 3: Group Discounts

Larger groups get better per-person rates:

```javascript
if (groupSize >= 10) {
  units.forEach(unit => {
    unit.newRetail = Math.round(unit.oldRetail * 0.90); // 10% off
  });
}
```

## API Reference

### Get Unit Pricing Rules

```bash
GET /api/ventrata/pricing-rules?subscriptionId={id}&productId={id}
```

### Create Pricing Rule

```bash
POST /api/ventrata/pricing-rules
```

### Update Pricing Rule

```bash
PATCH /api/ventrata/pricing-rules/{id}
```

### Delete Pricing Rule

```bash
DELETE /api/ventrata/pricing-rules/{id}
```

### Get Unit Price History

```bash
GET /api/ventrata/price-changes?unitType=ADULT&productId={id}
```

## Next Steps

- [Availability Management →](availability.md) - Manage when prices apply
- [Price Change History →](history.md) - Track all unit price changes
- [Undo Changes →](undo.md) - Revert unit price changes

---

**Questions?** Check the [Troubleshooting Guide](troubleshooting.md) or contact support.

