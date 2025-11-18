# Chapter 23: Channels 通信

## 概述

Channel 是 Go 中 goroutines 之間通信的主要方式，體現了"不要通過共享內存來通信，而應該通過通信來共享內存"的設計哲學。Channel 提供了類型安全的通信機制，是 Go 並發編程的核心。

## Node.js 與 Go 的對比

| 特性 | Node.js | Go |
|------|---------|-----|
| 通信機制 | EventEmitter, Promise | Channel |
| 類型安全 | 否（TypeScript 除外） | 是 |
| 阻塞 | 非阻塞（異步） | 可阻塞 |
| 方向控制 | 無 | 單向/雙向 |
| 緩衝 | 無直接概念 | 緩衝/非緩衝 |
| 關閉 | 手動清理 | close() |
| 多路復用 | Promise.race | select |

## 詳細概念解釋

### 1. Channel 類型

**無緩衝 Channel:**
- 發送和接收必須同時準備好
- 提供同步保證
- `make(chan Type)`

**緩衝 Channel:**
- 有固定容量的隊列
- 發送不阻塞直到緩衝區滿
- `make(chan Type, capacity)`

### 2. Channel 操作

```go
ch <- value    // 發送到 channel
value := <-ch  // 從 channel 接收
<-ch          // 接收並丟棄值
close(ch)     // 關閉 channel
```

### 3. Channel 方向

```go
chan T      // 雙向 channel
chan<- T    // 只發送 channel
<-chan T    // 只接收 channel
```

## 代碼示例

### 示例 1: 基本 Channel 操作

**Go:**
```go
package main

import "fmt"

func main() {
    // 創建無緩衝 channel
    ch := make(chan int)

    // 發送和接收必須在不同 goroutine
    go func() {
        ch <- 42  // 發送
    }()

    value := <-ch  // 接收
    fmt.Println("Received:", value)

    // 創建帶緩衝 channel
    bufferedCh := make(chan string, 2)

    // 可以連續發送，不會阻塞（直到緩衝區滿）
    bufferedCh <- "first"
    bufferedCh <- "second"

    fmt.Println("Received:", <-bufferedCh)
    fmt.Println("Received:", <-bufferedCh)

    // 使用 channel 同步
    done := make(chan bool)

    go func() {
        fmt.Println("Working...")
        done <- true
    }()

    <-done  // 等待完成
    fmt.Println("Done")
}
```

**Node.js 對比:**
```javascript
// Node.js 沒有直接的 channel 概念，使用 Promise 和 EventEmitter

const EventEmitter = require('events');

async function main() {
    // 使用 Promise 模擬無緩衝 channel
    let resolveValue;
    const channelPromise = new Promise(resolve => {
        resolveValue = resolve;
    });

    // 發送
    setTimeout(() => {
        resolveValue(42);
    }, 0);

    // 接收
    const value = await channelPromise;
    console.log("Received:", value);

    // 使用數組模擬帶緩衝 channel
    const bufferedCh = [];

    bufferedCh.push("first");
    bufferedCh.push("second");

    console.log("Received:", bufferedCh.shift());
    console.log("Received:", bufferedCh.shift());

    // 使用 Promise 同步
    const done = new Promise(resolve => {
        console.log("Working...");
        resolve(true);
    });

    await done;
    console.log("Done");
}

main();
```

### 示例 2: Channel 方向

**Go:**
```go
package main

import "fmt"

// 只能發送的 channel
func sendOnly(ch chan<- int) {
    ch <- 100
    // value := <-ch  // 編譯錯誤：無法從只發送 channel 接收
}

// 只能接收的 channel
func receiveOnly(ch <-chan int) {
    value := <-ch
    fmt.Println("Received:", value)
    // ch <- 200  // 編譯錯誤：無法向只接收 channel 發送
}

// 雙向 channel 可以轉換為單向 channel
func producer(ch chan<- int) {
    for i := 0; i < 5; i++ {
        ch <- i
    }
    close(ch)
}

func consumer(ch <-chan int) {
    for value := range ch {
        fmt.Println("Consumer received:", value)
    }
}

func main() {
    ch1 := make(chan int)

    go sendOnly(ch1)
    value := <-ch1
    fmt.Println("Main received:", value)

    ch2 := make(chan int)
    go func() {
        ch2 <- 200
    }()
    receiveOnly(ch2)

    // 生產者-消費者模式
    ch3 := make(chan int, 10)
    go producer(ch3)
    consumer(ch3)
}
```

