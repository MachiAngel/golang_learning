# Chapter 58: 性能優化與分析

## 概述

性能優化是構建高性能應用的關鍵。本章將介紹 Go 的性能分析工具（pprof、trace），以及與 Node.js 性能分析工具（node --inspect、clinic.js）的對比。

## Go vs Node.js 性能分析對比

### Node.js 性能分析
```bash
# 使用 Node.js Inspector
node --inspect app.js

# 使用 clinic.js
npm install -g clinic
clinic doctor -- node app.js
clinic flame -- node app.js

# 使用 0x
npm install -g 0x
0x app.js
```

### Go 性能分析
```bash
# CPU 分析
go test -cpuprofile=cpu.prof
go tool pprof cpu.prof

# 內存分析
go test -memprofile=mem.prof
go tool pprof mem.prof

# 執行追蹤
go test -trace=trace.out
go tool trace trace.out
```

## pprof：CPU 和內存分析

### 1. 啟用 pprof

#### 在 HTTP 服務中啟用

```go
package main

import (
    "log"
    "net/http"
    _ "net/http/pprof"  // 導入 pprof
)

func main() {
    // 你的路由設置
    http.HandleFunc("/", handler)

    // pprof 會自動註冊到 /debug/pprof/
    log.Println("Server starting on :8080")
    log.Fatal(http.ListenAndServe(":8080", nil))
}

func handler(w http.ResponseWriter, r *http.Request) {
    // 你的處理邏輯
    w.Write([]byte("Hello, World!"))
}
```

#### 使用 gorilla/mux

```go
package main

import (
    "log"
    "net/http"
    "net/http/pprof"

    "github.com/gorilla/mux"
)

func main() {
    r := mux.NewRouter()

    // 你的業務路由
    r.HandleFunc("/api/users", getUsers)

    // 註冊 pprof 路由
    r.HandleFunc("/debug/pprof/", pprof.Index)
    r.HandleFunc("/debug/pprof/cmdline", pprof.Cmdline)
    r.HandleFunc("/debug/pprof/profile", pprof.Profile)
    r.HandleFunc("/debug/pprof/symbol", pprof.Symbol)
    r.HandleFunc("/debug/pprof/trace", pprof.Trace)

    log.Fatal(http.ListenAndServe(":8080", r))
}
```

### 2. CPU 分析

#### 收集 CPU 分析數據

```bash
# 方法 1: 通過 HTTP 端點
curl http://localhost:8080/debug/pprof/profile?seconds=30 > cpu.prof

# 方法 2: 直接使用 pprof
go tool pprof http://localhost:8080/debug/pprof/profile?seconds=30
```

#### 分析 CPU 數據

```bash
# 交互式分析
go tool pprof cpu.prof

# 命令行
(pprof) top       # 顯示 CPU 占用最高的函數
(pprof) list main.handler  # 顯示函數源代碼
(pprof) web       # 生成調用圖（需要 graphviz）
(pprof) pdf       # 生成 PDF 報告
```

#### 代碼中手動控制

```go
package main

import (
    "os"
    "runtime/pprof"
    "time"
)

func main() {
    // 開始 CPU 分析
    f, err := os.Create("cpu.prof")
    if err != nil {
        panic(err)
    }
    defer f.Close()

    pprof.StartCPUProfile(f)
    defer pprof.StopCPUProfile()

    // 你的代碼
    doWork()
}

func doWork() {
    // 模擬工作負載
    for i := 0; i < 1000000; i++ {
        _ = fibonacci(20)
    }
}

func fibonacci(n int) int {
    if n <= 1 {
        return n
    }
    return fibonacci(n-1) + fibonacci(n-2)
}
```

### 3. 內存分析

#### 收集內存分析數據

```bash
# 通過 HTTP 端點
curl http://localhost:8080/debug/pprof/heap > mem.prof

# 直接使用 pprof
go tool pprof http://localhost:8080/debug/pprof/heap
```

#### 分析內存數據

```bash
# 交互式分析
go tool pprof mem.prof

# 常用命令
(pprof) top              # 內存占用最高的函數
(pprof) list allocator   # 查看分配代碼
(pprof) web              # 可視化
(pprof) alloc_space      # 按分配總量排序
(pprof) inuse_space      # 按當前使用量排序
```

