# 第三章：数据结构与类型系统

Janet 提供了丰富的数据结构，与 Clojure 非常相似。理解可变和不可变数据结构的区别是掌握 Janet 的关键。

## 3.1 类型概览

### 基本类型层次

```janet
# 标量类型（Scalar Types）
nil             # 空值
true false      # 布尔值
42 3.14 0xFF    # 数字（统一的数字类型）
"string"        # 不可变字符串
@"buffer"       # 可变字符缓冲区
:keyword        # 关键字
'symbol         # 符号

# 集合类型（Collection Types）
(1 2 3)         # 元组 (Tuple) - 不可变列表
@(1 2 3)        # 括号元组 (Paren Tuple)
[1 2 3]         # 元组 (Tuple) - 不可变数组
@[1 2 3]        # 数组 (Array) - 可变
{:a 1 :b 2}     # 结构体 (Struct) - 不可变映射
@{:a 1 :b 2}    # 表 (Table) - 可变映射

# 函数和抽象类型
(fn [x] x)      # 函数 (Function)
(coro ...)      # 纤程 (Fiber)
(peg/compile ...)  # PEG (Parser Expression Grammar)
```

### 类型检查

```janet
# 类型检查函数
(type 42)           # => :number
(type "hello")      # => :string
(type :keyword)     # => :keyword
(type [1 2 3])      # => :tuple
(type @[1 2 3])     # => :array

# 类型谓词
(number? 42)        # => true
(string? "hi")      # => true
(keyword? :key)     # => true
(tuple? [1 2])      # => true
(array? @[1 2])     # => true
(table? @{:a 1})    # => true
(struct? {:a 1})    # => true
(function? +)       # => true
(fiber? (fiber/new (fn [] nil)))  # => true

# 集合检查
(indexed? [1 2])    # => true (元组或数组)
(dictionary? {:a 1})  # => true (结构体或表)
(nil? nil)          # => true
(empty? [])         # => true
```

## 3.2 数字（Numbers）

### 数字类型

Janet 只有一种数字类型：双精度浮点数（IEEE 754）。但它可以精确表示 53 位整数。

```janet
# 整数表示
42
-17
0xFF        # 十六进制
0o777       # 八进制  
0b1010      # 二进制

# 浮点数
3.14
1.5e10
2.5e-3

# 数学运算
(+ 1 2 3)       # => 6
(- 10 3)        # => 7
(* 2 3 4)       # => 24
(/ 10 3)        # => 3.333...
(% 10 3)        # => 1 (取模)
(mod 10 3)      # => 1

# 比较
(< 1 2 3)       # => true
(> 3 2 1)       # => true
(= 1 1.0)       # => true
(!= 1 2)        # => true

# 数学函数
(math/abs -5)       # => 5
(math/floor 3.7)    # => 3
(math/ceil 3.2)     # => 4
(math/round 3.5)    # => 4
(math/sqrt 16)      # => 4
(math/pow 2 8)      # => 256
(math/sin (/ math/pi 2))  # => 1
```

### 数字的精度问题

```janet
# 整数精度（53 位）
(= (+ 9007199254740992 1) 9007199254740992)  # => true!
# 超出精度范围

# 浮点数精度
(= 0.1 (/ 1 10))    # => true
(= 0.3 (+ 0.1 0.2))  # => false! (浮点误差)

# 解决方法：使用 epsilon 比较
(defn approx= [a b &opt epsilon]
  (default epsilon 1e-10)
  (< (math/abs (- a b)) epsilon))

(approx= 0.3 (+ 0.1 0.2))  # => true
```

### 与其他 Lisp 的数字系统对比

| Lisp | 整数 | 浮点数 | 有理数 | 复数 | 任意精度 |
|------|------|--------|--------|------|----------|
| Common Lisp | ✓ | ✓ | ✓ | ✓ | ✓ (Bignum) |
| Scheme | ✓ | ✓ | ✓ | ✓ | 依实现 |
| Clojure | ✓ | ✓ | ✓ | 仅 JVM | ✓ (Bignum) |
| Janet | ✓ (53位) | ✓ | ✗ | ✗ | ✗ |

Janet 的数字系统更简单，但对于大多数系统编程已足够。

## 3.3 字符串与缓冲区

### 不可变字符串

