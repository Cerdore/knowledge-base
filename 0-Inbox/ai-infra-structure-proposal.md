# AI Infra 知识库结构优化方案

> 基于 LLM 推理综述 (arXiv:2404.14294v3) 的三层优化体系重构
>
> 作者背景：数据库内核 → AI Infra（推理方向）转型

---

## 一、现状问题分析

### 当前结构

```
2-Areas/AI-Infra/
└── MOC.md   ← 唯一文件，所有内容堆在此处
```

### 主要缺陷

| 问题 | 表现 |
|------|------|
| **空壳结构** | MOC 中 `[[wiki-links]]` 指向的文件均不存在 |
| **职责混淆** | 学习路线、知识分类、论文列表、面试题全部混在一个文件里 |
| **缺少层次** | 没有反映推理优化的三层体系（数据层/模型层/系统层） |
| **覆盖不全** | 缺少模型压缩（量化/蒸馏/剪枝）、替代架构（Mamba/SSM）等重要模块 |
| **资料散乱** | 框架、论文、概念、系统架构没有清晰分域 |

---

## 二、参考依据

基于 **《A Survey on Efficient Inference for Large Language Models》**（Zixuan Zhou et al., arXiv:2404.14294v3, 2024年7月）提出的核心分类体系：

```
高效 LLM 推理
├── 数据层优化（Data-Level）
│   ├── 输入压缩（Prompt Pruning / Summary / RAG）
│   └── 输出组织（SoT / Parallel Decoding）
├── 模型层优化（Model-Level）
│   ├── 高效结构设计（MoE / GQA / SSM 替代架构）
│   └── 模型压缩（量化 / 剪枝 / 蒸馏 / 动态推理）
└── 系统层优化（System-Level）
    ├── 推理引擎（FlashAttention / 投机解码 / Offloading）
    └── 服务系统（KV Cache 管理 / 连续批处理 / 调度 / PD分离）
```

---

## 三、建议目录结构

```
2-Areas/AI-Infra/
│
├── MOC.md                              # 入口导航（精简为纯链接地图）
│
├── 00-Overview/                        # 总览层：建立宏观认知
│   ├── AI-Infra-Landscape.md           # AI Infra 全景图（训练/推理/数据/MLOps）
│   └── LLM-Inference-Survey.md         # 综述精读笔记（arXiv:2404.14294）
│
├── 01-Foundations/                     # 基础层：必备前置知识
│   ├── Transformer-Architecture.md     # Transformer 结构与自注意力
│   ├── KV-Cache.md                     # KV Cache 原理、格式、增长规律
│   ├── GPU-Memory-Hierarchy.md         # HBM/SRAM/内存墙问题
│   └── CUDA-Basics.md                  # CUDA 编程模型、warp、shared memory
│
├── 02-Data-Level/                      # 数据层优化
│   ├── Input-Compression.md            # Prompt Pruning / Summary / Soft Prompt / RAG
│   └── Output-Organization.md          # SoT / APAR / DAG 并行解码
│
├── 03-Model-Level/                     # 模型层优化
│   ├── efficient-arch/
│   │   ├── MoE.md                      # 混合专家模型（Mixtral, Switch Transformer）
│   │   ├── GQA-MQA.md                  # 分组注意力、KV Cache 内存压缩
│   │   └── Alternative-Arch.md         # Mamba(SSM) / RWKV / Hyena
│   └── model-compression/
│       ├── Quantization.md             # PTQ/QAT, GPTQ, AWQ, SmoothQuant
│       ├── Pruning.md                  # SparseGPT, Wanda, Sparse Attention
│       ├── Knowledge-Distillation.md   # MiniLLM, Black-box KD
│       └── Dynamic-Inference.md        # Early Exit (CALM, SkipDecode)
│
├── 04-System-Level/                    # 系统层优化（核心重点）
│   ├── inference-engine/
│   │   ├── FlashAttention.md           # IO-aware Attention, v1/v2/v3
│   │   ├── Speculative-Decoding.md     # 草稿模型, Medusa, Eagle
│   │   ├── Kernel-Optimization.md      # Kernel Fusion, MegaBlocks
│   │   └── Offloading.md               # FlexGen, Powerinfer, llama.cpp
│   └── serving-system/
│       ├── KV-Cache-Management.md      # PagedAttention, LightLLM, FlashInfer
│       ├── Continuous-Batching.md      # ORCA, Sarathi, split-and-fuse
│       ├── Scheduling.md               # FastServe, VTC, FCFS
│       ├── PD-Separation.md            # Prefill/Decode解耦（DistServe, Splitwise）
│       └── Distributed-Inference.md    # 张量并行, 流水线并行, Infinite-LLM
│
├── 05-Frameworks/                      # 主流推理框架
│   ├── vLLM.md                         # PagedAttention, OpenAI 兼容接口
│   ├── SGLang.md                       # 结构化生成, RadixAttention
│   ├── TensorRT-LLM.md                 # NVIDIA 生态, 量化+融合
│   ├── DeepSpeed.md                    # ZeRO, split-and-fuse
│   └── Mooncake.md                     # KV Cache 池化 + PD 分离（FAST 2025）
│
├── 06-Papers/                          # 论文精读笔记
│   ├── FlashAttention-ICML2022.md      # IO感知注意力
│   ├── vLLM-SOSP2023.md                # PagedAttention
│   ├── DistServe-OSDI2024.md           # PD分离架构
│   └── Mooncake-FAST2025.md            # KV Cache池化+PD分离
│
├── 07-DB-to-AIInfra/                   # 数据库背景迁移路径（你的差异化优势）
│   ├── Skills-Mapping.md               # 技能对应表（存储引擎↔KV Cache等）
│   └── Learning-Path.md                # 推荐学习顺序（6周计划）
│
└── 08-Interview/                       # 面试准备
    ├── Common-Questions.md             # 高频八股（KV Cache/PD分离/PagedAttention等）
    └── System-Design.md                # 系统设计题（设计一个推理服务）
```

