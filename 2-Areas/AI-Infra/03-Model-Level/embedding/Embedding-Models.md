# Embedding 模型对比

## 概述

semantic-router 的 Rust 推理引擎（基于 Candle）支持 5 种 Embedding 模型，通过 `ModelFactory` 注册和 `select_embedding_model()` 智能选择。

## 模型对比表

| 模型 | 参数量 | 维度 | 最大长度 | 池化 | Matryoshka | Instruction-Aware |
|---|---|---|---|---|---|---|
| BERT/LoRA | varies | 768 | 512 | Mean | No | No |
| Qwen3-Embedding-0.6B | 0.6B | 1024 | 32,768 | LastToken | Yes (128/256/512/768) | Yes |
| Gemma-Embedding-300M | 300M | 768 | 2,048 | Mean | Yes (768/512/256/128) | No |
| mmBERT-32K-2D | ~120M | 768 | 32,768 | Mean | 2D (层+维度) | No |
| MultiModal-Embed-Small | ~120M | 384 | 512 | Mean (per-modality) | MRL (384/256/128/64/32) | No |

## 各模型详解

### BERT/LoRA

- **文件**: `traditional/bert.rs`, `lora/bert_lora.rs`
- **定位**: 传统分类 + LoRA 微调适配
- **池化**: Mean pooling — `sum(hidden * mask) / sum(mask)`
- **用途**: 分类任务（intent, PII, jailbreak, fact-check, feedback）的 embedding 特征提取

### Qwen3-Embedding-0.6B

- **文件**: `embedding/qwen3_embedding.rs`
- **定位**: 高质量长上下文 embedding
- **池化**: Last-token pooling — 取序列最后有效 token 的 hidden state
- **Matryoshka**: 支持 `[128, 256, 512, 768]` 维度截断（注意：hidden_size 实际为 1024，Matryoshka 维度列表来自 0.6B 配置）
- **Instruction-Aware**: 支持指令前缀 (`supports_instruction_aware() = true`)
- **Continuous Batching**: 通过 `ContinuousBatchScheduler` 支持动态批次合并

### Gemma-Embedding-300M

- **文件**: `embedding/gemma_embedding.rs`
- **定位**: 中序列最佳性价比（2K 上下文）
- **池化**: Mean pooling
- **Matryoshka**: 支持 `[768, 512, 256, 128]` 维度截断
- **优势**: 参数量小 (300M)，推理速度快

### mmBERT-32K-2D

- **文件**: `embedding/mmbert_embedding.rs`
- **定位**: 超长上下文 + 二维 Matryoshka
- **池化**: Mean pooling
- **2D Matryoshka**: 同时支持**层提前退出**（选择第 N 层 hidden state）+ **维度截断**（截断前 K 维）
- **32K 上下文**: 适合超长文档 embedding

### MultiModal-Embed-Small

- **文件**: `embedding/multimodal_embedding.rs`
- **定位**: 多模态（文本 + 图像 + 音频）
- **池化**: 每种模态使用固定池化方法（文本 Mean，图像 global-average）
- **MRL**: Matryoshka Representation Learning, 支持 `[384, 256, 128, 64, 32]`
- **维度**: 384（最小的 embedding 维度）

## Embedding 流水线

```
文本输入
  │
  ├─ 1. Tokenize → (input_ids, attention_mask)
  │
  ├─ 2. Model Forward → hidden states
  │
  ├─ 3. Pooling (pooling.rs)
  │     ├─ Mean:     sum(hidden * mask) / sum(mask)  [BERT, Gemma, mmBERT]
  │     ├─ LastToken: hidden[last_valid_position]      [Qwen3]
  │     └─ CLS:      hidden[:, 0, :]                  [Original BERT]
  │
  ├─ 4. 可选 Matryoshka 降维（截断前 N 维）
  │
  └─ 5. 可选 2D Matryoshka（mmBERT: 提前退出层 + 维度截断）
```

## 智能模型选择

`select_embedding_model()` (`classifiers/unified.rs:946`) 根据序列长度和优先级自动选择：

| 序列长度 | 条件 | 选择 |
|---|---|---|
| 0-512 | quality_priority > 0.7 | Qwen3 |
| 0-512 | latency_priority > 0.7 | Gemma |
| 0-512 | 默认 | Qwen3 |
| 513-2048 | 总是 | GemmaEmbedding |
| 2049-32768 | 总是 | Qwen3（唯一支持 32K 的模型） |

额外的强制条件：
- Matryoshka target_dim < 768 且 latency_priority > 0.5 → 强制 Gemma
- 选中模型不可用 → 回退到下一个可用模型

## Continuous Batching

文件: `embedding/continuous_batch_scheduler.rs`

- `ContinuousBatchScheduler` 动态合并并发请求
- 配置: `max_batch_size=32`, `max_wait_time_ms=5ms`
- 一次 forward pass 处理整个 batch，显著提升吞吐（声称 2-5x）
- 通过 `InitEmbeddingModelsBatched()` (Qwen3) 启动

## Core Traits

```rust
// 所有模型的基 trait
trait CoreModel: Send + Sync + Debug {
    type Config: Clone + Send + Sync + Debug;
    type Error: Error + Send + Sync + 'static;
    type Output: Send + Sync + Debug;

    fn model_type(&self) -> ModelType;
    fn forward(&self, input_ids: &Tensor, attention_mask: &Tensor) -> Result<Self::Output, Self::Error>;
    fn get_config(&self) -> &Self::Config;
}

// 长上下文 embedding 特有能力
trait LongContextEmbeddingCapable: CoreModel {
    fn get_max_sequence_length(&self) -> usize;      // 2048 ~ 32768
    fn get_embedding_dimension(&self) -> usize;       // 384 ~ 1024
    fn get_pooling_method(&self) -> PoolingMethod;    // Mean / LastToken / CLS
    fn supports_matryoshka(&self) -> bool;
    fn get_matryoshka_dimensions(&self) -> Vec<usize>;
    fn supports_instruction_aware(&self) -> bool;
    fn extract_embeddings(&self, hidden_states, attention_mask, target_dim) -> Result<Tensor>;
    fn optimal_embedding_batch_size(&self) -> usize;
    fn supports_parallel_batching(&self) -> bool;
}
```

## Go 侧 API

| 函数 | 用途 |
|---|---|
| `InitEmbeddingModels()` | 加载 Qwen3 + Gemma + mmBERT |
| `InitEmbeddingModelsBatched()` | Qwen3 + Continuous Batching |
| `InitMmBertEmbeddingModel()` | 仅加载 mmBERT |
| `InitMultiModalEmbeddingModel()` | 仅加载多模态模型 |
| `GetEmbeddingSmart()` | 自动模型路由 |
| `GetEmbeddingWithModelType()` | 手动指定模型类型 |
| `GetEmbedding2DMatryoshka()` | mmBERT 2D Matryoshka |
