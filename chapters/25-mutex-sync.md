# Chapter 25: 互斥鎖與同步 (Mutex & Sync)

## 概述

雖然 Go 鼓勵使用 channel 進行通信，但在某些場景下，使用互斥鎖（Mutex）和其他同步原語更加簡單高效。sync 包提供了多種同步工具，包括互斥鎖、讀寫鎖、等待組等，用於保護共享資源和協調 goroutines。

## Node.js 與 Go 的對比

| 特性 | Node.js | Go |
|------|---------|-----|
| 並發模型 | 單線程事件循環 | 多線程 |
| 鎖需求 | 較少（單線程） | 常見（多線程） |
| 互斥鎖 | 無原生支持 | `sync.Mutex` |
| 讀寫鎖 | 無原生支持 | `sync.RWMutex` |
| 等待組 | `Promise.all` | `sync.WaitGroup` |
| 原子操作 | 無原生支持 | `sync/atomic` |
| Once | 手動實現 | `sync.Once` |

## 詳細概念解釋

### 1. sync.Mutex（互斥鎖）

```go
var mu sync.Mutex

mu.Lock()    // 加鎖
// 臨界區
mu.Unlock()  // 解鎖
```

- **互斥**：同一時間只有一個 goroutine 可以持有鎖
- **阻塞**：其他 goroutine 會阻塞等待
- **非重入**：同一 goroutine 不能重複加鎖

### 2. sync.RWMutex（讀寫鎖）

```go
var rwmu sync.RWMutex

rwmu.RLock()    // 讀鎖
// 讀操作
rwmu.RUnlock()

rwmu.Lock()     // 寫鎖
// 寫操作
rwmu.Unlock()
```

- **多讀單寫**：多個 goroutine 可以同時讀，但寫是獨占的
- **適用場景**：讀多寫少

### 3. sync.WaitGroup（等待組）

```go
var wg sync.WaitGroup

wg.Add(1)    // 增加計數
go func() {
    defer wg.Done()  // 減少計數
    // 工作
}()

wg.Wait()    // 等待計數歸零
```

## 代碼示例

### 示例 1: Mutex 基礎

**Go:**
```go
package main

import (
    "fmt"
    "sync"
)

// 不使用鎖的問題示例
func withoutMutex() {
    fmt.Println("=== Without Mutex (Race Condition) ===")

    counter := 0
    var wg sync.WaitGroup

    for i := 0; i < 1000; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            counter++  // 競態條件
        }()
    }

    wg.Wait()
    fmt.Println("Counter:", counter)  // 結果不確定，小於 1000
}

// 使用鎖解決
func withMutex() {
    fmt.Println("\n=== With Mutex (Thread-Safe) ===")

    counter := 0
    var mu sync.Mutex
    var wg sync.WaitGroup

    for i := 0; i < 1000; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            mu.Lock()
            counter++
            mu.Unlock()
        }()
    }

    wg.Wait()
    fmt.Println("Counter:", counter)  // 正確結果：1000
}

// 使用 defer 確保解鎖
func withDeferUnlock() {
    fmt.Println("\n=== With Defer Unlock ===")

    var mu sync.Mutex
    data := make(map[string]int)

    increment := func(key string) {
        mu.Lock()
        defer mu.Unlock()  // 確保解鎖

        data[key]++
    }

    var wg sync.WaitGroup
    for i := 0; i < 100; i++ {
        wg.Add(1)
        go func(n int) {
            defer wg.Done()
            increment("key")
        }(i)
    }

    wg.Wait()
    fmt.Println("Data:", data)
}

func main() {
    withoutMutex()
    withMutex()
    withDeferUnlock()
}
```

