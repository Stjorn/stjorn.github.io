---
title: 'mem0学习笔记'
published: 2026-01-10
description: 'mem0可以说是做Agent长期记忆的一个很经典的框架了，所以这次特地找了个周末好好学了一下这个框架。'
tags: [AI]
category: note
draft: false
---

mem0可以说是做Agent长期记忆的一个很经典的框架了，所以这次特地找了个周末好好学了一下这个框架。

# mem0 是什么？它由哪些组件组成？

如果剥去花哨的外衣，Mem0 本质上是一个面向 LLM 长期记忆的 Harness工程。

它自己不生产算力和存储，而是定义了一套“记忆提取 -> 冲突检测 -> 自动更新”的标准化 Workflow，并通过工厂模式提供了极强的可插拔性。这种设计的巧妙之处在于：它把‘如何存向量’这种工程问题交给了专业的数据库，把‘如何理解对话’交给了专业的大模型，而 Mem0 自己只专注于扮演一个**‘总调度师’**和**‘数据清洗工’**的角色。正是这种 Harness 架构，使得 Mem0 既轻量，又具备了极高的上限。"

要理解它的全貌，我们必须先看看它是由哪些部件拼装起来的，以及它吐出来的数据到底长什么样。

## 1. 六大核心组件：`MemoryConfig`

在 `configs/base.py` 中，定义了整个框架最重要的配置类 `MemoryConfig`。当调用 `Memory.from_config()` 启动 `Mem0` 时，其实是在往里面装配这 6 个核心组件：

```python
class MemoryConfig(BaseModel):
    vector_store: VectorStoreConfig = ...    # 1. 向量库：负责存向量并做相似度检索
    llm: LlmConfig = ...                     # 2. 大模型：负责“思考”（提取事实、判断记忆冲突）
    embedder: EmbedderConfig = ...           # 3. 嵌入模型：负责把文字翻译成向量
    history_db_path: str = ...               # 4. 历史账本：本地 SQLite，记录记忆变更流水
    graph_store: GraphStoreConfig = ...      # 5. (可选) 图数据库：存实体关系（如 张三 -[喜欢]-> 咖啡）
    reranker: Optional[RerankerConfig] = ... # 6. (可选) 重排器：搜索完后，用大模型给结果重新打分排序
```

Mem0 自己并不生产 LLM，也不造向量数据库。它只是定义了这 6 个坑位。无论我们是想用 OpenAI 还是本地部署的 Qwen，想用轻量级的 Chroma 还是云端的 Qdrant，只需要在 Config 里换个名字，Mem0 就能立刻跑起来。它是一个极其标准的**“插拔式架构”**。

例如：

```
config = MemoryConfig(
        llm={
            "provider": "openai",
            "config": {
                "model": llm_model,
                "api_key": openai_api_key,
                "openai_base_url": openai_base_url,
            },
        },
        embedder={
            "provider": "huggingface",
            "config": {
                "model": str(model_path),
                "embedding_dims": embedding_dims,
            },
        },
        vector_store={
            "provider": "qdrant",
            "config": {
                "collection_name": "mem0",
                "path": ".mem0/qdrant",
                "embedding_model_dims": embedding_dims,
            },
        },
    )
```

## 2. 统一的数据标准 `MemoryItem`

因为 Mem0 支持了几十种不同的第三方数据库（Milvus、Pinecone、PGVector等），每个数据库查出来的数据结构都千奇百怪。有的把元数据叫 `payload`，有的叫 `metadata`。

为了不让上层业务代码崩溃，Mem0 在 `base.py` 里定义了一个**强类型的数据契约**：

```python
class MemoryItem(BaseModel):
    id: str
    memory: str
    hash: Optional[str]
    metadata: Optional[Dict[str, Any]]
    score: Optional[float]
    created_at: Optional[str]
    updated_at: Optional[str]
```

在后续的源码中我们可以看到，无论底层数据库返回的是什么奇葩格式，Mem0 都会在最后一步进行清洗验证，最终封装成 `MemoryItem` 返回。这保证了我们作为开发者，在调用 `mem.search()` 或 `mem.get_all()` 时，拿到的数据格式永远是一致的：一定有 `id`，一定有提取出的文本 `memory`，以及规范的时间戳。

