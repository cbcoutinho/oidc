# ADR-0001: Add Client Credentials and Token Exchange Grant Types

## Status

Proposed

## Context

The Nextcloud OIDC Identity Provider currently supports three OAuth 2.0 grant types:

1. **Authorization Code** (`authorization_code`) - Standard user authorization flow
2. **Refresh Token** (`refresh_token`) - Token refresh mechanism
3. **Implicit** (`implicit`) - Implicit flow (via response_type, not as token endpoint grant)

### Current Limitations

**Machine-to-Machine Authentication:**
There is no mechanism for service-to-service authentication where no user is present. Applications requiring backend-to-backend API access must either:
- Use service accounts with user credentials (security anti-pattern)
- Share user tokens between services (violates least-privilege principle)
- Implement custom authentication outside OAuth 2.0 standards

**Token Delegation and Down-scoping:**
There is no standardized way to:
- Exchange tokens between services with reduced privileges
- Implement delegation scenarios (service acting on behalf of user)
- Convert token types or audiences for different resource servers
- Create time-limited tokens with subset of original permissions

### Industry Standards

**RFC 6749 Section 4.4 (Client Credentials Grant):**
Widely adopted for machine-to-machine authentication. Supported by major identity providers including Auth0, Okta, Azure AD, Keycloak, and Google Cloud Identity.

**RFC 8693 (OAuth 2.0 Token Exchange):**
Emerging standard for token delegation and transformation. Adopted by Google Cloud IAM, AWS STS (Security Token Service), Azure AD, and Keycloak. Critical for microservices architectures and zero-trust security models.

### Use Cases

**Client Credentials:**
- Automated backup services accessing Nextcloud APIs
- Monitoring systems checking system health
- CI/CD pipelines deploying applications
- Scheduled batch jobs processing data
- Webhook receivers calling APIs
- MCP (Model Context Protocol) servers requiring persistent API access

**Token Exchange:**
- API gateway down-scoping tokens for backend services
- Microservice chains requiring propagated identity
- Cross-tenant federation scenarios
- Time-limited elevated privilege scenarios
- Token format conversion (JWT to opaque for legacy systems)

## Decision

We will implement both OAuth 2.0 grant types in two phases:

### Phase 1: Client Credentials Grant Type (RFC 6749 §4.4)

Add support for `grant_type=client_credentials` to enable machine-to-machine authentication without user context.

**Key Design Decisions:**

1. **Token Endpoint Extension:**
   - Accept `client_credentials` as valid grant_type in `OIDCApiController::getToken()`
   - Require confidential clients only (reject public clients)
   - Support optional `scope` parameter with validation against `Client.allowedScopes`

2. **Data Model Changes:**
   - Make `AccessToken.userId` nullable to support user-less tokens
   - Database migration: `ALTER TABLE oc_oidc_access_tokens MODIFY COLUMN user_id VARCHAR(255) NULL`

3. **JWT Generation:**
   - Set `sub` claim to client_id when userId is null
   - Omit user-specific claims (name, email, groups)
   - Include client-specific claims and granted scopes
   - Set appropriate `aud` and `azp` claims

4. **Response Format:**
   - Return `access_token`, `token_type`, `expires_in`, `scope`
   - Omit `refresh_token` (per RFC 6749 recommendation)
   - Omit `id_token` (no user identity)

5. **Security Controls:**
   - Require client authentication (client_id + client_secret)
   - Reject public clients (no client_secret)
   - Implement rate limiting on token endpoint
   - Scope restrictions: disallow user-related scopes (profile, email, etc.)
   - Skip user group validation (no user context)

### Phase 2: Token Exchange Grant Type (RFC 8693)

Add support for `grant_type=urn:ietf:params:oauth:grant-type:token-exchange` to enable token delegation, down-scoping, and transformation.

**Key Design Decisions:**

1. **Token Endpoint Extension:**
   - Accept RFC 8693 token exchange grant type
   - Support required parameters: `subject_token`, `subject_token_type`
   - Support optional parameters: `audience`, `resource`, `scope`, `requested_token_type`

2. **New Components:**
   - **TokenParser Service:** Parse and validate JWT and opaque tokens
   - **ExchangePolicy Service:** Implement authorization policies for token exchange
   - **ActorChainBuilder Service:** Build delegation chains with `act` claims
   - **TokenExchangeLog Entity:** Audit logging for exchanges

3. **Database Schema:**
   ```sql
   CREATE TABLE oc_oidc_token_exchanges (
       id BIGINT PRIMARY KEY AUTO_INCREMENT,
       source_token_id BIGINT,
       derived_token_id BIGINT,
       client_id BIGINT,
       subject_user_id VARCHAR(255),
       actor_client_id VARCHAR(255),
       requested_audience VARCHAR(500),
       original_scopes TEXT,
       granted_scopes TEXT,
       exchange_timestamp BIGINT,
       INDEX idx_source_token (source_token_id),
       INDEX idx_derived_token (derived_token_id)
   );
   ```

