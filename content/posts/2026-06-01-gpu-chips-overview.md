---
title: "GPU 芯片全景：国内外厂商对比、市场格局与异构部署实践"
date: "2026-06-01T10:00:00+08:00"
tags: ["ai"]
toc: true
---

随着大模型训练与推理需求的爆发，GPU 芯片从图形处理器演变为 AI 时代最核心的算力基础设施。本文系统梳理国内外主要 GPU 厂商的芯片系列、性能横向对比、市场格局，以及软件生态与异构混合部署的架构实践，供技术选型参考。

## 1. 为什么 GPU 在 AI 时代如此关键

### 1.1 从图形渲染到通用并行计算

GPU（Graphics Processing Unit）最初的设计目标是加速图形渲染，其核心特点是大量小型计算核心（相对 CPU 的少量大核）并行执行相同指令。这一特性与深度学习中大规模矩阵乘法的计算模式高度契合。

CPU 的设计哲学是降低**单任务延迟**：深流水线、乱序执行、大缓存，少量强力核心串行处理复杂逻辑。GPU 则以**吞吐量**为核心：数千个简单核心同时执行相同操作，用于处理矩阵乘、卷积等高度并行的数学运算。

2006 年 NVIDIA 发布 CUDA（Compute Unified Device Architecture），首次将 GPU 开放给通用计算场景，标志着 GPGPU（General-Purpose GPU）时代的开始。2012 年 AlexNet 在 ImageNet 竞赛中取得突破性成绩，其背后的关键之一就是 GPU 加速训练——GPU 正式进入 AI 主流视野。

### 1.2 大模型训练与推理的算力需求

大模型的算力需求可从以下几组数据量化：

- GPT-3（1750 亿参数）训练消耗约 3640 PFlop/s-day，相当于 1024 块 A100 运行约 34 天
- Llama 3 405B 的训练使用了超过 16000 块 H100
- 一次 GPT-4 的推理请求，需要多张 GPU 协同完成

训练与推理对 GPU 的需求有所不同：

| 维度 | 训练 | 推理 |
|------|------|------|
| 精度 | FP16/BF16/FP8 混合精度 | INT8 / FP8 / INT4 量化 |
| 显存需求 | 极高（模型 + 梯度 + 优化器状态） | 较低（仅模型权重 + KV Cache） |
| 批量大小 | 大 batch，高吞吐 | 小 batch，低延迟 |
| 通信需求 | 强依赖高速互联（All-Reduce） | 相对独立，单节点可行 |
| 核心指标 | TFLOPS（训练吞吐） | TTFT / TPS（首 Token 延迟 / 每秒 Token 数） |

这两类场景对 GPU 的侧重不同，也催生了"训练集群用高端 GPU，推理集群可用次一级或国产 GPU"的异构部署策略。

## 2. GPU 架构基础：读懂参数的底层逻辑

### 2.1 SIMT 架构与 Tensor Core

NVIDIA GPU 使用 **SIMT（Single Instruction, Multiple Threads）** 执行模型。线程以 32 个为一组形成 **Warp**，同一 Warp 内的线程在同一时钟周期执行相同指令。Warp 调度器负责在多个 Warp 之间切换，以掩盖内存访问的延迟。

```
SM (Streaming Multiprocessor)
 ├── Warp Scheduler × 4
 ├── CUDA Core（FP32）× 128
 ├── Tensor Core × 4（专用矩阵乘法单元）
 ├── L1 Cache / Shared Memory（可配置）
 └── LD/ST 单元、SFU
```

**Tensor Core** 是 Volta 架构（V100）以来 NVIDIA 在 AI 算力上的核心武器。一个 Tensor Core 在单个时钟周期内执行一次 4×4 矩阵乘加（MMA）运算，比普通 CUDA Core 的吞吐量高一个数量级。Hopper 架构（H100）进一步引入了 **Transformer Engine**，可在 FP8 和 FP16 之间动态切换精度，最大化矩阵乘的吞吐量。

### 2.2 核心指标解读：FLOPS、内存带宽、HBM 容量

评估 AI GPU 性能时，以下几个指标最为关键：

**算力（FLOPS）**
- FP32 / TF32 TFLOPS：训练精度，反映单精度浮点能力
- FP16 / BF16 TFLOPS：混合精度训练主力精度
- INT8 / FP8 TOPS：推理量化精度，数值通常是 FP16 的 2 倍（稀疏模式下可再翻倍）
- 注意区分是否开启**稀疏加速**（Sparsity），开启后标称性能翻倍，但需要权重满足 2:4 稀疏结构

**显存（HBM）**
- 容量（GB）：决定能加载多大的模型。70B 模型 BF16 精度需约 140 GB，FP8 需约 70 GB
- 带宽（TB/s）：影响矩阵乘法时的数据搬运速度，是 Memory-Bound 算子（如 Attention）的关键瓶颈

**互联带宽**
- GPU 间互联（NVLink / NVSwitch）：影响多卡训练时的 All-Reduce 通信效率
- 主机-GPU 带宽（PCIe 5.0 x16 = 64 GB/s）：相对于 NVLink 来说是明显瓶颈

**功耗（TDP）**
- H100 SXM 为 700 W，B200 达到 1000 W，数据中心供电密度是重要限制

### 2.3 互联技术：NVLink / PCIe / 光网络

```
节点内互联
  NVLink 4.0（H100）：单向 450 GB/s × 18 条 = 900 GB/s 双向
  NVSwitch（机柜内）：将多个 NVLink 域连成全互联拓扑

节点间互联
  InfiniBand NDR（400 Gb/s）：AI 训练集群标配，NVIDIA 收购 Mellanox 后垂直整合
  RoCE（RDMA over Converged Ethernet）：国内替代 IB 的主流方案，成本更低
  光网络（400G/800G）：超大规模集群骨干，华为 CloudFabric 等采用光电混合方案
```

节点内用 NVLink 实现 GPU 间全互联，节点间用 InfiniBand 或 RoCE 组成高带宽低延迟的 AI 网络，这是目前主流训练集群的标准架构。

## 3. 国际厂商 GPU 芯片系列

### 3.1 NVIDIA：Hopper / Blackwell 全系解析

NVIDIA 以绝对优势主导 AI 训练市场，其芯片产品线按架构迭代，近两代最为关键：

**Ampere 架构（2020）**

| 型号 | FP16 TFLOPS | 显存 | 带宽 | TDP |
|------|-------------|------|------|-----|
| A100 SXM 80G | 312 | 80GB HBM2e | 2.0 TB/s | 400 W |
| A100 PCIe 80G | 312 | 80GB HBM2e | 1.935 TB/s | 300 W |
| A10 | 125 | 24GB GDDR6 | 600 GB/s | 150 W |

