---
title: 'vLLM SGLang学习笔记'
published: 2026-07-09
description: '大模型推理加速真的是做agent很容易被忽视的一部分，但又非常重要'
tags: [AI]
category: note
draft: false
---

大模型推理加速真的是做agent很容易被忽视的一部分，但又非常重要。

# 大模型推理框架

## 基础概念

### 大模型推理过程

自回归解码：

模型输入任意长度 token 序列 \([t_0,t_1,...,t_{n-1}]\)，经过 Transformer 因果多头注意力前向传播后：

1. 输出隐藏向量序列 \(H = [h_0,h_1,...,h_{n-1}]\)，长度和输入严格相等；
2. h_i 的信息边界：只能看见 t_0 至 t_i 所有 token，看不见 \(t_{i+1}\) 及之后；
3. 每一个 h_i 单独过输出投影矩阵，生成一行 logits；
4. logits [i] 的数学含义：**仅基于 t_0 ~ t_i 这段上下文，预测下一个 token 的概率分布**。

eg: 输入 `[今天,天气,很好]`（n=3）

- h_0 只见过「今天」→ logits [0] = 只有 “今天” 时，下一词的预测分布
- h_1 见过「今天、天气」→ logits [1] = 两词上下文，下一词的预测分布
- h_2 见过完整三词 → logits [2] = 完整上下文，下一词的预测分布

当时的一个疑问：为啥明明确定了第二个词是天气，还要预测 “今天” 后面该接什么词？

模型算出 logits [0] 和 logits [1] 只是 Transformer 并行前向的附带产物，做续写生成**完全不会使用前两行 logits**，只取用最后一行 logits [-1] 做新词采样。不是模型要 “预测已知词”，只是一次性并行输入全序列时，架构强制产出全部位置的输出，真正使用时多余行会直接丢弃。

**推理完整分为两大阶段：Prefill + Decode**

Prefill 预填充阶段（处理用户原始 prompt）：

场景：用户提问完整输入，还没有生成任何输出 token。

执行步骤：

1. 将用户 prompt 全部编码为一整串 token：\(prompt = [t_0,t_1,...,t_{L-1}]\)，长度 L；

2. 一次性把整条长序列送入模型做并行前向传播；多头注意力同时计算全部 L 个 token 的隐藏向量 

   \(h_0 ~ h_{L-1}\)；

3. 得到完整 logits 矩阵 `shape=[L, vocab]`；

4. 只取最后一行 `logits[-1]`，通过贪心 / 采样选出**第一个要生成的新词 \(y_1\)**；

5. 把新词 \(y_1\) 拼接到原始 prompt 末尾，得到新序列：\(seq_1 = [t_0,...,t_{L-1}, y_1]\)；

6. 进入 Decode 循环。

特点：

- 一次性并行计算整条上下文，计算复杂度 \(O(L^2)\)；
- 只执行 1 次，每条请求只会跑一次 Prefill；
- 大量冗余计算产出前面无用的 logits 行，但这是并行输入不可避免的结果。



Decode 增量解码循环（逐一生成剩余输出）：

场景：Prefill 产出第一个新词后，循环生成后续所有 token 直到 EOS 结束符。

无KV缓存版的实现：

```
# seq 初始值：Prefill后拼接了第一个新词的完整长序列
while True:
    # 1. 把当前完整长序列全部重新送入模型
    logits, _ = model(seq)
    # 2. 依旧只取最后一行logits，预测下一个新词
    next_token = torch.argmax(logits[-1, :], dim=0, keepdim=True)
    # 3. 新词追加到序列末尾
    seq = torch.cat([seq, next_token], dim=-1)
    # 4. 命中结束符则终止循环
    if next_token == token_eos:
        break
```

特点：

1. 每一轮循环，**完整历史序列全部重新参与前向计算**；
2. 每一轮模型都会输出和当前序列等长的 logits 矩阵，但永远只使用最后一行；
3. 每一轮计算复杂度随序列变长持续升高：第 k 轮复杂度 \(O((L+k)^2)\)；
4. 循环的终止条件：预测出 EOS 结束 token，或达到全局最大长度`max_model_len`。

### 大模型服务需求

1. 易用性：不仅能快速适配市面上主流的开源大模型，还能让用户无缝接入现有的智能化应用或业务系统流程中。
2. 多用户支持：与只服务单个用户或有限群体的部署方式不同，大模型在线服务必须能够同时支持多个用户。
3. 高吞吐量和低延迟：大模型服务必须能够有效地处理大量请求并快速返回响应。高吞吐量确保模型可以同时处理多个请求而不出现瓶颈，而低延迟对于维持高质量的用户体验至关重要，尤其在需要及时反馈的实时应用流程中，比如 AI 实时对话。而这两点在通常情况下属于鱼和熊掌不可兼得的关系，需要进行设计平衡。

