# ADR 0001: Use DSL-Review as the public engineering journal repository

- Status: Accepted
- Date: 2026-04-29

## Context

The rebuilding effort needs a public place to record:
- development journals,
- design intent,
- architecture decisions,
- system evolution.

This public record should be easy to share in resumes or interviews, while sensitive drafts and internal-only notes remain elsewhere.

## Decision

Use the current `DSL-Review` workspace as the public engineering journal repository.

Keep code repositories and the public journal repository separate.

Sensitive notes, raw drafts, credentials, and internal collaboration context stay in a separate private repository.

## Consequences

### Positive
- Public design history becomes easy to share.
- The repository stays focused and readable.
- Review-related design can evolve without coupling directly to implementation repos.
- AI-assisted collaboration can refer to stable public documents.

### Trade-offs
- Code and journal history are split across repositories.
- Some decisions will need both a public summary and a private deeper note.

## Follow-up

- Keep `daily/`, `design/`, and `adr/` as the core structure.
- Add new entries as the Review, Infer, DomainStore, and AI Console boundaries evolve.
