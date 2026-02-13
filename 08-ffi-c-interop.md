# 第八章：FFI 与 C 互操作

FFI (Foreign Function Interface) 是 Janet 进行系统级编程的核心能力。通过 FFI，你可以直接调用 C 库，无需编写绑定代码。

## 8.1 FFI 基础

### 为什么需要 FFI？

```janet
# 场景1：调用系统 API
# 场景2：使用现有 C 库
# 场景3：性能关键代码用 C 实现
# 场景4：访问硬件或底层资源
```

### FFI 可用性

```janet
# 检查 FFI 是否可用
(if (dyn :ffi/native)
  (print "FFI is available")
  (print "FFI not available on this platform"))

# FFI 要求：
# - x86-64 架构
# - 非 Windows 系统（或特定配置的 Windows）
# - Janet 1.23.0+
```

## 8.2 基本 FFI 操作

### 加载动态库

```janet
# 加载系统库
(ffi/native "/usr/lib/x86_64-linux-gnu/libm.so.6")

# 或使用 ffi/context 
(def ctx (ffi/context))

# 设置当前库上下文
(ffi/native "/usr/lib/libexample.so")

# 查找符号
(def sin-ptr (ffi/lookup "sin"))
```

### 定义 C 函数签名

```janet
# 基本类型映射
# C 类型          Janet FFI 类型
# void            :void
# int             :int
# double          :double
# float           :float
# char *          :string
# void *          :ptr
# size_t          :size
# int8_t          :s8
# uint8_t         :u8
# int16_t         :s16
# uint16_t        :u16
# int32_t         :s32
# uint32_t        :u32
# int64_t         :s64
# uint64_t        :u64

# 定义函数签名
(def sin-sig (ffi/signature :double [:double]))
```

### 调用 C 函数

```janet
# 完整示例：调用 libm 的 sin 函数
(ffi/native "/usr/lib/x86_64-linux-gnu/libm.so.6")

(def sin-ptr (ffi/lookup "sin"))
(def sin-sig (ffi/signature :double [:double]))

# 创建可调用函数
(def sin-c (ffi/bind sin-sig sin-ptr))

# 调用
(sin-c 0)      # => 0
(sin-c 1.5708) # => ~1.0 (π/2)
```

### ffi/defbind 简化版

```janet
# 使用 defbind 宏简化定义
(ffi/native "/usr/lib/x86_64-linux-gnu/libm.so.6")

(ffi/defbind sin :double [x :double])
(ffi/defbind cos :double [x :double])
(ffi/defbind sqrt :double [x :double])

# 直接使用
(sin 0)      # => 0
(cos 0)      # => 1
(sqrt 16)    # => 4
```

## 8.3 结构体与指针

### 定义 C 结构体

```janet
# C 代码：
# struct Point {
#     double x;
#     double y;
# };

# Janet FFI 定义
(def point-type
  (ffi/struct
    :x :double
    :y :double))

# 创建结构体实例
(def p (ffi/struct-new point-type))

# 访问字段
(ffi/struct-get p :x)  # => 0.0
(ffi/struct-set p :x 3.0)
(ffi/struct-set p :y 4.0)

# 读取
(ffi/struct-get p :x)  # => 3.0
```

### 指针操作

```janet
# 获取结构体指针
(def p-ptr (ffi/pointer p))

# 指针转整数地址
(def addr (ffi/ptr-to-int p-ptr))

# 整数地址转指针
(def p-ptr2 (ffi/int-to-ptr addr))

# 指针偏移
(def offset-ptr (ffi/ptr-offset p-ptr 8))  # 偏移 8 字节

# 读写指针指向的内存
(ffi/write-double p-ptr 0 3.14)
(ffi/read-double p-ptr 0)  # => 3.14
```

### 数组与内存

```janet
# 分配内存
(def buffer (ffi/malloc 1024))  # 分配 1KB

# 写入数据
(ffi/write-int buffer 0 42)
(ffi/write-int buffer 4 43)
(ffi/write-int buffer 8 44)

# 读取数据
(ffi/read-int buffer 0)  # => 42
(ffi/read-int buffer 4)  # => 43

# 释放内存（重要！）
(ffi/free buffer)

# 使用 defer 确保释放
(defn with-buffer [size f]
  (def buf (ffi/malloc size))
  (defer (ffi/free buf)
    (f buf)))
```

