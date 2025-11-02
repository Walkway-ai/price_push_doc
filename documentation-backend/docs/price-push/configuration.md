# Price Push Configuration Guide

## Prerequisites

Before configuring Price Push, ensure you have:

- ✅ Active Walkway SaaS subscription
- ✅ Owner or Admin role in your subscription
- ✅ Ventrata account and API key
- ✅ Products configured in your system

## Step-by-Step Configuration

### Step 1: Access Subscription Settings

1. Log into your Walkway dashboard
2. Navigate to **Subscription Management**
3. Select your subscription
4. Click on **Settings** or **Edit**

!!! tip "Finding Subscription Settings"
    Look for the gear icon ⚙️ or "Settings" button in your subscription card.

### Step 2: Configure Ventrata API Key

The Ventrata API key connects your Walkway account to your Ventrata booking system.

**How to get your Ventrata API Key:**

1. Log into your Ventrata account
2. Go to **Settings** → **API Keys**
3. Click **Create New API Key**
4. Copy the generated key
5. Save it securely (you won't see it again!)

**Add the key to Walkway:**

1. In Walkway subscription settings, find **Ventrata Configuration**
2. Paste your API key in the **Ventrata API Key** field
3. Click **Save** or **Update**

!!! warning "Keep It Secret"
    Your API key is like a password. Never share it publicly or commit it to code repositories.

**What happens next:**
- ✅ API key is encrypted and stored securely
- ✅ Configuration timestamp is recorded
- ✅ All subscription members receive the API key
- ✅ System tests the connection to Ventrata

### Step 3: Enable Price Push

Once your Ventrata API key is configured:

1. In subscription settings, find **Price Push**
2. Toggle the **Enable Price Push** switch to ON
3. Confirm the action
4. Click **Save Changes**

**Behind the scenes:**
- ✅ Subscription `pricePush` field set to `true`
- ✅ All **Owners** get Price Push access
- ✅ All **Admins** get Price Push access
- ✅ Members and Viewers remain without access

!!! success "Automatic Propagation"
    You don't need to manually enable Price Push for each team member. The system does this automatically based on their role.

### Step 4: Verify Configuration

Check that everything is set up correctly:

1. Go to **User Management**
2. Click **Refresh** to reload user data
3. Check user details:
   - ✅ Owners should show Price Push: **Enabled**
   - ✅ Admins should show Price Push: **Enabled**
   - ❌ Members should show Price Push: **Disabled**
   - ❌ Viewers should show Price Push: **Disabled**

### Step 5: Test Price Push

Before using in production, test with a non-critical product:

1. Select a test product
2. Update its price
3. Click **Push to Ventrata**
4. Verify the price updated in Ventrata
5. Check the audit log for the change

!!! tip "Test Environment"
    If available, use a Ventrata test environment for initial testing.

## Configuration Checklist

Use this checklist to ensure proper setup:

- [ ] Ventrata account created
- [ ] Ventrata API key obtained
- [ ] API key added to Walkway
- [ ] Connection tested successfully
- [ ] Price Push enabled for subscription
- [ ] Owner/Admin access verified
- [ ] Test price push completed
- [ ] Audit logs reviewed
- [ ] Team members notified

## Advanced Configuration

### Multiple Subscriptions

If you manage multiple subscriptions:

1. Each subscription needs its **own Ventrata API key**
2. Enable Price Push **per subscription**
3. Members can belong to multiple subscriptions
4. Price Push access is **subscription-specific**

**Example:**
```
User John Doe:
├── Subscription A (Role: Admin)
│   └── Price Push: ✅ Enabled
└── Subscription B (Role: Member)
    └── Price Push: ❌ Disabled
```

### Member-Specific Settings

While Price Push propagates automatically, you can:

1. **Manually enable** for specific members (via User Management)
2. **Temporarily disable** for an admin (via User Management)
3. **Override** subscription settings on a per-user basis

!!! warning "Manual Overrides"
    Manual changes are NOT overwritten by automatic propagation unless the subscription settings change again.

### Ventrata API Key Rotation

For security, rotate your API keys regularly:

1. Generate new API key in Ventrata
2. Update in Walkway subscription settings
3. **Automatic**: New key propagates to all members
4. Revoke old key in Ventrata

**Propagation Example:**
```json
{
  "subscriptionId": "sub-123",
  "ventrataApiKey": "new-key-xyz",
  "updatedUsersCount": 5,
  "message": "API key propagated to all members"
}
```

## Troubleshooting Setup

### Issue: API Key Not Working

**Symptoms:**
- "Invalid API key" error
- Connection fails
- Price push rejected

**Solutions:**
1. ✅ Verify key was copied correctly (no extra spaces)
2. ✅ Check key hasn't expired
3. ✅ Confirm key has correct permissions in Ventrata
4. ✅ Test key directly in Ventrata API explorer

### Issue: Price Push Toggle Not Appearing

**Symptoms:**
- Toggle missing in subscription settings
- Cannot enable Price Push

**Solutions:**
1. ✅ Verify you have Owner or Admin role
2. ✅ Check subscription is active
3. ✅ Refresh the page
4. ✅ Clear browser cache
5. ✅ Contact support if persistent

### Issue: Members Not Getting Access

**Symptoms:**
- Owners/Admins show Price Push disabled
- Access not propagating

**Solutions:**
1. ✅ Verify Price Push enabled at subscription level
2. ✅ Check member roles are correct
3. ✅ Click "Refresh" in User Management
4. ✅ Check backend logs for propagation errors
5. ✅ Try disabling and re-enabling Price Push

## Security Best Practices

### API Key Security

- 🔐 Never share your API key
- 🔐 Store in password manager
- 🔐 Rotate keys every 90 days
- 🔐 Revoke compromised keys immediately
- 🔐 Use separate keys for testing and production

### Access Control

- 👥 Regularly review team member roles
- 👥 Remove inactive members
- 👥 Use least-privilege principle
- 👥 Audit Price Push usage monthly
- 👥 Document role changes

### Monitoring

- 📊 Review audit logs weekly
- 📊 Set up alerts for failed pushes
- 📊 Monitor unusual pricing patterns
- 📊 Track API key usage

## Configuration via API

For developers, you can configure Price Push programmatically:

### Enable Price Push
```bash
curl -X PATCH https://api.walkway.com/api/subscriptions/{id}/price-push \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"pricePush": true}'
```

### Update Ventrata API Key
```bash
curl -X PATCH https://api.walkway.com/api/subscriptions/{id}/ventrata-api-key \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"ventrataApiKey": "your-new-key"}'
```

### Update Subscription
```bash
curl -X PATCH https://api.walkway.com/api/subscriptions/{id} \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "pricePush": true,
    "ventrataApiKey": "your-key"
  }'
```

[View complete API documentation →](api-endpoints.md)

## Next Steps

After configuration:

1. [Manage Price Units →](price-units.md) - Set up ADULT, CHILD, SENIOR pricing
2. [Get Availability Data →](availability.md) - Fetch real-time availability
3. [Track Price Changes →](history.md) - View complete audit trail
4. [Explore Use Cases →](use-cases.md) - See real-world examples
5. [Read API Documentation →](api-endpoints.md) - Full API reference

---

**Need Help?** Check our [Troubleshooting Guide](troubleshooting.md) or contact support at support@walkway.com