## 3. 可自定义的 LLM prompt

除了这 6 个组件，`MemoryConfig` 的末尾还有两个 `LLM prompt` 配置项：

```python
    custom_fact_extraction_prompt: Optional[str]
    custom_update_memory_prompt: Optional[str]
```

Mem0 的核心机制是**“让 LLM 去阅读对话，并自己决定该提取什么记忆、该更新还是删除旧记忆”**。默认情况下，Mem0 有自己内置的 Prompt。但如果我们的业务场景很特殊——比如想让它专门提取病人的“过敏史”或“用药记录”——完全可以通过这两个字段，覆盖Mem0 的默认系统 Prompt。

**小结：**
`mem0` 通过 `MemoryConfig` 拼装了算力（LLM/Embedder）和存储（VectorDB/SQLite），又通过 `MemoryItem` 抹平了所有底层差异。

---

# mem0这个框架级百搭的适配器是怎么做的？

前面我们说，只需在 `MemoryConfig` 中传入 `{"provider": "qdrant"}` 或 `{"provider": "openai"}`，Mem0 就能乖乖地把这些底层组件挂载上来。但问题来了：Mem0 官方宣称支持几十种大模型（OpenAI、Anthropic、Ollama...）和二十多种向量数据库（Qdrant、Milvus、Pinecone...）。它是怎么把这些底层 SDK 完全不同、API 千奇百怪的第三方服务，塞进那个统一的 Harness 壳子里的？

答案在 `/utils/factory.py`这个文件中。这个文件比较简单直白，但它正是 Mem0 能够做到框架级百搭的秘诀所在。

## 1. 工厂模式

打开 `utils/factory.py`，我们可以看到五个主要的工厂类：`LlmFactory`、`EmbedderFactory`、`VectorStoreFactory`、`GraphStoreFactory` 和 `RerankerFactory`。

以 `VectorStoreFactory` 为例，它内部维护了一个巨大的字典映射：

```python
class VectorStoreFactory:
    provider_to_class = {
        "qdrant": "mem0.vector_stores.qdrant.Qdrant",
        "chroma": "mem0.vector_stores.chroma.ChromaDB",
        "pgvector": "mem0.vector_stores.pgvector.PGVector",
        "milvus": "mem0.vector_stores.milvus.MilvusDB",
        # ... 省略另外 20 多种数据库 ...
    }
```

当我们在 `MemoryConfig` 里写下 `provider="qdrant"` 时，工厂就会去查这个字典，找到对应的类路径。这就是标准的工厂模式：**将对象的创建与使用解耦**。主干业务代码永远不需要知道自己连的是什么数据库，它只管找工厂要人就行了。

## 2. 统一接口

虽然工厂根据字符串动态加载出了底层的类（比如 `mem0.vector_stores.qdrant.Qdrant` 和 `mem0.vector_stores.chroma.ChromaDB`），但这两个类的内部实现天差地别，主干代码该怎么调用它们？

答案是：**强迫它们继承同一个 Base 类**。

不管工厂吐出来的是什么怪物，在 Mem0 看来，它们都必须是 `VectorStoreBase`（位于 `vector_stores/base.py`）的子类。它们都**必须**实现以下四个核心方法：

- `insert()`
- `search()`
- `delete()`
- `update()`

这就保证了，无论底层是谁在干活，到了后面（第四章）的“总调度室”里，调度员永远只需要闭着眼睛调用 `self.vector_store.search()` 就行了。

## 3. 防呆设计

在 `LlmFactory.create` 方法中，工厂还构建了防呆机制。它会判断你传入的配置类型：

```python
elif isinstance(config, BaseLlmConfig):
    if config_class != BaseLlmConfig:
        # Convert to provider-specific config
        config_dict = {
            "model": config.model,
            "api_key": config.api_key,
            # ...
        }
        config = config_class(**config_dict)
```

有些用户可能传了一个通用的 `BaseLlmConfig`，但具体的 Provider（比如 Azure OpenAI）可能需要更严格的特有参数。工厂在这里自动充当了**“适配器”**，帮我们把通用配置安全地转化为对应厂商的强类型配置。

