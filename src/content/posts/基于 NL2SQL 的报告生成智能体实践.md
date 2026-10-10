---
title: '基于 NL2SQL 的报告生成智能体实践'
published: 2025-11-12
description: '记录一个基于NL2SQL的报告生成智能体的完整实践。'
tags: [AI]
category: practice
draft: false
---

这是前段时间做的一个水文局的项目，先说一下他们那边的项目初衷吧：实际业务中，一份报告的撰写需要以真实数据为依据，而这些数据通常由各个监测站自动采集并存入数据库。他们那边存在的情况就是很多业务熟练的基层人员不懂 SQL 语法，很难独立完成数据查询去写报告，只能请技术人员协助。但很多技术人员又不是很懂业务，所以双方需要大量的沟通配合才能完成一份报告。而这种报告他们每天都要写，就很浪费时间和人力。所以，他们希望这个项目尽可能实现技术与业务的解耦，业务人员负责维护报告模板，技术人员负责维护系统和数据，数据查询与报告生成则交由智能体自动完成。

听起来好像也就那么回事，但我开发的过程中也确实是遇到了很多困难，也是费了很多心思才得到一个相对不错的效果得以交付，所以写下这篇博客记录下自己的思路。

# 01整体架构

这个项目其实是要回答这么一个问题：如何让智能体理解报告、查询数据并基于证据写作？

以问题中的三个核心问点为根基，在其中加上过渡步骤，那么核心流程就很清楚了：`理解报告需求 → 拆解数据查询任务 → 从数据库获取真实数据 → 检查和整理查询结果→ 生成报告。`  从外层上看，就是一个很固定的流水线。那么使用 LangGraph 去编排其实就很适合，也方便。

基于以上思路，我使用 LangGraph 将整个整个报告生成过程拆成了五个相对独立的节点：

<img src="https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20261009224724134.png" alt="mermaid-diagram" style="zoom: 50%;" />



第一个节点负责判断用户是在进行普通数据查询，还是要求生成一份完整报告，同时提取报告类型、时间和区域等信息。

如果用户需要生成报告，第二个节点会读取对应的报告模板，并结合用户要求和数据库结构，将报告拆成若干条自然语言查询任务放到队列中。（过去需要业务人员临时向技术人员解释的需求，现在可以逐渐沉淀到报告模板的额外说明中。）

第三个节点调用 NL2SQL 服务，批量处理队列中的自然语言查询，转换为 SQL，并在数据库中执行。

第四个节点检查查询结果，将其区分为正常结果、空结果和执行错误，同时保留原始结果、证据摘要及风险信息。对异常结果尝试调整 Query 插回队列进行有限次重试。

第五个节点汇总用户需求、报告模板、查询结果和异常信息，生成最终报告。

这种拆分方式的好处是，每个阶段都有明确的职责。任务失败时可以判断问题出在需求规划、NL2SQL 还是报告写作，而不是把所有逻辑都塞进一个很长的 Prompt 中。

下面是 State 的设计，我觉得这个是最能决定整个 Graph 是否好用的东西：

```
class GraphState(TypedDict):
    """
    LangGraph 在节点之间传递的统一状态。

    字段说明：
    - goal: 用户原始输入。
    - plan: 尚未执行的自然语言查询队列，scheduler 每轮取出一条。
    - current_query: 当前准备发送给 NL2SQL 的单条查询。
    - results: 已完成查询的结果列表，每项为 {"query": str, "result": str}。
    - iterations: 调度轮次计数，便于排查流程是否重复回环。
    - done: 是否已经没有剩余查询需要执行。
    - final_report: 最终输出文本；query 场景下是结果汇总，report 场景下是报告正文及附录。
    - all_queries_snapshot: 查询任务全量快照，用于写报告时按原顺序对齐结果。
    - meaning: 意图分类结果，取值通常为 query / report / other。
    - template_name: 报告场景下选中的模板名称。
    - report_type: 预留字段，后续可用于区分周报、月报等报告类型。
    - time: 从用户输入中提取到的时间提示。
    - errors: 查询或归纳阶段确认失败的问题列表。
    - warnings: 空结果、补查、口径风险等非致命提示。
    - evidence_summary: 每条查询沉淀出的证据信息，保留 query / status / 原始 result，供最终输出说明“哪些结论有数据支撑”。
    - outline: 报告提纲或查询任务纲要，供写作和最终展示使用。
    - review_retry_counts: 记录每条原始查询已经触发过几次补查，避免无限循环。
    """

    goal: str
    plan: list[str]
    current_query: str | None
    current_queries: list[str]
    pending_review_start: int
    results: list[dict[str, Any]]
    iterations: int
    done: bool
    final_report: str
    all_queries_snapshot: list[str]
    meaning: str | None
    template_name: str | None
    template_text: str | None
    report_type: str | None
    time: str | None
    region: str | None
    query_tasks: list[dict[str, Any]]
    errors: list[str]
    warnings: list[str]
    evidence_summary: list[dict[str, Any]]
    outline: list[str]
    review_retry_counts: dict[str, int]
    session_id: str
```