**Node.js 對比:**
```javascript
// Node.js 單線程，通常不需要鎖
// 但在 Worker Threads 中可能需要

// 不使用鎖（單線程，沒有問題）
async function withoutMutex() {
    console.log("=== Without Mutex (No Race in Single Thread) ===");

    let counter = 0;
    const promises = [];

    for (let i = 0; i < 1000; i++) {
        promises.push((async () => {
            counter++;
        })());
    }

    await Promise.all(promises);
    console.log("Counter:", counter);  // 1000（單線程，順序執行）
}

// 使用 Worker Threads 會有競態問題
const { Worker } = require('worker_threads');

async function withWorkerThreads() {
    console.log("\n=== With Worker Threads ===");

    const sharedBuffer = new SharedArrayBuffer(4);
    const sharedArray = new Int32Array(sharedBuffer);

    const workers = [];
    for (let i = 0; i < 10; i++) {
        workers.push(new Promise((resolve, reject) => {
            const worker = new Worker(`
                const { parentPort, workerData } = require('worker_threads');
                const sharedArray = new Int32Array(workerData.buffer);

                // 沒有鎖保護的遞增（會有競態）
                for (let i = 0; i < 100; i++) {
                    sharedArray[0]++;
                }

                parentPort.postMessage('done');
            `, {
                eval: true,
                workerData: { buffer: sharedBuffer }
            });

            worker.on('message', resolve);
            worker.on('error', reject);
        }));
    }

    await Promise.all(workers);
    console.log("Counter:", sharedArray[0]);  // 可能小於 1000
}

// 使用 Atomics 實現原子操作
async function withAtomics() {
    console.log("\n=== With Atomics (Thread-Safe) ===");

    const sharedBuffer = new SharedArrayBuffer(4);
    const sharedArray = new Int32Array(sharedBuffer);

    const workers = [];
    for (let i = 0; i < 10; i++) {
        workers.push(new Promise((resolve, reject) => {
            const worker = new Worker(`
                const { parentPort, workerData } = require('worker_threads');
                const sharedArray = new Int32Array(workerData.buffer);

                // 使用原子操作
                for (let i = 0; i < 100; i++) {
                    Atomics.add(sharedArray, 0, 1);
                }

                parentPort.postMessage('done');
            `, {
                eval: true,
                workerData: { buffer: sharedBuffer }
            });

            worker.on('message', resolve);
            worker.on('error', reject);
        }));
    }

    await Promise.all(workers);
    console.log("Counter:", sharedArray[0]);  // 1000
}

async function main() {
    await withoutMutex();
    // await withWorkerThreads();  // 需要 Worker Threads 支持
    // await withAtomics();
}

main();
```

### 示例 2: RWMutex 讀寫鎖

