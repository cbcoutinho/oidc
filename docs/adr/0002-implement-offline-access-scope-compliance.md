# ADR-0002: Implement offline_access Scope Compliance

## Status

Proposed

## Context

### Current Behavior

The Nextcloud OIDC Identity Provider currently issues refresh tokens unconditionally to all clients, regardless of whether the `offline_access` scope was requested. This behavior violates the OpenID Connect Core 1.0 specification.

**Evidence:**
- `lib/Controller/OIDCApiController.php:304` - Refresh token always included in token response
- `lib/Util/DiscoveryGenerator.php:101-105` - `offline_access` not advertised in `scopes_supported`
- `lib/AppInfo/Application.php:25` - Default scope does not include `offline_access`

**Current token response:**
```php
$responseData = [
    'access_token' => $accessToken->getAccessToken(),
    'token_type' => 'Bearer',
    'expires_in' => $expireTime,
    'refresh_token' => $newCode,  // Always present
    'id_token' => $jwt,
];
```

### OpenID Connect Core 1.0 Requirements

**Section 11: Offline Access**

The specification states:

> OpenID Connect supports the use of refresh tokens to obtain new access tokens. This scope value requests that an OAuth 2.0 Refresh Token be issued that can be used to obtain an access token that grants access to the End-User's UserInfo Endpoint even when the End-User is not present (not logged in).

Key requirements:
1. **`offline_access` scope MUST result in a Refresh Token being issued**
2. **Without `offline_access`, Authorization Servers SHOULD NOT issue Refresh Tokens**
3. **User consent is important** - Users should explicitly consent to offline access
4. **Offline access means access when the user is NOT present/logged in**

### Why This Matters

**1. User Control and Privacy:**
- Users cannot distinguish between online-only and offline-capable applications
- No ability to deny offline access while still authorizing the application
- Reduced user control over their data and privacy

**2. Security Principle:**
- Offline access represents elevated privilege (access when user not present)
- Long-lived refresh tokens increase attack surface
- Should require explicit user consent

**3. Compliance and Auditing:**
- Cannot meet regulatory requirements for granular consent (GDPR, HIPAA, etc.)
- No audit trail distinguishing offline vs. online access grants
- Cannot revoke offline access separately from online access

**4. Standards Compliance:**
- Violates OpenID Connect Core 1.0 Section 11
- Breaks interoperability with compliant OIDC clients
- May cause issues in federated identity scenarios

**5. Enterprise Requirements:**
- Many enterprises require OIDC-compliant identity providers
- Certification programs check for proper offline_access implementation
- May block enterprise adoption

### Current Impact

**Positive (current convenience):**
- Background jobs automatically receive refresh tokens
- MCP servers work without additional configuration
- Simpler client implementation (no need to request offline_access)

**Negative (compliance and security):**
- Users cannot deny offline access
- Unnecessary refresh tokens issued to web apps that don't need them
- Non-compliant with OIDC specification
- Cannot implement proper consent granularity
- Security audits will flag this as non-compliant

### Use Cases Affected

**1. Background Jobs / MCP Servers:**
- **Current:** Automatically work with refresh tokens
- **After fix:** Must explicitly request `offline_access` scope
- **Example:** `scope=openid profile email offline_access`

**2. Interactive Web Applications:**
- **Current:** Unnecessarily receive refresh tokens
- **After fix:** Can use online-only access (no refresh token)
- **Benefit:** Reduced token storage and attack surface

**3. Mobile Applications:**
- **Current:** Always get refresh tokens
- **After fix:** Explicitly request when needed
- **Benefit:** Clear user consent for offline access

**4. Short-lived SPA Sessions:**
- **Current:** Get refresh tokens they may not need
- **After fix:** Can skip refresh tokens for better security

## Decision

We will implement proper `offline_access` scope support in compliance with OpenID Connect Core 1.0 Section 11, using a phased approach with backward compatibility.

### Core Implementation

**1. Make Refresh Token Issuance Conditional**

Update token endpoint to only issue refresh tokens when `offline_access` scope is present:

