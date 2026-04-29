# 2026-04-29 第一篇开发日记

## 背景

准备参考 `C:\Users\ian\source\repos\DSL`，重新建立 AI Reviewer / Review 部分。

这次重建不打算把 Review 再拆成许多细小 package，而是采用较粗粒度的 solution/package 边界。当前已明确的例子包括：
- Review
- Infer
- DomainStore
- AI Console

这些并不是完整且固定不变的全部模块列表，后续还可以根据系统演进继续补充或调整。

其中，Review 是一个完整 solution，也是一个清晰的产品边界。它负责把用户输入的 draft 先变成可确认的 review 结果，再交给 AI Console 确认，之后才进入 Infer 阶段。

公开的开发日记单独放在一个 repo 中，用于沉淀设计、记录演进，并作为将来对外展示工程思考的入口。

## 这次回看旧方案后的几个观察

参考旧 solution，可以看到一些已经形成的方向：

1. **Review 与 Infer 已经是两阶段**  
   在旧实现中，AI Console 不是拿用户 draft 直接推理生成结果，而是先经过 Review，再由 Review 的结果决定后续进入哪类 inference。`AiConsoleFlowController` 和 `AiConsoleInferenceExecutionService` 已经体现了这种分层。

2. **Review 结果已经承担“路由”和“缩小上下文”的职责**  
   旧 contracts 中的 `AiConsoleReviewResult` 已有这些信号：
   - `InferredIntent`
   - `OutcomeKind`
   - `SuggestedAssetName`
   - `MatchedTargets`
   - `DetectedReferences`
   - `ResponsibilityReview`
   - `ResolvedAssetKind`

   这说明 Review 本来就不仅是“分类”，而是 Infer 的前置裁决层。

3. **旧系统已经在 data 相关场景中形成了若干资产类型**  
   从旧代码可见，系统已经围绕这些资产做处理：
   - Schema
   - Source
   - Sink
   - SampleData
   - Workflow
   - Document

4. **旧系统已经存在非 AI 的结构化推断能力**  
   例如 `SchemaInferenceService` 可以从 CSV sample data 推断 schema。这说明重建时不应把所有判断都交给大模型，Review/Infer 应该优先组合规则、已有资产、结构化推断与 AI。

5. **旧系统已经有 MQ 通道**  
   `MqMessages` 说明旧系统已经通过 MQ 承载部分 AI 请求/响应。因此新方案里，Review 相关事件、开发日记采集事件与 OTel 的关系，需要从一开始就明确边界。

## AI Reviewer 的第一版目标

Review 的核心任务是：

> 用户输入一段 draft，系统创建一个 draft instance id。随后 Review 把 draft 解析为可确认的 topic 与 intent 结果，交还给 AI Console；只有确认后，才进入 Infer。

这里的关键不是立即生成 DSL，而是先把“用户想处理什么”说清楚。

## 1. Draft Instance

每次用户输入一段 draft，都创建一个 `draft instance id`。

这个 id 至少有几类用途：
- 关联一次 review 与后续 infer；
- 关联用户确认动作；
- 关联 telemetry、日志、事件和开发日记；
- 支持一个长 draft 被拆成多个 topic 实例后，仍能追溯到同一次输入。

旧系统里已经存在 `draftInstanceId` 的概念，并在 Review 与 Infer 间传递。重建时应把它提升成正式的一等概念，而不是零散字符串。

## 2. Topic 识别

Review 首先要判断 draft 在说什么 topic。

当前先收敛为三大类：
- Data
- Workflow
- Other

如果是 `Other`，表示这不是当前工具处理的内容，Review 应该在这一层终止，而不是继续把请求送入 Infer。

### 2.1 Data topic

Data topic 再细分为四类资产：

#### Schema
必填信息：
- 名字
- schema 的业务角度描述
- 列（列名、列数据类型）

可选信息：
- 约束
  - 不可空
  - 唯一
  - 是另一个 schema 某列的外键

#### Source
必填信息：
- 名字

可选信息：
- 数据适配器
- 连接字符串

#### Sample Data
必填信息：
- 名字
- 列
- 数据

#### Sink
必填信息：
- 名字

可选信息：
- 数据适配器
- 连接字符串

### 2.2 Workflow topic

Workflow 当前关注的信息包括：
- 名字（必填）
- 业务描述（必填）
- DSL 算子集合（必填）
- Sources（必填）
- Sink（可选）

### 2.3 Topic vector

这里打算用 vector 表达 topic，但目前更准确的理解应当是：

- vector 是 topic 识别与相似检索的支撑信息；
- 它不是 topic 本身的唯一权威表示；
- 权威结果仍然应是一个结构化 review result。

也就是说，Review 可以为每一个 topic 片段生成一个 vector 表示，用于：
- topic 分类；
- 已有资产/历史 draft 的相似召回；
- 后续 diary 与知识检索。

但系统对外应返回结构化结论，而不是只返回 embedding。

## 3. 一个 draft 可能包含多个 topic

这是这次设计里很重要的一点。

一个用户 draft 可能不是只描述一个对象，而是一次性描述多个 topic，甚至多个同类 topic。比如：
- 同时定义两个 source；
- 一边描述 schema，一边描述 sample data；
- 先描述 data，再描述 workflow。

因此 Review 不应假设“一次输入只有一个目标”。

更合理的设计是：
- 保留一个总的 `draft instance id`；
- 把 draft 拆成多个 `topic segment`；
- 每个 `topic segment` 生成一个独立的 topic vector 和结构化 review 结果；
- AI Console 在确认时按 segment 或按组合结果进行确认。

## 4. Data 资产之间的关系

Source / Sink / Schema / Sample Data 通常不是孤立存在，而是成套关系。