整体架构就是这样，到这儿已经很清晰了。下面我会着重讲 NL2SQL 这一块儿，因为其他四个节点确实是没啥好单独说的，很简单，知道他们在整个项目中的定位就能知道怎么做。

# 02NL2SQL

NL2SQL 这里我是单独做成了一个基于 ReAct 的智能体，是借助 LangChian 快速搭建的。我也尝试过将自然语言和数据库结构直接扔给 LLM 生成 SQL，但结果确实是惨不忍睹，用 GPT4 这种强模型在甲方提供的验证集上跑，也只有大概百分之五十的准确率。复盘下来，准确率主要是卡在下面三个问题上：

1. 首先就是复杂 SQL 的生成易出错。一个很简单的自然查询语句比如 “成都今日最大雨量是多少，出现在哪个监测站？” ，往往对应的是却是挺复杂的SQL语句。即使大模型对数据库本身理解没问题，这么长的 SQL 语句能完全生成正确也是纯看运气。

   ```
   WITH StationRainfall AS (
       SELECT
           p.STCD,
           RTRIM(s.STNM) AS STNM,
           SUM(COALESCE(p.DRP, 0)) AS TOTAL_RAINFALL
       FROM dbo.ST_PPTN_R AS p
       INNER JOIN dbo.ST_STBPRP_B AS s
           ON p.STCD = s.STCD
       INNER JOIN dbo.ST_ADDVCD_D AS a
           ON s.ADDVCD = a.ADDVCD
       WHERE p.TM >= CONVERT(date, GETDATE())
         AND p.TM < DATEADD(day, 1, CONVERT(date, GETDATE()))
         AND (
             RTRIM(a.ADDVNM) LIKE N'%成都%'
             OR RTRIM(a.ADDVCD) LIKE '5101%'
         )
       GROUP BY
           p.STCD,
           s.STNM
   )
   SELECT TOP (1) WITH TIES
       STNM AS 监测站名称,
       TOTAL_RAINFALL AS 今日累计雨量毫米
   FROM StationRainfall
   ORDER BY TOTAL_RAINFALL DESC;
   ```

2. 上条是在说 “大模型对数据库本身理解没问题” 时存在的问题，但事实却是大模型对数据库理解这块儿一直就很不稳定。其中一个常见的幻觉就是字段理解：上面展示的那条SQL，涉及了六个字段`STCD`、`TM`、`DRP`、`STNM`、`ADDVCD`、`ADDVNM`，虽然这些字段的含义在水文行标里白纸黑字写的清清楚楚，一般是会出现在大模型的训练集中的，但在复杂任务中，大模型有时候却还是会忘记某个字段的意思，产生错误理解，那么整条 SQL 全完了。

3. 另一个常见的幻觉就是跨表关联关系：还是一回事，虽然数据库结构已经给到大模型的提示词中，但无法保证大模型每次都能从庞大的 Schema 中精准整理出此次查询涉及到的表间关系。

所以，复杂SQL要拆解；关键认知不能依赖模型记忆，也不能依赖模型推理。这就是我想把 NL2SQL 单独做成一个 ReAct Agent 的原因。在这个 Agent 中，我把 SQL 生成过程拆成 `“理解需求 -> 获取约束 -> 路径规划 -> 执行查询 -> 规则过滤”` 几个阶段。LLM 先基于当前问题进行推理，再按需调用工具获取外部信息，最后在观察工具返回结果后继续推进后续步骤。