**小结：**
 Mem0 通过**简单工厂  + 统一接口**，完美地把复杂的底层依赖隔绝在了业务主流程之外。这也是为什么 Mem0 能在极短的时间内接入社区几十种大模型和数据库，却依然能保持主线代码清晰优雅的原因。

---

# mem0的数据清洗层是怎么做的？

搞过 Agent 的都知道，LLM是一个极度不可控的变量，即使我们搭好了统一的`MemoryItem`，也建好了能无缝切换底层引擎的`factory`,我们还是不能开始灌装数据。在真实的业务代码里，直接把 LLM 的输出扔给后端的数据库解析，程序有 80% 的概率会当场崩溃。我们让它提取事实并返回纯 JSON，它偏要在前面加一句废话“*Here is your JSON:*”；你让它返回一个数组，它偏给你包一层 Markdown 的代码块 ````json [ ... ] ````。所以 mem0 专门写了一个文件`/memory/utils.py`去做这种与大模型打交道的脏活。这个文件展示了真实的 AI 工程落地需要做哪些防御性编程。

## 1. 兜底 JSON 指令

当我们在调用大模型 API 时，如果开启了强制输出 JSON 的模式（例如 OpenAI 的 `response_format={"type": "json_object"}`），官方 API 有一个硬性规定：**Prompt 里必须显式包含 "json" 这个词**，否则会直接报错 400。

Mem0 是怎么防范用户忘记写这个词的呢？看这个 `ensure_json_instruction` 函数：

```python
def ensure_json_instruction(prompt: str) -> str:
    if "json" not in prompt.lower():
        return prompt + "\nYou must return your response in valid JSON format with a 'facts' key containing an array of strings."
    return prompt
```

这是一个极其优雅的防呆设计。如果在 `MemoryConfig` 里自定义了 Prompt，但粗心大意漏了 "json" 关键字，框架会自动在最后给你补上这句咒语，确保 API 调用绝对不会失败。

## 2. 扒掉 Markdown 外衣

即便我们在 prompt 里写的很清楚了，很多大模型（尤其是开源小模型）在输出 JSON 时，还是会习惯性地在外面包一层 Markdown 代码块。如果直接拿去 `json.loads()`，程序直接崩。

Mem0 准备了多层正则（Regex）防御机制：

**第一层：`remove_code_blocks`**

```python
def remove_code_blocks(text: str) -> str:
    # 去除开头的 ```json 和结尾的 ```
    text = re.sub(r"^```(?:json)?\n", "", text, flags=re.IGNORECASE | re.MULTILINE)
    text = re.sub(r"```$", "", text, flags=re.IGNORECASE | re.MULTILINE)
    # 甚至去除了类似 DeepSeek-R1 这种模型产生的 <think>...</think> 思考过程！
    text = re.sub(r"<think>.*?</think>", "", text, flags=re.DOTALL)
    return text.strip()
```

**第二层兜底：`extract_json`**
如果上面的正则没弄干净，或者大模型说了一堆废话“*好的，这是你需要的 JSON：{...} 希望对你有帮助！*”，Mem0 会使用最后的暴力手段：

```python
def extract_json(text: str) -> str:
    # 暴力寻找第一个 { 或 [，和最后一个 } 或 ]，强行把中间的内容抠出来
    match = re.search(r"(\{.*\}|\[.*\])", text, re.DOTALL)
    if match:
        return match.group(1)
    return text
```

这两步操作，堪称 LLM 落地应用的最佳实践，完美解决了 JSON 解析失败的痛点。

## 3. 奇葩数据矫正

有时候，模型虽然输出了合法的 JSON，但结构却完全不是我们想要的。

比如在提取记忆时，Mem0 期望模型返回一个纯字符串数组：`["喜欢无糖咖啡", "不喜欢加奶"]`。
但有些模型偏偏喜欢“自作聪明”，返回对象数组：`[{"fact": "喜欢无糖咖啡"}, {"fact": "不喜欢加奶"}]`。

Mem0 的 `normalize_facts` 函数专门负责给这种畸形数据进行矫正：

```python
def normalize_facts(facts):
    normalized = []
    for fact in facts:
        if isinstance(fact, str):
            normalized.append(fact)
        elif isinstance(fact, dict) and "fact" in fact:
            normalized.append(fact["fact"])
        # ... 强行洗成纯字符串列表
    return normalized