```php
$responseData = [
    'access_token' => $accessToken->getAccessToken(),
    'token_type' => 'Bearer',
    'expires_in' => $expireTime,
    'id_token' => $jwt,
];

// Only issue refresh token if offline_access was requested and granted
$scopeArray = preg_split('/ +/', $accessToken->getScope());
if (in_array('offline_access', $scopeArray)) {
    $responseData['refresh_token'] = $newCode;
    if ($refreshExpireTime !== 'never') {
        $responseData['refresh_expires_in'] = (int)$refreshExpireTime;
    }
    $this->logger->info('Issued refresh token for offline_access - User: ' . $uid . ', Client: ' . $client_id);
}
```

**2. Add offline_access to Discovery Metadata**

Advertise support in `.well-known/openid-configuration`:

```php
$defaultScopes = [
    'openid',
    'profile',
    'email',
    'roles',
    'groups',
    'offline_access',  // Add this
];
```

**3. Update Consent Screen**

Enhance user consent interface to show `offline_access` as a distinct, meaningful permission:
- Display icon indicating "access when you're away"
- Explain what offline access means in user-friendly language
- Allow users to grant/deny offline access separately
- Show offline access status in user settings for revocation

**4. Scope Validation**

Ensure `offline_access` is properly validated:
- Accept in authorization requests
- Preserve through consent flow
- Include in token claims
- Validate against client allowed scopes

### Backward Compatibility Strategy

**Configuration Option: `enforce_offline_access_scope`**

Add a new app configuration setting in admin settings:

```php
// Default: false (maintains current behavior)
$enforceOfflineAccess = $this->appConfig->getAppValueString(
    Application::APP_CONFIG_ENFORCE_OFFLINE_ACCESS,
    'false'
) === 'true';
```

**Behavior:**
- **`false` (default):** Issue refresh tokens unconditionally (current behavior)
- **`true`:** Require `offline_access` scope for refresh tokens (compliant behavior)

**Migration Timeline:**
1. **Phase 1 (v1.12.0):** Introduce setting, default to `false`
2. **Phase 2 (v1.13.0):** Encourage admins to enable via documentation
3. **Phase 3 (v2.0.0):** Change default to `true` (breaking change)

**Admin Interface:**

Add to Settings > OIDC:
```
☐ Enforce offline_access scope for refresh tokens (OIDC compliant)

When enabled, clients must explicitly request the 'offline_access'
scope to receive refresh tokens. This is required by the OpenID Connect
specification but may break existing clients that don't request this scope.

Recommendation: Enable this after updating your OAuth clients to request
the 'offline_access' scope in their authorization requests.
```

### Client Migration Support

**1. Auto-add offline_access to Existing Clients**

When upgrading, automatically add `offline_access` to all existing clients' allowed scopes:

```php
// Migration script
public function addOfflineAccessToClients() {
    $clients = $this->clientMapper->findAll();
    foreach ($clients as $client) {
        $allowedScopes = $client->getAllowedScopes();
        if (!empty($allowedScopes) && strpos($allowedScopes, 'offline_access') === false) {
            $client->setAllowedScopes($allowedScopes . ' offline_access');
            $this->clientMapper->update($client);
            $this->logger->info('Added offline_access to client: ' . $client->getClientIdentifier());
        }
    }
}
```

**2. Update Default Scope**

Option A: Keep default scope as-is (clients must explicitly request)
```php
public const DEFAULT_SCOPE = 'openid profile email roles';
```

Option B: Include offline_access in default (automatic for most clients)
```php
public const DEFAULT_SCOPE = 'openid profile email roles offline_access';
```

**Decision:** Option B for easier migration, can be overridden per-client

**3. Client Update Documentation**

Provide migration guide for client developers:

```markdown
## Migrating to offline_access Support

If your application needs refresh tokens (background jobs, long-lived sessions),
update your authorization request to include the offline_access scope:

### Before:
scope=openid profile email

### After:
scope=openid profile email offline_access

### Testing:
1. Make authorization request with new scope
2. Verify refresh_token in token response
3. Test refresh token flow
```

### Token Cleanup

When offline_access is not granted:
- Don't persist the refresh token code in database
- Set refresh token expiry to match access token expiry
- Clean up unused refresh token records periodically

```php
if (!in_array('offline_access', $scopeArray)) {
    // Don't rotate refresh token if offline_access not granted
    // Keep same code but mark for shorter expiry
    $accessToken->setRefreshed($this->time->getTime() + $expireTime);
}
```

### Security Enhancements

**1. Audit Logging**

