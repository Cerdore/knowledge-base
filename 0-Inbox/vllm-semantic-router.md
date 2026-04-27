# vLLM Semantic Router 项目介绍

## 一、项目是什么？

vLLM Semantic Router 是一个**智能路由器**，核心作用是：根据用户输入的内容，自动决定把请求转发给哪个 AI 模型。

举个例子：你有一个便宜的模型（如 Gemma）和一个贵的模型（如 GPT-5）。当用户问"1+1=?" 时，没必要用贵的模型，路由器会自动把这类简单问题发给便宜模型；当用户问复杂的数学证明时，路由器会自动转发给强大的模型。这就是 **Mixture-of-Models（模型混合）** 的核心思想。

---

## 二、项目整体结构

这是一个多语言单体仓库（Monorepo），包含以下主要子系统：

```
semantic-router/
├── src/
│   ├── semantic-router/    ← 核心路由引擎（Go 语言）
│   ├── vllm-sr/            ← CLI 工具 & 编排层（Python）
│   ├── fleet-sim/          ← 集群模拟器（Python）
│   └── training/           ← 模型训练（Python）
├── candle-binding/         ← ML 推理绑定（Rust）
├── deploy/                 ← K8s/Helm/OpenShift 部署配置
├── dashboard/              ← Web 管理面板（Go + React）
├── e2e/                    ← 端到端测试（Go + Python）
├── config/                 ← 规范配置文件
├── bench/                  ← 性能基准测试
├── docs/                   ← 开发者文档（ADR、计划等）
├── paper/                  ← ICLR 2026 学术论文（LaTeX）
└── website/                ← 用户文档网站（Docusaurus）
```

---

## 三、semantic-router 和 vllm-sr 的关系

这是初学者最容易混淆的地方：

```
┌─────────────────────────────────────────────┐
│                  用户                        │
│              $ vllm-sr serve                │
└─────────────────┬───────────────────────────┘
                  │
     ┌────────────▼────────────┐
     │    vllm-sr (Python)     │  ← CLI 工具层
     │  • 命令行入口            │    "启动/配置/管理"
     │  • 配置生成/校验          │
     │  • Docker 编排           │
     │  • 部署管理              │
     └────────────┬────────────┘
                  │ 启动并管理
     ┌────────────▼────────────┐
     │ semantic-router (Go)    │  ← 核心引擎
     │  • 请求拦截 & 分类        │    "真正干活的部分"
     │  • 信号评估 & 决策        │
     │  • 模型选择              │
     │  • 缓存 / 记忆 / RAG     │
     └────────────┬────────────┘
                  │ CGo 调用
     ┌────────────▼────────────┐
     │ candle-binding (Rust)   │  ← ML 推理
     │  • 文本 Embedding        │    "向量化 & 分类"
     │  • 模型推理              │
     └─────────────────────────┘
```

简单理解：

- **semantic-router（Go）** = 引擎本体，是真正运行时处理每个请求的组件
- **vllm-sr（Python）** = 遥控器/工具箱，用来配置、启动、管理这个引擎
- **candle-binding（Rust）** = 引擎里的"大脑"，负责机器学习推理

---

## 四、核心路由引擎 (semantic-router) 模块详解

引擎作为 Envoy gRPC 外部处理器运行，架构是一条处理流水线：

```
请求进入 → 信号分类 → 决策评估 → 模型选择 → 请求修改 → 转发到目标模型
```

### 关键模块

| 模块 | 路径 | 功能 |
|---|---|---|
| entry point | `cmd/` | 程序入口，启动 gRPC 服务器、API 服务器、K8s 控制器 |
| extproc | `pkg/extproc/` | 核心处理流水线——处理 Envoy 传来的每个请求/响应 |
| classification | `pkg/classification/` | 信号分类器——判断请求的"特征" |
| config | `pkg/config/` | 配置定义——所有信号规则、决策规则的数据结构 |
| decision | `pkg/decision/` | 决策引擎——用布尔规则树评估信号匹配结果 |
| selection | `pkg/selection/` | 模型选择算法——多个候选模型时选最优的（Elo、RouterDC、AutoMix 等 11 种算法） |
| cache | `pkg/cache/` | 语义缓存——相似问题直接返回缓存结果，不调用模型 |
| memory | `pkg/memory/` | 对话记忆——跨会话记住用户上下文 |
| hnsw | `pkg/hnsw/` | 向量索引——快速近似最近邻搜索 |
| vectorstore | `pkg/vectorstore/` | 向量存储——文档摄入、分块、嵌入、相似搜索 |
| tools | `pkg/tools/` | 工具选择——根据请求匹配相关的 function calling 工具 |
| observability | `pkg/observability/` | 可观测性——Prometheus 指标、OpenTelemetry 追踪、日志 |