#### 代碼中手動控制

```go
package main

import (
    "os"
    "runtime"
    "runtime/pprof"
)

func main() {
    doWork()

    // 寫入內存分析
    f, err := os.Create("mem.prof")
    if err != nil {
        panic(err)
    }
    defer f.Close()

    runtime.GC() // 先執行 GC
    if err := pprof.WriteHeapProfile(f); err != nil {
        panic(err)
    }
}

func doWork() {
    // 分配大量內存
    data := make([][]byte, 1000)
    for i := range data {
        data[i] = make([]byte, 1024*1024) // 1 MB
    }
}
```

### 4. Goroutine 分析

```bash
# 查看所有 goroutine
curl http://localhost:8080/debug/pprof/goroutine > goroutine.prof
go tool pprof goroutine.prof

# 或直接訪問
go tool pprof http://localhost:8080/debug/pprof/goroutine
```

### 5. 阻塞分析

```go
package main

import (
    "net/http"
    _ "net/http/pprof"
    "runtime"
)

func main() {
    // 啟用阻塞分析
    runtime.SetBlockProfileRate(1)

    // 你的代碼
    http.HandleFunc("/", handler)
    http.ListenAndServe(":8080", nil)
}
```

```bash
# 獲取阻塞分析
curl http://localhost:8080/debug/pprof/block > block.prof
go tool pprof block.prof
```

## trace：執行追蹤

trace 工具提供了更詳細的程序執行信息。

### 1. 收集 trace 數據

#### HTTP 端點

```bash
curl http://localhost:8080/debug/pprof/trace?seconds=5 > trace.out
```

#### 代碼中手動控制

```go
package main

import (
    "os"
    "runtime/trace"
)

func main() {
    f, err := os.Create("trace.out")
    if err != nil {
        panic(err)
    }
    defer f.Close()

    if err := trace.Start(f); err != nil {
        panic(err)
    }
    defer trace.Stop()

    // 你的代碼
    doWork()
}
```

### 2. 分析 trace 數據

```bash
# 啟動 trace 查看器
go tool trace trace.out

# 瀏覽器會自動打開，顯示：
# - View trace: 時間線視圖
# - Goroutine analysis: Goroutine 分析
# - Network blocking profile: 網絡阻塞
# - Synchronization blocking profile: 同步阻塞
# - Syscall blocking profile: 系統調用阻塞
# - Scheduler latency profile: 調度延遲
```

## Benchmark：性能基準測試

### 1. 基本 Benchmark

```go
// fibonacci_test.go
package main

import "testing"

func BenchmarkFibonacci(b *testing.B) {
    for i := 0; i < b.N; i++ {
        fibonacci(20)
    }
}

func BenchmarkFibonacciParallel(b *testing.B) {
    b.RunParallel(func(pb *testing.PB) {
        for pb.Next() {
            fibonacci(20)
        }
    })
}
```

運行 benchmark：
```bash
# 運行所有 benchmark
go test -bench=.

# 運行特定 benchmark
go test -bench=BenchmarkFibonacci

# 顯示內存分配
go test -bench=. -benchmem

# 設置運行時間
go test -bench=. -benchtime=10s

# CPU 分析
go test -bench=. -cpuprofile=cpu.prof
go tool pprof cpu.prof
```

### 2. 比較 Benchmark

```go
package main

import "testing"

// 方法 1: 使用 strings.Builder
func BenchmarkStringBuilder(b *testing.B) {
    for i := 0; i < b.N; i++ {
        var builder strings.Builder
        for j := 0; j < 100; j++ {
            builder.WriteString("hello")
        }
        _ = builder.String()
    }
}

// 方法 2: 使用 += 操作符
func BenchmarkStringConcat(b *testing.B) {
    for i := 0; i < b.N; i++ {
        var s string
        for j := 0; j < 100; j++ {
            s += "hello"
        }
        _ = s
    }
}

// 方法 3: 使用 bytes.Buffer
func BenchmarkBytesBuffer(b *testing.B) {
    for i := 0; i < b.N; i++ {
        var buf bytes.Buffer
        for j := 0; j < 100; j++ {
            buf.WriteString("hello")
        }
        _ = buf.String()
    }
}
```

