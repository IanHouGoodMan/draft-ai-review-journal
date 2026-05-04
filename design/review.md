# Review

_Last updated: 2026-05-04_

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

`Workflow` requires a `Source` input, not just a `Schema`.

A workflow draft is ready only if Review can resolve the input to a `Source`
whose `SampleData` is also ready. Sample data is required because Infer must
load it to validate the generated workflow.

A `Schema`-only input is not ready: Review must look up sources that use the
schema, pick one, and verify its sample data. If no such source exists, the
result is unresolved.

## Sample data input rule

A `SampleData` draft must be tied to a `Source` or `Sink` endpoint.

If the draft does not name an endpoint, Review infers candidate endpoints from
the sample's columns, sets `ReadyForInference = false`, and writes a
`SuggestedDraft` that names the best candidate. The user copies the suggestion,
edits it if needed, and resubmits.

The same pattern applies to any `Data` topic that needs an endpoint it did not
name: propose, do not auto-pick.

## Boundary

Infer does not re-decide `Topic`, `Intent`, or required references. Review
never silently substitutes a missing reference; it either resolves it through
the data relations above or returns `ReadyForInference = false` with a reason
and, when possible, a `SuggestedDraft`.

## Examples

### Schema based on existing schema

Draft:

> Create `JumpServerPerCountry` schema, based on `JumpServer`, drop `mgmt_ip`,
> `country` is unique.

If neither `Schema: JumpServer` nor `SourceSink: JumpServer` is resolved:

- Topic = `Data.Schema`, Name = `JumpServerPerCountry`
- Intent = `CreateNew`
- References = [ `Schema: JumpServer` (required, unresolved) ]
- ReadyForInference = `false`
- Description states the missing reference.

### Workflow with only a schema mentioned

Draft:

> Create a workflow that counts JumpServer rows per country, based on
> `JumpServer` schema.

Review looks up sources that use `JumpServer` schema:

- If exactly one source is found and its sample data is ready:
  - References = [ `Source: <found>` (required, resolved), `SampleData: <found>` (required, resolved) ]
  - ReadyForInference = `true`
- Otherwise:
  - References include the unresolved `Source` (and `SampleData` if the source has none)
  - ReadyForInference = `false`
  - Description explains which step failed (no source / multiple sources / source has no sample data)
  - `SuggestedDraft` names the most likely source

### Sample data without an endpoint

Draft:

> Add this sample: `host=fw-01, mgmt_ip=10.0.0.1, country=SG`.

Review infers from the columns that the likely endpoint is `JumpServer`:

- Topic = `Data.SampleData`
- Intent = `CreateNew`
- References = [ `SourceSink: JumpServer` (required, unresolved) ]
- ReadyForInference = `false`
- SuggestedDraft = `Add sample to JumpServer source: host=fw-01, mgmt_ip=10.0.0.1, country=SG.`
- Description explains the inference and asks the user to confirm.

