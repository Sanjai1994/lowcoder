# SCIM + SSO Fix Summary

## Problem Identified

When using the SCIM endpoint to create users with blank passwords (for SSO), the code was creating users with:

- ❌ **EMAIL auth source**
- ❌ Manually built `AuthUser` without `authContext`
- ❌ NullPointerException when calling `getSource()` or `toAuthConnection()`

This prevented SSO login because:

- User created with EMAIL source
- SSO attempts to authenticate with OIDC/OAuth source
- Sources don't match → login fails or duplicate user created

## Solutions Implemented

### Solution 1: Fixed NullPointerException (Previous)

**File**: `OrgApiService.java`

- Changed from manually building `AuthUser` to using `authenticateByForm`
- Sets up `authContext` properly with EMAIL auth config
- ✅ Fixes NPE
- ❌ Still creates EMAIL auth source (wrong for SSO)

### Solution 2: JIT Provisioning (Current - Recommended) ✅

**No code changes needed** - JIT is already built-in!

**How it works**:

1. Configure Entra ID OAuth provider in Lowcoder with `enableRegister: true`
2. User logs in via SSO for first time
3. Lowcoder automatically creates user with correct auth source (OIDC/OAuth)
4. User logged in successfully

**Benefits**:

- ✅ Correct auth source (Entra ID, not EMAIL)
- ✅ Simpler setup
- ✅ No SCIM needed
- ✅ Works out-of-the-box

### Solution 3: SCIM + JIT Combined (Alternative)

**File**: `UserController.java` - `createSCIMUserAndAddToOrg()`

- Creates placeholder user via SCIM (no auth connection)
- On first SSO login, JIT adds correct auth connection
- ✅ Supports SCIM for compliance
- ✅ Correct auth source via SSO
- ⚠️ More complex

## Recommended Approach

### For Most Users: Use JIT Provisioning Only

1. **Don't use SCIM endpoint** for user creation
2. **Configure Entra ID OAuth** in Lowcoder settings
3. **Enable registration**: `enableRegister: true`
4. **Users auto-created** on first SSO login
5. **Correct auth source** automatically

### For Compliance Requirements: Use SCIM + JIT

1. **Use updated SCIM endpoint** to pre-provision users
2. **Configure Entra ID OAuth** for SSO
3. **First login adds** correct auth connection via JIT
4. **Both audit trails** (SCIM + SSO)

## Files Modified

1. ✅ `/server/api-service/lowcoder-server/src/main/java/org/lowcoder/api/usermanagement/UserController.java`

   - Updated `createSCIMUserAndAddToOrg()` method
   - Creates placeholder users that work with JIT

2. ✅ `/docs/setup-and-run/self-hosting/JIT_PROVISIONING_WITH_ENTRA_ID.md`
   - Complete setup guide
   - Step-by-step instructions
   - Troubleshooting section

## Testing

### Test JIT Provisioning

```bash
# 1. Configure Entra ID provider in Lowcoder admin UI
# 2. Logout from Lowcoder
# 3. Visit login page
# 4. Click "Sign in with Entra ID"
# 5. Authenticate with Microsoft
# 6. Verify user created with Entra ID auth source
```

### Test SCIM + JIT

```bash
# 1. Call SCIM endpoint to create placeholder user
curl -X POST "https://your-domain.com/api/v1/organizations/{orgId}/scim/users" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"userName": "user@domain.com", "emails": [{"value": "user@domain.com"}]}'

# 2. User logs in via SSO
# 3. Verify user has Entra ID auth connection added
```

## Next Steps

1. **Rebuild Docker image** with the updated code

   ```bash
   docker build -f deploy/docker/Dockerfile --target lowcoder-ce-api-service -t lowcoderorg/lowcoder-ce-api-service:latest .
   ```

2. **Configure Entra ID** following the guide in `JIT_PROVISIONING_WITH_ENTRA_ID.md`

3. **Test user login** via SSO

4. **Verify correct auth source** in user management panel

## Key Takeaway

**JIT Provisioning is the recommended approach** for SSO with Entra ID:

- ✅ Simpler to set up
- ✅ Works correctly out-of-the-box
- ✅ No code changes required
- ✅ Proper auth source management

Only use SCIM if you have specific compliance requirements for pre-provisioning users.
