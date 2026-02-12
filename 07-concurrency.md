# 第七章：并发编程

Janet 的并发模型建立在 **纤程（Fibers）** 之上，这是一种轻量级协程机制。与其他 Lisp 方言相比，Janet 的并发更接近 Lua 的协程或 Go 的 goroutines，而不是传统的线程模型。

## 7.1 Fibers 基础

### 什么是 Fiber？

Fiber 是 Janet 中的协程（coroutine），是一种可以暂停和恢复执行的函数。与线程不同，Fibers 是协作式的，不会被抢占。

```janet
# 创建一个简单的 fiber
(def my-fiber 
  (fiber/new 
    (fn []
      (print "Step 1")
      (yield 42)
      (print "Step 2")
      (yield 100)
      (print "Step 3"))))

# 恢复 fiber 执行
(resume my-fiber)  # 打印 "Step 1"，返回 42
(resume my-fiber)  # 打印 "Step 2"，返回 100
(resume my-fiber)  # 打印 "Step 3"，fiber 完成
```

### Fiber 的生命周期

```janet
# Fiber 状态
(fiber/status my-fiber)
# 返回值：
# :new      - 刚创建，未运行
# :alive    - 运行中或已暂停
# :dead     - 已完成或出错
# :pending  - 等待 I/O

# 检查 fiber 是否可以恢复
(fiber/can-resume? my-fiber)
```

### 与 Scheme call/cc 的对比

熟悉 Scheme 的开发者会注意到 fibers 提供了类似 `call/cc` 的能力，但更加受限和实用：

```janet
# Scheme call/cc 风格（概念性）
# (call/cc (lambda (k) ... (k value) ...))

# Janet fiber 风格
(def fiber
  (fiber/new
    (fn []
      (def result (yield :need-input))
      (+ result 10))))

(resume fiber)           # 返回 :need-input
(resume fiber 20)        # 传入 20，返回 30
```

**主要区别：**

1. **控制流更简单** - 没有 call/cc 的逃逸延续（escape continuations）
2. **单向传递** - yield 返回到调用者，不能像 call/cc 那样随意跳转
3. **更易理解** - 明确的暂停/恢复语义

## 7.2 Fiber 的通信模式

### 值传递

```janet
# 通过 yield 和 resume 传递值
(def echo-fiber
  (fiber/new
    (fn []
      (forever
        (def msg (yield :waiting))
        (print "Received: " msg)))))

(resume echo-fiber)           # 返回 :waiting
(resume echo-fiber "Hello")   # 打印 "Received: Hello"
(resume echo-fiber "World")   # 打印 "Received: World"
```

### 生成器模式

```janet
# 实现一个简单的生成器
(defn range-gen [n]
  (fiber/new
    (fn []
      (for i 0 n
        (yield i)))))

(def gen (range-gen 5))
(resume gen)  # 0
(resume gen)  # 1
(resume gen)  # 2
# ...

# 使用 each 消费 fiber
(each x (range-gen 5)
  (print x))
```

### 与 Go Goroutines 的对比

```janet
# Go 风格 (概念性)
# go func() {
#     for msg := range ch {
#         process(msg)
#     }
# }()

# Janet fiber 风格
(defn worker []
  (fiber/new
    (fn []
      (forever
        (def msg (yield))
        (when msg
          (process msg))))))

(def w (worker))
(resume w)
(resume w {:task "data"})
```

**主要区别：**

1. **调度** - Go 有运行时调度器，Janet fibers 需要手动调度
2. **并行性** - Go goroutines 可以真正并行，Janet fibers 是协作式的
3. **通信** - Go 用 channels，Janet 用 yield/resume 或事件循环

## 7.3 错误处理

### Fiber 中的异常

