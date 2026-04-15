# C++ 面试八股 知识地图

> C++ 面试高频问题整理（适合数据库内核/AI Infra 岗位）

---

## 📚 GitHub 资源

| 仓库 | Stars | 说明 |
|------|-------|------|
| [cpp_interview](https://github.com/guaguaupup/cpp_interview) | 2.2k⭐ | C++后台服务器开发面经，有深度有广度 |
| [cpp_Interview](https://github.com/SYaoJun/cpp_Interview) | 500⭐ | C/C++ study notes |
| [cracking_the_cpp_interview](https://github.com/sky-co/cracking_the_cpp_interview) | 69⭐ | Cracking the C++ interview |

---

## 📋 知识模块

### 一、C++ 基础

- [[CPP-指针与引用]]
- [[CPP-内存管理]]
- [[CPP-智能指针]]
- [[CPP-const]]
- [[CPP-static]]
- [[CPP-inline]]
- [[CPP-volatile]]

### 二、面向对象

- [[CPP-类与对象]]
- [[CPP-构造与析构]]
- [[CPP-继承与多态]]
- [[CPP-虚函数]]
- [[CPP-虚继承]]
- [[CPP-访问控制]]

### 三、现代 C++（C++11/14/17/20）

- [[CPP-auto与decltype]]
- [[CPP-lambda表达式]]
- [[CPP-右值引用与移动语义]]
- [[CPP-perfect-forwarding]]
- [[CPP-智能指针详解]]
- [[CPP-并发编程]]
- [[CPP-模板元编程]]

### 四、STL

- [[CPP-vector]]
- [[CPP-map与unordered_map]]
- [[CPP-迭代器]]
- [[CPP-算法]]
- [[CPP-内存分配器]]

### 五、内存模型

- [[CPP-内存布局]]
- [[CPP-堆与栈]]
- [[CPP-内存对齐]]
- [[CPP-RAII]]
- [[CPP-对象生命周期]]

### 六、并发编程

- [[CPP-thread]]
- [[CPP-mutex与lock]]
- [[CPP-condition_variable]]
- [[CPP-atomic]]
- [[CPP-memory_order]]
- [[CPP-future与promise]]

### 七、编译与链接

- [[CPP-编译过程]]
- [[CPP-链接原理]]
- [[CPP-静态库与动态库]]
- [[CPP-符号解析]]
- [[CPP-ABI]]

### 八、性能优化

- [[CPP-性能分析工具]]
- [[CPP-缓存优化]]
- [[CPP-分支预测]]
- [[CPP-内存池]]
- [[CPP-零拷贝]]

---

## 📝 高频面试题

### 基础
1. 指针和引用的区别？
2. 智能指针原理？shared_ptr 循环引用问题？
3. new/delete vs malloc/free？
4. const 有哪些用法？

### 面向对象
1. 虚函数原理？虚表布局？
2. 构造函数可以是虚函数吗？
3. 析构函数为什么要虚函数？
4. 多继承的内存布局？

### 现代 C++
1. move 语义原理？
2. lambda 实现原理？
3. perfect forwarding 原理？
4. std::function vs 函数指针？

### STL
1. vector 扩容机制？
2. map vs unordered_map？
3. iterator 失效场景？
4. allocator 原理？

### 并发
1. std::mutex 原理？
2. memory_order 各选项含义？
3. 死锁检测与避免？
4. lock-free 编程？

### 编译链接
1. 编译过程四个阶段？
2. 静态链接 vs 动态链接？
3. 符号冲突如何解决？
4. ODR 原则？

---

## 🎯 数据库内核/AI Infra 重点

| 场景 | C++ 知识点 |
|------|-----------|
| 存储引擎 | 内存管理、RAII、内存池 |
| KV Cache | 智能指针、并发容器 |
| CUDA 编程 | 内存对齐、零拷贝 |
| 推理引擎 | 性能优化、缓存友好 |

---

## 📅 学习进度

- [ ] C++ 基础
- [ ] 面向对象
- [ ] 现代 C++
- [ ] STL
- [ ] 并发编程
- [ ] 编译链接
- [ ] 性能优化

---

*最后更新: 2026-04-15*