## vLLM

vLLM是一个开源的大模型推理加速框架，通过PagedAttention高效地管理attention中缓存的张量，实现了比HuggingFace Transformers高12-24倍的吞吐量。

### PagedAttention

PagedAttention 是 vLLM 的核心技术，它解决了 LLM 服务中内存的瓶颈问题。传统的注意力算法在自回归解码过程中，需要将所有输入 Token 的注意力键和值张量存储在 GPU 内存中，以生成下一个 Token。这些缓存的键和值张量通常被称为 KV 缓存。

#### KV缓存

KV Cache，该技术可以在不影响任何计算精度的前提下，通过**空间换时间**思想，提高推理性能，目前，各大推理框架都已实现并将其进行了封装且默认开启。前面生成式模型的推理过程中提到：每一轮循环，完整历史序列全部重新参与前向计算，大量算力被重复消耗。

以 `[今天,天气,很好]` 为例，生成「适、合、出、门」：

1. Prefill：一次性输入 3 个 token，算出 3 组 K、V；
2. Decode 第 1 轮：输入完整 4 个 token `[今天,天气,很好,适]`，**重新计算全部 4 个 token 的 K、V**；
3. Decode 第 2 轮：输入完整 5 个 token，**重新计算全部 5 个 token 的 K、V**；
4. Decode 第 3 轮：输入完整 6 个 token，**重新计算全部 6 个 token 的 K、V**；

浪费根源：`今天、天气、很好` 这 3 个 token 的 K、V 向量，从第一次算出后**永远不会改变**，但每一轮 Decode 循环都要重复投影、重复计算多头 K/V。

KV 缓存的核心目标：**把固定不变的历史 token 的 K、V 存到显存，Decode 阶段不再重复计算它们**。

Prefill 阶段：

1. 输入完整 prompt `[今天,天气,很好]`，并行计算全部 3 个 token 的多头 Q、K、V；
2. 计算完注意力后，不丢弃这 3 组 K、V，把每一层、每一头的 K、V 全部存入 GPU 显存，形成初始 KV 缓存；
3. 只用最后一位 hidden 算 logits，采样出第一个新词`适`；

此时缓存里保存：`K_hist, V_hist` → 对应「今天、天气、很好」3 个 token 的原生 K/V 向量。

Decode 阶段：

1. 仅计算新词自身的 Q、K、V

   本轮新词：`适`，只输入单 token `[适]`。仅计算这 1 个 token 多头对应的 `Q_new, K_new, V_new`，不再重算前面 3 个旧 token。

2. 拼接缓存 KV + 新词 KV（无融合，仅向量合并）

   读取显存里缓存的历史 KV，沿序列长度维度拼接：

$$
K_{full} = concat(K_{hist}, K_{new})
$$

$$
V_{full} = concat(V_{hist}, V_{new})
$$

​	拼接后 K/V 包含 4 个 token：今天、天气、很好、适。

3. 只用新词的 Q 做注意力匹配加权

​	只有 `Q_new` 参与计算：Q_new 和 K_full 全部 Key 做点积打分，得到新词和所有历史 token 的关联权重；然后权重乘全部 V_full 向量求和，得到新词的注意力输出；

4. 更新 KV 缓存，存入新词的原生 K_new、V_new

   将拼接后的 `K_full、V_full` 覆盖旧缓存，下一轮直接读取。缓存存的是第二步**每个 token 原始投影出来的 K、V**，不是第三步注意力加权后的中间结果。

下一轮循环，历史缓存已经包含 4 个 token，只需要输入新词`合`，重复上面流程。

**总结：**

1. Prefill：一次性算出 prompt 所有 token 的 K/V，存入显存初始化缓存。
2. Decode：每次只算 1 个新词的 QKV，复用缓存里全部历史 KV，避免重复计算旧 token；每轮生成新词后，把新词原生 K/V 追加进缓存，供下一轮复用。



但是 KV Cache 的使用受限于缓存大小。vLLM的作者发现大模型推理的性能瓶颈主要来自于内存： 1. 自回归过程中缓存的K和V张量非常大，在LLaMA-13B中，单个序列输入进来需要占用1.7GB内存  2. 内存占用是动态的，取决于输入序列的长度。由于碎片化和过度预留，现有的系统浪费了60%-80%的内存。所以，就有了PagedAttention。PagedAttention 灵感来自于**操作系统中虚拟内存和分页**的经典思想，它可以允许在非连续空间里存储连续的 KV 张量。具体来说：