```janet
(def faulty-fiber
  (fiber/new
    (fn []
      (yield 1)
      (error "Something went wrong")
      (yield 2))))

# 使用 resume 捕获错误
(resume faulty-fiber)        # 正常返回 1
(try
  (resume faulty-fiber)      # 抛出错误
  ([err] (print "Error: " err)))

# fiber 状态变为 :dead
(fiber/status faulty-fiber)  # :dead
```

### 安全恢复

```janet
# 使用 fiber/status 检查
(defn safe-resume [f &opt val]
  (if (= :alive (fiber/status f))
    (resume f val)
    nil))
```

## 7.4 事件循环与异步 I/O

### ev 模块简介

Janet 的 `ev` 模块提供了事件驱动的异步 I/O，类似于 Node.js 的事件循环。

```janet
# 创建一个异步任务
(ev/spawn
  (fn []
    (print "Task started")
    (ev/sleep 1)
    (print "Task completed after 1 second")))

# 启动事件循环（通常在 main 中）
# (ev/go (fn [] ...)) 会自动运行事件循环
```

### 异步 Sleep

```janet
# 同步 sleep（阻塞）
(os/sleep 1)

# 异步 sleep（让出控制权）
(defn async-countdown []
  (ev/spawn
    (fn []
      (for i 5 0 -1
        (print i)
        (ev/sleep 1))
      (print "Go!"))))

(async-countdown)
```

### 并发任务

```janet
# 同时运行多个异步任务
(ev/go
  (fn []
    (def results @[])
    
    # 启动多个并发任务
    (def fibers
      (map (fn [n]
             (ev/spawn
               (fn []
                 (ev/sleep (/ n 10))
                 (print "Task " n " done")
                 n)))
           (range 1 6)))
    
    # 等待所有任务完成
    (each f fibers
      (array/push results (ev/take f)))
    
    (pp results)))
```

## 7.5 Channel 模式

虽然 Janet 没有内置 Go 风格的 channels，但可以通过 `ev/chan` 实现类似的功能：

```janet
# 创建一个 channel
(def ch (ev/chan))

# 生产者 fiber
(ev/spawn
  (fn []
    (for i 0 5
      (ev/give ch i)
      (ev/sleep 0.5))
    (ev/chan-close ch)))

# 消费者 fiber
(ev/spawn
  (fn []
    (forever
      (def val (ev/take ch))
      (if (nil? val)
        (break)
        (print "Received: " val)))))
```

### Channel 缓冲

```janet
# 带缓冲的 channel
(def buffered-ch (ev/chan 10))

# 快速发送多个值
(ev/spawn
  (fn []
    (for i 0 10
      (ev/give buffered-ch i))
    (print "All values sent")))

# 慢速消费
(ev/spawn
  (fn []
    (repeat 10
      (ev/sleep 0.5)
      (print "Got: " (ev/take buffered-ch)))))
```

## 7.6 实战：Web 服务器

使用异步 I/O 构建简单的并发 web 服务器：

```janet
(import spork/netrepl)

(defn handle-connection [stream]
  (ev/spawn
    (fn []
      (defer (:close stream)
        (try
          # 读取请求
          (def request (ev/read stream 1024))
          
          # 简单处理
          (def response 
            (string "HTTP/1.1 200 OK\r\n"
                    "Content-Type: text/plain\r\n"
                    "\r\n"
                    "Hello from Janet!\n"))
          
          # 发送响应
          (ev/write stream response)
          
          ([err] 
           (eprint "Error handling request: " err)))))))

(defn start-server [port]
  (def server (net/listen "127.0.0.1" port))
  (print "Server listening on port " port)
  
  (forever
    (def conn (net/accept server))
    (handle-connection conn)))

# 启动服务器（在事件循环中）
# (ev/go (fn [] (start-server 8080)))
```

## 7.7 常见模式

### 超时处理