**Go:**
```go
package main

import (
    "fmt"
    "sync"
    "time"
)

type SafeMap struct {
    mu   sync.RWMutex
    data map[string]int
}

func NewSafeMap() *SafeMap {
    return &SafeMap{
        data: make(map[string]int),
    }
}

func (sm *SafeMap) Get(key string) (int, bool) {
    sm.mu.RLock()
    defer sm.mu.RUnlock()

    val, ok := sm.data[key]
    return val, ok
}

func (sm *SafeMap) Set(key string, value int) {
    sm.mu.Lock()
    defer sm.mu.Unlock()

    sm.data[key] = value
}

func (sm *SafeMap) Len() int {
    sm.mu.RLock()
    defer sm.mu.RUnlock()

    return len(sm.data)
}

func rwMutexDemo() {
    fmt.Println("=== RWMutex Demo ===")

    safeMap := NewSafeMap()
    var wg sync.WaitGroup

    // 多個寫入者
    for i := 0; i < 5; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            for j := 0; j < 10; j++ {
                key := fmt.Sprintf("key%d", id)
                safeMap.Set(key, j)
                time.Sleep(10 * time.Millisecond)
            }
        }(i)
    }

    // 多個讀取者（可以並發讀取）
    for i := 0; i < 10; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            for j := 0; j < 20; j++ {
                key := fmt.Sprintf("key%d", id%5)
                if val, ok := safeMap.Get(key); ok {
                    fmt.Printf("Reader %d: %s = %d\n", id, key, val)
                }
                time.Sleep(5 * time.Millisecond)
            }
        }(i)
    }

    wg.Wait()
    fmt.Println("Final size:", safeMap.Len())
}

// 性能對比：Mutex vs RWMutex
func performanceComparison() {
    fmt.Println("\n=== Performance Comparison ===")

    data := make(map[string]int)
    for i := 0; i < 1000; i++ {
        data[fmt.Sprintf("key%d", i)] = i
    }

    // 使用 Mutex
    testMutex := func() time.Duration {
        var mu sync.Mutex
        var wg sync.WaitGroup
        start := time.Now()

        // 99% 讀操作
        for i := 0; i < 10000; i++ {
            wg.Add(1)
            go func(id int) {
                defer wg.Done()
                mu.Lock()
                _ = data[fmt.Sprintf("key%d", id%1000)]
                mu.Unlock()
            }(i)
        }

        wg.Wait()
        return time.Since(start)
    }

    // 使用 RWMutex
    testRWMutex := func() time.Duration {
        var rwmu sync.RWMutex
        var wg sync.WaitGroup
        start := time.Now()

        // 99% 讀操作
        for i := 0; i < 10000; i++ {
            wg.Add(1)
            go func(id int) {
                defer wg.Done()
                rwmu.RLock()
                _ = data[fmt.Sprintf("key%d", id%1000)]
                rwmu.RUnlock()
            }(i)
        }

        wg.Wait()
        return time.Since(start)
    }

    mutexTime := testMutex()
    rwMutexTime := testRWMutex()

    fmt.Printf("Mutex time: %v\n", mutexTime)
    fmt.Printf("RWMutex time: %v\n", rwMutexTime)
    fmt.Printf("RWMutex is %.2fx faster\n", float64(mutexTime)/float64(rwMutexTime))
}

func main() {
    rwMutexDemo()
    performanceComparison()
}
```

**Node.js 對比:**
```javascript
// JavaScript 單線程環境下的"線程安全" Map
class SafeMap {
    constructor() {
        this.data = new Map();
    }

    get(key) {
        return this.data.get(key);
    }

    set(key, value) {
        this.data.set(key, value);
    }

    get size() {
        return this.data.size;
    }
}

async function safeMapDemo() {
    console.log("=== SafeMap Demo ===");

    const safeMap = new SafeMap();

    // 多個寫入者（異步）
    const writers = [];
    for (let i = 0; i < 5; i++) {
        writers.push((async (id) => {
            for (let j = 0; j < 10; j++) {
                const key = `key${id}`;
                safeMap.set(key, j);
                await new Promise(resolve => setTimeout(resolve, 10));
            }
        })(i));
    }

    // 多個讀取者
    const readers = [];
    for (let i = 0; i < 10; i++) {
        readers.push((async (id) => {
            for (let j = 0; j < 20; j++) {
                const key = `key${id % 5}`;
                const val = safeMap.get(key);
                if (val !== undefined) {
                    console.log(`Reader ${id}: ${key} = ${val}`);
                }
                await new Promise(resolve => setTimeout(resolve, 5));
            }
        })(i));
    }

    await Promise.all([...writers, ...readers]);
    console.log("Final size:", safeMap.size);
}

safeMapDemo();
```

### 示例 3: WaitGroup 等待組

