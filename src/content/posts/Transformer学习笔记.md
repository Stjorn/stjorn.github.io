---
title: 'Transformer学习笔记'
published: 2026-05-15
description: '这段时间在找实习，想着把之前学的东西都系统地复习一遍，先从Transformer开始！'
tags: [AI]
category: note
draft: false
---

这段时间在找实习，想着把之前学的东西都系统地复习一遍，先从Transformer开始！

# Transformer概述

2017 年，Google 在论文《 Attention is All you need 》中提出了 Transformer 模型，其使用 Self-Attention 结构取代了在 NLP  任务中常用的 RNN 网络结构。相比 RNN 网络结构，其最大的优点是可以**并行计算**和**长距离信息捕捉**。Transformer 的整体模型架构如下图所示：

![3c318ffbcd1b73c48c8bd433d479b9a7](https://cdn.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260415025317216.png)

目前这种 Encode-Decode 的 Transformer架构已经不常见了，主流的模型如 GPT 使用的是 Decode-only 架构，Bert 使用的是 Encode-only 架构。

我们可以把 Transformer 简单理解为一个函数，输入一个序列，预测下一个token是什么。先来了解一下它由哪些部分组成：

1. **输入**：Embedding  & 位置编码

2. **编码器/解码器**：Attention机制 & LayerNorm & 残差连接   →  FFN & 残差连接

3. **输出**：Liner层 & Softmax层

# 输入：Embedding  & 位置编码

## **Transformer 推理第一步：文本embedding**

原始输入文本 → 分词处理（Tokenization）→ 得到 token 序列 → 词汇表映射（Vocabulary Mapping） →  得到 token ID 序列 → 嵌入层（Embedding Layer）

![image-20260415030108049](https://cdn.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260415030108238.png)

因为 Transformer 自注意力是并行、无序的，而不是像 RNN 一样串行处理数据序列，所以必须引入外部位置编码来补全顺序。

## **Transformer 推理第二步：添加位置编码**

词嵌入矩阵 → 位置编码生成（Positional Encoding）→ 词嵌入矩阵＋位置编码 → 带位置信息的输入表示矩阵

![image-20260415030139893](https://cdn.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260415030140011.png)

RoPE 的本质是：利用复数乘法的几何意义（模长相乘、角度相加），不改变向量长度，只通过旋转向量的角度来注入位置信息。 当带角度的 *Q* 和 *K* 进行内积时，结果刚好只与它们的相对位置差有关。

# **编码器/解码器**：Attention机制 & LayerNorm & 残差连接 

Attention机制概括来说就是：考虑别的token对当前token在语义空间中的影响，让它能够非常准确的用高纬度的向量表达语义特征。

## **Transformer 推理第三步：多头自注意力机制**

线性变换生成Query、Key、Value矩阵 → 计算注意力得分（Attention Scores） → 得到注意力得分矩阵 → Softmax归一化＋加权求和 → 多头并行处理 → 输出上下文感知的表示矩阵

![image-20260415030224180](https://cdn.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260415030224307.png)

![](https://cdn.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260415030310502.png)多头机制的本质是:与其用一个高维的注意力头去捕捉所有语义，不如把维度切分成h个低维的”子头”(Head)，让不同的头去关注不同的特征子空间(比如有的头关注语法，有的头关注指代关系)，最后再拼起来。

多头注意力机制计算公式如下：
$$
\mathrm{MultiHead}(Q, K, V) = \mathrm{Concat}(\mathrm{head}_1, \dots, \mathrm{head}_h) W^O
$$
其中，每一个head的计算方式为：
$$
\mathrm{head}_i = \mathrm{Attention}(Q_i, K_i, V_i) = \mathrm{softmax}\left( \frac{Q_i K_i^\top}{\sqrt{d_k}} \right) V_i
$$


- Concat：把所有头计算出的结果在特征维度上拼接起来，恢复成原来的维度。
- *WO*：最后的线性投影矩阵。因为多个头是独立计算的，拼接后需要经过一次线性变换混合各个头的信息，输出最终结果。

## **Transformer 推理第四步：LayerNorm & 残差连接**

LayerNorm即Layer Normalization，是 Transformer中的层归一化，主要是为了稳定训练过程，防止梯度爆炸或消失。在《 Attention is All you need 》中位于残差连接之后，前馈网络之前。但在现如今大多位于残差连接之前。

先说一下残差连接，首次在《Deep Residual Learning for Image Recognition》中被提出，这篇论文是目前ai领域引用最高的论文。

![image-20260415030408099](https://cdn.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260415030408236.png)

残差连接的核心思想是将输入直接“跳过"子层加到输出上。也就是说遇到训练后效果下降的子层，会直接赋予一个很低的权重，将这个子层带来的训练效果的影响降到很低，从而选择性地保留训练效果好的子层。残差连接解决了深层网络训练困难，网络更深性能反而下降的问题。

然后是LayerNorm：

输入矩阵（经过注意力或者经过注意力和残差连接）→ 计算均值 → 计算标准差 → 标准化 → 缩放和平移 → 得到归一化后的矩阵。

就是正常的归一化过程，没什么好多说的。

## **Transformer 推理第五步：FFN & 残差连接**

引入前馈神经网络是为了通过非线性变换增加模型的表达能力。

FFN的处理过程：

输入矩阵 → 线性层1（升维）→ 激活函数 → 线性层2（降维）→ 残差连接 → 最终输出

![image-20260415030441839](https://cdn.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260415030441966.png)

# **输出**：Liner层 & Softmax层

## **Transformer 推理第六步：Liner层 & Softmax层**

解码器栈的输出是一个 float 向量。怎么把这个向量转换为一个词呢？通过一个线性层再加上一个 Softmax 层实现。

线性层是一个简单的全连接神经网络，其将解码器栈的输出向量映射到一个更长的向量，这个向量被称为 logits 向量。

现在假设我们的模型有 10000 个英文单词（模型的输出词汇表）。因此 logits 向量有 10000 个数字，每个数表示一个单词的分数。然后，Softmax 层会把这些分数转换为概率（把所有的分数转换为正数，并且加起来等于 1）。最后选择最高概率所对应的单词，作为这个时间步的输出。

# 手撕Transformer

前面我们理清了整个 Transformer 的推理流，现在来看看整个 Transformer 的代码是什么样的

我们将依次实现：**Token Embedding**、**RoPE (旋转位置编码)**、**Multi-Head Attention**、**Feed Forward Network (FFN)**，最后把它们组装成一个完整的 **Transformer Block**。

## 1. Token Embedding

```python
import torch
import torch.nn as nn
import math
import torch.nn.functional as F

class TokenEmbedding(nn.Module):
    def __init__(self, vocab_size, d_model):
        super().__init__()
        self.embedding = nn.Embedding(vocab_size, d_model)
        self.d_model = d_model

    def forward(self, x):
        # Embedding 结果需要乘以 sqrt(d_model) 进行放大
        # 目的：避免后续加上位置编码时，词向量自身的特征方差过小被位置编码淹没
        return self.embedding(x) * math.sqrt(self.d_model)
```

## 2. 旋转位置编码 (RoPE)

现代大模型的标配，利用复数乘法在 Attention 计算时注入相对位置信息。

```python
def precompute_freqs_cis(dim: int, end: int, theta: float = 10000.0):
    """预计算复数频率"""
    freqs = 1.0 / (theta ** (torch.arange(0, dim, 2)[: (dim // 2)].float() / dim))
    t = torch.arange(end, device=freqs.device, dtype=torch.float32)
    freqs = torch.outer(t, freqs).float()
    # 转换为复数张量 e^(ix) = cos(x) + i*sin(x)
    freqs_cis = torch.polar(torch.ones_ones_like(freqs), freqs)
    return freqs_cis

def apply_rotary_emb(xq: torch.Tensor, xk: torch.Tensor, freqs_cis: torch.Tensor):
    """应用旋转位置编码到 Q 和 K"""
    # 把最后两维转为复数
    xq_ = torch.view_as_complex(xq.float().reshape(*xq.shape[:-1], -1, 2))
    xk_ = torch.view_as_complex(xk.float().reshape(*xk.shape[:-1], -1, 2))
    
    # 调整 freqs_cis 形状以支持广播
    freqs_cis = freqs_cis.view(1, xq_.shape[1], 1, xq_.shape[-1])
    
    # 复数乘法完成旋转
    xq_out = xq_ * freqs_cis
    xk_out = xk_ * freqs_cis
    
    # 转回实数并展平
    xq_out = torch.view_as_real(xq_out).flatten(3)
    xk_out = torch.view_as_real(xk_out).flatten(3)
    return xq_out.type_as(xq), xk_out.type_as(xk)
```

## 3. 多头自注意力 (Multi-Head Attention)

多头注意力的核心在于利用 `view` 和 `transpose` 对矩阵进行拆分和重排，使得各个“头”可以并行计算 Attention 分数。

```python
class MultiHeadAttention(nn.Module):
    def __init__(self, d_model, num_heads):
        super().__init__()
        assert d_model % num_heads == 0, "d_model 必须能被 num_heads 整除"
        
        self.num_heads = num_heads
        self.d_model = d_model
        self.head_dim = d_model // num_heads
        
        # 线性映射层：生成 Q, K, V
        self.wq = nn.Linear(d_model, d_model, bias=False)
        self.wk = nn.Linear(d_model, d_model, bias=False)
        self.wv = nn.Linear(d_model, d_model, bias=False)
        
        # 输出映射层
        self.wo = nn.Linear(d_model, d_model, bias=False)

    def forward(self, x, freqs_cis=None, mask=None):
        batch_size, seq_len, _ = x.shape
        
        # 1. 线性映射并拆分为多头
        # 形状变化: (bs, seq_len, d_model) -> (bs, seq_len, num_heads, head_dim)
        Q = self.wq(x).view(batch_size, seq_len, self.num_heads, self.head_dim)
        K = self.wk(x).view(batch_size, seq_len, self.num_heads, self.head_dim)
        V = self.wv(x).view(batch_size, seq_len, self.num_heads, self.head_dim)
        
        # 2. 如果提供了 RoPE 的复数频率，就在这里旋转 Q 和 K
        if freqs_cis is not None:
            Q, K = apply_rotary_emb(Q, K, freqs_cis)
            
        # 3. 维度重排，准备计算 Attention
        # 形状变化: (bs, num_heads, seq_len, head_dim)
        Q = Q.transpose(1, 2)
        K = K.transpose(1, 2)
        V = V.transpose(1, 2)
        
        # 4. 计算 Attention Score: Q * K^T / sqrt(d_k)
        # K.transpose(-2, -1) 把最后两维转置
        scores = torch.matmul(Q, K.transpose(-2, -1)) / math.sqrt(self.head_dim)
        
        # 5. (可选) 加入 Mask，比如在 Decoder 中防止看到未来信息
        if mask is not None:
            scores = scores.masked_fill(mask == 0, -1e9)
            
        # 6. Softmax 归一化并乘以 V
        attn_weights = F.softmax(scores, dim=-1)
        # out 形状: (bs, num_heads, seq_len, head_dim)
        out = torch.matmul(attn_weights, V)
        
        # 7. 拼接所有头 (Concat)
        # 形状恢复: (bs, seq_len, d_model)
        out = out.transpose(1, 2).contiguous().view(batch_size, seq_len, self.d_model)
        
        # 8. 最后经过一次线性投影
        return self.wo(out)
```

## 4. 前馈神经网络 (FFN)

通常是先将维度放大 4 倍提取非线性特征，再压缩回原来的维度。

```python
class FeedForward(nn.Module):
    def __init__(self, d_model, hidden_dim=None):
        super().__init__()
        if hidden_dim is None:
            hidden_dim = d_model * 4
            
        self.w1 = nn.Linear(d_model, hidden_dim)
        self.w2 = nn.Linear(hidden_dim, d_model)
        # 激活函数这里用经典的 ReLU，大模型常换成 GeLU 或 SwiGLU
        self.act = nn.ReLU()

    def forward(self, x):
        return self.w2(self.act(self.w1(x)))
```

## 5. 组装：完整的 Transformer Block (Pre-LN 架构)

我们采用现代大模型主流的 **Pre-LN（先归一化再进入网络）** 结构，把上述组件拼装起来。

```python
class TransformerBlock(nn.Module):
    def __init__(self, d_model, num_heads):
        super().__init__()
        self.attention = MultiHeadAttention(d_model, num_heads)
        self.ffn = FeedForward(d_model)
        
        # 定义两个 LayerNorm 层
        self.norm1 = nn.LayerNorm(d_model)
        self.norm2 = nn.LayerNorm(d_model)

    def forward(self, x, freqs_cis=None, mask=None):
        # Pre-LN 架构: x -> LN -> Attention -> Add(x)
        h = x + self.attention(self.norm1(x), freqs_cis, mask)
        
        # x -> LN -> FFN -> Add(x)
        out = h + self.ffn(self.norm2(h))
        return out
```





补一下分词器：

**Tokenization**：当前的 LLM 几乎清一色采用**子词级**的分词算法，以在完整单词和单字符之间取得平衡。以下介绍几种常用算法及其原理。子词分词的共同思想是:将词汇拆解为频繁出现的子单元，从而压缩序列长度并缓解未登录词问题，同时控制词表规模。

字节对编码(BPE)
BPE 从初始字符集开始，迭代将文本中最频繁的相邻符号对合并为新符号，重复该过程直到达到预定词汇大小。例如，“hello”最初分成字母h、e、l、l、 o，BPE 可能先合并频繁的he和1o成新符号，得到he 1 1o，进一步合并得到hel 1o，乃至最终整体作为hello一个词元。现代LLM常用一种变体称为**字节级BPE**。它以256个字节作为基本单元，确保任何Unicode文本都能表示。具体做法是将输入文本的每个字符拆解为UTF-8字节序列(例如，汉字“中”可能表示为3个字节 `\xE4\xB8\xAD`，而英文字母‘A’则是单个字节 `\x41`)，然后对字节序列执行BPE合并。这使得初始词表固定为256个符号(对应单字节的所有可能取值)，无需预先收集全部字符。

WordPiece 词片段算法
WordPiece 是另一种广泛使用的子词分词算法，被BERT等模型采用。它与BPE的过程相似，也是从字符集出发迭代合并符号，但选择何种符号对进行合并的策略略有不同。具体而言，WordPiece 并不总选择全局频次最高的符号对合并，而是评估合并后对语言模型整体概率的提升，选择**能最大化似然(或互信息)**的合并对。直观来说，WordPiece在执行合并时会考虑若两个符号经常一起出现且单独出现时反而少见,则更有价值将它们合并。例如,假设合并 play 和 ##ing (##表示非词首)能显著提高语料的整体概率，那么这个合并就会被优先选择。这种基于统计互信息的准则可以避免某些高频但意义松散的拼接，理论上产生更有信息量的子词单元。不过总体而言，WordPiece与BPE的效果和产生的词表相近，二者都是通过固定数量的合并操作来生成子词词汇。