- 把**整块连续 KV 缓存**拆成**固定大小的独立 Block（页）**，通常一页存 16/32 个 token 的 K/V 向量；
- 每个固定长度的块可以看成虚拟内存中的页，token 可以看成字节，序列可以看成进程。那么 物理上：Block 分散在 GPU 显存各处，不需要连续；逻辑上：每个请求靠一张 **Block Table（页表）** 记录自己用到的所有物理块，对外看起来 KV 还是完整连续序列；全局统一页池：所有空闲 Block 放进公共池子，所有请求按需申领、用完归还，碎片彻底消失。

 Prefill 阶段：

输入 prompt 长度 N=3（今天、天气、很好），块大小 16：

1. 一次性并行计算全部 3 个 token 的 K/V；
2. 从全局页池申领**1 个空闲物理 Block**，把 3 组 KV 写入这个块；
3. 在当前请求的 Block Table 里新增一条映射：逻辑块 0 → 刚分配的物理块地址；
4. 只取序列末尾 logits 生成第一个新词，进入 Decode 循环。

 Decode 循环阶（逐词生成，动态扩容 KV）：

每轮只输入 1 个新词，以新词`适`为例：

1. 仅计算新词`适`的 Q_new、K_new、V_new；
2. 查询当前请求页表，找到最后一个逻辑物理块；
   - 如果最后一块还剩空位（当前块只存了 3 个 token，能再塞 13 个）：直接把新词 KV 写入当前块，**不新增 Block**；
   - 如果块已满：向全局池申领新物理 Block，更新页表追加映射；
3. 内核根据页表，跳跃读取所有分散的历史 Block K/V，和新词 K 拼接、完成注意力加权；
4. 循环结束（遇到 EOS）：请求销毁，把它所有 Block 全部归还全局页池，供其他请求复用。

除了更好的内存管理，PagedAttention 还有两个**常用场景**：

1. **Prefix Caching 前缀 KV 缓存复用（后面的RadixAttention更适合做这个）**：引擎会对已计算完成的 KV Block 生成全局哈希索引，当新请求的**开头连续 token 前缀**与历史请求完全一致时，可直接复用对应的物理 KV Block，无需重复执行 Prefill 计算、重复存储 KV 向量。

   这意味着：多个用户使用完全相同的系统提示词、固定知识库、Few-shot 示例时，所有请求共用同一批物理 KV Block，各自仅维护独立页表做逻辑映射，显存只存储一份 KV 数据，消除大量重复算力开销；多轮对话中，聊天会话每一轮都会拼接完整历史上下文和 system prompt，不变的对话前缀可直接复用 Block，仅对本轮新增的对话片段执行 Prefill，大幅降低长对话首 token 延迟 TTFT。

2. 低成本序列抢占 / 驱逐：显存不足时，可以直接回收低优先级请求的全部 Block 归还池，快速释放大量显存；传统连续大块很难局部回收。

### Continuous Batching

这是vLLM在架构调度加速上的一个优化。

**传统静态批处理痛点**：凑一批请求一起跑，必须等最慢那条生成完才能整批释放；短请求跑完也占着 GPU 空位干等，算力严重浪费。

**Continuous Batching（连续批）核心逻辑**：把调度粒度缩到**每生成 1 个 token 的单步迭代**，每一步都会重新更新批次：

- 任意请求生成结束，立刻释放它的 KV 显存空位；
- 排队的新请求马上补进空位，不用等整批结束；
- 批次里可同时混跑 Prefill（新用户输入）和 Decode（正在生成的对话）。

**底层依赖**：必须搭配 PagedAttention 分页 KV 才能实现动态回收、复用显存块；

![image-20260708155125645](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260708155126029.png)

### vLLM部署

vLLM部署很简单：

安装vllm:

```
pip install vllm
pip show vllm
```

下载一个模型：

```
pip install modelscope
modelscope download --model model_name --local_dir local_path
```

部署OpenAI API服务：

```
vllm serve /root/models/Qwen3-8B --api-key abc123 --served-model-name Qwen/Qwen3-8B --max_model_len 4096 --port 7890

# 多卡部署的话（以两卡为例）
CUDA_VISIBLE_DEVICES=0,1 vllm serve local_path \
    --api-key abc123 \                # 设置api-key
    --served-model-name Qwen/Qwen3-8B \  # API服务的模型名称
    --max_model_len 4096 \            # 最大处理长度（输入prompt + 生成内容的 token 总数）
    --tensor-parallel-size 2 \        # 张量并行数量，和使用GPU卡数保持一致
    --port 7890
```

## SGLang

### RadixAttention

