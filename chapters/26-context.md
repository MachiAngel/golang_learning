# Chapter 26: Context 包

## 概述

context 包是 Go 1.7 引入的標準庫，用於在 goroutines 之間傳遞請求範圍的數據、取消信號和截止時間。Context 是處理超時、取消和請求範圍值的標準方式，特別適合用於 HTTP 服務器、數據庫查詢等場景。

## Node.js 與 Go 的對比

| 特性 | Node.js | Go |
|------|---------|-----|
| 請求上下文 | req 對象 | context.Context |
| 超時控制 | setTimeout, AbortController | context.WithTimeout |
| 取消操作 | AbortController | context.WithCancel |
| 傳遞數據 | req.locals, res.locals | context.WithValue |
| 鏈式傳遞 | 中間件鏈 | context 樹 |
| 生命週期 | 請求-響應 | context 樹 |

## 詳細概念解釋

### 1. Context 接口

```go
type Context interface {
    Deadline() (deadline time.Time, ok bool)
    Done() <-chan struct{}
    Err() error
    Value(key interface{}) interface{}
}
```

### 2. Context 類型

- **Background**: 根 context，永不取消
- **TODO**: 不確定使用哪個 context 時的占位符
- **WithCancel**: 可手動取消
- **WithTimeout**: 超時自動取消
- **WithDeadline**: 到截止時間自動取消
- **WithValue**: 攜帶請求範圍的值

### 3. Context 樹

Context 形成樹狀結構，子 context 繼承父 context 的取消信號：

```
Background
  └── WithTimeout (5s)
      ├── WithValue ("userID", 123)
      └── WithCancel
          └── WithTimeout (1s)
```

## 代碼示例

### 示例 1: 基本 Context 使用

**Go:**
```go
package main

import (
    "context"
    "fmt"
    "time"
)

func basicContext() {
    fmt.Println("=== Basic Context ===")

    // 創建背景 context
    ctx := context.Background()
    fmt.Printf("Background context: %v\n", ctx)

    // 創建 TODO context
    todoCtx := context.TODO()
    fmt.Printf("TODO context: %v\n", todoCtx)
}

func withCancel() {
    fmt.Println("\n=== WithCancel ===")

    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()  // 確保釋放資源

    go func() {
        for {
            select {
            case <-ctx.Done():
                fmt.Println("Goroutine cancelled:", ctx.Err())
                return
            default:
                fmt.Println("Working...")
                time.Sleep(500 * time.Millisecond)
            }
        }
    }()

    time.Sleep(2 * time.Second)
    cancel()  // 取消 context
    time.Sleep(500 * time.Millisecond)
}

func withTimeout() {
    fmt.Println("\n=== WithTimeout ===")

    // 創建 2 秒超時的 context
    ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
    defer cancel()

    select {
    case <-time.After(3 * time.Second):
        fmt.Println("Operation completed")
    case <-ctx.Done():
        fmt.Println("Operation timeout:", ctx.Err())
    }
}

func withDeadline() {
    fmt.Println("\n=== WithDeadline ===")

    deadline := time.Now().Add(2 * time.Second)
    ctx, cancel := context.WithDeadline(context.Background(), deadline)
    defer cancel()

    select {
    case <-time.After(3 * time.Second):
        fmt.Println("Operation completed")
    case <-ctx.Done():
        fmt.Println("Deadline exceeded:", ctx.Err())

        if deadline, ok := ctx.Deadline(); ok {
            fmt.Println("Deadline was:", deadline)
        }
    }
}

func main() {
    basicContext()
    withCancel()
    withTimeout()
    withDeadline()
}
```

**Node.js 對比:**
```javascript
// Node.js 使用 AbortController 實現類似功能

async function basicAbortController() {
    console.log("=== Basic AbortController ===");

    const controller = new AbortController();
    const signal = controller.signal;

    setTimeout(() => controller.abort(), 2000);

    const worker = async () => {
        while (!signal.aborted) {
            console.log("Working...");
            await new Promise(resolve => setTimeout(resolve, 500));
        }
        console.log("Worker cancelled");
    };

    worker();

    await new Promise(resolve => {
        signal.addEventListener('abort', () => {
            console.log("Aborted:", signal.reason || 'cancelled');
            resolve();
        });
    });
}

async function withTimeout() {
    console.log("\n=== With Timeout ===");

    const controller = new AbortController();
    const timeoutId = setTimeout(() => controller.abort('timeout'), 2000);

    try {
        await new Promise((resolve, reject) => {
            controller.signal.addEventListener('abort', () => {
                reject(new Error('Operation timeout'));
            });

            setTimeout(() => {
                clearTimeout(timeoutId);
                resolve('completed');
            }, 3000);
        });
        console.log("Operation completed");
    } catch (err) {
        console.log(err.message);
    }
}

async function main() {
    await basicAbortController();
    await withTimeout();
}

main();
```

