# Glossary

_Last updated: 2026-05-03_

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
| Description        | string                     | Natural-language explanation.          |
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

## ReadyForInference

`false` if any of:

- Topic is `Unknown`
- Intent is `Unknown`
- `Modify` or `Delete` has unresolved required reference
- `CreateNew` depends on an existing object whose reference is unresolved
- `Workflow` has neither source nor schema input

When `false`, `Description` must state the reason.
