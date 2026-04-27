# Semantic-Router 架构详解

## 项目定位

vLLM Semantic Router — 生产级 LLM 路由系统，负责将请求智能分发到最合适的模型后端。项目为 **Rust + Go + TypeScript** 混合架构。

## 顶层架构

| 层 | 语言 | 目录 | 职责 |
|---|---|---|---|
| 推理引擎 | Rust (Candle) | `candle-binding/` | 模型推理、分类、Embedding 生成 |
| FFI 桥接 | Go + CGo | `src/semantic-router/` | C-callable 函数暴露，路由逻辑，缓存 |
| 配置/控制面 | YAML + TS | `config/`, `dashboard/` | 路由规则定义，可视化 DSL |

## 数据流

```
请求 → Envoy/ExtProc → Go RouteServer
                            │
                            ├─ 信号评估 (Rust FFI)
                            │   ├─ 关键词匹配 (BM25/N-gram)
                            │   ├─ 领域分类 (mmBERT/BERT)
                            │   ├─ PII/越狱检测
                            │   └─ Embedding 相似度
                            │
                            ├─ 决策匹配 (YAML 规则)
                            │   └─ 优先级排序 + 布尔表达式
                            │
                            ├─ 模型选择 (算法层)
                            │   └─ static/confidence/remom/...
                            │
                            └─ 插件执行
                                ├─ Semantic Cache (HNSW 加速)
                                ├─ System Prompt
                                └─ Safety Guards
```

## 模块一：Rust 推理引擎

### 模型架构

- **CoreModel trait** — 含关联类型 `Config`, `Error`, `Output`，定义 `forward()`, `model_type()`
- **PathSpecialization** — 路径特化：`supports_parallel()`, `optimal_batch_size()`, `get_confidence_threshold()`
- **LongContextEmbeddingCapable** — embedding 特有能力：`extract_embeddings()`, `supports_matryoshka()`, `supports_instruction_aware()`
- **ModelFactory** — 运行时模型注册和加载

### 支持的 Embedding 模型

| 模型 | 维度 | 最大长度 | 池化 | Matryoshka |
|---|---|---|---|---|
| BERT/LoRA | 768 | 512 | Mean | No |
| Qwen3-Embedding-0.6B | 1024 | 32K | LastToken | Yes |
| Gemma-Embedding-300M | 768 | 2K | Mean | Yes |
| mmBERT-32K-2D | 768 | 32K | Mean | 2D |
| MultiModal-Embed-Small | 384 | 512 | Mean | MRL |

详见 [[Embedding-Models]]

### 智能模型选择

`select_embedding_model()` 根据序列长度和优先级自动选择：
- 短序列 (0-512): 质量优先 → Qwen3, 延迟优先 → Gemma, 默认 → Qwen3
- 中序列 (513-2048): 总是 GemmaEmbedding
- 长序列 (2049-32768): 总是 Qwen3

### Continuous Batching

- `ContinuousBatchScheduler` 动态合并并发请求
- 配置: `max_batch_size=32`, `max_wait_time_ms=5ms`
- 一次 forward pass 处理整个 batch

### 分类器

- `DualPathUnifiedClassifier` — 双路径（LoRA + Traditional）分类调度
- `calculate_path_score()` — 多因子路径评分（置信度 + 延迟 + 成功率）
- 支持的分类任务：intent, PII, jailbreak, fact-check, feedback, modality, complexity

### SIMD 加速

- AVX2 (8-wide) / AVX-512 (16-wide) 点积加速
- CPU 特性运行时检测 (`golang.org/x/sys/cpu`)
- Go 手写汇编 (`.s` 文件)

---

## 模块二：Go 缓存层

### CacheBackend 接口

```go
type CacheBackend interface {
    FindSimilar(model, query string) ([]byte, bool, error)
    FindSimilarWithThreshold(model, query string, threshold float32) ([]byte, bool, error)
    AddEntry(...) error
    Close() error
}
```

### 5 种后端

| 后端 | HNSW | 持久化 | 适用场景 |
|---|---|---|---|
| InMemoryCache | 可选 | 无 | 纯内存，低延迟 |
| HybridCache | 强制 | Milvus | 内存搜索 + 持久化存储 |
| MilvusCache | Milvus 内置 | Milvus | 大规模持久化 |
| RedisCache | Redis 内置 | Redis | Redis 用户 |
| ValkeyCache | Valkey 内置 | Valkey | Valkey 用户 |