### 示例 2: Context 傳值

**Go:**
```go
package main

import (
    "context"
    "fmt"
)

type contextKey string

const (
    userIDKey  contextKey = "userID"
    requestIDKey contextKey = "requestID"
)

func withValue() {
    fmt.Println("=== WithValue ===")

    // 創建帶值的 context
    ctx := context.Background()
    ctx = context.WithValue(ctx, userIDKey, 123)
    ctx = context.WithValue(ctx, requestIDKey, "req-456")

    // 讀取值
    if userID, ok := ctx.Value(userIDKey).(int); ok {
        fmt.Println("User ID:", userID)
    }

    if requestID, ok := ctx.Value(requestIDKey).(string); ok {
        fmt.Println("Request ID:", requestID)
    }

    // 不存在的鍵
    if val := ctx.Value(contextKey("unknown")); val != nil {
        fmt.Println("Unknown:", val)
    } else {
        fmt.Println("Key not found")
    }
}

// 實際應用：HTTP 請求鏈
func processRequest(ctx context.Context) {
    userID := ctx.Value(userIDKey)
    requestID := ctx.Value(requestIDKey)

    fmt.Printf("Processing request %v for user %v\n", requestID, userID)

    // 傳遞給下游函數
    queryDatabase(ctx)
}

func queryDatabase(ctx context.Context) {
    userID := ctx.Value(userIDKey)
    fmt.Printf("Querying database for user %v\n", userID)

    // 可以訪問父 context 的值
}

func contextChain() {
    fmt.Println("\n=== Context Chain ===")

    ctx := context.Background()
    ctx = context.WithValue(ctx, userIDKey, 789)
    ctx = context.WithValue(ctx, requestIDKey, "req-001")

    processRequest(ctx)
}

// 注意事項：不要濫用 WithValue
func wrongUsage() {
    fmt.Println("\n=== Wrong Usage (Don't Do This) ===")

    // 錯誤：使用字符串作為鍵（可能衝突）
    ctx := context.WithValue(context.Background(), "user", "alice")

    // 正確：使用自定義類型作為鍵
    type userKey struct{}
    ctx = context.WithValue(context.Background(), userKey{}, "alice")

    // 錯誤：傳遞大量數據
    // Context 應該只傳遞請求範圍的元數據
    // 不應該傳遞可選參數或大對象
}

func main() {
    withValue()
    contextChain()
    wrongUsage()
}
```

**Node.js 對比:**
```javascript
// Express.js 風格的請求上下文

class RequestContext {
    constructor() {
        this.values = new Map();
    }

    set(key, value) {
        this.values.set(key, value);
    }

    get(key) {
        return this.values.get(key);
    }
}

function processRequest(ctx) {
    const userID = ctx.get('userID');
    const requestID = ctx.get('requestID');

    console.log(`Processing request ${requestID} for user ${userID}`);

    queryDatabase(ctx);
}

function queryDatabase(ctx) {
    const userID = ctx.get('userID');
    console.log(`Querying database for user ${userID}`);
}

function main() {
    console.log("=== Request Context ===");

    const ctx = new RequestContext();
    ctx.set('userID', 789);
    ctx.set('requestID', 'req-001');

    processRequest(ctx);
}

main();

// Express.js 實際用法
/*
app.use((req, res, next) => {
    req.userID = 123;
    req.requestID = 'req-456';
    next();
});

app.get('/api/data', (req, res) => {
    const userID = req.userID;
    const requestID = req.requestID;
    // 處理請求
});
*/
```

### 示例 3: HTTP 服務器中使用 Context

