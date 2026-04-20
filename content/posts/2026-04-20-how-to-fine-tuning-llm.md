---
title: "LLM 微调实战指南：从原理到落地的完整路径"
date: "2026-04-20T10:00:00+08:00"
tags: ["ai", "llm"]
toc: true
---


## 一、是否需要微调？

很多工程师一遇到大模型效果不好，第一反应是"我们去微调一下"。这个直觉并不总是错的，但它跳过了一个至关重要的问题：**微调真的是你现在需要的吗？**

微调不是万能药。它有成本（数据、算力、人力、维护），有风险（过拟合、灾难性遗忘、对齐破坏），有边界（不能弥补知识截止日期的缺陷，不能替代架构层面的问题）。在你投入资源之前，先把决策做对，比把微调做对更重要。

### 1.1 预训练(Pre-training)模型的能力边界

预训练大语言模型（LLM）是在海量文本上通过 Next Token Prediction 训练出来的，它内化了：

- **语言能力**：语法、语义
- **世界知识**：训练数据截止日前的大量事实
- **推理能力**：链式推理、类比、归纳
- **泛化能力**：zero-shot / few-shot 泛化

但它的边界同样清晰：

| 类型 | 描述 | 能否靠 Prompt 解决？ |
|------|------|----------------------|
| 知识截止 | 不知道训练后发生的事 | 需要 RAG 或工具调用 |
| 领域深度不足 | 医疗/法律细分知识密度低 | 部分可以，复杂情况需微调 |
| 语言风格统一 | 品牌语气、客服话术 | Prompt 可以但难以持续一致 |
| 私有数据注入 | 公司内部文档 | 需要 RAG 或微调 |
| 高并发低延迟 | 推理成本敏感 | 需要小模型微调 |

**核心判断原则**：如果你的问题本质是"模型不知道某件事"，RAG 是更优先的选择；如果是"模型知道但表达/行为方式不对"，微调才是你要的。

### 1.2 三种主流方案对比：Prompt Engineering vs RAG vs Fine-tuning

这三种方案不是竞争关系，而是递进关系。实践中应该先穷尽轻量方案，再考虑重量级方案。

```
Prompt Engineering  →  RAG  →  Fine-tuning
     最轻量                          最重量
     最快速                          最慢速
     最易维护                        最难维护
     效果有上限                      效果上限最高
```

**Prompt Engineering**

- 零成本，快速迭代
- 无需数据，无需训练
- 受 context window 限制
- 无法改变模型底层行为
- 可被用户 jailbreak

**RAG（检索增强生成）**

- 知识可实时更新
- 可溯源，减少幻觉
- 无需重训模型
- 检索质量决定上限
- 多跳推理效果一般
- 系统复杂度增加

**Fine-tuning**

- 改变模型底层行为
- 推理时无需外部系统
- 可压缩到小模型降低成本
- 需要高质量数据
- 训练成本高
- 知识更新困难

> **决策树**：先用 Prompt，不行再加 RAG，仍然不行再考虑微调。三者可以叠加使用（微调后的模型 + RAG 是很常见的生产方案）。

### 1.3 典型适用场景判断

**什么时候必须微调？**

#### 垂直领域知识（医疗 / 法律等）

通用大模型在医疗、法律等高度专业化领域面临两类问题：

1. **知识密度不足**：医学影像报告解读、药物相互作用判断、法律条文精确适用，这些知识在预训练语料中比例极低。
2. **表达规范不符**：医院的 SOAP 格式病历、律所的裁判文书格式，通用模型根本不知道。

这类场景通常需要"领域知识注入"+ "格式行为对齐"双重微调。

#### 风格控制（客服 / 写作）

如果你需要：
- 品牌语气（某品牌总是热情、简洁、使用特定词汇）
- 长期一致的角色扮演（智能客服"小美"）
- 特定写作风格（某博主的行文方式）

Prompt 可以在单次对话中做到，但在大规模生产环境下，Prompt 的稳定性远不如微调后的模型行为内化。

#### 结构化输出（JSON / 代码）

虽然现在大部分主流模型都支持 Function Calling 和 JSON Mode，但在：
- 超复杂自定义 Schema
- 私有 DSL（领域特定语言）
- 代码生成（特定框架/风格/API）

等场景下，微调后的稳定性和准确率仍然显著高于 Prompt 方案。

### 1.4 微调的成本与收益评估

在做最终决定前，建议填写这张成本评估表：

| 成本项 | 估算方式 | 备注 |
|--------|----------|------|
| **数据成本** | 人工标注 × 条数 × 单价 | 高质量数据是最贵的 |
| **算力成本** | GPU 小时 × 单价 | A100 约 $2-4/hr |
| **工程成本** | 工程师人天 × 日薪 | 通常被低估 |
| **维护成本** | 数据漂移后的再训练频率 | 长期最大成本 |
| **机会成本** | 用同等资源能做多少 RAG 迭代 | 重要但常被忽略 |

**收益量化**：

- 任务准确率提升（A/B 测试）
- 推理成本降低（小模型替代大模型）
- 延迟改善（去掉 RAG 检索链路）
- 用户满意度提升（NPS）

## 二、什么是模型微调(LLM Fine-tuning)？

很多工程师能跑通微调流程，但说不清楚微调到底改变了什么。这一章的目标是让你真正理解本质，而不只是知道怎么调用三方库。

### 2.1 从迁移学习到大模型微调

微调（Fine-tuning）的思想来源于迁移学习（Transfer Learning），这个概念在深度学习早期就已成熟。

**CV 时代的迁移学习**：ResNet 在 ImageNet 上预训练，提取到"边缘、纹理、形状"等通用视觉特征。把这个模型的前几层特征冻结，只训练后几层，就能在小数据集（猫狗分类、医学影像）上快速收敛。

**大模型时代的变化**：

- 预训练规模从百万参数到万亿参数
- 预训练数据从百万图片到万亿 Token
- 预训练任务从分类到语言建模（Next Token Prediction）
- 微调目标从"适应新任务"变为"对齐人类意图"

核心逻辑一脉相承：**预训练模型已经学到了大量通用能力，微调只是在此基础上做"最后一公里"的调整。**

### 2.2 微调本质：让模型"重新分布概率空间"

从数学角度理解微调，是理解它能做什么、不能做什么的关键。

语言模型本质上是一个**条件概率分布**：

```
P(token_t | token_1, token_2, ..., token_{t-1}; θ)
```

其中 θ 是模型参数。预训练把这个分布调整到"互联网文本的统计规律"。

