# Chapter 21: defer、panic 與 recover

## 概述

defer、panic 和 recover 是 Go 中處理資源清理和異常情況的機制。defer 用於延遲執行，panic 用於報告無法恢復的錯誤，recover 用於捕獲 panic。這三個特性組合起來提供了類似於 try/catch/finally 的功能，但更加靈活。

## Node.js 與 Go 的對比

| 特性 | Node.js | Go |
|------|---------|-----|
| 資源清理 | try/finally | defer |
| 拋出異常 | throw | panic |
| 捕獲異常 | catch | recover |
| 執行時機 | finally 在 return 前 | defer 在 return 後 |
| 多個清理 | 嵌套 try/finally | 多個 defer |
| 恢復 | catch 塊 | recover() |

## 詳細概念解釋

### 1. defer

defer 語句將函數調用推遲到包含它的函數返回之後執行：

- **執行時機**：函數返回前執行（在 return 語句之後）
- **執行順序**：後進先出（LIFO），類似棧
- **參數求值**：defer 語句出現時立即求值參數

### 2. panic

panic 用於報告嚴重錯誤，會中止當前函數的執行：

- 觸發所有已註冊的 defer
- 向調用棧上傳播
- 如果沒有被 recover 捕獲，程序崩潰

### 3. recover

recover 用於捕獲 panic，只能在 defer 函數中使用：

- 返回 panic 傳遞的值
- 使程序恢復正常執行
- 只能捕獲同一 goroutine 的 panic

## 代碼示例

### 示例 1: defer 基本用法

**Go:**
```go
package main

import "fmt"

func simpleDefer() {
    defer fmt.Println("1. 第一個 defer")
    defer fmt.Println("2. 第二個 defer")
    defer fmt.Println("3. 第三個 defer")

    fmt.Println("函數體")
}

func deferWithArgs() {
    x := 10

    // defer 語句的參數立即求值
    defer fmt.Println("defer 中的 x:", x)

    x = 20
    fmt.Println("修改後的 x:", x)
}

func deferInLoop() {
    for i := 0; i < 5; i++ {
        defer fmt.Println("循環中 defer:", i)
    }
    fmt.Println("循環結束")
}

func main() {
    fmt.Println("=== Simple Defer ===")
    simpleDefer()
    // 輸出順序：
    // 函數體
    // 3. 第三個 defer
    // 2. 第二個 defer
    // 1. 第一個 defer

    fmt.Println("\n=== Defer With Args ===")
    deferWithArgs()
    // 輸出：
    // 修改後的 x: 20
    // defer 中的 x: 10

    fmt.Println("\n=== Defer In Loop ===")
    deferInLoop()
    // 輸出：
    // 循環結束
    // 循環中 defer: 4
    // 循環中 defer: 3
    // 循環中 defer: 2
    // 循環中 defer: 1
    // 循環中 defer: 0
}
```

**Node.js 對比:**
```javascript
// JavaScript 沒有直接的 defer，但可以用 try/finally

function simpleFinally() {
    try {
        console.log("函數體");
    } finally {
        console.log("1. finally 塊");
    }
}

// 模擬多個 defer（使用數組）
function multipleDefer() {
    const defers = [];

    defers.push(() => console.log("1. 第一個 defer"));
    defers.push(() => console.log("2. 第二個 defer"));
    defers.push(() => console.log("3. 第三個 defer"));

    try {
        console.log("函數體");
    } finally {
        // 反向執行（LIFO）
        while (defers.length > 0) {
            const fn = defers.pop();
            fn();
        }
    }
}

function deferWithArgs() {
    let x = 10;

    // 立即捕獲 x 的值
    const deferredFn = ((capturedX) => {
        return () => console.log("defer 中的 x:", capturedX);
    })(x);

    try {
        x = 20;
        console.log("修改後的 x:", x);
    } finally {
        deferredFn();
    }
}

function deferInLoop() {
    const defers = [];

    for (let i = 0; i < 5; i++) {
        // 需要閉包捕獲 i
        defers.push(((capturedI) => {
            return () => console.log("循環中 defer:", capturedI);
        })(i));
    }

    try {
        console.log("循環結束");
    } finally {
        while (defers.length > 0) {
            defers.pop()();
        }
    }
}

console.log("=== Simple Finally ===");
simpleFinally();

console.log("\n=== Multiple Defer ===");
multipleDefer();

console.log("\n=== Defer With Args ===");
deferWithArgs();

console.log("\n=== Defer In Loop ===");
deferInLoop();
```

