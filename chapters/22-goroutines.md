# Chapter 22: Goroutines 並發模型

## 概述

Goroutine 是 Go 最核心的特性之一，它是 Go 實現並發的基礎。Goroutine 是輕量級的執行線程，由 Go 運行時管理，而不是操作系統線程。與 Node.js 的事件循環和異步編程不同，Go 使用 goroutines 實現真正的並發執行。

## Node.js 與 Go 的對比

| 特性 | Node.js | Go |
|------|---------|-----|
| 並發模型 | 事件循環 + 異步 | Goroutines + Channels |
| 線程 | 單線程（主線程） | 多線程（M:N 調度） |
| 異步操作 | async/await, Promise | Goroutines |
| 開銷 | 輕量（回調） | 極輕量（2KB 起始棧） |
| 阻塞 | 不阻塞事件循環 | Goroutine 可阻塞 |
| CPU 密集 | 阻塞事件循環 | 充分利用多核 |
| 創建 | `async function` | `go function()` |

## 詳細概念解釋

### 1. Goroutine vs Thread

**操作系統線程：**
- 重量級（MB 級棧空間）
- 切換成本高
- 數量有限

**Goroutine：**
- 輕量級（2KB 起始棧，可動態增長）
- 切換成本低（Go 調度器管理）
- 可以創建數十萬個

### 2. Go 調度器（M:N 調度）

```
G (Goroutine): 用戶級線程
M (Machine):   操作系統線程
P (Processor): 處理器（邏輯 CPU）

多個 Goroutines (G) 映射到少量 OS 線程 (M)
```

### 3. 並發 vs 並行

- **並發（Concurrency）**: 同時處理多個任務（邏輯概念）
- **並行（Parallelism）**: 同時執行多個任務（物理概念）

## 代碼示例

### 示例 1: 創建和使用 Goroutine

**Go:**
```go
package main

import (
    "fmt"
    "time"
)

func sayHello(name string) {
    for i := 0; i < 3; i++ {
        fmt.Printf("Hello from %s (iteration %d)\n", name, i)
        time.Sleep(100 * time.Millisecond)
    }
}

func main() {
    // 普通函數調用（同步）
    fmt.Println("=== Synchronous ===")
    sayHello("Sync")

    // 使用 goroutine（異步）
    fmt.Println("\n=== Asynchronous ===")
    go sayHello("Async-1")
    go sayHello("Async-2")

    // 等待 goroutines 完成
    time.Sleep(500 * time.Millisecond)

    // 匿名函數 goroutine
    go func() {
        fmt.Println("Anonymous goroutine")
    }()

    // 帶參數的匿名 goroutine
    message := "Hello from closure"
    go func(msg string) {
        fmt.Println(msg)
    }(message)

    time.Sleep(100 * time.Millisecond)
    fmt.Println("Main function ending")
}
```

**Node.js 對比:**
```javascript
// Node.js 使用 async/await 和 Promise

function sayHello(name) {
    return new Promise((resolve) => {
        let count = 0;
        const interval = setInterval(() => {
            console.log(`Hello from ${name} (iteration ${count})`);
            count++;
            if (count >= 3) {
                clearInterval(interval);
                resolve();
            }
        }, 100);
    });
}

async function main() {
    // 同步調用
    console.log("=== Synchronous ===");
    await sayHello("Sync");

    // 異步調用（並發）
    console.log("\n=== Asynchronous ===");
    Promise.all([
        sayHello("Async-1"),
        sayHello("Async-2")
    ]).then(() => {
        console.log("Both promises completed");
    });

    // 使用 async 函數
    (async () => {
        console.log("Anonymous async function");
    })();

    // 帶參數的異步函數
    const message = "Hello from closure";
    (async (msg) => {
        console.log(msg);
    })(message);

    // 等待
    await new Promise(resolve => setTimeout(resolve, 500));
    console.log("Main function ending");
}

main();
```

### 示例 2: Goroutine 與閉包

