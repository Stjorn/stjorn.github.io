---
title: 'RAG焚诀'
published: 2026-03-01
description: '今天来谈谈RAG实战中的一些细节。RAG这东西很简单，但是想做好还是需要一点细节。'
tags: [AI]
category: practice
draft: false
---

今天来谈谈RAG实战中的一些细节。RAG这东西很简单，但是想做好还是需要一点细节。

# RAG介绍

## 1. 什么是RAG?

RAG（Retrieval-Augmented Generation）是一种结合了 **信息检索（Retrieval）** 和 **文本生成（Generation）**的AI技术框架。它的核心思想是，在生成文本时，不仅依赖于模型本身的参数，还可以从外部知识库中检索相关的信息，以增强生成的内容。

## **2. RAG 的工作过程**

RAG分为离线和在线两个部分。

离线部分是指系统的准备阶段，主要完成**知识库的建立与向量化**。

在线部分主要包括两个阶段：

1. **检索（Retrieval）**：从知识库（如文档、数据库、互联网等）中检索出与输入问题最相关的内容。
2. **生成（Generation）**：利用检索到的信息作为额外的上下文输入，引导大语言模型（如GPT）生成更准确、更可靠的答案。

一个基本的RAG流程如下：

![image-20260416000747617](https://cdn.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260416000747927.png)

# 解析

首先是知识文档库中文档的解析，因为大多都是PDF，所以这里我直接以PDF为例。

在总体方案的选型上，我尝试过PyPDF2、Unstructured、Deepdoc。其中，Deepdoc的效果整体来说最好

| 方案         | 问题                                                         |
| ------------ | ------------------------------------------------------------ |
| PyPDF2       | 只能提取文本，无法处理复杂布局                               |
| Unstructured | 通用性强但复杂表格识别率一般                                 |
| Deepdoc      | 基于YOLOv10的布局分析+PaddleOCR，对中文文档友好，图表识别率较高 |

PDF的 pipeline：

```
输入PDF → 版面分析 → 元素分类 → 针对性解析 → 结构化输出
```

## **step1:  版面分析**

一般情况下，模型直接采用 Deepdoc 默认的 YOLOv10 即可。我也看到有人会改用 LayoutLMv3，但我没有试过。

该方案通常会识别七类版面元素：标题、正文、表格、图片、表格标题、图表标题、公式，并输出每个元素的 bbox 坐标、类别以及置信度。如果需要特殊的版面识别如印章、水印等，再去做专门的模型微调。

这里补充一个合同场景中很常见的问题：印章或签名经常会与正文发生重叠，导致 OCR 识别效果下降。我的处理思路是增加一个遮挡检查模块：当某个 text 块的 OCR 置信度低于 0.6，且与印章区域的 IoU 大于 0.3 时，先执行印章去除，再重新进行 OCR。具体做法上，可以用 `color histogram + connected component analysis` 来分离印章层和文本层，从而尽量减少遮挡对识别结果的影响。

```
  def compute_iou(box_a, box_b):
      """计算两个 bbox 的 IoU"""
      x0 = max(box_a["x0"], box_b["x0"])
      y0 = max(box_a["top"], box_b["top"])
      x1 = min(box_a["x1"], box_b["x1"])
      y1 = min(box_a["bottom"], box_b["bottom"])

      inter = max(0, x1 - x0) * max(0, y1 - y0)
      area_a = (box_a["x1"] - box_a["x0"]) * (box_a["bottom"] - box_a["top"])
      area_b = (box_b["x1"] - box_b["x0"]) * (box_b["bottom"] - box_b["top"])
      return inter / (area_a + area_b - inter + 1e-6)

  def remove_seal(image, seal_bbox, hsv_lower=(0, 80, 80), hsv_upper=(15, 255, 255)):
      """
      基于颜色直方图 + 连通域分析分离印章层，用修复算法填充印章区域。
      默认 HSV 范围针对红色印章，可根据实际印章颜色调整。
      """
      x0, y0 = int(seal_bbox["x0"]), int(seal_bbox["top"])
      x1, y1 = int(seal_bbox["x1"]), int(seal_bbox["bottom"])
      roi = image[y0:y1, x0:x1]

      hsv = cv2.cvtColor(roi, cv2.COLOR_BGR2HSV)

      # 红色在 HSV 空间中跨越 0 度，需要两个范围
      mask1 = cv2.inRange(hsv, np.array(hsv_lower), np.array(hsv_upper))
      mask2 = cv2.inRange(hsv, np.array([160, 80, 80]), np.array([180, 255, 255]))
      seal_mask = cv2.bitwise_or(mask1, mask2)

      # 连通域分析，过滤小噪点
      num_labels, labels, stats, _ = cv2.connectedComponentsWithStats(seal_mask)
      for i in range(1, num_labels):
          area = stats[i, cv2.CC_STAT_AREA]
          if area < 50:  # 面积太小的不是印章，清除
              seal_mask[labels == i] = 0

      # 膨胀 mask，确保覆盖印章边缘
      kernel = cv2.getStructuringElement(cv2.MORPH_ELLIPSE, (5, 5))
      seal_mask = cv2.dilate(seal_mask, kernel, iterations=2)

      # 用 inpaint 修复印章区域，恢复被遮挡的文本
      roi_clean = cv2.inpaint(roi, seal_mask, inpaintRadius=5, flags=cv2.INPAINT_TELEA)

      result = image.copy()
      result[y0:y1, x0:x1] = roi_clean
      return result

  def ocr_with_seal_removal(image, ocr_results, page_layout, ocr_engine,
                            conf_thr=0.6, iou_thr=0.3):
      """
      遮挡检查模块：当 text 块 OCR 置信度低且与印章区域重叠时，
      先去除印章再重新 OCR。

      参数:
          image:        原始页面图像 (numpy array)
          ocr_results:  OCR 结果列表, 每项包含 bbox 和 (text, score)
          page_layout:  版面分析结果, 每项包含 type / x0 / x1 / top / bottom
          ocr_engine:   OCR 引擎实例 (项目中的 OCR 类)
          conf_thr:     置信度阈值，低于此值视为可疑
          iou_thr:      IoU 阈值，高于此值视为被印章遮挡
      """
      seal_regions = [lt for lt in page_layout if lt["type"] in ("seal", "figure")]

      if not seal_regions:
          return ocr_results

      cleaned_image = None

      for i, (box, (text, score)) in enumerate(ocr_results):
          if score >= conf_thr:
              continue

          text_bbox = {
              "x0": min(p[0] for p in box), "x1": max(p[0] for p in box),
              "top": min(p[1] for p in box), "bottom": max(p[1] for p in box),
          }

          for seal in seal_regions:
              if compute_iou(text_bbox, seal) < iou_thr:
                  continue

              # 命中：低置信度 + 高重叠 → 去印章后重新识别
              if cleaned_image is None:
                  cleaned_image = image.copy()
                  for s in seal_regions:
                      cleaned_image = remove_seal(cleaned_image, s)

              new_text = ocr_engine.recognize(cleaned_image, np.array(box, dtype=np.float32))
              if new_text:
                  ocr_results[i] = (box, (new_text, score))
              break

      return ocr_results
```

## **step2：表格解析**

文档中通常有有边框表格和无边框表格两类表格。有边框表格deepdoc默认可以处理，而通过空格对齐的无边框表格，deepdoc的识别率一般。

我给出的方案是先用启发式规则判断有框表格和无框表格。如果是无框表格，调用MinerU 2.5来输出HTML结构化表格。

MinerU 2.5原理：基于Table Transformer架构，用row/column detection识别隐式表格结构，然后用cell matching重建单元格关系。

## **step3：扫描件/模糊图片处理**

有些PDF是做了双层的，或者里面直接贴了很多扫描件，针对这种模糊场景，我的优化思路是：先用拉普拉斯方差评估扫描图像的模糊程度，再做分级处理：清晰图直接进入解析流程；中度模糊图依次进行去噪、锐化、超分辨率和对比度增强；严重模糊图则直接切换到预处理能力更强的 MinerU 2.5，以提升整体识别鲁棒性。

```
def preprocess_scanned_image(img):
    # 模糊检测（拉普拉斯方差）

    blur_score = cv2.Laplacian(img, cv2.CV_64F).var()

    if blur_score > 100:  # 清晰图片
        return img

    elif 50 < blur_score <= 100:  # 中度模糊
        # 去噪 → 锐化 → 超分辨率 → 对比度增强
        img = cv2.fastNlMeansDenoising(img)
        img = cv2.filter2D(img, -1, sharpen_kernel)
        img = sr_model.predict(img)  # Real-ESRGAN
        img = cv2.convertScaleAbs(img, alpha=1.5, beta=10)

    else:  # 严重模糊（blur_score <= 50）
        # 直接用MinerU 2.5（它内置了更强的预处理）
        return minerU_process(img)

    return img
```

## **step4：非文本元素处理**

对于 PDF 中的图片和公式，我会在解析阶段先做类型识别，再分别转成文本化表示。流程图、示意图这类语义图片，使用多模态模型生成摘要描述；数据图表使用 chart-to-text 模型提取数据；数学公式则通过 LaTeX-OCR 转成 文本表达。这样做就可以把原本不可检索的非文本信息转换成可入库、可召回的结构化内容，再在分块阶段挂到对应 chunk 中。

# 分块

分块应该是整个RAG系统中最影响效果的一环了，很多RAG效果差，主要是这部分没做好。

RAG系统中，chunk的切分质量主要影响两个核心指标：

1. 检索召回率：chunk太大 → 噪音多，相似度计算不准；chunk太小 → 语义被割裂
2. 答案质量：chunk边界不合理 → 关键信息被截断，LLM无法正确理解

我觉得大部分人去做切块可能会是下面两种做法：

| 方案         | 做法                                       | 问题                           |
| ------------ | ------------------------------------------ | ------------------------------ |
| 固定长度切分 | 每 512- 1024 token切一刀，overlap 50 - 200 | 会截断句子，语义破碎           |
| 句子级切分   | 按句号切分，累计到大概 1024 token          | 长句子会超限；无法保留章节结构 |

基于这两种方案的坑点，给大家介绍一个实测好用的切块方案：**语义感知切分**（基于文档结构＋语义完整性）

核心思想为：

1. 优先按"章节"切分（保留完整语义单元）
2. 章节过长时，按"小节"切分
3. 小节仍过长时，按"段落"切分
4. 特殊元素（表格/图片）单独成chunk

```
def semantic_chunking(parsed_doc):
    """
    语义感知切分
    """
    chunks = []

    # Step 1: 识别文档结构
    sections = extract_hierarchy(parsed_doc)  # 后面详细讲

    for section in sections:
        # Step 2: 计算章节token数
        section_tokens = count_tokens(section.content)

        if section_tokens <= MAX_CHUNK_SIZE:  # 1024
            # 情况1: 章节长度合适，直接作为chunk
            chunks.append(create_chunk(section))
        else:
            # 情况2: 章节过长，递归切分
            chunks.extend(split_large_section(section))

    # Step 3: 添加overlap
    chunks = add_overlap(chunks, overlap_size=100)

    return chunks
```

## **step1:  识别文档结构**

各种文档的结构很复杂：

有的用数字编号：1. → 1.1 → 1.1.1

有的用中文编号：第一条 → （一）→ 1.

有的用标题大小：标题1 → 标题2 → 标题3

还有混合编号等

**我的解决方案：多策略融合**

```
def extract_hierarchy(parsed_doc):
    """
    提取文档层级结构

    优先级：
    1. 法律编号（第X条） > 数字编号（1.1） > 字母编号（a）
    2. 字体大小：H1 > H2 > H3
    3. 缩进层级
    """
    # 策略1: 正则匹配常见编号模式
    patterns = [
        r'^第[一二三四五六七八九十百]+条',   # 第三条
        r'^\d+\.\d+\.\d+',                  # 1.1.1
        r'^（([一二三四五]+)）',             # （一）
        r'^\d+\.',                          # 1.
    ]

    # 策略2: 利用解析模块输出的样式信息
    # parsed_doc包含: font_size, bold, indent_level

    # 策略3: 训练一个层级分类器（XGBoost）
    # 特征: 编号类型、字体大小、是否加粗、缩进、位置
    hierarchy_level = hierarchy_classifier.predict(features)

    return build_tree(hierarchy_level)
```

## Step2: 超长章节的递归切分

没什么好说的，直接上代码

```
def split_large_section(section, max_size=1024, min_size=256):
    """
    超长章节的切分策略
    原则：尽量保持语义完整性
    """
    chunks = []

    # 策略1: 先尝试按"小节"切
    subsections = section.get_subsections()
    if subsections:
        for sub in subsections:
            if count_tokens(sub) <= max_size:
                chunks.append(sub)
            else:
                # 递归切分
                chunks.extend(split_large_section(sub))
        return chunks

    # 策略2: 没有小节，按"段落"切
    paragraphs = section.get_paragraphs()
    current_chunk = []
    current_tokens = 0

    for para in paragraphs:
        para_tokens = count_tokens(para)

        # 关键判断：是否应该合并到当前chunk
        if current_tokens + para_tokens <= max_size:
            current_chunk.append(para)
            current_tokens += para_tokens
        else:
            # 保存当前chunk
            if current_tokens >= min_size:  # 避免太小的chunk
                chunks.append(merge(current_chunk))
                current_chunk = [para]
                current_tokens = para_tokens

    # 最后一个chunk
    if current_chunk:
        chunks.append(merge(current_chunk))

    # 策略3: 单个段落仍超长，按句子切（最后手段）
    chunks = [split_by_sentence(c) if count_tokens(c) > max_size else c
              for c in chunks]

    return chunks
```

后面还可以做专门的语义完整性检查模块降低截断率

## Step3: overlap策略

overlap的作用：防止关键信息被切分到两个chunk的边界，导致检索遗漏。

经过我的实际测试，overlap在 100 tokens左右是性价比最优点。

但是固定overlap有个问题：可能在句子中间截断。所以可以做成基于句子边界的overlap形式：

```
def add_smart_overlap(chunks, overlap_tokens=100):
    """
    智能overlap：确保overlap边界是完整句子
    """
    result = []

    for i, chunk in enumerate(chunks):
        if i == 0:
            result.append(chunk)
            continue

        # 获取前一个chunk的最后N个tokens
        prev_chunk = chunks[i-1]
        overlap_text = get_last_n_tokens(prev_chunk.content, overlap_tokens)

        # 关键：找到最近的句子边界
        overlap_text = truncate_to_sentence_boundary(overlap_text)

        # 合并
        new_content = overlap_text + chunk.content
        result.append(create_chunk(new_content, chunk.metadata))

    return result
    
 def truncate_to_sentence_boundary(text):
    """
    截断到最近的句子边界
    """
    # 找到最后一个句号/问号/叹号的位置
    sentence_ends = ['.', '。', '?', '？', '!', '！']
    last_end = -1
    for end in sentence_ends:
        pos = text.rfind(end)
        if pos > last_end:
            last_end = pos

    if last_end > 0:
        return text[last_end+1:]  # 返回最后一个完整句子之后的部分
    else:
        return text  # 找不到句子边界，返回原文
```

## Step4: 特殊元素处理

现在说说 特殊元素（表格/图片）单独成chunk 的细节

**表格的切分策略**

小表格：3行5列，300 tokens → 可以整体作为 chunk

大表格：50行10列，5000 tokens → 超过了max_size

我的方案：

```
def handle_table(table, max_size=1024):
    """
    表格的智能切分
    """
    table_tokens = count_tokens(table)

    if table_tokens <= max_size:
        # 情况1: 表格不大，整体作为chunk
        return [create_table_chunk(table)]

    else:
        # 情况2: 大表格，按"语义单元"切分

        # 策略A: 如果表格有分组
        if has_row_groups(table):
            return split_by_row_groups(table)

        # 策略B: 按固定行数切分，但保留表头
        else:
            chunks = []
            header = table.header
            rows_per_chunk = estimate_rows_per_chunk(table, max_size)

            for i in range(0, len(table.rows), rows_per_chunk):
                chunk_rows = table.rows[i:i+rows_per_chunk]
                # 关键：每个chunk都包含表头
                chunk = Table(header=header, rows=chunk_rows)
                chunks.append(create_table_chunk(chunk))

            return chunks
```

**图片的处理**

```
def handle_image(image):
    """
    图片的处理策略
    """
    # 策略1: 用多模态模型生成描述
    if is_chart_or_diagram(image):
        # 对于流程图、示意图
        description = gpt5.generate_description(image)
        return create_chunk(
            content=description,
            metadata={'type': 'image', 'image_path': image.path}
        )

    # 策略2: 对于数据图表，提取结构化数据
    elif is_data_chart(image):
        # 用Deplot模型提取数据
        data = deplot.extract(image)
        return create_chunk(
            content=f"图表数据: {data}",
            metadata={'type': 'chart', 'image_path': image.path}
        )

    # 策略3: OCR提取文字
    else:
        text = ocr.extract(image)
        return create_chunk(content=text, metadata={'type': 'image'})
```

## 元数据设计

chunk里不仅有content，还要有metadata。metadata最起码要有下面这些信息：

1. 基础信息：文档 id、chunk唯一 id、页码
2. 结构信息：所属章节标题、章节路径、层级深度（用于答案溯源）
3. 类型信息: text / table / image 、是否是关键条款（用于检索加权）
4. 位置信息：在原PDF中的坐标、前一个chunk、后一个chunk（用于上下文扩展，如果检索到chunk语义不完整，自动拉取前后chunk）

# 向量入库

这部分没什么坑点，推荐一些向量化模型和向量库吧。

## 主流向量化模型

| 模型                | 语言覆盖           | 典型版本                                                     | 亮点 & 场景                                                  |
| ------------------- | ------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| BGE                 | 中 - 英 + 多语     | bge-base-en/v1.5、bge-large-zh、bge-m3                       | 中文效果出色、8K 上下文、配套同名 reranker；MTEB 榜单同尺寸第一梯队 |
| E5                  | 英语 / 多语        | e5-base-v2、multilingual-e5-*、e5-mistral-7B-instruct        | 微调成本低，社区基线；新版 -v2 提升长文表现                  |
| GTE                 | 中 - 英 / 多语     | gte-base-en-v1.5、gte-multilingual-base、gte-Qwen2-7B-instruct | 8K 上下文、在 MTEB 多语榜登顶；已有官方 reranker 发布        |
| Instructor          | 英语               | Instructor-base / XL                                         | Instruction-tuning，一句提示可换任务，做分类 / 排序也很方便  |
| Jina Embeddings v2  | 英 / 中 / 多语版本 | jina-embeddings-v2-base-zh/en                                | 8K 长上下文、推理快；配合 Jina ColBERT 做长文检索            |
| MiniLM / all-MiniLM | 英语               | all-MiniLM-L6-v2                                             | 33M 参数的轻量模型，CPU 端极快，做边端检索常用               |

一般情况下无脑 **bge-m3**

- 中文或中英混合：`bge-m3` 或 `bge-large-zh`
- 多语言：`gte-multilingual-base` 或 `bge-m3`
- 资源紧张 / 边缘设备：`e5-small` 或 `MiniLM`
- 长文 ≥ 8K token：`Jina Embeddings v2`

## 主流向量库

| 向量库 / 数据库      | 部署方式         | 核心亮点                                             | 适用场景                                  |
| -------------------- | ---------------- | ---------------------------------------------------- | ----------------------------------------- |
| FAISS                | 本地库（无服务） | 速度极快、轻量、纯内存 / 磁盘支持、百亿级向量规模    | 本地实验、原型开发、中小数据量快速检索    |
| Chroma               | 本地 / 嵌入式    | 开箱即用、零配置、专为 RAG 优化、自带持久化          | 新手入门、个人项目、快速验证原型          |
| Qdrant               | 自部署 / 云托管  | 高性能开源、支持复杂过滤、分片分布式、高并发         | 中大型生产项目、高要求 RAG 系统           |
| Milvus / Zilliz      | 自部署 / 云托管  | 企业级分布式、支持百亿级向量、功能全面、国内生态完善 | 超大规模数据、金融 / 政务等企业级场景     |
| Pinecone             | 云托管服务       | 全托管免运维、自动扩容、高并发稳定、支持实时更新     | SaaS 产品、不想维护基础设施的生产项目     |
| Weaviate             | 自部署 / 云托管  | 向量 + 知识图谱融合、混合检索、支持多模态            | 知识图谱 + RAG 融合项目、复杂语义关联场景 |
| Elasticsearch (8.0+) | 自部署 / 云托管  | 文本 + 向量混合检索、兼容现有 ES 生态、成熟稳定      | 已有 ES 集群的老项目改造、混合检索需求    |

**RAG 项目组合推荐：**

- 嵌入模型：**bge-m3**
- 向量库：**Chroma（开发） / Qdrant（生产）**，企业更偏向于用 **Milvus**，但大部分企业的数据规模其实根本没到那个级别，而**Qdrant** 部署简单，运维成本低，内存占用低，查询延迟低。

# 向量检索

检索这一环是RAG的灵魂。

现在业界主流的做法都是**混合检索＋动态权重**了，架构也很成熟：

```
Query输入
  ↓
意图识别（分类：精确 vs 语义）
  ↓
  ├──→向量检索 → Top-K candidates + scores
  ↓
  └──→关键词检索 → Top-K candidates + scores
  ↓
分数归一化 + 动态加权融合
  ↓
去重 + 合并（RRF / 加权求和）
  ↓
粗排结果（Top-10）
  ↓
精排（Cross-Encoder重排序）
  ↓
最终Top-5 → 输入LLM
```

## 向量检索的技术细节

### **query查询优化**

用户query通常很短，但文档chunk很长，向量相似度计算不准，所以需要 query 扩展：

```
def expand_query(query, method='llm'):
    """
    Query扩展：将短query扩展为更丰富的表达

    方法1: 用LLM改写
    方法2: 用同义词库扩展
    方法3: 用历史query学习
    """
    if method == 'llm':
        # 用LLM生成query的多种表达
        prompt = f"""
用户问题：{query}

请生成3个语义相同但表达不同的问题，用于检索：
1. 更口语化的表达
2. 更专业化的表达
3. 包含相关术语的表达
"""
        expanded_queries = llm.generate(prompt)

        # 对每个扩展query做向量检索，合并结果
        all_results = []
        for q in expanded_queries:
            results = vector_search(q, top_k=3)
            all_results.extend(results)

        # 去重 + 重新排序
        return rerank(all_results)
```

考虑到这种query扩展会极大增加查询延迟，所以只对核心业务场景使用query扩展，对简单FAQ不扩展。

### **负样本挖掘**

bge系列模型其实已经很强了，但在专业领域（比如保险领域）仍有提升空间。必要时可以构建数据进行微调：

```
# 三元组：(query, positive_doc, negative_doc)
training_data = [
    {
        'query': '核辐射在保障范围吗',
        'positive': '第3条 责任免除：核辐射、核爆炸...',  # 正样本
        'negative': '第2条 保险责任：本保险承保...'       # 负样本
    },
    ...
]
```

关键是负样本要怎么选：

1. 随机负样本：随机一个chunk。但是这种模型基本上很难学到东西。

2. 难负样本：用当前检索模型，选择排名2-5但不相关的chunk。这些chunk"看起来相关"，但语义不对。这个方法给到夯，效果很好，模型可以学到更细粒度的区分。

   ```
   def mine_hard_negatives(query, positive_doc, top_k=10):
       """
       挖掘难负样本
       """
       # 用当前模型检索
       candidates = vector_search(query, top_k=top_k)
   
       # 排除正样本
       hard_negatives = [c for c in candidates if c != positive_doc]
   
       # 取top 2-5（这些是"看起来相关但实际不相关"的）
       return hard_negatives[1:5]
   ```

### **Milvus的配置**

```
# 索引类型：HNSW（层次化小世界图）
index_params = {
    "index_type": "HNSW",
    "metric_type": "IP",   # 内积（余弦相似度的等价形式）
    "params": {
        "M": 16,            # 每层的邻居数（越大越精确，但更慢）
        "efConstruction": 200  # 构建索引时的搜索宽度
    }
}

# 搜索参数
search_params = {
    "metric_type": "IP",
    "params": {"ef": 64}  # 搜索宽度（越大越精确）
}
```

## 关键词检索的技术细节

关键词检索现在都是用 **BM25**：
$$
\text{score}(q, d) = \sum_{q_i \in q} \text{IDF}(q_i) \times \frac{f(q_i, d) \cdot (k_1 + 1)}{f(q_i, d) + k_1 \cdot \left(1 - b + b \cdot \frac{|d|}{\text{avgdl}}\right)}
$$

| 符号    | 核心含义            | 补充说明                                                     |
| ------- | ------------------- | ------------------------------------------------------------ |
| f(qi,d) | 词qi在文档d中的频率 | 即词频，统计该查询词在目标文档中出现的总次数，是 BM25 得分的核心基础项 |
| \|d\|   | 文档长度            | 目标文档的总词数（token 数），用于文档长度归一化计算         |
| avgdl   | 平均文档长度        | 整个检索文档库中，所有文档的平均词数，是长度归一化的基准值   |
| k1      | 词频饱和参数        | 行业通用取值为 1.2~2.0，用于控制词频对得分的影响上限，避免高频词堆砌导致的得分虚高 |
| b       | 长度归一化参数      | 行业通用取值为 0.75，用于平衡长文档和短文档的得分差异，避免长文档因词数更多获得不合理的高分 |

现在BM25都是一键设置，算法原理倒是不需要去深入理解了。在检索之前最好先进行一下文本的预处理。

### **文本预处理**

```
def preprocess_text(text):
    """
    关键词检索的文本预处理
    """
    # Step 1: 分词（用jieba + 自定义词典）
    words = jieba.cut(text)

    # Step 2: 去除停用词
    stopwords = load_stopwords()  # "的", "了", "在"等
    words = [w for w in words if w not in stopwords]

    # Step 3: 同义词替换（关键！）
    synonym_dict = {
        '小孩': '儿童',
        '孩子': '儿童',
        '未成年人': '儿童',
        '摔伤': '意外伤害',
        '摔倒': '意外伤害',
        ...
    }
    words = [synonym_dict.get(w, w) for w in words]

    # Step 4: 提取关键词（可选，用于长文本）
    if len(words) > 20:
        words = extract_keywords(text, top_k=10)  # 用TF-IDF提取

    return words
```

### **Milvus的配置**

```
{
  "settings": {
    "analysis": {
      "analyzer": {
        "insurance_analyzer": {
          "type": "custom",
          "tokenizer": "ik_max_word", // 中文分词
          "filter": [
            "lowercase",
            "insurance_synonym", // 同义词过滤器
            "insurance_stop"     // 停用词过滤器
          ]
        }
      },
      "filter": {
        "insurance_synonym": {
          "type": "synonym",
          "synonyms": [
            "孩子,儿童,小孩,未成年人",
            "摔伤,摔倒,跌倒 => 意外伤害",
            ...
          ]
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "content": {
        "type": "text",
        "analyzer": "insurance_analyzer",
        "fields": {
          "keyword": { // 精确匹配字段
            "type": "keyword"
          }
        }
      },
      "section_title": {
        "type": "text",
        "boost": 2.0 // 标题权重加倍
      }
    }
  }
}
```

## 混合检索的融合策略

### **分数归一化**

向量检索的分数范围：0.6-0.95（余弦相似度），BM25 的分数范围：0-50+（无上界），不能直接相加，需要进行一步归一化：

min-max归一化

```
def normalize_scores(scores):
    """
    将分数归一化到[0, 1]
    """
    min_score = min(scores)
    max_score = max(scores)

    if max_score == min_score:
        return [0.5] * len(scores)  # 避免除零

    normalized = [(s - min_score) / (max_score - min_score) for s in scores]
    return normalized
```

更好的方法 : Z-score归一化

```
def z_score_normalize(scores):
    """
    Z-score归一化：处理异常值更鲁棒
    """
    mean = np.mean(scores)
    std = np.std(scores)

    if std == 0:
        return [0.5] * len(scores)

    z_scores = [(s - mean) / std for s in scores]

    # 映射到[0, 1]（用sigmoid）
    normalized = [1 / (1 + np.exp(-z)) for z in z_scores]
    return normalized
```

 Z-score 归一化相比 min-max 归一化，对异常值更鲁棒，适合处理 BM25 这类容易出现极端高分的分数分布：

1. 先通过 Z-score 标准化，将分数转为均值为 0、方差为 1 的标准分布
2. 再用 Sigmoid 函数将结果映射到 `[0, 1]` 区间
3. 加入了标准差为 0 的保护逻辑，避免除零错误

### **动态权重的设计**

意图识别：用LLM做意图识别

```
INTENT_CLASSIFIER_PROMPT = """
你是一个XX问答系统的意图分类器。

用户的问题可以分为两类：
1. 精确查询：包含专业术语、明确的概念，需要精确匹配
   - 例子：XXXX

2. 语义查询：口语化表达、描述场景，需要理解语义
   - 例子：XXXX

请判断以下问题属于哪一类，只回答"精确"或"语义"：

问题：{query}
分类：
"""

def classify_intent(query):
    """
    用LLM做意图识别
    """
    prompt = INTENT_CLASSIFIER_PROMPT.format(query=query)
    response = llm.generate(prompt, max_tokens=5)

    if "精确" in response:
        return "exact"
    elif "语义" in response:
        return "semantic"
    else:
        # 兜底：用启发式规则
        return heuristic_classify(query)


def heuristic_classify(query):
    """
    启发式规则（作为LLM的备份）
    """
    exact_keywords = ['XXX', 'YYY', 'ZZZ', 'SSS', 'TTT', 'WWW']
    semantic_keywords = ['111', '222', '333', '444', '555']

    for kw in exact_keywords:
        if kw in query:
            return "exact"

    for kw in semantic_keywords:
        if kw in query:
            return "semantic"

    # 默认：如果query很短（<5字），倾向于精确查询
    if len(query) < 5:
        return "exact"
    else:
        return "semantic"
```

权重的动态调整：

```
def get_fusion_weights(intent):
    """
    根据意图返回融合权重
    返回: (vector_weight, bm25_weight)
    """
    if intent == "exact":
        # 精确查询：更依赖关键词匹配
        return (0.3, 0.7)
    elif intent == "semantic":
        # 语义查询：更依赖向量检索
        return (0.7, 0.3)
    else:
        # 不确定：平均权重
        return (0.5, 0.5)
```

### **结果融和算法**

倒数排名融合（RRF）

```
def rrf_fusion(vector_results, bm25_results, k=60):
    """
    RRF融合：对排名融合，而不是分数融合

    公式：RRF(d) = Σ 1/(k + rank_i(d))
    """
    doc_scores = {}

    # 向量检索的排名贡献
    for rank, (doc_id, _) in enumerate(vector_results, start=1):
        doc_scores[doc_id] = 1 / (k + rank)

    # BM25的排名贡献
    for rank, (doc_id, _) in enumerate(bm25_results, start=1):
        if doc_id in doc_scores:
            doc_scores[doc_id] += 1 / (k + rank)
        else:
            doc_scores[doc_id] = 1 / (k + rank)

    # 排序
    ranked = sorted(doc_scores.items(), key=lambda x: x[1], reverse=True)
    return ranked[:10]
```

## 精排的技术细节

向量检索只考虑query和chunk的向量相似度，是独立打分，BM25只考虑词匹配，无法理解语义相关性。

举例：

Query: "孩子摔伤住院，意外险能赔吗？"

Chunk A: "第 2 条 保险责任：本保险承保意外伤害导致的医疗费用..."

Chunk B: "第 3 条 责任免除：未成年人在校园内的伤害不予赔付..."

粗排可能给 Chunk B 更高分（因为包含 "未成年人"" 伤害 " 等关键词），但实际上 Chunk A 才是正确答案。

精排要做的：理解 query 和 chunk 的深层语义关系

### **Cross-Encoder 的原理**

将 query 和候选段落拼接在一起输入 RoBERTa。对比 Bi-Encoder vs Cross-Encoder：

| 特性      | Bi-Encoder（粗排）      | Cross-Encoder（精排）       |
| --------- | ----------------------- | --------------------------- |
| 输入      | query 和 doc 分别编码   | query 和 doc 拼接后一起编码 |
| Attention | query 和 doc 不交互     | query 和 doc 充分交互       |
| 速度      | 快（可预计算 doc 向量） | 慢（需要实时计算）          |
| 精度      | 中                      | 高                          |

### **Rerank 模型推荐**

| 类型                   | 代表模型                                                     | 特点                                                         |
| ---------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 交叉编码 Cross-Encoder | `BAAI/bge-reranker-base`                                     | 挺好。慢了点                                                 |
| 交叉编码 Cross-Encoder | `cross-encoder/ms-marco-MiniLM-L6-v2` (Sentence-Transformers) | 英文检索圈最常用 “万金油” 精排，推理只需 2–3 ms / 段落 (Hugging Face) |
| 交叉编码 Cross-Encoder | `gte-multilingual-reranker-base`                             | 多语 + 中文官方精排，直接接 GTE embedding (Hugging Face)     |
| Late-Interaction       | `jina-colbert-v2`                                            | ColBERT 结构，长文检索时精度 / 速度折中好 (Hugging Face)     |
| 稀疏 / 混合            | `SPLADE-v2`                                                  | 生成词项稀疏向量，可和稠密向量做 hybrid 检索 (GitHub)        |

**组合推荐**

1. 经典流水线：`BGE-base` 检索 top 100 → `bge-reranker-base` 精排
2. 多语场景：`gte-multilingual-base` + `gte-multilingual-reranker`
3. GPU 紧张：`e5-small` + `MiniLM-L6-cross-encoder`（batch 推理）
4. 长文 / 8 K：`jina-embeddings-v2` + `jina-colbert-v2`，段内匹配更稳

Rerank模型同样可以选择进行模型微调进行领域增强，和Embedding模型一样，这里不赘述了。

# LLM输出

这里又要去谈多轮对话管理和系统设计了，这里就先提供一个架构，具体的等有空再写吧：

```
用户输入（第N轮）
↓
对话历史加载（前N-1轮）
↓
话题连续性检测
├─→ [话题切换] 清空历史，按单轮处理
└─→ [话题延续] 进入多轮处理流程
  ↓
  指代消解 + Query改写
  ↓
  检索（使用改写后的query）
  ↓
  生成答案（历史上下文 + 检索结果 + 当前query）
  ↓
  更新对话历史
```

Ok，以上就是构建一个RAG系统的全部焚诀！

"RAG已死"已经被念叨一年多了，不知道什么时候就真的会被out了，时代发展太快根本学不过来啊！