**Node.js 對比:**
```javascript
// TypeScript 可以提供類型限制，但 JavaScript 運行時無法強制

// 使用類封裝來模擬方向控制
class SendOnlyChannel {
    constructor(buffer = []) {
        this._buffer = buffer;
    }

    send(value) {
        this._buffer.push(value);
    }

    // 不提供 receive 方法
}

class ReceiveOnlyChannel {
    constructor(buffer) {
        this._buffer = buffer;
    }

    receive() {
        return this._buffer.shift();
    }

    // 不提供 send 方法
}

class Channel {
    constructor(capacity = 0) {
        this._buffer = [];
        this._capacity = capacity;
    }

    send(value) {
        this._buffer.push(value);
    }

    receive() {
        return this._buffer.shift();
    }

    asSendOnly() {
        return new SendOnlyChannel(this._buffer);
    }

    asReceiveOnly() {
        return new ReceiveOnlyChannel(this._buffer);
    }
}

// 生產者-消費者
async function producer(ch) {
    for (let i = 0; i < 5; i++) {
        ch.send(i);
    }
}

async function consumer(ch) {
    let value;
    while ((value = ch.receive()) !== undefined) {
        console.log("Consumer received:", value);
        await new Promise(resolve => setTimeout(resolve, 100));
    }
}

async function main() {
    const ch = new Channel();
    await producer(ch.asSendOnly());
    await consumer(ch.asReceiveOnly());
}

main();
```

### 示例 3: 關閉 Channel

**Go:**
```go
package main

import "fmt"

func closeDemo() {
    ch := make(chan int, 3)

    // 發送一些值
    ch <- 1
    ch <- 2
    ch <- 3

    close(ch)  // 關閉 channel

    // 可以繼續接收，直到 channel 為空
    fmt.Println(<-ch)  // 1
    fmt.Println(<-ch)  // 2
    fmt.Println(<-ch)  // 3
    fmt.Println(<-ch)  // 0 (int 的零值)

    // 檢查 channel 是否關閉
    value, ok := <-ch
    fmt.Printf("Value: %d, OK: %v\n", value, ok)  // 0, false

    // 向已關閉的 channel 發送會 panic
    // ch <- 4  // panic!
}

func rangeDemo() {
    ch := make(chan int, 5)

    // 發送數據
    go func() {
        for i := 0; i < 5; i++ {
            ch <- i
        }
        close(ch)  // 必須關閉，否則 range 會阻塞
    }()

    // 使用 range 接收，直到 channel 關閉
    for value := range ch {
        fmt.Println("Range received:", value)
    }
}

func multipleProducers() {
    ch := make(chan int)
    done := make(chan bool)

    // 啟動多個生產者
    for i := 0; i < 3; i++ {
        go func(id int) {
            for j := 0; j < 3; j++ {
                ch <- id*10 + j
            }
            done <- true
        }(i)
    }

    // 等待所有生產者完成後關閉 channel
    go func() {
        for i := 0; i < 3; i++ {
            <-done
        }
        close(ch)
    }()

    // 接收所有數據
    for value := range ch {
        fmt.Println("Received:", value)
    }
}

func main() {
    fmt.Println("=== Close Demo ===")
    closeDemo()

    fmt.Println("\n=== Range Demo ===")
    rangeDemo()

    fmt.Println("\n=== Multiple Producers ===")
    multipleProducers()
}
```

**Node.js 對比:**
```javascript
const EventEmitter = require('events');

async function closeDemo() {
    const ch = [1, 2, 3];

    console.log(ch.shift());  // 1
    console.log(ch.shift());  // 2
    console.log(ch.shift());  // 3
    console.log(ch.shift());  // undefined

    // JavaScript 中需要手動檢查
    const value = ch.shift();
    console.log(`Value: ${value}, OK: ${value !== undefined}`);
}

async function rangeDemo() {
    const emitter = new EventEmitter();
    const values = [];

    // 生產者
    setTimeout(() => {
        for (let i = 0; i < 5; i++) {
            emitter.emit('data', i);
        }
        emitter.emit('close');
    }, 0);

    // 消費者
    await new Promise((resolve) => {
        emitter.on('data', (value) => {
            console.log("Range received:", value);
            values.push(value);
        });

        emitter.on('close', () => {
            console.log("Channel closed");
            resolve();
        });
    });
}

async function multipleProducers() {
    const emitter = new EventEmitter();
    let activeProducers = 3;

    // 啟動多個生產者
    for (let i = 0; i < 3; i++) {
        (async (id) => {
            for (let j = 0; j < 3; j++) {
                emitter.emit('data', id * 10 + j);
                await new Promise(resolve => setTimeout(resolve, 10));
            }

            activeProducers--;
            if (activeProducers === 0) {
                emitter.emit('close');
            }
        })(i);
    }

    // 接收所有數據
    await new Promise((resolve) => {
        emitter.on('data', (value) => {
            console.log("Received:", value);
        });

        emitter.on('close', () => {
            resolve();
        });
    });
}

async function main() {
    console.log("=== Close Demo ===");
    await closeDemo();

    console.log("\n=== Range Demo ===");
    await rangeDemo();

    console.log("\n=== Multiple Producers ===");
    await multipleProducers();
}

main();
```

