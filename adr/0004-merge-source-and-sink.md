# ADR 0004 - Merge source and sink as `SourceSink`

_Date: 2026-05-03_

## Status

Accepted.

## Context

In Review, source and sink are both data endpoints, both reference a schema,
and both can relate to sample data. Distinguishing them at this layer adds
vocabulary without changing the readiness logic.

## Decision

Use a single `DataKind` value `SourceSink`. Direction (source vs sink) is a
detail handled inside DomainStore / Infer, not in `ReviewResult`.

## Consequences

- One reference type instead of two.
- Reference resolution rules stay short.
- If a future stage needs to distinguish direction at the Review layer, this
  ADR is revisited.