RadixAttention 是 SGLang 核心 KV 缓存管理机制，底层依然复用分页 Block 存储 KV 张量，上层用**基数树 (Radix Tree)** 替代全局哈希表，统一管理所有请求上下文，解决 vLLM PagedAttention 前缀复用门槛高、公共 System 重复计算的痛点。

尽管 PagedAttention 可以做前缀缓存，但限制还是比较多，一般固定 Block=16/32 token，只有整块 100% 匹配才能复用，只要一块里有 1 个 token 不一样，这块以及后面所有块全部缓存失效。真实场景下固定上下文的长度很难刚好是 16/32 整数倍，必然存在尾部半块（半块是不生成哈希索引的，只有完整的块才能缓存） → 缓存链条中断 → System 后半段 + 用户问题全部重新 Prefill。所以 PagedAttention 做不到**“只算差异化用户问题”**。

RadixAttention 核心目标：用基数树（压缩前缀树）实现**任意长度公共前缀跨请求自动共享 KV**，token 级精准复用，完美适配 Agent、批量 RAG、多分支 fork 场景。

**核心数据结构：基数树 Radix Tree**

基数树 = 压缩 Trie，所有请求上下文统一挂载在一棵全局树上。

每个树节点保存 4 样东西：

1. `tokens`：一段变长 token 序列（不受固定 Block 限制，可以是 500、3000 任意长度）；
2. `kv_ptr`：这段 token 对应的一批GPU 显存 KV 缓存物理地址集合；
3. `ref_count` 引用计数：多少活跃请求正在使用这段 KV；
4. 子节点字典：分叉后的不同后缀分支。

结构示例：

```
根节点
└── 节点A [3000token 固定Agent System] ref_count=100
     ├── 子节点B [用户query：今天天气]
     └── 子节点C [用户query：计算1+1]
```

100 条请求共用节点 A 的 KV，仅差异化 query 单独新建子节点。



Prefill 阶段：

1. 新请求 token 序列送入调度器，先遍历全局 Radix 树做最长公共前缀匹配

   - 匹配到 `[t0 ~ tk]` 公共段：直接读取树节点绑定的 GPU KV 缓存，跳过这一段所有 Prefill 并行计算

   - 剩余未匹配后缀 `[tk+1 ~ tn]`：仅对这段增量 token 执行标准 Prefill 并行前向，生成新 KV 向量

2. 把「复用的前缀 KV + 新计算后缀 KV」拼接成当前请求完整 KV 上下文，参与注意力计算，采样第一个生成 token

3. 树节点更新（核心区别 vLLM）

   - 若匹配末端无分叉：将新生成后缀 KV 封装为新子节点挂载到匹配节点下，对应`ref_count +=1`

   - 若匹配中途存在分叉：自动分裂原有树节点，公共段保留父节点，两条差异化后缀各自新建独立子节点

4. 当前请求绑定一条**树路径索引**（不再是独立 Block 页表），记录整条上下文在基数树上的节点链路

Decode 阶段：

1. 单新词`y_new`进入前向，仅计算新词`Q_new`（标准 KV 缓存逻辑不变，不会重算历史 token QKV）

2. 读取当前请求绑定的 Radix 树整条路径全部历史 KV，和新词 K/V 拼接做注意力加权

3. 判断当前树路径最后一个节点是否还有空余存储容量：

   - 有剩余空间：新词 K/V 直接追加写入末端节点，**不新增树节点，仅更新节点内 KV 数据**

   - 节点存满：新建叶子节点挂载在当前路径末尾，`ref_count`维持不变（仅当前请求独占叶子）

4. 采样下一个新词，循环 Decode 直到 EOS 终止

5. 树回收：沿着该请求完整树路径，对每一个节点执行`ref_count -= 1`。遍历路径：

   -  若节点`ref_count == 0`：标记可驱逐，递归回收该整条叶子子树显存（LRU 淘汰时优先清理）

   -  若节点`ref_count > 0`：代表还有其他请求复用，保留节点不销毁（全局 System 前缀永久留存）

所有生成的输出 token 只会追加到当前请求私有叶子分支，不会修改、分裂全局共享的 System 根节点；多 fork 多分支推理时，每条分支仅新增独立叶子，根公共节点全程共享、不重复存储 KV。

另外值得一提的是，用SGLang搞agent无论是用本地模型还是调云端api，都有一定优化：

- GPU RadixCache：本地模型，显存存 KV，省 GPU 算力与显存；
- CPU 客户端 RadixCache：调用云端 API，内存存 token，省 OpenAI 计费与网络延迟。（仅缓存 token 文本序列，无法操作云端模型的 GPU KV 缓存，只能减少上传 token 数量，不能跳过云端模型内部 Prefill 计算；）