```janet
(defn with-timeout [f timeout]
  (def result (ev/chan 1))
  
  # 运行任务
  (ev/spawn
    (fn []
      (try
        (ev/give result [:ok (f)])
        ([err]
         (ev/give result [:error err])))))
  
  # 超时计时器
  (ev/spawn
    (fn []
      (ev/sleep timeout)
      (ev/give result [:timeout nil])))
  
  # 等待第一个结果
  (ev/take result))

# 使用示例
(ev/go
  (fn []
    (def [status value] 
      (with-timeout 
        (fn [] 
          (ev/sleep 2)
          "completed")
        1))
    (if (= status :timeout)
      (print "Task timed out!")
      (print "Result: " value))))
```

### Fan-out/Fan-in

```janet
# Fan-out: 将工作分发给多个 worker
(defn fan-out [jobs num-workers]
  (def results (ev/chan (* 2 num-workers)))
  (def job-chan (ev/chan num-workers))
  
  # 启动 workers
  (repeat num-workers
    (ev/spawn
      (fn []
        (forever
          (def job (ev/take job-chan))
          (if (nil? job)
            (break)
            (ev/give results (process-job job)))))))
  
  # 分发任务
  (ev/spawn
    (fn []
      (each job jobs
        (ev/give job-chan job))
      # 发送结束信号
      (repeat num-workers
        (ev/give job-chan nil))))
  
  results)

# Fan-in: 收集结果
(defn collect-results [ch n]
  (def results @[])
  (repeat n
    (array/push results (ev/take ch)))
  results)
```

### Worker Pool

```janet
(defn worker-pool [num-workers work-fn]
  (def jobs (ev/chan 100))
  (def results (ev/chan 100))
  
  # 启动 worker fibers
  (def workers
    (map (fn [i]
           (ev/spawn
             (fn []
               (forever
                 (def job (ev/take jobs))
                 (if (nil? job)
                   (break)
                   (ev/give results (work-fn job)))))))
         (range num-workers)))
  
  {:jobs jobs
   :results results
   :workers workers})

# 使用示例
(def pool (worker-pool 4 (fn [x] (* x x))))

# 提交任务
(ev/spawn
  (fn []
    (for i 1 11
      (ev/give (pool :jobs) i))
    (ev/give (pool :jobs) nil)))

# 收集结果
(ev/spawn
  (fn []
    (repeat 10
      (print "Result: " (ev/take (pool :results))))))
```

## 7.8 常见陷阱

### 陷阱 1：忘记事件循环

```janet
# ❌ 错误：这不会执行
(ev/spawn (fn [] (print "Hello")))
# 需要事件循环运行

# ✅ 正确：使用 ev/go
(ev/go
  (fn []
    (ev/spawn (fn [] (print "Hello")))))
```

### 陷阱 2：死锁

```janet
# ❌ 错误：死锁
(def ch (ev/chan))
(ev/take ch)  # 永远阻塞

# ✅ 正确：在不同的 fiber 中
(def ch (ev/chan))
(ev/spawn (fn [] (ev/give ch 42)))
(ev/take ch)  # 返回 42
```

### 陷阱 3：未关闭的 Channel

```janet
# ❌ 错误：生产者退出，但消费者还在等待
(def ch (ev/chan))
(ev/spawn
  (fn []
    (for i 0 3
      (ev/give ch i))))
      # 忘记关闭！

(ev/spawn
  (fn []
    (forever
      (def val (ev/take ch))
      (if (nil? val) (break))  # 永远不会触发
      (print val))))

# ✅ 正确：关闭 channel
(def ch (ev/chan))
(ev/spawn
  (fn []
    (for i 0 3
      (ev/give ch i))
    (ev/chan-close ch)))  # 关闭以通知消费者
```

### 陷阱 4：混用同步和异步操作

```janet
# ❌ 错误：在异步上下文中使用同步阻塞
(ev/go
  (fn []
    (ev/spawn (fn [] (os/sleep 1)))  # 阻塞整个事件循环！
    (ev/spawn (fn [] (print "I'm blocked")))))

# ✅ 正确：使用异步版本
(ev/go
  (fn []
    (ev/spawn (fn [] (ev/sleep 1)))  # 非阻塞
    (ev/spawn (fn [] (print "I run concurrently")))))
```

