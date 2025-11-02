# Price Change History

## Overview

Every price change pushed to Ventrata is tracked in a comprehensive audit trail. This allows you to:

- 📊 See who changed what and when
- 🔍 Analyze pricing trends over time
- 📈 Track performance of price adjustments
- ⏮️ Revert changes if needed
- 📑 Generate compliance reports

## What Gets Tracked?

### Every Change Includes

| Field | Description | Example |
|-------|-------------|---------|
| **Who** | User who made the change | john.doe@company.com |
| **What** | Products, options, units affected | City Tour - Adult pricing |
| **When** | Exact timestamp | 2025-11-01 10:30:00 UTC |
| **Where** | Product/Option/Availability ID | prod-123, opt-456, avail-789 |
| **Before** | Old prices for all units | €50.00 |
| **After** | New prices for all units | €55.00 |
| **Why** | Optional change reason | "Seasonal price increase" |
| **Status** | Success or failure | Successfully pushed to Ventrata |

### Change Record Structure

```json
{
  "id": "change-uuid-123",
  "subscriptionId": "sub-456",
  "userId": "user-789",
  "userEmail": "john.doe@company.com",
  "productId": "prod-city-tour",
  "productName": "City Walking Tour",
  "optionId": "opt-morning",
  "optionName": "Morning Departure",
  "availabilityId": "avail-2025-12-25",
  "localDateTimeStart": "2025-12-25T10:00:00",
  "units": [
    {
      "id": "unit-change-001",
      "unitType": "ADULT",
      "oldRetail": 5000,
      "oldCurrency": "EUR",
      "newRetail": 5500,
      "newCurrency": "EUR",
      "changeAmount": 500,
      "changePercentage": 10.0
    },
    {
      "id": "unit-change-002",
      "unitType": "CHILD",
      "oldRetail": 2500,
      "oldCurrency": "EUR",
      "newRetail": 2750,
      "newCurrency": "EUR",
      "changeAmount": 250,
      "changePercentage": 10.0
    }
  ],
  "reason": "Seasonal price increase for Christmas period",
  "pushedToVentrata": true,
  "pushedAt": "2025-11-01T10:30:15Z",
  "ventrataResponse": {
    "success": true,
    "message": "Prices updated successfully"
  },
  "createdAt": "2025-11-01T10:30:00Z",
  "updatedAt": "2025-11-01T10:30:15Z"
}
```

## Viewing Change History

### Get All Changes

**Endpoint:** `GET /api/ventrata/price-changes`

```bash
curl -X GET "https://api.walkway.com/api/ventrata/price-changes?page=1&limit=50" \
  -H "Authorization: Bearer TOKEN"
```

**Response:**
```json
{
  "data": [
    {
      "id": "change-123",
      "userEmail": "john@company.com",
      "productName": "City Tour",
      "units": [...],
      "createdAt": "2025-11-01T10:30:00Z"
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 50,
    "total": 245,
    "totalPages": 5
  }
}
```

### Filter by Date Range

Get changes within specific period:

```bash
GET /api/ventrata/price-changes?\
startDate=2025-11-01&\
endDate=2025-11-30
```

### Filter by User

See changes made by specific user:

```bash
GET /api/ventrata/price-changes?userId=user-123
```

### Filter by Product

Get all changes for a product:

```bash
GET /api/ventrata/price-changes?productId=prod-456
```

### Filter by Unit Type

Track changes to specific unit types:

```bash
GET /api/ventrata/price-changes?unitType=ADULT
```

### Combined Filters

```bash
GET /api/ventrata/price-changes?\
productId=prod-city-tour&\
unitType=ADULT&\
startDate=2025-11-01&\
endDate=2025-11-30&\
userId=user-123
```

## Analyzing Change History

### Price Trends

Track how prices have changed over time:

