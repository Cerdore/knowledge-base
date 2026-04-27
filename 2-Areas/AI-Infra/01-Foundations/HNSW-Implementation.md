# HNSW 实现深度分析

## 概述

HNSW (Hierarchical Navigable Small World) 是 semantic-router 中 Semantic Cache 插件的核心向量搜索加速器。它将缓存查找从 O(n) 线性扫描降至 O(log n)。

## 在架构中的位置

```
请求 → RouteServer → Decision Matched → Post-Route Plugins
                                              │
                                              └─ Semantic Cache Plugin
                                                      │
                                                      ├─ InMemoryCache.FindSimilar()
                                                      │       └─ hnswIndex.searchKNN()
                                                      │
                                                      └─ HybridCache.FindSimilarWithThreshold()
                                                              └─ searchKNNHybridWithThreshold()
```

## 两套 HNSW 实现

semantic-router 中存在两套**独立**的 HNSW 实现，各自有独立的 SIMD 汇编文件：

| 实现 | 位置 | 用途 |
|---|---|---|
| 独立 `hnsw` 包 | `pkg/hnsw/hnsw.go` | 通用实现，当前未在生产路由路径中直接调用 |
| 缓存嵌入式 HNSW | `pkg/cache/inmemory_cache.go` | 生产路径实现，与 CacheBackend 紧密集成 |

两套实现的 SIMD 文件独立不共享：`pkg/hnsw/simd_distance_amd64.s` 和 `pkg/cache/simd_distance_amd64.s`。

## 数据结构（缓存 HNSW）

```go
type HNSWNode struct {
    entryIndex int           // 指向 entries 切片的索引
    neighbors  map[int][]int // 层 → 邻居索引列表
    maxLayer   int           // 该节点的最高层
}

type HNSWIndex struct {
    nodes          []*HNSWNode     // 节点切片
    nodeIndex      map[int]*HNSWNode // O(1) 查找
    entryPoint     int             // 全局入口点
    maxLayer       int
    M              int             // 每层双向链接数 (默认 16)
    Mmax           int             // 上层 M
    Mmax0          int             // 层 0 M*2
    efConstruction int             // 构建时动态候选列表大小 (默认 200)
    ml             float64         // 1/ln(M)，层级概率参数
}
```

## 核心算法

### 构建（`addNode()`）

1. **层级分配** — `selectLevel()` 用指数衰减随机分配：`level = int(-log(rand.Float64()) * ml)`，其中 `ml = 1/ln(M)`。约 93.75% 节点在层 0。
2. **入口点下降** — 首个节点成为全局入口点。后续节点从入口点开始，逐层贪心 1-NN 下降找到该层的进入点。
3. **邻居连接** — 从节点层级到层 0 的每一层：
   - 用 `efConstruction` 搜索候选邻居
   - 选择 M 个最近邻居（层 0 用 Mmax0 = M*2）
   - 创建双向链接
   - 若邻居超过 M 条边，用 `selectNeighbors()` 剪枝
4. **入口点更新** — 若节点层级 > maxLayer，成为新入口点。

### 搜索（`searchKNN()` / `searchLayer()`）

1. 从入口点、最高层开始
2. 每层（> 0）做贪心 1-NN 下降
3. 层 0 做完整 ef-NN 搜索
4. 使用两个优先队列：
   - **minHeap** — 待探索候选（按距离升序）
   - **maxHeap** — 当前最佳结果（按距离降序，用于快速淘汰）
5. 结果按距离升序返回（最近优先）

### 距离计算

距离 = **负点积**（假设向量已 L2 归一化）：
```
distance(a, b) = -dot_product(a, b)
```
归一化后点积 = 余弦相似度，取负后距离越小越相似（适配优先队列语义）。

## SIMD 加速

dot_product 通过 CPU 特性检测在运行时选择最快路径：

| 实现 | 并行度 | 条件 |
|---|---|---|
| AVX-512 | 16-wide float32 | `minLen >= 16` + CPU 支持 |
| AVX2 | 8-wide float32 | `minLen >= 8` + CPU 支持 |
| Scalar | 1-wide float32 | 兜底 |

- CPU 检测：`golang.org/x/sys/cpu`
- 构建标签：`amd64 && !purego` (SIMD)，`!amd64 || purego` (兜底)
- 手写 Go 汇编文件 (`.s`)

## 限制

- **不支持删除** — 缓存条目过期/驱逐后标记索引为 stale，下次插入触发全量 `rebuildHNSWIndex()`
- **HybridCache** 的 HNSW 强制启用；**InMemoryCache** 的 HNSW 通过 `UseHNSW` 配置可选

## 配置参数

| 参数 | 默认值 | 说明 |
|---|---|---|
| `M` | 16 | 每层双向链接数 |
| `Mmax0` | M*2 | 层 0 最大链接数 |
| `efConstruction` | 200 | 构建时动态候选列表 |
| `efSearch` | 50 | 搜索时动态候选列表 |

## Bug 记录

### Bug 1: 插入入口点下降错误 (PR #1787)

**状态**: 已合并到 main

**问题**: `addNode()` 从上层下降到下层时，每层都从全局 `entryPoint` 重新开始搜索，而非携带上层找到的最佳候选。

**影响**: 层 0 邻居连接到错误的节点（靠近全局入口点但远离真实最近邻居）。

**修复**: 引入 `currentEntryPoint` 变量，逐层传递上层搜索结果：
```go
currentEntryPoint := h.entryPoint
for lc := h.maxLayer; lc > level; lc-- {
    candidates := h.searchLayer(embedding, currentEntryPoint, 1, lc, entries)
    if len(candidates) > 0 {
        currentEntryPoint = candidates[0]  // 传递给下一层
    }
}
```

同期修复了堆极性 bug（trim 时可能驱逐最佳元素）和提前退出方向错误。

### Bug 2: 时间戳伪随机 (commit `f3ef2066`)

**状态**: 在分支 `fix/hnsw-randomness` 上，**尚未合并到 main**

**问题**: `selectLevel()` 使用 `time.Now().UnixNano() % 1000000` 作为随机源。批量插入时同一纳秒内多次调用返回相同值。

**影响**: 所有节点分配到同一层级，HNSW 退化为单层 O(n) 线性搜索。

**性能对比** (384 维, k=10, Apple M1 Pro, 倍数为线性搜索基线):

| N | 退化前 | 修复后 |
|---|--------|--------|
| 1,000 | 0.7x | 0.9x |
| 5,000 | 1.3x | 2.2x |
| 10,000 | 2.2x | 3.7x |
| 20,000 | 3.1x | 6.3x |

**修复**: 替换为 `math/rand/v2` 的 `rand.Float64()`（Go 1.22+ 自动种子，并发安全）。

## 与其他向量索引的关系

HNSW 仅在 **Go 层**的 Semantic Cache 中使用。其他场景：

- **MemoryBackend** (VectorStore): 暴力余弦相似度，无索引
- **MilvusBackend**: Milvus 内置 HNSW
- **ValkeyBackend**: Valkey 原生 HNSW（可配置 `index_m`, `index_ef_construction`）
- **RedisCache**: Redis 内置向量索引