A100 是上一代训练集群的主力，目前大量存量部署仍在运行。

**Hopper 架构（2022）**

| 型号 | BF16 TFLOPS | FP8 TFLOPS | 显存 | 带宽 | TDP |
|------|-------------|------------|------|------|-----|
| H100 SXM5 | 989 | 1979 | 80GB HBM3 | 3.35 TB/s | 700 W |
| H200 SXM5 | 989 | 1979 | 141GB HBM3e | 4.8 TB/s | 700 W |
| H100 PCIe | 756 | 1513 | 80GB HBM2e | 2.0 TB/s | 350 W |
| H20（中国版） | 296 | 592 | 96GB HBM3 | 4.0 TB/s | 350 W |

> H100 相对 A100 的 BF16 算力提升约 3×，引入 FP8 精度和 Transformer Engine 是最核心的突破。H200 与 H100 算力相同，但 HBM3e 将显存从 80GB 扩展到 141GB、带宽从 3.35 提升到 4.8 TB/s，专为大模型推理中 KV Cache 瓶颈而生。

**Blackwell 架构（2024）**

Blackwell 是 NVIDIA 目前最新一代架构，主要设计变化如下：

- **Dual-die 芯片设计**：B100/B200 由两个 GPC die 通过 NV-HBI（10 TB/s）连接，单卡逻辑上等效一块芯片，基于 TSMC 4NP 工艺，封装 208 亿晶体管
- **FP4 精度支持**：B200 FP4 推理算力达 9000 TFLOPS，相比 FP8 再翻倍
- **第五代 NVLink**：单 GPU 双向带宽提升至 1.8 TB/s

| 型号 | BF16 TFLOPS | FP8 TFLOPS | FP4 TFLOPS | 显存 | 带宽 | TDP |
|------|-------------|------------|------------|------|------|-----|
| B100 SXM | 1800 | 3500 | 7000 | 192GB HBM3e | 8.0 TB/s | 700 W |
| B200 SXM | 2250 | 4500 | 9000 | 192GB HBM3e | 8.0 TB/s | 1000 W |

**GB200 Grace Blackwell Superchip** 是单个封装模块：1 颗 Grace CPU + 2 颗 B200 GPU，通过 NVLink-C2C（900 GB/s）连接，形成统一内存空间。**GB200 NVL72** 机架由 36 个 Grace Blackwell Superchip 组成（36 颗 Grace CPU + 72 颗 B200 GPU），通过第五代 NVLink 构成统一互联域，总计 1.4 TB 显存，FP8 总算力超过 130 PFLOPS，液冷整机功率约 120 kW。

出口管制持续波动：2025 年美国对中国芯片出口政策经历多次反转——4 月扩大限制范围将 H20 纳入管控，7 月暂时恢复 H20 出口但附加 15% 附加税，8 月再度要求停产 H20；9 月中国网信办宣布禁止国内企业采购英伟达 AI 芯片；12 月特朗普政府宣布对通过安全审核的特定客户允许出口 H200 和 MI325X 等。当前中国市场出口管制政策仍处于动态调整中。

### 3.2 AMD：Instinct MI300 系列

AMD 以 MI300 系列作为 AI 加速器主力，与 NVIDIA 在 HPC 和推理市场正面竞争。

**MI300 架构亮点：芯片堆叠（Chiplet + HBM）**

MI300X 采用 XCD（Accelerator Complex Die）+ HBM3 3D 封装，将多个计算 Die 与内存 Die 直接叠放，实现超高带宽。MI350 系列升级至第四代 CDNA（CDNA 4）架构，基于 TSMC 3nm 制程，每 CU 的 16-bit 算力相比 CDNA 3 提升约 2 倍：

| 型号 | 架构 | BF16 TFLOPS | FP8 TFLOPS | 显存 | 带宽 | TDP |
|------|------|-------------|------------|------|------|-----|
| MI300X | CDNA 3 | 1307 | 2614 | 192GB HBM3 | 5.3 TB/s | 750 W |
| MI325X | CDNA 3 | 1307 | 2614 | 256GB HBM3e | 6.0 TB/s | 750 W |
| MI350X | CDNA 4 | ~2300 | ~4600 | 288GB HBM3e | 8.0 TB/s | 1000 W |
| MI355X | CDNA 4 | ~2300 | ~4600 | 288GB HBM3e | 8.0 TB/s | 1000 W |

MI355X 在 MI350X 基础上新增 MXFP4 和 MXFP6 原生数据类型支持，FP4 推理峰值超过 20 PFLOPS，定位超大规模推理场景。

MI300X 大显存优势使其成为推理 70B+ 参数模型的可行单卡方案（H100 仅 80GB 显存），Meta、微软 Azure 均有规模部署。

**ROCm 软件栈**是 AMD 的开源 CUDA 替代方案。PyTorch 对 ROCm 的支持在 2.0 后逐步完善，vLLM 也提供 ROCm 后端，但整体生态成熟度仍落后 CUDA（详见第 7 章）。

### 3.3 Intel：Gaudi 2/3 与 Xe 架构

Intel 在 AI 加速器市场经历过曲折：收购 Habana Labs 后推出 Gaudi 系列，定位中低端训练/推理市场，主打性价比。

| 型号 | BF16 TFLOPS | 显存 | 带宽 | TDP |
|------|-------------|------|------|-----|
| Gaudi 2 | 432 | 96GB HBM2e | 2.45 TB/s | 600 W |
| Gaudi 3 | 1835 | 128GB HBM2e | 3.7 TB/s | 900 W |

Gaudi 系列内置 24 个 100GbE RoCE 端口，支持节点间直连无需额外 InfiniBand 交换机，在中等规模训练集群（<256 卡）中有成本优势。

Intel 同时推进 **Xe GPU**（Ponte Vecchio / Rialto Bridge）用于 HPC，但在 AI 加速器赛道相对边缘化。2024 年 Intel 宣布裁减 Gaudi 相关业务规模，市场前景存疑。

### 3.4 Google TPU v4/v5：自研 AI 加速器

Google TPU（Tensor Processing Unit）是专为 TensorFlow/JAX 优化的 ASIC 加速器，不对外零售，仅通过 GCP 以云服务形式提供。

| 型号 | BF16 TFLOPS/芯片 | 显存 | 显存带宽 | 用途 |
|------|------------------|------|----------|------|
| TPU v4 | 275 | 32GB HBM | — | 训练 |
| TPU v5e | 197 | 16GB HBM | 819 GB/s | 高密度推理 |
| TPU v5p | 459 | 95GB HBM | 2765 GB/s | 大规模训练 |
| TPU v6e（Trillium） | ~918 | 32GB HBM | 1600 GB/s | 训练 + 推理，已 GA |
| TPU v7（Ironwood） | ~4614 | 192GB HBM | 7200 GB/s | 推理为主，2025 年 4 月发布 |

