# Chapter 24: Select 語句

## 概述

select 是 Go 中用於處理多個 channel 操作的控制結構，類似於 switch 語句但專門用於 channel。它讓一個 goroutine 可以等待多個通信操作，哪個 channel 準備好就執行哪個，實現了多路復用的功能。

## Node.js 與 Go 的對比

| 特性 | Node.js | Go |
|------|---------|-----|
| 多路復用 | `Promise.race()` | `select` |
| 超時處理 | `Promise.race()` + `setTimeout` | `select` + `time.After` |
| 默認分支 | 無 | `default` |
| 隨機選擇 | 無 | 多個 case 就緒時隨機選擇 |
| 阻塞行為 | 總是異步 | 可阻塞或非阻塞 |

## 詳細概念解釋

### 1. select 語法

```go
select {
case value := <-ch1:
    // 從 ch1 接收到值
case ch2 <- value:
    // 向 ch2 發送值
case <-time.After(duration):
    // 超時
default:
    // 所有 case 都未就緒時執行（非阻塞）
}
```

### 2. select 特性

- **隨機選擇**：多個 case 就緒時隨機選擇一個
- **阻塞**：沒有 default 時會阻塞，直到某個 case 就緒
- **非阻塞**：有 default 時不會阻塞
- **nil channel**：從 nil channel 讀取或寫入會永遠阻塞

## 代碼示例

### 示例 1: 基本 select 用法

**Go:**
```go
package main

import (
    "fmt"
    "time"
)

func basicSelect() {
    fmt.Println("=== Basic Select ===")

    ch1 := make(chan string)
    ch2 := make(chan string)

    go func() {
        time.Sleep(100 * time.Millisecond)
        ch1 <- "from ch1"
    }()

    go func() {
        time.Sleep(200 * time.Millisecond)
        ch2 <- "from ch2"
    }()

    // select 會等待第一個就緒的 channel
    select {
    case msg1 := <-ch1:
        fmt.Println("Received:", msg1)
    case msg2 := <-ch2:
        fmt.Println("Received:", msg2)
    }
}

func multipleSelect() {
    fmt.Println("\n=== Multiple Select ===")

    ch1 := make(chan string)
    ch2 := make(chan string)

    go func() {
        time.Sleep(100 * time.Millisecond)
        ch1 <- "from ch1"
    }()

    go func() {
        time.Sleep(200 * time.Millisecond)
        ch2 <- "from ch2"
    }()

    // 可以多次 select
    for i := 0; i < 2; i++ {
        select {
        case msg1 := <-ch1:
            fmt.Println("Received:", msg1)
        case msg2 := <-ch2:
            fmt.Println("Received:", msg2)
        }
    }
}

func selectWithDefault() {
    fmt.Println("\n=== Select with Default ===")

    ch := make(chan int)

    // 非阻塞接收
    select {
    case value := <-ch:
        fmt.Println("Received:", value)
    default:
        fmt.Println("No value ready, doing something else")
    }

    // 非阻塞發送
    select {
    case ch <- 42:
        fmt.Println("Sent value")
    default:
        fmt.Println("No receiver ready")
    }
}

func main() {
    basicSelect()
    multipleSelect()
    selectWithDefault()
}
```

**Node.js 對比:**
```javascript
async function basicRace() {
    console.log("=== Basic Race ===");

    const promise1 = new Promise(resolve => {
        setTimeout(() => resolve("from promise1"), 100);
    });

    const promise2 = new Promise(resolve => {
        setTimeout(() => resolve("from promise2"), 200);
    });

    // Promise.race 返回第一個完成的 Promise
    const result = await Promise.race([promise1, promise2]);
    console.log("Received:", result);
}

async function multipleRace() {
    console.log("\n=== Multiple Race ===");

    const promise1 = new Promise(resolve => {
        setTimeout(() => resolve("from promise1"), 100);
    });

    const promise2 = new Promise(resolve => {
        setTimeout(() => resolve("from promise2"), 200);
    });

    // 需要手動處理多次
    const result1 = await Promise.race([promise1, promise2]);
    console.log("Received:", result1);

    // 第二次需要等待剩餘的
    await new Promise(resolve => setTimeout(resolve, 150));
    const result2 = "from promise2";
    console.log("Received:", result2);
}

async function nonBlocking() {
    console.log("\n=== Non-blocking ===");

    const pending = new Promise(() => {});  // 永遠不 resolve

    // 使用 Promise.race 實現非阻塞
    const immediate = Promise.resolve("default");
    const result = await Promise.race([pending, immediate]);
    console.log("No value ready, doing something else:", result);
}

async function main() {
    await basicRace();
    await multipleRace();
    await nonBlocking();
}

main();
```

