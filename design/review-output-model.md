# Review 输出模型草稿

## 目标

定义 AI Reviewer 的 Review 阶段应该输出什么结构，供：
- AI Console 展示与确认；
- Infer 加载更小的 prompt 与 tool set；
- DomainStore 记录关系与索引；
- Telemetry / Diary 记录过程事件。

这个模型的重点不是“实现类怎么命名”，而是先稳定语义边界。

## 设计原则

1. **一个 draft，对应一个顶层结果**  
   顶层结果负责汇总一次 review 的整体状态。

2. **一个 draft 可拆成多个 segment**  
   每个 segment 对应一个相对独立的 topic。

3. **推断结果必须显式化**  
   名字、描述、依赖、关系、匹配目标都不能只藏在内部推理过程中。

4. **结构化结果优先于向量结果**  
   vector 是辅助信息，不能代替权威输出。

5. **确认友好**  
   输出必须适合在 AI Console 中逐项展示与确认。

## 顶层输出

建议顶层模型暂定为 `ReviewResult`，包含：

- `draftInstanceId`
- `outcome`
- `warnings`
- `clarificationQuestions`
- `segments`
- `reviewSummary`
- `traceContext`

### 字段说明

#### `draftInstanceId`
贯穿一次 draft 的统一标识。

#### `outcome`
整体 outcome，建议先收敛为：
- `ReadyForConfirmation`
- `NeedsClarification`
- `OutOfScope`
- `CannotProceed`

#### `warnings`
不阻塞流程，但需要展示给用户的信息。

#### `clarificationQuestions`
需要用户补充的信息。

#### `segments`
本次 draft 拆分后的 topic 列表。

#### `reviewSummary`
面向 AI Console 的简短摘要，帮助用户快速理解系统如何看待本次输入。

#### `traceContext`
与 telemetry / diary / MQ 事件关联的追踪信息。

## Segment 输出

建议每个 segment 暂定为 `ReviewSegmentResult`。

每个 segment 至少包括：
- `segmentId`
- `originalText`
- `normalizedText`
- `topicCategory`
- `topicKind`
- `intent`
- `outcome`
- `confidence`
- `suggestedName`
- `suggestedBusinessDescription`
- `matchedTargets`
- `detectedReferences`
- `missingDependencies`
- `declaredRelations`
- `inferredFields`
- `topicVectorRef`

## Segment 字段说明

### `segmentId`
segment 自身标识，用于确认、事件、日志与 Infer 分发。

### `originalText`
用户原始输入中属于该 segment 的片段。

### `normalizedText`
清洗后的版本，便于后续分类、检索和比对。

### `topicCategory`
大类：
- `Data`
- `Workflow`
- `Other`

### `topicKind`
更具体的种类：
- `Schema`
- `Source`
- `SampleData`
- `Sink`
- `Workflow`
- `Unknown`

### `intent`
当前先收敛为：
- `Create`
- `Edit`
- `Delete`
- `Other`

### `outcome`
segment 级结果，建议先收敛为：
- `ReadyForConfirmation`
- `NeedsClarification`
- `DependencyMissing`
- `OutOfScope`
- `CannotProceed`

### `confidence`
当前分类与识别置信度。

### `suggestedName`
如果用户未明确给出名字，可由 Review 推断。

### `suggestedBusinessDescription`
对业务意图的结构化表达。

### `matchedTargets`
当 intent 为 edit/delete 时，列出可能匹配的已有目标。

### `detectedReferences`
Review 从 draft 或当前上下文中识别出的引用对象。

### `missingDependencies`
当前无法直接进入 Infer 的缺失依赖。

### `declaredRelations`
当前 segment 显式或隐式声明的资产关系。

### `inferredFields`
记录哪些字段是系统推断出来的，便于 AI Console 直观展示。

### `topicVectorRef`
segment 对应的向量引用或检索键，而不是 embedding 本体。

## Supporting Models

### `MatchedTarget`
建议包括：
- `assetType`
- `assetName`
- `namespace`
- `confidence`
- `reason`

### `DetectedReferences`
建议包括：
- `schemas`
- `sources`
- `sinks`
- `sampleData`
- `workflows`
- `documents`

### `MissingDependency`
建议包括：
- `dependencyType`
- `dependencyName`
- `reason`
- `canBeCreatedInline`

### `DeclaredRelation`
建议包括：
- `relationType`
- `fromKind`
- `fromName`
- `toKind`
- `toName`
- `isInferred`

### `InferredField`
建议包括：
- `fieldName`
- `value`
- `reason`
- `confidence`

### `TraceContext`
建议包括：
- `traceId`
- `spanId`
- `correlationId`

## 为什么要保留 Segment 级 outcome

因为一个 draft 可以部分成功。

例如：
- segment A 是一个清晰的 Schema create；
- segment B 是一个模糊的 Workflow edit；
- segment C 是工具不支持的 Other。

如果只有顶层 outcome，就很难表达这种混合状态。保留 segment 级 outcome 后，AI Console 可以：
- 确认可继续的部分；
- 提示需澄清的部分；
- 过滤超出范围的部分。

## 与旧模型的关系

旧 DSL 中的 `AiConsoleReviewResult` 已经具备这些方向：
- intent
- outcome
- suggested name
- matched targets
- references
- responsibility
- resolved asset kind

新模型不是推翻旧思路，而是把它进一步结构化，特别是：
- 引入多 segment；
- 显式表达推断字段；
- 显式表达依赖；
- 显式表达关系；
- 为 diary / telemetry 保留 trace context。

## AI Console 如何使用这个模型

AI Console 至少需要展示：
- 本次 draft 被拆成了几个 segment；
- 每个 segment 的 topic 与 intent；
- 哪些名字或描述是推断出来的；
- 哪些依赖缺失；
- 哪些 segment 可以继续进入 Infer；
- 用户确认后将加载哪类 Infer prompt/tool set。

## Infer 如何消费这个模型

Infer 不再重新解释原始 draft，而是消费确认后的 segment：
- 对 `Schema + Create`，加载 schema create prompt/tool set；
- 对 `SampleData + Create`，加载 sample data prompt/tool set；
- 对 `Workflow + Edit`，加载 workflow edit prompt/tool set。

这样可以显著减少 Infer 的上下文噪音。

## 当前建议

当前可先把 Review 输出模型稳定在以下层次：
- 顶层 `ReviewResult`
- segment 级 `ReviewSegmentResult`
- 配套 supporting models

等确认交互和 relation graph 设计补齐后，再决定最终 contracts 命名与字段是否精简。