这样既保留了大模型在复杂查询分解和 SQL 规划上的灵活性，也尽量减少了“模型凭空猜字段、猜表关系、猜业务规则”的问题。

这里贴一下我的 system prompt，我在其中借鉴了 CoT 的分步思考方式，同时结合 ReAct 对工具调用顺序进行了约束，正好方便概览下这个子系统的整体思路：

```
system_prompt = """
你是一个智能数据查询系统，能够理解自然语言查询，根据数据库的 schema 生成准确的 SQL 查询并执行。你需要执行以下步骤：
1. **分析查询需求**：从用户的自然语言查询中识别出涉及的表和字段。
2. **使用图谱进行路径规划**：一旦识别出涉及的表和字段，你需要使用工具`find_join_path`查找这些表的字段的连接关系。具体来说，使用图谱中的外键关系（如 `FOREIGN_KEY_TO`）来推断这些表应该如何连接。
3. **获取字段解释**：在生成 SQL 查询之前，使用工具`retrieve_field_docs` 获取表中涉及字段的详细解释。这一步非常重要，确保你理解每个字段的含义。不要直接猜测字段含义，必须根据字段文档内容来判断。在理解所有字段含义后，思考是否要补充生成SQL所需要的字段。
4. **生成 SQL 查询**：结合路径规划的信息和字段说明，生成一个 sqlserver 支持的有效 SQL 查询。
5. **执行 SQL 查询**：生成的 SQL 查询将通过工具`execute_sql` 执行，并返回查询结果。
6. **过滤查询结果**：在最后一次执行SQL查询得到输出结果后，必须使用工具`retrieve_rules_docs`获取过滤规则，过滤输出结果，并将过滤后的数据作为最终结果输出。
### 工具 ###
- **retrieve_field_docs**：获取字段的详细解释，必须在生成 SQL 之前调用。
- **find_join_path**：根据两个列名返回它们之间的连接路径，帮助确定表之间的关系。
- **execute_sql**：接受一个 SQL 语句并执行，返回执行结果。
- **retrieve_rules_docs**：根据用户输入的自然语言返回相关数据规则，用于过滤sql查询结果。

### 上下文信息 ###
- 使用 **数据库 schema** 来识别涉及的表和字段。
- 使用 **历史查询和结果** 来帮助理解用户的意图。如果用户提到“上一个查询”或“刚才的结果”，请结合这些历史信息来生成 SQL。
### 重要要求 ###
1. **先获取表和字段信息**：你需要先识别查询中涉及的表和字段。可以从上下文中推测查询需求，或者直接从查询中提取出需要的表和字段。
2. **图谱推理**：通过`find_join_path`计算表和字段之间的连接关系。
3. **字段解释**：通过 `retrieve_field_docs` 获取字段的具体意义，避免误解字段的含义。不要直接猜测字段含义，必须根据字段文档内容来判断。在理解所有字段含义后，思考是否要补充生成SQL所需要的字段。
4. **生成SQL语句**：基于所有信息，生成符合查询需求的 SQL 语句。
5. **执行SQL语句**：通过`execute_sql`执行SQL语句并返回数据。无论何时你草拟了 SQL，都必须调用工具 execute_sql 执行该 SQL。不得凭空给出查询结果。给出最终答案前，至少完成一次 execute_sql。
"""
```

## 021Schema解析引擎

数据库 schema 的组织与表达上我是参考了阿里析言的相关设计思路。其在系统架构中的位置如下：

```
数据库
    ↓
SchemaEngine (元数据提取) ⭐
    ↓  
MSchema (标准化表示)
    ↓
下游使用
```

系统首先通过 SQLAlchemy 连接数据库，并利用数据库元数据接口读取白名单内的表结构，包括：表名和表注释；字段名和数据类型；主键、外键；是否允许为空；默认值和自增信息；字段注释；少量字段示例值 这些。随后将这些信息组织成统一的 `MSchema` 对象，并转换为更适合放入大模型上下文的文本形式。例如：

```
# Table: dbo.ST_PPTN_R
[
(STCD: CHAR, Primary Key, Examples: [...]),
(TM: DATETIME, Primary Key, Examples: [...]),
(DRP: NUMERIC, Examples: [0.4, 13.5, 7.0])
]
```