**Go:**
```go
package main

import (
    "fmt"
    "time"
)

func closureProblem() {
    fmt.Println("=== Closure Problem ===")

    // 錯誤示例：所有 goroutine 共享變量
    for i := 0; i < 5; i++ {
        go func() {
            fmt.Println("Wrong:", i)  // 可能都打印 5
        }()
    }

    time.Sleep(100 * time.Millisecond)
}

func closureSolution1() {
    fmt.Println("\n=== Solution 1: Pass as Parameter ===")

    for i := 0; i < 5; i++ {
        go func(n int) {
            fmt.Println("Correct:", n)
        }(i)  // 傳遞參數
    }

    time.Sleep(100 * time.Millisecond)
}

func closureSolution2() {
    fmt.Println("\n=== Solution 2: Local Variable ===")

    for i := 0; i < 5; i++ {
        i := i  // 創建局部變量
        go func() {
            fmt.Println("Correct:", i)
        }()
    }

    time.Sleep(100 * time.Millisecond)
}

func main() {
    closureProblem()
    closureSolution1()
    closureSolution2()
}
```

**Node.js 對比:**
```javascript
async function closureProblem() {
    console.log("=== Closure Problem ===");

    // JavaScript 中使用 var 會有類似問題
    for (var i = 0; i < 5; i++) {
        setTimeout(() => {
            console.log("Wrong:", i);  // 都打印 5
        }, 10);
    }

    await new Promise(resolve => setTimeout(resolve, 100));
}

async function closureSolution1() {
    console.log("\n=== Solution 1: let (Block Scope) ===");

    // 使用 let 自動解決
    for (let i = 0; i < 5; i++) {
        setTimeout(() => {
            console.log("Correct:", i);
        }, 10);
    }

    await new Promise(resolve => setTimeout(resolve, 100));
}

async function closureSolution2() {
    console.log("\n=== Solution 2: IIFE ===");

    for (var i = 0; i < 5; i++) {
        ((n) => {
            setTimeout(() => {
                console.log("Correct:", n);
            }, 10);
        })(i);
    }

    await new Promise(resolve => setTimeout(resolve, 100));
}

async function main() {
    await closureProblem();
    await closureSolution1();
    await closureSolution2();
}

main();
```

### 示例 3: Goroutine 並發模式

**Go:**
```go
package main

import (
    "fmt"
    "sync"
    "time"
)

// 模式 1: 簡單並發任務
func parallelTasks() {
    fmt.Println("=== Parallel Tasks ===")

    var wg sync.WaitGroup

    tasks := []string{"Task 1", "Task 2", "Task 3", "Task 4"}

    for _, task := range tasks {
        wg.Add(1)
        go func(name string) {
            defer wg.Done()
            fmt.Printf("Starting %s\n", name)
            time.Sleep(100 * time.Millisecond)
            fmt.Printf("Completed %s\n", name)
        }(task)
    }

    wg.Wait()
    fmt.Println("All tasks completed")
}

// 模式 2: Worker Pool
func workerPool() {
    fmt.Println("\n=== Worker Pool ===")

    jobs := make(chan int, 10)
    results := make(chan int, 10)

    // 啟動 3 個 worker
    for w := 1; w <= 3; w++ {
        go func(id int) {
            for job := range jobs {
                fmt.Printf("Worker %d processing job %d\n", id, job)
                time.Sleep(100 * time.Millisecond)
                results <- job * 2
            }
        }(w)
    }

    // 發送 5 個任務
    for j := 1; j <= 5; j++ {
        jobs <- j
    }
    close(jobs)

    // 收集結果
    for r := 1; r <= 5; r++ {
        result := <-results
        fmt.Printf("Result: %d\n", result)
    }
}

// 模式 3: Fan-out, Fan-in
func fanOutFanIn() {
    fmt.Println("\n=== Fan-out, Fan-in ===")

    // 輸入
    input := make(chan int, 5)
    for i := 1; i <= 5; i++ {
        input <- i
    }
    close(input)

    // Fan-out: 啟動多個 worker
    channels := make([]<-chan int, 3)
    for i := 0; i < 3; i++ {
        channels[i] = worker(input)
    }

    // Fan-in: 合併結果
    result := merge(channels...)

    // 打印結果
    for val := range result {
        fmt.Println("Result:", val)
    }
}

func worker(input <-chan int) <-chan int {
    output := make(chan int)
    go func() {
        defer close(output)
        for val := range input {
            output <- val * val
        }
    }()
    return output
}

func merge(channels ...<-chan int) <-chan int {
    var wg sync.WaitGroup
    output := make(chan int)

    multiplex := func(c <-chan int) {
        defer wg.Done()
        for val := range c {
            output <- val
        }
    }

    wg.Add(len(channels))
    for _, c := range channels {
        go multiplex(c)
    }

    go func() {
        wg.Wait()
        close(output)
    }()

    return output
}

func main() {
    parallelTasks()
    workerPool()
    fanOutFanIn()
}
```