```

## 4. 图谱安全防御

如果我们在配置里开启了 `GraphStore`（图数据库），提取出的关系数据最终是要拼装成 Cypher 语句写进 Neo4j 的。
这时候，如果文本里夹杂了英文单引号 `'`、括号 `()` 等特殊符号，极易引发 Cypher 语法错误，甚至造成注入攻击。

`sanitize_relationship_for_cypher` 函数列了一个长长的黑名单：

```python
char_map = {
    ",": "_comma_",
    ".": "_period_",
    "'": "_squote_",
    '"': "_dquote_",
    # ... 把所有危险的标点全部替换成下划线包围的安全字符
}
```

这种近乎偏执的数据清洗，保证了写入图谱时的高可用性。

---

**小结：**

**大模型应用落地，80% 的代码量其实都在做各种 Edge Case 的兼容和数据清洗的脏活**。Mem0 之所以能被称作一个成熟的框架，很大程度上是因为它替开发者把这些脏活累活全包了。

---

# mem0是怎么做长期记忆的？

经过前面的铺垫，我们有了标准的配置引擎、百搭的底层工厂，以及专门防范大模型发疯的清洗专员。现在，所有的准备工作都已经就绪。我们终于可以谈谈 Mem0 真正的大脑——`memory/main.py`了。也是整个框架作为“Agent”而非“死板数据库”的灵魂所在。

在这个文件里，对外暴露的 API 主要分为两类：

- **纯 CRUD 接口**：`update()`, `delete()`, `get()`, `get_all()`。它们直接操作底层数据库，不涉及大模型的思考。
- **智能代理接口**：`add()` 和 `search()`。这也是我们要重点拆解的“记忆生命周期”。

## 1. `add()`：记忆是怎么被“思考”出来的？

普通的向量数据库（比如 Chroma 或 Qdrant）是怎么存聊天的？我们传给它一段话，它算成向量，塞进去。如果你连说 10 遍“我喜欢无糖咖啡”，它就会存 10 条一模一样的记录。如果第二天你说“我不喜欢无糖了，我要全糖”，它依然会存进去，导致你的系统里同时存在两条自相矛盾的“事实”。

**这就是 Mem0 的护城河：它不会“死记硬背”，而是通过设计一个Workflow来实现记忆的自更新。**

当你调用 `mem.add("我喜欢喝无糖咖啡")` 时，`main.py` 内部经历了这四步：

#### 第一步：提取新事实 (Fact Extraction)

如果是 `infer=True`（默认开启智能推断），Mem0 绝不会把这句话原封不动地存起来。
它会调用 LLM，利用系统内置的 Prompt，把冗长的口水话提炼成一条条独立且客观的事实。比如把“*我最近感觉有点胖了，以后还是喝无糖咖啡吧*” 提炼成干净的：`["用户决定改喝无糖咖啡"]`。

#### 第二步：检索旧记忆 (Semantic Search)

拿到这句新事实后，Mem0 会调用 Embedding 模型把它转成向量，然后去底层向量库（`vector_store`）里发起一次检索：**“针对当前用户（`user_id`），之前有没有存过跟咖啡相关的历史记录？”**

#### 第三步：LLM 仲裁决策 (Action Decision)

这一步是整个框架最性感的代码。
Mem0 把第二步搜到的**“旧记忆”**和第一步提取出的**“新事实”**一起打包，扔给 LLM 充当“仲裁法官”，问它：“请对比新旧信息，告诉我接下来该怎么办？”

LLM 思考后，会返回一个带有 `event_type` 标签的 JSON。它只有四个选项：

- `ADD`：这是一条全新的信息，之前没提过，添加。
- `UPDATE`：信息发生冲突了（比如以前存的是全糖，现在改无糖了），更新。
- `DELETE`：信息失效或用户要求遗忘，删除。
- `NONE`：这条信息之前早就存过了，完全重复，忽略。

#### 第四步：落地执行 (Execution)