**Go:**
```go
package main

import (
    "fmt"
    "sync"
    "time"
)

func basicWaitGroup() {
    fmt.Println("=== Basic WaitGroup ===")

    var wg sync.WaitGroup

    for i := 1; i <= 5; i++ {
        wg.Add(1)

        go func(id int) {
            defer wg.Done()

            fmt.Printf("Worker %d starting\n", id)
            time.Sleep(time.Duration(id*100) * time.Millisecond)
            fmt.Printf("Worker %d done\n", id)
        }(i)
    }

    wg.Wait()
    fmt.Println("All workers done")
}

// 錯誤用法：在 goroutine 內部 Add
func wrongWaitGroup() {
    fmt.Println("\n=== Wrong WaitGroup Usage ===")

    var wg sync.WaitGroup

    for i := 1; i <= 5; i++ {
        go func(id int) {
            wg.Add(1)  // 錯誤：應該在啟動 goroutine 前 Add
            defer wg.Done()

            fmt.Printf("Worker %d\n", id)
        }(i)
    }

    wg.Wait()  // 可能在所有 Add 執行前就 Wait，導致過早結束
    fmt.Println("Done (might not wait for all)")
}

// 嵌套 WaitGroup
func nestedWaitGroup() {
    fmt.Println("\n=== Nested WaitGroup ===")

    var outerWg sync.WaitGroup

    for i := 1; i <= 3; i++ {
        outerWg.Add(1)

        go func(groupID int) {
            defer outerWg.Done()

            fmt.Printf("Group %d starting\n", groupID)

            var innerWg sync.WaitGroup
            for j := 1; j <= 3; j++ {
                innerWg.Add(1)

                go func(workerID int) {
                    defer innerWg.Done()

                    fmt.Printf("  Group %d, Worker %d\n", groupID, workerID)
                    time.Sleep(50 * time.Millisecond)
                }(j)
            }

            innerWg.Wait()
            fmt.Printf("Group %d done\n", groupID)
        }(i)
    }

    outerWg.Wait()
    fmt.Println("All groups done")
}

func main() {
    basicWaitGroup()
    wrongWaitGroup()
    nestedWaitGroup()
}
```

**Node.js 對比:**
```javascript
async function basicPromiseAll() {
    console.log("=== Basic Promise.all ===");

    const workers = [];

    for (let i = 1; i <= 5; i++) {
        workers.push((async (id) => {
            console.log(`Worker ${id} starting`);
            await new Promise(resolve => setTimeout(resolve, id * 100));
            console.log(`Worker ${id} done`);
        })(i));
    }

    await Promise.all(workers);
    console.log("All workers done");
}

async function nestedPromiseAll() {
    console.log("\n=== Nested Promise.all ===");

    const groups = [];

    for (let i = 1; i <= 3; i++) {
        groups.push((async (groupID) => {
            console.log(`Group ${groupID} starting`);

            const workers = [];
            for (let j = 1; j <= 3; j++) {
                workers.push((async (workerID) => {
                    console.log(`  Group ${groupID}, Worker ${workerID}`);
                    await new Promise(resolve => setTimeout(resolve, 50));
                })(j));
            }

            await Promise.all(workers);
            console.log(`Group ${groupID} done`);
        })(i));
    }

    await Promise.all(groups);
    console.log("All groups done");
}

async function main() {
    await basicPromiseAll();
    await nestedPromiseAll();
}

main();
```

### 示例 4: sync.Once 單次執行

**Go:**
```go
package main

import (
    "fmt"
    "sync"
)

var (
    instance *Singleton
    once     sync.Once
)

type Singleton struct {
    data string
}

func GetInstance() *Singleton {
    once.Do(func() {
        fmt.Println("Creating singleton instance...")
        instance = &Singleton{data: "singleton data"}
    })
    return instance
}

func onceDemo() {
    fmt.Println("=== Once Demo ===")

    var wg sync.WaitGroup

    // 多個 goroutine 同時調用
    for i := 0; i < 10; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()

            inst := GetInstance()
            fmt.Printf("Goroutine %d got instance: %p\n", id, inst)
        }(i)
    }

    wg.Wait()
    // "Creating singleton instance..." 只會打印一次
}

// 實際應用：一次性初始化
var (
    config     map[string]string
    configOnce sync.Once
)

func loadConfig() {
    configOnce.Do(func() {
        fmt.Println("Loading configuration...")
        config = map[string]string{
            "host": "localhost",
            "port": "8080",
        }
    })
}

func getConfig(key string) string {
    loadConfig()  // 可以多次調用，但只會執行一次
    return config[key]
}

func configDemo() {
    fmt.Println("\n=== Config Demo ===")

    var wg sync.WaitGroup

    for i := 0; i < 5; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()

            host := getConfig("host")
            fmt.Printf("Goroutine %d: host = %s\n", id, host)
        }(i)
    }

    wg.Wait()
}

func main() {
    onceDemo()
    configDemo()
}
```