**Go:**
```go
package main

import (
    "context"
    "fmt"
    "net/http"
    "time"
)

func handler(w http.ResponseWriter, r *http.Request) {
    // 從請求獲取 context
    ctx := r.Context()

    // 添加超時
    ctx, cancel := context.WithTimeout(ctx, 3*time.Second)
    defer cancel()

    // 模擬長時間操作
    select {
    case <-time.After(2 * time.Second):
        fmt.Fprintf(w, "Request completed successfully\n")
    case <-ctx.Done():
        // 客戶端斷開連接或超時
        err := ctx.Err()
        fmt.Printf("Request cancelled: %v\n", err)
        http.Error(w, err.Error(), http.StatusRequestTimeout)
    }
}

func slowHandler(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()

    // 模擬數據庫查詢
    result, err := queryDatabaseWithContext(ctx)
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }

    fmt.Fprintf(w, "Result: %s\n", result)
}

func queryDatabaseWithContext(ctx context.Context) (string, error) {
    // 模擬慢查詢
    resultCh := make(chan string)

    go func() {
        time.Sleep(5 * time.Second)
        resultCh <- "database result"
    }()

    select {
    case result := <-resultCh:
        return result, nil
    case <-ctx.Done():
        return "", ctx.Err()
    }
}

// 中間件：添加請求 ID
func requestIDMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        requestID := r.Header.Get("X-Request-ID")
        if requestID == "" {
            requestID = fmt.Sprintf("req-%d", time.Now().UnixNano())
        }

        // 將請求 ID 添加到 context
        ctx := context.WithValue(r.Context(), "requestID", requestID)
        r = r.WithContext(ctx)

        next.ServeHTTP(w, r)
    })
}

func main() {
    http.HandleFunc("/", handler)
    http.HandleFunc("/slow", slowHandler)

    fmt.Println("Server starting on :8080")
    http.ListenAndServe(":8080", nil)
}
```

**Node.js 對比:**
```javascript
const express = require('express');
const app = express();

// 中間件：添加請求 ID
app.use((req, res, next) => {
    req.requestID = req.get('X-Request-ID') || `req-${Date.now()}`;
    next();
});

// 基本處理器
app.get('/', async (req, res) => {
    const controller = new AbortController();
    const timeoutId = setTimeout(() => controller.abort(), 3000);

    try {
        // 模擬長時間操作
        await new Promise((resolve, reject) => {
            controller.signal.addEventListener('abort', () => {
                reject(new Error('Request timeout'));
            });

            setTimeout(() => {
                clearTimeout(timeoutId);
                resolve();
            }, 2000);
        });

        res.send('Request completed successfully\n');
    } catch (err) {
        console.log('Request cancelled:', err.message);
        res.status(408).send(err.message);
    }
});

// 慢處理器
app.get('/slow', async (req, res) => {
    const controller = new AbortController();
    req.on('close', () => controller.abort());

    try {
        const result = await queryDatabase(controller.signal);
        res.send(`Result: ${result}\n`);
    } catch (err) {
        res.status(500).send(err.message);
    }
});

async function queryDatabase(signal) {
    return new Promise((resolve, reject) => {
        signal.addEventListener('abort', () => {
            reject(new Error('Query cancelled'));
        });

        setTimeout(() => {
            resolve('database result');
        }, 5000);
    });
}

app.listen(8080, () => {
    console.log('Server starting on :8080');
});
```

### 示例 4: Context 最佳實踐