### 信号分类器支持的信号类型（共 18 种）

| 信号 | 说明 |
|---|---|
| keyword | 关键词匹配（子串、BM25、n-gram、模糊匹配） |
| embedding | 语义相似度（向量余弦相似度） |
| category/domain | 领域/意图分类（数学、编程、写作等） |
| jailbreak | 越狱检测 |
| pii | 敏感信息检测 |
| language | 语言检测 |
| context | 上下文感知 |
| structure | 请求结构分析 |
| complexity | 复杂度评分 |
| reask | 重复问题检测 |
| preference | 偏好分类 |
| fact_check | 事实核查信号 |
| modality | 模态检测（文本 vs 图片生成） |
| authz | 授权级别 |
| kb | 知识库信号 |
| conversation | 对话流信号 |
| projection | 路由投影 |

---

## 五、执行流程（一次请求的完整路径）

**1. 用户发送请求**

```
POST /v1/chat/completions
{"messages": [{"role": "user", "content": "证明费马大定理"}]}
```

**2. Envoy 代理拦截请求**，通过 gRPC 发送给 Semantic Router（ExtProc）

**3. Router 提取用户问题** → `"证明费马大定理"`

**4. 信号分类**（并行评估所有信号）：

```
├── 关键词匹配: "费马" → math 相关
├── Embedding 相似度: 与"数学证明"原型相似度 0.92
├── 复杂度评分: 高复杂度
└── 领域分类: mathematics
```

**5. 决策引擎评估**：

```
Decision: "complex-math"
Rule: (category == math) AND (complexity > 0.8)
结果: ✅ 匹配!
```

**6. 模型选择**：

```
候选模型: [GPT-5, Claude Opus, DeepSeek-R1]
算法: Elo 评分 + LatencyAware
选中: Claude Opus (评分最高且延迟可接受)
```

**7. 请求修改**：

- 将请求 body 中的 `model` 字段改为目标模型名
- 可选：注入 system prompt、应用 RAG、添加工具等

**8. Envoy 将修改后的请求转发到 Claude Opus 的 API 端点**

**9. 响应返回**（可选：写入缓存、提取记忆、幻觉检测等）

---

## 六、输入 & 输出

**输入：**

- **配置文件** (`config.yaml`)：定义路由规则、模型端点、插件等
- **API 请求**：OpenAI 兼容格式的 `/v1/chat/completions` 请求
- **模型文件**：本地 Embedding/分类模型（通过 HuggingFace 下载）

**输出：**

- **路由决策**：修改后的请求（`model` 字段被替换为目标模型名）
- **可选修改**：注入 system prompt、RAG 上下文、工具定义等
- **遥测数据**：Prometheus 指标、OpenTelemetry 追踪、路由重放日志

---

## 七、vLLM CLI 工具 (vllm-sr) 的主要命令

```
vllm-sr serve       # 一键启动：路由器 + Envoy + 监控面板
vllm-sr validate    # 校验配置文件
vllm-sr eval        # 不部署，直接评估路由决策
vllm-sr chat        # 通过路由器发送一次对话请求
vllm-sr status      # 查看运行状态
vllm-sr config       # 配置生成/导入/迁移
```

---

## 八、一句话总结

> **vllm-sr** 是方向盘（你操作的 CLI 工具），**semantic-router** 是发动机（Go 写的核心路由引擎），**candle-binding** 是火花塞（Rust 写的 ML 推理）。三者配合工作，实现了一个根据请求内容智能选择合适的 AI 模型的系统。
