# Price Push Troubleshooting

## Common Issues and Solutions

This guide helps you resolve common Price Push problems. Issues are organized by symptom for easy navigation.

---

## Issue: Price Push Toggle Not Visible

### Symptoms
- Price Push toggle missing in subscription settings
- "Price Push" option not showing anywhere
- Feature appears to be disabled

### Possible Causes & Solutions

#### 1. Insufficient Permissions

**Check:**
- Are you logged in as Owner or Admin?
- Do you have `canEditSubscription` permission?

**Solution:**
```
✅ Ask subscription Owner to:
1. Verify your role (should be Owner or Admin)
2. Check your permissions in Member Management
3. Upgrade your role if needed
```

#### 2. Subscription Not Active

**Check:**
- Is your subscription status "ACTIVE"?
- Has your trial expired?

**Solution:**
```
✅ Contact billing to:
1. Verify subscription status
2. Renew if expired
3. Activate trial/paid plan
```

#### 3. Browser Cache Issue

**Solution:**
```
✅ Clear browser data:
1. Press Ctrl+Shift+Delete (Windows) or Cmd+Shift+Delete (Mac)
2. Select "Cached images and files"
3. Clear data and reload page
```

**Or use hard refresh:**
```
Windows/Linux: Ctrl + Shift + R
Mac: Cmd + Shift + R
```

---

## Issue: Price Push Enabled But Not Working

### Symptoms
- Toggle shows "enabled" but prices don't push
- Users can't access Price Push feature
- Error messages when trying to push prices

### Possible Causes & Solutions

#### 1. Ventrata API Key Missing

**Check:**
```bash
GET /api/subscriptions/{id}/ventrata-api-key
```

**Solution:**
```
✅ Configure API key:
1. Get key from Ventrata dashboard
2. Go to Subscription Settings
3. Add key in "Ventrata Configuration"
4. Save and test connection
```