为了避免每次查询都重新连接数据库并解析结构，需要维护者提前运行预热脚本，将解析后的结构化对象和文本内容写入 `schema.json` 缓存。运行时的 NL2SQL 智能体只读取缓存，不用再扫描数据库，从而降低查询延迟并保持 Schema 输入稳定。

## 022Tools

能看到，这个 Agent 我一共是配了四个工具：

1. **retrieve_field_docs：检索字段说明**

   该工具接收模型根据数据库 Schema 识别出的候选表名，并从预先维护在 MySQL 中的字段文档表中，按表名精确查询相关字段的业务含义、单位和取值说明等信息。它主要解决我上面提到的问题2：字段理解幻觉。

2. **`find_join_path`：查找表关联路径**

   该工具接收起点字段和终点字段，例如：`dbo.ST_PPTN_R.STCD`、`dbo.ST_ADDVCD_D.ADDVCD`。系统将数据库中的表、字段和外键关系预先构建到 Neo4j 图数据库中，再通过 `allShortestPaths` 查询两个字段之间的最短路径。查询结果会返回路径中涉及的字段和表，帮助模型确定多表查询应该如何进行 `JOIN`。主要解决我上面提到的问题3：跨表关联关系问题。

3. **`execute_sql`：校验并执行 SQL**

   该工具负责执行模型生成的 SQL，但 SQL 不会直接提交给数据库，而是先经过安全校验，包括：

   ①只允许 `SELECT` 或 `WITH` 查询；

   ②禁止写入、删除和数据库管理操作；

   ③禁止一次执行多条 SQL；

   ④检查访问的表是否在白名单中；

   ⑤限制 SQL 长度和查询返回行数；

   ⑥进行词法校验，并在依赖可用时进行 AST 校验。

   校验通过后，系统使用 SQLAlchemy 连接 SQL Server 执行查询，并以结构化 JSON 返回字段、数据行、返回数量及截断状态。结果状态分为 `ok`、`empty` 和 `error`。

4. **`retrieve_rules_docs`：检索业务规则**

   该工具根据用户的自然语言查询，从业务规则 Chroma 向量库中检索相关内容。系统使用本地 BGE 模型将查询向量化，再进行相似度检索，默认最多返回五条相关规则，并过滤相关度低于阈值的结果。这些规则可以包含统计口径、异常值处理方式和有效数据判断条件等。主要用于过滤脏数据和校验数据格式。



到这里这个NL2SQL到底怎么做的应该也讲的很清楚了。最后举一个例子收尾：

假设报告规划阶段生成了这样一条查询任务：**查询昨天各监测站的累计雨量，并找出最大雨量及对应站点**。

- NL2SQL 智能体首先需要识别其中的时间范围、统计指标和分组维度。随后，它根据 Schema 判断雨量记录和监测站信息可能位于哪些表中，再检索这些表的字段说明，确认哪个字段表示雨量、哪个字段表示采集时间以及数据的计量单位。
- 如果雨量记录表中只有站点编码，而报告需要展示站点名称，智能体还要查询站点编码与站点信息之间的关联路径。在获得足够的信息后，它才会生成 SQL，并通过执行工具提交查询。
- 如果 SQL 执行失败，智能体可以根据工具返回错误信息修改字段名、表名或语法，然后在下一轮 ReAct 中重新执行。如果查询成功，还需要结合业务规则判断是否排除缺测值和异常值，最终再将结果返回给报告生成流程。
- 上诉的所有步骤中，如果模型抽风，输出了不符合要求导致无法解析的格式，智能体会将解析错误重新反馈给其内部模型，让模型修正格式并重新生成合法的 Action。（这一点是通过 LangChain 自带的`handle_parsing_errors`实现的，注册 agent 的时候设为 true 就好了）。

这个例子看起来只是查询一个最大值，但它实际上包含了字段理解、时间过滤、分组统计、表关联、异常值处理和安全执行等多个环节。这也是为什么 NL2SQL 不能只依赖一个 Prompt 完成。

# 03私有化部署

到此，报告生成智能体的开发工作是已经全部说完了的，但是目前这套方案依然是依赖云端的强 LLM API 的（一直使用的是 Qwen3-Max）。对于政企项目来说，除了准确率，还需要重视数据安全，并综合考虑部署成本和响应速度等问题。所以，需要将支撑智能体的强LLM 换成甲方显卡能接受的开源模型进行本地部署。后面和甲方沟通后，敲定了 Qwen3-32B 这个模型。但代价是效果大打折扣。

