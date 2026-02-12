# 第六章：模块与项目组织

Janet 提供了简洁而强大的模块系统，本章介绍如何组织和管理 Janet 项目。

## 6.1 模块基础

### 创建模块

```janet
# mymodule.janet
(defn add [x y] (+ x y))
(defn multiply [x y] (* x y))

(def PI 3.14159)

# 导出符号（可选，默认导出所有）
```

### 导入模块

```janet
# 基本导入
(import ./mymodule)
(mymodule/add 1 2)

# 带前缀导入
(import ./mymodule :prefix "")
(add 1 2)  # 直接使用

# 重命名
(import ./mymodule :as m)
(m/add 1 2)

# 只导入特定符号
(import ./mymodule :prefix "" :only [add multiply])
(add 1 2)
(multiply 3 4)
```

## 6.2 模块路径

### 路径解析

```janet
# 相对路径
(import ./mymodule)       # 当前目录
(import ../utils/helper)  # 上级目录

# 绝对路径
(import /usr/local/lib/janet/mylib)

# 模块路径搜索（类似 Python 的 sys.path）
(module/paths)  # 查看搜索路径
```

### 模块加载机制

```janet
# import 会：
# 1. 查找模块文件
# 2. 编译（如果需要）
# 3. 执行模块代码
# 4. 缓存结果

# 强制重新加载
(import ./mymodule :fresh true)

# 检查是否已加载
(module/cache)  # 查看已加载模块
```

## 6.3 私有与公共

### 私有定义

```janet
# mymodule.janet

# 公共函数
(defn public-function [x]
  (+ x (private-helper x)))

# 私有函数（- 前缀约定）
(defn- private-helper [x]
  (* x 2))

# 使用
(import ./mymodule)
(mymodule/public-function 5)   # OK
# (mymodule/private-helper 5)  # 不可访问（约定）
```

### 显式导出

```janet
# mymodule.janet

(def- internal-value 42)  # 私有
(def public-value 100)    # 公共

# 只导出特定符号
(def exports
  {:add add
   :multiply multiply})

# 或使用 module/export
```

## 6.4 包管理器 JPM

### JPM 基础

```bash
# 初始化项目
jpm init

# 安装依赖
jpm deps

# 构建项目
jpm build

# 运行测试
jpm test

# 清理构建产物
jpm clean

# 安装包
jpm install spork
```

### project.janet 文件

```janet
# project.janet
(declare-project
  :name "my-project"
  :description "A Janet project"
  :author "Your Name"
  :license "MIT"
  :url "https://github.com/user/project"
  :repo "git+https://github.com/user/project.git")

# 声明依赖
(declare-source
  :source ["src"])

# 声明可执行文件
(declare-executable
  :name "my-app"
  :entry "src/main.janet")

# 声明测试
(declare-source
  :source ["test"]
  :prefix "test")
```

## 6.5 项目结构

### 标准结构

```
my-project/
├── project.janet       # 项目配置
├── README.md
├── LICENSE
├── .gitignore
├── src/
│   ├── main.janet      # 主入口
│   ├── module1.janet
│   ├── module2.janet
│   └── utils/
│       └── helper.janet
├── test/
│   ├── test-module1.janet
│   └── test-module2.janet
├── examples/
│   └── example.janet
└── build/              # 构建产物（gitignore）
```

### 示例项目

```janet
# src/main.janet
(import ./config)
(import ./handlers)

(defn main [& args]
  (print "Starting application...")
  (handlers/run (config/load)))

# src/config.janet
(defn load []
  {:host "localhost"
   :port 8080
   :debug (os/getenv "DEBUG")})

# src/handlers.janet
(import ./utils/logger)

(defn run [config]
  (logger/info "Config loaded: " config)
  # 应用逻辑
  )
```

## 6.6 常用库

### Spork - 标准扩展库

```janet
# 安装
# jpm install spork

# JSON
(import spork/json)
(json/encode {:key "value"})
(json/decode `{"key": "value"}`)

# 命令行参数解析
(import spork/argparse)
(def args (argparse/parse))

# HTTP 服务器
(import spork/http)
(http/server "0.0.0.0" 8080 handler)

# 模板引擎
(import spork/temple)
(temple/render "Hello {{ name }}" {:name "World"})
```

想系统了解 Spork 的模块地图、安装方式和最佳实践，请参见 [第十五章：Spork 扩展标准库](./15-spork-library.md)。

### 其他有用的库

```janet
# 数据库
(import sqlite3)      # SQLite
(import pq)           # PostgreSQL

# 测试
(import testament)
(import judge)

# 网络
(import circlet)      # HTTP 服务器
(import http)         # HTTP 客户端

# 实用工具
(import sh)           # Shell 命令
(import path)         # 路径操作
```

## 6.7 构建和发布

### 构建可执行文件

