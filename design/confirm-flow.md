# Confirm Flow 草稿

## 目标

定义 AI Reviewer 的 Review 阶段结束后，AI Console 如何承接结构化结果、向用户展示、让用户确认，并把确认后的结果交给 Infer。

这份文档关注的是：
- Confirm 在整体流程中的位置；
- 用户需要确认什么；
- 哪些内容可以修改；
- 多 segment 情况下如何确认；
- Confirm 之后如何进入 Infer。

## Confirm 的定位

Confirm 不是可有可无的 UI 步骤，而是 Review 与 Infer 之间的正式边界。

建议整体流程稳定为：

1. 用户输入 draft；
2. 系统创建 `draftInstanceId`；
3. Review 输出结构化结果；
4. AI Console 展示 Confirm 界面；
5. 用户确认、修正或拒绝部分结果；
6. 生成 `ConfirmedReviewResult`；
7. Infer 只消费确认后的结果。

这里的核心原则是：

> Infer 不直接消费原始 draft，而是消费用户确认后的 Review 结果。

## 为什么 Confirm 必须存在

Confirm 至少解决四个问题：

1. **Review 允许推断**  
   名字、业务描述、引用关系、匹配目标都可能是系统推断的，因此必须有显式确认环节。

2. **一个 draft 可能包含多个 segment**  
   用户可能只想继续其中一部分。

3. **依赖关系可能不完整**  
   用户需要看到缺少什么，或者修正引用。

4. **Infer 需要更小、更准的上下文**  
   Confirm 让系统在进入 Infer 前，把结构收口到最明确的状态。

## Confirm 的输入

AI Console 的 Confirm 页面至少应接收：
- `draftInstanceId`
- `ReviewResult`
- 当前上下文中的已选 references
- 当前工作区命名空间或作用域
- 历史确认信息（如果是回到编辑后再次 review）

## Confirm 的输出

建议 Confirm 阶段输出 `ConfirmedReviewResult`。

至少包括：
- `draftInstanceId`
- `confirmedSegments`
- `rejectedSegments`
- `notes`
- `confirmedAt`
- `confirmedBy`

其中每个 confirmed segment 应包括：
- `segmentId`
- `confirmedTopicCategory`
- `confirmedTopicKind`
- `confirmedIntent`
- `confirmedName`
- `confirmedBusinessDescription`
- `confirmedReferences`
- `confirmedRelations`
- `executionEligibility`

## 用户需要确认什么

### 1. Topic
用户需要知道系统把该 segment 识别成了什么：
- Data / Workflow / Other
- Schema / Source / Sink / SampleData / Workflow

### 2. Intent
用户需要知道系统理解为：
- Create
- Edit
- Delete
- Other

### 3. 名字与业务描述
如果这些字段是推断出来的，必须明显标出，并允许用户修改。

### 4. 匹配目标
对于 Edit / Delete，用户需要确认系统匹配到的现有目标是否正确。

### 5. 关系与引用
用户需要看到：
- Source 使用哪个 Schema；
- SampleData 对应哪个 Schema；
- Workflow 依赖哪些 Source / Sink；
- 哪些关系是推断出来的。

### 6. 缺失依赖
如果某个 segment 还缺少依赖，用户需要知道：
- 缺什么；
- 是否可以在本次 draft 内联创建；
- 是否必须先暂停 Infer。

## Confirm 页面建议的操作

每个 segment 至少支持：
- `Confirm`
- `EditBeforeConfirm`
- `Reject`
- `ReturnToDraft`

### `Confirm`
接受当前 segment 的结构化结果。

### `EditBeforeConfirm`
修改系统推断的字段，例如名字、描述、引用关系。

### `Reject`
明确表示该 segment 不应进入 Infer。

### `ReturnToDraft`
回到原 draft 继续编辑，再重新执行 Review。

## 多 Segment 的确认策略

一个 draft 可能包含多个 segment，因此 Confirm 需要支持两种粒度：

### 1. Segment 级确认
逐个 segment 确认。

适合：
- 混合 topic 输入；
- 部分清晰、部分模糊；
- 用户只想继续一部分内容。

### 2. 批量确认
当多个 segment 都足够清晰时，支持一次性确认多个 segment。

但即使支持批量确认，底层仍应保留 segment 级确认结果，避免把细节压平。

## Confirm 与依赖关系

Confirm 阶段不只是显示结果，还应帮助用户理解依赖。

例如：
- `SampleData` 依赖 `Schema`；
- `Workflow` 依赖 `Source`；
- `Edit` / `Delete` 依赖目标匹配成功。

建议对每个 segment 给出一个 `executionEligibility`：
- `Ready`
- `BlockedByDependency`
- `BlockedByClarification`
- `Rejected`

这样 Infer 只处理 `Ready` 的 segment。

## Confirm 与 Topic Relation Graph

Confirm 页面应直接消费 Review 输出中的 relation 信息。

例如可以向用户展示：
- `CustomerCsvSource UsesSchema CustomerSchema`
- `CustomerCsvSample ConformsToSchema CustomerSchema`
- `OrderWorkflow ConsumesSource CustomerCsvSource`

如果某条关系是推断出来的，应明确显示为 inferred，并允许用户修正。

## Confirm 与 Infer 的边界

Confirm 完成后，Infer 不应再做这些事情：
- 重判 topic；
- 重判 intent；
- 重猜名字；
- 重猜主引用关系；
- 在没有用户确认的情况下切换目标资产。

Infer 可以做的是：
- 根据 confirmed topic 与 intent 选择 prompt；
- 根据 confirmed references 加载最小资产上下文；
- 根据 confirmed relations 生成或修改目标产物；
- 返回生成结果供用户继续查看或采纳。

## 一个建议的 Confirm 输出层次

### 顶层
- `draftInstanceId`
- `confirmedSegments`
- `rejectedSegments`
- `overallOutcome`

### Segment 级
- `segmentId`
- `confirmedAssetKind`
- `confirmedIntent`
- `confirmedFields`
- `confirmedReferences`
- `confirmedRelations`
- `executionEligibility`

## Diary / Telemetry 事件

Confirm 阶段建议至少发出这些事件：
- `ReviewConfirmationStarted`
- `SegmentConfirmed`
- `SegmentEditedBeforeConfirm`
- `SegmentRejected`
- `ConfirmationCompleted`
- `ConfirmationReturnedToDraft`
- `InferDispatchPrepared`

这些事件应关联：
- `draftInstanceId`
- `segmentId`
- `traceId`
- `reviewOutcome`
- `confirmedOutcome`

## 当前结论

Confirm 是 AI Console 中承接 Review 与 Infer 的核心交互层。

它的主要职责是：
- 把 Review 的结构化结果展示给用户；
- 让用户确认或修正推断；
- 让多 segment 输入能够部分继续、部分拒绝；
- 为 Infer 生成最小、稳定、已确认的输入。

如果 Review 是“收口层”，那么 Confirm 就是“定稿层”。