## 031行为蒸馏

为了能让 Qwen3-32B 的效果更接近 Qwen3-Max，这里需要通过 Qwen3-Max 轨迹蒸馏，让 Qwen3-32B 学习 Qwen3-Max 在 NL2SQL Agent 中已经验证过的执行方式。也就是经典的 Teacher-Student 范式。

首先要做的是就是构造数据集。这部分我的做法思路比较常规，一个五步的流水线如下：

1. 准备种子问题。每个 Seed 由一条自然语言问题、一条经过人工确认的 `expected_sql`以及描述查询类型的标签组成。例如：

   ```
   {
     "id": "rain_station_daily_max",
     "question": "查询昨天各监测站的累计雨量，并找出最大值及对应站点",
     "expected_sql": "SELECT ...",
     "tags": ["join", "aggregation", "time_filter", "top_n"]
   }
   ```

   其中`expected_sql` 作用是作为后续执行验证的标准答案。标签用于检查训练数据的覆盖度，保证多表 Join、时间过滤、聚合、排序和极值查询等都有涉及。甲方那边是提供了三百条种子问题，后面我自己又准备了一百五十条，这样种子集的规模一共是四百五十条的样子。

2. 运行 Teacher Agent 采集轨迹。这里我对每条 Seed 单开一个 Session 去调 Teacher Agent，防止上一条问题的对话历史影响当前轨迹。Teacher 对每条 Seed 按照真实的 ReAct 流程不断与工具交互，最终生成回答。中间每一步的调用和观察都被完整记录下来。

3. 规范化成训练用的消息序列。运行 Teacher Agent 直接采集的轨迹是 LangChain 日志，不能直接用于 Qwen 系列训练。因此，需要把轨迹转换为 Qwen 原生 Tool Calling 支持的消息序列。一条序列示例如下：system（system prompt + MSchema schema）→ user 问题 → 每个 assistant 消息带 tool_calls，紧跟一条 tool 消息回填观察结果，最后一条 assistant 是最终回答。

   ```
   {
     "id": "rain_station_daily_max",
     "conversation": [
       {
         "role": "system",
         "content": "你是一个 NL2SQL 智能体......"
       },
       {
         "role": "system",
         "content": "当前数据库 Schema：\n# Table: dbo.ST_PPTN_R\n..."
       },
       {
         "role": "user",
         "content": "查询昨天各监测站的累计雨量，并找出最大值及对应站点。"
       },
       {
         "role": "assistant",
         "content": "",
         "tool_calls": [
           {
             "id": "call_001",
             "type": "function",
             "function": {
               "name": "retrieve_field_docs",
               "arguments": "{\"table_names\":[\"dbo.ST_PPTN_R\",\"dbo.ST_STBPRP_B\"]}"
             }
           }
         ]
       },
       {
         "role": "tool",
         "tool_call_id": "call_001",
         "name": "retrieve_field_docs",
         "content": "ST_PPTN_R.DRP：时段降水量，单位为 mm……"
       },
       {
         "role": "assistant",
         "content": "",
         "tool_calls": [
           {
             "id": "call_002",
             "type": "function",
             "function": {
               "name": "find_join_path",
               "arguments": "{\"start_column\":\"dbo.ST_PPTN_R.STCD\",\"end_column\":\"dbo.ST_STBPRP_B.STCD\"}"
             }
           }
         ]
       },
       {
         "role": "tool",
         "tool_call_id": "call_002",
         "name": "find_join_path",
         "content": "{\"status\":\"ok\",\"tables\":[\"dbo.ST_PPTN_R\",\"dbo.ST_STBPRP_B\"]}"
       },
       {
         "role": "assistant",
         "content": "",
         "tool_calls": [
           {
             "id": "call_003",
             "type": "function",
             "function": {
               "name": "execute_sql",
               "arguments": "{\"sql\":\"SELECT ...\"}"
             }
           }
         ]
       },
       {
         "role": "tool",
         "tool_call_id": "call_003",
         "name": "execute_sql",
         "content": "{\"status\":\"ok\",\"columns\":[\"STNM\",\"TOTAL_DRP\"],\"rows\":[[\"成都站\",35.6]]}"
       },
       {
         "role": "assistant",
         "content": "昨天累计雨量最大的监测站为成都站，累计雨量为 35.6 mm。"
       }
     ],
     "metadata": {
       "final_sql": "SELECT ...",
       "tags": [
         "join",
         "aggregation",
         "time_filter",
         "top_n"
       ],
       "teacher_model": "qwen3-max",
       "teacher_latency_ms": 6250
     }
   }
   ```