**Node.js 對比:**
```javascript
// JavaScript 單例模式
class Singleton {
    static instance = null;
    static creating = false;

    constructor() {
        if (Singleton.instance) {
            return Singleton.instance;
        }

        console.log("Creating singleton instance...");
        this.data = "singleton data";
        Singleton.instance = this;
    }

    static getInstance() {
        if (!Singleton.instance) {
            Singleton.instance = new Singleton();
        }
        return Singleton.instance;
    }
}

async function singletonDemo() {
    console.log("=== Singleton Demo ===");

    const promises = [];

    for (let i = 0; i < 10; i++) {
        promises.push((async (id) => {
            const inst = Singleton.getInstance();
            console.log(`Promise ${id} got instance:`, inst);
        })(i));
    }

    await Promise.all(promises);
}

// 一次性初始化（使用 Promise）
let configPromise = null;
let config = null;

async function loadConfig() {
    if (!configPromise) {
        configPromise = (async () => {
            console.log("Loading configuration...");
            await new Promise(resolve => setTimeout(resolve, 100));
            config = {
                host: "localhost",
                port: "8080"
            };
        })();
    }
    await configPromise;
}

async function getConfig(key) {
    await loadConfig();  // 可以多次調用，但只會執行一次
    return config[key];
}

async function configDemo() {
    console.log("\n=== Config Demo ===");

    const promises = [];

    for (let i = 0; i < 5; i++) {
        promises.push((async (id) => {
            const host = await getConfig("host");
            console.log(`Promise ${id}: host = ${host}`);
        })(i));
    }

    await Promise.all(promises);
}

async function main() {
    await singletonDemo();
    await configDemo();
}

main();
```

### 示例 5: sync/atomic 原子操作

**Go:**
```go
package main

import (
    "fmt"
    "sync"
    "sync/atomic"
)

func atomicCounter() {
    fmt.Println("=== Atomic Counter ===")

    var counter int64
    var wg sync.WaitGroup

    for i := 0; i < 1000; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            atomic.AddInt64(&counter, 1)
        }()
    }

    wg.Wait()
    fmt.Println("Counter:", counter)  // 1000
}

func atomicOperations() {
    fmt.Println("\n=== Atomic Operations ===")

    var value int64 = 100

    // Add
    atomic.AddInt64(&value, 50)
    fmt.Println("After Add(50):", value)  // 150

    // CompareAndSwap (CAS)
    swapped := atomic.CompareAndSwapInt64(&value, 150, 200)
    fmt.Println("CAS(150, 200):", swapped, "value:", value)  // true, 200

    swapped = atomic.CompareAndSwapInt64(&value, 150, 300)
    fmt.Println("CAS(150, 300):", swapped, "value:", value)  // false, 200

    // Load
    loaded := atomic.LoadInt64(&value)
    fmt.Println("Load:", loaded)  // 200

    // Store
    atomic.StoreInt64(&value, 500)
    fmt.Println("After Store(500):", value)  // 500

    // Swap
    old := atomic.SwapInt64(&value, 999)
    fmt.Println("Swap(999), old:", old, "new:", value)  // 500, 999
}

// 實現自旋鎖
type SpinLock struct {
    flag int32
}

func (sl *SpinLock) Lock() {
    for !atomic.CompareAndSwapInt32(&sl.flag, 0, 1) {
        // 自旋等待
    }
}

func (sl *SpinLock) Unlock() {
    atomic.StoreInt32(&sl.flag, 0)
}

func spinLockDemo() {
    fmt.Println("\n=== SpinLock Demo ===")

    var sl SpinLock
    counter := 0
    var wg sync.WaitGroup

    for i := 0; i < 100; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()

            sl.Lock()
            counter++
            sl.Unlock()
        }()
    }

    wg.Wait()
    fmt.Println("Counter:", counter)  // 100
}

func main() {
    atomicCounter()
    atomicOperations()
    spinLockDemo()
}
```