### 示例 2: defer 用於資源清理

**Go:**
```go
package main

import (
    "fmt"
    "os"
)

// 文件操作示例
func readFile(filename string) error {
    file, err := os.Open(filename)
    if err != nil {
        return err
    }
    defer file.Close()  // 確保文件被關閉

    // 讀取文件...
    fmt.Println("讀取文件:", filename)
    return nil
}

// 鎖的使用示例
type Resource struct {
    data string
}

func (r *Resource) Lock() {
    fmt.Println("獲取鎖")
}

func (r *Resource) Unlock() {
    fmt.Println("釋放鎖")
}

func useResource(r *Resource) {
    r.Lock()
    defer r.Unlock()  // 確保鎖被釋放

    // 使用資源...
    fmt.Println("使用資源")
}

// 多個資源清理
func multipleResources() {
    fmt.Println("打開資源 1")
    defer fmt.Println("關閉資源 1")

    fmt.Println("打開資源 2")
    defer fmt.Println("關閉資源 2")

    fmt.Println("打開資源 3")
    defer fmt.Println("關閉資源 3")

    fmt.Println("執行操作")
}

func main() {
    fmt.Println("=== File Example ===")
    readFile("test.txt")

    fmt.Println("\n=== Lock Example ===")
    r := &Resource{data: "important"}
    useResource(r)

    fmt.Println("\n=== Multiple Resources ===")
    multipleResources()
}
```

**Node.js 對比:**
```javascript
const fs = require('fs').promises;

// 文件操作（async/await + finally）
async function readFile(filename) {
    let file;
    try {
        file = await fs.open(filename, 'r');
        console.log("讀取文件:", filename);
        // 讀取文件...
    } finally {
        if (file) {
            await file.close();
        }
    }
}

// 鎖的使用
class Resource {
    constructor(data) {
        this.data = data;
    }

    lock() {
        console.log("獲取鎖");
    }

    unlock() {
        console.log("釋放鎖");
    }
}

function useResource(r) {
    r.lock();
    try {
        console.log("使用資源");
    } finally {
        r.unlock();
    }
}

// 多個資源清理
function multipleResources() {
    const cleanups = [];

    try {
        console.log("打開資源 1");
        cleanups.push(() => console.log("關閉資源 1"));

        console.log("打開資源 2");
        cleanups.push(() => console.log("關閉資源 2"));

        console.log("打開資源 3");
        cleanups.push(() => console.log("關閉資源 3"));

        console.log("執行操作");
    } finally {
        while (cleanups.length > 0) {
            cleanups.pop()();
        }
    }
}

console.log("=== File Example ===");
readFile("test.txt").catch(err => console.log("Error:", err.message));

console.log("\n=== Lock Example ===");
const r = new Resource("important");
useResource(r);

console.log("\n=== Multiple Resources ===");
multipleResources();
```

### 示例 3: panic 和 recover 基礎

**Go:**
```go
package main

import "fmt"

func willPanic() {
    panic("something went wrong!")
}

func safeFunction() {
    defer func() {
        if r := recover(); r != nil {
            fmt.Println("Recovered from panic:", r)
        }
    }()

    fmt.Println("開始執行")
    willPanic()
    fmt.Println("這行不會執行")
}

func panicWithDefer() {
    defer fmt.Println("defer 1")
    defer fmt.Println("defer 2")

    panic("panic occurred")

    fmt.Println("這行不會執行")
}

func recoverPanic() {
    defer func() {
        if r := recover(); r != nil {
            fmt.Println("Recovered:", r)
        }
    }()

    panicWithDefer()
}

func main() {
    fmt.Println("=== Safe Function ===")
    safeFunction()
    fmt.Println("程序繼續運行")

    fmt.Println("\n=== Recover Panic ===")
    recoverPanic()
    fmt.Println("程序繼續運行")
}
```