Trillium（TPU v6e）于 2024 年底正式 GA，单芯片算力约为 v5e 的 4.7 倍，能效提升 67%，单 Pod 最大 256 芯片，可组成 91 EFLOPS 的训练集群。

**TPU v7 Ironwood** 于 2025 年 4 月 Google Cloud Next 发布，是首款明确以大规模推理为主要目标的 TPU 代际，单芯片达到约 4614 TFLOPS BF16，192GB HBM，7.2 TB/s 带宽，9216 芯片集群可组成超大推理池。Google 内部 Gemini 系列大模型的训练和推理均在 TPU Pod 上完成。

对于非 Google 生态用户，TPU 的主要使用门槛在于 JAX 编程模型和 XLA 编译器与 PyTorch/CUDA 生态的差异，算子行为和性能调优经验不可直接迁移。

## 4. 国产 GPU 芯片系列

在出口管制与国产替代大背景下，中国涌现出一批 AI 芯片创业公司和国企研究所。以下按当前综合实力排序介绍。

### 4.1 华为昇腾（Ascend 910B / 910C）

昇腾系列是目前国产 GPU 中生态最完整、算力最强、市场份额最大的产品线，由华为海思设计。

**芯片规格（厂商公布 / 业界估测数据）**

| 型号 | FP16 TFLOPS | 显存 | 带宽 | TDP | 制程 | 备注 |
|------|-------------|------|------|-----|------|------|
| Ascend 910B | 256 | 64GB HBM2 | 900 GB/s | 400 W | 7nm（SMIC） | 存量主力 |
| Ascend 910B3 | ~320 | 64GB HBM2 | 1.2 TB/s | 450 W | 7nm+ | 910B 改版 |
| Ascend 910C | ~800 | 128GB HBM3 | 2.4 TB/s | ~600 W | 6nm | Dual-die，2025 年大规模量产 |

910C 采用双 Die 封装（将两颗 910B Die 集成，类似 B200 设计），显存从 64GB 扩展至 128GB，业界估测算力约为 H100 的 80%。2025 年一季度进入大规模量产阶段，全年出货量估计在 70~80 万颗。受制于国内代工厂工艺成熟度，实际效能与台积电同代制程仍存在差距。

**软件栈：CANN + MindSpore**

华为围绕昇腾构建了完整软件栈：
- **CANN**（Compute Architecture for Neural Networks）：算子库 + 编译器，类比 CUDA
- **MindSpore**：华为自研深度学习框架，原生支持昇腾
- **PyTorch_npu**：PyTorch 的昇腾适配插件，允许现有 PyTorch 代码通过修改设备字符串（`cuda` → `npu`）运行在昇腾上

华为昇腾在国内云（华为云 ModelArts）、国企训练集群中有大量部署，与 NVIDIA 相比最大差距仍在软件生态和社区活跃度。

### 4.2 寒武纪（MLU 系列）

寒武纪是国内较早上市的 AI 芯片公司（2020 年科创板），产品线覆盖云端训练/推理：

| 型号 | FP16 TFLOPS | INT8 TOPS | 显存 | 带宽 | TDP |
|------|-------------|-----------|------|------|-----|
| MLU370-X4 | 24（FP32）/ 256 | 256 | 24GB LPDDR5 | 614 GB/s | 150 W |
| MLU370-X8 | 24（FP32）/ 256 | 256 | 48GB LPDDR5 | 1.2 TB/s | 300 W |
| MLU590-H8 | ~300 | ~600 | 64GB HBM2 | 2.0 TB/s | 550 W |

MLU370 采用 7nm Chiplet 工艺，首次引入 LPDDR5 内存（带宽是前代 GDDR6 的 3 倍），MLU-Link 提供每卡 200 GB/s 的跨芯片直连带宽，定位中等规模推理。MLU590 是面向训练市场的旗舰卡，2024 年量产，FP16 算力约 300 TFLOPS。

软件栈为 **Cambricon Neuware SDK**，提供 BANG-C 编程语言（类 CUDA C）、CNToolkit，以及 PyTorch/TensorFlow 适配层。

### 4.3 沐曦（MetaX）

沐曦是成立于 2020 年的 GPU 初创公司，创始团队来自 AMD，主打 GPGPU 路线（与国内多数 ASIC 路线不同），兼容 CUDA 生态是其核心卖点。

| 型号 | FP16/BF16 TFLOPS | INT8 TOPS | 显存 | 带宽 | 形态 |
|------|------------------|-----------|------|------|------|
| C500 | 280 | 560 | 64GB HBM2e | 1.8 TB/s | PCIe 板卡 |
| C550 | ~280 | ~560 | 64GB HBM2e | 约 1.8 TB/s | OAM 模组（支持 OAM 1.5/2.0）|

C500 TF32 算力 140 TFLOPS，FP16/BF16 280 TFLOPS，64GB HBM2e，带宽 1.8 TB/s，搭载自研 MetaXLink 高速互联，支持单机 8 卡全互联，互联带宽达 896 GB/s。C550 为 OAM 模组形态，规格与 C500 接近但面向服务器主板直插场景。

**MXMACA 编程框架**在 API 层面与 HIP 高度对齐，通过转译层可运行大部分 CUDA 代码，相比昇腾等纯 ASIC 路线，软件迁移摩擦更低。

### 4.4 摩尔线程（Moore Threads）

摩尔线程由前 NVIDIA 高管张建中创立（2020 年），主打图形 + AI 双用途 GPGPU，采用台积电制程：

| 型号 | FP16/BF16 TFLOPS | INT8 TOPS | 显存 | 带宽 | TDP | 备注 |
|------|------------------|-----------|------|------|-----|------|
| MTT S80 | ~12（FP32）| — | 16GB GDDR6 | 512 GB/s | 80 W | 消费级，2022 年 |
| MTT S4000 | 100 | 200 | 48GB GDDR6 | 768 GB/s | 450 W | 第三代 MUSA 架构，128 Tensor Cores |

MTT S4000 是摩尔线程目前主力 AI 推理卡，48GB GDDR6（16 Gbps），支持 MTLink 1.0 多卡互联。2025 年 12 月摩尔线程发布全新 **Huagang（华港）架构**，下一代芯片 "华山" 定位对标 B200，显存带宽和算力密度据称接近 Blackwell 水平，具体规格待官方发布。

**MUSA（Moore Threads Unified System Architecture）** 框架通过 `mt-musacc` 编译器将 CUDA 代码转译，API 层面与 CUDA 兼容，是国内兼容路线的代表产品之一。