**Node.js 對比:**
```javascript
// JavaScript 使用 Atomics（需要 SharedArrayBuffer）

async function atomicOperations() {
    console.log("=== Atomic Operations ===");

    const buffer = new SharedArrayBuffer(8);
    const view = new Int32Array(buffer);

    view[0] = 100;

    // Add
    Atomics.add(view, 0, 50);
    console.log("After Add(50):", view[0]);  // 150

    // CompareExchange (CAS)
    const old1 = Atomics.compareExchange(view, 0, 150, 200);
    console.log("CAS(150, 200), old:", old1, "value:", view[0]);  // 150, 200

    const old2 = Atomics.compareExchange(view, 0, 150, 300);
    console.log("CAS(150, 300), old:", old2, "value:", view[0]);  // 200, 200

    // Load
    const loaded = Atomics.load(view, 0);
    console.log("Load:", loaded);  // 200

    // Store
    Atomics.store(view, 0, 500);
    console.log("After Store(500):", view[0]);  // 500

    // Exchange (Swap)
    const old = Atomics.exchange(view, 0, 999);
    console.log("Exchange(999), old:", old, "new:", view[0]);  // 500, 999
}

atomicOperations();
```

## 重點總結

### 同步工具對比

| 工具 | Go | Node.js |
|------|-----|---------|
| 互斥鎖 | `sync.Mutex` | 無（Worker Threads 可用） |
| 讀寫鎖 | `sync.RWMutex` | 無 |
| 等待組 | `sync.WaitGroup` | `Promise.all` |
| 單次執行 | `sync.Once` | 手動實現 |
| 原子操作 | `sync/atomic` | `Atomics` (SharedArrayBuffer) |

### 最佳實踐

1. **選擇合適的同步工具**：
   - 簡單計數：`sync/atomic`
   - 保護共享數據：`sync.Mutex`
   - 讀多寫少：`sync.RWMutex`
   - 等待多個 goroutine：`sync.WaitGroup`
   - 一次性初始化：`sync.Once`

2. **避免死鎖**：
   - 獲取鎖的順序一致
   - 避免在持有鎖時調用未知函數
   - 使用 defer 確保解鎖
   - 設置超時機制

3. **性能考慮**：
   - 優先使用 channel 通信
   - 減小臨界區範圍
   - 考慮使用原子操作
   - 讀多寫少使用 RWMutex

## 練習題

### 練習 1: 線程安全的計數器
實現一個線程安全的計數器，支持遞增、遞減、獲取值。

### 練習 2: 緩存系統
實現一個帶過期時間的線程安全緩存。

### 練習 3: 連接池
實現一個資源池，限制並發訪問數量。

### 練習 4: 信號量
使用 channel 或鎖實現信號量。

### 練習 5: 生產者消費者
使用鎖和條件變量實現生產者-消費者模式。

### 答案提示

**練習 1:**
```go
type Counter struct {
    mu    sync.Mutex
    value int64
}

func (c *Counter) Inc() {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.value++
}

func (c *Counter) Dec() {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.value--
}

func (c *Counter) Value() int64 {
    c.mu.Lock()
    defer c.mu.Unlock()
    return c.value
}
```
