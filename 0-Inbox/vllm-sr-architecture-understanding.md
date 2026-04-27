# vLLM Semantic Router 架构理解

**目的：** 帮助开发者快速理解 vLLM Semantic Router 的架构设计、核心管道和数据流，为阅读源码和贡献代码提供主干上下文。

---

## 1. 定位：SR 在 vLLM 生态中的位置

```
User/Client
  │
  ▼
┌──────────────────────┐
│   Envoy Gateway       │  ← 入口网关（HTTP/gRPC proxy）
└──────────┬───────────┘
           │ ext_proc gRPC stream
           ▼
┌──────────────────────┐
│   Semantic Router     │  ← 我们在这里：前置路由层
│   (ext_proc filter)   │
│                      │
│   职责：              │
│   - 意图分类          │
│   - 模型选择          │
│   - 请求改写          │
│   - 安全检测          │
└──────────┬───────────┘
           │ 透传请求到选定的模型
           ▼
┌──────────────────────┐
│   vLLM Engine         │  ← 推理引擎（GPU调度、KV cache等）
│   (OpenAI-compatible) │
└──────────────────────┘
```

**关键区分：** Semantic Router 不做推理，只做路由。它是一个 Envoy ext_proc 插件，在请求到达 vLLM 引擎前决定"由哪个模型处理这个请求"。理解这一点就不会把 routing 逻辑和 serving/inference 逻辑混淆。

---

## 2. 核心四层管道

每个请求通过四层管道转化为最终的模型路由决策：

### 管道全景

```
┌──────────────────────────────────────────────────────────┐
│                     Request Arrives                       │
│  HTTP Headers + OpenAI Chat Completions JSON Body         │
│  {model: "auto", messages: [{role:"user", content:"..."}]}│
└────────────────────────┬─────────────────────────────────┘
                         │
                         ▼
┌──────────────────────────────────────────────────────────┐
│  ╔══════════════════════════════════════════════╗         │
│  ║  Stage 1: Signal Evaluation                 ║         │
│  ║  17 classifiers run in parallel             ║         │
│  ╚══════════════════════════════════════════════╝         │
│  IN:  用户消息 + 对话历史 + HTTP headers                 │
│  OUT: SignalResults (17 个 []string 切片)                │
│                                                            │
│  17 种信号:                                                │
│  keyword → 关键词正则/模糊匹配                              │
│  embedding → 向量相似度                                    │
│  domain → MMLU 领域分类（CS/math/biology...14 种）          │
│  complexity → 问题难度（easy/hard）                         │
│  modality → AR / DIFFUSION / BOTH                         │
│  language → 语言检测（zh/en/es...）                        │
│  context → token 数范围                                    │
│  structure → 问题结构（多问题、编号步骤...）                   │
│  authz → 用户 RBAC 角色（从 header 读取）                   │
│  jailbreak → 越狱检测                                      │
│  pii → PII 检测                                           │
│  kb → 知识库信号                                           │
│  fact_check → 事实核查需求                                  │
│  user_feedback → 用户反馈检测（满意/错误/需要澄清）           │
│  reask → 重复提问不满检测                                   │
│  preference → 偏好路由（外接 LLM）                          │
│  conversation → 对话结构分析                                │
│  projection → 组合信号推导输出                              │
└────────────────────────┬─────────────────────────────────┘
                         │
                         ▼
┌──────────────────────────────────────────────────────────┐
│  ╔══════════════════════════════════════════════╗         │
│  ║  Stage 2: Decision Engine                   ║         │
│  ║  Boolean Rule Tree evaluation               ║         │
│  ╚══════════════════════════════════════════════╝         │
│  IN:  SignalMatches (SignalResults 的等价物)             │
│  OUT: DecisionResult {Decision, Confidence}              │
│                                                            │
│  对每个 decision 检查 rule tree:                           │
│                                                            │
│  code_decision:                    math_decision:          │
│    AND                               OR                     │
│    ├─ domain:computer science        ├─ domain:math         │
│    ├─ keyword:code_keywords          └─ keyword:math_kw    │
│    └─ NOT jailbreak:strict                                │
│                                                            │
│  匹配的 decisions 按 Tier→Confidence→Priority 排序取第一   │
└────────────────────────┬─────────────────────────────────┘
                         │
                         ▼
┌──────────────────────────────────────────────────────────┐
│  ╔══════════════════════════════════════════════╗         │
│  ║  Stage 3: Model Selection                   ║         │
│  ║  Selection Registry (10 algorithms)         ║         │
│  ╚══════════════════════════════════════════════╝         │
│  IN:  Decision.ModelRefs → SelectionContext               │
│  OUT: ModelRef {Model, LoRAName, Weight}                  │
│                                                            │
│  Selection Algorithms:                                     │
│  ┌───────────┬──────────────────────────────────┐         │
│  │ static    │ 配置文件中的静态分数               │         │
│  │ elo       │ Bradley-Terry 成对比较             │         │
│  │ router_dc │ 双对比学习 embedding 匹配          │         │
│  │ automix   │ POMDP 级联：小模型→验证→升级       │         │
│  │ hybrid    │ Elo + RouterDC + Cost 加权混合     │         │
│  │ knn       │ KNN 查询→质量加权投票              │         │
│  │ kmeans    │ 聚类→性能效率路由                  │         │
│  │ svm       │ SVM RBF kernel 分类               │         │
│  │ mlp       │ Candle GPU 神经网络               │         │
│  │ rl_driven │ Thompson Sampling RL 个性化        │         │
│  │ gmtrouter │ 异构图学习个性化                   │         │
│  │ latency   │ TPOT/TTFT 百分位延迟感知           │         │
│  └───────────┴──────────────────────────────────┘         │
└────────────────────────┬─────────────────────────────────┘
                         │
                         ▼
┌──────────────────────────────────────────────────────────┐
│  ╔══════════════════════════════════════════════╗         │
│  ║  Stage 4: Route Output                      ║         │
│  ╚══════════════════════════════════════════════╝         │
│  OUT: selectedModel = "deepseek-v31"                     │
│                                                            │
│  请求被转发到选定模型，response header 注入:                 │
│  x-vsr-selected-model: deepseek-v31                       │
│  x-vsr-selected-decision: code_decision                   │
│  x-vsr-selected-category: computer science                │
│  x-vsr-matched-signals: ...                               │
└──────────────────────────────────────────────────────────┘
```