```janet
# project.janet
(declare-executable
  :name "myapp"
  :entry "src/main.janet"
  :install true)
```

```bash
# 构建
jpm build

# 安装到系统
jpm install

# 使用
myapp --help
```

### 发布到 Janet Package Manager

```bash
# 1. 创建 GitHub 仓库
# 2. 添加 project.janet
# 3. 打标签
git tag v1.0.0
git push origin v1.0.0

# 4. 提交到 Janet 包索引
# https://github.com/janet-lang/pkgs
```

## 6.8 开发工作流

### REPL 驱动开发

```janet
# 在 REPL 中开发
$ janet

# 加载模块
janet> (import ./src/mymodule)

# 测试函数
janet> (mymodule/some-function)

# 修改代码后重新加载
janet> (import ./src/mymodule :fresh true)

# 继续开发...
```

### 热重载

```janet
# dev.janet - 开发辅助脚本
(var last-modified @{})

(defn check-reload [path]
  (def stat (os/stat path))
  (def mtime (stat :mtime))
  (def old-mtime (get last-modified path))
  
  (when (or (nil? old-mtime) (> mtime old-mtime))
    (put last-modified path mtime)
    (import path :fresh true)
    (print "Reloaded: " path)
    true))

(defn watch-and-reload [paths]
  (while true
    (each path paths
      (check-reload path))
    (ev/sleep 1)))

# 使用
(watch-and-reload ["./src/main.janet" "./src/utils.janet"])
```

## 6.9 文档

### 文档注释

```janet
# 使用 doc 字符串
(defn add
  "Add two numbers together.
  
  Arguments:
    x - First number
    y - Second number
  
  Returns:
    The sum of x and y"
  [x y]
  (+ x y))

# 查看文档
(doc add)
```

### 生成文档

```bash
# 使用 jpm
jpm gendoc

# 或使用 mdz
jpm install mdz
mdz src/*.janet
```

## 6.10 测试

### 单元测试

```janet
# test/test-mymodule.janet
(import testament :prefix "")
(import ../src/mymodule)

(deftest "add function"
  (is (= (mymodule/add 1 2) 3))
  (is (= (mymodule/add -1 1) 0)))

(deftest "multiply function"
  (is (= (mymodule/multiply 2 3) 6))
  (is (= (mymodule/multiply 0 5) 0)))

# 运行测试
# jpm test
```

### 集成测试

```janet
# test/integration-test.janet
(import ../src/main)

(defn test-full-workflow []
  # 测试完整流程
  (def result (main/run-workflow))
  (assert (= result :success)))
```

## 6.11 最佳实践

### 项目组织原则

1. **单一职责** - 每个模块负责一个功能域
2. **清晰的依赖** - 避免循环依赖
3. **公私分明** - 使用 `-` 前缀标记私有
4. **文档完整** - 为公共 API 写文档
5. **测试覆盖** - 关键功能必须测试

### 命名约定

```janet
# 模块名：小写，短横线分隔
my-module.janet

# 文件夹：复数形式
utils/
handlers/
models/

# 测试文件：test- 前缀
test-mymodule.janet
```

### 依赖管理

```janet
# 最小化依赖
# 优先使用标准库
# 考虑嵌入小型依赖

# project.janet
(declare-project
  :dependencies [
    "spork"           # 常用工具
    "sqlite3"         # 数据库
  ])
```

## 6.12 实战：完整项目示例

### CLI 工具项目

```
janet-cli-tool/
├── project.janet
├── README.md
├── src/
│   ├── main.janet
│   ├── cli.janet
│   ├── commands/
│   │   ├── init.janet
│   │   ├── build.janet
│   │   └── deploy.janet
│   └── utils/
│       ├── logger.janet
│       └── config.janet
└── test/
    └── test-commands.janet
```

```janet
# project.janet
(declare-project
  :name "mytool"
  :description "A CLI tool"
  :dependencies ["spork"])

(declare-executable
  :name "mytool"
  :entry "src/main.janet"
  :install true)

# src/main.janet
(import ./cli)

(defn main [& args]
  (cli/run args))

# 直接执行
(main ;(slice (dyn :args) 1))
```

## 6.13 总结

本章学习了：

- ✓ 模块的创建和导入
- ✓ 模块路径和加载机制
- ✓ JPM 包管理器
- ✓ 项目结构最佳实践
- ✓ 常用库和工具
- ✓ 构建和发布流程
- ✓ 开发工作流
- ✓ 测试和文档

### 关键要点

1. **模块即文件** - 简单直接
2. **JPM 很强大** - 标准化的构建工具
3. **结构清晰** - 遵循约定
4. **测试优先** - 持续集成
5. **文档完整** - 方便维护

---

← [上一章：宏与元编程](./05-macros-metaprogramming.md) | [返回目录](./README.md) | [下一章：并发编程](./07-concurrency.md) →