4. **Token Type Support:**
   - `urn:ietf:params:oauth:token-type:access_token`
   - `urn:ietf:params:oauth:token-type:refresh_token`
   - `urn:ietf:params:oauth:token-type:id_token`
   - `urn:ietf:params:oauth:token-type:jwt`

5. **Security Controls:**
   - **Privilege Escalation Prevention:** New token scopes MUST be ≤ original scopes
   - **Authorization Policies:** Explicit policies defining which clients can exchange tokens
   - **Delegation Limits:** Maximum chain depth (default: 5 levels)
   - **Audit Logging:** Comprehensive logging of all exchanges (success/failure)
   - **Revocation Propagation:** Revoking source token revokes derived tokens
   - **Impersonation Controls:** Whitelist of allowed impersonation pairs

6. **Policy Framework:**
   - Database-driven policy configuration
   - Per-client exchange permissions
   - Configurable delegation rules
   - Audience/resource restrictions

### Discovery Metadata Updates

Update `DiscoveryGenerator.php` to advertise new capabilities:

```php
$grantTypesSupported = [
    'authorization_code',
    'implicit',
    'refresh_token',
    'client_credentials',
    'urn:ietf:params:oauth:grant-type:token-exchange',
];

// For token exchange
$tokenTypesSupported = [
    'urn:ietf:params:oauth:token-type:access_token',
    'urn:ietf:params:oauth:token-type:refresh_token',
    'urn:ietf:params:oauth:token-type:id_token',
    'urn:ietf:params:oauth:token-type:jwt',
];
```

## Consequences

### Positive Consequences

**Client Credentials:**
- Enables secure machine-to-machine authentication
- Eliminates need for service accounts with user credentials
- Aligns with OAuth 2.0 best practices
- Simplifies automation and integration scenarios
- Low implementation complexity and risk
- Improves security posture by removing shared credentials

**Token Exchange:**
- Enables microservices architectures with proper token management
- Supports zero-trust security models with scoped access
- Facilitates cross-service delegation with full audit trail
- Allows token down-scoping for least-privilege access
- Enables multi-tenant and federation scenarios
- Provides standardized token transformation

**Both:**
- Increases compatibility with enterprise identity management systems
- Positions Nextcloud as comprehensive OAuth 2.0/OIDC provider
- Reduces custom authentication implementations
- Improves security through standardization

### Negative Consequences

**Client Credentials:**
- Database schema change requires migration
- Breaking change if applications depend on userId being non-null
- Additional testing required for null user scenarios
- Potential confusion about when to use client_credentials vs. authorization_code

**Token Exchange:**
- Significant implementation complexity (1-2 weeks effort)
- High security risk if policies not properly configured
- Requires careful policy design and review
- Additional database table for audit logs
- Complex revocation logic with token chains
- Performance impact of token validation and policy evaluation
- Requires comprehensive security testing
- Documentation complexity for policy configuration

**Both:**
- Increases attack surface (new endpoints/grant types)
- Requires additional monitoring and alerting
- More complex token validation in resource servers
- Potential for misconfiguration leading to security issues

### Risks and Mitigations

**Risk: Client credentials abuse**
- Mitigation: Rate limiting, scope restrictions, audit logging, client expiration

**Risk: Token exchange privilege escalation**
- Mitigation: Strict scope validation, policy enforcement, comprehensive audit logs

**Risk: Delegation chain exploitation**
- Mitigation: Maximum chain depth, time-bounded tokens, revocation propagation

**Risk: Performance degradation**
- Mitigation: Caching of policy decisions, efficient token lookup, database indexing

**Risk: Complex policy misconfiguration**
- Mitigation: Secure defaults (deny-all), policy validation, administrative UI, clear documentation

## Implementation Plan

### Phase 1: Client Credentials (2-4 days)

**Step 1: Data Model Changes**
- [ ] Make `AccessToken.userId` nullable in entity
- [ ] Create database migration for nullable user_id column
- [ ] Update `AccessTokenMapper` to handle null userId
- [ ] Add unit tests for null userId scenarios

**Step 2: Token Endpoint Updates**
- [ ] Update `OIDCApiController::getToken()` grant type validation
- [ ] Add client_credentials flow logic
- [ ] Implement scope parameter handling and validation
- [ ] Skip user-related logic (group checks, user lookups) when userId is null
- [ ] Update response to omit refresh_token and id_token

**Step 3: JWT Generation**
- [ ] Update `JwtGenerator::generateAccessToken()` for client-only tokens
- [ ] Set `sub` claim to client_id when userId is null
- [ ] Omit user claims (name, email, groups)
- [ ] Add client-specific claims

**Step 4: Discovery and Documentation**
- [ ] Update `DiscoveryGenerator.php` grant_types_supported
- [ ] Add client_credentials documentation
- [ ] Document scope recommendations