### 4.5 燧原科技（Enflame）

燧原科技由腾讯投资，基于 GCU（General Compute Unit）架构，面向云端 AI 训练/推理：

| 型号 | FP16 TFLOPS | 显存 | 带宽 | TDP |
|------|-------------|------|------|-----|
| T20 | 128 | 32GB HBM2 | 1.0 TB/s | 250 W |
| T21 | 256 | 32GB HBM2e | 1.2 TB/s | 300 W |
| CloudBlazer X2 | ~512 | 64GB HBM2e | 2.0 TB/s | 450 W |

软件栈为 **TopsRider**，提供 PyTorch/TensorFlow 适配。腾讯云内部有部分工作负载使用燧原卡。

### 4.6 壁仞科技（Biren Technology）

壁仞科技融资超过 47 亿人民币，BR100 是其首款高端通用 GPU，2022 年发布时标称性能超越 A100：

| 型号 | FP16 TFLOPS | 显存 | 带宽 | TDP | 形态 |
|------|-------------|------|------|-----|------|
| BR100（OAM）| 1000（标称）| 64GB HBM2e | 2.0 TB/s | 400 W | OAM |
| BR100（PCIe）| 1000（标称）| 64GB HBM2e | 2.0 TB/s | 300 W | PCIe |

BR100 于 2024 年中期进入量产，已在中国电信等运营商的千卡集群中部署，中国电信单集群持续训练运行超 30 天未中断。2025 年壁仞科技通过港交所上市聆讯，拟于 2026 年 1 月赴港 IPO，融资规模超 6 亿美元，下一代芯片 BR200 系列正在研发中。

### 4.7 天数智芯（Iluvatar CoreX）

天数智芯专注于 GPGPU，采用 CUDA 兼容路线，产品主要面向推理场景：

| 型号 | FP16 TFLOPS | 显存 | 带宽 | TDP |
|------|-------------|------|------|-----|
| BI-V100 | 128 | 32GB GDDR6 | 614 GB/s | 250 W |
| BI-V150 | 256 | 64GB HBM2e | 1.6 TB/s | 350 W |

软件栈为 **IXUCA SDK**，支持 CUDA 转译。主要客户在金融、互联网推理场景。

## 5. 横向性能对比

### 5.1 FP16/BF16 训练算力对比

| 芯片 | BF16 TFLOPS（Dense）| 备注 |
|------|---------------------|------|
| AMD MI355X | ~2300 | CDNA 4，3nm，MXFP4 20+ PFLOPS |
| NVIDIA B200 | 2250 | Blackwell，稀疏 4500，FP4 9000 |
| NVIDIA B100 | 1800 | Blackwell，稀疏 3500 |
| AMD MI350X | ~2300 | CDNA 4，3nm，1000W TBP |
| Intel Gaudi 3 | 1835 | 含稀疏，FP8 1.8 PFLOPS |
| AMD MI325X | 1307 | CDNA 3，256GB HBM3e |
| AMD MI300X | 1307 | CDNA 3，192GB HBM3，稀疏 2614 |
| NVIDIA H100 SXM | 989 | Hopper，稀疏 1979 |
| NVIDIA H200 SXM | 989 | H100 算力相同，141GB HBM3e |
| 华为 Ascend 910C | ~800 | 业界估测，Dual-die |
| Intel Gaudi 2 | 432 | — |
| 寒武纪 MLU590 | ~300 | 厂商公布 |
| 华为 Ascend 910B | 256 | 7nm（SMIC）|
| 沐曦 C500 | 280 | 官方规格，HBM2e |
| 天数 BI-V150 | 256 | HBM2e，PCIe 4.0 |
| 摩尔线程 S4000 | 100 | GDDR6，FP16/BF16 |
| NVIDIA A100 | 312（FP16） | 上一代参考基准 |

### 5.2 显存容量 & 内存带宽对比

显存容量决定能装入多大模型，带宽决定访存密集型算子（Attention、LayerNorm）的速度上限：

| 芯片 | 显存 | 带宽 | HBM 代 |
|------|------|------|--------|
| AMD MI355X / MI350X | 288 GB | 8.0 TB/s | HBM3e |
| NVIDIA B200 | 192 GB | 8.0 TB/s | HBM3e |
| AMD MI325X | 256 GB | 6.0 TB/s | HBM3e |
| AMD MI300X | 192 GB | 5.3 TB/s | HBM3 |
| NVIDIA H200 | 141 GB | 4.8 TB/s | HBM3e |
| Intel Gaudi 3 | 128 GB | 3.7 TB/s | HBM2e |
| NVIDIA H100 | 80 GB | 3.35 TB/s | HBM3 |
| 华为 Ascend 910C | 128 GB | 2.4 TB/s | HBM3 |
| 沐曦 C500 | 64 GB | 1.8 TB/s | HBM2e |
| 寒武纪 MLU590 | 64 GB | 2.0 TB/s | HBM2 |
| 华为 Ascend 910B | 64 GB | 900 GB/s | HBM2 |

> **关键结论**：AMD MI300X/MI325X 以 192/256 GB 显存在超大模型推理场景极具竞争力。国产卡中 Ascend 910C 的显存和带宽提升明显，但与 H100 的带宽差距仍约 30%。

### 5.3 功耗与能效比

| 芯片 | TDP | BF16 TFLOPS/W |
|------|-----|---------------|
| NVIDIA B100 | 700 W | 2.57 |
| NVIDIA H100 SXM | 700 W | 1.41 |
| AMD MI300X | 750 W | 1.74 |
| Intel Gaudi 3 | 900 W | 2.04 |
| 华为 Ascend 910C | ~600 W | ~1.30 |
| NVIDIA A100 | 400 W | 0.78 |

Blackwell 在能效上有显著提升（相同 TDP 算力接近翻倍），这对数据中心 PUE 成本有重要意义。

### 5.4 互联带宽对比

节点内 GPU 间互联是大规模训练的关键瓶颈：

| 方案 | 带宽 | 方案 |
|------|------|------|
| NVLink 4.0（H100）| 900 GB/s 双向 | 8 卡全互联（NVSwitch） |
| NVLink 5.0（B200）| 1.8 TB/s 双向 | 8 卡全互联 |
| AMD Infinity Fabric | 896 GB/s 双向 | MI300X 内部 |
| 昇腾 HCCS | ~400 GB/s | 8 卡互联 |
| PCIe 5.0 x16 | 128 GB/s 双向 | 无高速互联时的降级方案 |

## 6. 市场格局与客户生态

### 6.1 全球 GPU 服务器市场份额