運行對比：
```bash
go test -bench=BenchmarkString -benchmem
```

### 3. 使用 benchstat 比較結果

```bash
# 安裝 benchstat
go install golang.org/x/perf/cmd/benchstat@latest

# 運行 benchmark 多次並保存結果
go test -bench=. -count=10 > old.txt

# 修改代碼後再次運行
go test -bench=. -count=10 > new.txt

# 比較結果
benchstat old.txt new.txt
```

## 性能優化實例

### 1. 減少內存分配

#### 問題代碼
```go
func processData(data []string) []string {
    result := []string{}  // 每次擴容都會分配新內存
    for _, item := range data {
        result = append(result, strings.ToUpper(item))
    }
    return result
}
```

#### 優化代碼
```go
func processDataOptimized(data []string) []string {
    result := make([]string, 0, len(data))  // 預分配容量
    for _, item := range data {
        result = append(result, strings.ToUpper(item))
    }
    return result
}
```

### 2. 使用 sync.Pool

#### 問題代碼
```go
func handler(w http.ResponseWriter, r *http.Request) {
    buffer := new(bytes.Buffer)  // 每次請求都分配
    // 使用 buffer
    buffer.WriteString("Hello")
    w.Write(buffer.Bytes())
}
```

#### 優化代碼
```go
var bufferPool = sync.Pool{
    New: func() interface{} {
        return new(bytes.Buffer)
    },
}

func handlerOptimized(w http.ResponseWriter, r *http.Request) {
    buffer := bufferPool.Get().(*bytes.Buffer)
    defer func() {
        buffer.Reset()
        bufferPool.Put(buffer)
    }()

    buffer.WriteString("Hello")
    w.Write(buffer.Bytes())
}
```

### 3. 避免不必要的字符串轉換

#### 問題代碼
```go
func contains(data []byte, substr string) bool {
    return strings.Contains(string(data), substr)  // 不必要的轉換
}
```

#### 優化代碼
```go
func containsOptimized(data []byte, substr string) bool {
    return bytes.Contains(data, []byte(substr))
}
```

### 4. 使用 strings.Builder

#### 問題代碼
```go
func buildString(items []string) string {
    result := ""
    for _, item := range items {
        result += item  // O(n²) 複雜度
    }
    return result
}
```

#### 優化代碼
```go
func buildStringOptimized(items []string) string {
    var builder strings.Builder
    builder.Grow(len(items) * 10)  // 預分配
    for _, item := range items {
        builder.WriteString(item)
    }
    return builder.String()
}
```

## Node.js 性能分析對比

### 使用 Node.js Inspector

```javascript
// 啟動應用
node --inspect app.js

// 或在代碼中啟動
require('inspector').open(9229, '0.0.0.0', true);

// Chrome DevTools
// 打開 chrome://inspect
```

### 使用 clinic.js

```bash
# Doctor（整體診斷）
clinic doctor -- node app.js

# Flame（火焰圖）
clinic flame -- node app.js

# Bubbleprof（異步分析）
clinic bubbleprof -- node app.js
```

### 使用 autocannon 壓測

```bash
# 安裝
npm install -g autocannon

# 壓測
autocannon -c 100 -d 30 http://localhost:3000
```

### 對比

| 特性 | Go (pprof) | Node.js (Inspector) |
|------|------------|---------------------|
| CPU 分析 | 優秀 | 良好 |
| 內存分析 | 優秀 | 良好 |
| 易用性 | 中等 | 簡單（瀏覽器） |
| 可視化 | 需要額外工具 | 內置 |
| 性能開銷 | 低 | 中等 |

## 實際優化案例

### 案例：優化 HTTP 處理器

#### 初始版本