### 示例 2: 超時模式

**Go:**
```go
package main

import (
    "fmt"
    "time"
)

func timeoutPattern() {
    fmt.Println("=== Timeout Pattern ===")

    ch := make(chan string)

    go func() {
        time.Sleep(2 * time.Second)
        ch <- "result"
    }()

    select {
    case result := <-ch:
        fmt.Println("Got result:", result)
    case <-time.After(1 * time.Second):
        fmt.Println("Timeout!")
    }
}

func perOperationTimeout() {
    fmt.Println("\n=== Per-Operation Timeout ===")

    ch := make(chan string)

    for i := 0; i < 3; i++ {
        go func(id int) {
            time.Sleep(time.Duration(id*500) * time.Millisecond)
            ch <- fmt.Sprintf("result %d", id)
        }(i)

        // 每個操作都有獨立的超時
        select {
        case result := <-ch:
            fmt.Println("Got:", result)
        case <-time.After(800 * time.Millisecond):
            fmt.Printf("Operation %d timeout\n", i)
        }
    }
}

func overallTimeout() {
    fmt.Println("\n=== Overall Timeout ===")

    ch := make(chan int)
    timeout := time.After(1 * time.Second)

    go func() {
        for i := 0; ; i++ {
            time.Sleep(300 * time.Millisecond)
            ch <- i
        }
    }()

    for {
        select {
        case value := <-ch:
            fmt.Println("Received:", value)
        case <-timeout:
            fmt.Println("Overall timeout!")
            return
        }
    }
}

func main() {
    timeoutPattern()
    perOperationTimeout()
    overallTimeout()
}
```

**Node.js 對比:**
```javascript
async function timeoutPattern() {
    console.log("=== Timeout Pattern ===");

    const workPromise = new Promise(resolve => {
        setTimeout(() => resolve("result"), 2000);
    });

    const timeoutPromise = new Promise((_, reject) => {
        setTimeout(() => reject(new Error("Timeout!")), 1000);
    });

    try {
        const result = await Promise.race([workPromise, timeoutPromise]);
        console.log("Got result:", result);
    } catch (err) {
        console.log(err.message);
    }
}

async function perOperationTimeout() {
    console.log("\n=== Per-Operation Timeout ===");

    for (let i = 0; i < 3; i++) {
        const workPromise = new Promise(resolve => {
            setTimeout(() => resolve(`result ${i}`), i * 500);
        });

        const timeoutPromise = new Promise((_, reject) => {
            setTimeout(() => reject(new Error(`Operation ${i} timeout`)), 800);
        });

        try {
            const result = await Promise.race([workPromise, timeoutPromise]);
            console.log("Got:", result);
        } catch (err) {
            console.log(err.message);
        }
    }
}

async function overallTimeout() {
    console.log("\n=== Overall Timeout ===");

    let cancelled = false;
    setTimeout(() => {
        cancelled = true;
        console.log("Overall timeout!");
    }, 1000);

    let i = 0;
    while (!cancelled) {
        await new Promise(resolve => setTimeout(resolve, 300));
        console.log("Received:", i++);
    }
}

async function main() {
    await timeoutPattern();
    await perOperationTimeout();
    await overallTimeout();
}

main();
```

### 示例 3: 實現多路復用

