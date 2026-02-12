# 第十四章：实战项目

通过完整的实战项目，将所学的 Janet 知识应用到系统级开发中。

## 14.1 项目 1：命令行工具 - 文件搜索器

### 需求分析

实现一个类似 `grep` 的文件搜索工具，支持：
- 递归搜索目录
- 正则表达式匹配
- 显示行号和上下文
- 彩色输出

### 实现

```janet
# filesearch.janet

(def VERSION "1.0.0")

(defn print-usage []
  (print `
File Search Tool v` VERSION `

Usage: janet filesearch.janet [options] PATTERN [PATH]

Options:
  -r, --recursive    Recursively search directories
  -n, --line-number  Show line numbers
  -i, --ignore-case  Case insensitive search
  -c, --context N    Show N lines of context
  -h, --help         Show this help message

Example:
  janet filesearch.janet -r -n "TODO" ./src
`))

(defn parse-args [args]
  (def opts @{:pattern nil
              :path "."
              :recursive false
              :line-number false
              :ignore-case false
              :context 0})
  
  (var i 0)
  (while (< i (length args))
    (def arg (get args i))
    (case arg
      "-r" (put opts :recursive true)
      "--recursive" (put opts :recursive true)
      "-n" (put opts :line-number true)
      "--line-number" (put opts :line-number true)
      "-i" (put opts :ignore-case true)
      "--ignore-case" (put opts :ignore-case true)
      "-c" (do
            (set i (inc i))
            (put opts :context (scan-number (get args i))))
      "--context" (do
                   (set i (inc i))
                   (put opts :context (scan-number (get args i))))
      "-h" (do (print-usage) (os/exit 0))
      "--help" (do (print-usage) (os/exit 0))
      (if (nil? (opts :pattern))
        (put opts :pattern arg)
        (put opts :path arg)))
    (set i (inc i)))
  
  (when (nil? (opts :pattern))
    (print-usage)
    (os/exit 1))
  
  opts)

(defn search-file [path pattern opts]
  (def lines (string/split "\n" (slurp path)))
  (def matches @[])
  
  (def peg-pattern
    (if (opts :ignore-case)
      (peg/compile ~(* (any (if-not ,pattern 1)) ,pattern))
      (peg/compile ~(* (any (if-not ,pattern 1)) (capture ,pattern)))))
  
  (each [line-num line] (pairs lines)
    (when (string/find pattern line)
      (array/push matches {:line-num (inc line-num) :line line})))
  
  (when (> (length matches) 0)
    (print "\n" path ":")
    (each match matches
      (if (opts :line-number)
        (printf "%d: %s" (match :line-num) (match :line))
        (print (match :line))))))

(defn search-directory [path pattern opts]
  (each entry (os/dir path)
    (def full-path (string path "/" entry))
    (def stat (os/stat full-path))
    
    (cond
      # 跳过隐藏文件
      (string/has-prefix? "." entry) nil
      
      # 递归搜索子目录
      (and (opts :recursive) (= (stat :mode) :directory))
      (search-directory full-path pattern opts)
      
      # 搜索文件
      (= (stat :mode) :file)
      (try
        (search-file full-path pattern opts)
        ([err]
         (eprintf "Error reading %s: %s" full-path err))))))

(defn main [& args]
  (def opts (parse-args args))
  (def path (opts :path))
  (def pattern (opts :pattern))
  
  (def stat (os/stat path))
  (if (= (stat :mode) :directory)
    (search-directory path pattern opts)
    (search-file path pattern opts)))

# 运行
(main ;(slice (dyn :args) 1))
```

### 增强功能

```janet
# 添加彩色输出
(def ANSI_RED "\e[31m")
(def ANSI_GREEN "\e[32m")
(def ANSI_YELLOW "\e[33m")
(def ANSI_RESET "\e[0m")

(defn colorize [text color]
  (string color text ANSI_RESET))

# 在匹配处高亮
(defn highlight-match [line pattern]
  (string/replace pattern (colorize pattern ANSI_RED) line))
```

## 14.2 项目 2：网络爬虫

### 功能设计

- 递归抓取网页
- 遵守 robots.txt
- 并发下载
- 速率限制
- 导出数据

### 核心实现