### 管道中流经的核心类型

| 阶段 | 输入类型 | 输出类型 | 所在文件 |
|------|---------|---------|---------|
| 1. Signal Eval | `signalEvaluationInput` + HTTP headers | `*classification.SignalResults` (17 个 `[]string` 字段) | `req_filter_classification.go` → `classifier_signal_eval.go` |
| 2. Decision Engine | `*decision.SignalMatches`（SignalResults 的拷贝） | `*decision.DecisionResult` (`Decision *config.Decision`, `Confidence float64`, `MatchedRules []string`) | `decision/engine.go` |
| 3. Model Selection | `*selection.SelectionContext` (`Query`, `CategoryName`, `CandidateModels []ModelRef`, `CostWeight/QualityWeight`) | `*config.ModelRef` (`Model`, `LoRAName`, `Weight`) | `selection/selector.go` |
| 4. Route Output | `selectedModel string` | HTTP 请求转发 + VSR response headers | `req_filter_classification.go` |

---

## 3. 关键包职责

```
pkg/
├── config/           ← 配置体系的中心
│   ├── canonical_config.go       ← v0.3 YAML 契约：routing: {signals, decisions, projections}
│   ├── canonical_global.go       ← global: {router, services, stores, integrations}
│   ├── signal_config.go          ← 17 种 signal rule 类型定义
│   ├── decision_config.go        ← Decision + RuleNode 树 + AlgorithmConfig + ModelRef
│   ├── selection_config.go       ← 10 种 selection 算法的配置结构
│   ├── validator.go              ← 配置验证入口（编排 10+ validator）
│   └── validator_*.go            ← 各子系统的合约验证
│
├── classification/   ← 信号分类器（"意图识别"层）
│   ├── classifier_signal_dispatch.go ← 17 个 signalDispatch 并行执行
│   ├── classifier_signal_eval.go     ← EvaluateAllSignalsWithHeaders() 入口
│   └── classifier_signal_*.go        ← 各信号的具体实现
│
├── decision/          ← 决策引擎（"策略匹配"层）
│   ├── engine.go                    ← DecisionEngine + evalNode() 递归求值
│   └── trace.go                     ← 决策追踪树（布尔表达式的可解释性）
│
├── selection/         ← 模型选择（"最优模型"层）
│   ├── selector.go                  ← Selector 接口 + Registry 模式 + SelectionContext
│   ├── elo/ router_dc/ automix/ ... ← 各算法的独立包实现
│   └── feedback.go                  ← 反馈收集与 Elo 更新
│
├── extproc/           ← Envoy ext_proc 集成（"请求生命周期"层）
│   ├── req_filter.go                ← 总入口：请求拦截 + 生命周期管理
│   ├── req_filter_classification.go ← performDecisionEvaluation() 主函数
│   ├── req_filter_classification_runtime.go  ← signal eval + decision engine + selection 调用
│   ├── req_filter_classification_signal.go   ← 信号评估输入准备 + 压缩 + context 注入
│   ├── req_filter_cache.go          ← 语义缓存（handleCaching → FindSimilar → cache hit/miss）
│   └── req_filter_looper.go         ← 多模型循环（confidence/ratings/remom 算法）
│
├── headers/           ← HTTP header 常量定义（所有 x-vsr-* headers）
├── authz/             ← 认证链（CredentialResolver → HeaderInjectionProvider → StaticConfigProvider）
├── routerreplay/      ← 路由决策回放（Record → Writer/Reader/Enricher → memory/redis/postgres/milvus）
└── modelselection/    ← ML 模型选择（Linfa/Candle Rust binding：KNN/KMeans/SVM/MLP）
```

