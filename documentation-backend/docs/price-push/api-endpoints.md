# Price Push API Endpoints

## Overview

This page documents the API endpoints related to Price Push functionality. These endpoints allow you to programmatically manage Price Push settings and propagate changes to team members.

## Authentication

All API requests require a valid JWT token in the Authorization header:

```bash
Authorization: Bearer YOUR_JWT_TOKEN
```

## Base URL

```
https://api.walkway.com
```

---

## Subscription Endpoints

### Enable/Disable Price Push

Enable or disable Price Push for a subscription. This automatically propagates to all Owner and Admin members.

**Endpoint:** `PATCH /api/subscriptions/:id/price-push`

**Parameters:**
- `id` (path): Subscription UUID

**Request Body:**
```json
{
  "pricePush": true
}
```

**Response:**
```json
{
  "id": "sub-uuid-123",
  "pricePush": true,
  "updatedUsersCount": 3,
  "eligibleRoles": ["OWNER", "ADMIN"],
  "message": "Price push enabled successfully. Propagated to 3 eligible members (OWNER/ADMIN)."
}
```

**cURL Example:**
```bash
curl -X PATCH https://api.walkway.com/api/subscriptions/sub-123/price-push \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"pricePush": true}'
```

**Role Required:** Owner or Admin with `canEditSubscription` permission

---

### Update Ventrata API Key

Set or update the Ventrata API key for a subscription. The key is automatically propagated to all members.

**Endpoint:** `PATCH /api/subscriptions/:id/ventrata-api-key`

**Parameters:**
- `id` (path): Subscription UUID

**Request Body:**
```json
{
  "ventrataApiKey": "your-ventrata-api-key"
}
```

**Response:**
```json
{
  "id": "sub-uuid-123",
  "ventrataApiKey": "your-ventrata-api-key",
  "ventrataConfiguredAt": "2025-11-01T10:30:00Z",
  "updatedUsersCount": 5,
  "message": "API key propagated to all subscription members"
}
```

**cURL Example:**
```bash
curl -X PATCH https://api.walkway.com/api/subscriptions/sub-123/ventrata-api-key \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"ventrataApiKey": "your-key-here"}'
```

**Role Required:** Owner or Admin with `canEditSubscription` permission

---

### Get Ventrata API Key

Retrieve the Ventrata API key for a subscription.

**Endpoint:** `GET /api/subscriptions/:id/ventrata-api-key`

**Parameters:**
- `id` (path): Subscription UUID

**Response:**
```json
{
  "ventrataApiKey": "your-ventrata-api-key",
  "ventrataConfiguredAt": "2025-11-01T10:30:00Z"
}
```

**cURL Example:**
```bash
curl -X GET https://api.walkway.com/api/subscriptions/sub-123/ventrata-api-key \
  -H "Authorization: Bearer YOUR_TOKEN"
```

**Role Required:** Member with `canViewAnalytics` permission

---

### Update Subscription (with Price Push)

Update subscription details including Price Push settings. Changes are automatically propagated to members.

**Endpoint:** `PATCH /api/subscriptions/:id`

**Parameters:**
- `id` (path): Subscription UUID

**Request Body:**
```json
{
  "pricePush": true,
  "ventrataApiKey": "new-api-key",
  "markets": ["Paris", "London"],
  "channels": ["Viator", "GetYourGuide"]
}
```

**Response:**
```json
{
  "id": "sub-uuid-123",
  "pricePush": true,
  "ventrataApiKey": "new-api-key",
  "markets": ["Paris", "London"],
  "channels": ["Viator", "GetYourGuide"],
  "propagation": {
    "pricePushUpdated": 3,
    "ventrataApiKeyUpdated": 5
  }
}
```

**cURL Example:**
```bash
curl -X PATCH https://api.walkway.com/api/subscriptions/sub-123 \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "pricePush": true,
    "ventrataApiKey": "new-key"
  }'
```

**Role Required:** Owner or Admin with `canEditSubscription` permission