```janet
# crawler.janet

(import spork/json)
(import spork/base64)

(defn parse-url [url]
  # 简化的 URL 解析
  (def pattern (peg/compile
                ~{:protocol (capture (to "://"))
                  :host (capture (to (+ "/" :end)))
                  :path (capture (to :end))
                  :main (* :protocol "://" :host :path)}))
  
  (def result (peg/match pattern url))
  (when result
    {:protocol (get result 0)
     :host (get result 1)
     :path (or (get result 2) "/")}))

(defn fetch-page [url]
  (def parsed (parse-url url))
  (when parsed
    (def conn (net/connect (parsed :host) "80"))
    (defer (:close conn)
      (def request
        (string/format "GET %s HTTP/1.1\r\nHost: %s\r\nConnection: close\r\n\r\n"
                       (parsed :path) (parsed :host)))
      (:write conn request)
      
      # 读取响应
      (def response @"")
      (def buf @"")
      (while (:read conn 4096 buf)
        (buffer/push response buf)
        (buffer/clear buf))
      
      (string response))))

(defn extract-links [html base-url]
  # 提取所有链接
  (def pattern (peg/compile
                ~(* "href=\"" (capture (to "\"")))))
  (def matches (peg/match pattern html))
  
  # 过滤和规范化链接
  (def links @[])
  (each match matches
    (when (and match (not (string/has-prefix? "#" match)))
      (array/push links match)))
  links)

(defn crawl [start-url max-depth max-pages]
  (def visited @{})
  (def to-visit @[[start-url 0]])
  (def pages @[])
  
  (while (and (> (length to-visit) 0)
              (< (length pages) max-pages))
    (def [url depth] (array/pop to-visit))
    
    (when (and (not (visited url))
               (<= depth max-depth))
      (printf "Crawling: %s (depth %d)" url depth)
      (put visited url true)
      
      (try
        (def html (fetch-page url))
        (array/push pages {:url url :depth depth :size (length html)})
        
        # 提取链接
        (when (< depth max-depth)
          (def links (extract-links html url))
          (each link links
            (array/push to-visit [link (inc depth)])))
        
        # 速率限制
        (ev/sleep 1)
        
        ([err]
         (eprintf "Error crawling %s: %s" url err)))))
  
  pages)

(defn main [& args]
  (when (< (length args) 1)
    (print "Usage: janet crawler.janet <start-url> [max-depth] [max-pages]")
    (os/exit 1))
  
  (def start-url (get args 0))
  (def max-depth (or (scan-number (get args 1)) 2))
  (def max-pages (or (scan-number (get args 2)) 50))
  
  (print "Starting crawler...")
  (def results (crawl start-url max-depth max-pages))
  
  # 保存结果
  (spit "crawl-results.json" (json/encode results))
  (printf "Crawled %d pages, results saved to crawl-results.json"
          (length results)))

(main ;(slice (dyn :args) 1))
```

## 14.3 项目 3：日志分析器

### 功能

- 解析多种日志格式
- 统计分析
- 生成报告
- 实时监控

### 实现

