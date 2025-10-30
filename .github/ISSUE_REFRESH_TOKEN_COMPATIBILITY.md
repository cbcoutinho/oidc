# [BREAKING CHANGE] Implement OpenID Connect offline_access Scope for Refresh Tokens

## Summary

The OIDC Identity Provider app currently issues refresh tokens unconditionally to all clients, regardless of whether the `offline_access` scope was requested. This behavior violates the OpenID Connect Core 1.0 specification (Section 11).

This issue tracks the implementation of proper `offline_access` scope support with a phased migration plan to maintain backward compatibility.

## Current Behavior

**Refresh tokens are always issued** when clients exchange an authorization code for tokens, regardless of the scopes requested.

```http
POST /apps/oidc/token
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code&
code=xxx&
client_id=xxx&
client_secret=xxx

# Response always includes refresh_token:
{
  "access_token": "...",
  "token_type": "Bearer",
  "expires_in": 900,
  "refresh_token": "...",  // ← Always present
  "id_token": "..."
}
```

## Expected Behavior (OIDC Compliant)

According to [OpenID Connect Core 1.0 Section 11](https://openid.net/specs/openid-connect-core-1_0.html#OfflineAccess):

> The use of Refresh Tokens is not exclusive to the offline_access use case. **However**, when `offline_access` is requested, the Authorization Server MUST issue a Refresh Token. When `offline_access` is **NOT** requested, the Authorization Server **SHOULD NOT** issue a Refresh Token.

**Expected behavior:**

1. Client requests `scope=openid profile email offline_access`
2. User consents to offline access
3. Token response includes `refresh_token`

**Without offline_access:**

1. Client requests `scope=openid profile email` (no offline_access)
2. User consents to requested scopes
3. Token response does NOT include `refresh_token`

## Why This Matters

### 1. User Privacy and Control
Users should have explicit control over whether applications can access their data when they're not present (offline access). Currently, users cannot deny this capability.

### 2. Security Best Practice
Refresh tokens are long-lived credentials. Issuing them unnecessarily increases the attack surface. Applications that don't need offline access shouldn't receive refresh tokens.

### 3. Standards Compliance
Non-compliance with OpenID Connect Core 1.0 affects:
- Enterprise certification requirements
- Interoperability with other OIDC providers
- Regulatory compliance (GDPR, HIPAA, etc.)

### 4. Clear Consent Semantics
The `offline_access` scope provides clear semantics: "This app wants to access your data even when you're not using it." Users understand and can make informed decisions.

## Impact Analysis

### Who is Affected?

**✅ Currently Working (will continue working with migration plan):**
- Background job applications that use refresh tokens
- MCP (Model Context Protocol) servers requiring persistent access
- Mobile applications with long-lived sessions
- Monitoring and automation tools

**⚠️ May Need Updates:**
- Client applications that don't explicitly request `offline_access` in their scope
- Applications assuming refresh tokens are always available
- Custom integrations relying on unconditional refresh token issuance

**✅ Benefits:**
- Interactive web applications that don't need offline access (reduced token storage)
- Users gain granular control over offline access
- Administrators gain visibility into offline access grants

## Proposed Solution

Implement `offline_access` scope support with a **phased, backward-compatible migration**:

### Phase 1: Introduction (v1.12.0) - No Breaking Changes

**Add configuration option** (default: disabled for backward compatibility):

```
Settings > OIDC > ☐ Enforce offline_access scope for refresh tokens
```

**When disabled (default):**
- Current behavior maintained (always issue refresh tokens)
- No breaking changes

**When enabled:**
- Refresh tokens only issued if `offline_access` scope requested
- OIDC compliant behavior

**Also includes:**
- Add `offline_access` to discovery document (`scopes_supported`)
- Auto-add `offline_access` to existing clients' allowed scopes
- Update documentation with migration guidance

### Phase 2: Encouragement (v1.13.0) - Still No Breaking Changes

- Documentation encourages admins to enable enforcement
- Admin UI shows recommendation to enable
- Client migration guide published
- Deprecation notice for unconditional refresh token issuance

### Phase 3: Default Enforcement (v2.0.0) - Breaking Change

- Configuration option defaults to **enabled**
- Clients MUST request `offline_access` to receive refresh tokens
- Admins can temporarily disable if needed
- Clear migration path documented

## Migration Guide for Client Developers

### If your application needs refresh tokens (background jobs, long-lived sessions):

**Before:**
```http
GET /apps/oidc/authorize?
  client_id=xxx&
  redirect_uri=xxx&
  response_type=code&
  scope=openid profile email
```

**After:**
```http
GET /apps/oidc/authorize?
  client_id=xxx&
  redirect_uri=xxx&
  response_type=code&
  scope=openid profile email offline_access  ← Add this
```

### Testing Your Application

1. **Update authorization request** to include `offline_access` scope
2. **Test token endpoint response** - verify `refresh_token` is present
3. **Test refresh token flow** - verify you can exchange refresh tokens for new access tokens
4. **Test consent screen** - verify offline_access permission is displayed

### If your application does NOT need refresh tokens:

No action required. Your application will continue working and will benefit from improved security (no unnecessary refresh tokens).

## Implementation Details

### Files Affected

- `lib/Controller/OIDCApiController.php` - Conditional refresh token issuance
- `lib/Util/DiscoveryGenerator.php` - Add offline_access to scopes_supported
- `lib/AppInfo/Application.php` - Configuration constant
- Admin settings UI - Toggle for enforcement
- Consent screen - Display offline_access permission
- User settings - View/revoke offline access grants

### Configuration

**New app config setting:**
```php
'enforce_offline_access_scope' => false  // Default in v1.12.0
'enforce_offline_access_scope' => true   // Default in v2.0.0
```

**Admin can configure via:**
- Admin settings UI (Settings > OIDC)
- OCC command: `occ config:app:set oidc enforce_offline_access_scope --value=true`

## Timeline

| Version | Expected Date | Milestone |
|---------|---------------|-----------|
| **v1.12.0** | Q1 2025 | Implementation with toggle (default: off) |
| **v1.13.0** | Q2 2025 | Encouragement phase |
| **v2.0.0** | Q3 2025 | Default enforcement (breaking change) |

**Total migration window:** ~6 months from v1.12.0 to v2.0.0

## Testing Needed

### Before Release (v1.12.0)

- [ ] Unit tests for offline_access logic
- [ ] Integration tests with/without offline_access scope
- [ ] Test enforcement enabled/disabled modes
- [ ] Test consent screen displays offline_access
- [ ] Test discovery document includes offline_access
- [ ] Test existing clients get offline_access in allowed scopes
- [ ] Test backward compatibility mode
- [ ] Test user settings dashboard for offline access management
- [ ] Performance testing for token endpoint
- [ ] Security review of implementation

### Community Testing

We need help testing this change with various client types:
- Background job applications
- MCP servers
- Mobile applications
- Web applications (SPA and server-side)
- API-only applications
- Federated identity scenarios

## Questions for Discussion

1. **Default Scope**: Should `offline_access` be included in the default scope (`openid profile email roles offline_access`)? This would make migration easier but might grant offline access more broadly than necessary.

2. **Migration Timeline**: Is 6 months sufficient for the migration window? Should we extend to 9 or 12 months?

3. **Consent Default**: Should `offline_access` be pre-checked on the consent screen if requested in the scope?

4. **Existing Clients**: Should we auto-add `offline_access` to all existing clients during upgrade? Or require manual configuration?

5. **Admin Override**: Should there be a per-client override to exempt specific clients from enforcement?

## References

- **OpenID Connect Core 1.0:** [Section 11 - Offline Access](https://openid.net/specs/openid-connect-core-1_0.html#OfflineAccess)
- **RFC 6749 (OAuth 2.0):** [Section 1.5 - Refresh Token](https://datatracker.ietf.org/doc/html/rfc6749#section-1.5)
- **OAuth 2.0 Security BCP:** [Refresh Token Best Practices](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-security-topics#section-4.13)
- **Related ADR:** [ADR-0002: Implement offline_access Scope Compliance](../docs/adr/0002-implement-offline-access-scope-compliance.md)

## How to Provide Feedback

Please comment on this issue if:
- You maintain a client application that uses this OIDC provider
- You have concerns about the migration timeline
- You have suggestions for the implementation approach
- You have questions about how this affects your use case
- You want to help test the implementation

## Labels

- `enhancement` - New feature implementation
- `breaking-change` - Will break clients in v2.0.0 if not updated
- `compliance` - Standards compliance issue
- `needs-feedback` - Community input requested
- `phased-rollout` - Multi-version implementation plan

## Related Issues

- #XXX - User consent management
- #XXX - Token lifecycle management
- #XXX - Client credentials grant type support

---

**Note to maintainers:** This issue is based on [ADR-0002](../docs/adr/0002-implement-offline-access-scope-compliance.md). Please review the ADR for complete implementation details, testing strategy, and technical specifications.