**Step 5: Testing**
- [ ] Unit tests for client_credentials flow
- [ ] Integration tests for token endpoint
- [ ] Security tests (public client rejection, scope validation)
- [ ] Manual testing with various clients

**Step 6: Security Hardening**
- [ ] Implement rate limiting
- [ ] Add audit logging
- [ ] Configure scope restrictions
- [ ] Security review

### Phase 2: Token Exchange (1-2 weeks)

**Step 1: Foundation Services**
- [ ] Create `TokenParser` service for JWT and opaque token parsing
- [ ] Implement token signature validation
- [ ] Add token expiration checks
- [ ] Unit tests for token parsing

**Step 2: Policy Framework**
- [ ] Design policy data model
- [ ] Create `ExchangePolicy` service
- [ ] Implement policy evaluation engine
- [ ] Add default deny-all policy
- [ ] Create policy configuration API
- [ ] Unit tests for policy engine

**Step 3: Database Schema**
- [ ] Create `oc_oidc_token_exchanges` table migration
- [ ] Create `TokenExchangeLog` entity
- [ ] Create `TokenExchangeLogMapper`
- [ ] Add indexes for performance

**Step 4: Token Endpoint Extension**
- [ ] Add token exchange grant type handling
- [ ] Implement subject_token validation
- [ ] Add scope down-scoping logic
- [ ] Implement audience/resource validation
- [ ] Generate new tokens with proper claims

**Step 5: Delegation Support**
- [ ] Create `ActorChainBuilder` service
- [ ] Implement `act` claim generation
- [ ] Add delegation depth validation
- [ ] Support `may_act` claim

**Step 6: Revocation**
- [ ] Implement revocation propagation
- [ ] Add cascade delete for derived tokens
- [ ] Update introspection endpoint

**Step 7: Discovery and Documentation**
- [ ] Update `DiscoveryGenerator.php` with token exchange support
- [ ] Document policy configuration
- [ ] Provide delegation examples
- [ ] Security best practices guide

**Step 8: Testing**
- [ ] Unit tests for all new services
- [ ] Integration tests for token exchange flows
- [ ] Security tests (privilege escalation prevention)
- [ ] Policy enforcement tests
- [ ] Delegation chain tests
- [ ] Performance tests
- [ ] Manual security review

## Open Questions

### Client Credentials
1. **Refresh Token Support:** Should we support refresh tokens for client_credentials? (RFC says optional, typically NO)
   - **Recommendation:** NO - clients can re-authenticate with credentials
2. **Default Scopes:** What default scopes if none requested?
   - **Recommendation:** Empty scopes (explicit scope required)
3. **Grant Type Restrictions:** Add per-client allowed grant_types field?
   - **Recommendation:** YES - add in future iteration for flexibility

### Token Exchange
1. **Delegation Patterns:** Support impersonation, delegation, or both?
   - **Recommendation:** Start with delegation only, add impersonation later
2. **Maximum Chain Depth:** What limit?
   - **Recommendation:** 5 levels (configurable)
3. **Policy Storage:** Database or config file?
   - **Recommendation:** Database for runtime changes, with API
4. **Revocation Strategy:** Cascade or independent?
   - **Recommendation:** Cascade by default, with override flag
5. **Actor Tokens:** Support 3-legged exchange (actor parameter)?
   - **Recommendation:** Phase 3 enhancement

## Alternatives Considered

### Alternative 1: Client Credentials Only
**Decision:** Rejected
**Rationale:** Solves M2M but not delegation/down-scoping. Both grant types provide complementary capabilities.

### Alternative 2: Custom Nextcloud-specific Grant Type
**Decision:** Rejected
**Rationale:** Non-standard approach reduces interoperability. Standards-based approach preferred.

### Alternative 3: Service Account with Standard Authorization Code
**Decision:** Rejected
**Rationale:** Security anti-pattern. Service accounts create audit/security issues. Client credentials is the correct solution.

### Alternative 4: Implement Token Exchange Only
**Decision:** Rejected
**Rationale:** Token exchange has higher complexity and risk. Client credentials provides foundation and immediate value.

### Alternative 5: Use API Keys Instead
**Decision:** Rejected
**Rationale:** API keys are not OAuth 2.0 standard, don't support scopes/expiration properly, and create separate authentication mechanism.

## References

- [RFC 6749: The OAuth 2.0 Authorization Framework](https://datatracker.ietf.org/doc/html/rfc6749)
- [RFC 6749 Section 4.4: Client Credentials Grant](https://datatracker.ietf.org/doc/html/rfc6749#section-4.4)
- [RFC 8693: OAuth 2.0 Token Exchange](https://datatracker.ietf.org/doc/html/rfc8693)
- [RFC 7523: JSON Web Token (JWT) Profile for OAuth 2.0](https://datatracker.ietf.org/doc/html/rfc7523)
- [OAuth 2.0 Security Best Current Practice](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-security-topics)

## Related ADRs

None (this is the first ADR)

## Date

2025-10-30

## Authors

- Architecture analysis and proposal