### 示例 4: Channel 模式

**Go:**
```go
package main

import (
    "fmt"
    "time"
)

// 模式 1: Pipeline（管道）
func generator(nums ...int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for _, n := range nums {
            out <- n
        }
    }()
    return out
}

func square(in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for n := range in {
            out <- n * n
        }
    }()
    return out
}

func pipeline() {
    fmt.Println("=== Pipeline ===")

    // 構建管道
    nums := generator(1, 2, 3, 4, 5)
    squared := square(nums)

    // 消費結果
    for result := range squared {
        fmt.Println(result)
    }
}

// 模式 2: Fan-out（扇出）
func fanOut(in <-chan int, n int) []<-chan int {
    channels := make([]<-chan int, n)

    for i := 0; i < n; i++ {
        channels[i] = square(in)
    }

    return channels
}

// 模式 3: Fan-in（扇入）
func fanIn(channels ...<-chan int) <-chan int {
    out := make(chan int)

    for _, ch := range channels {
        ch := ch  // 捕獲循環變量
        go func() {
            for value := range ch {
                out <- value
            }
        }()
    }

    return out
}

func fanOutFanIn() {
    fmt.Println("\n=== Fan-out, Fan-in ===")

    nums := generator(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)

    // Fan-out: 分配到多個 worker
    workers := fanOut(nums, 3)

    // Fan-in: 合併結果
    results := fanIn(workers...)

    // 注意：需要知道何時關閉 results channel
    // 這裡簡化處理
    count := 0
    for result := range results {
        fmt.Println(result)
        count++
        if count == 10 {
            break
        }
    }
}

// 模式 4: Timeout（超時）
func timeoutPattern() {
    fmt.Println("\n=== Timeout Pattern ===")

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

// 模式 5: Quit Channel（退出通道）
func quitPattern() {
    fmt.Println("\n=== Quit Pattern ===")

    quit := make(chan bool)
    data := make(chan int)

    // Worker
    go func() {
        for {
            select {
            case value := <-data:
                fmt.Println("Processing:", value)
            case <-quit:
                fmt.Println("Worker quitting")
                return
            }
        }
    }()

    // 發送一些數據
    for i := 0; i < 5; i++ {
        data <- i
        time.Sleep(100 * time.Millisecond)
    }

    // 發送退出信號
    quit <- true
    time.Sleep(100 * time.Millisecond)
}

func main() {
    pipeline()
    fanOutFanIn()
    timeoutPattern()
    quitPattern()
}
```

**Node.js 對比:**
```javascript
// 模式 1: Pipeline
async function* generator(...nums) {
    for (const n of nums) {
        yield n;
    }
}

async function* square(input) {
    for await (const n of input) {
        yield n * n;
    }
}

async function pipeline() {
    console.log("=== Pipeline ===");

    const nums = generator(1, 2, 3, 4, 5);
    const squared = square(nums);

    for await (const result of squared) {
        console.log(result);
    }
}

// 模式 2: 超時
async function timeoutPattern() {
    console.log("\n=== Timeout Pattern ===");

    const workPromise = new Promise((resolve) => {
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

// 模式 3: 取消
async function quitPattern() {
    console.log("\n=== Quit Pattern ===");

    let shouldQuit = false;
    const data = [];

    // Worker
    const worker = async () => {
        while (!shouldQuit) {
            if (data.length > 0) {
                const value = data.shift();
                console.log("Processing:", value);
            }
            await new Promise(resolve => setTimeout(resolve, 50));
        }
        console.log("Worker quitting");
    };

    // 啟動 worker
    const workerPromise = worker();

    // 發送數據
    for (let i = 0; i < 5; i++) {
        data.push(i);
        await new Promise(resolve => setTimeout(resolve, 100));
    }

    // 發送退出信號
    shouldQuit = true;
    await workerPromise;
}

async function main() {
    await pipeline();
    await timeoutPattern();
    await quitPattern();
}

main();
```

### 示例 5: 緩衝 vs 無緩衝

