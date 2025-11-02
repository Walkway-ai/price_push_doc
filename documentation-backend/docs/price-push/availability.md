# Availability Management

## What is Availability?

Availability represents specific dates, times, or slots when your tours are offered. Each availability instance can have its own pricing, allowing you to implement dynamic pricing strategies.

## Availability Structure

### Basic Availability Object

```json
{
  "id": "avail-2025-12-25-10:00",
  "productId": "city-tour",
  "optionId": "morning-slot",
  "localDateTimeStart": "2025-12-25T10:00:00",
  "localDateTimeEnd": "2025-12-25T14:00:00",
  "allDay": false,
  "available": true,
  "status": "AVAILABLE",
  "vacancies": 20,
  "capacity": 20,
  "units": [
    {
      "unitType": "ADULT",
      "retail": 5500,
      "currency": "EUR"
    }
  ]
}
```

**Key Fields:**
- `id`: Unique identifier for this availability
- `localDateTimeStart`: When the tour starts (local timezone)
- `localDateTimeEnd`: When the tour ends (local timezone)
- `available`: Whether bookings are accepted
- `vacancies`: Available spots remaining
- `capacity`: Total capacity
- `units`: Pricing for each unit type

## Getting Availability

### Fetch Available Dates

**Endpoint:** `GET /api/ventrata/availability`

**Query Parameters:**
- `productId` (required): Product UUID
- `optionId` (optional): Specific option
- `localDateStart` (required): Start date (YYYY-MM-DD)
- `localDateEnd` (required): End date (YYYY-MM-DD)

**Example Request:**
```bash
curl -X GET "https://api.walkway.com/api/ventrata/availability?\
productId=city-tour&\
optionId=morning-slot&\
localDateStart=2025-12-01&\
localDateEnd=2025-12-31" \
  -H "Authorization: Bearer TOKEN"
```

**Example Response:**
```json
{
  "product": {
    "id": "city-tour",
    "internalName": "City Walking Tour"
  },
  "availability": [
    {
      "id": "avail-2025-12-25-10:00",
      "localDateTimeStart": "2025-12-25T10:00:00",
      "localDateTimeEnd": "2025-12-25T14:00:00",
      "status": "AVAILABLE",
      "vacancies": 20,
      "capacity": 20,
      "units": [
        {
          "unitType": "ADULT",
          "retail": 5500,
          "currency": "EUR"
        },
        {
          "unitType": "CHILD",
          "retail": 2750,
          "currency": "EUR"
        }
      ]
    },
    {
      "id": "avail-2025-12-26-10:00",
      "localDateTimeStart": "2025-12-26T10:00:00",
      "localDateTimeEnd": "2025-12-26T14:00:00",
      "status": "AVAILABLE",
      "vacancies": 15,
      "capacity": 20,
      "units": [
        {
          "unitType": "ADULT",
          "retail": 5500,
          "currency": "EUR"
        }
      ]
    }
  ]
}
```

## Availability Status

### Status Types

| Status | Description | Bookable | Use Case |
|--------|-------------|----------|----------|
| `AVAILABLE` | Open for bookings | ✅ Yes | Normal availability |
| `SOLD_OUT` | All spots taken | ❌ No | Fully booked |
| `LIMITED` | Few spots left | ✅ Yes | Low availability warning |
| `UNAVAILABLE` | Not bookable | ❌ No | Closed day, maintenance |
| `ON_REQUEST` | Requires confirmation | ⚠️ Maybe | Special bookings only |

### Status Logic

```javascript
function getAvailabilityStatus(vacancies, capacity) {
  if (vacancies === 0) return 'SOLD_OUT';
  if (vacancies <= capacity * 0.2) return 'LIMITED'; // 20% or less
  return 'AVAILABLE';
}
```

## Dynamic Pricing with Availability

### Price by Demand

Adjust prices based on availability:

```javascript
function calculateDynamicPrice(basePrice, vacancies, capacity) {
  const fillRate = 1 - (vacancies / capacity);
  
  if (fillRate >= 0.8) {
    // 80%+ full: increase price by 20%
    return Math.round(basePrice * 1.20);
  } else if (fillRate >= 0.5) {
    // 50-80% full: increase price by 10%
    return Math.round(basePrice * 1.10);
  } else {
    // Less than 50% full: normal price
    return basePrice;
  }
}
```

**Example:**
```javascript
const basePrice = 5000; // €50
const vacancies = 5;
const capacity = 20;

const dynamicPrice = calculateDynamicPrice(basePrice, vacancies, capacity);
// fillRate = 0.75 (75% full)
// Result: €55.00 (10% increase)
```

### Price by Date

Adjust prices based on date:

```javascript
function getPriceByDate(basePrice, date) {
  const dayOfWeek = date.getDay();
  const isWeekend = dayOfWeek === 0 || dayOfWeek === 6;
  
  if (isWeekend) {
    return Math.round(basePrice * 1.25); // 25% weekend surcharge
  }
  
  return basePrice;
}
```

### Price by Time

Different prices for different times of day:

```json
{
  "availabilities": [
    {
      "id": "morning-slot",
      "localDateTimeStart": "2025-12-25T09:00:00",
      "units": [
        {
          "unitType": "ADULT",
          "retail": 4500,
          "currency": "EUR"
        }
      ]
    },
    {
      "id": "afternoon-slot",
      "localDateTimeStart": "2025-12-25T14:00:00",
      "units": [
        {
          "unitType": "ADULT",
          "retail": 5000,
          "currency": "EUR"
        }
      ]
    },
    {
      "id": "sunset-slot",
      "localDateTimeStart": "2025-12-25T18:00:00",
      "units": [
        {
          "unitType": "ADULT",
          "retail": 6500,
          "currency": "EUR"
        }
      ]
    }
  ]
}
```

**Result:**
- Morning (9am): €45 (cheaper, less popular)
- Afternoon (2pm): €50 (standard price)
- Sunset (6pm): €65 (premium, most popular)

## Updating Availability Pricing

### Update Single Availability

**Endpoint:** `PATCH /api/ventrata/availability/{id}/pricing`

```bash
curl -X PATCH https://api.walkway.com/api/ventrata/availability/avail-123/pricing \
  -H "Authorization: Bearer TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
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
      }
    ]
  }'
```

**Response:**
```json
{
  "availabilityId": "avail-123",
  "priceChangeId": "change-456",
  "updatedUnits": 2,
  "pushedToVentrata": true,
  "pushedAt": "2025-11-01T10:30:00Z"
}
```

### Bulk Update Multiple Availabilities

Update pricing for multiple dates at once:

**Endpoint:** `POST /api/ventrata/availability/bulk-pricing`

```json
{
  "productId": "city-tour",
  "optionId": "morning-slot",
  "dateRange": {
    "start": "2025-12-01",
    "end": "2025-12-31"
  },
  "units": [
    {
      "unitType": "ADULT",
      "newRetail": 5500,
      "currency": "EUR"
    }
  ]
}
```

**Response:**
```json
{
  "updatedCount": 31,
  "priceChangeIds": ["change-1", "change-2", "..."],
  "pushedToVentrata": true
}
```

## Availability Filters

### Filter by Status

Get only available slots:

```bash
GET /api/ventrata/availability?status=AVAILABLE&localDateStart=2025-12-01
```

### Filter by Vacancies

Get slots with at least 10 vacancies:

```bash
GET /api/ventrata/availability?minVacancies=10&localDateStart=2025-12-01
```

### Filter by Price Range

Get availabilities within price range:

```bash
GET /api/ventrata/availability?minPrice=4000&maxPrice=6000&localDateStart=2025-12-01
```

## Real-Time Availability

### Check Current Availability

Get real-time availability for today:

```bash
GET /api/ventrata/availability/real-time?productId=city-tour
```