NVIDIA 在 AI 训练加速器市场的份额长期维持在 **80% 以上**（部分机构估计 2023-2024 年超过 90%），硬件与 CUDA 软件生态相互强化，构成其主要竞争壁垒。

```
2024 年 AI 加速器市场（数据中心 GPU/加速器）
  NVIDIA     ~88%
  AMD        ~5-7%
  Intel      ~2-3%
  其他        <2%（含 Google TPU 自用部分）
```

NVIDIA 在 H100 供货紧张时期（2023 年）的单卡现货价格曾炒至 5 万美元以上，供不应求的局面持续至 2024 年底。

### 6.2 中国市场：国产替代的进展

美国对华 AI 芯片出口管制自 2022 年以来持续收紧，并在 2025 年进入高度波动期：

- **2022 年 10 月**：A100/H100 禁售，降级版 A800/H800 短暂可售
- **2023 年 10 月**：A800/H800 纳入管控，H20/L20/L40S 成为合规版本
- **2025 年 4 月**：Trump 政府将 H20 纳入管控，全面暂停对华出口
- **2025 年 7 月**：暂时恢复 H20 出口，但对每张卡征收 15% 附加税
- **2025 年 8 月**：NVIDIA 再度被要求停产 H20
- **2025 年 9 月**：中国网信办宣布禁止国内企业采购英伟达 AI 芯片，要求转向国产
- **2025 年 12 月**：美国宣布对通过安全审查的特定客户允许出口 H200 和 AMD MI325X

政策的持续波动使得企业难以依赖单一供应链，这也是推动异构多供应商部署架构的重要现实原因。

**国内市场现状（2025 年底）**：
- 华为昇腾：出货量最大，910C 全年出货估计达 70~80 万颗，华为云、运营商算力中心大规模部署
- 壁仞科技：BR100 已在运营商千卡集群落地，2026 年赴港 IPO
- 摩尔线程：2025 年末发布 Huagang 新架构，正冲击 AI 推理市场
- 寒武纪：上市公司，持续亏损，主要客户在政府和央企算力中心
- 沐曦 / 天数智芯：主要面向中小规模推理场景
- 整体来看，国产 GPU 在**算力密度、显存带宽、软件生态和系统稳定性**上与 NVIDIA H100 仍有 1-2 代差距，与 B200/Rubin 差距更大

**政策推动**：
- "东数西算"工程、算力补贴、政府采购倾斜国产
- 国有云厂商（电信云、联通云、移动云）被要求提高国产算力比例

### 6.3 主要云厂商采购策略

| 云厂商 | 主力 GPU | 说明 |
|--------|----------|------|
| AWS | NVIDIA H100/H200 | UltraCluster，也有 Trainium（自研）|
| Azure | NVIDIA H100/H200 + AMD MI300X | 与 OpenAI 深度绑定，大量 H100 |
| GCP | NVIDIA H100 + TPU v5 | 自研 TPU 与 NVIDIA 双轨 |
| 阿里云 | NVIDIA H20/A800 + 自研含光 | 自研推理芯片 + 合规 NVIDIA |
| 腾讯云 | NVIDIA H20/A800 + 燧原 | 腾讯战略投资燧原 |
| 华为云 | 昇腾 910B/910C | 以昇腾为主，自成体系 |
| 百度云 | NVIDIA H20 + 昆仑芯（自研） | 昆仑芯侧重搜索推理 |

### 6.4 典型行业客户

**大模型公司**
- OpenAI：微软 Azure H100 集群（据报超 10 万卡）
- Anthropic：GCP + AWS H100/H200
- DeepSeek：据报使用大量 H800（出口管制版 H100）

**国内大模型**
- 智谱 AI、月之暗面、百川智能：主要使用 NVIDIA H20 + 部分昇腾
- 华为内部大盘古模型：全昇腾集群

**自动驾驶**
- Tesla：自研 Dojo 超算 + A100 训练集群
- 国内厂商（地平线、黑芝麻等）：主要使用 NVIDIA，同时推自研推理芯片

**HPC / 科研**
- 超算中心：GPU 集群与 CPU 集群混合，部分采购国产加速器（昇腾、寒武纪）

## 7. 软件生态：GPU 的护城河

### 7.1 CUDA 生态：为何护城河如此之深

CUDA（Compute Unified Device Architecture）自 2006 年发布至今已 18 年，积累了：

- **数十万个 CUDA 优化算子库**：cuBLAS、cuDNN、NCCL、cuSPARSE……
- **主流深度学习框架**的原生后端：PyTorch 的 ATen 张量库与 CUDA 深度耦合
- **高性能推理库**：TensorRT、FlashAttention-CUDA、vLLM 的 PagedAttention
- **数十年的编译器优化积累**：nvcc、CUDA Graphs、PTX 中间表示

对开发者而言，切换到非 CUDA 平台意味着：
1. 所有 CUDA 算子需要重写或转译
2. FlashAttention 等关键优化需要移植验证
3. 调试工具（Nsight、nvprof）不可用
4. 部分闭源库（TensorRT）无法使用

这解释了为什么 AMD MI300X 在显存容量上超越 H100，企业采购时仍大量选择 NVIDIA：现有 CUDA 代码库的迁移摩擦远高于硬件差价。

### 7.2 AMD ROCm：差距几何？

ROCm（Radeon Open Compute）是 AMD 的开源 CUDA 替代方案，近年来有实质性进展：

```
ROCm 近期里程碑：
  PyTorch 2.0+  官方支持 ROCm 后端
  vLLM 0.4+     支持 ROCm，MI300X 推理性能接近 H100
  Flash Attention  ROCm 版本已可用（由社区维护）
  TGI (HuggingFace)  支持 ROCm
```

**仍有差距的领域**：
- NCCL 等同完整度（rccl 有功能差异）
- 算子覆盖率约为 CUDA 的 85-90%
- 调优工具链成熟度
- 社区问题解答和文档密度

**实际评估建议**：对于已经跑通的 PyTorch 推理工作负载，ROCm + MI300X 是可行的 NVIDIA 替代方案，尤其在显存需求大（>80GB）的场景下成本优势明显。

### 7.3 华为 CANN & MindSpore

CANN 是昇腾的核心计算框架，分层架构如下：

```
应用层
  ├── MindSpore / PyTorch_npu / TensorFlow_npu
  └── ModelArts 平台服务

算子层
  ├── AscendCL（底层运行时 API，类比 CUDA Driver API）
  ├── ACLNN（神经网络算子库，类比 cuDNN）
  └── HCCL（通信库，类比 NCCL）

硬件层
  └── Ascend 910 系列芯片
```

PyTorch_npu 插件通过 `torch.device("npu")` 接入，对现有 PyTorch 代码侵入性较小，但遇到未适配的自定义算子时仍需手动实现 Ascend C 版本。

