# Review

_Last updated: 2026-05-05_

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

`Schema` may have optional constraints such as unique, not-null, and foreign key.

## Workflow input rule

`Workflow` requires a `Source` input, not just a `Schema`.

A workflow draft is ready only if Review can resolve the input to a `Source`
whose `SampleData` is also ready. Sample data is required because Infer must
load it to validate the generated workflow.

A `Schema`-only input is not ready. Review may look up sources that use the
schema and propose one in `SuggestedDraft`, but it must not silently replace the
missing source. The proposed source must also have ready sample data.

## Sample data input rule

A `SampleData` draft must be tied to a `Source` or `Sink` endpoint.

If the draft does not name an endpoint, Review infers candidate endpoints from
the sample's columns, sets `ReadyForInference = false`, and writes a
`SuggestedDraft` that names the best candidate. The user copies the suggestion,
edits it if needed, and resubmits.

The same pattern applies to any `Data` topic that needs an endpoint it did not
name: propose, do not auto-pick.

## Source/Sink creation rule

A `SourceSink` must reference a `Schema`.

If a source/sink draft omits the endpoint name but clearly references one
schema, Review may infer the endpoint name from the schema and emit a
`SuggestedDraft`. This is ready only when the schema reference is resolved and
the remaining endpoint fields have safe defaults.

## Boundary

Infer does not re-decide `Topic`, `Intent`, or required references. Review
never silently substitutes a missing reference; it either resolves it through
the data relations above or returns `ReadyForInference = false` with a reason
and, when possible, a `SuggestedDraft`.

`Issues` carry missing references and warnings. Any `Error` issue blocks Infer;
`Info` issues do not.

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
- Issues state the missing reference.

### Workflow with only a schema mentioned

Draft:

> Create a workflow that counts JumpServer rows per country, based on
> `JumpServer` schema.

Review looks up sources that use `JumpServer` schema:

- References include an unresolved required `Source`
- ReadyForInference = `false`
- Issues explain that workflow input must be a source with ready sample data
- `SuggestedDraft` may name the most likely source when one can be inferred

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

## Acceptance cases

### 1. Create schema from nothing

Precondition: no data, no workflow.

Draft:

> Create `JumpServer` with columns `country`, `jump_host`, `mgmt_ip`,
> `mgmt_fqdn`. `jump_host`, `mgmt_ip`, and `mgmt_fqdn` are unique.

Expected:

- Topic = `Data.Schema`, Name = `JumpServer`
- Intent = `CreateNew`
- References = []
- SuggestedDraft = null
- ReadyForInference = `true`
- Issues = []

### 7. Create workflow from source with ready sample data

Precondition: `Schema: JumpServer`, `SourceSink: JumpServer`, and ready
`SampleData: JumpServer` exist.

Draft:

> Create a workflow that counts reachable jump servers by country from
> `JumpServer` source.

Expected:

- Topic = `Workflow`
- Intent = `CreateNew`
- References = [ `SourceSink: JumpServer` (required, resolved), `SampleData: JumpServer` (required, resolved) ]
- SuggestedDraft = null
- ReadyForInference = `true`
- Issues = []

### 8. Create workflow from source without ready sample data

Precondition: `Schema: JumpServer` and `SourceSink: JumpServer` exist; sample
data is missing or not ready.

Draft:

> Create a workflow that counts jump servers by country from `JumpServer`
> source.

Expected:

- Topic = `Workflow`
- Intent = `CreateNew`
- References = [ `SourceSink: JumpServer` (required, resolved), `SampleData: JumpServer` (required, unresolved) ]
- SuggestedDraft = null
- ReadyForInference = `false`
- Issues include: workflow input source has no ready sample data.

### 9. Create workflow from schema only

Precondition: `Schema: JumpServer`, `SourceSink: JumpServer`, and ready
`SampleData: JumpServer` exist.

Draft:

> Create a workflow that counts jump servers by country from `JumpServer`
> schema.

Expected:

- Topic = `Workflow`
- Intent = `CreateNew`
- References = [ `Schema: JumpServer` (required, resolved), `SourceSink: JumpServer` (required, unresolved) ]
- SuggestedDraft = `Create a workflow that counts jump servers by country from JumpServer source.`
- ReadyForInference = `false`
- Issues include: workflow input must name a source; schema alone is not enough.

### 10. Modify missing target

Precondition: no `Schema: JumpServerPerCountry` exists.

Draft:

> Modify `JumpServerPerCountry`, make `country` unique.

Expected:

- Topic = `Data.Schema`, Name = `JumpServerPerCountry`
- Intent = `Modify`
- References = [ `Schema: JumpServerPerCountry` (required, unresolved) ]
- SuggestedDraft = null
- ReadyForInference = `false`
- Issues include: modify requires an existing target reference.

### 11. Delete missing target

Precondition: no `SourceSink: OldJumpServer` exists.

Draft:

> Delete `OldJumpServer` source.

Expected:

- Topic = `Data.SourceSink`, Name = `OldJumpServer`
- Intent = `Delete`
- References = [ `SourceSink: OldJumpServer` (required, unresolved) ]
- SuggestedDraft = null
- ReadyForInference = `false`
- Issues include: delete requires an existing target reference.

### 12. Unknown topic

Precondition: any state.

Draft:

> Make it better.

Expected:

- Topic = `Unknown`
- Intent = `Unknown`
- References = []
- SuggestedDraft = null
- ReadyForInference = `false`
- Issues include: Review cannot identify the target topic or operation.

### 2. Add schema description

Precondition: `Schema: JumpServer` exists; its description is empty.

Draft:

> `JumpServer` stores jump servers per country. When an RFS host is in DMZ,
> connect through the matching country's jump server instead of connecting
> directly by SSH.

Expected:

- Topic = `Data.Schema`, Name = `JumpServer`
- Intent = `Modify`
- References = [ `Schema: JumpServer` (required, resolved) ]
- SuggestedDraft = null
- ReadyForInference = `true`
- Issues = []

### 3. Add sample data without source

Precondition: `Schema: JumpServer` exists; no `SourceSink: JumpServer` exists.

Draft:

> `country,jump_host,mgmt_ip,mgmt_fqdn`  
> `UK,UKJ01,10.23.40.11,uk01.j.com`  
> `US,US01,10.23.41.11,us01.j.com`

Expected:

- Topic = `Data.SampleData`
- Intent = `CreateNew`
- References = []
- SuggestedDraft = null
- ReadyForInference = `false`
- Issues include: `Schema: JumpServer` matches the columns, but no source/sink
  exists; create a source/sink before adding sample data.

### 4. Create source without schema or name

Precondition: no schema.

Draft:

> Create source.

Expected:

- Topic = `Data.SourceSink`
- Intent = `CreateNew`
- References = []
- ReadyForInference = `false`
- Issues include: missing source name and missing schema reference.

### 5. Create default source from schema

Precondition: `Schema: JumpServer` exists.

Draft:

> Create source using `JumpServer`.

Expected:

- Topic = `Data.SourceSink`, Name = `JumpServer`
- Intent = `CreateNew`
- References = [ `Schema: JumpServer` (required, resolved) ]
- SuggestedDraft = `Create source named JumpServer using JumpServer schema, default file adapter, empty connection string.`
- ReadyForInference = `true`
- Issues include an `Info`: source name was inferred from schema.

### 6. Create named source from schema

Precondition: `Schema: JumpServer` and `SourceSink: JumpServer` exist.

Draft:

> Create source named `ReachableJumpServer` using `JumpServer`. It contains only
> reachable jump servers. Unreachable jump servers caused by environment changes
> or failures are excluded.

Expected:

- Topic = `Data.SourceSink`, Name = `ReachableJumpServer`
- Intent = `CreateNew`
- References = [ `Schema: JumpServer` (required, resolved) ]
- Description preserves the reachable/unreachable business meaning.
- SuggestedDraft = null
- ReadyForInference = `true`
- Issues = []