最基本的理解是：
- 一个 `Schema` 可以被多个 `Source` 或 `Sink` 引用；
- 每个 `Source/Sink + Schema` 的组合，可能对应不同的 `Sample Data`；
- 因此 `Sample Data` 不能只简单附着在 `Schema` 上，而应能够表达它与具体 source 或 sink 的配对关系。

例如：
- `Src1` 使用 `Schema1`，对应 `SampleData1`
- `Src2` 使用 `Schema1`，对应 `SampleData2`

这里 `SampleData1` 和 `SampleData2` 不同，即使它们都基于同一个 `Schema1`。

这意味着 Review 与 DomainStore 都需要能表达一种“资产关系图”而非单表孤立记录。至少要能表达：
- Source -> Schema
- Sink -> Schema
- Source -> SampleData
- Sink -> SampleData
- SampleData -> Schema
- Workflow -> Source
- Workflow -> Sink

后续无论落图还是落关系表，Review 在输出时都应保留这种关系信息。

## 5. Intent 识别

Topic 之外，Review 还要判断用户的 intent。

当前 intent 先收敛为：
- 新增
- 修改
- 删除
- 其他

只有当 intent 是针对系统支持的 topic，并且属于新增 / 修改 / 删除时，这个工具才继续处理。

### 5.1 新增的依赖关系

新增不是简单 create。新增时通常会伴随依赖检查，例如：
- 新增 Sample Data，往往依赖 Schema 已存在或本 draft 中可同时建立；
- 新增 Workflow，依赖 Source 已存在或在本次 draft 中声明；
- 新增 Sink 可能依赖目标 Schema；
- 删除或修改则要先匹配现有目标。

旧系统里已经有 `ResponsibilityReview`、`MissingAssets` 和 dependency gate 的痕迹。重建时，这部分可以从“隐含逻辑”提升为 Review 的正式职责。

## 6. 名字与业务描述的推断

用户提供的 draft 不一定明确写出名字，也不一定把业务描述写得结构化。

因此 Review 应支持：
- 推断名字；
- 推断业务描述；
- 把推断结果明确返回给 AI Console；
- 让用户知道系统补全了哪些字段。

这件事的意义不只是方便 Infer，更是为了形成一种可学习的交互：
- 用户知道这次系统怎样理解了 draft；
- 用户知道下次如果想减少歧义，应该怎样补充输入。

旧系统中的 `SuggestedAssetName` 已经说明这个方向是成立的。新方案中应进一步扩展到每个 topic segment 级别。

## 7. Review 与 AI Console、Infer 的关系

Review 的结果不是最终产物，而是一个“待确认的、面向 Infer 的准备结果”。

建议流程保持为：

1. 用户输入 draft；
2. 系统创建 `draft instance id`；
3. Review 执行：
   - topic 分类；
   - intent 分类；
   - topic 拆分；
   - 依赖检查；
   - 名字/描述推断；
   - 关联已有资产与关系；
4. AI Console 向用户展示 review 结果并要求确认；
5. 只有确认后，才进入 Infer；
6. Infer 根据已确认的 topic 和 intent，加载相应 prompt 和 tool calling set。

这样做的一个直接收益是：
- 避免 Infer 阶段 prompt 过多；
- 避免 tool calling set 过大；
- 降低无关工具被带入上下文的概率；
- 让推理更可控，也更容易调试。

这与旧系统的分层方向是一致的，但新方案会把这种边界设计得更明确。

## 8. Review 不只是分类器，而是“收口层”

这次重新思考之后，我更倾向把 Review 定义为一个收口层：

- 它负责把自由文本 draft 收口为有限、稳定、可确认的操作模型；
- 它负责把“不该进入 Infer 的内容”挡在外面；
- 它负责把多 topic 的输入拆分；
- 它负责把依赖、关系和推断结果显式化；
- 它负责为 Infer 提供足够小而精确的上下文。

因此 Review 不是一个简单的分类器，而是 AI Reviewer 中 Review solution 的核心价值所在。

## 9. 当前形成的第一版设计判断

基于旧 solution 与本次草稿，当前先形成以下判断：

1. `Review` 应作为独立 solution / package 边界存在。  
2. Review 与 Infer 必须明确分层，Review 先行，Confirm 在中间。  
3. `draft instance id` 要成为贯穿 review / confirm / infer / telemetry / diary 的统一标识。  
4. Review 需要支持一个 draft 拆分为多个 topic segment。  
5. Topic 当前先稳定为 `Data / Workflow / Other`。  
6. Data topic 当前先稳定为 `Schema / Source / SampleData / Sink`。  
7. Intent 当前先稳定为 `新增 / 修改 / 删除 / 其他`。  
8. 名字、业务描述和引用关系都允许由 Review 推断，但必须显式返回给用户确认。  
9. Source / Sink / Schema / SampleData 应以关系图思维表达，而不是平铺孤立资产。  
10. Infer 应依据 review 后的 topic 与 intent，选择更小的 prompt 和 tool set。

## 10. 当前命名判断

当前更倾向采用下面这组命名：

- **AI Reviewer**：对外的产品能力名、用户可感知的角色名；
- **Review**：内部子系统名、solution/package 边界名、阶段名；
- **Infer**：Review 之后的推理生成阶段；
- **AI Console**：用户确认与进入 Infer 的交互入口。

也就是说，对外可以说“AI Reviewer”，对内架构与代码边界仍然可以保持 `Review` 这个简洁名称。

## 下一步

下一步优先做两件事：

1. 把 Review 阶段的结构化输出模型设计出来；
2. 继续把 topic relation graph 和 confirm 交互补成单独设计文档。
