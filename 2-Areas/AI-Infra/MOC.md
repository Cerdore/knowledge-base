# AI Infra 知识地图

> 从数据库内核转向 AI Infra 的学习路线

## 🎯 转型方向

### 方向一：AI 数据组织（数据开发）
适合数据库背景，技能迁移度高
- [[Apache-Iceberg]] — 数据湖表格式
- [[Ray]] — 分布式 AI 计算
- [[Spark]] — 大数据处理
- [[Flink]] — 流批一体

### 方向二：推理引擎开发
难度较高，需要 GPU 编程基础
- [[CUDA]] — GPU 编程基础
- [[cuDNN]] — 深度学习算子库
- [[CUTLASS]] — 高性能矩阵计算
- [[vLLM]] — PagedAttention 推理引擎
- [[SGLang]] — 结构化生成语言
- [[TensorRT-LLM]] — NVIDIA 推理优化

---

## 📚 核心概念链

### LLM 基础
```
[[Transformer]] → [[KV-Cache]] → [[PD分离]]
[[Attention变体]] → [[FlashAttention]] → [[PagedAttention]]
```

### 推理优化
```
[[Continuous-Batching]] → [[Chunked-Prefill]] → [[Speculative-Decoding]]
[[KV-Cache-压缩]] → [[Prefix-Caching]] → [[KV-Cache-池化]]
```

### 系统架构
```
[[Mooncake]] → [[DistServe]] → [[LoongServe]]
```

---

## 📖 必读论文

| 论文 | 会议 | 重点 |
|------|------|------|
| [[Mooncake-FAST2025]] | FAST 2025 ⭐ | PD分离 + KV Cache池化 |
| [[DistServe-OSDI2024]] | OSDI 2024 | PD分离架构 |
| [[vLLM-SOSP2023]] | SOSP 2023 | PagedAttention |
| [[FlashAttention-ICML2022]] | ICML 2022 | IO感知Attention |

---

## 🔗 技能迁移：数据库 → AI Infra

| 数据库内核技能 | AI Infra 对应 |
|---------------|---------------|
| 存储引擎 | KV Cache 存储/管理 |
| 查询优化 | 推理调度优化 |
| 事务并发 | 请求批处理/调度 |
| 内存管理 | GPU 内存池化 |
| 缓冲池 | KV Cache 池化 |

---

## 📅 学习进度

- [ ] Transformer 基础复习
- [ ] KV Cache 原理深入
- [ ] PD分离架构理解
- [ ] CUDA 编程入门
- [ ] vLLM 源码阅读
- [ ] Mooncake 论文精读

---

## 📝 面试高频问题

1. KV Cache 的作用和优化方法？
2. PD分离为什么能提升吞吐？
3. PagedAttention 如何解决内存碎片？
4. Mooncake 的调度策略？
5. CUDA kernel 优化思路？

---

*最后更新: 2026-04-14*