**微调做的事情**：

在微调数据集上继续优化 θ，让模型对特定输入的输出概率分布向期望方向偏移。

举个例子：
- 预训练后，给定"患者出现发热、咳嗽"，模型可能以相对均匀的概率生成各种续写
- 微调医疗数据后，模型会更高概率地输出规范的临床问诊格式

**这带来一个重要推论**：微调不能凭空创造模型不懂的知识，它只能重分配已有知识的概率权重。如果预训练语料中某个领域的知识极度稀疏，微调的效果也会有限。

**"注入新知识"的正确方式**：通过继续预训练（Continued Pretraining）在大量领域文档上训练，让模型先"见过"这些知识，再做 SFT 对齐行为。

### 2.3 SFT 与 Instruction Tuning 本质区别

这两个词经常混用，但它们的侧重点有细微差别。

**SFT（Supervised Fine-Tuning，监督微调）**

广义上的监督微调，指在带标签的输入-输出对上训练。损失函数通常是交叉熵：

```
L = -∑ log P(y_i | x, y_{<i}; θ)
```

SFT 是一个通用概念，可以用于分类、生成、翻译等各种任务。

**Instruction Tuning（指令微调）**

是 SFT 的一种特殊形式，专门针对"指令-回复"格式的数据。其核心论文是 FLAN（Wei et al., 2021）和 InstructGPT（Ouyang et al., 2022）。

Instruction Tuning 的数据格式：
```
[INST] 请将以下文本翻译成英文：{text} [/INST] {translation}
```

它解决的本质问题是：预训练模型擅长"续写"，但不擅长"听指令"。Instruction Tuning 把模型从"自动完成机"变成"指令执行器"。

**区别总结**：
- SFT = 广义监督微调
- Instruction Tuning = 以"遵循指令"为目标的特定 SFT
- 所有 Instruction Tuning 都是 SFT，但 SFT 不一定是 Instruction Tuning

### 2.4 微调 vs 继续预训练（Continued Pretraining）

这是两个经常被混淆的概念。

| 维度 | 微调（SFT）| 继续预训练 |
|------|-----------|------------|
| **目标** | 对齐行为/风格/格式 | 注入领域知识 |
| **数据格式** | 指令-回复对 | 无标注原始文本 |
| **数据量** | 千-万级别 | 百万-十亿 Token |
| **训练成本** | 相对低 | 极高 |
| **改变什么** | 行为模式 | 知识表示 |
| **典型场景** | 客服机器人、代码助手 | 医疗/法律专有模型 |

**最佳实践**：对于需要深度领域能力的场景，标准流程是：

```
Base Model → Continued Pretraining（领域语料） → SFT（指令对齐） → [RLHF/DPO]
```

## 三、微调技术全景图：该选哪一条路？

### 3.1 全量微调（Full Fine-tuning）：能力强但成本高

全量微调更新模型的所有参数。对于 7B 模型，参数量约 70 亿，以 BF16 精度存储需要 ~14GB 显存，加上梯度和优化器状态，实际需要 **~60-80GB 显存**（使用 AdamW 的情况下）。

**Adam 优化器的显存分解**：
```
参数: 2 bytes × N
梯度: 2 bytes × N  
一阶矩: 4 bytes × N
二阶矩: 4 bytes × N
总计: ~12 bytes × N
```

**适用场景**：
- 有充足算力（多卡 A100/H100 集群）
- 需要最高的微调效果
- 领域分布与预训练差异极大

**实际情况**：大多数公司和个人开发者没有这个条件，这也是 PEFT 方法兴起的根本原因。

### 3.2 参数高效微调（PEFT）全家桶

PEFT（Parameter-Efficient Fine-Tuning）的核心思想：**只更新少量参数，达到接近全量微调的效果。**

#### LoRA（Low-Rank Adaptation）

Hu et al., 2021 提出。核心思想：

> 大模型在预训练后，权重矩阵的内在维度（intrinsic rank）很低。我们不需要更新整个权重矩阵，只需要训练一个低秩分解矩阵。

数学表达：

```
W' = W + ΔW = W + B × A

其中：
W ∈ R^{d×k}（原始权重，冻结）
A ∈ R^{r×k}（低秩矩阵，可训练，r << d）
B ∈ R^{d×r}（低秩矩阵，可训练）
```

可训练参数量：`r × (d + k)` vs 全量的 `d × k`，当 `r=8, d=k=4096` 时，参数量减少 **256 倍**。

#### Adapter

Houlsby et al., 2019 提出。在 Transformer 层内插入小型 Adapter 模块（两个线性层 + 非线性激活），只训练 Adapter 参数。

```
原始层输出 → Adapter（down-project → 激活 → up-project + 残差） → 下一层
```

**LoRA vs Adapter 的核心区别**：Adapter 在推理时引入额外计算（增加延迟），而 LoRA 可以通过权重合并（merge）在推理时无额外开销。这是 LoRA 成为主流的重要原因。

#### Prefix Tuning / Prompt Tuning

在输入的 embedding 层或每层的 key/value 前添加可学习的"软提示"（soft prompt）：

```
[prefix tokens（可学习）] + [真实输入 tokens]
```

**优点**：参数极少，模型完全冻结  
**缺点**：效果不如 LoRA，调参困难，长任务表现不稳定

### 3.3 LoRA 深入：为什么低秩可以 work？

这是很多文章跳过的核心问题，理解它才能知道 LoRA 的边界在哪里。

**直觉解释**：

想象模型的权重更新矩阵 ΔW。全量微调时，ΔW 可以是任意形状。但实验发现，训练后的 ΔW 的**有效秩（effective rank）非常低**——大部分"信息"集中在少数几个方向上。

这类似于 PCA：虽然数据在高维空间，但大部分方差可以用少数主成分解释。

**数学解释（奇异值分解视角）**：

对权重更新 ΔW 做 SVD 分解：

```
ΔW = U × Σ × V^T
```

实验表明，Σ 的奇异值分布极度不均匀——前几个奇异值远大于其余。这说明 ΔW 的"信息"确实集中在低秩子空间中。

LoRA 本质上是在说：**我用 B×A 来近似这个低秩子空间，而不是存储完整的 ΔW。**

**LoRA 的边界**：
- 当任务需要的权重更新本身就是高秩时（如极大领域偏移），低秩近似会损失信息
- rank 的选择需要根据任务复杂度调整（通常 r=8~64）

