# AI Reviewer 设计草稿：Review 阶段

## 设计目标

Review 阶段负责把用户的自由文本 draft 转换成结构化、可确认、可进入 Infer 的结果。

本文采用如下命名约定：
- **AI Reviewer**：对外能力名称；
- **Review**：AI Reviewer 内部的核心阶段与 solution/package 边界名称。

它的职责不是直接生成最终 DSL，而是先解决下面这些问题：
- 这段 draft 属于什么 topic；
- topic 是否在工具处理范围内；
- 用户意图是新增、修改、删除还是其他；
- draft 是否包含多个 topic；
- 是否缺少依赖；
- 哪些名字、描述、关系需要推断；
- 应该把什么最小上下文交给 Infer。

## 参考旧系统后的继承点

从旧 DSL solution 可以继承以下思想：

- Review 与 Infer 分阶段；
- AI Console 在两者之间承担确认职责；
- Review 结果已经具备 intent、matched target、responsibility、resolved asset kind 等结构；
- 一部分 data 推断可由规则和结构化推断承担，而不必完全依赖 LLM；
- MQ 已经被用于 AI 相关请求/响应传输。

## Review 输入

建议至少包括：

- `draftInstanceId`
- `draftText`
- `namespace` 或当前工作上下文
- 用户当前显式选中的 references
- 用户当前 mode hint（如 create/edit/delete/help）
- 已知资产索引摘要

## Review 输出

建议 Review 输出至少具备以下信息：

### 顶层结果
- `draftInstanceId`
- `outcome`
- `warnings`
- `clarificationQuestions`
- `segments`

### Segment 结果
每个 segment 对应 draft 中一个可独立确认的 topic。

每个 segment 建议至少包含：
- `segmentId`
- `originalText`
- `topicCategory`：Data / Workflow / Other
- `topicKind`：Schema / Source / SampleData / Sink / Workflow / Unknown
- `intent`：Create / Edit / Delete / Other
- `confidence`
- `suggestedName`
- `suggestedBusinessDescription`
- `matchedTargets`
- `detectedReferences`
- `missingDependencies`
- `declaredRelations`
- `topicVectorRef`

## Topic 分类

当前先稳定为三大类：
- `Data`
- `Workflow`
- `Other`

### Data 子类
- `Schema`
- `Source`
- `SampleData`
- `Sink`

### Workflow
- `Workflow`

### Other
不进入 Infer，直接在 Review 阶段收口。

## Intent 分类

当前先稳定为：
- `Create`
- `Edit`
- `Delete`
- `Other`

只有 topic 与 intent 都落在支持范围内，才允许进入确认和后续 Infer。

## 多 Topic 拆分

一个 draft 可能包含多个 topic，甚至多个同类 topic。

因此 Review 需要支持：
- 一个 draft 对应多个 segment；
- 每个 segment 有独立 topic/intention/result；
- AI Console 可以逐个确认或整体确认；
- Infer 可以按 segment 分发。

## 关系建模

对于 data topic，不能只识别资产种类，还要表达关系。

当前最重要的关系包括：
- `Source -> Schema`
- `Sink -> Schema`
- `Source -> SampleData`
- `Sink -> SampleData`
- `SampleData -> Schema`
- `Workflow -> Source`
- `Workflow -> Sink`

原因是：同一个 Schema 可被多个 Source 或 Sink 复用，但不同 Source/Sink 组合下的 SampleData 可能不同。

## 推断策略

Review 可以推断：
- 名字
- 业务描述
- 引用关系
- 缺失依赖

但所有推断结果必须显式返回给 AI Console，供用户确认。

## Review 与 Infer 的边界

Review 阶段解决“做什么”。
Infer 阶段解决“如何生成或修改目标产物”。

因此 Infer 不应重新承担以下职责：
- topic 主分类
- intent 主分类
- 是否属于工具范围
- 大范围依赖判定
- 多 topic 拆分

Infer 只消费确认后的 segment 结果，并选择更小的 prompt 与 tool set。

## Topic Vector 的位置

vector 更适合作为辅助能力，而不是唯一语义事实来源。

可用于：
- 分类辅助
- 相似草稿召回
- 历史设计检索
- 开发日记与设计文档的语义关联

但对外权威结果仍然应是结构化 review output。

## 与开发日记的关系

Review 阶段天然适合输出开发日记事件，例如：
- draft 已创建
- segment 已拆分
- topic 已判定
- intent 已判定
- dependency gate 未通过
- 用户已确认
- 已进入 infer

这些事件应与 `draftInstanceId` 和 trace id 关联，供后续公开/私有日记系统消费。

## 当前结论

Review 应定义为：

> 一个将自由文本 draft 收口为有限、稳定、可确认操作模型的阶段。

它既是 Infer 的前置裁决层，也是 AI Console 的主要确认输入来源。