**Go:**
```go
package main

import (
    "context"
    "errors"
    "fmt"
    "time"
)

// 1. 函數第一個參數應該是 context
func goodFunction(ctx context.Context, arg string) error {
    select {
    case <-ctx.Done():
        return ctx.Err()
    default:
        fmt.Println("Processing:", arg)
        return nil
    }
}

// 2. 不要將 context 存儲在結構體中
type BadService struct {
    ctx context.Context  // 不推薦
}

type GoodService struct {
    // 不存儲 context
}

func (s *GoodService) DoWork(ctx context.Context) error {
    // 將 context 作為參數傳遞
    return goodFunction(ctx, "work")
}

// 3. 不要傳遞 nil context
func badUsage() {
    // 錯誤
    // goodFunction(nil, "test")

    // 正確
    goodFunction(context.Background(), "test")
}

// 4. 使用自定義類型作為 context key
type contextKey int

const (
    userKey contextKey = iota
    requestKey
)

func goodContextValue() {
    ctx := context.WithValue(context.Background(), userKey, "alice")
    if user, ok := ctx.Value(userKey).(string); ok {
        fmt.Println("User:", user)
    }
}

// 5. 超時鏈
func timeoutChain() {
    // 父 context: 10 秒超時
    parentCtx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()

    // 子 context: 5 秒超時（會先觸發）
    childCtx, cancel := context.WithTimeout(parentCtx, 5*time.Second)
    defer cancel()

    select {
    case <-time.After(7 * time.Second):
        fmt.Println("Work completed")
    case <-childCtx.Done():
        fmt.Println("Child timeout:", childCtx.Err())
    }
}

// 6. 正確處理取消
func correctCancellation(ctx context.Context) error {
    resultCh := make(chan string)
    errCh := make(chan error)

    go func() {
        // 模擬工作
        time.Sleep(2 * time.Second)

        // 檢查是否已取消
        select {
        case <-ctx.Done():
            errCh <- ctx.Err()
            return
        default:
            resultCh <- "result"
        }
    }()

    select {
    case result := <-resultCh:
        fmt.Println("Got result:", result)
        return nil
    case err := <-errCh:
        return err
    case <-ctx.Done():
        return ctx.Err()
    }
}

// 7. 多個操作共享 context
func multipleOperations(ctx context.Context) error {
    // 所有操作共享同一個 context
    if err := operation1(ctx); err != nil {
        return err
    }

    if err := operation2(ctx); err != nil {
        return err
    }

    return operation3(ctx)
}

func operation1(ctx context.Context) error {
    select {
    case <-ctx.Done():
        return ctx.Err()
    case <-time.After(100 * time.Millisecond):
        return nil
    }
}

func operation2(ctx context.Context) error {
    select {
    case <-ctx.Done():
        return ctx.Err()
    case <-time.After(100 * time.Millisecond):
        return nil
    }
}

func operation3(ctx context.Context) error {
    select {
    case <-ctx.Done():
        return ctx.Err()
    case <-time.After(100 * time.Millisecond):
        return nil
    }
}

func main() {
    fmt.Println("=== Good Context Value ===")
    goodContextValue()

    fmt.Println("\n=== Timeout Chain ===")
    timeoutChain()

    fmt.Println("\n=== Correct Cancellation ===")
    ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
    defer cancel()
    correctCancellation(ctx)

    fmt.Println("\n=== Multiple Operations ===")
    ctx2, cancel2 := context.WithTimeout(context.Background(), 1*time.Second)
    defer cancel2()
    if err := multipleOperations(ctx2); err != nil {
        fmt.Println("Error:", err)
    } else {
        fmt.Println("All operations completed")
    }
}
```

### 示例 5: 實際應用場景

**Go:**
```go
package main

import (
    "context"
    "fmt"
    "sync"
    "time"
)

// 場景 1: API 調用鏈
func apiGateway(ctx context.Context) (string, error) {
    ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
    defer cancel()

    // 並行調用多個服務
    var wg sync.WaitGroup
    results := make(chan string, 3)
    errors := make(chan error, 3)

    services := []func(context.Context) (string, error){
        userService,
        productService,
        orderService,
    }

    for _, svc := range services {
        wg.Add(1)
        go func(service func(context.Context) (string, error)) {
            defer wg.Done()

            result, err := service(ctx)
            if err != nil {
                errors <- err
                return
            }
            results <- result
        }(svc)
    }

    // 等待所有服務完成
    go func() {
        wg.Wait()
        close(results)
        close(errors)
    }()

    // 收集結果
    var response string
    for i := 0; i < len(services); i++ {
        select {
        case result := <-results:
            response += result + " "
        case err := <-errors:
            return "", err
        case <-ctx.Done():
            return "", ctx.Err()
        }
    }

    return response, nil
}

func userService(ctx context.Context) (string, error) {
    select {
    case <-time.After(1 * time.Second):
        return "user_data", nil
    case <-ctx.Done():
        return "", ctx.Err()
    }
}

func productService(ctx context.Context) (string, error) {
    select {
    case <-time.After(1 * time.Second):
        return "product_data", nil
    case <-ctx.Done():
        return "", ctx.Err()
    }
}

func orderService(ctx context.Context) (string, error) {
    select {
    case <-time.After(1 * time.Second):
        return "order_data", nil
    case <-ctx.Done():
        return "", ctx.Err()
    }
}

// 場景 2: 工作池與取消
func workerPool(ctx context.Context, jobs <-chan int) {
    var wg sync.WaitGroup

    for i := 1; i <= 3; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()

            for {
                select {
                case job, ok := <-jobs:
                    if !ok {
                        fmt.Printf("Worker %d: jobs channel closed\n", id)
                        return
                    }
                    fmt.Printf("Worker %d processing job %d\n", id, job)
                    time.Sleep(500 * time.Millisecond)

                case <-ctx.Done():
                    fmt.Printf("Worker %d cancelled\n", id)
                    return
                }
            }
        }(i)
    }

    wg.Wait()
}

func workerPoolDemo() {
    fmt.Println("=== Worker Pool Demo ===")

    ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
    defer cancel()

    jobs := make(chan int, 10)

    // 啟動工作池
    go workerPool(ctx, jobs)

    // 發送任務
    for i := 1; i <= 10; i++ {
        jobs <- i
        time.Sleep(200 * time.Millisecond)
    }

    close(jobs)
    time.Sleep(4 * time.Second)
}

// 場景 3: 優雅關閉
func gracefulShutdown() {
    fmt.Println("\n=== Graceful Shutdown ===")

    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()

    // 啟動服務
    go runService(ctx)

    // 模擬運行一段時間
    time.Sleep(2 * time.Second)

    // 發送關閉信號
    fmt.Println("Shutting down...")
    cancel()

    // 等待清理
    time.Sleep(1 * time.Second)
}

func runService(ctx context.Context) {
    ticker := time.NewTicker(500 * time.Millisecond)
    defer ticker.Stop()

    for {
        select {
        case <-ticker.C:
            fmt.Println("Service working...")
        case <-ctx.Done():
            fmt.Println("Service cleanup...")
            time.Sleep(500 * time.Millisecond)
            fmt.Println("Service stopped")
            return
        }
    }
}

func main() {
    fmt.Println("=== API Gateway Demo ===")
    ctx := context.Background()
    result, err := apiGateway(ctx)
    if err != nil {
        fmt.Println("Error:", err)
    } else {
        fmt.Println("Result:", result)
    }

    workerPoolDemo()
    gracefulShutdown()
}
```

