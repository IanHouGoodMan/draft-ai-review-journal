# Topic Relation Graph 草稿

## 目标

定义 AI Reviewer 在 Review 阶段如何表达 topic 之间、资产之间以及 segment 之间的关系。

这份文档关注的不是底层存储选型，而是先稳定语义：
- 哪些节点需要存在；
- 哪些关系需要被显式表达；
- Review 输出中最少要保留哪些关系信息；
- AI Console、Infer、DomainStore 后续如何消费这些关系。

## 为什么需要 Relation Graph

仅仅识别出 `Schema`、`Source`、`Sink`、`SampleData`、`Workflow` 还不够。

原因是用户 draft 往往描述的是一组相互关联的对象，而不是孤立对象。例如：
- 一个 Source 使用哪个 Schema；
- 一个 SampleData 是为哪个 Source 准备的；
- 一个 Workflow 使用哪些 Sources；
- 一个 Sink 输出采用哪个 Schema。

如果 Review 只返回资产类型，不返回关系，那么：
- AI Console 很难正确展示确认信息；
- Infer 难以选择最合适的 prompt 与 tool set；
- DomainStore 很难沉淀长期可检索的资产网络；
- 后续修改与删除时也难以正确定位影响范围。

因此，Review 必须把关系显式化。

## Graph 的范围

当前先覆盖 Review 第一阶段真正需要的关系，不追求一次把所有图语义定义完。

### 当前优先节点类型
- `Draft`
- `Segment`
- `Schema`
- `Source`
- `Sink`
- `SampleData`
- `Workflow`
- `Document`

### 当前优先关系类型
- `Draft -> Segment`
- `Segment -> Asset`
- `Source -> Schema`
- `Sink -> Schema`
- `SampleData -> Schema`
- `Source -> SampleData`
- `Sink -> SampleData`
- `Workflow -> Source`
- `Workflow -> Sink`
- `Document -> Asset`

## 基本建模原则

### 1. Draft 与 Asset 分开
Draft 是输入载体，不是最终资产。

因此应区分：
- 用户本次输入了什么；
- Review 从中识别出了哪些资产与关系。

### 2. Segment 是一等概念
一个 draft 可能被拆成多个 segment，因此很多关系最好通过 segment 作为中间层表达。

例如：
- `Draft d1 contains Segment s1`
- `Segment s1 describes Schema schema_customer`
- `Segment s2 describes Source src_customer_csv`

### 3. 关系允许“推断”与“显式声明”并存
用户可能明确写出：
- `Src1 uses Schema1`

也可能只隐含表达：
- “客户导入源使用客户表结构”

因此关系边应至少能区分：
- `Declared`
- `Inferred`

### 4. 关系需要有置信度与理由
特别是在 Review 阶段，很多关系是通过匹配或推断得出的，应该保留：
- confidence
- reason

### 5. Review 输出不必返回完整图数据库结构
Review 输出只需返回足够支持确认和 Infer 的轻量图信息。真正的长期持久化可由 DomainStore 再做转换。

## 节点定义建议

### Draft Node
建议最少包括：
- `draftInstanceId`
- `createdAt`
- `originalText`

### Segment Node
建议最少包括：
- `segmentId`
- `draftInstanceId`
- `originalText`
- `topicCategory`
- `topicKind`
- `intent`

### Asset Node
资产节点当前可统一抽象为：
- `assetKind`
- `assetName`
- `namespace`
- `isExisting`
- `isInferred`

其中 `assetKind` 目前先包括：
- `Schema`
- `Source`
- `Sink`
- `SampleData`
- `Workflow`
- `Document`

## 关系定义建议

### Draft -> Segment
表示一次 draft 被拆成哪些 segment。

建议关系名：
- `ContainsSegment`

### Segment -> Asset
表示一个 segment 主要描述了哪个资产。

建议关系名：
- `Describes`

如果一个 segment 中含有多个资产，也可以保留：
- `DescribesPrimary`
- `DescribesRelated`

### Source -> Schema
表示某 source 使用某 schema。

建议关系名：
- `UsesSchema`

### Sink -> Schema
表示某 sink 产出或依赖某 schema。

建议关系名：
- `UsesSchema`

### SampleData -> Schema
表示 sample data 对应哪个 schema。

建议关系名：
- `ConformsToSchema`

### Source -> SampleData
表示某 source 的示例数据。

建议关系名：
- `HasSampleData`

### Sink -> SampleData
表示某 sink 的示例数据。