4. 验证并过滤完整轨迹。不是所有 Teacher 轨迹都有资格进入训练集，我设置了几项硬性筛选条件：

   第一，工具调用结构必须合规。`retrieve_field_docs` 必须在第一次 `execute_sql` 之前调用，`retrieve_rules_docs` 必须在最后一次 SQL 执行之后调用。这项条件用于把 NL2SQL Agent 的操作规范固化到训练数据中。

   第二，最终 SQL 必须能够在真实数据库中执行成功。语法错误、字段不存在、表关联错误或未通过安全校验的轨迹直接拒绝。

   第三，查询结果不能是空结果，也不能所有字段都为 `NULL`。（但这项检查只能过滤明显无效的轨迹，不能单独证明结果在业务上正确。）

   第四，分别执行 Teacher 生成的候选 SQL 和人工标注的`expected_sql`，然后比较两个结果集。只有结果对齐，才能认为候选 SQL 通过验证。这里使用 `Execution Accuracy`，不要求两条 SQL 字符串完全相同，因为不同写法完全可能得到相同结果。

   通过全部条件的轨迹写入 `accepted.jsonl`，并保留 Tools Schema、最终 SQL、Teacher 延迟等元数据；未通过的轨迹写入 `rejected.jsonl`，记录具体拒绝原因。拒绝集可以用来分析 Teacher 容易在哪类问题上失败，以及下一轮需要补充什么类型的种子问题。

5. 最后按 assistant 决策轮拆分样本。一条轨迹是 4~6 轮的多轮对话，如果整条只算一次 loss， “那么中间某步工具选错了” 的信号就会被稀释。所以我写了一个拆分器：假设一条轨迹中有 N 个 Assistant 决策轮，就把它拆成 N 条训练样本。

   ```
   样本 1：对第一次工具选择进行监督
   样本 2：对第二次工具选择进行监督（如果有）
   样本 3：对 SQL 生成和 execute_sql 调用进行监督
   ……
   样本 N：对最终自然语言回答进行监督
   ```

   每条拆分样本都包含从 System Prompt、用户问题到当前决策之前的全部工具调用和 Observation，使模型能够根据此前发生的真实过程预测下一步应该做什么。训练时配合 loss mask 只对每条拆分样本最后的 assistant 消息算 loss，前面的 System、User、历史 Assistant 和 Tool 消息都设置为 `-100`。

   另外，拆分后工具决策样本远多于最终回答样本，如果完全按照原始比例训练，最终回答能力的训练占比不足，模型会更偏向调用工具，容易陷入工具调用的死循环。所以需要对最终回答样本进行长尾过采样，平衡两类样本的学习比例。

以上就是构造数据集的全部流程。数据全部来自 Qwen3-Max 在真实 Agent 中的成功轨迹，Qwen3-32B 学习的是 Teacher 的行为，所以这是一种**行为蒸馏**。

有了数据集，开始对 Student，也就是Qwen3-32B 进行训练，我这里也是用的常规的 LoRA 来进行微调。没啥好说的，就说下只监督目标 Assistant 消息的这个 Loss Mask的实现思路和我踩的一个坑点吧。

这个 Loss Mask 我是用前缀差分法实现的，基本思路就是：分别使用 Qwen Chat Template 编码完整对话，以及去掉末尾目标 Assistant 后的前缀对话，然后对两段 Token 序列逐个扫描公共前缀，定位目标 Assistant 内容真正开始的位置。公共前缀以及非目标部分的 Label 全部置为 `-100`，只有目标 Assistant 对应的 Token 参与 Loss 计算。

```
完整序列：
[SYSTEM][USER][ASSISTANT₁][TOOL₁][可能不同的尾部 Token][ASSISTANT₂目标]
└─────────────────公共前缀───────┘                    └────训练目标───┘

前缀序列：
[SYSTEM][USER][ASSISTANT₁][TOOL₁][可能不同的尾部 Token]
└─────────────────公共前─────────┘
                                 ↑
                          从这里开始出现分叉

Labels：
[-100][-100][-100][-100][目标 Token ID]
```