```javascript
// Get price trend for a product
async function getPriceTrend(productId, unitType, startDate, endDate) {
  const changes = await fetch(
    `/api/ventrata/price-changes?` +
    `productId=${productId}&` +
    `unitType=${unitType}&` +
    `startDate=${startDate}&` +
    `endDate=${endDate}`
  );
  
  return changes.data.map(change => ({
    date: change.createdAt,
    price: change.units.find(u => u.unitType === unitType).newRetail,
    changedBy: change.userEmail
  }));
}

// Example result:
[
  { date: "2025-11-01", price: 5000, changedBy: "john@company.com" },
  { date: "2025-11-15", price: 5500, changedBy: "jane@company.com" },
  { date: "2025-12-01", price: 6000, changedBy: "john@company.com" }
]
```

### Average Price Changes

Calculate average price increase/decrease:

```javascript
function calculateAveragePriceChange(changes) {
  const totalChange = changes.reduce((sum, change) => {
    const unit = change.units[0]; // Assuming single unit for simplicity
    return sum + unit.changePercentage;
  }, 0);
  
  return totalChange / changes.length;
}

// Result: Average 8.5% price increase
```

### Most Active Users

See who makes the most pricing changes:

```javascript
function getMostActiveUsers(changes) {
  const userCounts = {};
  
  changes.forEach(change => {
    userCounts[change.userEmail] = (userCounts[change.userEmail] || 0) + 1;
  });
  
  return Object.entries(userCounts)
    .sort(([, a], [, b]) => b - a)
    .map(([email, count]) => ({ email, count }));
}

// Result:
[
  { email: "john@company.com", count: 45 },
  { email: "jane@company.com", count: 32 },
  { email: "bob@company.com", count: 18 }
]
```

## Export Change History

### Export to CSV

**Endpoint:** `GET /api/ventrata/price-changes/export`

```bash
curl -X GET "https://api.walkway.com/api/ventrata/price-changes/export?\
format=csv&\
startDate=2025-01-01&\
endDate=2025-12-31" \
  -H "Authorization: Bearer TOKEN" \
  --output price-changes-2025.csv
```

**CSV Format:**
```csv
Date,Time,User,Product,Option,Unit Type,Old Price,New Price,Change %,Reason,Status
2025-11-01,10:30:00,john@company.com,City Tour,Morning,ADULT,€50.00,€55.00,+10%,Seasonal increase,Success
2025-11-01,14:15:00,jane@company.com,Beach Tour,Afternoon,ADULT,€40.00,€45.00,+12.5%,High demand,Success
```

### Export to JSON

```bash
curl -X GET "https://api.walkway.com/api/ventrata/price-changes/export?\
format=json&\
startDate=2025-01-01&\
endDate=2025-12-31" \
  -H "Authorization: Bearer TOKEN" \
  --output price-changes-2025.json
```

### Export to PDF Report

Generate formatted PDF report:

```bash
curl -X POST "https://api.walkway.com/api/ventrata/price-changes/report" \
  -H "Authorization: Bearer TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "startDate": "2025-01-01",
    "endDate": "2025-12-31",
    "format": "pdf",
    "includeCharts": true,
    "includeSummary": true
  }' \
  --output annual-pricing-report-2025.pdf
```

## Change Statistics

### Get Summary Statistics

**Endpoint:** `GET /api/ventrata/price-changes/stats`

```bash
GET /api/ventrata/price-changes/stats?startDate=2025-11-01&endDate=2025-11-30
```

**Response:**
```json
{
  "period": {
    "start": "2025-11-01",
    "end": "2025-11-30"
  },
  "totals": {
    "totalChanges": 156,
    "successfulChanges": 154,
    "failedChanges": 2,
    "successRate": 98.7
  },
  "priceChanges": {
    "averageIncrease": 8.5,
    "averageDecrease": -5.2,
    "largestIncrease": 25.0,
    "largestDecrease": -15.0
  },
  "byUnitType": {
    "ADULT": {
      "changes": 78,
      "averageChange": 9.2
    },
    "CHILD": {
      "changes": 78,
      "averageChange": 7.8
    }
  },
  "byUser": [
    {
      "userEmail": "john@company.com",
      "changes": 89,
      "percentage": 57.1
    },
    {
      "userEmail": "jane@company.com",
      "changes": 67,
      "percentage": 42.9
    }
  ],
  "topProducts": [
    {
      "productId": "prod-city-tour",
      "productName": "City Walking Tour",
      "changes": 45
    },
    {
      "productId": "prod-beach-tour",
      "productName": "Beach Adventure",
      "changes": 32
    }
  ]
}
```