Log all offline_access grants and refresh token issuances:
```php
$this->logger->info('Offline access granted', [
    'user' => $uid,
    'client' => $client_id,
    'timestamp' => $this->time->getTime(),
    'scopes' => $scope,
]);
```

**2. User Settings Dashboard**

Add to user settings:
- List of clients with offline access
- Timestamp of grant
- Last refresh token use
- Revoke button for each client

**3. Admin Monitoring**

Add admin dashboard showing:
- Clients requesting offline_access most frequently
- Users with most offline access grants
- Refresh token usage statistics

## Consequences

### Positive Consequences

**User Control:**
- Users can explicitly consent to offline access
- Clear distinction between online and offline capabilities
- Ability to revoke offline access separately
- Improved privacy and security awareness

**Security:**
- Reduced refresh token issuance for apps that don't need it
- Smaller attack surface (fewer long-lived tokens)
- Better audit trail for offline access
- Compliance with security best practices

**Standards Compliance:**
- Fully compliant with OpenID Connect Core 1.0 Section 11
- Improves interoperability with other OIDC implementations
- Enables enterprise OIDC certification
- Meets regulatory compliance requirements

**Developer Experience:**
- Clear, standards-based API
- Better documentation alignment with OIDC ecosystem
- Explicit offline access requirements
- Consistent with major providers (Auth0, Okta, Keycloak)

### Negative Consequences

**Breaking Change (when enforced):**
- Existing clients not requesting offline_access will stop receiving refresh tokens
- Background jobs may fail if not updated
- MCP servers need configuration updates
- Requires coordinated client updates

**Migration Complexity:**
- Need to communicate changes to all client developers
- Phased rollout required for backward compatibility
- Testing across all client types
- Support burden during transition

**Implementation Effort:**
- Moderate complexity (2-3 days)
- Database migration for scope handling
- Consent screen updates
- Documentation updates
- Testing requirements

**User Experience:**
- Additional consent prompt for offline_access
- Users may be confused about "offline access" terminology
- Need clear explanations of implications

### Risks and Mitigations

**Risk: Clients break during migration**
- **Mitigation:** Configuration toggle with long phase-in period
- **Mitigation:** Auto-add offline_access to existing clients
- **Mitigation:** Comprehensive migration documentation

**Risk: Users deny offline_access unintentionally**
- **Mitigation:** Clear consent screen language
- **Mitigation:** Contextual help explaining implications
- **Mitigation:** Allow re-consent if user changes mind

**Risk: Background jobs fail without refresh tokens**
- **Mitigation:** Early communication to MCP/job developers
- **Mitigation:** Default scope includes offline_access
- **Mitigation:** Clear error messages when refresh tokens missing

**Risk: Confusion about "offline" terminology**
- **Mitigation:** Use user-friendly language like "access when you're away"
- **Mitigation:** Tooltips and help text
- **Mitigation:** Documentation with clear examples

## Implementation Plan

### Phase 1: Foundation (v1.12.0) - 2-3 days

**Step 1: Add Configuration Option**
- [ ] Add `APP_CONFIG_ENFORCE_OFFLINE_ACCESS` constant
- [ ] Add admin setting UI toggle
- [ ] Default to `false` (current behavior)
- [ ] Add help text explaining impact

**Step 2: Update Discovery Metadata**
- [ ] Add `offline_access` to `$defaultScopes` in DiscoveryGenerator
- [ ] Update scopes_supported in discovery document
- [ ] Verify discovery endpoint output

**Step 3: Implement Conditional Refresh Token Logic**
- [ ] Update `OIDCApiController::getToken()` with offline_access check
- [ ] Add configuration check for enforcement
- [ ] Add logging for refresh token issuance
- [ ] Handle both enforcement modes (on/off)

**Step 4: Scope Handling**
- [ ] Ensure offline_access accepted in authorization requests
- [ ] Preserve offline_access through consent flow
- [ ] Store offline_access in AccessToken scope field
- [ ] Validate offline_access against client allowed scopes

**Step 5: Database Migration**
- [ ] Create migration to add offline_access to existing clients' allowed scopes
- [ ] Test migration on dev/staging
- [ ] Add rollback capability

**Step 6: Testing**
- [ ] Unit tests for offline_access logic
- [ ] Integration tests for token endpoint
- [ ] Test with enforcement enabled/disabled
- [ ] Test scope filtering and validation
- [ ] Test backward compatibility mode