**Node.js 對比:**
```javascript
function willThrow() {
    throw new Error("something went wrong!");
}

function safeFunction() {
    try {
        console.log("開始執行");
        willThrow();
        console.log("這行不會執行");
    } catch (err) {
        console.log("Caught error:", err.message);
    }
}

function throwWithFinally() {
    try {
        throw new Error("error occurred");
        console.log("這行不會執行");
    } finally {
        console.log("defer 2");
        console.log("defer 1");
    }
}

function recoverError() {
    try {
        throwWithFinally();
    } catch (err) {
        console.log("Recovered:", err.message);
    }
}

console.log("=== Safe Function ===");
safeFunction();
console.log("程序繼續運行");

console.log("\n=== Recover Error ===");
recoverError();
console.log("程序繼續運行");
```

### 示例 4: 實際應用場景

**Go:**
```go
package main

import (
    "fmt"
    "time"
)

// 場景 1: HTTP 服務器路由處理
func handleRequest(path string) {
    defer func() {
        if r := recover(); r != nil {
            fmt.Printf("Panic in handler for %s: %v\n", path, r)
            // 記錄錯誤日誌
            // 返回 500 錯誤給客戶端
        }
    }()

    fmt.Printf("Handling request: %s\n", path)

    if path == "/panic" {
        panic("intentional panic")
    }

    fmt.Println("Request handled successfully")
}

// 場景 2: 性能測試
func measureTime(name string) func() {
    start := time.Now()

    return func() {
        elapsed := time.Since(start)
        fmt.Printf("%s took %s\n", name, elapsed)
    }
}

func slowOperation() {
    defer measureTime("slowOperation")()

    time.Sleep(100 * time.Millisecond)
    fmt.Println("Operation completed")
}

// 場景 3: 事務管理
type Transaction struct {
    committed bool
}

func (t *Transaction) Commit() {
    fmt.Println("提交事務")
    t.committed = true
}

func (t *Transaction) Rollback() {
    fmt.Println("回滾事務")
}

func executeTransaction() error {
    tx := &Transaction{}

    defer func() {
        if !tx.committed {
            tx.Rollback()
        }
    }()

    // 執行操作...
    fmt.Println("執行事務操作")

    // 如果出錯，return 前會自動回滾
    // return errors.New("operation failed")

    tx.Commit()
    return nil
}

// 場景 4: 改變返回值
func namedReturn() (result int) {
    defer func() {
        result++  // 修改返回值
    }()

    return 10
}

func anonymousReturn() int {
    result := 10

    defer func() {
        result++  // 不會影響返回值
    }()

    return result
}

func main() {
    fmt.Println("=== HTTP Handler ===")
    handleRequest("/home")
    handleRequest("/panic")
    handleRequest("/about")

    fmt.Println("\n=== Performance Measurement ===")
    slowOperation()

    fmt.Println("\n=== Transaction ===")
    executeTransaction()

    fmt.Println("\n=== Return Values ===")
    fmt.Println("Named return:", namedReturn())      // 11
    fmt.Println("Anonymous return:", anonymousReturn()) // 10
}
```