**依赖方向（上层依赖下层，下层不知上层）：**

```
extproc ──→ classification ──→ config
  │              │
  └──→ decision ─┘         selection
         │                    │
         └──→ config ←────────┘
```

---

## 4. 一次请求的完整时间线

```
0ms  │  Envoy 收到请求，触发 ext_proc RequestHeaders 阶段
     │
1ms  │  extproc.req_filter.go: 解析 OpenAI JSON body
     │  提取 model 字段、messages 数组
     │
2ms  │  ├─ handleCaching(): 检查语义缓存
     │  │   └─ 命中 → 直接返回缓存响应（含 VSR headers），结束
     │  │   └─ 未命中 → 注册 pending cache write，继续
     │
3ms  │  ├─ performDecisionEvaluation():
     │  │
     │  │   ├─ prepareSignalEvaluationInput()
     │  │   │   提取 user message + conversation history
     │  │   │   PromptCompression（如果启用）：TextRank + Position + TF-IDF
     │  │   │
     │  │   ├─ evaluateSignalsForDecision()          ← ⚡ 最耗时阶段
     │  │   │   17 个 signal dispatch 并行执行
     │  │   │   每个 dispatch: 正则/embedding/LLM调用
     │  │   │   → *classification.SignalResults
     │  │   │
     │  │   ├─ runDecisionEngine()
     │  │   │   SignalResults → SignalMatches
     │  │   │   对每个 decision: evalNode(Rules, signals)
     │  │   │   selectBestDecision(matched)
     │  │   │   → *decision.DecisionResult
     │  │   │
     │  │   └─ finalizeDecisionEvaluation()
     │  │       ├─ applyDecisionResultToContext()
     │  │       │   设置 ctx.VSRSelectedDecision / Category
     │  │       │
     │  │       └─ selectDecisionRuntimeModel()      ← ⚡ 仅 auto model
     │  │           ├─ buildSelectionContext()
     │  │           │   组装 CandidateModels + weights + cache affinity
     │  │           │
     │  │           └─ selectModelFromCandidates()
     │  │               registry.Get(method).Select(ctx, selCtx)
     │  │               → "deepseek-v31"
     │  │
10ms │  ├─ 决策插件执行（system prompt 注入、RAG、memory、hallucination detection）
     │
11ms │  ├─ 修改请求 body 中的 model 字段为 "deepseek-v31"
     │  ├─ 设置 x-vsr-destination-endpoint header
     │  └─ 记录 router replay
     │
12ms │  └─ 返回给 Envoy：继续转发请求到目标模型
     │
     │  ... upstream (vLLM engine) 处理 ...
     │
500ms│  ResponseHeaders 阶段: 注入 VSR response headers
     │  ResponseBody 阶段: 保存 response body 到 cache + replay
```

