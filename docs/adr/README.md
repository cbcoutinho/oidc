# Architecture Decision Records (ADRs)

This directory contains Architecture Decision Records (ADRs) for the Nextcloud OIDC Identity Provider.

## What is an ADR?

An Architecture Decision Record (ADR) is a document that captures an important architectural decision made along with its context and consequences. ADRs help teams:

- Understand why decisions were made
- Onboard new team members faster
- Review past decisions
- Avoid repeating past mistakes

## ADR Format

Each ADR follows this structure:

1. **Title**: Short, descriptive title
2. **Status**: Proposed, Accepted, Deprecated, Superseded
3. **Context**: Background and problem description
4. **Decision**: What was decided and why
5. **Consequences**: Expected positive and negative outcomes
6. **Alternatives Considered**: Other options that were rejected

## ADR List

### Active

- [ADR-0001: Add Client Credentials and Token Exchange Grant Types](./0001-add-client-credentials-and-token-exchange-grant-types.md) - Proposed
- [ADR-0002: Implement offline_access Scope Compliance](./0002-implement-offline-access-scope-compliance.md) - Proposed

### Superseded

None

### Deprecated

None

## Creating a New ADR

1. Copy the template or follow the format of existing ADRs
2. Use sequential numbering: `NNNN-title-with-dashes.md`
3. Start with Status: Proposed
4. Update this README with a link to the new ADR
5. Submit for review via pull request

## Status Definitions

- **Proposed**: Under discussion, not yet accepted
- **Accepted**: Decision has been approved and implemented
- **Deprecated**: No longer applicable, kept for historical reference
- **Superseded**: Replaced by another ADR (link to successor)

## Further Reading

- [Documenting Architecture Decisions by Michael Nygard](https://cognitect.com/blog/2011/11/15/documenting-architecture-decisions)
- [ADR GitHub Organization](https://adr.github.io/)