---

## 四、MOC 重构建议

现有 MOC.md 过于臃肿，建议重构为**纯导航地图**，内容迁移到对应子目录：

```markdown
# AI Infra 知识地图

## 入门路径
[[00-Overview/AI-Infra-Landscape]] → [[01-Foundations/Transformer-Architecture]] → [[01-Foundations/KV-Cache]]

## 核心技术链
[[04-System-Level/serving-system/KV-Cache-Management]] → [[04-System-Level/serving-system/PD-Separation]]
[[04-System-Level/inference-engine/FlashAttention]] → [[04-System-Level/inference-engine/Speculative-Decoding]]

## 框架实践
[[05-Frameworks/vLLM]] | [[05-Frameworks/SGLang]] | [[05-Frameworks/Mooncake]]

## 论文清单
[[06-Papers/vLLM-SOSP2023]] | [[06-Papers/Mooncake-FAST2025]]

## 我的优势路径
[[07-DB-to-AIInfra/Skills-Mapping]] → [[07-DB-to-AIInfra/Learning-Path]]
```

---

## 五、优先填充顺序

基于面试准备目标，建议按以下顺序优先创建实质内容：

### 第一优先级（面试高频，两周内完成）
1. `01-Foundations/KV-Cache.md` — 几乎所有问题的基础
2. `04-System-Level/serving-system/KV-Cache-Management.md` — PagedAttention
3. `04-System-Level/serving-system/PD-Separation.md` — Mooncake 核心思想
4. `04-System-Level/inference-engine/FlashAttention.md` — 系统层必问
5. `04-System-Level/inference-engine/Speculative-Decoding.md` — 无损加速方向
6. `08-Interview/Common-Questions.md` — 面试直接使用

### 第二优先级（技术深度，三到四周）
7. `03-Model-Level/model-compression/Quantization.md` — 最主流压缩手段
8. `04-System-Level/serving-system/Continuous-Batching.md` — 吞吐优化基础
9. `05-Frameworks/vLLM.md` — 行业标准推理框架
10. `06-Papers/Mooncake-FAST2025.md` — 最新核心论文精读

### 第三优先级（拓展广度）
11. 模型层优化（MoE, GQA 等）
12. 数据层优化（RAG 等）
13. 分布式推理
14. 替代架构（Mamba/SSM）

---

## 六、与综述的对应关系

| 综述章节 | 本知识库位置 |
|----------|-------------|
| Section 4：数据层优化 | `02-Data-Level/` |
| Section 5.1：高效架构 | `03-Model-Level/efficient-arch/` |
| Section 5.2：模型压缩 | `03-Model-Level/model-compression/` |
| Section 6.1：推理引擎 | `04-System-Level/inference-engine/` |
| Section 6.2：服务系统 | `04-System-Level/serving-system/` |
| Section 7：应用场景 | 分散在各子目录（长上下文/边缘部署等） |

---

## 七、Key Insights（来自综述）

做优先级决策时的参考依据：

- **量化** 是当前最主流的压缩手段（W4A16 在 Decoding 阶段加速最显著）
- **FlashAttention + vLLM** 是业界部署的标准配置（综述 Table 6）
- **投机解码（Eagle）** 是无损加速的最佳方向（端到端 2.77~3.74x）
- **PD 分离** 是大规模推理服务的核心架构演进方向
- **KV Cache 管理** 是推理系统的核心瓶颈，也是面试高频考点

---

*生成日期: 2026-04-23 | 参考: arXiv:2404.14294v3*