### 3.4 QLoRA：如何在消费级 GPU 上训练大模型

Dettmers et al., 2023 提出的 QLoRA 是让大模型微调"飞入寻常百姓家"的关键技术。

**三个核心创新**：

**① 4-bit NormalFloat（NF4）量化**

设计了一种针对正态分布的 4-bit 量化格式（大模型权重近似正态分布），相比普通 INT4 量化精度损失更小。

**② 双量化（Double Quantization）**

对量化常数本身再做量化，每个参数平均节省额外 0.37 bits。

**③ 分页优化器（Paged Optimizers）**

利用 CPU/GPU 统一内存管理，在 GPU 显存压力大时将优化器状态分页到 CPU 内存，避免 OOM。

**实际效果**：65B 模型可在单张 48GB A100 上训练，7B 模型可在 **单张消费级 24GB GPU（RTX 3090/4090）** 上微调。

```python
# QLoRA 典型配置
from transformers import BitsAndBytesConfig

bnb_config = BitsAndBytesConfig(
    load_in_4bit=True,
    bnb_4bit_use_double_quant=True,
    bnb_4bit_quant_type="nf4",
    bnb_4bit_compute_dtype=torch.bfloat16
)
```

### 3.5 技术选型指南

| 方法 | 显存需求 | 训练速度 | 推理开销 | 效果 | 适用场景 |
|------|---------|---------|---------|------|---------|
| 全量微调 | ⭐⭐⭐⭐⭐ | 慢 | 无额外 | 最好 | 土豪 + 极端场景 |
| LoRA | ⭐⭐ | 快 | 无额外（merge后）| 很好 | **主流推荐** |
| QLoRA | ⭐ | 较慢 | 无额外（merge后）| 好 | 消费级 GPU |
| Adapter | ⭐⭐ | 快 | 有轻微延迟 | 较好 | 多任务切换 |
| Prefix Tuning | ⭐ | 最快 | 有轻微影响 | 一般 | 极度资源受限 |

**选型决策树**：

```
有多卡 A100+？
  → 是：全量微调（对效果要求极高） or LoRA（平衡效果与成本）
  → 否：单卡显存 >= 24GB？
         → 是：LoRA
         → 否：QLoRA
```

## 四、数据：决定微调上限的核心变量

> "数据是微调效果的天花板，训练是逼近天花板的过程。"

### 4.1 数据格式设计

数据格式不只是格式，它决定了模型学习的"信号形态"。

#### 指令式（Alpaca 格式）

由 Stanford Alpaca 推广的格式，适合单轮问答：

```json
{
  "instruction": "将以下句子翻译成英文",
  "input": "今天天气真好",
  "output": "The weather is really nice today"
}
```

**适用**：单轮问答、分类、摘要、翻译  
**局限**：无法建模多轮对话上下文

#### 对话式（ShareGPT 格式）

由 ShareGPT 数据集推广，适合多轮对话：

```json
{
  "conversations": [
    {"from": "human", "value": "帮我分析一下这份合同的风险"},
    {"from": "gpt", "value": "好的，我来分析..."},
    {"from": "human", "value": "第三条款有问题吗？"},
    {"from": "gpt", "value": "第三条款存在以下风险..."}
  ]
}
```

**适用**：对话系统、客服、助手类应用

#### 任务型（自定义 Schema）

针对高度结构化输出的场景自定义格式：

```json
{
  "system": "你是一个医学信息提取助手，输出必须是合法 JSON",
  "user": "从以下文本中提取患者信息：{text}",
  "assistant": "{\"name\": \"张三\", \"age\": 45, \"diagnosis\": \"2型糖尿病\"}"
}
```

**关键原则**：格式设计要与推理时的使用方式完全一致。训练和推理的 prompt 格式不一致是导致微调效果差的首要原因之一。

### 4.2 高质量数据的标准

**信息密度**

每条数据应该包含有价值的"新信息"。以下是两条质量差异极大的示例：

低质量：
```
Q: 什么是糖尿病？
A: 糖尿病是一种慢性疾病，需要注意饮食。
```

高质量：
```
Q: 2型糖尿病的诊断标准是什么？
A: 根据WHO 2023诊断标准，满足以下任一条件即可诊断：
   1. 空腹血糖（FPG）≥7.0 mmol/L（至少两次）
   2. 75g口服葡萄糖耐量试验（OGTT）2小时血糖≥11.1 mmol/L
   3. 随机血糖≥11.1 mmol/L伴有高血糖症状
   4. HbA1c≥6.5%（标准化检测方法）
```

**一致性**

同类问题的回答方式应该一致：
- 语言风格统一
- 格式模板一致
- 知识来源一致（不同标注者用不同教材）

一致性问题是团队标注时最容易忽视的坑。建议：制定详细的标注指南，并定期做 Inter-Annotator Agreement（IAA）检测。

**无歧义**

每条数据的"正确答案"应该是清晰的、无争议的。医学/法律领域的"灰区"问题需要特别谨慎——不如不加入数据集，也不要加入有争议的回答。

### 4.3 数据清洗与过滤策略

**去重**：使用 MinHash LSH 对相似数据去重，避免模型对某种模式过度拟合。

```python
# 使用 datasketch 进行 MinHash 去重
from datasketch import MinHash, MinHashLSH

lsh = MinHashLSH(threshold=0.8, num_perm=128)
# 对每条数据生成 MinHash，相似度 > 0.8 的认为是重复
```

**质量过滤**：

1. **基于规则**：过滤过短（< 50 tokens）、过长（> 4096 tokens）、乱码、HTML 标签残留的数据
2. **基于模型打分**：用一个较好的模型（如 GPT-4o）对数据质量打分，过滤低分数据
3. **基于困惑度（Perplexity）过滤**：用预训练模型计算困惑度，过滤困惑度异常高（乱码）或异常低（套话）的数据

**有毒内容过滤**：使用 Perspective API 或专用分类器过滤有害内容，避免微调后模型产生有害输出。

### 4.4 小数据场景：如何用 1k 数据调出效果

1000 条数据完全可以调出有用的模型，前提是：

**1. 聚焦单一任务**：1000 条数据覆盖 10 个任务，等于每个任务只有 100 条。聚焦 1-2 个核心任务。

**2. 数据质量远比数量重要**：用 1000 条精心设计的数据，远胜 10000 条低质量数据。每一条都应该是"典型示范"。

**3. 覆盖边界情况**：不要让 1000 条数据都是"普通情况"。系统性地收集边界 case、困难 case、反例。