---

## User Endpoints

### Get All User Flags

Retrieve Price Push status and other feature flags for all users.

**Endpoint:** `GET /api/users/flags`

**Response:**
```json
{
  "user-id-1": {
    "onDemand": false,
    "viatorPartner": false,
    "directSell": false,
    "pricePush": true
  },
  "user-id-2": {
    "onDemand": true,
    "viatorPartner": false,
    "directSell": true,
    "pricePush": false
  }
}
```

**cURL Example:**
```bash
curl -X GET https://api.walkway.com/api/users/flags \
  -H "Authorization: Bearer YOUR_TOKEN"
```

**Use Case:** Check Price Push status for multiple users at once

---

### Get User Ventrata API Key

Get the Ventrata API key configured for a specific user.

**Endpoint:** `GET /api/users/:id/ventrata-api-key`

**Parameters:**
- `id` (path): User UUID

**Response:**
```json
{
  "ventrataApiKey": "user-api-key",
  "ventrataConfiguredAt": "2025-11-01T10:30:00Z"
}
```

**cURL Example:**
```bash
curl -X GET https://api.walkway.com/api/users/user-123/ventrata-api-key \
  -H "Authorization: Bearer YOUR_TOKEN"
```

---

### Update User Ventrata API Key

Set or update the Ventrata API key for a specific user.

**Endpoint:** `PATCH /api/users/:id/ventrata-api-key`

**Parameters:**
- `id` (path): User UUID

**Request Body:**
```json
{
  "ventrataApiKey": "new-api-key"
}
```

**Response:**
```json
{
  "id": "user-123",
  "ventrataApiKey": "new-api-key",
  "ventrataConfiguredAt": "2025-11-01T10:30:00Z"
}
```

**cURL Example:**
```bash
curl -X PATCH https://api.walkway.com/api/users/user-123/ventrata-api-key \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"ventrataApiKey": "new-key"}'
```

**Role Required:** Owner or Admin

---

## Admin Endpoints

### Update User Price Push Access

Manually enable or disable Price Push for a specific user (overrides automatic propagation).

**Endpoint:** `PATCH /api/admin/:id/price-push-access`

**Parameters:**
- `id` (path): User UUID

**Request Body:**
```json
{
  "pricePush": true
}
```

**Response:**
```json
{
  "id": "user-123",
  "email": "user@example.com",
  "pricePush": true,
  "message": "Price push access updated successfully"
}
```

**cURL Example:**
```bash
curl -X PATCH https://api.walkway.com/api/admin/user-123/price-push-access \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"pricePush": true}'
```

**Role Required:** Admin

!!! warning "Manual Override"
    This endpoint creates a manual override. Future automatic propagation from subscription changes will overwrite this setting.

---

## Member Management Endpoints

### Add Member to Subscription

Add a new member to a subscription. Price Push access is automatically configured based on role.

**Endpoint:** `POST /api/subscriptions/:subscriptionId/members`

**Parameters:**
- `subscriptionId` (path): Subscription UUID

**Request Body:**
```json
{
  "email": "newmember@example.com",
  "role": "ADMIN"
}
```

**Response:**
```json
{
  "id": "member-123",
  "userId": "user-456",
  "subscriptionId": "sub-789",
  "role": "ADMIN",
  "user": {
    "id": "user-456",
    "email": "newmember@example.com",
    "pricePush": true
  }
}
```

**Automatic Behavior:**
- If role is `OWNER` or `ADMIN`: `pricePush` inherits from subscription
- If role is `MEMBER` or `VIEWER`: `pricePush` = false

**cURL Example:**
```bash
curl -X POST https://api.walkway.com/api/subscriptions/sub-123/members \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "email": "newadmin@example.com",
    "role": "ADMIN"
  }'
```

---

### Update Member Role

Change a member's role. Price Push access is automatically adjusted.

**Endpoint:** `PATCH /api/subscriptions/:subscriptionId/members/:memberId`