MindSpore 原生支持昇腾且性能更优，但其编程范式（函数式变换、静态图优先）与 PyTorch 差异显著，迁移成本高。

### 7.4 国产 GPU SDK 对比

| 厂商 | 编程接口 | CUDA 兼容层 | PyTorch 适配 | 成熟度 |
|------|----------|-------------|--------------|--------|
| 华为昇腾 | CANN / Ascend C | 无（异构 ASIC） | PyTorch_npu | ★★★★☆ |
| 寒武纪 | BANG-C / CNToolkit | 无 | torch_mlu | ★★★☆☆ |
| 沐曦 | MXMACA（类 HIP） | 转译层 | torch_musa | ★★★☆☆ |
| 摩尔线程 | MUSA | 转译层 | torch_musa | ★★★☆☆ |
| 燧原 | TopsRider | 部分兼容 | tops-torch | ★★☆☆☆ |
| 天数智芯 | IXUCA SDK | 转译层 | torch_dipu | ★★☆☆☆ |

### 7.5 框架兼容性矩阵

| 框架/推理引擎 | NVIDIA CUDA | AMD ROCm | 昇腾 CANN | 其他国产 |
|--------------|-------------|----------|-----------|----------|
| PyTorch 2.x | 原生 | 官方 | 插件 | 各自插件 |
| TensorFlow 2.x | 支持 | 部分 | 插件 | 部分 |
| vLLM | 支持 | ROCm 后端 | 社区分支 | 部分 |
| LMDeploy | 支持 | 不支持 | 华为维护 | 不支持 |
| TensorRT-LLM | 支持 | 不支持 | 不支持 | 不支持 |
| MindIE | 不支持 | 不支持 | 华为专属 | 不支持 |
| Ollama | 支持 | 支持 | 不支持 | 不支持 |
| FlashAttention | 完整 | ROCm 版 | 部分 | 不支持 |

## 8. 异构 GPU 混合部署：架构分析与技术选型

### 8.1 为什么需要异构混合部署

在以下场景中，企业往往被迫或主动选择多厂商 GPU 混合部署：

1. **供货约束**：H100 等高端卡受出口管制或交期限制，紧急算力需求需用国产卡填充
2. **成本分层**：训练任务对性能敏感，用 NVIDIA 旗舰；推理任务可用更便宜的替代卡
3. **合规要求**：政府/央企项目要求一定比例使用国产算力
4. **存量资产**：已有 A100 集群不退役，新增 H100 节点，混合运营降低迁移成本

### 8.2 架构层次：从驱动到调度的全栈视图

异构集群的技术栈自底向上分为五层：

| 层次 | 主要组件 | 说明 |
|------|----------|------|
| 应用层 | PyTorch、vLLM、LMDeploy、MindSpore 等 | 训练框架与推理引擎，通过各厂商 Python 插件或后端接入底层算力 |
| 编排层 | Kubernetes + Volcano / Koordinator / HAMi | 负责工作负载调度、资源配额、Gang Scheduling 与 GPU 虚拟化 |
| 资源层 | nvidia-device-plugin、ascend-device-plugin、cambricon-device-plugin、HAMi device-plugin | 将物理 GPU 暴露为 K8s Extended Resource，支持虚拟化切分 |
| 运行时 | containerd + NVIDIA Container Toolkit / cri-hook | 容器启动时注入 GPU 设备挂载，按 runtimeClass 区分不同厂商 |
| 驱动层 | CUDA Driver、Ascend Driver、寒武纪驱动等 | 各厂商驱动独立安装，通常可在同一宿主机上共存 |

**多厂商共存的核心挑战**：
- 驱动层：同一节点上不同厂商驱动互不影响（通常可共存）
- 设备识别：各厂商有独立的 Device Plugin，标签和 Resource Name 不同
- 调度层：需要统一调度异构设备，避免碎片化
- 通信层：跨厂商 GPU 之间无法直接用 NVLink/HCCL，必须走 CPU 中转或 RoCE

### 8.3 设备接入层：K8s Device Plugin 机制与各厂商适配

K8s Device Plugin 框架（`k8s.io/device-plugin`）是 GPU 等扩展资源接入 Kubernetes 的标准机制：

```
kubelet
  └── 监听 /var/lib/kubelet/device-plugins/*.sock
        ├── nvidia.com/gpu  → nvidia-device-plugin
        ├── huawei.com/Ascend910  → ascend-device-plugin
        ├── cambricon.com/mlu  → cambricon-device-plugin
        └── iluvatar.ai/gpu  → iluvatar-device-plugin
```

各厂商 Device Plugin 实现 `ListAndWatch`、`Allocate` 两个核心 gRPC 接口，将 GPU 设备暴露为 K8s Extended Resource，Pod 通过 `resources.limits` 申请：

```yaml
resources:
  limits:
    nvidia.com/gpu: "1"      # NVIDIA GPU
    # 或
    huawei.com/Ascend910: "1"  # 昇腾 NPU
```

**异构集群中的标签策略**：通过 Node Label 区分硬件类型，配合 `nodeSelector` 或 `nodeAffinity` 将 Pod 调度到正确类型节点：

```yaml
nodeSelector:
  accelerator: nvidia-h100  # 或 huawei-ascend910c
```

### 8.4 调度层技术选型

原生 K8s 调度器对 AI 工作负载有天然局限（不支持 Gang Scheduling、不感知拓扑）。以下是三个主流调度方案的对比：

#### Volcano