```janet
# 字符串字面量
"hello world"
"multi\nline\nstring"
"unicode: 你好"

# 长字符串（支持换行，保留格式）
``
This is a
multi-line string
with preserved formatting
``

# 字符串操作
(string "hello" " " "world")  # => "hello world"
(string/join ["a" "b" "c"] ",")  # => "a,b,c"
(string/split "," "a,b,c")   # => @["a" "b" "c"]
(string/trim "  hello  ")    # => "hello"
(string/upper "hello")       # => "HELLO"
(string/lower "HELLO")       # => "hello"

# 字符串查找
(string/find "world" "hello world")  # => 6
(string/has-prefix? "hello" "hello world")  # => true
(string/has-suffix? "world" "hello world")  # => true

# 字符串切片
(string/slice "hello" 0 3)   # => "hel"
(string/slice "hello" 2)     # => "llo"
```

### 可变缓冲区

缓冲区是可变的字节数组，用于高效的字符串构建：

```janet
# 创建缓冲区
(def buf @"")
(def buf2 @"initial")

# 添加内容
(buffer/push buf "hello")
(buffer/push buf " " "world")
buf  # => @"hello world"

# 缓冲区作为字符串构建器
(defn build-string [& parts]
  (def buf @"")
  (each part parts
    (buffer/push buf (string part)))
  (string buf))

(build-string "The answer is " 42)  # => "The answer is 42"

# 缓冲区切片
(buffer/slice @"hello world" 0 5)  # => @"hello"

# 清空缓冲区
(buffer/clear buf)
```

### 字符串格式化

```janet
# string/format (类似 printf)
(string/format "Hello, %s!" "World")  # => "Hello, World!"
(string/format "Number: %d" 42)       # => "Number: 42"
(string/format "Float: %.2f" 3.14159) # => "Float: 3.14"
(string/format "Hex: %x" 255)         # => "Hex: ff"

# 格式说明符
# %s - 字符串
# %d - 十进制整数
# %x - 十六进制
# %f - 浮点数
# %% - 字面 %

# 字符串插值（宏）
(defmacro str [& forms]
  ~(string ,;forms))

(def name "Alice")
(str "Hello, " name "!")  # => "Hello, Alice!"
```

## 3.4 符号与关键字

### 符号（Symbols）

符号是标识符，用于变量名和函数名：

```janet
# 符号字面量
'x
'my-variable
'+  # 符号可以是运算符

# 创建符号
(symbol "dynamic-name")  # => dynamic-name
(symbol 'prefix "-" 'suffix)  # => prefix-suffix

# 符号操作
(symbol? 'x)              # => true
(= 'x 'x)                 # => true
(= 'x 'y)                 # => false

# 符号转字符串
(string 'my-symbol)       # => "my-symbol"

# 关键字转符号
(symbol :keyword)         # => keyword
```

### 关键字（Keywords）

关键字是自求值的符号，常用于映射的键：

```janet
# 关键字字面量
:name
:age
:key-with-dashes

# 关键字作为函数（获取映射值）
(def person {:name "Alice" :age 30})
(:name person)            # => "Alice"
(:age person)             # => 30
(:missing person)         # => nil
(:missing person "default")  # => "default" (带默认值)

# 关键字vs符号
(keyword? :key)           # => true
(symbol? :key)            # => false
(= :key :key)             # => true
(= :key 'key)             # => false

# 转换
(keyword 'sym)            # => :sym
(symbol :key)             # => sym
```

### 与 Common Lisp/Clojure 的对比

```lisp
; Common Lisp - 关键字在 keyword 包中
:keyword  ; => :KEYWORD (大写)
(keywordp :keyword)  ; => T

; Clojure - 与 Janet 几乎相同
:keyword
(keyword? :keyword)  ; => true
(:keyword {:keyword "value"})  ; => "value"
```

```janet
# Janet - 与 Clojure 非常相似
:keyword
(keyword? :keyword)  # => true
(:keyword {:keyword "value"})  # => "value"
```

## 3.5 元组（Tuples）- 不可变序列

### 元组基础

元组是不可变的、有序的集合：