**Parameters:**
- `subscriptionId` (path): Subscription UUID
- `memberId` (path): Member UUID

**Request Body:**
```json
{
  "role": "ADMIN"
}
```

**Response:**
```json
{
  "id": "member-123",
  "role": "ADMIN",
  "user": {
    "id": "user-456",
    "email": "user@example.com"
  }
}
```

**Automatic Behavior:**
- **Promotion** (MEMBER → ADMIN): Price Push automatically enabled
- **Demotion** (ADMIN → MEMBER): Price Push automatically disabled

**cURL Example:**
```bash
curl -X PATCH https://api.walkway.com/api/subscriptions/sub-123/members/member-456 \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"role": "ADMIN"}'
```

---

## Error Responses

### Common Error Codes

| Status Code | Error | Description |
|-------------|-------|-------------|
| 400 | Bad Request | Invalid pricePush value or missing required fields |
| 401 | Unauthorized | Invalid or missing JWT token |
| 403 | Forbidden | User doesn't have required permissions |
| 404 | Not Found | Subscription or user not found |
| 409 | Conflict | Resource already exists |
| 500 | Internal Server Error | Server error, check logs |

### Error Response Format

```json
{
  "statusCode": 400,
  "message": "pricePush must be a boolean value",
  "error": "Bad Request"
}
```

---

## Rate Limiting

API endpoints are rate-limited to prevent abuse:

- **Standard endpoints**: 100 requests per minute
- **Bulk operations**: 10 requests per minute
- **Admin endpoints**: 50 requests per minute

**Rate Limit Headers:**
```
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 95
X-RateLimit-Reset: 1635782400
```

---

## Webhooks (Coming Soon)

Webhook notifications for Price Push events:

- `price_push.enabled` - Price Push enabled for subscription
- `price_push.disabled` - Price Push disabled for subscription
- `price_push.member_updated` - Member Price Push access changed
- `ventrata.key_updated` - Ventrata API key updated

---

## Code Examples

### JavaScript/TypeScript

```typescript
// Enable Price Push for subscription
async function enablePricePush(subscriptionId: string, token: string) {
  const response = await fetch(
    `https://api.walkway.com/api/subscriptions/${subscriptionId}/price-push`,
    {
      method: 'PATCH',
      headers: {
        'Authorization': `Bearer ${token}`,
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({ pricePush: true }),
    }
  );
  
  if (!response.ok) {
    throw new Error(`HTTP error! status: ${response.status}`);
  }
  
  const data = await response.json();
  console.log(`Price Push enabled for ${data.updatedUsersCount} members`);
  return data;
}
```

### Python

```python
import requests

def enable_price_push(subscription_id, token):
    url = f"https://api.walkway.com/api/subscriptions/{subscription_id}/price-push"
    headers = {
        "Authorization": f"Bearer {token}",
        "Content-Type": "application/json"
    }
    data = {"pricePush": True}
    
    response = requests.patch(url, headers=headers, json=data)
    response.raise_for_status()
    
    result = response.json()
    print(f"Price Push enabled for {result['updatedUsersCount']} members")
    return result
```

### PHP

```php
<?php
function enablePricePush($subscriptionId, $token) {
    $url = "https://api.walkway.com/api/subscriptions/{$subscriptionId}/price-push";
    
    $ch = curl_init($url);
    curl_setopt($ch, CURLOPT_CUSTOMREQUEST, "PATCH");
    curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode(['pricePush' => true]));
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
    curl_setopt($ch, CURLOPT_HTTPHEADER, [
        "Authorization: Bearer {$token}",
        "Content-Type: application/json"
    ]);
    
    $result = curl_exec($ch);
    curl_close($ch);
    
    return json_decode($result, true);
}
?>
```

---

## Next Steps

- [Configuration Guide](configuration.md) - Set up Price Push
- [Troubleshooting](troubleshooting.md) - Common issues and solutions
- [How It Works](how-it-works.md) - Technical details

---

**Need Help?** Contact API support at api-support@walkway.com