**Node.js 對比:**
```javascript
// 模式 1: 並行任務
async function parallelTasks() {
    console.log("=== Parallel Tasks ===");

    const tasks = ["Task 1", "Task 2", "Task 3", "Task 4"];

    const promises = tasks.map(task => {
        return new Promise(async (resolve) => {
            console.log(`Starting ${task}`);
            await new Promise(r => setTimeout(r, 100));
            console.log(`Completed ${task}`);
            resolve();
        });
    });

    await Promise.all(promises);
    console.log("All tasks completed");
}

// 模式 2: Worker Pool（使用 async 隊列）
async function workerPool() {
    console.log("\n=== Worker Pool ===");

    const jobs = [1, 2, 3, 4, 5];
    const results = [];
    const workerCount = 3;

    async function worker(id, jobQueue) {
        while (jobQueue.length > 0) {
            const job = jobQueue.shift();
            if (job === undefined) break;

            console.log(`Worker ${id} processing job ${job}`);
            await new Promise(resolve => setTimeout(resolve, 100));
            results.push(job * 2);
        }
    }

    const workers = [];
    for (let i = 1; i <= workerCount; i++) {
        workers.push(worker(i, jobs));
    }

    await Promise.all(workers);

    results.forEach(result => {
        console.log(`Result: ${result}`);
    });
}

// 模式 3: Fan-out, Fan-in
async function fanOutFanIn() {
    console.log("\n=== Fan-out, Fan-in ===");

    const input = [1, 2, 3, 4, 5];

    // Fan-out: 啟動多個 worker
    const workerCount = 3;
    const workersResults = [];

    for (let i = 0; i < workerCount; i++) {
        const promise = Promise.all(
            input.map(async (val) => val * val)
        );
        workersResults.push(promise);
    }

    // Fan-in: 合併結果
    const allResults = await Promise.all(workersResults);
    const results = allResults.flat();

    results.forEach(val => {
        console.log("Result:", val);
    });
}

async function main() {
    await parallelTasks();
    await workerPool();
    await fanOutFanIn();
}

main();
```

### 示例 4: Goroutine 洩漏問題