```janet
# loganalyzer.janet

(defn parse-nginx-log [line]
  # 解析 Nginx 访问日志
  (def pattern
    (peg/compile
     ~{:ip (capture (some (+ (range "09") ".")))
       :date (capture (between "[" "]"))
       :method (capture (some :a))
       :path (capture (to "\""))
       :status (capture (some :d))
       :main (* :ip " - - " :date " \"" :method " " :path "\" " :status)}))
  
  (def result (peg/match pattern line))
  (when result
    {:ip (get result 0)
     :date (get result 1)
     :method (get result 2)
     :path (get result 3)
     :status (scan-number (get result 4))}))

(defn analyze-logs [log-file]
  (def stats @{:total 0
               :by-status @{}
               :by-path @{}
               :by-ip @{}
               :errors @[]})
  
  # 读取日志
  (def f (file/open log-file :read))
  (defer (file/close f)
    (while (def line (file/read f :line))
      (when-let [entry (parse-nginx-log line)]
        (++ (stats :total))
        
        # 按状态码统计
        (def status (entry :status))
        (put (stats :by-status) status
             (inc (get (stats :by-status) status 0)))
        
        # 按路径统计
        (def path (entry :path))
        (put (stats :by-path) path
             (inc (get (stats :by-path) path 0)))
        
        # 按 IP 统计
        (def ip (entry :ip))
        (put (stats :by-ip) ip
             (inc (get (stats :by-ip) ip 0)))
        
        # 收集错误
        (when (>= status 400)
          (array/push (stats :errors) entry)))))
  
  stats)

(defn generate-report [stats]
  (print "\n=== Log Analysis Report ===\n")
  
  (printf "Total requests: %d\n" (stats :total))
  
  # 状态码分布
  (print "\nStatus Code Distribution:")
  (each [status count] (pairs (stats :by-status))
    (printf "  %d: %d (%.2f%%)"
            status count
            (* 100 (/ count (stats :total)))))
  
  # Top 10 路径
  (print "\nTop 10 Paths:")
  (def sorted-paths
    (sorted (pairs (stats :by-path))
            (fn [[_ c1] [_ c2]] (> c1 c2))))
  (each [path count] (take 10 sorted-paths)
    (printf "  %s: %d" path count))
  
  # Top 10 IP
  (print "\nTop 10 IPs:")
  (def sorted-ips
    (sorted (pairs (stats :by-ip))
            (fn [[_ c1] [_ c2]] (> c1 c2))))
  (each [ip count] (take 10 sorted-ips)
    (printf "  %s: %d" ip count))
  
  # 错误统计
  (printf "\nTotal errors: %d" (length (stats :errors))))

(defn main [& args]
  (when (< (length args) 1)
    (print "Usage: janet loganalyzer.janet <log-file>")
    (os/exit 1))
  
  (def log-file (get args 0))
  (print "Analyzing logs...")
  
  (def stats (analyze-logs log-file))
  (generate-report stats))

(main ;(slice (dyn :args) 1))
```

## 14.4 项目 4：简单的 HTTP 服务器框架

### 设计目标

- 路由系统
- 中间件支持
- 模板引擎
- 静态文件服务

### 实现

```janet
# webframework.janet

(defn make-router []
  @{:routes @[]
    :middleware @[]})

(defn add-route [router method path handler]
  (array/push (router :routes)
              {:method method :path path :handler handler}))

(defn add-middleware [router mw]
  (array/push (router :middleware) mw))

(defn match-route [routes method path]
  (find (fn [route]
          (and (= (route :method) method)
               (= (route :path) path)))
        routes))

(defn apply-middleware [handler middleware]
  (reduce (fn [h mw] (mw h))
          handler
          (reverse middleware)))

(defn handle-request [router req]
  (def route (match-route (router :routes)
                          (req :method)
                          (req :path)))
  
  (if route
    (let [handler (apply-middleware (route :handler)
                                    (router :middleware))]
      (handler req))
    {:status 404 :body "Not Found"}))

# 中间件示例
(defn logger-middleware [handler]
  (fn [req]
    (printf "[%s] %s %s"
            (os/date)
            (req :method)
            (req :path))
    (handler req)))

(defn cors-middleware [handler]
  (fn [req]
    (def resp (handler req))
    (merge resp {:headers {"Access-Control-Allow-Origin" "*"}})))

# 使用示例
(def app (make-router))

(add-middleware app logger-middleware)
(add-middleware app cors-middleware)

(add-route app "GET" "/"
           (fn [req]
             {:status 200
              :body "<h1>Welcome!</h1>"}))

(add-route app "GET" "/api/hello"
           (fn [req]
             {:status 200
              :headers {"Content-Type" "application/json"}
              :body `{"message": "Hello, World!"}`}))

# 启动服务器
(defn start-server [router host port]
  (defn handler [stream]
    (defer (:close stream)
      # 解析请求
      (def request-str (:read stream 4096))
      (def req (parse-http request-str))
      
      # 处理请求
      (def resp (handle-request router req))
      
      # 发送响应
      (def response (make-response (resp :status)
                                   (resp :body)
                                   (get resp :headers @{})))
      (:write stream response)))
  
  (printf "Server running on http://%s:%s" host port)
  (net/server host port handler))

# (start-server app "127.0.0.1" "8080")
```

## 14.5 项目 5：系统监控守护进程

### 完整实现