**Go:**
```go
package main

import (
    "fmt"
    "time"
)

func multiplexer(ch1, ch2 <-chan string) <-chan string {
    out := make(chan string)

    go func() {
        defer close(out)
        for {
            select {
            case msg, ok := <-ch1:
                if !ok {
                    ch1 = nil  // 關閉的 channel 設為 nil
                    continue
                }
                out <- fmt.Sprintf("Ch1: %s", msg)
            case msg, ok := <-ch2:
                if !ok {
                    ch2 = nil
                    continue
                }
                out <- fmt.Sprintf("Ch2: %s", msg)
            }

            // 兩個 channel 都關閉時退出
            if ch1 == nil && ch2 == nil {
                return
            }
        }
    }()

    return out
}

func multiplexDemo() {
    fmt.Println("=== Multiplexer Demo ===")

    ch1 := make(chan string)
    ch2 := make(chan string)

    // 啟動發送者
    go func() {
        for i := 0; i < 3; i++ {
            ch1 <- fmt.Sprintf("msg%d", i)
            time.Sleep(100 * time.Millisecond)
        }
        close(ch1)
    }()

    go func() {
        for i := 0; i < 3; i++ {
            ch2 <- fmt.Sprintf("data%d", i)
            time.Sleep(150 * time.Millisecond)
        }
        close(ch2)
    }()

    // 接收多路復用的消息
    out := multiplexer(ch1, ch2)
    for msg := range out {
        fmt.Println("Received:", msg)
    }
}

// 帶優先級的多路復用
func priorityMultiplex(high, low <-chan string) <-chan string {
    out := make(chan string)

    go func() {
        defer close(out)
        for {
            select {
            case msg, ok := <-high:
                if !ok {
                    high = nil
                    continue
                }
                out <- fmt.Sprintf("[HIGH] %s", msg)
            default:
                // 只有在高優先級沒有數據時才檢查低優先級
                select {
                case msg, ok := <-low:
                    if !ok {
                        low = nil
                        continue
                    }
                    out <- fmt.Sprintf("[LOW] %s", msg)
                case msg, ok := <-high:
                    if !ok {
                        high = nil
                        continue
                    }
                    out <- fmt.Sprintf("[HIGH] %s", msg)
                }
            }

            if high == nil && low == nil {
                return
            }
        }
    }()

    return out
}

func priorityDemo() {
    fmt.Println("\n=== Priority Multiplex Demo ===")

    high := make(chan string, 10)
    low := make(chan string, 10)

    // 發送混合消息
    high <- "urgent1"
    high <- "urgent2"
    low <- "normal1"
    low <- "normal2"
    high <- "urgent3"
    low <- "normal3"

    close(high)
    close(low)

    // 接收，高優先級優先
    out := priorityMultiplex(high, low)
    for msg := range out {
        fmt.Println(msg)
    }
}

func main() {
    multiplexDemo()
    priorityDemo()
}
```

**Node.js 對比:**
```javascript
const EventEmitter = require('events');

async function multiplexDemo() {
    console.log("=== Multiplexer Demo ===");

    const emitter1 = new EventEmitter();
    const emitter2 = new EventEmitter();
    const output = new EventEmitter();

    let closed1 = false;
    let closed2 = false;

    // 多路復用
    emitter1.on('data', msg => {
        output.emit('data', `Ch1: ${msg}`);
    });

    emitter1.on('close', () => {
        closed1 = true;
        if (closed1 && closed2) {
            output.emit('close');
        }
    });

    emitter2.on('data', msg => {
        output.emit('data', `Ch2: ${msg}`);
    });

    emitter2.on('close', () => {
        closed2 = true;
        if (closed1 && closed2) {
            output.emit('close');
        }
    });

    // 發送者 1
    (async () => {
        for (let i = 0; i < 3; i++) {
            emitter1.emit('data', `msg${i}`);
            await new Promise(resolve => setTimeout(resolve, 100));
        }
        emitter1.emit('close');
    })();

    // 發送者 2
    (async () => {
        for (let i = 0; i < 3; i++) {
            emitter2.emit('data', `data${i}`);
            await new Promise(resolve => setTimeout(resolve, 150));
        }
        emitter2.emit('close');
    })();

    // 接收
    await new Promise(resolve => {
        output.on('data', msg => {
            console.log("Received:", msg);
        });

        output.on('close', () => {
            resolve();
        });
    });
}

async function priorityDemo() {
    console.log("\n=== Priority Demo ===");

    const high = ["urgent1", "urgent2", "urgent3"];
    const low = ["normal1", "normal2", "normal3"];

    // 處理優先級隊列
    while (high.length > 0 || low.length > 0) {
        if (high.length > 0) {
            console.log(`[HIGH] ${high.shift()}`);
        } else if (low.length > 0) {
            console.log(`[LOW] ${low.shift()}`);
        }
    }
}

async function main() {
    await multiplexDemo();
    await priorityDemo();
}

main();
```

### 示例 4: 退出和取消模式