**Go:**
```go
package main

import (
    "fmt"
    "time"
)

func unbufferedDemo() {
    fmt.Println("=== Unbuffered Channel ===")

    ch := make(chan int)

    // 無緩衝 channel 必須有接收者才能發送
    go func() {
        fmt.Println("Goroutine: Sending...")
        ch <- 42
        fmt.Println("Goroutine: Sent!")
    }()

    time.Sleep(100 * time.Millisecond)
    fmt.Println("Main: Receiving...")
    value := <-ch
    fmt.Println("Main: Received:", value)
}

func bufferedDemo() {
    fmt.Println("\n=== Buffered Channel ===")

    ch := make(chan int, 2)

    // 緩衝 channel 可以在沒有接收者的情況下發送
    fmt.Println("Main: Sending...")
    ch <- 1
    fmt.Println("Main: Sent 1")
    ch <- 2
    fmt.Println("Main: Sent 2")

    // 第三次發送會阻塞（如果沒有接收者）
    go func() {
        fmt.Println("Goroutine: Sending...")
        ch <- 3
        fmt.Println("Goroutine: Sent 3")
    }()

    time.Sleep(100 * time.Millisecond)
    fmt.Println("Main: Receiving...")
    fmt.Println("Received:", <-ch)
    fmt.Println("Received:", <-ch)
    fmt.Println("Received:", <-ch)
}

func channelLength() {
    fmt.Println("\n=== Channel Length ===")

    ch := make(chan int, 5)

    ch <- 1
    ch <- 2
    ch <- 3

    fmt.Println("Length:", len(ch))      // 3
    fmt.Println("Capacity:", cap(ch))    // 5

    <-ch
    fmt.Println("After receive:")
    fmt.Println("Length:", len(ch))      // 2
    fmt.Println("Capacity:", cap(ch))    // 5
}

func main() {
    unbufferedDemo()
    bufferedDemo()
    channelLength()
}
```

**Node.js 對比:**
```javascript
const EventEmitter = require('events');

async function unbufferedDemo() {
    console.log("=== Unbuffered (Promise) ===");

    let resolve;
    const promise = new Promise(r => { resolve = r; });

    // 發送（resolve promise）
    setTimeout(() => {
        console.log("Timeout: Sending...");
        resolve(42);
        console.log("Timeout: Sent!");
    }, 0);

    await new Promise(r => setTimeout(r, 100));
    console.log("Main: Receiving...");
    const value = await promise;
    console.log("Main: Received:", value);
}

async function bufferedDemo() {
    console.log("\n=== Buffered (Array) ===");

    const buffer = [];
    const maxSize = 2;

    console.log("Main: Sending...");
    buffer.push(1);
    console.log("Main: Sent 1");
    buffer.push(2);
    console.log("Main: Sent 2");

    // 第三次發送（需要等待空間）
    setTimeout(() => {
        console.log("Timeout: Sending...");
        buffer.push(3);
        console.log("Timeout: Sent 3");
    }, 0);

    await new Promise(r => setTimeout(r, 100));
    console.log("Main: Receiving...");
    console.log("Received:", buffer.shift());
    console.log("Received:", buffer.shift());
    console.log("Received:", buffer.shift());
}

async function bufferInfo() {
    console.log("\n=== Buffer Info ===");

    const buffer = [];
    const maxSize = 5;

    buffer.push(1, 2, 3);

    console.log("Length:", buffer.length);     // 3
    console.log("Capacity:", maxSize);         // 5

    buffer.shift();
    console.log("After receive:");
    console.log("Length:", buffer.length);     // 2
    console.log("Capacity:", maxSize);         // 5
}

async function main() {
    await unbufferedDemo();
    await bufferedDemo();
    await bufferInfo();
}

main();
```

## 重點總結

### Channel vs Node.js 通信

| 特性 | Go Channel | Node.js |
|------|-----------|---------|
| 同步機制 | 阻塞 | 異步（Promise） |
| 類型安全 | 是 | 否（TS 可以） |
| 緩衝控制 | 內建 | 手動實現 |
| 方向控制 | 編譯時檢查 | 運行時約定 |
| 關閉語義 | 內建 close() | 手動管理 |

### 最佳實踐

1. **何時使用緩衝**：
   - 已知容量限制
   - 減少 goroutine 阻塞
   - 批量處理

2. **關閉 Channel**：
   - 發送者關閉，接收者檢查
   - 使用 range 自動處理關閉
   - 避免重複關閉

3. **方向控制**：
   - 函數參數使用單向 channel
   - 提高代碼可讀性
   - 編譯時防錯

4. **避免死鎖**：
   - 確保發送有接收者
   - 正確關閉 channel
   - 使用 select 處理多個 channel

## 練習題

### 練習 1: 有限緩衝隊列
實現一個線程安全的有限緩衝隊列。

### 練習 2: 請求限流器
使用 channel 實現一個限流器，控制並發請求數。

### 練習 3: Pipeline 處理
實現一個數據處理管道：讀取 → 過濾 → 轉換 → 輸出。

### 練習 4: 廣播
實現一個廣播機制，一個發送者，多個接收者。

### 練習 5: 優先級隊列
使用多個 channel 實現優先級隊列。

### 答案提示

**練習 1:**
```go
type Queue struct {
    ch chan interface{}
}

func NewQueue(size int) *Queue {
    return &Queue{ch: make(chan interface{}, size)}
}

func (q *Queue) Enqueue(item interface{}) {
    q.ch <- item
}

func (q *Queue) Dequeue() interface{} {
    return <-q.ch
}
```