**4. 用更强的 Base Model**：在同等数据量下，更强的 Base Model 起点更高，微调后效果也更好。

**5. 考虑数据增强**：
   - 用 GPT-4 对种子数据进行扩写
   - 用模板生成变体
   - 中英文互译扩充

**6. 使用更小的 r（LoRA rank）**：小数据容易过拟合，小 rank 有正则化作用。

### 4.5 多任务混合训练：比例与 Curriculum Learning

当你有多个任务需要同时微调时，任务比例是关键超参数。

**经验原则**：

- **按任务重要性分配比例**：核心任务占更高比例
- **平衡大小数据集**：对大数据集采样、对小数据集上采样，避免小任务被大任务淹没
- **避免极端比例**：某个任务占比 < 5% 时，模型可能几乎学不到该任务

```python
# 典型混合比例示例
datasets = {
    "core_task": (dataset_A, 0.5),     # 核心任务 50%
    "secondary_task": (dataset_B, 0.3), # 次要任务 30%
    "format_task": (dataset_C, 0.2),   # 格式任务 20%
}
```

**Curriculum Learning（课程学习）**：

模拟人类学习"由易到难"的过程：

1. 先用简单、干净的数据训练（前 30% epochs）
2. 逐步引入复杂、困难的数据（中间 40% epochs）
3. 最后用最困难的边界 case 精调（最后 30% epochs）

实践中，Curriculum Learning 在数据分布差异大的场景中效果显著。

### 4.6 行业数据处理要点

**医疗领域**：
- 必须去除 PHI（Protected Health Information）：姓名、身份证、医院名等
- 用 ICD-10/SNOMED CT 等标准术语替换自由文本诊断
- 注意中西医术语混用问题

**金融领域**：
- 时间戳敏感：过时的财务数据可能比没有数据更危险
- 监管文本需特别小心：法规变更频繁，必须标注数据来源时间

**法律领域**：
- 司法辖区区别对待：大陆法系 vs 普通法系、各省地方性法规
- 判例时效：被推翻的判决不能作为正面示例
- 必须区分"陈述法律事实"和"提供法律建议"（后者有执照要求）

## 五、实战：从 0 到 1 完成一次微调

### 5.1 硬件与成本估算

| 模型规模 | 方法 | 最低显存 | 推荐硬件 | 云成本估算/小时 |
|---------|------|---------|---------|----------------|
| 7B | QLoRA | 12GB | RTX 3080 / A10 | $0.5-1.5 |
| 7B | LoRA | 20GB | RTX 4090 / A10G | $1-2 |
| 13B | QLoRA | 16GB | RTX 4090 / A10G | $1-2 |
| 13B | LoRA | 40GB | A100 40G | $2-4 |
| 70B | QLoRA | 48GB | A100 80G | $3-5 |
| 70B | LoRA | 160GB+ | 4× A100 80G | $12-20 |

**典型微调成本**（7B 模型，5000 条数据，LoRA）：
- 训练时间：约 1-3 小时
- 云成本：$2-10
- 调参迭代：× 5-10 次 = $20-100

### 5.2 技术栈选择

**推荐组合**：

```
Hugging Face Transformers     # 模型加载、训练循环
+ PEFT                        # LoRA/QLoRA 实现
+ TRL（SFTTrainer）           # 简化的 SFT 训练封装
+ Accelerate / DeepSpeed      # 多卡训练
+ wandb / tensorboard         # 训练监控
```

**Transformers**：模型生态最全，几乎所有主流模型都有支持。缺点是版本更新快，依赖复杂。

**PEFT**：Hugging Face 官方 PEFT 库，LoRA/QLoRA/Prefix Tuning 等一键配置。

**DeepSpeed**：微软开发的分布式训练框架，支持 ZeRO（Zero Redundancy Optimizer）系列显存优化策略：
- ZeRO-1：分片优化器状态
- ZeRO-2：分片梯度
- ZeRO-3：分片模型参数（最省显存，但通信开销最大）

**Accelerate**：更轻量的分布式训练封装，配置简单，适合中小规模训练。

### 5.3 微调流程

#### Step 1：选 Base Model

选型原则：
- **能力匹配**：不要选超出任务需求的模型（成本浪费）
- **License 合规**：商用场景必须检查 license（LLaMA 3.1 允许商用，部分模型不允许）
- **中文支持**：中文任务首选 Qwen2.5、Baichuan2、ChatGLM4 等
- **社区生态**：模型越流行，遇到问题越容易找到解决方案

常用 Base Model 参考：

| 场景 | 推荐模型 |
|------|---------|
| 中文通用 | Qwen2.5-7B-Instruct |
| 英文通用 | LLaMA-3.1-8B-Instruct |
| 代码 | DeepSeek-Coder-7B |
| 轻量边缘 | Qwen2.5-1.5B |

#### Step 2：构建 Dataset

```python
from datasets import Dataset

# 构建数据集
data = [
    {
        "messages": [
            {"role": "system", "content": "你是一个专业的医疗信息助手"},
            {"role": "user", "content": "2型糖尿病的诊断标准是什么？"},
            {"role": "assistant", "content": "根据最新诊断标准..."}
        ]
    }
]

dataset = Dataset.from_list(data)
```

#### Step 3：配置 LoRA

```python
from peft import LoraConfig, TaskType

lora_config = LoraConfig(
    task_type=TaskType.CAUSAL_LM,
    r=16,                          # rank
    lora_alpha=32,                 # 缩放系数，通常 = 2×r
    target_modules=["q_proj", "v_proj", "k_proj", "o_proj"],
    lora_dropout=0.05,
    bias="none",
)
```

**target_modules 选择**：
- 最小配置：`["q_proj", "v_proj"]`（最省显存）
- 标准配置：`["q_proj", "k_proj", "v_proj", "o_proj"]`（推荐）
- 全量注意力：加上 `["gate_proj", "up_proj", "down_proj"]`（FFN 层，效果更好但显存更多）

#### Step 4：启动训练

```python
from trl import SFTTrainer
from transformers import TrainingArguments

training_args = TrainingArguments(
    output_dir="./output",
    num_train_epochs=3,
    per_device_train_batch_size=4,
    gradient_accumulation_steps=4,   # 等效 batch_size = 16
    warmup_steps=100,
    learning_rate=2e-4,
    bf16=True,                        # 使用 BF16 精度
    logging_steps=10,
    save_strategy="epoch",
    evaluation_strategy="epoch",
    report_to="wandb",
)

trainer = SFTTrainer(
    model=model,
    args=training_args,
    train_dataset=train_dataset,
    eval_dataset=eval_dataset,
    peft_config=lora_config,
    max_seq_length=2048,
)

trainer.train()
```