```janet
# sysmon.janet

(defn get-metrics []
  @{:timestamp (os/time)
    :cpu (get-cpu-usage)
    :memory (get-memory-info)
    :disk (get-disk-usage "/")})

(defn check-alerts [metrics thresholds]
  (def alerts @[])
  
  # CPU 告警
  (when (> (get-in metrics [:cpu :usage]) (thresholds :cpu-max))
    (array/push alerts
                (string/format "CPU usage high: %.2f%%"
                              (get-in metrics [:cpu :usage]))))
  
  # 内存告警
  (when (> (get-in metrics [:memory :usage]) (thresholds :mem-max))
    (array/push alerts
                (string/format "Memory usage high: %.2f%%"
                              (get-in metrics [:memory :usage]))))
  
  alerts)

(defn send-alert [message]
  # 发送告警（邮件/Slack/webhook）
  (print "ALERT: " message))

(defn monitor-loop [config]
  (def interval (get config :interval 60))
  (def thresholds (get config :thresholds
                       {:cpu-max 80 :mem-max 90}))
  
  (while true
    (def metrics (get-metrics))
    
    # 记录指标
    (def log-entry (string/format "[%s] CPU: %.2f%%, Memory: %.2f%%\n"
                                  (os/date (metrics :timestamp))
                                  (get-in metrics [:cpu :usage])
                                  (get-in metrics [:memory :usage])))
    (spit "sysmon.log" log-entry :append)
    
    # 检查告警
    (def alerts (check-alerts metrics thresholds))
    (each alert alerts
      (send-alert alert))
    
    (ev/sleep interval)))

(defn main [& args]
  (def config @{:interval 60
                :thresholds {:cpu-max 80 :mem-max 90}})
  
  # 守护进程化
  # (daemonize)
  
  # 写入 PID 文件
  (write-pidfile "/var/run/sysmon.pid")
  
  # 注册信号处理
  (var running true)
  (os/sigaction :term
                (fn [_]
                  (print "Shutting down...")
                  (set running false)))
  
  # 主监控循环
  (try
    (monitor-loop config)
    ([err]
     (eprintf "Monitor error: %s" err)))
  
  # 清理
  (remove-pidfile "/var/run/sysmon.pid"))

(main ;(slice (dyn :args) 1))
```

## 14.6 项目总结

### 学到的技能

1. **命令行工具开发**
   - 参数解析
   - 文件系统操作
   - 用户交互

2. **网络应用**
   - HTTP 客户端和服务器
   - 异步 I/O
   - 并发处理

3. **数据处理**
   - 日志解析
   - 统计分析
   - 报告生成

4. **系统编程**
   - 进程管理
   - 资源监控
   - 守护进程

5. **软件工程**
   - 模块化设计
   - 错误处理
   - 测试和调试

### 进阶方向

- **性能优化** - 使用 FFI 加速关键路径
- **分布式系统** - 添加远程通信和协调
- **Web 应用** - 完整的 Web 框架
- **数据库集成** - SQLite、PostgreSQL 等
- **容器化** - Docker 部署
- **监控和日志** - Prometheus、Grafana 集成

## 14.7 部署建议

### 打包应用

```bash
# 使用 JPM 构建
jpm build

# 创建独立可执行文件
janet -c filesearch.janet -o filesearch

# 或使用 jpm
jpm install
```

### 系统集成

```bash
# 创建 systemd 服务
cat > /etc/systemd/system/sysmon.service << EOF
[Unit]
Description=System Monitor Daemon
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/janet /opt/sysmon/sysmon.janet
Restart=always

[Install]
WantedBy=multi-user.target
EOF

# 启动服务
systemctl enable sysmon
systemctl start sysmon
```

## 14.8 总结

通过这些实战项目，你已经掌握了：

- ✓ 完整的应用开发流程
- ✓ Janet 在实际场景中的应用
- ✓ 系统级编程的最佳实践
- ✓ 代码组织和模块化
- ✓ 部署和维护

继续探索和构建更复杂的项目，Janet 的简洁和强大会让你的开发效率大大提升！

---

← [上一章：测试与调试](./13-testing-debugging.md) | [返回目录](./README.md) | [下一章：Spork 扩展标准库](./15-spork-library.md) →