## 8.4 实战：调用系统 API

### 示例 1：调用 POSIX API

```janet
# 调用 getpid() 获取进程 ID
(ffi/native)  # 加载当前进程符号

(ffi/defbind getpid :int [])
(ffi/defbind getuid :int [])
(ffi/defbind getgid :int [])

(print "PID: " (getpid))
(print "UID: " (getuid))
(print "GID: " (getgid))
```

### 示例 2：文件操作

```janet
# 使用 POSIX open/read/write/close
(ffi/native)

# open 函数
(ffi/defbind c-open :int [path :string flags :int mode :int])
(ffi/defbind c-read :int [fd :int buf :ptr count :size])
(ffi/defbind c-write :int [fd :int buf :ptr count :size])
(ffi/defbind c-close :int [fd :int])

# O_RDONLY = 0, O_WRONLY = 1, O_RDWR = 2
(def O_RDONLY 0)
(def O_WRONLY 1)
(def O_CREAT 64)

# 读取文件
(defn read-file-ffi [path]
  (def fd (c-open path O_RDONLY 0))
  (when (< fd 0)
    (error "Failed to open file"))
  
  (defer (c-close fd)
    (def buf (ffi/malloc 4096))
    (defer (ffi/free buf)
      (def n (c-read fd buf 4096))
      (if (< n 0)
        (error "Read failed")
        (string/from-bytes (ffi/read-bytes buf 0 n))))))
```

### 示例 3：网络编程

```janet
# Socket API
(ffi/native)

(ffi/defbind c-socket :int [domain :int type :int protocol :int])
(ffi/defbind c-connect :int [sockfd :int addr :ptr addrlen :size])
(ffi/defbind c-send :int [sockfd :int buf :ptr len :size flags :int])
(ffi/defbind c-recv :int [sockfd :int buf :ptr len :size flags :int])
(ffi/defbind c-close :int [fd :int])

# AF_INET = 2, SOCK_STREAM = 1
(def AF_INET 2)
(def SOCK_STREAM 1)

# 创建 TCP socket
(def sock (c-socket AF_INET SOCK_STREAM 0))
```

## 8.5 与 C 代码集成

### 方式 1：编写 Janet C 模块

```c
/* mymodule.c */
#include <janet.h>

static Janet my_add(int32_t argc, Janet *argv) {
    janet_fixarity(argc, 2);
    double x = janet_getnumber(argv, 0);
    double y = janet_getnumber(argv, 1);
    return janet_wrap_number(x + y);
}

static const JanetReg cfuns[] = {
    {"add", my_add, "(add x y)\n\nAdd two numbers."},
    {NULL, NULL, NULL}
};

JANET_MODULE_ENTRY(JanetTable *env) {
    janet_cfuns(env, "mymodule", cfuns);
}
```

编译模块：

```bash
# 编译为共享库
gcc -shared -fPIC -I/path/to/janet \
    -o mymodule.so mymodule.c

# 或使用 JPM
jpm build
```

在 Janet 中使用：

```janet
(import ./mymodule)
(mymodule/add 1 2)  # => 3
```

### 方式 2：嵌入 Janet 到 C 程序

```c
/* embed.c */
#include <janet.h>
#include <stdio.h>

int main() {
    // 初始化 Janet
    janet_init();
    
    // 创建环境
    JanetTable *env = janet_core_env(NULL);
    
    // 执行 Janet 代码
    const char *code = "(+ 1 2 3)";
    Janet result;
    JanetSignal status = janet_dobytes(
        env, 
        (const uint8_t *)code, 
        strlen(code), 
        "main", 
        &result
    );
    
    if (status == JANET_SIGNAL_OK) {
        printf("Result: %f\n", janet_unwrap_number(result));
    } else {
        printf("Error!\n");
    }
    
    // 清理
    janet_deinit();
    return 0;
}
```

编译：

```bash
gcc embed.c -I/path/to/janet -ljanet -o embed
./embed
# 输出: Result: 6.000000
```

### 方式 3：以 raylib 为例（与 C/C++ 项目互动）

raylib 是 C 写的图形库，也常在 C++ 项目中使用。实践里建议先写一层很薄的 C 包装，再给 Janet 调用，这样可以同时规避 C++ 名字改编问题并简化类型映射。

