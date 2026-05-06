# Copilot Instructions

## Project Guidelines
- User prefers coarse-grained modularization: Review, Infer, DomainStore, and AI Console should each be a complete solution/package boundary, rather than further splitting AI Review into many separate packages. This list is not exhaustive and may be manually adjusted by the user; it serves as examples rather than a fixed complete module list.
- User prefers keeping code repositories private.
- Writing interfaces and abstract classes is a more effective way to communicate architecture and intent with AI.
- User prefers the DraftReviewer module to stay transport-agnostic and independent of MQ; message queues should only appear at hosting/integration boundaries such as a worker-hosted reviewer or communication with Designer, which simplifies testing.

## AI Reviewer Guidelines
- For AI Reviewer simplification, assume each draft describes only one topic/item initially, rather than supporting multiple topic segments.
- Review results should be simplified to topic-intent-reference-description-readiness, with unknown or missing required references explained clearly.