### 5.4 核心超参数解释

| 超参数 | 典型值 | 调参逻辑 |
|--------|--------|---------|
| `learning_rate` | 1e-4 ~ 5e-4 | 太大→不稳定，太小→收敛慢。从 2e-4 开始 |
| `num_train_epochs` | 1-5 | 数据越多 epochs 越少，小数据可以 3-5 |
| `batch_size`（等效）| 16-128 | 越大越稳定，受显存限制用梯度累积补偿 |
| `warmup_ratio` | 0.03-0.1 | 防止初期 loss spike，从 0.05 开始 |
| `lora_r` | 8-64 | 任务越复杂 r 越大，小数据用小 r 防过拟合 |
| `lora_alpha` | = 2×r | 基本上不用单独调 |
| `lora_dropout` | 0.05-0.1 | 小数据适当增大防过拟合 |
| `max_seq_length` | 1024-4096 | 按实际数据长度分布选择，过大浪费显存 |

**调参逻辑的核心原则**：

1. **先固定大部分参数，只调 learning_rate 和 r**
2. **关注 train loss 和 eval loss 的差距**：差距扩大 = 过拟合
3. **使用 learning rate finder**（如 `trainer.find_lr()`）自动搜索起始 lr

### 5.5 训练监控与 Debug

**Loss 不下降怎么办？**

| 现象 | 可能原因 | 排查方法 |
|------|---------|---------|
| Loss 从一开始就不降 | LR 太小 / 数据格式错误 | 检查几条数据的 tokenized 结果 |
| Loss 下降后震荡 | LR 太大 | 降低 10 倍再试 |
| Loss 降到某值不动 | 数据集太小 / 模型表达能力不足 | 加数据 / 换更大的 r |
| Loss 先降后快速升 | 过拟合 | 减小 epoch / 增加 dropout |
| Loss 变 NaN | 数值溢出 | 检查是否有无效数据、使用 bf16 替代 fp16 |

**过拟合识别**：

```
# 健康状态
Train Loss: 1.2 → 0.8 → 0.6
Eval Loss:  1.3 → 0.9 → 0.7   两者同步下降

# 过拟合
Train Loss: 1.2 → 0.6 → 0.3 → 0.1
Eval Loss:  1.3 → 0.9 → 1.1 → 1.5   Eval Loss 反弹
```

监控工具：**Weights & Biases（W&B）** 是训练监控的行业标准，强烈推荐。

## 六、模型评估：如何判断"真的变强了"

> 评估是微调中最容易被忽视、也最容易出错的环节。很多"微调成功"的模型，只是在特定测试集上看起来好，实际部署后暴露各种问题。

### 6.1 自动评估指标（优缺点分析）

**Perplexity（困惑度）**

```
PPL = exp(-1/N × ∑ log P(token_i | context))
```

- **含义**：模型对测试集的"惊讶程度"，越低越好
- **局限**：
  - 只衡量语言流畅性，不衡量回答正确性
  - 模型可以通过套话降低 PPL 而不提供有用信息
  - 不同模型的 PPL 不可直接比较（tokenizer 不同）
- **适用**：监控训练稳定性，不适合作为主要评估指标

**BLEU / ROUGE**

- **BLEU**：测量 n-gram 精确匹配（常用于翻译）
- **ROUGE-L**：测量最长公共子序列（常用于摘要）
- **局限**：
  - 语义正确但表达不同会被严重低估
  - `"患者血糖偏高"` 和 `"患者血糖异常升高"` 语义几乎相同，BLEU 差异极大
  - 在开放性对话任务上基本失效
- **适用**：翻译、摘要等有标准参考答案的任务

### 6.2 大模型评估的正确方式

**LLM-as-a-Judge**

用强大的 LLM（如 GPT-4o、Claude）作为裁判，对模型输出进行打分：

```python
judge_prompt = """
请评估以下模型回答的质量，从1-5分打分并给出理由：

问题：{question}
参考答案：{reference}
模型回答：{response}

评分维度：
1. 准确性（是否与参考答案一致）
2. 完整性（是否涵盖关键信息）
3. 流畅性（语言是否自然）
4. 安全性（是否有有害内容）

请以 JSON 格式输出：{{"score": X, "reasoning": "..."}}
"""
```

**注意 LLM-as-a-Judge 的偏差**：
- 偏好更长的回答（verbosity bias）
- 偏好与自己生成风格相似的回答（self-enhancement bias）
- 解决方案：使用多个不同 LLM 打分取平均，或设计对抗性测试

**人工评测设计**

人工评测不是"让几个人看一下"，而是需要：

1. **设计评分 Rubric**：明确每个分数对应的标准
2. **标注者培训**：统一理解，减少主观差异
3. **Blind 评测**：标注者不知道哪个是微调前/后的模型
4. **计算 IAA**：Cohen's Kappa > 0.6 才算可信的人工评测
5. **样本量**：至少 100-200 条，覆盖不同子任务

### 6.3 Benchmark 使用

**C-Eval / CMMLU**：中文综合能力基准测试，覆盖 52 个学科领域。

**注意事项**：
- Benchmark 只衡量特定维度，不能代表实际任务效果
- 高 Benchmark 分数不等于业务效果好（数据污染、分布不匹配）
- 建议自建"业务 Benchmark"：从真实用户问题中抽样，比任何公开 Benchmark 都有价值

### 6.4 常见失败模式分析

**① 灾难性遗忘（Catastrophic Forgetting）**

微调后，模型在新任务上变好，但在原有通用能力上显著退化。

- **表现**：以前能做的翻译、写作变差了
- **原因**：微调数据分布与预训练分布差异太大，覆盖了原有知识
- **解决方案**：
  - 混合少量通用数据（General Replay）
  - 使用 EWC（Elastic Weight Consolidation）保护重要参数
  - 降低学习率（对原有参数更温和）
  - LoRA 天然对遗忘有一定防护（原始权重冻结）

**② 幻觉增强（Hallucination Amplification）**

微调后模型更"自信"，但错误更多了。

- **表现**：模型生成听起来专业但实际错误的回答
- **原因**：微调数据本身有错误，或 SFT 让模型学会了"强行给答案"
- **解决方案**：
  - 严格过滤训练数据中的错误
  - 加入拒绝回答的训练数据（"我不确定"也是正确答案）
  - 使用 DPO/RLHF 对齐

