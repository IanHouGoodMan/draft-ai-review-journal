# DSL Review Journal

Public engineering journal for rebuilding the AI Reviewer / Review part of the DSL system.

## Purpose

This repository is for public design notes and development journals.

It is intended to:
- document architecture and design intent,
- provide a public record that can be shared in resumes or interviews,
- reduce repeated explanation during AI-assisted design and implementation.

Sensitive notes, drafts, credentials, and internal-only details should stay in a separate private repository.

## Working model

The system is being reshaped with coarse-grained boundaries.

Current examples include:
- Review
- Infer
- DomainStore
- AI Console
- ... (other boundaries may be added as needed)

Each boundary is intended to be a complete solution/package boundary, instead of splitting Review into many small packages at the start.

## Repository structure

- `daily/` — journal entries
- `design/` — design documents
- `adr/` — architecture decision records

See also:
- `daily/README.md`
- `design/README.md`
- `adr/README.md`

## Initial documents

- `daily/2026-04-29-first-entry.md`
- `design/ai-review-review-stage.md`
- `design/review-output-model.md`
- `design/topic-relation-graph.md`
- `design/confirm-flow.md`
- `adr/0001-use-dsl-review-as-public-engineering-journal-repository.md`
- `adr/0002-adopt-coarse-grained-solution-boundaries.md`
