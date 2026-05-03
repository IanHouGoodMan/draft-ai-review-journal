# Review

_Last updated: 2026-05-03_

Review turns one draft into one `ReviewResult`. Infer consumes the confirmed
`ReviewResult`, never the raw draft.

Terms: see [glossary.md](./glossary.md).

## Data relations

Used to resolve `References`:

- `Source/Sink → Schema`
- `Source/Sink + Schema → SampleData`
- `Schema → Source/Sink*`
- `SampleData → (Source/Sink, Schema)`

A required `Schema` is satisfied by a `Schema` reference, or by a `Source/Sink`
reference whose schema can be resolved through these relations. Same logic for
`SampleData`.

## Workflow input rule

`Workflow` requires source or schema input. If only schema is present, Review
may set `ReadyForInference = true`; Infer fills in a default source.

## Boundary

Infer does not re-decide `Topic`, `Intent`, or required references.

## Example

Draft:

> Create `JumpServerPerCountry` schema, based on `JumpServer`, drop `mgmt_ip`,
> `country` is unique.

If neither `Schema: JumpServer` nor `SourceSink: JumpServer` is resolved:

- Topic = `Data.Schema`, Name = `JumpServerPerCountry`
- Intent = `CreateNew`
- References = [ `Schema: JumpServer` (required, unresolved) ]
- ReadyForInference = `false`
- Description states the missing reference.