```c
/* raylib_bridge.c */
#include "raylib.h"

#ifdef __cplusplus
extern "C" {
#endif

/* 以下示例使用 C99 复合字面量 */
void rl_init_window(int w, int h, const char *title) { InitWindow(w, h, title); }
void rl_begin_drawing(void) { BeginDrawing(); }
void rl_end_drawing(void) { EndDrawing(); }
void rl_clear_background(int r, int g, int b, int a) { ClearBackground((Color) {r, g, b, a}); }
/* text, x, y, size, r, g, b, a */
void rl_draw_text(const char *text, int x, int y, int size, int r, int g, int b, int a) {
    DrawText(text, x, y, size, (Color) {r, g, b, a});
}
int rl_window_should_close(void) { return WindowShouldClose() ? 1 : 0; }
void rl_close_window(void) { CloseWindow(); }

#ifdef __cplusplus
}
#endif
```

```bash
# Linux 示例（C）
# 如果 raylib 不在系统默认路径，追加 -I/path/to/raylib/include -L/path/to/raylib/lib
gcc -shared -fPIC raylib_bridge.c -lraylib -o libraylib_bridge.so

# Linux 示例（C++ 项目中编译同一桥接文件）
g++ -shared -fPIC raylib_bridge.c -lraylib -o libraylib_bridge.so
```

```janet
(ffi/native "./libraylib_bridge.so")

(ffi/defbind rl_init_window :void [w :int h :int title :string])
(ffi/defbind rl_begin_drawing :void [])
(ffi/defbind rl_end_drawing :void [])
(ffi/defbind rl_clear_background :void [r :int g :int b :int a :int])
(ffi/defbind rl_draw_text :void [text :string x :int y :int size :int r :int g :int b :int a :int])
(ffi/defbind rl_window_should_close :int [])
(ffi/defbind rl_close_window :void [])

(rl_init_window 800 450 "Janet + raylib")
(while (= 0 (rl_window_should_close))
  (rl_begin_drawing)
  (rl_clear_background 30 30 30 255)
  (rl_draw_text "Hello from Janet FFI" 190 200 24 245 245 245 255)
  (rl_end_drawing))
(rl_close_window)
```

这个模式同样适用于 SDL、OpenGL 包装层或你自己的 C++ 引擎导出接口（`extern "C"`）。

## 8.6 C API 核心函数

### 类型转换

```c
// Janet -> C
double janet_getnumber(const Janet *argv, int32_t n);
int32_t janet_getinteger(const Janet *argv, int32_t n);
const char *janet_getcstring(const Janet *argv, int32_t n);
void *janet_getpointer(const Janet *argv, int32_t n);

// C -> Janet
Janet janet_wrap_number(double x);
Janet janet_wrap_integer(int32_t x);
Janet janet_wrap_string(const uint8_t *str);
Janet janet_wrap_pointer(void *ptr);
Janet janet_wrap_nil(void);
Janet janet_wrap_true(void);
Janet janet_wrap_false(void);
```

### 集合操作

```c
// 数组
JanetArray *janet_array(int32_t capacity);
void janet_array_push(JanetArray *array, Janet x);
Janet janet_array_pop(JanetArray *array);

// 表
JanetTable *janet_table(int32_t capacity);
Janet janet_table_get(JanetTable *table, Janet key);
void janet_table_put(JanetTable *table, Janet key, Janet value);

// 结构体
const JanetKV *janet_struct_begin(int32_t count);
const JanetKV *janet_struct_end(JanetKV *st);
```

### 错误处理

```c
// 抛出错误
void janet_panic(const char *message);
void janet_panicf(const char *format, ...);

// 参数检查
void janet_fixarity(int32_t argc, int32_t expected);
void janet_arity(int32_t argc, int32_t min, int32_t max);
```

## 8.7 性能考虑

### FFI 性能

```janet
# FFI 调用有开销（比原生 C 慢）
# 适合：
# - 调用频率低但功能重要的函数
# - 大批量数据处理（调用次数少）

# 不适合：
# - 高频调用的简单函数
# - 内循环中的操作

# 示例：错误的用法
(for i 0 1000000
  (sin-c i))  # 慢！每次都有 FFI 开销

# 正确的用法：批量操作
(def data (array/new 1000000))
(for i 0 1000000
  (put data i i))
# 然后用 C 函数批量处理
```

### 优化技巧