## Real-Time Change Tracking

### Websocket Connection

Subscribe to real-time price changes:

```javascript
const ws = new WebSocket('wss://api.walkway.com/ws/price-changes');

ws.onopen = () => {
  ws.send(JSON.stringify({
    action: 'subscribe',
    subscriptionId: 'sub-123',
    token: 'your-jwt-token'
  }));
};

ws.onmessage = (event) => {
  const change = JSON.parse(event.data);
  console.log('New price change:', change);
  
  // Update UI in real-time
  updatePricingDashboard(change);
};
```

### Change Notifications

Get notified of price changes:

```json
{
  "type": "price_change",
  "changeId": "change-123",
  "productName": "City Tour",
  "userEmail": "john@company.com",
  "summary": "Adult price increased from €50 to €55 (+10%)",
  "timestamp": "2025-11-01T10:30:00Z"
}
```

## Compliance & Auditing

### Generate Audit Report

For compliance purposes:

```bash
POST /api/ventrata/price-changes/audit-report
```

```json
{
  "startDate": "2025-01-01",
  "endDate": "2025-12-31",
  "includeFailedChanges": true,
  "includeUserDetails": true,
  "reportFormat": "pdf",
  "certifiedReport": true
}
```

**Report Includes:**
- ✅ All price changes with timestamps
- ✅ User authentication logs
- ✅ Failed attempts and reasons
- ✅ System events and errors
- ✅ Digital signature for authenticity

### Retention Policy

Change history is retained according to your plan:

| Plan | Retention Period |
|------|------------------|
| Starter | 90 days |
| Professional | 1 year |
| Enterprise | Unlimited |

!!! info "Archive Old Data"
    Export historical data periodically to keep your own archive beyond retention limits.

## Comparing Changes

### Compare Two Dates

See how prices have changed between two dates:

```bash
POST /api/ventrata/price-changes/compare
```

```json
{
  "productId": "prod-city-tour",
  "date1": "2025-01-01",
  "date2": "2025-12-31"
}
```

**Response:**
```json
{
  "product": "City Walking Tour",
  "comparison": [
    {
      "unitType": "ADULT",
      "priceOn2025-01-01": 4500,
      "priceOn2025-12-31": 5500,
      "changeAmount": 1000,
      "changePercentage": 22.2,
      "numberOfChanges": 5
    },
    {
      "unitType": "CHILD",
      "priceOn2025-01-01": 2250,
      "priceOn2025-12-31": 2750,
      "changeAmount": 500,
      "changePercentage": 22.2,
      "numberOfChanges": 5
    }
  ]
}
```

## Best Practices

### 1. Document Change Reasons

Always include a reason for price changes:

```javascript
await pushPriceChange({
  productId: "city-tour",
  units: [...],
  reason: "Competitor analysis: matching TourCompany X pricing"
});
```

### 2. Review History Regularly

Set up weekly reviews:
- Who is making changes?
- Are changes justified?
- Are trends positive or negative?

### 3. Monitor Failed Changes

Failed changes might indicate:
- API key issues
- Network problems
- Invalid data
- Permission issues

```bash
GET /api/ventrata/price-changes?pushedToVentrata=false
```

### 4. Set Up Alerts

Create alerts for unusual activity:

```javascript
// Alert if price increase exceeds 20%
if (changePercentage > 20) {
  sendAlert({
    type: 'large_price_increase',
    changeId: change.id,
    percentage: changePercentage
  });
}
```

## Next Steps

- [Undo Changes →](undo.md) - Learn how to revert price changes
- [API Endpoints →](api-endpoints.md) - Full API reference
- [Troubleshooting →](troubleshooting.md) - Common issues

---

**Questions about change history?** Contact support at support@walkway.com