vLLM / SGLang 性能边界：RadixAttention 基数树有树查找、锁竞争、额外内存指针开销；PagedAttention 扁平哈希表逻辑极简、CPU 调度开销更低。所以流量有无公共前缀，直接决定两者的适用性。

### Compressed FSM

Compressed FSM（压缩有限状态机）是 SGLang 原生高性能约束解码引擎，专门解决 JSON、正则、工具调用等结构化输出场景。对比 vLLM 外挂 Outlines/Xgrammar 逐 Token 掩码方案，核心创新是**单路径合并 + 跳跃前向解码（Jump-Forward）**，大幅减少模型前向次数，同时底层锁死非法 Token，保证输出 100% 合规。

#### FSM

FSM 是一套**规则状态流转系统**，这个场景下就是用来严格限制输出只能符合 JSON / 正则。

组成只有三样：

1. 状态 State：代表你现在写到哪一步了
2. 转移边 Edge：写完某个字符 / Token 后，能跳到下一个状态
3. 合法终止态：走完完整 JSON 才算合规

eg：强制输出 `{"name":xxx}`

完整书写顺序：`{` → `"` → `n` → `a` → `m` → `e` → `"` → `:` → `xxx` → `}`

普通 FSM 会拆成一串连续状态：

S0（初始） -{→ S1 -"→ S2 -n→ S3 -a→ S4 -m→ S5 -e→ S6 -"→ S7 -:→ S8（分叉点，后面可以写字符串 / 数字）→ S9 -}

这里有两种 FSM 路径：

- 单一路径：当前状态只有唯一一条合法转移，没有别的选择。比如写完`{`之后，下一个合法字符只能是双引号，没有第二个选项。
- 分叉路径：当前状态有多个可选转移，必须交给模型判断。比如写完`"name":` 后面，可以写字符串、数字、布尔值，多个合法选项，必须调用模型采样。

传统的FSM 解码（比如 vLLM 用的 Outlines/Xgrammar），每一轮只能生成 1 个 Token，走 1 次状态跳转。

1. 当前状态 S0，只允许输出`{` → 跑一次模型，采样`{`，跳到 S1
2. 当前状态 S1，只允许输出`"` → 再跑一次模型，采样`"`，跳到 S2
3. 当前状态 S2，只允许输出`n` → 第三次跑模型，采样`n`，跳到 S3
4. …… 一直重复，直到`:`, 一共要跑 8 次模型前向才能到分叉点 S8

这一整段是固定不变的语法，没有任何选择，但模型要**重复计算 8 次，算力全浪费**。而且每一步都要遍历词表、生成掩码、屏蔽非法 Token，开销叠加。



**单路径合并：**SGLang 的压缩逻辑是把**一长串连续、无任何分叉的单转移状态**，合并成**一条超大压缩边**，不再一步一跳。

原始 9 段单步状态：S0→S1→S2→S3→S4→S5→S6→S7→S8→S9

S0→S1→S2→S3→S4→S5→S6→S7 全部是唯一转移，无分叉，Compressed FSM 直接压缩成：S0 —— 完整字符串`{"name":` ——→ S8→ 完整字符串`}`

**跳跃前向解码：**走到压缩路径起点时，**不用逐 Token 调用模型**，引擎直接把整段固定字符串一次性填充到输出里，直接跳到分叉点 S8。只有到达分叉点（有多种可选内容），才真正执行一次模型前向、做 Token 采样。原来 8 次模型计算 → 现在 0 次，直接跳过。

拿一个实际的例子走一遍：模型必须输出 `{"tool":"search","args":{"keyword":"xxx"}}`

1. 编译阶段（只执行一次，缓存起来反复用）：我们需要写一段 Pydantic 类描述 JSON 格式： `Pydantic Schema`，SGLang 后台自动做三件事：1） 自动转正则文法，再生成一套最细粒度的传统 FSM；2） 自动扫描所有连续单转移链，全部压缩为长边；3）保存压缩后的 FSM到全局缓存，相同 Schema 复用不用重复编译。

2. Prefill 阶段：模型处理完 System Prompt、用户问题后，引擎给当前这条对话绑定**压缩 FSM 的起点状态**。此时还没开始生成 JSON，只是把规则挂载到这条请求上。

