# ADR 0002: Adopt coarse-grained solution boundaries

- Status: Accepted
- Date: 2026-04-29

## Context

The original system may evolve into multiple independent repositories and solutions.

A key concern is reducing repeated cross-cutting changes, especially when design is still evolving and AI-assisted modification is frequent.

Over-splitting AI Reviewer into many small packages too early would increase:
- versioning overhead,
- local debugging cost,
- contract churn,
- cross-repository coordination complexity.

## Decision

Adopt coarse-grained solution and package boundaries.

Current examples include:
- Review
- Infer
- DomainStore
- AI Console

This list is illustrative rather than exhaustive and may be adjusted as the system evolves.

Within this approach:
- each major boundary is a complete solution/package boundary,
- Review is not split into many tiny packages at the current stage,
- finer decomposition can be reconsidered later after boundaries stabilize.

## Consequences

### Positive
- Clearer ownership and lower coordination cost.
- Better control of AI-assisted changes.
- Easier public explanation of architecture.
- Simpler versioning and packaging strategy at the current stage.

### Trade-offs
- Some internal responsibilities remain grouped together for now.
- Certain future refactorings may still be needed after the design stabilizes.

## Follow-up

- Continue documenting the Review boundary first.
- Keep public documentation aligned with this coarse-grained approach.
- Revisit finer-grained packaging only when a boundary shows long-term stability.