```go
package main

import (
    "encoding/json"
    "net/http"
)

type User struct {
    ID   int    `json:"id"`
    Name string `json:"name"`
}

func getUsers(w http.ResponseWriter, r *http.Request) {
    users := []User{
        {ID: 1, Name: "Alice"},
        {ID: 2, Name: "Bob"},
    }

    data, _ := json.Marshal(users)
    w.Header().Set("Content-Type", "application/json")
    w.Write(data)
}
```

#### Benchmark 測試

```go
func BenchmarkGetUsers(b *testing.B) {
    req := httptest.NewRequest("GET", "/users", nil)
    w := httptest.NewRecorder()

    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        getUsers(w, req)
    }
}
```

#### 優化版本

```go
var userPool = sync.Pool{
    New: func() interface{} {
        return new(bytes.Buffer)
    },
}

func getUsersOptimized(w http.ResponseWriter, r *http.Request) {
    users := []User{
        {ID: 1, Name: "Alice"},
        {ID: 2, Name: "Bob"},
    }

    buf := userPool.Get().(*bytes.Buffer)
    defer func() {
        buf.Reset()
        userPool.Put(buf)
    }()

    json.NewEncoder(buf).Encode(users)

    w.Header().Set("Content-Type", "application/json")
    w.Write(buf.Bytes())
}
```

## 監控工具

### 1. Prometheus + Grafana

```go
package main

import (
    "net/http"

    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promauto"
    "github.com/prometheus/client_golang/prometheus/promhttp"
)

var (
    httpRequestsTotal = promauto.NewCounterVec(
        prometheus.CounterOpts{
            Name: "http_requests_total",
            Help: "Total number of HTTP requests",
        },
        []string{"method", "endpoint"},
    )

    httpRequestDuration = promauto.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "http_request_duration_seconds",
            Help:    "HTTP request duration in seconds",
            Buckets: prometheus.DefBuckets,
        },
        []string{"method", "endpoint"},
    )
)

func metricsMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        timer := prometheus.NewTimer(httpRequestDuration.WithLabelValues(r.Method, r.URL.Path))
        defer timer.ObserveDuration()

        next.ServeHTTP(w, r)

        httpRequestsTotal.WithLabelValues(r.Method, r.URL.Path).Inc()
    })
}

func main() {
    http.Handle("/metrics", promhttp.Handler())
    http.Handle("/", metricsMiddleware(http.HandlerFunc(handler)))
    http.ListenAndServe(":8080", nil)
}
```

## 重點總結

### Go 性能分析工具

1. **pprof**
   - CPU 分析
   - 內存分析
   - Goroutine 分析
   - 阻塞分析

2. **trace**
   - 詳細的執行追蹤
   - Goroutine 調度
   - GC 事件

3. **benchmark**
   - 性能基準測試
   - 回歸測試
   - 比較優化效果

### 優化技巧

1. **減少分配**
   - 預分配切片容量
   - 使用 sync.Pool
   - 避免不必要的轉換

2. **並發優化**
   - 使用 goroutine
   - 控制並發數
   - 避免鎖競爭

3. **算法優化**
   - 選擇合適的數據結構
   - 減少時間複雜度
   - 緩存計算結果

## 練習題

### 練習 1: 性能分析

為以下代碼進行性能分析，找出瓶頸：
```go
func processLargeData(data []int) int {
    sum := 0
    for _, v := range data {
        sum += fibonacci(v)
    }
    return sum
}
```

### 練習 2: Benchmark 對比

對比以下三種方法的性能：
1. map 查找
2. 切片線性搜索
3. 二分搜索

### 練習 3: 內存優化

優化以下代碼以減少內存分配：
```go
func buildHTML(items []string) string {
    html := "<ul>"
    for _, item := range items {
        html += "<li>" + item + "</li>"
    }
    html += "</ul>"
    return html
}
```

### 練習 4: 並發優化

使用 goroutine 優化批量處理：
```go
func processBatch(items []Item) []Result {
    results := make([]Result, len(items))
    for i, item := range items {
        results[i] = process(item)  // 耗時操作
    }
    return results
}
```

### 練習 5: 實現監控

為 HTTP 服務添加：
1. 請求計數
2. 響應時間分布
3. 錯誤率監控

---

下一章：[Chapter 59: 生產環境最佳實踐](59-production.md)