3. Decode 循环阶段：分两种情况处理。

   情况 A：当前处于压缩单路径（无选择） 引擎直接把整条固定字符串追加输出，FSM 状态一次性跳到路径末尾分叉点；**不调用 Transformer 模型，无 GPU 算力消耗**。例：走到`{"tool":`，直接补全整段固定语法，不用推理。

   情况 B：当前到达分叉节点（多选择） 比如`"args":`后面可以填字符串 / 数字数组，存在多条合法边：

   1）根据压缩 FSM 算出当前所有合法 Token 集合；

   2）GPU 算子批量把非法 Token logit 设为负无穷；

   3）仅在合法候选里采样 1 个 Token；

   4）根据采样结果，跳转到下一个 FSM 状态，继续循环。

   所有输出严格贴合 FSM 规则，不存在半残缺 JSON，业务不需要写`try-except`、字符串清洗、重试逻辑。

和 RadixAttention 的协同：RadixAttention 复用 System 前缀，削减 Prefill 算力；Compressed FSM 跳过 JSON 固定语法段，削减 Decode 循环；两者叠加，Agent 端到端延迟、GPU 显存、云端 API token 成本三重优化。

### EAGLE 投机解码

前面可以看到标准的自回归 decode 流程每一轮只能生成 1 个 token，完整一轮 GPU 前向只能产出 1 个字。GPU 算力大量浪费，TPOT（单 token 生成延迟）高。

投机解码核心思路：**先用轻量草稿单元一次性预生成多候选 token，再交给主大模型单次并行批量校验，一次性接受连续多个合法 token，减少主模型前向总次数**。

投机解码基础逻辑：任何投机解码的方案都是“猜 + 验”。主大模型生成较准，但干活慢；草稿模块干活很快，但是生成会出错。所以投机解码的步骤固定为两步：1.草稿阶段  草稿模块快速一口气猜接下来 k 个词，几乎不耗算力。2. 验证阶段  主大模型一次性把草稿模块猜的所有词全部审阅一遍，并行校验；从第一个词依次核对，全对就全部收下；遇到第一个猜错的，只保留前面正确的，错误及后面全部扔掉，重新生成；校验用严格拒绝采样算法，最终输出文本和纯大模型生成完全一模一样，不会降低回答质量，无损加速。

vLLM 采用的投机解码方式为 Medusa：不新增独立小模型，仅外挂多个独立预测头读取主模型**最后一层输出logit**，预测后续 token。这种方式读取的信息少，草稿准确率低，接受率仅 50%~65%，加速上限 只有1.8~2.5 倍。

SGLang 主推 EAGLE-3：1. 不用 token 预测，预测中间特征 `hidden state`。Transformer 每一层都会输出高维特征向量（hidden state），里面包含完整上下文语义信息；2. 独立极轻量草稿网络。不是外挂几个 MLP 头，而是一套单独 1~2 层的微型 Transformer 草稿模型（显存占用只占主模型 10%~15%；），基于主模型中间特征，递归生成多条候选 token 树（Tree Draft），校验时并行一次性全部核对。容错更高，平均单次能收下更多 token。

完整走一遍 EAGLE 单轮流程：当前上下文已经处理完毕，准备继续生成文本

1. 主模型输出多层中间特征。主模型刚算出当前位置全部层 `hidden state`，不用丢弃，直接喂给 EAGLE 草稿网络。

2. EAGLE 草稿网络快速生成候选 token 树。轻量草稿网络基于特征递归预测，一次性生成多条候选序列，比如树深度 4，多条 4 词草稿。这一步 GPU 开销极小，速度极快。

3. 主模型一次性并行校验整棵候选树。把原始上下文 + 树里所有候选 token 拼成长序列，**只运行 1 次主模型前向**。依靠特制 Tree Attention 掩码，并行算出每一个候选位置真实单词概率。

4. 逐位置比对，批量接受正确 token。从树根第一个词开始，用**拒绝采样规则**对比草稿预测和主模型真实概率：匹配达标：收下该 token；一旦出现不匹配：停止，丢弃该位置及后面所有草稿。（比如草稿树最优路径是`今天天气`，主模型校验后前 3 个完全匹配，第 4 个猜错；本轮直接一次性收下`今天天`，只消耗 1 次大模型计算，原本需要 3 次。）

5. 更新 KV 缓存，进入下一轮。把收下的 token 追加到上下文，更新 `Radix KV` 缓存，再次循环 EAGLE 流程直到 EOS 结束。

适用场景：长对话、长 CoT 推理、工具调用 JSON 输出（大量连续可变文本）；批量 RAG 抽取、批量结构化生成；7B/13B/34B 中大型模型，显存充足；吞吐优先业务，能接受少量显存占用。超短生成和极小模型就不要用EAGLE了，草稿网络带来的开销得不偿失。

和 RadixAttention/ Compressed FSM的协同：Radix 负责 Prefill 公共前缀复用，FSM 负责固定 JSON 片段跳过，EAGLE 负责 Decode 可变文本批量多 token 并行生成，三者配合是 SGLang 做 Agent 的完整加速栈。