**Go:**
```go
package main

import (
    "fmt"
    "time"
)

// 問題：Goroutine 洩漏
func goroutineLeakProblem() {
    ch := make(chan int)

    // 這個 goroutine 會永遠阻塞，因為沒有人讀取 channel
    go func() {
        ch <- 1
        fmt.Println("This will never print")
    }()

    // main 函數結束，但 goroutine 仍在運行
}

// 解決方案 1: 使用帶緩衝的 channel
func goroutineLeakSolution1() {
    ch := make(chan int, 1)  // 帶緩衝

    go func() {
        ch <- 1
        fmt.Println("Sent to buffered channel")
    }()

    time.Sleep(100 * time.Millisecond)
}

// 解決方案 2: 使用 context 取消
func goroutineLeakSolution2() {
    done := make(chan bool)

    go func() {
        select {
        case <-done:
            fmt.Println("Goroutine cancelled")
            return
        case <-time.After(5 * time.Second):
            fmt.Println("Work completed")
        }
    }()

    time.Sleep(100 * time.Millisecond)
    close(done)  // 取消 goroutine
    time.Sleep(100 * time.Millisecond)
}

// 問題：未等待 goroutine 完成
func notWaitingProblem() {
    for i := 0; i < 5; i++ {
        go func(n int) {
            fmt.Println("Number:", n)
        }(i)
    }
    // main 可能在 goroutines 執行前就結束了
}

// 解決方案：使用 WaitGroup
func notWaitingSolution() {
    var wg sync.WaitGroup

    for i := 0; i < 5; i++ {
        wg.Add(1)
        go func(n int) {
            defer wg.Done()
            fmt.Println("Number:", n)
        }(i)
    }

    wg.Wait()  // 等待所有 goroutine 完成
}

func main() {
    fmt.Println("=== Leak Solution 1 ===")
    goroutineLeakSolution1()

    fmt.Println("\n=== Leak Solution 2 ===")
    goroutineLeakSolution2()

    fmt.Println("\n=== Wait Solution ===")
    notWaitingSolution()
}
```

**Node.js 對比:**
```javascript
// Node.js 中的類似問題

// 問題：Promise 未處理
function promiseLeakProblem() {
    return new Promise((resolve) => {
        // 永遠不 resolve
        setTimeout(() => {
            console.log("This will eventually print");
        }, 1000);
    });
    // 調用者如果 await 會永遠等待
}

// 解決方案：使用超時
async function promiseLeakSolution() {
    const timeoutPromise = new Promise((_, reject) =>
        setTimeout(() => reject(new Error("Timeout")), 100)
    );

    const workPromise = new Promise((resolve) =>
        setTimeout(() => resolve("Done"), 5000)
    );

    try {
        const result = await Promise.race([workPromise, timeoutPromise]);
        console.log(result);
    } catch (err) {
        console.log("Cancelled:", err.message);
    }
}

// 問題：不等待異步操作
async function notWaitingProblem() {
    for (let i = 0; i < 5; i++) {
        // 沒有 await，可能不會執行
        (async () => {
            console.log("Number:", i);
        })();
    }
}

// 解決方案：使用 Promise.all
async function notWaitingSolution() {
    const promises = [];

    for (let i = 0; i < 5; i++) {
        promises.push(
            (async (n) => {
                console.log("Number:", n);
            })(i)
        );
    }

    await Promise.all(promises);
}

async function main() {
    console.log("=== Leak Solution ===");
    await promiseLeakSolution();

    console.log("\n=== Wait Solution ===");
    await notWaitingSolution();
}

main();
```

### 示例 5: CPU 密集型任務

**Go:**
```go
package main

import (
    "fmt"
    "runtime"
    "sync"
    "time"
)

// CPU 密集型計算
func fibonacci(n int) int {
    if n <= 1 {
        return n
    }
    return fibonacci(n-1) + fibonacci(n-2)
}

// 串行計算
func serialCalculation() time.Duration {
    start := time.Now()

    for i := 0; i < 10; i++ {
        _ = fibonacci(30)
    }

    return time.Since(start)
}

// 並行計算
func parallelCalculation() time.Duration {
    start := time.Now()
    var wg sync.WaitGroup

    // 設置使用的 CPU 核心數
    runtime.GOMAXPROCS(runtime.NumCPU())

    for i := 0; i < 10; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            _ = fibonacci(30)
        }()
    }

    wg.Wait()
    return time.Since(start)
}

func main() {
    fmt.Println("CPU 核心數:", runtime.NumCPU())

    serialTime := serialCalculation()
    fmt.Printf("串行計算耗時: %v\n", serialTime)

    parallelTime := parallelCalculation()
    fmt.Printf("並行計算耗時: %v\n", parallelTime)

    speedup := float64(serialTime) / float64(parallelTime)
    fmt.Printf("加速比: %.2fx\n", speedup)
}
```