### Phase 2: User Experience (v1.12.0 or v1.13.0) - 2 days

**Step 1: Update Consent Screen**
- [ ] Add offline_access to consent UI
- [ ] Display clear icon and description
- [ ] Explain "access when you're away" concept
- [ ] Show timestamp of offline access grants

**Step 2: User Settings Dashboard**
- [ ] Create section for offline access grants
- [ ] List clients with offline access
- [ ] Show last refresh token usage
- [ ] Add revoke buttons
- [ ] Update timestamp on revocation

**Step 3: Documentation**
- [ ] User-facing documentation explaining offline_access
- [ ] Client migration guide
- [ ] Admin configuration guide
- [ ] Troubleshooting section
- [ ] Update README with scope requirements

### Phase 3: Enhancement (v1.13.0+) - 1-2 days

**Step 1: Admin Monitoring**
- [ ] Dashboard showing offline_access statistics
- [ ] Client-level offline access requests
- [ ] User-level offline access grants
- [ ] Refresh token usage metrics

**Step 2: Security Hardening**
- [ ] Comprehensive audit logging
- [ ] Refresh token cleanup for non-offline access
- [ ] Rate limiting on refresh token grants
- [ ] Anomaly detection for offline access patterns

**Step 3: Update Default Behavior**
- [ ] Include offline_access in DEFAULT_SCOPE
- [ ] Update default client configurations
- [ ] Test with various client types

### Phase 4: Enforcement Transition (v2.0.0) - Breaking Change

**Step 1: Pre-release Communication**
- [ ] Announce breaking change in v2.0.0
- [ ] Email administrators about enforcement
- [ ] Update changelog prominently
- [ ] Provide migration timeline

**Step 2: Enable by Default**
- [ ] Change default to `enforce_offline_access_scope = true`
- [ ] Keep configuration option for admins to disable temporarily
- [ ] Add warning if enforcement disabled

**Step 3: Validation**
- [ ] Test all known client integrations
- [ ] Coordinate with major client developers
- [ ] Monitor support channels for issues
- [ ] Provide hotfix capability if needed

## Testing Strategy

### Unit Tests

```php
class OfflineAccessTest extends TestCase {
    public function testRefreshTokenIssuedWithOfflineAccess() {
        // Test: scope includes offline_access -> refresh_token present
    }

    public function testRefreshTokenNotIssuedWithoutOfflineAccess() {
        // Test: scope lacks offline_access -> no refresh_token
    }

    public function testBackwardCompatibilityMode() {
        // Test: enforcement disabled -> always issue refresh_token
    }

    public function testDiscoveryIncludesOfflineAccess() {
        // Test: scopes_supported includes offline_access
    }

    public function testScopeValidation() {
        // Test: offline_access validated against client allowed scopes
    }
}
```

### Integration Tests

```php
class OfflineAccessIntegrationTest extends TestCase {
    public function testAuthorizationCodeFlowWithOfflineAccess() {
        // Full flow: authorize -> token endpoint -> verify refresh_token
    }

    public function testAuthorizationCodeFlowWithoutOfflineAccess() {
        // Full flow: authorize without scope -> no refresh_token
    }

    public function testRefreshTokenFlowAfterOfflineAccessRevoked() {
        // Revoke offline_access -> refresh token should fail
    }

    public function testConsentScreenShowsOfflineAccess() {
        // Verify offline_access appears in consent UI
    }

    public function testClientMigration() {
        // Verify existing clients get offline_access added
    }
}
```

### Manual Testing Checklist

- [ ] Authorization request with offline_access scope
- [ ] Authorization request without offline_access scope
- [ ] Consent screen displays offline_access correctly
- [ ] Token response includes refresh_token when appropriate
- [ ] Token response excludes refresh_token when appropriate
- [ ] Refresh token flow works with offline_access
- [ ] Refresh token flow fails without offline_access (when enforced)
- [ ] Admin can toggle enforcement setting
- [ ] Discovery document shows offline_access in scopes_supported
- [ ] User can view offline access grants in settings
- [ ] User can revoke offline access for specific clients
- [ ] Backward compatibility mode works correctly

## Open Questions

### 1. Default Scope Inclusion

**Question:** Should `offline_access` be included in the DEFAULT_SCOPE constant?