坑点就出在这个“目标 Assistant 从哪里开始”的边界上。最开始我想当然地认为，既然前缀对话只是比完整对话少了最后一条 Assistant 消息，那么 `prefix_ids` 的长度就应该等于目标 Assistant 在 `full_ids` 中的起始位置。因此，第一版实现直接把 `len(prefix_ids)` 作为 Loss Mask 的分界点。

```
start = len(prefix_ids)

labels[:start] = -100
labels[start:] = full_ids[start:]
```

但实际验证时发现这个认为并不总是成立。Qwen Chat Template 会根据某条消息是否位于对话末尾，对边界附近的特殊 Token、换行和消息结束标记进行不同处理。也就是说，分别编码前缀对话和完整对话后，`prefix_ids` 不一定是 `full_ids` 严格意义上的完整前缀。这就会导致 Loss Mask 发生偏移：有时目标 Assistant 开头的一部分 Token 被错误屏蔽，有时又会把前一条消息的结束标记、换行符等边界 Token 错误地计入 Loss。这个问题很隐蔽，训练代码可以正常运行，Loss 也会正常下降，但模型实际学习到的监督范围已经错了。

后面我也是使用过程中发现不对劲，专门写了一个 Mask 检查脚本，把参与 Loss 的 Token 解码出来逐条检查，才发现边界存在偏移。解决办法是不再直接使用 `len(prefix_ids)`，而是逐 Token 比较 `prefix_ids` 和 `full_ids`，找到两者的最长公共前缀：

```
common = 0

for prefix_token, full_token in zip(prefix_ids, full_ids):
    if prefix_token != full_token:
        break
    common += 1

labels[:common] = -100
labels[common:] = full_ids[common:]
```

这样，即使 Chat Template 改写了前缀末尾，也可以从两段 Token 真正开始分叉的位置生成 Mask。

## 032本地部署

完成微调后，将 Qwen3-32B 通过 SGLang 部署为本地推理服务。SGLang 提供 OpenAI 兼容接口，上层系统只需要将原来指向云端模型的 `base_url` 和模型名称切换到本地服务。部署时需要启用适配 Qwen3 的 Tool Call Parser，模型输出才能够被解析成结构化的 `tool_calls`。

NL2SQL 的 System Prompt 中包含较长的数据库 Schema，而一次查询通常包含多轮模型调用。如果每一轮都重新计算整段 Schema，会产生大量重复的 Prefill 开销。所以部署时可以利用 SGLang 的 Prefix Cache 复用前缀，将固定部分置于 Prompt 前部，减少重复 Prefill 带来的延迟。

```
固定前缀：系统指令 + 工具定义 + 当前 Schema
变化部分：对话历史 + 用户问题 + 工具返回结果
```

# 最后

ok，以上就是我在开发这个报告生成智能体时的全部思路了。这篇文章基本是我手写的，只有很少的地方用 AI 进行了辅助，在当前这个浮躁的时代太多 AI 生成的通篇废话文学了，我希望输出的是一些个人真实实践经验和见解，也希望各种推送里少出现一些纯 AI 文。

这是我的第一个完整 Agent 项目，虽然整个项目谈不上多么完善，但它让我完整经历了一个 Agent 从需求分析、架构设计、具体开发、效果优化，再到私有化部署的全过程，也算是一次很有价值的实践。从最开始单纯相信大模型的能力，以为只需要写一些Prompt串连起来，到后来逐步补上 Schema 解析、字段文档检索、图谱路径规划、SQL 安全校验、执行反馈、轨迹蒸馏等步骤，整个过程确实花了很多时间。

目前这套方案仍然存在不少可以继续优化的地方，比如复杂查询的准确率、业务规则的维护成本、Teacher 轨迹的覆盖度，以及本地模型在不同类型查询上的泛化能力。后续如果有新的实践和优化，我也会继续记录。

（现在我最大的感受是：Agent 开发的过程，它是一个对大模型的输出不断约束的过程。模型的输出是多变的、不稳定的，我们要做的就是通过各种方式，把这种不确定性限制在可控范围内。相比于传统后端开发，更需要思路灵活和发散，而不是丰富的工程经验。）