**Response:**
```json
{
  "productId": "city-tour",
  "asOf": "2025-11-01T10:30:00Z",
  "todayAvailability": [
    {
      "id": "avail-today-14:00",
      "localDateTimeStart": "2025-11-01T14:00:00",
      "status": "AVAILABLE",
      "vacancies": 12,
      "capacity": 20
    }
  ]
}
```

### Webhook Notifications

Subscribe to availability changes:

```json
{
  "event": "availability.sold_out",
  "availabilityId": "avail-123",
  "productId": "city-tour",
  "localDateTimeStart": "2025-12-25T10:00:00",
  "previousVacancies": 1,
  "currentVacancies": 0
}
```

## Availability Strategies

### Strategy 1: Early Bird Pricing

Lower prices for bookings far in advance:

```javascript
function getEarlyBirdPrice(basePrice, daysUntilTour) {
  if (daysUntilTour >= 60) {
    return Math.round(basePrice * 0.80); // 20% off for 60+ days
  } else if (daysUntilTour >= 30) {
    return Math.round(basePrice * 0.90); // 10% off for 30+ days
  }
  return basePrice;
}
```

### Strategy 2: Last-Minute Deals

Discount unsold inventory:

```javascript
function getLastMinutePrice(basePrice, daysUntilTour, vacancies, capacity) {
  const fillRate = 1 - (vacancies / capacity);
  
  if (daysUntilTour <= 3 && fillRate < 0.5) {
    return Math.round(basePrice * 0.75); // 25% off if less than 50% full
  }
  
  return basePrice;
}
```

### Strategy 3: Peak Season Pricing

Higher prices during popular periods:

```javascript
const PEAK_SEASONS = [
  { start: '2025-06-15', end: '2025-08-31' }, // Summer
  { start: '2025-12-20', end: '2026-01-05' }  // Christmas/New Year
];

function isPeakSeason(date) {
  return PEAK_SEASONS.some(season => {
    return date >= new Date(season.start) && date <= new Date(season.end);
  });
}

function getPeakSeasonPrice(basePrice, date) {
  if (isPeakSeason(date)) {
    return Math.round(basePrice * 1.30); // 30% increase
  }
  return basePrice;
}
```

## Availability Calendar View

### Monthly Overview

```javascript
// Get all availability for December 2025
const december2025 = await fetch(
  '/api/ventrata/availability/calendar?year=2025&month=12&productId=city-tour'
);
```

**Response Format:**
```json
{
  "year": 2025,
  "month": 12,
  "days": [
    {
      "date": "2025-12-01",
      "availabilities": 2,
      "totalVacancies": 40,
      "averagePrice": 5500,
      "status": "AVAILABLE"
    },
    {
      "date": "2025-12-25",
      "availabilities": 1,
      "totalVacancies": 0,
      "averagePrice": 6000,
      "status": "SOLD_OUT"
    }
  ]
}
```

## Performance Tips

### Caching

Cache availability data to reduce API calls:

```javascript
const CACHE_DURATION = 5 * 60 * 1000; // 5 minutes

let availabilityCache = {
  data: null,
  timestamp: null
};

async function getAvailability(productId, dateRange) {
  const now = Date.now();
  
  if (availabilityCache.data && 
      (now - availabilityCache.timestamp) < CACHE_DURATION) {
    return availabilityCache.data;
  }
  
  const data = await fetchAvailability(productId, dateRange);
  availabilityCache = { data, timestamp: now };
  
  return data;
}
```

### Pagination

For large date ranges, use pagination:

```bash
GET /api/ventrata/availability?page=1&limit=50&localDateStart=2025-01-01&localDateEnd=2025-12-31
```

## Next Steps

- [Price Change History →](history.md) - Track availability pricing changes
- [Price Unit Management →](price-units.md) - Manage unit types
- [Undo Changes →](undo.md) - Revert availability price changes

---

**Need help with availability?** Check the [Troubleshooting Guide](troubleshooting.md).

