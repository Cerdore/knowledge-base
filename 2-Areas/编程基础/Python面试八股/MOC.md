# Python 面试八股 知识地图

> Python 面试高频问题整理（适合 AI Infra 岗位）

---

## 📚 GitHub 资源

| 仓库 | Stars | 说明 |
|------|-------|------|
| [python_interview_questions](https://github.com/yakimka/python_interview_questions) | 2.5k⭐ | Python Developer 面试问题 |
| [python-interview-questions](https://github.com/Devinterview-io/python-interview-questions) | 400⭐ | Python 面试问题与答案 |
| [300-python-Interview-questions](https://github.com/kiransagar1/300-python-Interview-questions-and-solutions) | 140⭐ | 300道Python面试题 |

---

## 📋 知识模块

### 一、Python 基础

- [[Python-数据类型]]
- [[Python-字符串]]
- [[Python-列表与元组]]
- [[Python-字典]]
- [[Python-集合]]
- [[Python-深拷贝与浅拷贝]]
- [[Python-可变与不可变]]

### 二、函数与装饰器

- [[Python-函数参数]]
- [[Python-闭包]]
- [[Python-装饰器]]
- [[Python-生成器]]
- [[Python-迭代器]]
- [[Python-yield]]
- [[Python-lambda]]

### 三、面向对象

- [[Python-类与对象]]
- [[Python-继承与多态]]
- [[Python-魔术方法]]
- [[Python-属性访问]]
- [[Python-MRO]]
- [[Python-元类]]

### 四、并发编程

- [[Python-多线程]]
- [[Python-多进程]]
- [[Python-协程]]
- [[Python-asyncio]]
- [[Python-GIL]]
- [[Python-线程池]]
- [[Python-进程池]]

### 五、内存管理

- [[Python-内存模型]]
- [[Python-GC机制]]
- [[Python-引用计数]]
- [[Python-内存泄漏]]
- [[Python-对象池]]

### 六、模块与包

- [[Python-导入机制]]
- [[Python-包管理]]
- [[Python-虚拟环境]]
- [[Python-pip]]
- [[Python-命名空间]]

### 七、常用库

- [[Python-os与sys]]
- [[Python-re正则]]
- [[Python-json]]
- [[Python-logging]]
- [[Python-unitest]]
- [[Python-typing]]

### 八、AI Infra 相关

- [[Python-PyTorch基础]]
- [[Python-NumPy]]
- [[Python-CUDA绑定]]
- [[Python-性能优化]]
- [[Python-C扩展]]

---

## 📝 高频面试题

### 基础
1. Python 数据类型有哪些？可变 vs 不可变？
2. list vs tuple？什么时候用 tuple？
3. dict 实现原理？哈希冲突处理？
4. 深拷贝 vs 浅拷贝？

### 函数
1. 装饰器原理？如何带参数？
2. 生成器 vs 迭代器？
3. yield 原理？
4. 闭包原理？变量捕获？

### 面向对象
1. __new__ vs __init__？
2. MRO 如何计算？
3. 元类用途？
4. @property 原理？

### 并发
1. GIL 是什么？为什么有 GIL？
2. 多线程 vs 多进程 vs 协程？
3. asyncio 原理？
4. 如何绕过 GIL？

### 内存
1. Python GC 机制？
2. 引用计数 + 分代回收？
3. 内存泄漏排查？
4. __slots__ 作用？

### AI Infra 相关
1. PyTorch tensor vs NumPy array？
2. 如何写 CUDA kernel 绑定？
3. Python 性能优化技巧？
4. ctypes vs cffi？

---

## 🎯 AI Infra 重点

| 场景 | Python 知识点 |
|------|--------------|
| 推理引擎 | asyncio、协程、并发 |
| 数据处理 | NumPy、生成器、内存优化 |
| CUDA 绑定 | ctypes、cffi、C扩展 |
| 模型训练 | PyTorch、typing |

---

## 📅 学习进度

- [ ] Python 基础
- [ ] 函数与装饰器
- [ ] 面向对象
- [ ] 并发编程
- [ ] 内存管理
- [ ] AI Infra 相关

---

*最后更新: 2026-04-15*