[Volcano](https://volcano.sh) 是 CNCF 孵化项目（华为捐献），专为 AI/HPC 批调度设计：

```
核心能力：
  Gang Scheduling  ── 全量 Pod 同时就绪才开始运行（训练必需）
  Queue 机制       ── 多租户公平调度，支持优先级抢占
  Job Plugin       ── PyTorchJob / TFJob / MPIJob 原生集成
  拓扑感知         ── NUMA / GPU 拓扑亲和性
```

适用场景：**分布式训练**（All-Reduce 类任务对 Gang Scheduling 依赖强），多租户算力平台。

#### Koordinator

[Koordinator](https://koordinator.sh) 由阿里云开源，侧重在线离线混部场景的资源超卖与 QoS 保障：

```
核心能力：
  ColocationProfile  ── 在线推理 + 离线训练混部
  QoS 分级           ── LSE/LSR/LS/BE 四级，CPU/Memory 动态超卖
  GPU Share          ── 细粒度 GPU 共享（显存 + 算力维度）
  Descheduler        ── 运行时重调度，优化碎片
```

适用场景：**推理与训练混部**，提升 GPU 利用率；对训练任务 SLA 要求不极致的场景。

#### HAMi（Heterogeneous AI computing virtualization Middleware）

[HAMi](https://github.com/Project-HAMi/HAMi)（原名 k8s-vgpu-scheduler，腾讯开源后更名）是目前最成熟的 **异构 GPU 虚拟化统一管理** 方案：

```
核心能力：
  多厂商统一管理  ── 支持 NVIDIA / 昇腾 / 寒武纪 / 沐曦 等
  显存切割        ── 将一块物理 GPU 切分为多个虚拟 GPU（显存隔离）
  算力限制        ── 对虚拟 GPU 的 SM 使用率进行限额
  拓扑感知调度    ── 优先分配同一 NVLink 域内的 GPU
```

HAMi 的架构分为三部分：

```
HAMi 架构
  ├── scheduler-extender  ── 扩展原生调度器，注入异构感知逻辑
  ├── device-plugin       ── 替换/包装厂商 device-plugin，上报虚拟设备
  └── container hook      ── 运行时拦截，注入显存/算力限制 so 库
```

Pod 申请方式：

```yaml
resources:
  limits:
    nvidia.com/gpu: "1"
    nvidia.com/gpumem: "8192"   # 申请 8GB 显存
    nvidia.com/gpucores: "50"   # 申请 50% SM 算力
```

**三种方案对比**

| 维度 | Volcano | Koordinator | HAMi |
|------|---------|-------------|------|
| 核心定位 | 批调度 / Gang | 混部 / 超卖 | GPU 虚拟化 / 异构统一 |
| Gang Scheduling | 原生支持 | 支持 | 依赖外部调度器 |
| GPU 共享切分 | 不支持 | 粗粒度 | 显存+算力级 |
| 异构 GPU 统一管理 | 需自定义 | 需自定义 | 多厂商原生支持 |
| 在线/离线混部 | 有限 | 核心特性 | 不支持 |
| 社区活跃度 | ★★★★★ | ★★★★☆ | ★★★★☆ |
| 生产案例 | 华为云、腾讯云 | 阿里云 | 腾讯云、多家互联网 |

**推荐选型原则**：
- 大规模分布式训练平台 → **Volcano** 为主，补充 HAMi 做 GPU 共享
- 推理+训练混部、提升 GPU 利用率 → **Koordinator + HAMi**
- 国产 GPU + NVIDIA 异构混合、显存级共享 → **HAMi** 是目前最成熟选择

### 8.5 通信层：NCCL vs 国产通信库

分布式训练的核心通信原语是 **All-Reduce**（梯度聚合）和 **All-to-All**（MoE 专家并行）。

```
NVIDIA NCCL（NVIDIA Collective Communications Library）
  ├── 支持 NVLink + InfiniBand + RoCE 多种传输后端
  ├── Ring-AllReduce / Tree-AllReduce 算法
  ├── 与 PyTorch distributed 深度集成（默认 backend="nccl"）
  └── 高度优化，接近硬件带宽上限

华为 HCCL（Huawei Collective Communication Library）
  ├── 昇腾专属，支持 HCCS（片间互联）+ RoCE
  ├── PyTorch_npu 通过 backend="hccl" 接入
  └── 接口与 NCCL 基本对齐，但跨厂商混用不支持

其他国产通信库
  ├── CNCL（Cambricon Communication Library，寒武纪）
  ├── 摩尔线程 MUCL
  └── 各自仅支持本厂 GPU，无统一抽象
```

**异构集群通信的核心问题**：NCCL 不认识昇腾 NPU，HCCL 不认识 NVIDIA GPU，两者无法直接参与同一个通信组。

**主流解法**：

方案一：**拓扑切分，避免跨厂商通信**
将训练任务按模型并行拆分，使每个 All-Reduce 通信域内部只有同厂商 GPU，跨域通信通过 CPU 桥接（代价是带宽下降）：

```
节点 A（NVIDIA 8× H20）── NCCL All-Reduce ──┐
                                              ├── CPU 汇聚 → 参数服务器模式
节点 B（昇腾 8× 910C）── HCCL All-Reduce ──┘
```

方案二：**统一通信抽象层（实验性）**
- OpenMPI + UCX：通过 UCX 传输层屏蔽底层 RDMA 差异，但性能损失较大
- 学术界有研究将 NCCL 和 HCCL 统一到 [UCC](https://github.com/openucx/ucc)（Unified Collective Communication），工程化程度有限

**实践建议**：当前阶段，异构训练最可行方案是**数据并行分域**（同一 All-Reduce 域内保持单一厂商），而非真正的跨厂商张量并行。

### 8.6 推理框架多后端支持

推理场景的异构部署比训练容易，因为推理任务通常单卡或少卡独立，无需跨卡通信：

**vLLM**
- 架构：基于 PagedAttention 的高效 KV Cache 管理
- 支持后端：CUDA（最完整）、ROCm（AMD，官方支持）、昇腾（社区分支 `vllm-ascend`）
- 多后端通过 `Platform` 抽象层实现，`--device cuda/rocm/npu` 切换

```python
# NVIDIA
vllm serve meta-llama/Llama-3-8B --device cuda

# AMD ROCm
vllm serve meta-llama/Llama-3-8B --device rocm

# 昇腾（需安装 vllm-ascend 插件）
vllm serve meta-llama/Llama-3-8B --device npu
```

**LMDeploy**
- 由上海人工智能实验室开源，同时支持 NVIDIA（TurboMind 引擎）和昇腾（PyTorch 后端）
- 对国产 GPU 的支持在国内开源推理框架中最为成熟

**MindIE**
- 华为自研推理引擎，专为昇腾优化，集成在 华为云 ModelArts 中
- 性能优于通用框架在昇腾上的表现，但仅支持昇腾硬件

**Triton Inference Server（NVIDIA）**
- 支持 CUDA / TensorRT，不支持非 NVIDIA 后端

| 推理框架 | NVIDIA | AMD ROCm | 昇腾 | 其他国产 | 开源 |
|----------|--------|----------|------|----------|------|
| vLLM | 支持 | 支持 | 社区支持 | 部分 | 开源 |
| LMDeploy | 支持 | 不支持 | 支持 | 不支持 | 开源 |
| MindIE | 不支持 | 不支持 | 支持 | 不支持 | 部分开源 |
| TGI | 支持 | 支持 | 不支持 | 不支持 | 开源 |
| TensorRT-LLM | 支持 | 不支持 | 不支持 | 不支持 | 开源 |

### 8.7 典型落地模式：训练与推理的异构分工

当前最实际可行的异构混合部署架构如下：

```
┌────────────────────────────────────────────────────────┐
│                  统一 Kubernetes 集群                   │
│                                                        │
│  ┌─────────────────────┐   ┌────────────────────────┐  │
│  │   训练节点池          │   │   推理节点池            │  │
│  │  (NVIDIA H20/A800)   │   │  (昇腾 910C / 国产卡)  │  │
│  │                      │   │                      │  │
│  │  PyTorch + NCCL      │   │  vLLM/LMDeploy       │  │
│  │  Volcano GangJob     │   │  + Koordinator 混部   │  │
│  │  8× GPU / 节点       │   │  HAMi GPU 切分共享     │  │
│  └──────────┬───────────┘   └───────────┬──────────┘  │
│             │                           │             │
│  ┌──────────▼───────────────────────────▼──────────┐  │
│  │           统一模型仓库 / 推理服务网关               │  │
│  │        （训练产出模型 → 部署到推理节点）             │  │
│  └─────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────┘
```

**设计要点**：
- 训练与推理节点通过 Node Label 隔离（`role: training / inference`）
- 训练任务提交到 Volcano Queue，推理服务通过 Deployment + HPA 弹性伸缩
- 模型格式在训练完成后统一转换（如 `.safetensors`），推理引擎各自加载

### 8.8 混合部署的已知坑与最佳实践

**坑 1：驱动版本冲突**
多厂商驱动共存时，容器运行时 hook（`nvidia-container-toolkit`、`cri-hook`）可能冲突，导致 Pod 启动失败。
→ **解法**：各厂商驱动安装在独立命名空间，containerd config 中为不同节点配置不同 runtimeClass。

**坑 2：算子缺失导致推理精度异常**
国产 GPU 的 PyTorch 适配层对部分 CUDA 算子回退到 CPU 执行，默默影响精度或性能，日志中没有明显报错。
→ **解法**：启用 `torch_npu` 的算子覆盖日志，对比 NVIDIA 基准输出；关键模型在上线前做 FP16 vs 量化精度对比测试。

**坑 3：NCCL 版本与框架版本的绑定**
PyTorch 内置 NCCL，集群节点上手动安装的 NCCL 版本与之不匹配时，多机训练出现 `Timeout` 或 `Unhandled system error`。
→ **解法**：统一使用 NCCL 内置版本（`NCCL_DEBUG=INFO` 验证版本一致性），或使用容器化固定依赖。

**坑 4：HAMi 显存切分与驱动版本强绑定**
HAMi 的算力限制基于 `libvgpu.so` 注入，与 CUDA Driver 版本强绑定，升级驱动后可能失效。
→ **解法**：锁定节点 CUDA Driver 版本，升级前在测试节点验证 HAMi 功能正常。

**坑 5：异构集群的监控盲区**
Prometheus + DCGM Exporter 只覆盖 NVIDIA GPU，昇腾有独立的 `mindx-exporter`，指标名称和语义不一致。
→ **解法**：为不同厂商 GPU 部署各自的 exporter，在 Grafana 中按 `gpu_vendor` label 聚合展示。

**最佳实践清单**：
- [ ] 统一 Node Label 规范（`gpu.vendor`, `gpu.model`, `gpu.memory`）
- [ ] 使用 runtimeClass 区分不同 GPU 运行时
- [ ] HAMi + Volcano 组合处理训练场景的 Gang Scheduling + GPU 虚拟化
- [ ] 推理节点启用 HAMi 显存切分，提升 GPU 利用率（目标 > 70%）
- [ ] 建立跨厂商的统一监控大盘（DCGM + mindx-exporter + 自定义 exporter）
- [ ] 关键模型上线前在目标 GPU 上完成精度基准测试

## 9. 展望：下一代 GPU 与竞争格局演变

### 9.1 NVIDIA Rubin 路线图

**Rubin 架构**已于 2026 年 1 月 CES 正式发布并进入量产，是 NVIDIA 当前最新一代架构：

| 规格项 | Rubin GPU |
|--------|-----------|
| 制程 | TSMC 3nm（N3/N3P） |
| 晶体管数 | 336 亿 |
| FP4 算力 / 芯片 | 50 PFLOPS（推理）/ 35 PFLOPS（训练）|
| BF16 算力 / 芯片 | ~12,500 TFLOPS（估算）|
| 显存 | 288GB HBM4 |
| 显存带宽 | 22 TB/s（约为 B200 的 2.75×）|
| NVLink 6 带宽 | 3.6 TB/s 双向 / GPU |

**Vera Rubin NVL72** 机架：36 颗 Vera CPU + 72 颗 Rubin GPU，FP4 推理总算力 3.6 EFLOPS，训练总算力 2.5 EFLOPS，相比 GB200 NVL72 分别提升约 5× 和 3.5×。

NVIDIA 路线图：Blackwell（2024）→ Rubin（2026）→ Rubin Ultra（2027）→ Feynman（2028+），保持年度迭代节奏。

### 9.2 国产芯片的追赶节奏

国产芯片与 NVIDIA 的差距可以从几个维度量化：

| 维度 | 当前差距 | 追赶难度 |
|------|----------|----------|
| 单卡峰值算力 | 1-2 代（1-3 年） | 中（制程受限） |
| 显存带宽 | 1-2 代 | 高（HBM 供应链） |
| 软件生态 | 5-10 年 | 极高（网络效应） |
| 互联带宽（NVLink 等价） | 3-5 倍差距 | 高 |
| 系统可靠性 | 显著差距 | 中高 |

最难逾越的障碍不是单卡峰值算力，而是软件生态的累积效应：FlashAttention、PagedAttention、Triton kernel 等关键优化均以 CUDA 为首要平台实现，其他平台的移植在时间和完整性上持续落后。

短期内（3-5 年），国产 GPU 最可能突破的路径是：
1. **推理场景特化**：不追求通用性，对特定模型架构深度优化
2. **CUDA 兼容转译**：降低迁移成本，直接复用 CUDA 生态
3. **算力堆叠弥补单卡差距**：用 4 卡 910C 替代 1 卡 H100 的集群级算力

### 9.3 算力即基础设施的长期趋势

从需求侧看，几个趋势值得关注：

- 国内政策层面，算力国产化已纳入长期规划，国有云和央企算力中心的采购结构将持续向国产倾斜
- Scaling Law 尚未失效，预训练仍是大模型能力的主要来源，对 H100/H200 级别算力的需求短期内不会消退
- 推理侧需求增速快于训练：每次 API 调用都消耗 GPU，大规模商业化部署的推理卡需求量正在超过训练卡
- 数据中心电力成本已成为头部 AI 公司的主要运营成本项，能效（TFLOPS/W）将越来越影响硬件选型

对于技术团队，在出口管制和供货不确定性持续的背景下，构建与具体 GPU 厂商解耦的推理部署架构（vLLM 多后端 + HAMi 统一调度）是降低供应链风险的可行路径，而非可选优化。