**Node.js 對比:**
```javascript
const { Worker } = require('worker_threads');
const os = require('os');

// CPU 密集型計算
function fibonacci(n) {
    if (n <= 1) return n;
    return fibonacci(n - 1) + fibonacci(n - 2);
}

// 串行計算
async function serialCalculation() {
    const start = Date.now();

    for (let i = 0; i < 10; i++) {
        fibonacci(30);
    }

    return Date.now() - start;
}

// 並行計算（使用 Worker Threads）
async function parallelCalculation() {
    const start = Date.now();

    const workers = [];
    for (let i = 0; i < 10; i++) {
        workers.push(new Promise((resolve, reject) => {
            const worker = new Worker(`
                const { parentPort } = require('worker_threads');
                function fibonacci(n) {
                    if (n <= 1) return n;
                    return fibonacci(n - 1) + fibonacci(n - 2);
                }
                parentPort.postMessage(fibonacci(30));
            `, { eval: true });

            worker.on('message', resolve);
            worker.on('error', reject);
        }));
    }

    await Promise.all(workers);
    return Date.now() - start;
}

async function main() {
    console.log("CPU 核心數:", os.cpus().length);

    const serialTime = await serialCalculation();
    console.log(`串行計算耗時: ${serialTime}ms`);

    const parallelTime = await parallelCalculation();
    console.log(`並行計算耗時: ${parallelTime}ms`);

    const speedup = serialTime / parallelTime;
    console.log(`加速比: ${speedup.toFixed(2)}x`);
}

main();
```

## 重點總結

### Goroutines vs Node.js 異步

| 特性 | Goroutines | Node.js Async |
|------|-----------|---------------|
| 並發模型 | CSP（通信順序進程） | 事件循環 |
| 真正並行 | 是（多核） | 否（單線程） |
| CPU 密集 | 高效 | 阻塞事件循環 |
| 內存開銷 | 極低 (2KB) | 低（閉包） |
| 創建成本 | 極低 | 低 |
| 數量限制 | 幾乎無限 | 受內存限制 |

### 最佳實踐

1. **合理使用 Goroutines**：
   - I/O 密集型任務
   - 可並發執行的獨立任務
   - 不要為每個小操作都創建 goroutine

2. **避免 Goroutine 洩漏**：
   - 確保 goroutine 能正常退出
   - 使用 context 控制生命週期
   - 正確關閉 channel

3. **正確等待完成**：
   - 使用 sync.WaitGroup
   - 使用 channel 同步
   - 不要讓 main 過早退出

4. **控制並發數量**：
   - 使用 worker pool 模式
   - 使用帶緩衝的 channel 限流
   - 避免創建過多 goroutine

## 練習題

### 練習 1: 並發下載器
實現一個並發 URL 下載器，限制最大並發數。

### 練習 2: 超時處理
實現一個函數，在指定時間內執行任務，超時則取消。

### 練習 3: 生產者-消費者
實現生產者-消費者模式，多個生產者，多個消費者。

### 練習 4: 並發 Map-Reduce
實現一個並發的 Map-Reduce 框架。

### 練習 5: 請求合併
實現一個請求合併器，將短時間內的多個相同請求合併為一個。

### 答案提示

**練習 1:**
```go
func DownloadConcurrent(urls []string, maxConcurrent int) {
    var wg sync.WaitGroup
    semaphore := make(chan struct{}, maxConcurrent)

    for _, url := range urls {
        wg.Add(1)
        go func(u string) {
            defer wg.Done()

            semaphore <- struct{}{}        // 獲取令牌
            defer func() { <-semaphore }() // 釋放令牌

            download(u)
        }(url)
    }

    wg.Wait()
}
```
