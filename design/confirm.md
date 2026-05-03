# Confirm

_Last updated: 2026-05-03_

Confirm is the gate between Review and Infer. Infer consumes the confirmed
`ReviewResult`, not the draft.

Terms: see [glossary.md](./glossary.md).

## Actions

| Action            | Precondition                | Effect                                 |
| ----------------- | --------------------------- | -------------------------------------- |
| ConfirmAndInfer   | `ReadyForInference = true`  | Hand off `ReviewResult` to Infer.      |
| EditReferences    | —                           | Edit `References`; recompute readiness.|
| ReturnToDraft     | —                           | Re-edit draft and re-Review.           |
| Cancel            | —                           | End without Infer.                     |

## Telemetry events

All events carry `DraftId`, `Topic`, `Intent`, `ReadyForInference`, `TraceId`.

- `ReviewConfirmationStarted`
- `ReviewConfirmed`
- `ReferencesEdited`
- `ReturnedToDraft`
- `ReviewCancelled`
- `InferDispatchPrepared`