**Go:**
```go
package main

import (
    "fmt"
    "time"
)

func workerWithQuit() {
    fmt.Println("=== Worker with Quit ===")

    quit := make(chan bool)
    done := make(chan bool)

    go func() {
        defer func() { done <- true }()

        ticker := time.NewTicker(100 * time.Millisecond)
        defer ticker.Stop()

        for {
            select {
            case <-ticker.C:
                fmt.Println("Working...")
            case <-quit:
                fmt.Println("Worker stopped")
                return
            }
        }
    }()

    time.Sleep(500 * time.Millisecond)
    quit <- true
    <-done
}

func multipleWorkers() {
    fmt.Println("\n=== Multiple Workers ===")

    quit := make(chan bool)
    done := make(chan bool)

    worker := func(id int) {
        defer func() { done <- true }()

        for {
            select {
            case <-time.After(200 * time.Millisecond):
                fmt.Printf("Worker %d working\n", id)
            case <-quit:
                fmt.Printf("Worker %d stopped\n", id)
                return
            }
        }
    }

    // 啟動多個 worker
    for i := 1; i <= 3; i++ {
        go worker(i)
    }

    time.Sleep(600 * time.Millisecond)

    // 通知所有 worker 停止
    close(quit)

    // 等待所有 worker 完成
    for i := 0; i < 3; i++ {
        <-done
    }

    fmt.Println("All workers stopped")
}

func selectWithSend() {
    fmt.Println("\n=== Select with Send ===")

    ch := make(chan int)
    quit := make(chan bool)

    go func() {
        for i := 0; ; i++ {
            select {
            case ch <- i:
                fmt.Println("Sent:", i)
            case <-quit:
                fmt.Println("Sender stopped")
                return
            }
            time.Sleep(100 * time.Millisecond)
        }
    }()

    // 接收幾個值
    for i := 0; i < 3; i++ {
        fmt.Println("Received:", <-ch)
    }

    quit <- true
    time.Sleep(100 * time.Millisecond)
}

func main() {
    workerWithQuit()
    multipleWorkers()
    selectWithSend()
}
```

**Node.js 對比:**
```javascript
async function workerWithQuit() {
    console.log("=== Worker with Quit ===");

    let shouldQuit = false;
    const done = new Promise(resolve => {
        const interval = setInterval(() => {
            if (shouldQuit) {
                console.log("Worker stopped");
                clearInterval(interval);
                resolve();
                return;
            }
            console.log("Working...");
        }, 100);
    });

    await new Promise(resolve => setTimeout(resolve, 500));
    shouldQuit = true;
    await done;
}

async function multipleWorkers() {
    console.log("\n=== Multiple Workers ===");

    let shouldQuit = false;

    const worker = async (id) => {
        while (!shouldQuit) {
            await new Promise(resolve => setTimeout(resolve, 200));
            if (!shouldQuit) {
                console.log(`Worker ${id} working`);
            }
        }
        console.log(`Worker ${id} stopped`);
    };

    // 啟動多個 worker
    const workers = [];
    for (let i = 1; i <= 3; i++) {
        workers.push(worker(i));
    }

    await new Promise(resolve => setTimeout(resolve, 600));

    // 通知所有 worker 停止
    shouldQuit = true;

    // 等待所有 worker 完成
    await Promise.all(workers);

    console.log("All workers stopped");
}

async function selectWithSend() {
    console.log("\n=== Select with Send ===");

    let shouldQuit = false;
    const ch = [];

    // 發送者
    const sender = (async () => {
        let i = 0;
        while (!shouldQuit) {
            ch.push(i);
            console.log("Sent:", i);
            i++;
            await new Promise(resolve => setTimeout(resolve, 100));
        }
        console.log("Sender stopped");
    })();

    // 接收幾個值
    for (let i = 0; i < 3; i++) {
        while (ch.length === 0) {
            await new Promise(resolve => setTimeout(resolve, 10));
        }
        console.log("Received:", ch.shift());
    }

    shouldQuit = true;
    await sender;
}

async function main() {
    await workerWithQuit();
    await multipleWorkers();
    await selectWithSend();
}

main();
```

### 示例 5: 空 select 和隨機選擇

**Go:**
```go
package main

import (
    "fmt"
    "time"
)

func emptySelect() {
    fmt.Println("=== Empty Select ===")

    // 空 select 會永遠阻塞
    go func() {
        select {}
    }()

    time.Sleep(100 * time.Millisecond)
    fmt.Println("Main continues (goroutine is blocked forever)")
}

func randomSelection() {
    fmt.Println("\n=== Random Selection ===")

    ch1 := make(chan string, 1)
    ch2 := make(chan string, 1)
    ch3 := make(chan string, 1)

    // 所有 channel 都準備好
    ch1 <- "from ch1"
    ch2 <- "from ch2"
    ch3 <- "from ch3"

    // select 會隨機選擇一個就緒的 case
    for i := 0; i < 3; i++ {
        select {
        case msg := <-ch1:
            fmt.Println("Selected:", msg)
        case msg := <-ch2:
            fmt.Println("Selected:", msg)
        case msg := <-ch3:
            fmt.Println("Selected:", msg)
        }
    }
}

func fairScheduling() {
    fmt.Println("\n=== Fair Scheduling ===")

    ch1 := make(chan int)
    ch2 := make(chan int)

    // 發送者 1（快速）
    go func() {
        for i := 0; i < 10; i++ {
            ch1 <- i
            time.Sleep(10 * time.Millisecond)
        }
        close(ch1)
    }()

    // 發送者 2（慢速）
    go func() {
        for i := 100; i < 110; i++ {
            ch2 <- i
            time.Sleep(50 * time.Millisecond)
        }
        close(ch2)
    }()

    // select 會公平地從兩個 channel 接收
    for ch1 != nil || ch2 != nil {
        select {
        case val, ok := <-ch1:
            if !ok {
                ch1 = nil
                continue
            }
            fmt.Println("From ch1:", val)
        case val, ok := <-ch2:
            if !ok {
                ch2 = nil
                continue
            }
            fmt.Println("From ch2:", val)
        }
    }
}

func main() {
    emptySelect()
    randomSelection()
    fairScheduling()
}
```