## 7.9 与其他 Lisp 方言的对比

### Common Lisp

```lisp
; Common Lisp - bordeaux-threads
(bt:make-thread 
  (lambda () (format t "Hello~%")))

; 需要锁和条件变量来同步
(defvar *lock* (bt:make-lock))
```

```janet
# Janet - 更高级的抽象
(ev/spawn (fn [] (print "Hello")))

# 通过 channels 自动同步
(def ch (ev/chan))
```

### Clojure

```clojure
; Clojure - core.async
(go
  (<! (timeout 1000))
  (println "Done"))
```

```janet
# Janet - 类似但更简单
(ev/go
  (fn []
    (ev/sleep 1)
    (print "Done")))
```

### Scheme (Guile)

```scheme
; Scheme - 使用 call/cc
(call/cc
  (lambda (k)
    (k 42)))
```

```janet
# Janet - 使用 fiber
(def f (fiber/new (fn [] (yield 42))))
(resume f)
```

## 7.10 性能考虑

### Fiber 开销

```janet
# Fibers 很轻量
(time
  (do
    (def fibers (seq [i :range [0 10000]]
                  (fiber/new (fn [] i))))
    (map resume fibers)))
# 创建和恢复 10000 个 fiber 通常只需几毫秒
```

### 事件循环性能

```janet
# 异步 I/O 适合高并发
# 数千个并发连接通常没问题
(defn benchmark-concurrency []
  (ev/go
    (fn []
      (def start (os/clock))
      
      # 10000 个并发任务
      (def tasks
        (map (fn [i]
               (ev/spawn
                 (fn []
                   (ev/sleep 0.01)
                   i)))
             (range 10000)))
      
      # 等待所有完成
      (map ev/take tasks)
      
      (print "Time: " (- (os/clock) start) " seconds"))))
```

## 7.11 实践练习

### 练习 1：生产者-消费者

实现一个生产者-消费者模型，使用 channel 在它们之间传递数据。

```janet
# 你的代码
(defn producer [ch num-items]
  # 生成 num-items 个项目并发送到 ch
  )

(defn consumer [ch name]
  # 从 ch 接收并处理项目
  )

# 测试：2 个生产者，3 个消费者
```

### 练习 2：并行文件处理

读取多个文件并并发处理它们的内容。

```janet
(defn process-files [filenames]
  # 为每个文件启动一个 fiber
  # 并发读取和处理
  # 返回所有结果
  )
```

### 练习 3：速率限制器

实现一个速率限制器，限制每秒的操作次数。

```janet
(defn rate-limiter [max-per-second]
  # 返回一个函数，该函数在被调用时
  # 遵守速率限制
  )

# 使用
(def limiter (rate-limiter 5))
(for i 0 20
  (limiter)
  (print "Request " i))
```

## 小结

本章我们学习了：

- ✓ Fibers（协程）的基本概念和使用
- ✓ 与 Scheme call/cc 和 Go goroutines 的对比
- ✓ 事件循环和异步 I/O
- ✓ Channel 和并发通信模式
- ✓ 实用的并发编程模式
- ✓ 常见陷阱和性能考虑

### 关键要点

1. **Fibers 是协作式的** - 不会被抢占，需要显式 yield
2. **ev 模块提供事件循环** - 类似 Node.js 的异步模型
3. **Channels 用于通信** - 比直接 yield/resume 更结构化
4. **避免阻塞操作** - 在异步上下文中使用 ev/sleep 而非 os/sleep
5. **轻量且高效** - 可以轻松创建数千个 fibers

---

← [上一章：函数式编程](./06-functional-programming.md) | [返回目录](./README.md) | [下一章：FFI 与 C 互操作](./08-ffi-c-interop.md) →