**Node.js 對比:**
```javascript
// 場景 1: HTTP 路由處理
function handleRequest(path) {
    try {
        console.log(`Handling request: ${path}`);

        if (path === '/panic') {
            throw new Error("intentional error");
        }

        console.log("Request handled successfully");
    } catch (err) {
        console.log(`Error in handler for ${path}:`, err.message);
        // 記錄錯誤日誌
        // 返回 500 錯誤給客戶端
    }
}

// 場景 2: 性能測試
function measureTime(name, fn) {
    const start = Date.now();
    try {
        return fn();
    } finally {
        const elapsed = Date.now() - start;
        console.log(`${name} took ${elapsed}ms`);
    }
}

function slowOperation() {
    return measureTime("slowOperation", () => {
        // 模擬慢操作
        const start = Date.now();
        while (Date.now() - start < 100) {}
        console.log("Operation completed");
    });
}

// 場景 3: 事務管理
class Transaction {
    constructor() {
        this.committed = false;
    }

    commit() {
        console.log("提交事務");
        this.committed = true;
    }

    rollback() {
        console.log("回滾事務");
    }
}

function executeTransaction() {
    const tx = new Transaction();

    try {
        console.log("執行事務操作");

        // 如果出錯會跳到 catch
        // throw new Error("operation failed");

        tx.commit();
    } catch (err) {
        if (!tx.committed) {
            tx.rollback();
        }
        throw err;
    } finally {
        if (!tx.committed) {
            tx.rollback();
        }
    }
}

// 場景 4: 改變返回值
function modifyReturn() {
    let result = 10;

    try {
        return result;
    } finally {
        result++;  // 不會影響返回值
    }
}

// 使用示例
console.log("=== HTTP Handler ===");
handleRequest("/home");
handleRequest("/panic");
handleRequest("/about");

console.log("\n=== Performance Measurement ===");
slowOperation();

console.log("\n=== Transaction ===");
try {
    executeTransaction();
} catch (err) {
    console.log("Transaction failed:", err.message);
}

console.log("\n=== Return Values ===");
console.log("Return value:", modifyReturn());  // 10
```

### 示例 5: defer 的陷阱

**Go:**
```go
package main

import "fmt"

// 陷阱 1: 循環中的 defer
func deferInLoopWrong() {
    for i := 0; i < 5; i++ {
        defer fmt.Println("defer:", i)  // 所有 defer 都會累積
    }
    fmt.Println("函數結束")
    // 可能導致內存問題
}

func deferInLoopRight() {
    for i := 0; i < 5; i++ {
        func(n int) {
            defer fmt.Println("defer:", n)
            // 處理邏輯
        }(i)  // 立即執行
    }
    fmt.Println("函數結束")
}

// 陷阱 2: defer 與閉包
func deferClosure() {
    x := 1

    defer func() {
        fmt.Println("defer 閉包 x:", x)  // 會使用最終的 x 值
    }()

    x = 2
    fmt.Println("當前 x:", x)
}

func deferClosureFixed() {
    x := 1

    defer func(capturedX int) {
        fmt.Println("defer 捕獲 x:", capturedX)
    }(x)  // 立即求值參數

    x = 2
    fmt.Println("當前 x:", x)
}

// 陷阱 3: defer 與返回值
func returnVsDefer() (x int) {
    defer func() {
        x++  // 會修改返回值
    }()

    return 5  // 實際返回 6
}

// 陷阱 4: panic 在 defer 中
func panicInDefer() {
    defer func() {
        fmt.Println("defer 1")
        panic("panic in defer 1")
    }()

    defer func() {
        if r := recover(); r != nil {
            fmt.Println("Recovered in defer 2:", r)
        }
        fmt.Println("defer 2")
    }()

    panic("original panic")
}

func main() {
    fmt.Println("=== Defer in Loop Wrong ===")
    deferInLoopWrong()

    fmt.Println("\n=== Defer in Loop Right ===")
    deferInLoopRight()

    fmt.Println("\n=== Defer Closure ===")
    deferClosure()

    fmt.Println("\n=== Defer Closure Fixed ===")
    deferClosureFixed()

    fmt.Println("\n=== Return vs Defer ===")
    fmt.Println("Result:", returnVsDefer())

    fmt.Println("\n=== Panic in Defer ===")
    func() {
        defer func() {
            if r := recover(); r != nil {
                fmt.Println("Outer recover:", r)
            }
        }()
        panicInDefer()
    }()
}
```