建议关系名：
- `HasSampleData`

### Workflow -> Source
表示 workflow 的输入来源。

建议关系名：
- `ConsumesSource`

### Workflow -> Sink
表示 workflow 的输出目标。

建议关系名：
- `ProducesSink`

### Document -> Asset
表示文档中定义或描述了哪个资产。

建议关系名：
- `Defines`
- `Describes`

## 为什么 SampleData 不能只挂在 Schema 上

这是当前最关键的设计点之一。

同一个 Schema 可能被多个 Source 或 Sink 复用，但不同 Source / Sink 对应的 SampleData 可以不同。

例如：
- `Src1 -> Schema1 -> SampleData1`
- `Src2 -> Schema1 -> SampleData2`

在这里：
- `Schema1` 相同；
- `SampleData1` 与 `SampleData2` 不同；
- 差异来自不同 source 的上下文。

因此至少要允许同时表达：
- `Source -> Schema`
- `Source -> SampleData`
- `SampleData -> Schema`

只保留 `SampleData -> Schema` 会丢失与具体 source/sink 的关联。

## Review 输出中应如何表达关系

Review 不需要直接输出完整图库格式，但建议至少输出一个轻量 `declaredRelations` 集合。

每一条 relation 至少包括：
- `relationType`
- `fromKind`
- `fromName`
- `toKind`
- `toName`
- `isInferred`
- `confidence`
- `reason`

例如：
- `UsesSchema(Source:Src1 -> Schema:Schema1)`
- `HasSampleData(Source:Src1 -> SampleData:SampleData1)`
- `ConformsToSchema(SampleData:SampleData1 -> Schema:Schema1)`
- `ConsumesSource(Workflow:OrderFlow -> Source:Src1)`

## Segment 之间的关系

除了资产之间的关系，还应允许表达 segment 之间的依赖。

例如：
- segment A 创建 schema；
- segment B 创建 sample data，并引用 segment A 的 schema；
- segment C 创建 workflow，并引用 segment B 中提到的 source。

因此建议补充：
- `SegmentDependsOnSegment`

这个关系特别适合：
- 同一 draft 中的多 topic 联合确认；
- Infer 阶段的执行排序；
- 缺失依赖判定。

## 与 AI Console 的关系

AI Console 至少需要借助 relation graph 做三件事：

1. 展示本次 draft 中有哪些资产是成组出现的；
2. 展示哪些关系是推断出来的，需要用户重点确认；
3. 在用户确认时，避免只确认单个孤立资产而忽略它的依赖链。

例如用户看到的确认界面可以表达为：
- 新建 `Schema: CustomerSchema`
- 新建 `Source: CustomerCsvSource`
- 确认 `CustomerCsvSource UsesSchema CustomerSchema`
- 新建 `SampleData: CustomerCsvSample`
- 确认 `CustomerCsvSample ConformsToSchema CustomerSchema`

## 与 Infer 的关系

Infer 需要关系图来缩小生成上下文。

例如：
- 如果 segment 是 `SampleData + Create`，Infer 需要优先加载关联 schema；
- 如果 segment 是 `Workflow + Create`，Infer 需要优先加载相关 source/sink；
- 如果 segment 是 `Schema + Edit`，Infer 需要知道有哪些下游 source/workflow 可能受影响。

因此关系图并不只是给展示用，也是 Infer 的输入裁剪器。

## 与 DomainStore 的关系

DomainStore 可以把 Review 的轻量 relation 结果进一步沉淀为：
- 关系表；
- 图模型；
- 向量与图的混合索引；
- 影响分析查询入口。

但 Review 不必提前承担所有持久化复杂度。

## 一个建议的最小结构

当前可以先把关系输出简化为两层：

### Segment 内关系
用于表达某个 segment 自身声明的资产关系。

### Draft 内依赖关系
用于表达多个 segment 之间的先后与依赖。

这样已经足够支撑：
- AI Console 确认；
- Infer 路由；
- Diary 记录；
- 后续 DomainStore 演进。

## 当前结论

当前建议把 Topic Relation Graph 看作 Review 的必要输出之一，而不是后置补充信息。

至少在第一版中，Review 应保证：
- 能把一个 draft 拆成多个 segment；
- 能识别每个 segment 的主资产；
- 能输出关键资产关系；
- 能表达 segment 间依赖；
- 能区分声明关系与推断关系。

等后续 confirm 交互文档补齐后，再决定是否把 relation graph 进一步抽象成正式 contract。