```janet
# 创建元组
[1 2 3]
(tuple 1 2 3)
'(1 2 3)  # 引用的列表也是元组

# 访问元素
(def t [1 2 3 4 5])
(t 0)              # => 1
(get t 1)          # => 2
(get t 10)         # => nil (越界返回 nil)
(get t 10 "default")  # => "default"

# 元组操作
(length [1 2 3])   # => 3
(first [1 2 3])    # => 1
(last [1 2 3])     # => 3
(empty? [])        # => true

# 切片
(slice [1 2 3 4 5] 1 3)  # => [2 3]
(slice [1 2 3 4 5] 2)    # => [3 4 5]

# 元组是不可变的
(def t [1 2 3])
# (put t 0 99)  # 错误！不能修改元组
```

### 元组的结构化绑定

```janet
# 解构元组
(let [[a b c] [1 2 3]]
  (+ a b c))  # => 6

# 部分解构
(let [[first & rest] [1 2 3 4]]
  [first rest])  # => [1 [2 3 4]]

# 嵌套解构
(let [[x [y z]] [1 [2 3]]]
  (+ x y z))  # => 6

# 忽略部分元素
(let [[a _ c] [1 2 3]]
  (+ a c))  # => 4
```

### 元组操作

```janet
# 连接（创建新元组）
(tuple/join [1 2] [3 4])  # => [1 2 3 4]
(tuple ;[1 2] ;[3 4])     # => (1 2 3 4)

# 映射
(map inc [1 2 3])  # => @[2 3 4] (返回数组！)
(seq [x :in [1 2 3]] (inc x))  # => @[2 3 4]

# 过滤
(filter even? [1 2 3 4 5])  # => @[2 4]

# 折叠
(reduce + 0 [1 2 3 4 5])  # => 15
```

## 3.6 数组（Arrays）- 可变序列

### 数组基础

数组是可变的、有序的集合：

```janet
# 创建数组
@[1 2 3]
(array 1 2 3)
(array/new 10)  # 容量为10的空数组

# 访问与修改
(def arr @[1 2 3])
(arr 0)              # => 1
(put arr 0 99)       # 修改第一个元素
arr                  # => @[99 2 3]

# 添加元素
(array/push arr 4)   # 末尾添加
arr                  # => @[99 2 3 4]

# 删除元素
(array/pop arr)      # 删除末尾，返回 4
(array/remove arr 0) # 删除索引0，返回 99
arr                  # => @[2 3]

# 插入元素
(array/insert arr 0 1)  # 在索引0插入1
arr                     # => @[1 2 3]
```

### 数组操作

```janet
# 连接
(array/concat @[1 2] @[3 4])  # => @[1 2 3 4]

# 切片（返回新数组）
(array/slice @[1 2 3 4 5] 1 3)  # => @[2 3]

# 排序（原地修改）
(def arr @[3 1 4 1 5])
(sort arr)  # arr 现在是 @[1 1 3 4 5]

# 自定义排序
(sort arr >)  # 降序

# 反转
(reverse @[1 2 3])  # => @[3 2 1]

# 去重（需要排序）
(distinct @[1 2 2 3 3 3])  # => @[1 2 3]
```

### 数组 vs 元组

```janet
# 性能对比
# 元组：
# - 创建快
# - 内存占用小
# - 适合函数返回值、临时数据

# 数组：
# - 可修改
# - 动态增长
# - 适合累积结果、缓存

# 转换
(tuple ;@[1 2 3])   # 数组 => 元组
(array ;[1 2 3])    # 元组 => 数组
```

## 3.7 结构体（Structs）- 不可变映射

### 结构体基础

结构体是不可变的键值对集合：

```janet
# 创建结构体
{:name "Alice" :age 30}
(struct :name "Alice" :age 30)

# 访问
(def person {:name "Alice" :age 30 :city "NYC"})
(person :name)         # => "Alice"
(get person :age)      # => 30
(get person :missing)  # => nil
(:name person)         # => "Alice" (关键字作为函数)

# 结构体是不可变的
# (put person :age 31)  # 错误！
```

### 结构体操作

```janet
# 合并（创建新结构体）
(merge {:a 1} {:b 2})  # => {:a 1 :b 2}
(merge {:a 1 :b 2} {:b 3})  # => {:a 1 :b 3} (后者覆盖)

# 键和值
(keys {:a 1 :b 2})    # => (:a :b)
(values {:a 1 :b 2})  # => (1 2)
(pairs {:a 1 :b 2})   # => @[(:a 1) (:b 2)]

# 检查键
(has-key? {:a 1} :a)  # => true
(has-key? {:a 1} :b)  # => false

# 更新（函数式，创建新结构体）
(defn update [dict key f]
  (merge dict {key (f (get dict key))}))

(update {:count 5} :count inc)  # => {:count 6}
```