[See Configuration Guide →](configuration.md#step-2-configure-ventrata-api-key)

#### 2. Users Not in Owner/Admin Role

**Check:**
```bash
GET /api/users/flags
# Look for pricePush: true/false for each user
```

**Expected:**
- Owners: `pricePush: true` ✅
- Admins: `pricePush: true` ✅
- Members: `pricePush: false` ❌
- Viewers: `pricePush: false` ❌

**Solution:**
```
✅ If wrong roles:
1. Go to User Management
2. Change role to Admin or Owner
3. Price Push will enable automatically
4. User should refresh their session
```

#### 3. Backend Not Propagating Changes

**Check Backend Logs:**
```
Look for:
✅ "Price Push propagé à X membres OWNER/ADMIN"
✅ "Ventrata API Key propagée à X membres"

If missing, backend may have failed.
```

**Solution:**
```
✅ Retry propagation:
1. Disable Price Push (toggle off)
2. Wait 5 seconds
3. Enable Price Push (toggle on)
4. Check logs for confirmation
5. Verify in User Management
```

---

## Issue: Members Can't See Their Price Push Status

### Symptoms
- User Management doesn't show pricePush field
- After enabling, users still show "disabled"
- Inconsistent data between backend and frontend

### Possible Causes & Solutions

#### 1. Frontend Not Loading Flags

**Check Browser Console:**
```javascript
// Should see:
GET /api/users/flags
// Response should include pricePush field
```

**If `pricePush` is missing from response:**

**Solution:**
```
✅ Backend fix required:
1. Restart backend server
2. Verify /api/users/flags includes pricePush
3. Check user.service.ts has pricePush in select
4. Regenerate Prisma client if needed
```

#### 2. User Session Not Refreshed

**Solution:**
```
✅ Refresh user session:
1. Click "Refresh" button in User Management
2. Or log out and log back in
3. Or clear browser cache
```

#### 3. Database Not Updated

**Check Database:**
```sql
SELECT id, email, "pricePush", "ventrataApiKey" 
FROM "User" 
WHERE id = 'user-id-here';
```

**If pricePush is NULL or wrong:**

**Solution via API:**
```bash
# Manual fix (use only if automatic propagation failed)
curl -X PATCH https://api.walkway.com/api/admin/user-id/price-push-access \
  -H "Authorization: Bearer TOKEN" \
  -d '{"pricePush": true}'
```

!!! warning "Last Resort"
    Manual updates should only be used if automatic propagation fails. They may be overwritten by future subscription changes.

---

## Issue: Role Change Not Updating Price Push

### Symptoms
- Member promoted to Admin but still no Price Push
- Admin demoted to Member but still has Price Push
- Price Push doesn't match role

### Possible Causes & Solutions

#### 1. Backend Logic Not Executing

**Check Backend Logs:**
```
Expected logs:
✅ "Promotion vers ADMIN: Price Push activé pour user@example.com"
OR
⚠️ "Rétrogradation vers MEMBER: Price Push désactivé for user@example.com"
```

**If logs are missing:**

**Solution:**
```
✅ Force role update:
1. Change role back to original
2. Save changes
3. Wait 5 seconds
4. Change role to desired role again
5. Save and verify logs
```

#### 2. User Needs to Refresh Session

**Solution:**
```
✅ User must:
1. Log out completely
2. Log back in
3. Changes should be visible
```

#### 3. Database Transaction Failed

**Check Backend Error Logs:**
```
Look for:
❌ "Transaction already closed"
❌ "Failed to update user"
❌ Database connection errors
```

**Solution:**
```
✅ If transaction errors:
1. Restart backend service
2. Check database connectivity
3. Retry role change
4. Contact support if persistent
```

---

## Issue: Ventrata API Key Not Propagating

### Symptoms
- API key set at subscription level
- Members don't receive the key
- Some members have key, others don't

### Possible Causes & Solutions

#### 1. Check Who Should Receive Key

**Important:** Ventrata API key propagates to **ALL members**, not just Owner/Admin.

**Verify propagation:**
```bash
# Check subscription
GET /api/subscriptions/{id}/ventrata-api-key

# Check individual users
GET /api/users/{user-id}/ventrata-api-key
```

**Solution if missing:**
```
✅ Re-save API key:
1. Go to Subscription Settings
2. Update Ventrata API Key (can be same key)
3. Save changes
4. Check backend logs for "propagée à X membres"
5. Verify each user received it
```

#### 2. New Members Not Inheriting Key

**For newly added members:**

**Solution:**
```
✅ Automatic inheritance:
1. New members should inherit automatically
2. If not, remove and re-add member
3. Or manually update via API:

curl -X PATCH /api/users/{user-id}/ventrata-api-key \
  -H "Authorization: Bearer TOKEN" \
  -d '{"ventrataApiKey": "key-here"}'
```

---

## Issue: Price Push Works But Audit Trail Missing

### Symptoms
- Prices update successfully
- No record in audit log
- Can't track who made changes

### Solution

Check audit trail location:

```bash
GET /api/ventrata/price-changes
# Filter by:
# - subscriptionId
# - userId
# - dateRange
```

**If truly missing:**
```
✅ Contact support with:
1. Timestamp of price change
2. User who made the change
3. Products affected
4. Screenshots if available
```

---

## Issue: "Transaction Already Closed" Error

### Symptoms
- Backend logs show "Transaction already closed"
- Price push fails with 500 error
- Operations timeout

### Possible Causes & Solutions

#### 1. Prisma Transaction Timeout

**This was fixed in recent update. If you still see it:**

**Check Code:**
```typescript
// Transactions should have timeout options:
await this.prisma.$transaction(async (tx) => {
  // ... code ...
}, {
  maxWait: 10000,  // Should be present
  timeout: 30000,  // Should be present
});
```

**Solution:**
```
✅ Update backend:
1. Pull latest code from repository
2. Run: npm install
3. Run: npx prisma generate
4. Restart backend service
```

#### 2. Long-Running Operations

**If timeouts persist:**

**Solution:**
```
✅ Optimize:
1. Reduce batch sizes
2. Process in smaller chunks
3. Contact support for performance tuning
```

---

## Issue: Wrong Users Getting Price Push Access

### Symptoms
- Members have Price Push (should be disabled)
- Viewers have Price Push (should be disabled)
- Inconsistent access across team

### Immediate Fix

**Via Admin Panel:**
```
1. Go to User Management
2. For each affected user:
   - If Member/Viewer with pricePush=true → Set to false
   - If Owner/Admin with pricePush=false → Set to true
3. Save changes
```

**Via API:**
```bash
# Fix individual user
curl -X PATCH /api/admin/{user-id}/price-push-access \
  -H "Authorization: Bearer TOKEN" \
  -d '{"pricePush": false}'  # or true
```

### Root Cause Investigation

1. Check subscription Price Push status
2. Review recent role changes
3. Check for manual overrides
4. Review backend logs for propagation errors

---

## Diagnostic Checklist

When experiencing issues, run through this checklist:

### Backend Health

- [ ] Backend server is running
- [ ] Database is accessible
- [ ] No errors in backend logs
- [ ] Prisma client is up to date (`npx prisma generate`)
- [ ] Latest code deployed

### Subscription Configuration

- [ ] Subscription is ACTIVE
- [ ] Price Push is enabled (toggle on)
- [ ] Ventrata API key is configured
- [ ] API key is valid (test in Ventrata)

### User Configuration

- [ ] User has correct role
- [ ] User session is fresh (logged in recently)
- [ ] User pricePush matches role expectations
- [ ] User has ventrataApiKey if subscription has one

### Frontend

- [ ] Browser cache cleared
- [ ] Using latest frontend version
- [ ] No JavaScript errors in console
- [ ] API responses include pricePush field

---

## Getting Help

If you've tried everything and still have issues:

### 1. Collect Information

```
✅ Gather:
- Subscription ID
- User ID(s) affected
- Exact error messages
- Screenshots
- Backend logs (if accessible)
- Steps to reproduce
```

### 2. Check Status Page

Visit: status.walkway.com

### 3. Contact Support

**Email:** support@walkway.com

**Include:**
- All information from step 1
- This troubleshooting guide steps you tried
- Your urgency level

**Response Times:**
- Critical (site down): < 1 hour
- High (feature broken): < 4 hours
- Normal (questions): < 24 hours

---

## Prevention Tips

### For Administrators

✅ **Weekly:**
- Review user roles
- Verify Price Push access matches roles
- Check audit logs

✅ **Monthly:**
- Rotate Ventrata API keys
- Review team member list
- Remove inactive users

✅ **Quarterly:**
- Full access audit
- Update documentation
- Train new team members

### For Developers

✅ **Before Deployment:**
- Run all tests
- Check Prisma migrations
- Verify environment variables
- Test Price Push in staging

✅ **After Deployment:**
- Monitor error logs
- Check propagation works
- Verify API responses include pricePush
- Test with real user account

---

## Known Issues

### Current Known Issues

**None at this time** ✅

Last updated: November 2025

### Recently Fixed

- ✅ Transaction timeout errors (Fixed: Nov 2025)
- ✅ pricePush missing from /api/users/flags (Fixed: Nov 2025)
- ✅ No automatic propagation on subscription edit (Fixed: Nov 2025)

---

**Still stuck?** Contact our technical team at tech-support@walkway.com with detailed information about your issue.