## 重點總結

### Context vs Node.js 對比

| 特性 | Go Context | Node.js |
|------|-----------|---------|
| 請求上下文 | Context 樹 | req 對象 |
| 超時控制 | WithTimeout | AbortController + timeout |
| 取消操作 | Done() channel | AbortController |
| 傳遞值 | WithValue | req.locals |
| 生命週期 | 明確定義 | 請求週期 |

### 最佳實踐

1. **函數簽名**：
   - Context 應該是第一個參數
   - 參數名使用 `ctx`
   - 不要存儲在結構體中

2. **創建 Context**：
   - 使用 `context.Background()` 作為根
   - 不要傳遞 `nil` context
   - 使用 `context.TODO()` 作為占位符

3. **傳遞值**：
   - 只傳遞請求範圍的數據
   - 使用自定義類型作為鍵
   - 不要傳遞可選參數

4. **取消處理**：
   - 總是 `defer cancel()`
   - 檢查 `ctx.Done()`
   - 處理 `ctx.Err()`

5. **超時設置**：
   - 合理設置超時時間
   - 考慮超時鏈
   - 提前取消釋放資源

## 練習題

### 練習 1: HTTP 客戶端
實現一個支持超時和取消的 HTTP 客戶端。

### 練習 2: 數據庫查詢
實現帶 context 的數據庫查詢函數。

### 練習 3: 批處理系統
實現一個批處理系統，支持取消和超時。

### 練習 4: 服務編排
實現一個服務編排器，並行調用多個服務。

### 練習 5: 限流器
使用 context 實現一個請求限流器。

### 答案提示

**練習 1:**
```go
func HTTPGet(ctx context.Context, url string) (string, error) {
    req, err := http.NewRequestWithContext(ctx, "GET", url, nil)
    if err != nil {
        return "", err
    }

    client := &http.Client{}
    resp, err := client.Do(req)
    if err != nil {
        return "", err
    }
    defer resp.Body.Close()

    body, err := io.ReadAll(resp.Body)
    if err != nil {
        return "", err
    }

    return string(body), nil
}
```

**練習 2:**
```go
func QueryUser(ctx context.Context, db *sql.DB, userID int) (*User, error) {
    query := "SELECT id, name FROM users WHERE id = ?"

    var user User
    err := db.QueryRowContext(ctx, query, userID).Scan(&user.ID, &user.Name)
    if err != nil {
        return nil, err
    }

    return &user, nil
}
```

---

## 結語

Context 是 Go 並發編程的重要工具，掌握它對於編寫可靠的並發程序至關重要。合理使用 Context 可以優雅地處理超時、取消和請求範圍數據傳遞。

**下一步學習建議：**
1. 實踐使用 Context 編寫 HTTP 服務
2. 學習 Context 在數據庫操作中的應用
3. 探索 gRPC 等框架中 Context 的使用
4. 深入理解 Context 的內部實現

祝學習愉快！