**③ 模式坍塌（Pattern Collapse / 复读机）**

模型陷入重复输出同一段文本的循环。

- **表现**：`"好的，我来帮您解答。好的，我来帮您解答。好的，我来帮您解答..."`
- **原因**：
  - 学习率过高导致训练不稳定
  - 重复性高的训练数据（每条都以固定套话开头）
  - Repetition Penalty 在推理时未设置
- **解决方案**：
  - 检查数据多样性
  - 推理时设置 `repetition_penalty=1.1-1.3`
  - 降低学习率

## 七、部署与推理优化：让模型真正可用

### 7.1 LoRA 权重合并 vs 动态加载

微调完成后，你会得到 Base Model + LoRA Adapter 两部分。有两种使用方式：

**权重合并（Merge）**

将 ΔW = B×A 与原始权重 W 合并：`W' = W + B×A`

```python
from peft import PeftModel

# 加载 base model 和 adapter
model = AutoModelForCausalLM.from_pretrained(base_model_path)
model = PeftModel.from_pretrained(model, adapter_path)

# 合并
merged_model = model.merge_and_unload()
merged_model.save_pretrained("./merged_model")
```

- 推理速度与原始模型完全相同
- 不需要 PEFT 库推理
- 每个任务需要单独的完整模型（多任务场景成本高）

**动态加载（Dynamic Loading）**

保持 Base Model 不变，为不同任务加载不同 Adapter。

- 多任务共享 Base Model（节省显存）
- 动态切换任务无需重加载整个模型
- 推理时有轻微额外开销
- 适用工具：**PEFT + vLLM LoRA 动态加载**

### 7.2 模型量化方案对比

量化（Quantization）是将模型权重从高精度（FP16/BF16）压缩到低精度（INT8/INT4），以减少显存占用和推理成本。

| 方案 | 精度损失 | 压缩率 | 速度提升 | 推荐场景 |
|------|---------|-------|---------|---------|
| BF16（无量化）| 无 | 1× | 基准 | 精度要求最高 |
| INT8（LLM.int8）| 很小 | 2× | 1.2-1.5× | 平衡精度与成本 |
| GPTQ INT4 | 小 | 4× | 2-3× | **主流推荐** |
| AWQ INT4 | 很小 | 4× | 2-3× | 精度敏感场景 |
| GGUF INT4 | 小 | 4× | 2-3× | CPU 推理 / 本地部署 |

**GPTQ vs AWQ**：

- **GPTQ**：基于二阶优化的量化，速度快但对异常值处理弱
- **AWQ**（Activation-aware Weight Quantization）：保护重要权重（根据激活值识别），量化精度更高

实践中，AWQ 在大多数任务上略优于 GPTQ，但速度相当。

```python
# 使用 AutoAWQ 量化
from awq import AutoAWQForCausalLM

model = AutoAWQForCausalLM.from_pretrained(model_path)
quant_config = {"zero_point": True, "q_group_size": 128, "w_bit": 4}
model.quantize(tokenizer, quant_config=quant_config)
model.save_quantized("./quantized_model")
```

### 7.3 推理框架选择

**vLLM**

- PagedAttention 技术，显著提升吞吐量（相比 Transformers 可提升 10-20×）
- 支持 LoRA 动态加载
- OpenAI 兼容 API
- 推荐场景：**在线推理服务，高并发**

```bash
python -m vllm.entrypoints.openai.api_server \
    --model ./merged_model \
    --port 8000 \
    --gpu-memory-utilization 0.9
```

**TensorRT-LLM**

- 英伟达官方推理优化框架
- 极致性能，但编译时间长，适配成本高
- 推荐场景：**性能要求极高，用英伟达 GPU，愿意付工程成本**

**TGI（Text Generation Inference）**

- Hugging Face 官方推理框架
- 对 HF 模型生态支持最好，部署最简单
- 推荐场景：**快速上线，HF 模型，工程资源有限**

**框架选型总结**：

```
需要极致性能（延迟 / 吞吐）→ vLLM（优先）或 TensorRT-LLM
需要快速上线 → TGI
CPU 推理 / 本地 → llama.cpp + GGUF
```

### 7.4 服务化架构设计

**典型生产架构**：

```
用户请求
    ↓
负载均衡（Nginx / K8s Ingress）
    ↓
推理服务（vLLM × N 副本）
    ↓
模型缓存层（KV Cache 共享）
    ↓
日志 & 监控（Prometheus + Grafana）
```

**关键优化点**：

**并发优化**：
- 使用 `continuous batching`（vLLM 默认支持）而非静态 batching
- 设置合适的 `max_num_seqs` 和 `max_model_len`

**缓存优化**：
- Prefix Cache：对于有固定 system prompt 的场景，缓存 prefix 的 KV Cache 可节省 30-50% 计算
- 语义缓存：用向量数据库缓存相似问题的答案（适合高重复性场景）

**成本优化**：
- 使用 Spot 实例（AWS Spot / 阿里云抢占式）可节省 70% 成本，需做好容错
- 根据业务流量弹性扩缩容
- 低峰期降低副本数

## 八、进阶：从"能用"到"做得好"

### 8.1 对齐问题：RLHF vs DPO

SFT 之后，模型能"回答问题"，但不一定"回答得好"——它可能不够安全、不够符合人类偏好。这就是对齐（Alignment）要解决的问题。

**RLHF（Reinforcement Learning from Human Feedback）**

三阶段流程：
1. SFT：基础指令对齐
2. 训练 Reward Model（RM）：人工标注哪个回答更好
3. PPO 强化学习：用 RM 作为奖励函数，优化语言模型

RLHF 的问题：
- **复杂**：三个模型（SFT model, RM, RL policy）同时训练
- **不稳定**：PPO 训练极难调参
- **计算成本高**：需要 online rollout

**DPO（Direct Preference Optimization）**

Rafailov et al., 2023 提出的简化方案，绕过 Reward Model，直接在偏好数据上优化：

```
数据格式：(prompt, chosen_response, rejected_response)
```

DPO 将 RLHF 目标转化为一个更简单的分类损失：让模型对 chosen 响应的概率 > rejected 响应的概率。

```python
from trl import DPOTrainer

dpo_trainer = DPOTrainer(
    model,
    ref_model,  # 参考模型（SFT 后的 checkpoint）
    args=training_args,
    beta=0.1,   # KL 惩罚系数
    train_dataset=preference_dataset,
)
```

