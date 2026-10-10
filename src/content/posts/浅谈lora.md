---
title: '浅谈LoRA SFT'
published: 2025-09-02
description: '这次来谈谈LoRA微调'
tags: [AI]
category: note
draft: false
---

这次来谈谈LoRA微调。

# 一、基础概念

先看一下LLM训练的全过程：

![image-20260602224902327](https://cdn.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260602224902401.png)

在预训练阶段，模型在大规模标注文本上学习对下一个token的分布表示；在下游应用中，通过微调可使模型适应特定任务领域或风格（如问答、代码生成、对话系统等）。**微调本质是教会模型遵循用户指令，把知识按照要求表达出来。**

微调分为全参微调和低参微调两大类，低参微调又有Prompt-Tuning、 P-Tuning、 Prefix-Tuning、Adapter Tuning、 LoRA等多种方法，本篇就重点讲解目前工业界应用最广泛的 LoRA 微调。

LoRA 微调：通过矩阵分解将大矩阵近似表示为两个小矩阵的乘积，只训练分解后的两个小矩阵，可大幅降低显存需求。例如，将 \(4096×4096\) 的矩阵分解为 (4096×4) 和 \(4×4096\) 相乘，参数数量从 160 万下降到 3 万左右，降低了 98%。

假设这是一个4×5的矩阵：
$$
W=
\begin{bmatrix}
w_{11} & w_{12} & \dots & w_{1n} \\
w_{21} & w_{22} & \dots & w_{2n} \\
\vdots & \vdots & \ddots & \vdots \\
w_{m1} & w_{m2} & \dots & w_{mn}
\end{bmatrix}
$$
LoRA 不改动原模型权重，用两个低秩小矩阵相乘模拟原权重的微调增量：
$$
\Delta W = A \times B
$$

$$
A=
\begin{bmatrix}
a_{11} & a_{12} \\
a_{21} & a_{22} \\
a_{31} & a_{32} \\
a_{41} & a_{42}
\end{bmatrix},\quad
B=
\begin{bmatrix}
b_{11} & b_{12} & b_{13} & b_{14} & b_{15} \\
b_{21} & b_{22} & b_{23} & b_{24} & b_{25}
\end{bmatrix}
$$

矩阵乘积A×B是一个4×5的矩阵：
$$
A\times B=
\begin{bmatrix}
a_{11}b_{11}+a_{12}b_{21} & a_{11}b_{12}+a_{12}b_{22} & \cdots & a_{11}b_{15}+a_{12}b_{25} \\
a_{21}b_{11}+a_{22}b_{21} & a_{21}b_{12}+a_{22}b_{22} & \cdots & a_{21}b_{15}+a_{22}b_{25} \\
\vdots & \vdots & \ddots & \vdots \\
a_{41}b_{11}+a_{42}b_{21} & a_{41}b_{12}+a_{42}b_{22} & \cdots & a_{41}b_{15}+a_{42}b_{25}
\end{bmatrix}
$$
仅训练A、B实现参数高效微调，训练结束后W+微调增量可以合并回原权重，推理无额外计算开销：
$$
h = (W + BA)x = (W + \Delta W)x
$$
**参数对比：**原W参数4×5=20个；LoRA (A+B) 总参数4×2+2×5=18，r越小参数量压缩越夸张。

虽然Lora微调能极大地节省计算开销，但是能做全量微调还是尽量不要做 LoRA 微调，LoRA 微调精度有损失，只适用于部分简单场景，涉及重要能力的微调需用全量微调。

前向：

![951246378bac825f801188d5cb2eda2f](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260925015217108.jpg)

反向：

<img src="https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260925015249681.jpg" alt="4785ba0eee64070c659880df9c513dcd" style="zoom: 33%;" />

# 二、手撕实战

这里以对`DeepSeek-R1-Distill-Qwen-7B`这个模型做医学问答方面的微调为例。

需要用到的包如下：

torch：深度学习底层框架PyTorch

transformers：LLM 加载和 Tokenizer 核心库

datasets：数据集处理工具

peft：参数高效微调库

trl：LLM 训练流水线库

accelerate：简易分布式 / 混合精度工具

wandb：训练日志可视化平台

deepspeed：微软超大模型显存优化框架

整体链路如下：

```
torch(底层) → transformers(模型+分词) + datasets(数据) → peft(LoRA) + trl(SFT训练器) → accelerate/deepspeed(分布式显存优化) → wandb(日志监控)
```

## 1. 配置模型和数据路径

```
model_id_or_path="DeepSeek-R1-Distill-Qwen-7B"
dataset_path="dataset/medical_o1_sft_Chinese.json"
ouput_dir="sft_lora_model_int_output_new"

tokenizer=AutoTokenizer.from_pretrained(model_id_or_path,trust_remote_code=True)
model=AutoModelForCasualLM.from_pretrained(model_id_or_path,trust_remote_code=True)
```

## 2. 加载数据集

```
print("Loadding Dataset...")
dataset=load_dataset("json",data_files=dataset_path,split="train")
print(f"数据集一共有{len(dataset)}组数据")
print(f"数据集:{dataset}")
print(f"数据集列{dataset.column_names}")
```

![image-20260604171427906](https://cdn.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260604171428330.png)

## 3. 处理数据集

```
train_prompt_style_zh = """
请根据下方指令和提供的具体问题，撰写一个恰当的回复。
在回答前，请仔细思考问题，并展现一步一步的思考过程，以确保回复的逻辑性和准确性。

### 指令:
你是一位医学专家，精通临床推理、诊断和治疗计划。
请回答下面的医学问题。

### 问题:
{}

### 回答:
<think>
{}
</think>
{}
"""
```

固定对话 Prompt 模板，3 个`{}`依次占位：

第 1 个`{}`：数据集字段`Question` → 用户医学问题

第 2 个`{}`：数据集字段`Complex_CoT` → CoT 思考推理过程（分步解题思路）

第 3 个`{}`：数据集字段`Response` → 最终标准答案

```
# eos_token：模型结束标识符（如<|endoftext|>），每条样本末尾强制拼接结束符，告诉模型一句话到此结束，微调时学习终止生成。
EOS_TOKEN=tokenizer.eos_token 
def formatting_prompts_func(examples):
    inputs=examples["Question"]
    cots=examples["Complex_CoT"]
    outputs=examples["Response"]
    texts=[]
    for input,cot,output in zip(inputs,cots,outputs):
        text=train_prompt_style_zh.format(input,cot,output)+EOS_TOKEN
        texts.append(text)
    return {
        "text":texts,
    }
```

最终生成新字段`text`，是后续 Trainer 训练唯一输入字段。因为我们后面是用 trl 来训练，所以把输入输出拼接成一个 text 就可以了。如果想正常用 peft 来训练的话，需要把数据集处理成`text`, `input_ids`, `attention_mask`这三个标准的列然后传入模型去训练。trl 库是对整个流程进行简化，只需要数据集里有 text 这一列就可以了。

```
print("Formating dataset...")
# 数据集映射处理
dataset=dataset.map(formatting_prompts_func,batched=True)
print("Dataset after formatting",dataset)
print("Dataset[0]",dataset[0]["text"])

if tokenizer.pad_token is None:
    tokenizer.pad_token=tokenizer.eos_token
```

pad是padding 填充符，batch 训练时补齐短句长度必需。很多大模型（Llama、Qwen、Mistral）原生没有 pad_token，只有 eos_token，所以赋值`pad_token = eos_token`，避免 Trainer 训练时报错。

![image-20260604232112870](https://cdn.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260604232113185.png)

## 4. 模型量化

```
# 配置量化
bnb_config=BitsAndBytesConfig(
    #这里我采用的是4bit量化，想改成8bit量化直接把下面四个参数替换成load_in_8bit=True就行
    load_in_4bit=True,   # 开启 4bit 量化
    bnb_4bit_use_double_quant=True, # 双重量化，更省显存
    bnb_4bit_quant_type="nf4",  # 用更精准的 4bit 格式（QLoRA 专用）
    bnb_4bit_compute_dtype=torch.bfloat16 # 计算时用 bf16
)
# 加载模型 
model=AutoModelForCausalLM.from_pretrained(
    model_id_or_path,
    quantization_config=bnb_config, # 传入量化配置
    device_map="auto",   # 自动把模型分配到 GPU/CPU
    trust_remote_code=True # 运行模型自定义代码
)
print("model loaded")
print(model)
```

![image-20260605110112078](https://cdn.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260605110112567.png)

## 5. LoRA配置

```
# 把量化模型设置成可以训练的模式。
model = prepare_model_for_kbit_training(model)
```

主要有下面三个作用：

1. 关闭模型的冻结，让量化层支持梯度更新
2. 关闭不需要的缓存，节省显存
3. 让量化权重可以参与反向传播

```
lora_config = LoraConfig(
    r=16,   # LoRA 的秩
    lora_alpha=16,   # 缩放系数，一般设置成和r一样就行
    # 把 LoRA 加在哪些层上。下面是给模型的所有线性层都加上了LoRA，q/k/v/o是注意力层，gate/up/down是前馈层
    target_modules=["q_proj","k_proj","v_proj","o_proj","gate_proj","up_proj","down_proj"], 
    lora_dropout=0, # 随机丢弃一部分训练权重，防止过拟合
    bias="none",   # 是否训练偏置项
    task_type="CAUSAL_LM"  # 模型类型：自回归大模型
)
```

```
# 给模型绑定上LoRA适配器，冻结原模型的全部权重，新增少量可训练的LoRA权重
model = get_peft_model(model, lora_config)
print("Lora applied to the model")
model.print_trainable_parameters()
```

![image-20260605110243404](https://cdn.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260605110243526.png)

## 6. 训练配置

```
# wandb 实验可视化日志
import wandb
wandb.init(project="test")

print("Config Train Arguements")

# 训练核心超参
training_arguments=TrainingArguments(
    output_dir=ouput_dir,     # LoRA权重保存目录
    per_device_train_batch_size=1,   # 每个设备的训练批次大小。这里一张显卡一次只喂 1 条数据进模型训练。
    gradient_accumulation_steps=8,   # 梯度累积步数。这里是连续累积 8 步的梯度，再更新一次模型权重。
    optim="adamw_8bit",   # 优化器。这里使用最常用的8bit 精度的 AdamW 优化器

    save_steps=500,   # 每训练多少步自动保存一次 LoRA 权重
    logging_steps=2,  # 每几步打印一次训练日志

    learning_rate=1e-4,  # 学习率。LoRA 微调常用范围：1e-4 ~ 3e-4。太大模型不收敛，太小训练太慢
    num_train_epochs=1,  # 训练几个轮次
    max_grad_norm=0.3,   # 梯度裁剪，防止梯度过大导致模型训练崩溃。大模型微调常用：0.3~1.0
    lr_scheduler_type="cosine",  # 学习率调度策略。这里使用的是余弦退火，学习率会按照余弦曲线慢慢下降。
    weight_decay=0.01,   # 权重衰减，防止模型过拟合。常用值：0.01 ~ 0.1
    warmup_ratio=0.03,   # 学习率预热比例。这里训练前 3% 的步数，学习率从 0 慢慢升到设定值，让训练初期更稳定。
    fp16=False,     # 混合精度训练。半精度（关闭）
    bf16=True,      # 混合精度训练。脑浮点精度（开启）

    report_to="wandb",  # 日志上传到wandb
)

print("SFT TRAINER START...")

# TRL库专用的有监督微调训练器
trainer=SFTTrainer(
    model=model,
    tokenizer=tokenizer,
    args=training_arguments,
    train_dataset=dataset,

    peft_config=lora_config,
    dataset_text_field="text",

    dataset_num_proc=2,  # 2进程并行做数据集tokenizer预处理，提速
    max_seq_length=2048, # 单条文本token上限2048，超长截断、不足补padding。
    packing=False,       # 关闭序列打包
)

print("Start training...")
trainer.train()
print("End training...")
```

nohup.out如下：

```
The following values were not passed to `accelerate launch` and had defaults used instead:
	`--num_processes` was set to a value of `1`
	`--num_machines` was set to a value of `1`
	`--mixed_precision` was set to a value of `'no'`
	`--dynamo_backend` was set to a value of `'no'`
To avoid this warning pass in values for each of the problematic parameters or run `accelerate config`.

Loading weights:   0%|          | 0/339 [00:00<?, ?it/s]
Loading weights:  31%|███       | 104/339 [00:00<00:00, 1032.65it/s]
Loading weights:  61%|██████▏   | 208/339 [00:00<00:00, 949.35it/s] 
Loading weights: 100%|██████████| 339/339 [00:00<00:00, 1122.88it/s]
Loadding Dataset...
数据集一共有20171组数据
数据集:Dataset({
    features: ['Question', 'Complex_CoT', 'Response'],
    num_rows: 20171
})
数据集列['Question', 'Complex_CoT', 'Response']
Formating dataset...
Dataset after formatting Dataset({
    features: ['Question', 'Complex_CoT', 'Response', 'text'],
    num_rows: 20171
})
Dataset[0] 
请根据下方指令和提供的具体问题，撰写一个恰当的回复。
在回答前，请仔细思考问题，并展现一步一步的思考过程，以确保回复的逻辑性和准确性。

### 指令:
你是一位医学专家，精通临床推理、诊断和治疗计划。
请回答下面的医学问题。

### 问题:
根据描述，一个1岁的孩子在夏季头皮出现多处小结节，长期不愈合，且现在疮大如梅，溃破流脓，口不收敛，头皮下有空洞，患处皮肤增厚。这种病症在中医中诊断为什么病？

### 回答:
<think>
这个小孩子在夏天头皮上长了些小结节，一直都没好，后来变成了脓包，流了好多脓。想想夏天那么热，可能和湿热有关。才一岁的小孩，免疫力本来就不强，夏天的湿热没准就侵袭了身体。

用中医的角度来看，出现小结节、再加上长期不愈合，这些症状让我想到了头疮。小孩子最容易得这些皮肤病，主要因为湿热在体表郁结。

但再看看，头皮下还有空洞，这可能不止是简单的头疮。看起来病情挺严重的，也许是脓肿没治好。这样的情况中医中有时候叫做禿疮或者湿疮，也可能是另一种情况。

等一下，头皮上的空洞和皮肤增厚更像是疾病已经深入到头皮下，这是不是说明有可能是流注或瘰疬？这些名字常描述头部或颈部的严重感染，特别是有化脓不愈合，又形成通道或空洞的情况。

仔细想想，我怎么感觉这些症状更贴近瘰疬的表现？尤其考虑到孩子的年纪和夏天发生的季节性因素，湿热可能是主因，但可能也有火毒或者痰湿造成的滞留。

回到基本的症状描述上看，这种长期不愈合又复杂的状况，如果结合中医更偏重的病名，是不是有可能是涉及更深层次的感染？

再考虑一下，这应该不是单纯的瘰疬，得仔细分析头皮增厚并出现空洞这样的严重症状。中医里头，这样的表现可能更符合‘蚀疮’或‘头疽’。这些病名通常描述头部严重感染后的溃烂和组织坏死。

看看季节和孩子的体质，夏天又湿又热，外邪很容易侵入头部，对孩子这么弱的免疫系统简直就是挑战。头疽这个病名听起来真是切合，因为它描述的感染严重，溃烂到出现空洞。

不过，仔细琢磨后发现，还有个病名似乎更为合适，叫做‘蝼蛄疖’，这病在中医里专指像这种严重感染并伴有深部空洞的情况。它也涵盖了化脓和皮肤增厚这些症状。

哦，该不会是夏季湿热，导致湿毒入侵，孩子的体质不能御，其病情发展成这样的感染？综合分析后我觉得‘蝼蛄疖’这个病名真是相当符合。
</think>
从中医的角度来看，你所描述的症状符合“蝼蛄疖”的病症。这种病症通常发生在头皮，表现为多处结节，溃破流脓，形成空洞，患处皮肤增厚且长期不愈合。湿热较重的夏季更容易导致这种病症的发展，特别是在免疫力较弱的儿童身上。建议结合中医的清热解毒、祛湿消肿的治疗方法进行处理，并配合专业的医疗建议进行详细诊断和治疗。
<｜end▁of▁sentence｜>
Config BitsAndBytes for int4 quantization...

Loading weights:   0%|          | 0/339 [00:00<?, ?it/s]
Loading weights:   0%|          | 1/339 [00:01<10:39,  1.89s/it]
Loading weights:   1%|          | 2/339 [00:09<29:31,  5.26s/it]/home/stw/miniconda3/envs/peft/lib/python3.10/site-packages/bitsandbytes/backends/cuda/ops.py:213: FutureWarning: _check_is_size will be removed in a future PyTorch release along with guard_size_oblivious.     Use _check(i >= 0) instead.
  torch._check_is_size(blocksize)

Loading weights:   1%|          | 4/339 [00:11<13:50,  2.48s/it]
Loading weights:   1%|▏         | 5/339 [00:12<11:35,  2.08s/it]
Loading weights:   2%|▏         | 6/339 [00:13<09:50,  1.77s/it]
Loading weights:   3%|▎         | 10/339 [00:13<03:43,  1.47it/s]
Loading weights:   4%|▎         | 12/339 [00:13<02:44,  1.98it/s]
Loading weights:   5%|▍         | 16/339 [00:15<02:27,  2.19it/s]
Loading weights:   5%|▌         | 17/339 [00:17<03:28,  1.54it/s]
Loading weights:   5%|▌         | 18/339 [00:18<04:20,  1.23it/s]
Loading weights:   6%|▋         | 22/339 [00:19<02:23,  2.20it/s]
Loading weights:   7%|▋         | 24/339 [00:19<01:58,  2.65it/s]
Loading weights:   8%|▊         | 28/339 [00:21<02:08,  2.42it/s]
Loading weights:   9%|▊         | 29/339 [00:24<03:46,  1.37it/s]
Loading weights:   9%|▉         | 30/339 [00:25<04:31,  1.14it/s]
Loading weights:  10%|█         | 34/339 [00:26<02:27,  2.06it/s]
Loading weights:  11%|█         | 36/339 [00:26<01:56,  2.59it/s]
Loading weights:  12%|█▏        | 40/339 [00:27<01:46,  2.82it/s]
Loading weights:  12%|█▏        | 41/339 [00:28<02:06,  2.36it/s]
Loading weights:  12%|█▏        | 42/339 [00:28<02:09,  2.29it/s]
Loading weights:  14%|█▎        | 46/339 [00:29<01:11,  4.08it/s]
Loading weights:  14%|█▍        | 48/339 [00:29<00:57,  5.06it/s]
Loading weights:  15%|█▌        | 52/339 [00:29<00:44,  6.49it/s]
Loading weights:  16%|█▌        | 54/339 [00:30<00:53,  5.29it/s]
Loading weights:  18%|█▊        | 62/339 [00:30<00:24, 11.20it/s]
Loading weights:  19%|█▉        | 65/339 [00:30<00:25, 10.92it/s]
Loading weights:  20%|██        | 68/339 [00:30<00:22, 12.19it/s]
Loading weights:  22%|██▏       | 76/339 [00:30<00:15, 17.27it/s]
Loading weights:  23%|██▎       | 79/339 [00:31<00:16, 15.71it/s]
Loading weights:  26%|██▌       | 88/339 [00:31<00:11, 22.53it/s]
Loading weights:  27%|██▋       | 91/339 [00:31<00:12, 19.34it/s]
Loading weights:  29%|██▉       | 100/339 [00:31<00:09, 25.90it/s]
Loading weights:  30%|███       | 103/339 [00:32<00:10, 21.94it/s]
Loading weights:  33%|███▎      | 112/339 [00:32<00:07, 28.55it/s]
Loading weights:  34%|███▍      | 116/339 [00:32<00:08, 25.46it/s]
Loading weights:  37%|███▋      | 124/339 [00:32<00:06, 30.91it/s]
Loading weights:  38%|███▊      | 128/339 [00:32<00:07, 27.57it/s]
Loading weights:  40%|████      | 136/339 [00:33<00:06, 33.13it/s]
Loading weights:  41%|████▏     | 140/339 [00:33<00:06, 28.88it/s]
Loading weights:  44%|████▎     | 148/339 [00:33<00:05, 34.79it/s]
Loading weights:  45%|████▍     | 152/339 [00:33<00:06, 30.77it/s]
Loading weights:  47%|████▋     | 160/339 [00:33<00:04, 37.01it/s]
Loading weights:  48%|████▊     | 164/339 [00:34<00:07, 23.89it/s]
Loading weights:  51%|█████     | 172/339 [00:34<00:05, 30.52it/s]
Loading weights:  52%|█████▏    | 176/339 [00:34<00:05, 28.29it/s]
Loading weights:  54%|█████▍    | 184/339 [00:34<00:04, 33.45it/s]
Loading weights:  55%|█████▌    | 188/339 [00:34<00:05, 29.65it/s]
Loading weights:  58%|█████▊    | 196/339 [00:34<00:04, 34.61it/s]
Loading weights:  59%|█████▉    | 200/339 [00:35<00:05, 27.66it/s]
Loading weights:  61%|██████▏   | 208/339 [00:35<00:03, 32.96it/s]
Loading weights:  63%|██████▎   | 212/339 [00:35<00:04, 28.06it/s]
Loading weights:  65%|██████▍   | 220/339 [00:35<00:03, 32.82it/s]
Loading weights:  66%|██████▌   | 224/339 [00:35<00:04, 28.38it/s]
Loading weights:  68%|██████▊   | 232/339 [00:36<00:03, 34.03it/s]
Loading weights:  70%|██████▉   | 236/339 [00:36<00:03, 29.36it/s]
Loading weights:  72%|███████▏  | 244/339 [00:36<00:02, 35.75it/s]
Loading weights:  73%|███████▎  | 248/339 [00:36<00:02, 31.35it/s]
Loading weights:  76%|███████▌  | 256/339 [00:36<00:02, 37.12it/s]
Loading weights:  77%|███████▋  | 260/339 [00:37<00:02, 32.22it/s]
Loading weights:  79%|███████▉  | 268/339 [00:37<00:01, 38.57it/s]
Loading weights:  81%|████████  | 273/339 [00:37<00:02, 24.02it/s]
Loading weights:  83%|████████▎ | 280/339 [00:37<00:02, 28.51it/s]
Loading weights:  84%|████████▍ | 284/339 [00:38<00:02, 24.47it/s]
Loading weights:  86%|████████▌ | 292/339 [00:38<00:01, 30.53it/s]
Loading weights:  87%|████████▋ | 296/339 [00:38<00:01, 27.77it/s]
Loading weights:  90%|████████▉ | 304/339 [00:38<00:01, 34.74it/s]
Loading weights:  91%|█████████ | 308/339 [00:38<00:01, 26.98it/s]
Loading weights:  93%|█████████▎| 316/339 [00:38<00:00, 33.28it/s]
Loading weights:  94%|█████████▍| 320/339 [00:39<00:00, 28.95it/s]
Loading weights:  97%|█████████▋| 328/339 [00:40<00:00, 13.57it/s]
Loading weights:  98%|█████████▊| 331/339 [00:40<00:00, 13.98it/s]
Loading weights: 100%|██████████| 339/339 [00:40<00:00,  8.37it/s]
wandb: [wandb.login()] Loaded credentials for https://api.wandb.ai from WANDB_API_KEY.
wandb: Currently logged in as: 3262818908 (3262818908-personal) to https://api.wandb.ai. Use `wandb login --relogin` to force relogin
wandb: setting up run awwiyurj
wandb: Tracking run with wandb version 0.27.0
wandb: Run data is saved locally in /home/stw/PEFT/wandb/run-20260605_115430-awwiyurj
wandb: Run `wandb offline` to turn off syncing.
wandb: Syncing run trim-thunder-2
wandb: ⭐️ View project at https://wandb.ai/3262818908-personal/test
wandb: 🚀 View run at https://wandb.ai/3262818908-personal/test/runs/awwiyurj
[transformers] warmup_ratio is deprecated and will be removed in v5.2. Use `warmup_steps` instead.
/home/stw/miniconda3/envs/peft/lib/python3.10/site-packages/huggingface_hub/utils/_deprecation.py:100: FutureWarning: Deprecated argument(s) used in '__init__': dataset_text_field, dataset_num_proc, max_seq_length. Will not be supported from version '0.13.0'.

Deprecated positional argument(s) used in SFTTrainer, please use the SFTConfig to set these arguments instead.
  warnings.warn(message, FutureWarning)
[transformers] warmup_ratio is deprecated and will be removed in v5.2. Use `warmup_steps` instead.
/home/stw/miniconda3/envs/peft/lib/python3.10/site-packages/trl/trainer/sft_trainer.py:300: UserWarning: You passed a `max_seq_length` argument to the SFTTrainer, the value you passed will override the one in the `SFTConfig`.
  warnings.warn(
/home/stw/miniconda3/envs/peft/lib/python3.10/site-packages/trl/trainer/sft_trainer.py:314: UserWarning: You passed a `dataset_num_proc` argument to the SFTTrainer, the value you passed will override the one in the `SFTConfig`.
  warnings.warn(
/home/stw/miniconda3/envs/peft/lib/python3.10/site-packages/trl/trainer/sft_trainer.py:328: UserWarning: You passed a `dataset_text_field` argument to the SFTTrainer, the value you passed will override the one in the `SFTConfig`.
  warnings.warn(
[transformers] The tokenizer has new PAD/BOS/EOS tokens that differ from the model config and generation config. The model config and generation config were aligned accordingly, being updated with the tokenizer's values. Updated tokens: {'bos_token_id': 151646, 'pad_token_id': 151643}.
model loaded
Qwen2ForCausalLM(
  (model): Qwen2Model(
    (embed_tokens): Embedding(152064, 3584)
    (layers): ModuleList(
      (0-27): 28 x Qwen2DecoderLayer(
        (self_attn): Qwen2Attention(
          (q_proj): Linear4bit(in_features=3584, out_features=3584, bias=True)
          (k_proj): Linear4bit(in_features=3584, out_features=512, bias=True)
          (v_proj): Linear4bit(in_features=3584, out_features=512, bias=True)
          (o_proj): Linear4bit(in_features=3584, out_features=3584, bias=False)
        )
        (mlp): Qwen2MLP(
          (gate_proj): Linear4bit(in_features=3584, out_features=18944, bias=False)
          (up_proj): Linear4bit(in_features=3584, out_features=18944, bias=False)
          (down_proj): Linear4bit(in_features=18944, out_features=3584, bias=False)
          (act_fn): SiLUActivation()
        )
        (input_layernorm): Qwen2RMSNorm((3584,), eps=1e-06)
        (post_attention_layernorm): Qwen2RMSNorm((3584,), eps=1e-06)
      )
    )
    (norm): Qwen2RMSNorm((3584,), eps=1e-06)
    (rotary_emb): Qwen2RotaryEmbedding()
  )
  (lm_head): Linear(in_features=3584, out_features=152064, bias=False)
)
Config Lora
Lora applied to the model
trainable params: 40,370,176 || all params: 7,655,986,688 || trainable%: 0.5273
start train
Config Train Arguements
SFT TRAINER START...
Start training...

  0%|          | 0/2522 [00:00<?, ?it/s]/home/stw/miniconda3/envs/peft/lib/python3.10/site-packages/torch/_dynamo/eval_frame.py:1298: UserWarning: torch.utils.checkpoint: the use_reentrant parameter should be passed explicitly. Starting in PyTorch 2.9, calling checkpoint without use_reentrant will raise an exception. use_reentrant=False is recommended, but if you need to preserve the current default behavior, you can pass use_reentrant=True. Refer to docs for more details on the differences between the two variants.
  return fn(*args, **kwargs)

  0%|          | 1/2522 [09:55<416:57:49, 595.43s/it]
  0%|          | 2/2522 [15:46<316:06:25, 451.58s/it]
                                                     

  0%|          | 2/2522 [15:46<316:06:25, 451.58s/it]
  0%|          | 3/2522 [21:25<279:57:43, 400.10s/it]
  0%|          | 4/2522 [27:20<267:21:31, 382.24s/it]
                                                     

  0%|          | 4/2522 [27:20<267:21:31, 382.24s/it]
  0%|          | 5/2522 [33:09<259:03:45, 370.53s/it]
  0%|          | 6/2522 [39:10<256:40:06, 367.25s/it]
                                                     

  0%|          | 6/2522 [39:10<256:40:06, 367.25s/it]
  0%|          | 7/2522 [44:30<245:39:45, 351.64s/it]
```

这里我的电脑配置太拉了，就不等它训练完了，太慢了

# 三、分布式训练

上面我们说的都是单GPU的情况，如果想去做多卡训练，也很简单：

1.在项目根目录新建ds_zero2.json，怎么写可以参考：https://huggingface.co/docs/accelerate/usage_guides/deepspeed。

也可以参考我这个：

```
{
    "bf16": {
        "enabled": true
    },
    "zero_optimization": {
        "stage": 2,
        "offload_optimizer": {
            "device": "none"
        },
        "allgather_partitions": true,
        "allgather_bucket_size": 5e8,
        "overlap_comm": true,
        "reduce_scatter": true,
        "reduce_bucket_size": 5e8,
        "contiguous_gradients": true
    },
    "gradient_accumulation_steps": "auto",
    "gradient_clipping": "auto",
    "train_batch_size": "auto",
    "train_micro_batch_size_per_gpu": "auto"
}
```

2.TrainingArguments 中加一行 `deepspeed="./ds_zero2.json"`

3.运行（这里以2卡为例）：`CUDA_VISIBLE_DEVICES=0,1 accelerate launch --num_processes=2 test.py` 

# 四、LlamaFactory框架

最后推荐一下LlamaFactory这个框架吧，这个框架可以实现模型训练的可视化操作，我觉得无论是新手还是老手使用这个框架去进行工作都是更方便的：

![image-20260605131250424](https://cdn.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260605131250840.png)

前面的学习可以说是为了夯实基本功，但真正干活的时候还是怎么方便怎么来。