---

## 5. Extension Points（扩展点）

理解这些扩展点可以帮助判断一个新功能应该在哪一层实现：

| 扩展需求 | 插入位置 | 机制 |
|---------|---------|------|
| 新增一种信号 | `classification/` + `config/signal_config.go` | 实现新的 `signalDispatch`，注册到 `buildSignalDispatchers()` |
| 新增一种 decision 匹配策略 | `decision/engine.go` | 扩展 `selectBestDecision()` |
| 新增一种 model selection 算法 | `selection/` 下新建包 | 实现 `Selector` 接口，注册到 `GlobalRegistry` |
| 新增路由干预方式（hint/binding） | `extproc/req_filter_classification.go` | 在 `performDecisionEvaluation()` 中按优先级插入拦截分支 |
| 新增一种 response 处理 | `extproc/req_filter.go` | ResponseHeaders/ResponseBody 回调 |
| 新增 config 契约 | `config/canonical_config.go` | 扩展 `CanonicalRouting`，添加对应的 validator |
| 新增 replay 存储后端 | `routerreplay/store/` | 实现 `Storage` 接口（Writer+Reader+Enricher），注册到 factory |

---

## 6. 重要设计模式

### 6.1 Registry 模式（selection 层）

```go
// 所有 selection 算法通过统一接口注册
type Selector interface {
    Select(ctx context.Context, selCtx *SelectionContext) (*SelectionResult, error)
    Method() SelectionMethod
    Tier() AlgorithmTier
    UpdateFeedback(ctx context.Context, feedback *Feedback) error
}

// 全局注册表在 init() 中注册各算法
var GlobalRegistry = NewRegistry()
// elo.Register(), router_dc.Register(), ...
```

好处：新增算法只需实现接口并注册，不改动 extproc 层代码。

### 6.2 RuleNode 布尔树（decision 层）

```yaml
# YAML 中表达：
decisions:
  - name: "code_decision"
    rules:
      operator: "AND"
      conditions:
        - type: "domain"
          name: "computer science"
        - operator: "NOT"
          conditions:
            - type: "jailbreak"
              name: "strict"
```

递归求值（`evalNode` → `evalAND`/`evalOR`/`evalNOT` → `evalLeaf`），支持任意嵌套深度。

### 6.3 Credential Chain（authz 层）

```
HeaderInjectionProvider  →  StaticConfigProvider
(ext_authz header)          (YAML access_key)
       ↓ 失败                      ↓ 失败
       └────────── fail-open/fail-closed ──────────┘
```

Provider chain 模式：第一个成功的 provider 返回 key，后续跳过。支持 fail-open（无认证后端也可用）和 fail-closed（安全优先）。

### 6.4 Cache-Aside 模式（cache 层）

```
请求 → cache lookup → hit → 直接返回（不跑信号/decision/selection）
                    → miss → 完整管道 → 异步写入 cache
```

特殊处理：RAG/memory 个性化请求跳过 cache（读+写都跳），避免泛化/隐私泄露。

---

## 7. 代码阅读路线

**第一次阅读建议按这个顺序：**

1. `pkg/headers/headers.go` — 先熟悉所有 header 常量，了解系统有哪些概念
2. `pkg/config/config.go` + `pkg/config/decision_config.go` — 理解 `RouterConfig`、`Decision`、`RuleNode`、`ModelRef`
3. `pkg/config/signal_config.go` — 理解 17 种 signal rule 的定义
4. `pkg/decision/engine.go` — 理解 decision engine 如何工作（`evalNode` 函数是核心）
5. `pkg/selection/selector.go` — 理解 `SelectionContext` 和 `Selector` 接口
6. `pkg/extproc/req_filter.go` — 理解 ext_proc 的整体生命周期
7. `pkg/extproc/req_filter_classification.go` — 理解 `performDecisionEvaluation()` 主函数
8. `pkg/extproc/req_filter_classification_runtime.go` — 理解信号评估→决策→选模型的具体调用
9. `pkg/classification/classifier_signal_dispatch.go` — 理解 17 种信号的并行调度
10. `pkg/authz/chain.go` — 理解 credential 链