**RLHF vs DPO 选型**：
- 小团队 / 快速迭代 → **DPO**（实现简单，效果接近）
- 大团队 / 对齐质量要求极高 → RLHF（更强但成本高）
- 数据有限 → **SimPO / ORPO**（DPO 的改进变体，不需要参考模型）

### 8.2 长上下文与位置编码扩展

标准 Transformer 的位置编码（RoPE）有上下文长度限制。扩展方法：

**YaRN（Yet another RoPE extensioN）**

- 通过插值（interpolation）而非外推扩展位置编码
- 少量微调（约 1000 steps）即可将 8K 扩展到 128K
- Mistral 的长上下文版本使用此方法

**LongLoRA**

- 用 Shift Short Attention 替代标准全注意力降低训练成本
- 结合 LoRA 在单卡上完成长上下文微调

**实践建议**：如果你的任务需要处理长文档，优先选择原生支持长上下文的 Base Model（如 Qwen2.5 系列支持 128K），而不是自己扩展。

### 8.3 多模态微调简介

视觉语言模型（VLM）的微调与纯文本模型有所不同：

**典型架构**：

```
图像 → Vision Encoder（如 CLIP ViT）→ 投影层（MLP）→ LLM
```

微调时通常：
1. 第一阶段：只微调投影层（对齐视觉和文本空间）
2. 第二阶段：端到端微调（LLM + 投影层，可选是否微调 Vision Encoder）

常用框架：**LLaVA**、**InternVL**、**Qwen-VL**

### 8.4 持续学习与避免遗忘

在业务迭代中，你可能需要不断加入新数据重新微调。这带来两个挑战：

1. **遗忘旧能力**（Catastrophic Forgetting）
2. **数据积累成本**（全量重训越来越贵）

**解决方案**：

- **Experience Replay**：每次重训时混入一小部分旧数据（5-10%）
- **EWC（Elastic Weight Consolidation）**：对重要参数施加正则化约束
- **模型合并（Model Merging）**：将新旧模型的权重取加权平均（SLERP、DARE、TIES 等算法）

模型合并是近年来的热门方向，可以无需额外训练，直接融合不同任务的 LoRA Adapter。

### 8.5 联邦学习 & 隐私场景

某些场景（医疗、金融）的数据无法集中，需要隐私计算：

**联邦学习（Federated Learning）**：

- 各参与方在本地训练，只上传梯度/参数更新，不上传原始数据
- 框架：Flower（flwr）、PySyft
- 挑战：通信成本高，梯度攻击风险（梯度可被逆推出原始数据）

**差分隐私（Differential Privacy）**：

在梯度中加入可校准的噪声，防止模型记忆训练数据：

```python
# 使用 opacus 实现差分隐私训练
from opacus import PrivacyEngine

privacy_engine = PrivacyEngine()
model, optimizer, data_loader = privacy_engine.make_private_with_epsilon(
    module=model,
    optimizer=optimizer,
    data_loader=data_loader,
    epochs=3,
    target_epsilon=8.0,   # 隐私预算
    target_delta=1e-5,
    max_grad_norm=1.0,
)
```

## 九、完整案例：把流程跑通

### 9.1 案例一：垂直领域问答系统（医疗）

**需求背景**：某三甲医院希望构建一个面向患者的医疗知识问答助手，要求：
- 能够准确回答常见疾病的症状、诊断、治疗问题
- 必须使用规范的医学术语
- 不能给出具体用药建议（避免法律风险）
- 遇到超出能力范围的问题，主动建议就医

**技术方案**：

```
Base Model: Qwen2.5-7B-Instruct（中文能力强）
微调方法: QLoRA（单卡 RTX 4090 可跑）
数据来源: 
  - 医学教材 QA 对（3000 条）
  - 医院 FAQ 数据（1500 条）
  - 拒绝回答示例（500 条）
总计: ~5000 条
```

**数据示例**：

```json
// 正面示例：专业回答
{
  "messages": [
    {"role": "system", "content": "你是一个专业的医疗知识助手。你只提供医学科普信息，不提供具体诊疗建议。"},
    {"role": "user", "content": "我最近总是口渴，是糖尿病吗？"},
    {"role": "assistant", "content": "口渴是多种疾病的常见症状，不能仅凭口渴判断是否为糖尿病。糖尿病的典型症状包括"三多一少"：多饮、多食、多尿和体重减轻。\n\n建议您：\n1. 记录口渴频率和饮水量\n2. 观察是否有其他症状\n3. 到医院进行空腹血糖和HbA1c检测\n\n准确的诊断需要医生结合检验结果进行判断，建议您及时就诊。"}
  ]
}

// 拒绝示例：超出范围的问题
{
  "messages": [
    {"role": "system", "content": "你是一个专业的医疗知识助手..."},
    {"role": "user", "content": "我血糖7.5，应该吃什么药？"},
    {"role": "assistant", "content": "用药方案需要医生根据您的完整检查结果、既往病史、并发症情况等综合判断后才能制定。我无法提供具体用药建议。\n\n请您携带检查报告前往内分泌科或糖尿病专科门诊就诊，医生会为您制定个性化的治疗方案。"}
  ]
}
```

**训练配置**：

```python
lora_config = LoraConfig(
    r=16,
    lora_alpha=32,
    target_modules=["q_proj", "k_proj", "v_proj", "o_proj"],
    lora_dropout=0.05,
)

training_args = TrainingArguments(
    num_train_epochs=3,
    per_device_train_batch_size=2,
    gradient_accumulation_steps=8,  # 等效 bs=16
    learning_rate=2e-4,
    warmup_ratio=0.05,
    lr_scheduler_type="cosine",
)
```

**评估方式**：

1. 自建 Benchmark：200 条标准问题（50 条常见问题、50 条边界问题、50 条需拒绝的问题、50 条困难问题）
2. GPT-4o 自动打分（准确性、安全性、规范性）
3. 3 位医学背景同事人工评测

**实测效果**：拒绝率从 Base Model 的 ~20%（应拒绝却回答）降低到 ~3%；规范术语使用率从 ~60% 提升到 ~95%。

### 9.2 案例二：风格化写作助手

**需求背景**：某自媒体团队希望构建一个能模仿其品牌语气的写作助手：
- 风格：直接、有洞察、适度幽默、商业感强
- 输出：公众号标题、开头段落、金句提炼
- 数据：过去 3 年的 500 篇历史文章