**Options:**
- **A:** Include it - `'openid profile email roles offline_access'`
  - Pros: Easier migration, most clients get it automatically
  - Cons: May grant offline access unnecessarily
- **B:** Exclude it - `'openid profile email roles'`
  - Pros: Explicit opt-in, better security posture
  - Cons: More client updates required

**Recommendation:** Option A during migration period, consider Option B for v2.0+

### 2. Enforcement Timeline

**Question:** How long should the phase-in period be?

**Options:**
- **A:** 2 releases (v1.12 -> v1.13 -> v2.0) - ~6 months
- **B:** 3 releases (v1.12 -> v1.13 -> v1.14 -> v2.0) - ~9 months
- **C:** 1 year minimum before enforcement

**Recommendation:** Option A (6 months) with clear communication

### 3. Consent Default

**Question:** Should offline_access be pre-checked on consent screen?

**Options:**
- **A:** Pre-checked (easier UX, likely approved)
- **B:** Unchecked (explicit user action required)
- **C:** Pre-checked only if in requested scope

**Recommendation:** Option C (follows requested scope)

### 4. Refresh Token Cleanup

**Question:** Should we actively clean up refresh tokens when offline_access not granted?

**Options:**
- **A:** Keep them but with short expiry (access token lifetime)
- **B:** Delete them immediately
- **C:** Don't generate them at all

**Recommendation:** Option C (don't generate) - cleaner implementation

### 5. Client Allowed Scopes

**Question:** Should we add offline_access to all existing clients automatically?

**Options:**
- **A:** Yes, auto-add during migration
- **B:** No, require manual configuration
- **C:** Add only if client currently uses refresh tokens

**Recommendation:** Option A for simplest migration

## Alternatives Considered

### Alternative 1: Maintain Current Non-Compliant Behavior

**Description:** Accept the non-compliance, document it as intentional deviation

**Pros:**
- No breaking changes
- No migration complexity
- Current integrations continue working

**Cons:**
- Violates OIDC specification
- Cannot achieve enterprise certification
- Poor user control and privacy
- May fail compliance audits

**Decision:** Rejected - Standards compliance is important for long-term viability

### Alternative 2: Immediate Enforcement Without Backward Compatibility

**Description:** Immediately require offline_access, no configuration option

**Pros:**
- Clean, compliant implementation
- No technical debt
- Clear user consent

**Cons:**
- Breaking change for all existing clients
- High support burden
- May cause service disruptions
- Poor user experience during transition

**Decision:** Rejected - Too disruptive, need gradual migration

### Alternative 3: Separate Refresh Token Endpoint

**Description:** Create new endpoint for refresh tokens requiring offline_access

**Pros:**
- Fully backward compatible
- Clear separation of concerns

**Cons:**
- Non-standard approach
- Confusing for developers
- Double implementation
- Doesn't solve the core compliance issue

**Decision:** Rejected - Violates OIDC standards further

### Alternative 4: Prompt-based Consent Instead of Scope

**Description:** Use `prompt=consent` to trigger offline_access instead of scope

**Pros:**
- Different mechanism than scope

**Cons:**
- Violates OIDC specification (offline_access MUST be scope)
- Confusing semantics
- Non-interoperable

**Decision:** Rejected - Not standards-compliant

## Related ADRs

- [ADR-0001: Add Client Credentials and Token Exchange Grant Types](./0001-add-client-credentials-and-token-exchange-grant-types.md) - Should be implemented before or alongside to ensure new grant types respect offline_access from the start

## Dependencies

**Blocks:**
- ADR-0001 implementation (should respect offline_access)
- Enterprise OIDC certification
- Proper user consent management

**Requires:**
- User consent framework (already exists)
- Scope validation system (already exists)
- Admin configuration system (already exists)

## References

- [OpenID Connect Core 1.0 - Section 11: Offline Access](https://openid.net/specs/openid-connect-core-1_0.html#OfflineAccess)
- [OAuth 2.0 RFC 6749 - Section 1.5: Refresh Token](https://datatracker.ietf.org/doc/html/rfc6749#section-1.5)
- [OAuth 2.0 Security Best Current Practice - Refresh Tokens](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-security-topics#section-4.13)
- [OIDC Certification Program](https://openid.net/certification/)

## Date

2025-10-30

## Authors

- Architecture analysis and proposal
