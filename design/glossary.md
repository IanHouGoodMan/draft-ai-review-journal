# Glossary

_Last updated: 2026-05-05_

Authoritative terms used by AI Reviewer. Defined here only.

## Draft

User input. One draft describes one object or one operation.

## ReviewResult

Output of Review. Fields:

| Field              | Type                       | Notes                                  |
| ------------------ | -------------------------- | -------------------------------------- |
| DraftId            | string                     | Correlates Review, Confirm, Infer.     |
| InputDraft         | string                     | Original user text.                    |
| Topic              | Topic                      | See below.                             |
| Intent             | Intent                     | See below.                             |
| References         | Reference[]                | May be empty.                          |
| Description        | string?                    | Optional natural-language explanation. |
| SuggestedDraft     | string?                    | Optional draft text the user can copy back to resubmit. |
| Issues             | ReviewIssue[]             | Empty when there is nothing to report. |
| ReadyForInference  | bool                       | Gate into Infer.                       |

## Topic

`Category` + optional `DataKind` + `Name`.

- Category: `Data` | `Workflow` | `Unknown`
- DataKind (when Category = Data): `Schema` | `SampleData` | `SourceSink` | `None`

`SourceSink` merges source and sink; both are data endpoints.

## Intent

`CreateNew` | `Modify` | `Delete` | `Unknown`

## Reference

| Field         | Type                                              |
| ------------- | ------------------------------------------------- |
| ReferenceType | `SourceSink` \| `Schema` \| `SampleData` \| `Workflow` |
| Name          | string                                            |
| IsRequired    | bool                                              |
| IsResolved    | bool                                              |
| Description   | string                                            |

## ReviewIssue

User-visible note produced by Review.

| Field       | Type             | Notes                                      |
| ----------- | ---------------- | ------------------------------------------ |
| Severity    | `Info` \| `Error` | `Error` blocks Infer.                      |
| Message     | string           | Reason or warning.                         |
| Missing     | Reference?       | Optional missing reference.                |

## ReadyForInference

`false` if any of:

- Topic is `Unknown`
- Intent is `Unknown`
- `Modify` or `Delete` has unresolved required reference
- `CreateNew` depends on an existing object whose reference is unresolved
- `Workflow` has no resolved `Source` with ready `SampleData`
- Any `ReviewIssue` has severity `Error`

When `false`, `Issues` must state the reason. `Description` may restate it for
the user.