**Node.js 對比:**
```javascript
async function randomSelection() {
    console.log("=== Random Selection ===");

    const promises = [
        Promise.resolve("from promise1"),
        Promise.resolve("from promise2"),
        Promise.resolve("from promise3")
    ];

    // Promise.race 總是選擇最先 resolve 的
    // 如果都已經 resolve，選擇第一個
    for (let i = 0; i < 3; i++) {
        const result = await promises[i];
        console.log("Selected:", result);
    }
}

async function fairScheduling() {
    console.log("\n=== Fair Scheduling ===");

    const stream1 = [];
    const stream2 = [];

    // 生產者 1（快速）
    (async () => {
        for (let i = 0; i < 10; i++) {
            stream1.push(i);
            await new Promise(resolve => setTimeout(resolve, 10));
        }
        stream1.push(null);  // 結束標記
    })();

    // 生產者 2（慢速）
    (async () => {
        for (let i = 100; i < 110; i++) {
            stream2.push(i);
            await new Promise(resolve => setTimeout(resolve, 50));
        }
        stream2.push(null);  // 結束標記
    })();

    // 交替檢查兩個流
    let active1 = true;
    let active2 = true;

    while (active1 || active2) {
        await new Promise(resolve => setTimeout(resolve, 20));

        if (stream1.length > 0) {
            const val = stream1.shift();
            if (val === null) {
                active1 = false;
            } else {
                console.log("From stream1:", val);
            }
        }

        if (stream2.length > 0) {
            const val = stream2.shift();
            if (val === null) {
                active2 = false;
            } else {
                console.log("From stream2:", val);
            }
        }
    }
}

async function main() {
    await randomSelection();
    await fairScheduling();
}

main();
```

## 重點總結

### select vs Promise.race

| 特性 | Go select | Promise.race |
|------|----------|--------------|
| 用途 | 多 channel 操作 | 多 Promise 競爭 |
| 阻塞 | 可阻塞 | 非阻塞（異步） |
| 默認分支 | 有 default | 無 |
| 隨機性 | 隨機選擇就緒 case | 確定性 |
| 多次選擇 | 循環中使用 | 需要重新創建 |

### 最佳實踐

1. **使用 select 場景**：
   - 等待多個 channel
   - 實現超時
   - 非阻塞通信
   - 取消操作

2. **避免陷阱**：
   - nil channel 會永遠阻塞
   - 空 select 會永遠阻塞
   - 關閉的 channel 立即返回零值

3. **模式應用**：
   - 超時：`select` + `time.After`
   - 退出：`select` + `quit channel`
   - 多路復用：`select` + 多個 channel
   - 優先級：嵌套 `select` + `default`

## 練習題

### 練習 1: 超時重試
實現一個帶超時的重試機制。

### 練習 2: 速率限制器
使用 select 實現一個速率限制器。

### 練習 3: 心跳檢測
實現一個心跳檢測系統，超時則認為連接斷開。

### 練習 4: 優雅關閉
實現一個 Worker Pool 的優雅關閉機制。

### 練習 5: 多源合併
合併多個 channel 的數據，支持超時和取消。

### 答案提示

**練習 3:**
```go
func heartbeat(interval, timeout time.Duration) {
    heartbeat := make(chan bool)

    go func() {
        ticker := time.NewTicker(interval)
        defer ticker.Stop()

        for {
            select {
            case <-ticker.C:
                heartbeat <- true
            }
        }
    }()

    for {
        select {
        case <-heartbeat:
            fmt.Println("Heartbeat received")
        case <-time.After(timeout):
            fmt.Println("Connection lost!")
            return
        }
    }
}
```