（简单补充一下拒绝采用规则：对比草稿、主模型对同一个词的预测概率，用 0~1 随机数按比例决定留不留；一旦某个词被拒，后面所有草稿全部作废，从主模型修正分布重采样，全程保证生成文本和纯大模型输出完全一致，是投机解码无损加速的根基。）

### DSL基础语法

LLM初始化

```
import sgl
from pydantic import BaseModel

# 1. 连本地SGL服务（GPU推理，主流）
sgl.set_default_backend(sgl.RuntimeEndpoint("http://127.0.0.1:30000"))

# 2. 直连OpenAI云端API（无本地模型）
# sgl.set_default_backend(sgl.OpenAI(api_key="xxx"))
```

定义 Agent 函数核心装饰器 `@sgl.function`：所有生成逻辑包在装饰器下，支持变量插值、多轮生成、结构化约束

```
@sgl.function
def agent_chat(user_input: str):
    # 直接写prompt文本，{变量}插值
    yield f"""你是工具智能体，只能输出JSON工具调用
用户问题：{user_input}
"""
    # 生成输出，存到key="tool_call"
    res = sgl.gen("tool_call", max_tokens=256)
    return res
```

sgl.gen 核心参数（结构化 Agent 常用）

```
# 定义约束Schema
class ToolSchema(BaseModel):
    tool: str
    keyword: str

# 带JSON格式强制约束（自动触发Compressed FSM）
sgl.gen(
    "output_key",
    max_tokens=200,
    temperature=0,
    schema=ToolSchema  # 直接传Pydantic类
)

# 正则约束替代schema
# sgl.gen("res", regex=r"\{.*\}")
```

多分支并行 fork /join：同时跑多条推理分支，复用 Radix 客户端缓存，HTTP 接口做不到

```
@sgl.function
def multi_search(q):
    yield f"问题：{q}"
    # 并行两条工具调用分支
    with sgl.fork():
        yield sgl.gen("search1", schema=ToolSchema)
    with sgl.fork():
        yield sgl.gen("search2", schema=ToolSchema)
    # join 收集所有分支结果
    results = sgl.join()
    return results
```

读取生成结果、调用执行

```
# 调用函数
output = agent_chat("成都天气")
# 取出生成的json字符串
json_str = output["tool_call"]

# 批量批量请求（自动复用Radix缓存）
prompts = [agent_chat("问题1"), agent_chat("问题2")]
batch_res = sgl.run_batch(prompts)
```

变量复用、多轮对话追加

```
@sgl.function
def round_chat(history, new_q):
    yield f"历史对话：{history}"
    yield f"用户新问题：{new_q}"
    yield sgl.gen("reply")
```

最小 Agent Demo