### 结构体的解构

```janet
# 解构结构体
(let [{:name name :age age} {:name "Alice" :age 30}]
  (string name " is " age))  # => "Alice is 30"

# 简写形式（同名绑定）
(let [{:name name :age age} person]
  ...)

# 带默认值的解构
(defn greet [& {:name "Guest"}]
  (string "Hello, " name))

(greet)              # => "Hello, Guest"
(greet :name "Bob")  # => "Hello, Bob"
```

## 3.8 表（Tables）- 可变映射

### 表基础

表是可变的键值对集合：

```janet
# 创建表
@{:name "Alice" :age 30}
(table :name "Alice" :age 30)

# 访问与修改
(def person @{:name "Alice" :age 30})
(person :name)         # => "Alice"
(put person :age 31)   # 修改
(put person :city "NYC")  # 添加新键
person  # => @{:name "Alice" :age 31 :city "NYC"}

# 删除键
(put person :city nil)  # 删除 :city
```

### 表操作

```janet
# 合并（修改第一个表）
(def t1 @{:a 1})
(merge-into t1 @{:b 2})
t1  # => @{:a 1 :b 2}

# 表作为集合（值为 true）
(def set @{:a true :b true :c true})
(set :a)  # => true
(set :d)  # => nil

# 频率计数
(defn frequencies [coll]
  (def freq @{})
  (each x coll
    (put freq x (inc (get freq x 0))))
  freq)

(frequencies [1 2 2 3 3 3])  # => @{1 1 2 2 3 3}
```

### 表 vs 结构体

```janet
# 使用原则：
# - 默认使用结构体（不可变，更安全）
# - 需要修改时使用表
# - 需要动态添加/删除键时使用表

# 转换
(table ;(pairs {:a 1 :b 2}))  # 结构体 => 表
(struct ;(pairs @{:a 1 :b 2}))  # 表 => 结构体
```

## 3.9 集合通用操作

### 序列操作（适用于所有集合）

```janet
# map - 映射
(map inc [1 2 3])           # => @[2 3 4]
(map + [1 2 3] [4 5 6])     # => @[5 7 9]

# filter - 过滤
(filter even? [1 2 3 4])    # => @[2 4]
(filter |(> $ 0) [-1 0 1])  # => @[1]

# reduce - 折叠
(reduce + 0 [1 2 3 4])      # => 10
(reduce max (first arr) arr)  # 找最大值

# each - 迭代（副作用）
(each x [1 2 3]
  (print x))

# map - 索引映射
(map-indexed (fn [i x] [i x]) [:a :b :c])
# => @[[0 :a] [1 :b] [2 :c]]

# 列表推导（seq宏）
(seq [x :in [1 2 3]
      :when (even? x)]
  (* x x))
# => @[4]
```

### 高级序列操作

```janet
# take - 取前n个
(take 3 [1 2 3 4 5])        # => @[1 2 3]

# drop - 跳过前n个
(drop 2 [1 2 3 4 5])        # => @[3 4 5]

# take-while - 取直到条件不满足
(take-while |(< $ 5) [1 2 3 7 8])  # => @[1 2 3]

# partition - 分组
(partition 2 [1 2 3 4 5 6])  # => @[[1 2] [3 4] [5 6]]

# interleave - 交错
(interleave [1 2 3] [:a :b :c])  # => @[1 :a 2 :b 3 :c]

# zip - 配对（实际使用 map）
(map tuple [1 2 3] [:a :b :c])  # => @[(1 :a) (2 :b) (3 :c)]

# flatten - 扁平化
(flatten [[1 2] [3 4] [5]])  # => @[1 2 3 4 5]
```

## 3.10 惰性序列与生成器

### 生成器（Generators）

```janet
# 使用 generate 创建惰性序列
(defn naturals []
  (generate [n 0]
    (do
      (yield n)
      (set n (inc n)))))

# 使用生成器
(take 5 (naturals))  # => @[0 1 2 3 4]

# 斐波那契生成器
(defn fibonacci []
  (generate [a 0 b 1]
    (do
      (yield a)
      (def temp b)
      (set b (+ a b))
      (set a temp))))

(take 10 (fibonacci))  # => @[0 1 1 2 3 5 8 13 21 34]

# 过滤生成器
(defn evens []
  (generate [n 0]
    (do
      (yield n)
      (set n (+ n 2)))))

(take 5 (evens))  # => @[0 2 4 6 8]
```