拿到判决书后，代码进入一个大循环（`for memory in parsed_messages:`）。根据 `event_type` 的不同，分别调用内部私有方法：

- `_create_memory` -> 算向量，存入向量库。
- `_update_memory` -> 覆盖旧向量的 Payload。
- `_delete_memory` -> 从数据库抹除。
  无论执行哪一步，Mem0 都会同时调用 `self.db.add_history`，往本地的 SQLite 数据库里**记下一笔不可篡改的历史流水账**。

通过这四步，Mem0 完美解决了 RAG 系统中长期存在的**“记忆冗余”**和**“记忆矛盾”**问题。

## 2. `search()`：并发与重排工程

相比于 `add()` 复杂的智能决策，`search()` 的逻辑更加直接，但也做了极其深厚的工程优化。

当你调用 `mem.search("用户喜欢喝什么")` 时：

1. **向量化**：先把问题转成 Embedding 向量。

2. **并发检索（Concurrency）**：如果我们的配置里同时开启了图数据库（Graph）和向量数据库（Vector），Mem0 不会傻傻地串行等待。它利用 `ThreadPoolExecutor` 开启了多线程并发查询，极大降低了检索的延迟（Latency）。

   ```python
   with concurrent.futures.ThreadPoolExecutor() as executor:
       future_memories = executor.submit(...)
       future_graph_entities = executor.submit(...)
   ```

3. **混合与重排 (Rerank)**：当两路检索把初步的相似记录带回来后，Mem0 并不是简单地拼接。如果配置里挂载了 `Reranker`（比如 Cohere），它会让重排模型基于问题对这些候选记忆进行二次交叉打分，确保最精准、最相关的记忆永远排在第一位。

## 3. `get_all()`：精准的元数据过滤

这里需要特别对比一下 `get_all()` 和 `search()` 的本质区别。

- **`search()` 是基于语义的（Embedding + ANN）**：我们输入“咖啡”，它能搜出“卡布奇诺”或者“星巴克”，因为它懂语义。
- **`get_all()` 则是纯粹的数据库精确匹配（Metadata Filtering）**：它**完全不涉及**大模型和嵌入模型。它只是单纯地去向量库里下发一条 SQL 式的指令：“把 `user_id="xxx"` 并且 `created_at > 昨天` 的记录给我原封不动拿出来。”

如果我们想把某个用户的历史记忆当成背景资料全部塞给大模型，用 `get_all` 速度最快、消耗 Token 最少；如果你想让大模型精准回答某个具体问题，用 `search` 才最准。

---

# 总结

到这里，Mem0 的源码解析就全部结束了。

如果用一句话来概括 Mem0 的核心思想，那就是：
**“Mem0 不是把整段对话直接塞进向量库，而是先用 LLM 从对话里抽取‘值得长期记住的事实’，再和已有记忆比对，决定新增、更新还是删除，最后把结果持久化。”**

它巧妙地把“如何存数据”这个工程问题交给了底层百搭的工厂（`factory.py`），把“如何清洗数据”交给了正则防腐层（`utils.py`），而自己则端坐在 `main.py` 的调度室里，指挥着 LLM 扮演好“记忆仲裁官”的角色。

这种**“Harness 挂载 + 智能代理流”**的设计，极大地降低了开发者在业务层实现“长文本记忆管理”的门槛。如果想自己动手撸一个带有“自我进化能力”的 RAG 系统，Mem0 绝对是完美的参考教材！



附一个mem0接入agent的最小使用示例：