```
import sgl
import json
from pydantic import BaseModel, Literal
from typing import Optional

# 初始化LLM
sgl.set_default_backend(sgl.RuntimeEndpoint("http://127.0.0.1:30000"))

# 工具输出结构（Compressed FSM）
class SingleTool(BaseModel):
    tool: Literal["search", "calculate"]
    keyword: str

# 支持一次返回多个工具调用数组
class MultiToolCall(BaseModel):
    calls: list[SingleTool]
    finish: Literal["yes", "no"]  # no=还要继续调用工具 yes=信息足够，直接回答

# 简单本地工具实现
def search(keyword: str) -> str:
    if "成都天气" in keyword:
        return "成都今日气温26℃，湿度65%，多云"
    elif "北京天气" in keyword:
        return "北京今日气温22℃，小雨"
    return f"搜索结果：{keyword}"

def calculate(expr: str) -> str:
    try:
        val = eval(expr)
        return f"计算结果 {expr} = {val}"
    except Exception as e:
        return f"计算失败：{str(e)}"

# 工具分发执行
def execute_tools(call_list: list[SingleTool]) -> list[str]:
    outputs = []
    for item in call_list:
        if item.tool == "search":
            outputs.append(search(item.keyword))
        elif item.tool == "calculate":
            outputs.append(calculate(item.keyword))
    return outputs

# DSL函数：生成工具调用/判断是否结束
@sgl.function
def agent_plan(history_context: str, user_question: str):
    prompt = """
你拥有两个工具：
1. search：查询城市天气、百科资讯
2. calculate：数学表达式四则运算

规则：
1. 需要查资料/计算就输出calls数组，finish填no；
2. 信息充足不需要工具时，finish填yes，calls留空；
3. 可以一次输出多个工具并行调用；
只输出JSON，禁止额外文字。

历史上下文（包含过往工具调用与返回结果）：
{history_context}
用户当前问题：{user_question}
"""
    yield prompt.format(history_context=history_context, user_question=user_question)
    res = sgl.gen("plan_out", max_tokens=300, temperature=0, schema=MultiToolCall)
    return res

# DSL函数：最终整理自然语言回答
@sgl.function
def gen_final_answer(full_context: str, user_question: str):
    prompt = """
基于下面完整上下文（用户问题、所有工具调用与工具返回数据），用通顺中文完整回答用户，不要调用工具、不要输出JSON。
完整上下文：
{full_context}
用户原始提问：{user_question}
"""
    yield prompt.format(full_context=full_context, user_question=user_question)
    ans = sgl.gen("answer", max_tokens=400)
    return ans

if __name__ == "__main__":
    user_q = "北京天气怎么样，123*456等于多少？顺便说下成都湿度"
    history = ""
    max_loop = 5  # 防止无限循环，限制最多5轮工具调用

    # ===================== 多轮工具循环核心逻辑 =====================
    for loop in range(max_loop):
        print(f"\n===== 第{loop+1}轮工具规划 =====")
        # 模型生成工具计划
        plan_res = agent_plan(history, user_q)
        plan_data = json.loads(plan_res["plan_out"])
        finish_flag = plan_data["finish"]
        tool_calls = plan_data["calls"]

        print(f"模型输出工具调用：{plan_data}")

        # 分支1：信息足够，停止工具调用，生成最终回答
        if finish_flag == "yes":
            print("模型判断无需更多工具，生成最终答案")
            final = gen_final_answer(history, user_q)
            print("\n【最终回答】", final["answer"])
            break

        # 分支2：存在多个工具，批量执行
        tool_returns = execute_tools(tool_calls)
        print(f"工具执行结果列表：{tool_returns}")

        # 把本轮工具调用+结果追加到上下文，下一轮模型可见
        round_log = f"\n【第{loop+1}轮工具调用记录】\n调用列表：{tool_calls}\n工具返回：{tool_returns}"
        history += round_log
    else:
        # 循环达到上限强制终止
        print("\n达到最大工具调用轮次限制，强制生成回答")
        final = gen_final_answer(history, user_q)
        print("【最终回答】", final["answer"])
```

### SGLang部署

安装SGLang:

```
# 比较稳定的一个版本
pip install "sglang[all]==0.5.10"
pip show sglang
```

下载模型：

```
pip install modelscope
modelscope download --model model_name --local_dir local_path
```

部署 OpenAI 兼容 API 服务：

```
python -m sglang.launch_server /root/models/Qwen3-8B \
--api-key abc123 \                  # 设置接口鉴权密钥
--served-model-name Qwen/Qwen3-8B \ # API调用时使用的模型名称
--max-model-len 4096 \              # 最大处理长度（输入prompt + 生成内容token总数）
--port 30000                        # 服务监听端口
--speculative-algorithm EAGLE3 \                    # 开启EAGLE投机解码总开关
--speculative-draft-model-path lmsys/SGLang-EAGLE3-Qwen3-8B \ # 配套草稿模型权重路径
--speculative-num-draft-tokens 4                    # 草稿模型单次预生成token数量，推荐2~8

这个是开启了EAGLE投机解码的，不想开直接把最后三行删了就行了，RadixAttention是默认开启的，Compressed FSM也无需启动参数，调用接口时在extra_body传入json_schema自动生效。

## 多卡部署（以两卡为例）
CUDA_VISIBLE_DEVICES=0,1 python -m sglang.launch_server local_path \
--api-key abc123 \                      # 设置接口鉴权key
--served-model-name Qwen/Qwen3-8B \     # API调用时使用的模型名
--max-model-len 4096 \                  # 上下文最大总token长度（输入+输出）
--tp-size 2 \                           # 张量并行卡数，和指定GPU数量一致
--port 30000
--speculative-algorithm EAGLE3 \                    # 开启EAGLE投机解码总开关
--speculative-draft-model-path lmsys/SGLang-EAGLE3-Qwen3-8B \ # 配套草稿模型权重路径
--speculative-num-draft-tokens 4                    # 草稿模型单次预生成token数量，推荐2~8


常用附加参数（按需拼接）
--host 0.0.0.0 \                    # 允许外部机器访问服务
--gpu-memory-utilization 0.85 \     # GPU显存占用上限
--max-running-requests 32 \          # 最大并发请求数
--log-level info \                  # 日志输出级别
--enable-metrics \                 # 开启监控指标采集
--stream-interval 1 \              # 流式推送token间隔，单位token，1=逐token实时返回
--stream-output                    # 开启分片流式输出底层支持（默认已启用，显式写更稳妥）
```

