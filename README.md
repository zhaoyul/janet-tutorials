# Janet 编程语言深度教程

> 为熟悉 Lisp 生态的开发者准备的系统级编程指南

## 关于本教程

本教程面向对 Lisp 生态（Common Lisp、Scheme、Clojure）有深入了解的资深开发者，旨在帮助你快速掌握 Janet 语言的核心特性，并能够使用它进行系统级开发。

Janet 是一门现代的动态函数式编程语言，结合了 Lisp 的强大表达力、Clojure 的设计理念，以及 Lua 的轻量级特性。它特别适合：
- 系统脚本开发
- 嵌入式应用场景
- 快速原型开发
- 网络服务编写
- C/C++ 项目的脚本层

## 为什么选择 Janet？

从 Lisp 开发者的角度看，Janet 的独特优势：

1. **极小的体积** - 完整运行时 < 1MB，比 Racket 或 Clojure/JVM 小数百倍
2. **快速启动** - 毫秒级启动，适合作为系统脚本替代 shell/Python
3. **原生执行** - 直接编译为本地可执行文件，无需 VM
4. **可嵌入性** - 单个 C 文件即可嵌入，比 Guile Scheme 更简单
5. **现代特性** - 内置协程、事件循环、PEG 解析器、网络支持
6. **Clojure 风格** - 借鉴了 Clojure 的数据结构和语法习惯
7. **强大的 FFI** - 直接调用 C 库，无需编写绑定代码

## 教程目录

### [第一章：快速入门](./01-getting-started.md)
- 安装和环境配置
- REPL 使用
- 第一个 Janet 程序
- 与其他 Lisp 的语法对比

### [第二章：核心语言特性](./02-core-language.md)
- S-表达式和基本语法
- 词法作用域与闭包
- 求值模型
- 特殊形式详解
- 与 Scheme/Common Lisp/Clojure 的区别

### [第三章：数据结构与类型系统](./03-data-structures.md)
- 基本类型：nil, boolean, number, string, buffer, symbol, keyword
- 集合类型：tuple, array, struct, table
- 持久化数据结构的设计
- 类型系统与类型检查
- 模式匹配

### [第四章：函数式编程](./04-functional-programming.md)
- 函数定义与调用
- 高阶函数
- 闭包与词法作用域
- 函数组合与管道
- 惰性序列
- 递归与尾调用优化

### [第五章：宏与元编程](./05-macros-metaprogramming.md)
- 宏系统基础
- 准引用（quasiquote）与反引用
- 编译期求值
- 代码生成模式
- 与 Common Lisp 宏的对比
- 卫生宏（Hygiene）问题

### [第六章：模块与项目组织](./06-modules-projects.md)
- 模块系统
- import 与 require
- 包管理器 JPM
- 项目结构最佳实践
- 构建和分发

### [第七章：并发编程：纤程与事件循环](./07-concurrency.md)
- 纤程（Fibers）基础
- 协程式并发模型
- 事件循环（Event Loop）
- 异步 I/O
- 与 Scheme 的 call/cc 对比
- 与 Go 的 goroutine 对比

### [第八章：FFI 与 C 互操作](./08-ffi-c-interop.md)
- FFI 模块使用
- 调用 C 函数
- 结构体与指针操作
- 嵌入 Janet 到 C 程序
- 编写 Janet C 扩展
- 与 Common Lisp CFFI 对比

### [第九章：网络编程](./09-networking.md)
- TCP/UDP Socket 编程
- HTTP 客户端与服务器
- 异步网络 I/O
- 实战：构建 Web 服务器
- 实战：构建 REST API

### [第十章：系统编程](./10-system-programming.md)
- 文件系统操作
- 进程管理
- 信号处理
- 系统调用
- 跨平台编程考虑
- 实战：系统监控工具

### [第十一章：PEG 解析器](./11-peg-parsing.md)
- PEG 语法入门
- 词法分析与语法分析
- 构建 DSL
- 实战：配置文件解析器
- 实战：简单编译器前端

### [第十二章：性能优化](./12-performance.md)
- Janet 性能模型
- 性能分析工具
- 内存管理与 GC
- 优化技巧
- 与 C 代码集成提升性能

### [第十三章：测试与调试](./13-testing-debugging.md)
- 单元测试
- 断言与错误处理
- REPL 驱动开发
- 调试技巧
- 日志与追踪

### [第十四章：实战项目](./14-practical-projects.md)
- 项目 1：命令行工具（CLI Tool）
- 项目 2：网络爬虫
- 项目 3：日志分析器
- 项目 4：简单的脚本语言解释器
- 项目 5：系统监控守护进程

### [第十五章：Spork 扩展标准库](./15-spork-library.md)
- Spork 概览与安装
- 模块地图与功能分类
- 典型模块实战
- Spork 工具链与项目集成

### [附录 A：与其他 Lisp 的详细对比](./appendix-a-lisp-comparison.md)
- Janet vs Common Lisp
- Janet vs Scheme (Racket, Guile)
- Janet vs Clojure
- Janet vs other Lisps (newLISP, PicoLisp)
- 迁移指南

### [附录 B：Janet 核心 API 速查](./appendix-b-api-reference.md)
- 核心函数速查表
- 常用宏速查表
- 标准库概览

### [附录 C：资源与社区](./appendix-c-resources.md)
- 官方文档
- 优秀的第三方库
- 社区与贡献指南
- 推荐阅读

## 学习路径建议

### 快速路径（1-2天）
如果你已经熟悉 Lisp，可以快速浏览：
- 第一章：快速入门
- 第二章：核心语言特性
- 第八章：FFI 与 C 互操作
- 第十四章：实战项目

### 系统级开发路径（1周）
专注于系统编程：
- 第一至六章：基础部分
- 第七章：并发编程
- 第八章：FFI 与 C 互操作
- 第九章：网络编程
- 第十章：系统编程
- 第十四章：实战项目

### 完整学习路径（2-3周）
深入学习所有内容，包括高级主题和最佳实践。

## 前置要求

- 熟悉至少一种 Lisp 方言（Common Lisp、Scheme、Clojure 等）
- 理解函数式编程概念
- 基础的 C 语言知识（用于理解 FFI）
- Unix/Linux 基础知识

## 如何使用本教程

1. **动手实践** - 每个章节都包含可运行的代码示例，建议在 REPL 中逐一尝试
2. **完成练习** - 章节末尾的练习题帮助巩固知识
3. **参考对比** - 充分利用与其他 Lisp 的对比说明，快速理解差异
4. **实战项目** - 通过实战项目将所学知识整合应用

## 配套代码

所有示例代码位于对应章节的目录中，可以直接运行：

```bash
# 克隆仓库
git clone https://github.com/zhaoyul/janet-tutorials.git
cd janet-tutorials

# 安装 Janet（如果还未安装）
# 参见第一章的详细说明

# 运行示例
janet 01-getting-started/examples/hello.janet
```

## 贡献

欢迎贡献改进建议、错误修正和新的示例！请提交 Issue 或 Pull Request。

## 许可证

本教程采用 MIT 许可证。

## 致谢

感谢 Janet 语言的创建者 Calvin Rose 和整个 Janet 社区。

---

开始你的 Janet 之旅：[第一章：快速入门](./01-getting-started.md) →
