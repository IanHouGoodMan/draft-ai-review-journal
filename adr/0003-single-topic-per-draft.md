# ADR 0003 - Single topic per draft (first phase)

_Date: 2026-05-03_

## Status

Accepted.

## Context

Earlier drafts of the Review design supported multiple topics, multiple
segments, and a full draft/segment relation graph. This made the Review,
Confirm, and Infer contracts hard to stabilize.

## Decision

A draft describes exactly one object or one operation. Review returns exactly
one `ReviewResult`. Confirm operates on exactly one `ReviewResult`.

## Consequences

- Smaller contracts; easier to express as interfaces and unit tests.
- AI Console renders a single result, not a list.
- Multi-topic drafts are out of scope; if needed later, split upstream or
  revisit this ADR.
