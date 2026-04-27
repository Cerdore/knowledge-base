# AI Infra 知识地图

> 从数据库内核转向 AI Infra（推理方向）

---

## 入门路径

[[00-Overview/AI-Infra-Landscape]] → [[01-Foundations/Transformer-Architecture]] → [[01-Foundations/KV-Cache]] → [[01-Foundations/HNSW-Implementation]]

## 三层优化体系

| 层级 | 目录 | 核心问题 |
|------|------|---------|
| 数据层 | [[02-Data-Level/Input-Compression]] | 如何减少推理输入/输出量 |
| 模型层 | [[03-Model-Level/efficient-arch/MoE]] | 如何设计/压缩模型结构 |
| 模型层 | [[03-Model-Level/embedding/Embedding-Models]] | 5 种 Embedding 模型对比分析 |
| 系统层 | [[04-System-Level/serving-system/KV-Cache-Management]] | 如何高效执行推理 |

## 核心技术链

```
FlashAttention → PagedAttention → KV Cache 管理 → PD 分离 → 分布式推理
```

[[04-System-Level/inference-engine/FlashAttention]]
→ [[04-System-Level/serving-system/KV-Cache-Management]]
→ [[04-System-Level/serving-system/PD-Separation]]
→ [[04-System-Level/serving-system/Distributed-Inference]]

## 框架实践

[[05-Frameworks/vLLM]] | [[05-Frameworks/SGLang]] | [[05-Frameworks/Mooncake]] | [[05-Frameworks/Semantic-Router-Architecture]]

## 论文清单

[[06-Papers/FlashAttention-ICML2022]] | [[06-Papers/vLLM-SOSP2023]]
[[06-Papers/DistServe-OSDI2024]] | [[06-Papers/Mooncake-FAST2025]]

## 我的优势路径（数据库 → AI Infra）

[[07-DB-to-AIInfra/Skills-Mapping]] → [[07-DB-to-AIInfra/Learning-Path]]

## 面试准备

[[08-Interview/Common-Questions]] | [[08-Interview/System-Design]]

---

*最后更新: 2026-04-25 | 参考: arXiv:2404.14294v3*