**关键洞察**：风格微调的核心不是"让模型写更多文字"，而是"让模型内化创作者的思维模式"。

**数据构建策略**：

```
原始文章 → 拆解为：
  - 主题 → 标题（150 对）
  - 主题 + 角度 → 开头段落（200 对）
  - 段落 → 提炼金句（150 对）
  - 完整文章 → 仿写新文章（100 对，GPT-4 辅助生成）
```

**难点**：500 篇文章跨越 3 年，风格有演变。需要用最近 6 个月数据占更高权重（60%），早期数据占较低权重（20%）。

**推理 Prompt 设计**：

```
你是[品牌名]的内容创作助手，风格指南：
- 直接切入核心，不废话
- 用数据和案例支撑观点
- 结尾有行动指引
- 避免：鸡汤、过度谦虚、空话套话

任务：为主题"{topic}"写一个吸引人的公众号标题（5-15字）
```

**评估**：让团队成员在盲测中区分"AI写"和"人写"，区分正确率 < 60% 视为成功（随机猜测是 50%）。

## 十、总结：一份可执行的微调指南

### 10.1 微调决策 Checklist

在你开始任何微调工作之前，过一遍这份 Checklist：

**问题定义**
- [ ] 我能用一句话描述需要解决的核心问题吗？
- [ ] 这个问题是否真的无法通过优化 Prompt 解决？
- [ ] 这个问题是否真的无法通过 RAG 解决？
- [ ] 预期的效果提升是否值得投入？

**数据准备**
- [ ] 我有至少 500 条高质量训练数据吗？
- [ ] 数据格式与推理时 Prompt 格式一致吗？
- [ ] 数据覆盖了边界情况和负样本吗？
- [ ] 数据经过了去重和质量过滤吗？

**技术选型**
- [ ] 我的显存支持选定的微调方法吗？
- [ ] 我选择了合适的 Base Model 吗（许可证、能力、语言）？
- [ ] 我有评估集（与训练集不重叠）吗？

**工程准备**
- [ ] 我有完整的训练监控方案吗？
- [ ] 我有模型版本管理方案吗？
- [ ] 我知道如何回滚到上一个版本吗？

**部署准备**
- [ ] 我有清晰的成功指标（量化）吗？
- [ ] 我有线上 A/B 测试方案吗？
- [ ] 我知道什么情况下需要重新微调吗？

### 10.2 常见坑

**坑 1：数据格式训练时和推理时不一致**

这是最常见的 bug，没有之一。训练时的 chat template 与推理时不同，会导致模型完全不工作或效果极差。

```python
# 正确做法：用相同的 apply_chat_template
from transformers import AutoTokenizer

tokenizer = AutoTokenizer.from_pretrained(model_path)
# 训练和推理都用这个方法处理输入
formatted = tokenizer.apply_chat_template(messages, tokenize=False)
```

**坑 2：忘记设置 pad_token**

很多模型没有 pad_token，不设置会报错或影响 batch 训练：

```python
tokenizer.pad_token = tokenizer.eos_token
model.config.pad_token_id = tokenizer.eos_token_id
```

**坑 3：只在 loss 上判断训练好坏**

Train loss 下降不代表任务效果提升。必须同时监控 eval loss，以及定期做人工抽样检查。

**坑 4：忽略数据中的标签泄露**

如果 system prompt 里包含了答案相关信息，模型学到的是"找到并复制 system 里的答案"而不是真正理解任务。

**坑 5：在 instruction 部分计算 loss**

SFT 时应该只在 response 部分计算 loss，instruction 部分的 token 应该 mask 掉（label = -100）。很多实现忘记这一点。

```python
# 确认 SFTTrainer 的 dataset_text_field 或使用 DataCollatorForSeq2Seq
# 自动处理 instruction mask
```

**坑 6：过早停止训练**

Loss 曲线前期波动是正常的，不要因为短暂的 loss 上升就停止。等至少 1/3 的训练结束再判断趋势。

**坑 7：推理时没有设置合理的生成参数**

```python
# 推理时建议明确设置这些参数
outputs = model.generate(
    input_ids,
    max_new_tokens=512,
    temperature=0.7,
    top_p=0.9,
    repetition_penalty=1.1,  # 防止复读
    do_sample=True,
)
```

### 10.3 学习路径与工具推荐

**学习路径**（按优先级）：

```
1. 理解 Transformer 架构（Attention is All You Need）
2. 理解语言模型预训练（GPT/BERT 原理）
3. 跑通一次 LoRA 微调（用 LLaMA-Factory 或 Unsloth）
4. 深入理解 PEFT 原理（读 LoRA 原论文）
5. 学习数据工程（数据清洗、质量评估）
6. 学习对齐技术（DPO / RLHF）
7. 学习推理优化（vLLM、量化）
```

**工具推荐**：

| 类别 | 工具 | 推荐理由 |
|------|------|---------|
| 一键微调框架 | **LLaMA-Factory** | 支持最广泛的模型和方法，适合快速上手 |
| 高性能微调 | **Unsloth** | 比标准 LoRA 快 2-5 倍，适合时间敏感场景 |
| 数据管理 | **Argilla** | 开源数据标注和管理平台 |
| 训练监控 | **Weights & Biases** | 行业标准，可视化强大 |
| 推理服务 | **vLLM** | 高并发场景首选 |
| 模型评估 | **lm-evaluation-harness** | Benchmark 评估标准工具 |
| 量化工具 | **AutoAWQ / AutoGPTQ** | 模型量化的最佳实践 |
| 本地推理 | **Ollama** | 本地开发测试，极易上手 |

**关键论文清单**：

- LoRA：*LoRA: Low-Rank Adaptation of Large Language Models*（Hu et al., 2021）
- QLoRA：*QLoRA: Efficient Finetuning of Quantized LLMs*（Dettmers et al., 2023）
- InstructGPT：*Training language models to follow instructions*（Ouyang et al., 2022）
- DPO：*Direct Preference Optimization*（Rafailov et al., 2023）
- Scaling Laws：*Scaling Laws for Neural Language Models*（Kaplan et al., 2020）

## 写在最后

大模型微调是一个"看起来门槛低，做好门槛极高"的领域。跑通一次微调只需要几行代码，但真正调出一个生产可用的模型，需要对数据、算法、工程、评估都有深刻的理解。

这篇文章能帮你建立一个完整的认知地图，但每个章节都可以单独展开写一篇同等深度的文章。建议你选取最贴近自己业务的场景，动手跑起来——在实践中学到的，永远比读文章深刻。