```
import json
import os

from dotenv import load_dotenv
from openai import OpenAI
from pathlib import Path

from mem0 import Memory
from mem0.configs.base import MemoryConfig
from sentence_transformers import SentenceTransformer

def build_memory() -> Memory:
    llm_model = os.getenv("LLM_MODEL")
    embedding_path = os.getenv("EMBEDDING_PATH")
    openai_api_key = os.getenv("OPENAI_API_KEY")
    openai_base_url = os.getenv("OPENAI_BASE_URL")

    if not openai_api_key:
        raise RuntimeError("Missing OPENAI_API_KEY. Add it to .env before running mem0_demo.py")
    if not llm_model:
        raise RuntimeError("Missing LLM_MODEL. Add it to .env before running mem0_demo.py")
    if not embedding_path:
        raise RuntimeError("Missing EMBEDDING_PATH. Add it to .env before running mem0_demo.py")

    model_path = Path(embedding_path)
    if not model_path.exists():
        raise RuntimeError(f"EMBEDDING_PATH does not exist: {model_path}")

    embedding_model = SentenceTransformer(str(model_path))
    embedding_dims = embedding_model.get_embedding_dimension()

    config = MemoryConfig(
        llm={
            "provider": "openai",
            "config": {
                "model": llm_model,
                "api_key": openai_api_key,
                "openai_base_url": openai_base_url,
            },
        },
        embedder={
            "provider": "huggingface",
            "config": {
                "model": str(model_path),
                "embedding_dims": embedding_dims,
            },
        },
        vector_store={
            "provider": "qdrant",
            "config": {
                "collection_name": "mem0",
                "path": ".mem0/qdrant",
                "embedding_model_dims": embedding_dims,
            },
        },
    )
    return Memory(config=config)


USER_ID = "whyulooksad"
SYSTEM_PROMPT = "你是一个带长期记忆的 AI 助手。回答时优先参考历史记忆，回答要简洁准确。"


def close_memory(memory) -> None:
    memory.close()

    vector_store = getattr(memory, "vector_store", None)
    if vector_store and getattr(vector_store, "client", None):
        vector_store.client.close()

    telemetry_store = getattr(memory, "_telemetry_vector_store", None)
    if telemetry_store and getattr(telemetry_store, "client", None):
        telemetry_store.client.close()


def build_llm_client() -> tuple[OpenAI, str]:
    api_key = os.getenv("OPENAI_API_KEY")
    base_url = os.getenv("OPENAI_BASE_URL")
    model = os.getenv("LLM_MODEL")

    if not api_key:
        raise RuntimeError("Missing OPENAI_API_KEY in .env")
    if not base_url:
        raise RuntimeError("Missing OPENAI_BASE_URL in .env")
    if not model:
        raise RuntimeError("Missing LLM_MODEL in .env")

    return OpenAI(api_key=api_key, base_url=base_url), model


def format_memories(search_result: dict) -> str:
    items = search_result.get("results", [])
    if not items:
        return "暂无历史记忆"
    return "\n".join(f"- {item['memory']}" for item in items)


def ask_agent(client: OpenAI, model: str, memory, user_input: str) -> None:
    memories = memory.search(user_input, user_id=USER_ID, limit=5)
    memory_text = format_memories(memories)

    messages = [
        {"role": "system", "content": SYSTEM_PROMPT},
        {
            "role": "user",
            "content": f"历史记忆：\n{memory_text}\n\n用户问题：{user_input}",
        },
    ]

    print("=== 检索到的记忆 ===")
    print(json.dumps(memories, ensure_ascii=False, indent=2))

    print("\n=== 注入 Prompt ===")
    print(messages[1]["content"])

    response = client.chat.completions.create(
        model=model,
        messages=messages,
        temperature=0.2,
    )
    answer = response.choices[0].message.content or ""

    print("\n=== Agent 回复 ===")
    print(answer)

    add_result = memory.add(
        [
            {"role": "user", "content": user_input},
            {"role": "assistant", "content": answer},
        ],
        user_id=USER_ID,
        metadata={"source": "agent-chat", "topic": "agent-memory-demo"},
    )
    print("\n=== 写回记忆 ===")
    print(json.dumps(add_result, ensure_ascii=False, indent=2))


def main() -> None:
    load_dotenv()
    memory = build_memory()
    client, model = build_llm_client()
    try:
        ask_agent(client, model, memory, "结合你的历史记忆，概括一下我平时研究的技术方向。")
    finally:
        close_memory(memory)


if __name__ == "__main__":
    main()
```

核心api:

```
# 1. 添加记忆
mem.add("内容", user_id="xxx")

# 2. 语义搜索
mem.search("问题", user_id="xxx")

# 3. 获取该用户所有记忆
mem.get_all(user_id="xxx")

# 4. 手动更新记忆
mem.update(memory_id="xxx", data="新内容")

# 5. 手动删除记忆
mem.delete(memory_id="xxx")
```