### 纤程（Fibers）作为生成器

```janet
# 使用纤程实现生成器
(defn make-generator [f]
  (fiber/new f))

(def gen (make-generator
          (fn []
            (yield 1)
            (yield 2)
            (yield 3))))

(resume gen)  # => 1
(resume gen)  # => 2
(resume gen)  # => 3
(resume gen)  # => nil (完成)
```

## 3.11 模式匹配

### Match 表达式

```janet
# 基本模式匹配
(defn describe [x]
  (match x
    0 "zero"
    1 "one"
    2 "two"
    _ "other"))

# 类型匹配
(defn type-describe [x]
  (match (type x)
    :number "a number"
    :string "a string"
    :array "an array"
    _ "something else"))

# 解构匹配
(defn point-quadrant [[x y]]
  (match [x y]
    [0 0] "origin"
    [0 _] "on y-axis"
    [_ 0] "on x-axis"
    [x y] (if (and (> x 0) (> y 0))
            "quadrant I"
            "other quadrant")))

# 嵌套匹配
(match {:type :circle :radius 5}
  {:type :circle :radius r} (string "Circle with radius " r)
  {:type :rectangle :width w :height h} (string "Rectangle " w "x" h)
  _ "Unknown shape")
```

## 3.12 与其他 Lisp 的数据结构对比

### 总体对比

| 数据结构 | Common Lisp | Scheme | Clojure | Janet |
|----------|-------------|--------|---------|-------|
| 列表 | Cons cells | Cons cells | 持久化列表 | 元组 |
| 向量 | `#(...)` | `#(...)` | `[...]` | `[...]` / `@[...]` |
| 映射 | Hash table | Hash table | `{...}` | `{...}` / `@{...}` |
| 集合 | 需要库 | 需要库 | `#{...}` | 表（手动） |
| 可变性 | 默认可变 | 默认可变 | 默认不可变 | 明确区分 |
| 持久化 | 否 | 否 | 是 | 否 |

### Janet 的设计哲学

1. **明确的可变性** - `@` 前缀表示可变，清晰明了
2. **简单而非持久化** - 不像 Clojure 的持久化数据结构，简单复制
3. **性能优先** - 数组和表的性能优于不可变版本
4. **实用主义** - 提供最常用的数据结构，避免过度设计

## 3.13 实践练习

### 练习 1：数据转换

实现一个函数，将嵌套的结构体转换为表：

```janet
(defn struct->table [s]
  # TODO: 递归转换所有嵌套的结构体为表
  )

# 测试
(struct->table {:a 1 :b {:c 2}})
# => @{:a 1 :b @{:c 2}}
```

### 练习 2：频率分析

实现一个函数，找出数组中出现最多的元素：

```janet
(defn most-frequent [arr]
  # TODO
  )

(most-frequent [1 2 2 3 3 3])  # => 3
```

### 练习 3：惰性序列

实现一个埃拉托斯特尼筛法的素数生成器：

```janet
(defn primes []
  # TODO: 返回无限素数序列
  )

(take 10 (primes))  # => @[2 3 5 7 11 13 17 19 23 29]
```

<details>
<summary>参考答案</summary>

```janet
(defn primes []
  (generate [n 2 seen @{}]
    (when (not (seen n))
      (yield n)
      # 标记所有倍数
      (var k (* n n))
      (while (< k (+ n 1000))
        (put seen k true)
        (set k (+ k n))))
    (set n (inc n))))
```
</details>

## 3.14 总结

本章我们学习了：

- ✓ Janet 的类型系统
- ✓ 数字、字符串、符号、关键字
- ✓ 不可变集合：元组、结构体
- ✓ 可变集合：数组、表
- ✓ 序列操作和生成器
- ✓ 模式匹配

### 关键要点

1. **@ 前缀表示可变** - 清晰的命名约定
2. **默认不可变** - 鼓励函数式编程
3. **丰富的序列操作** - map、filter、reduce 等
4. **惰性求值** - 通过生成器实现
5. **与 Clojure 高度相似** - 数据结构语法几乎相同

---

← [上一章：核心语言特性](./02-core-language.md) | [返回目录](./README.md) | [下一章：函数式编程](./04-functional-programming.md) →