```janet
# 1. 批量传递数据
(defn batch-sin [values]
  # 在 C 中循环处理整个数组
  # 而不是每个值都调用一次
  )

# 2. 缓存函数指针
(def sin-cached (ffi/defbind sin :double [:double]))

# 3. 使用 Janet 内置函数优先
(math/sin x)  # 优于 FFI 调用
```

## 8.8 与其他 Lisp 的 FFI 对比

### Common Lisp CFFI

```lisp
; Common Lisp
(cffi:defcfun "sin" :double
  (x :double))

(sin 1.0)
```

```janet
# Janet
(ffi/defbind sin :double [x :double])
(sin 1.0)
```

相似度很高！

### Scheme FFI

```scheme
; Chicken Scheme
(foreign-declare "#include <math.h>")
(define sin (foreign-lambda double "sin" double))
```

```janet
# Janet
(ffi/native)
(ffi/defbind sin :double [x :double])
```

### Clojure JNI/JNA

```clojure
; Clojure (使用 JNA)
(require '[net.n01se.clojure-jna :as jna])
(def sin (jna/to-fn Double [Double] "sin"))
```

Janet 的 FFI 更直接，无需 JVM 层。

## 8.9 安全性注意事项

### 内存安全

```janet
# 危险操作示例
(def ptr (ffi/malloc 100))
# 忘记 free -> 内存泄漏

(ffi/free ptr)
# 使用已释放的指针 -> 段错误

# 安全实践
(defn with-memory [size f]
  (def ptr (ffi/malloc size))
  (defer (ffi/free ptr)
    (f ptr)))  # 自动释放

# 使用
(with-memory 1024
  (fn [buf]
    # 使用 buf
    ))  # 自动清理
```

### 类型安全

```janet
# FFI 不检查类型！
(ffi/defbind dangerous :void [p :ptr])

(dangerous "string")  # 可能崩溃！
(dangerous 12345)     # 可能崩溃！

# 添加包装检查
(defn safe-dangerous [p]
  (assert (= (type p) :pointer) "Expected pointer")
  (dangerous p))
```

## 8.10 实战项目：系统信息工具

创建一个获取系统信息的工具：

```janet
# sysinfo.janet
(ffi/native)

# 定义结构体
(def utsname-type
  (ffi/struct
    :sysname [:char 65]
    :nodename [:char 65]
    :release [:char 65]
    :version [:char 65]
    :machine [:char 65]))

# 绑定 uname 函数
(ffi/defbind c-uname :int [buf :ptr])

# 获取系统信息
(defn get-system-info []
  (def buf (ffi/struct-new utsname-type))
  (defer (ffi/free buf)
    (when (= (c-uname buf) 0)
      {:system (ffi/read-string buf 0)
       :node (ffi/read-string buf 65)
       :release (ffi/read-string buf 130)
       :version (ffi/read-string buf 195)
       :machine (ffi/read-string buf 260)})))

# 使用
(pp (get-system-info))
```

## 8.11 练习

### 练习 1：包装 C 库

包装一个简单的 C 库（如 zlib）：

```janet
(defmodule zlib
  # TODO: 绑定 compress 和 decompress 函数
  )
```

### 练习 2：系统调用

实现文件监控（使用 inotify 或类似 API）：

```janet
(defn watch-file [path callback]
  # TODO: 使用 FFI 实现文件监控
  )
```

### 练习 3：性能对比

对比 FFI sin 和 Janet math/sin 的性能：

```janet
(defn benchmark [f n]
  # TODO: 测试调用 f n 次的时间
  )

(benchmark math/sin 100000)
(benchmark sin-c 100000)
```

## 8.12 总结

本章学习了：

- ✓ FFI 基础和类型系统
- ✓ 调用 C 函数和库
- ✓ 结构体和指针操作
- ✓ 编写 Janet C 模块
- ✓ 嵌入 Janet 到 C 程序
- ✓ 性能和安全考虑

### 关键要点

1. **直接调用 C** - 无需编写绑定代码
2. **类型系统简单** - 基本 C 类型映射清晰
3. **手动内存管理** - 需要注意内存安全
4. **性能权衡** - FFI 有开销，但访问无限可能
5. **实用主义** - 为系统编程而设计

---

← [上一章：并发编程](./07-concurrency.md) | [返回目录](./README.md) | [下一章：网络编程](./09-networking.md) →
