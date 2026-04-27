# Knowledge Base

> 个人知识管理库 — 从数据库内核转向 AI Infra（推理方向）

---

## 目录结构

```
Knowledge-Base/
│
├── 0-Inbox/               # 快速捕获：临时笔记、待整理资料、草稿方案
│                           # 定期清空 — 要么归档到对应目录，要么删除
│
├── 1-Projects/             # 当前项目：有明确目标和截止日期的短期任务
│   └── 面试准备/           #   AI Infra 面试问题清单与答案
│
├── 2-Areas/                # 持续领域：长期关注、持续积累的知识领域
│   ├── AI-Infra/           #   🎯 转型目标 — 推理优化全栈知识
│   ├── 数据库内核/         #   💼 当前技能 — 存储/查询/事务
│   └── 编程基础/           #   基础技能 — C++/Java/Python 面试八股
│
├── 3-Resources/            # 参考资料：外部资源、书籍笔记、论文笔记、代码片段
│   ├── books/              #   书籍原文（PDF 等二进制不入 git）
│   ├── 论文笔记/           #   论文精读笔记
│   ├── 书籍笔记/           #   读书笔记
│   └── 代码片段/           #   可复用的代码模板
│
├── 4-Archives/             # 归档：已完成项目、过时资料、不再活跃的领域
│
└── 5-Templates/            # 笔记模板：Concept / Paper / Insight / Experience 等
```

---

## AI-Infra 知识体系（三层优化架构）

基于 arXiv:2404.14294v3 综述的三层分类体系：

```
2-Areas/AI-Infra/
├── MOC.md                  # 纯导航地图
├── 00-Overview/            # 总览：AI Infra 全景 + 综述笔记
├── 01-Foundations/         # 基础：Transformer / KV Cache / GPU / CUDA / HNSW
├── 02-Data-Level/          # 数据层：输入压缩、输出组织
├── 03-Model-Level/         # 模型层：MoE / GQA / 量化 / 剪枝 / 蒸馏
│   └── embedding/          #   Embedding 模型对比分析
├── 04-System-Level/        # 系统层：FlashAttention / PD分离 / 调度 / 分布式
├── 05-Frameworks/          # 框架：vLLM / SGLang / TensorRT-LLM / Mooncake / Semantic-Router
├── 06-Papers/              # 论文精读
├── 07-DB-to-AIInfra/       # 迁移路径：技能对应表 + 学习路线
└── 08-Interview/           # 面试：高频八股 + 系统设计题
```

---

## 使用方法

### 1. 快速捕获（Inbox）
新想法、链接、草稿先放入 `0-Inbox/`，定期整理到对应目录。

### 2. 知识检索
- 通过 MOC（Map of Content）导航：每个领域都有一个 MOC.md 作为入口
- Obsidian 图谱 + 双链 `[[...]]` 跳转
- 从 `2-Areas/AI-Infra/MOC.md` 开始是推荐入口

### 3. 创建笔记
- 从 `5-Templates/` 选择对应模板
- Concept 模板适合技术概念
- Paper 模板适合论文精读
- Insight 模板适合记录"顿悟时刻"

### 4. 填充优先级（AI-Infra）
1. `01-Foundations/KV-Cache.md` — 所有问题的基础
2. `04-System-Level/serving-system/KV-Cache-Management.md` — PagedAttention
3. `04-System-Level/serving-system/PD-Separation.md` — Mooncake 核心
4. `04-System-Level/inference-engine/FlashAttention.md` — 系统层必问
5. `04-System-Level/inference-engine/Speculative-Decoding.md` — 无损加速
6. `08-Interview/Common-Questions.md` — 面试直接使用

---

## 核心知识地图

- [[2-Areas/AI-Infra/MOC]] — AI Infra 知识地图（主入口）
- [[2-Areas/数据库内核/MOC]] — 数据库内核领域
- [[2-Areas/编程基础/MOC]] — 编程基础与面试八股

---

## 配套工具

- **Obsidian** — 笔记管理与双链图谱
- **skill-graphify** — 论文/代码库知识提取
- **obsidian-daily** — 每日学习记录

---

*创建日期: 2026-04-14 | 结构重构: 2026-04-25*