### HNSW 向量索引

详见 [[HNSW-Implementation]]

核心特点：
- 距离计算：负点积（假设向量 L2 归一化）
- 层级分配：指数衰减随机，约 93.75% 节点在层 0
- 搜索：逐层 1-NN 贪心下降 + 层 0 完整 ef-NN 搜索
- 限制：不支持删除，过期触发全量重建

### 两套 HNSW 实现

1. 独立 `hnsw` 包 — 通用实现
2. 缓存嵌入式 HNSW — 生产路径实现

两套实现各有独立的 SIMD 汇编文件，代码不共享。

---

## 模块三：路由决策系统

### 信号层

| 信号 | 实现方法 | 用途 |
|---|---|---|
| Keywords | BM25 + N-gram + Fuzzy | 关键词匹配 |
| Embeddings | 候选示例 + 阈值 | 语义相似度 |
| Domains | mmBERT-32K 分类器 | 学术领域分类 |
| Fact check | 分类器 | 事实验证判定 |
| Reask detection | 分类器 | 重复问题检测 |
| PII | 混合模式 + 分类器 | PII 检测 |
| Jailbreak | DeBERTa v3 + Qwen3 Guard | 注入检测 |
| Complexity | 分类器 | 推理复杂度 |
| Modality | 分类器 | AR/DIFFUSION/BOTH |
| Language | 检测器 | zh/es 语言检测 |

### 决策层 — 路由算法

| 算法 | 描述 |
|---|---|
| `static` | 固定路由 |
| `confidence` | 置信度路由 (logprob/margin) |
| `ratings` | 多模型并发评分 |
| `remom` | 树状思维链 + 广度调度 |
| `router_dc` | Embedding 驱动判别 |
| `automix` | 自动混合选择 |
| `latency_aware` | 延迟感知选择 |

### 匹配流程

1. 预处理插件（越狱/PII 筛查）
2. 并行信号评估
3. 信号编译成布尔/浮点事实
4. 按优先级匹配路由规则（AND/OR 布尔表达式，first-match wins）
5. 选中路由的算法从 modelRefs 中选择模型
6. 后处理插件（semantic-cache → system_prompt → safety guards）
7. 转发到模型后端

---

## 模块四：向量存储层

VectorStoreBackend — OpenAI 兼容的 Vector Stores API：

| 后端 | HNSW | 持久化 |
|---|---|---|
| MemoryBackend | 无（暴力余弦相似度） | 无 |
| MilvusBackend | Milvus 内置 | Milvus |
| ValkeyBackend | Valkey 内置 | Valkey |
| LlamaStackBackend | 远程 | Llama Stack |

MemoryBackend 还实现了 `HybridSearcher`（BM25 + N-gram + 向量混合搜索）。

---

## 模块五：前端 DSL

TypeScript AST 定义 (`dashboard/frontend/src/types/dsl.ts`)：
- `ASTRouteDecl` — 路由声明
- `BoolExprNode` — 布尔表达式树
- `ASTAlgoSpec` — 算法配置
- `ASTPluginRef` — 插件引用

---

## 模块依赖关系

```
                    ┌──────────────┐
                    │  Envoy/ExtProc│
                    └──────┬───────┘
                           │
                    ┌──────▼───────┐
                    │  Go RouteServer│
                    └──────┬───────┘
                           │
        ┌──────────────────┼──────────────────┐
        │                  │                  │
   ┌────▼────┐      ┌──────▼──────┐    ┌─────▼─────┐
   │ 信号评估 │      │  决策匹配    │    │  插件执行  │
   │ (Rust FFI)│      │  (YAML规则) │    │           │
   └─────────┘      └─────────────┘    └─────┬─────┘
                                              │
                          ┌───────────────────┼──────────────┐
                          │                   │              │
                    ┌─────▼─────┐     ┌──────▼──────┐  ┌───▼────┐
                    │Semantic   │     │System Prompt│  │Safety  │
                    │Cache(HNSW)│     │             │  │Guards  │
                    └───────────┘     └─────────────┘  └────────┘
```

---

## 相关文档

- [[HNSW-Implementation]] — HNSW 算法深入分析
- [[Embedding-Models]] — 5 种 Embedding 模型对比
- [[KV-Cache]] — KV Cache 机制
- [[Transformer-Architecture]] — Transformer 基础