**Node.js 對比:**
```javascript
// JavaScript 中的類似問題

// 問題 1: 循環中的清理
function cleanupInLoopWrong() {
    const cleanups = [];

    for (let i = 0; i < 5; i++) {
        cleanups.push(() => console.log("cleanup:", i));
    }

    try {
        console.log("函數結束");
    } finally {
        while (cleanups.length > 0) {
            cleanups.pop()();
        }
    }
}

function cleanupInLoopRight() {
    for (let i = 0; i < 5; i++) {
        (function(n) {
            try {
                // 處理邏輯
            } finally {
                console.log("cleanup:", n);
            }
        })(i);
    }
    console.log("函數結束");
}

// 問題 2: 閉包捕獲
function closureCaptureWrong() {
    let x = 1;
    const cleanup = () => console.log("cleanup x:", x);

    try {
        x = 2;
        console.log("當前 x:", x);
    } finally {
        cleanup();  // 使用最終的 x 值
    }
}

function closureCaptureRight() {
    let x = 1;
    const cleanup = ((capturedX) => {
        return () => console.log("cleanup x:", capturedX);
    })(x);

    try {
        x = 2;
        console.log("當前 x:", x);
    } finally {
        cleanup();
    }
}

console.log("=== Cleanup in Loop Wrong ===");
cleanupInLoopWrong();

console.log("\n=== Cleanup in Loop Right ===");
cleanupInLoopRight();

console.log("\n=== Closure Capture Wrong ===");
closureCaptureWrong();

console.log("\n=== Closure Capture Right ===");
closureCaptureRight();
```

## 重點總結

### defer vs finally

| 特性 | Go defer | JS finally |
|------|----------|------------|
| 執行時機 | return 後 | return 前 |
| 多個清理 | 多個 defer | 嵌套 try/finally |
| 參數求值 | 立即求值 | 需要閉包 |
| 修改返回值 | 可以（命名返回） | 不可以 |

### panic vs throw

| 特性 | Go panic | JS throw |
|------|----------|----------|
| 用途 | 嚴重錯誤 | 所有異常 |
| 恢復 | recover | catch |
| 傳播 | 向上傳播 | 向上傳播 |
| 常用度 | 較少使用 | 常用 |

### 最佳實踐

1. **defer 用途**：
   - 資源清理（文件、鎖、連接）
   - 解鎖互斥鎖
   - 記錄執行時間
   - 恢復 panic

2. **避免過度使用 panic**：
   - 優先使用 error 返回值
   - panic 用於真正無法恢復的錯誤
   - 庫代碼應該返回 error 而非 panic

3. **recover 注意事項**：
   - 只在 defer 中調用
   - 只捕獲同一 goroutine 的 panic
   - 捕獲後記錄日誌

4. **defer 陷阱**：
   - 注意循環中的 defer
   - 小心閉包捕獲
   - 理解參數求值時機

## 練習題

### 練習 1: 資源池管理
實現一個資源池，使用 defer 確保資源被正確歸還。

### 練習 2: 安全的 HTTP 處理器
創建一個 HTTP 處理器包裝函數，使用 recover 捕獲 panic。

### 練習 3: 執行時間測量
實現一個通用的執行時間測量函數。

### 練習 4: 事務管理器
實現數據庫事務管理，自動處理提交和回滾。

### 練習 5: 優雅的錯誤處理
設計一個錯誤處理系統，結合 defer、panic 和 error。

### 答案提示

**練習 2:**
```go
func SafeHandler(handler func()) {
    defer func() {
        if r := recover(); r != nil {
            log.Printf("Panic recovered: %v", r)
            // 返回 500 錯誤
        }
    }()

    handler()
}

func HandleRequest(w http.ResponseWriter, r *http.Request) {
    SafeHandler(func() {
        // 處理請求
        // 即使 panic 也不會崩潰整個服務器
    })
}
```

**練習 3:**
```go
func Measure(name string) func() {
    start := time.Now()
    return func() {
        fmt.Printf("%s: %v\n", name, time.Since(start))
    }
}

func SlowFunc() {
    defer Measure("SlowFunc")()
    // 執行操作...
}
```
