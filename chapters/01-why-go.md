# 第一章：為什麼選擇 Go？

## 章節概述

本章將幫助 Node.js 開發者理解為什麼要學習 Go 語言。我們將從多個維度對比 Node.js 和 Go，包括性能、並發模型、類型系統、生態系統等，讓你了解 Go 的優勢以及何時應該選擇 Go。

## Node.js vs Go：關鍵對比

### 1. 語言特性對比

| 特性 | Node.js (JavaScript) | Go |
|------|---------------------|-----|
| **類型系統** | 動態類型 | 靜態類型 |
| **編譯/解釋** | JIT 編譯（V8引擎） | 編譯為原生機器碼 |
| **並發模型** | 事件循環 + 異步I/O | Goroutines + Channels |
| **內存管理** | 垃圾回收（V8 GC） | 垃圾回收（更高效） |
| **執行速度** | 快速（但受限於單線程） | 非常快速（多核並行） |
| **啟動時間** | 較慢（需要加載大量依賴） | 極快（單一二進制文件） |
| **部署** | 需要 Node.js 運行時 + node_modules | 單一可執行文件 |

### 2. 性能對比

#### CPU 密集型任務

**Node.js 的挑戰：**
```javascript
// Node.js - 單線程，CPU 密集任務會阻塞
// fibonacci.js
function fibonacci(n) {
  if (n <= 1) return n;
  return fibonacci(n - 1) + fibonacci(n - 2);
}

// 這會阻塞整個事件循環
const start = Date.now();
const result = fibonacci(40);
console.log(`Result: ${result}, Time: ${Date.now() - start}ms`);
// 輸出：Result: 102334155, Time: ~1200ms
```

**Go 的優勢：**
```go
// Go - 可以輕鬆利用多核心
// fibonacci.go
package main

import (
    "fmt"
    "time"
)

func fibonacci(n int) int {
    if n <= 1 {
        return n
    }
    return fibonacci(n-1) + fibonacci(n-2)
}

func main() {
    start := time.Now()
    result := fibonacci(40)
    fmt.Printf("Result: %d, Time: %v\n", result, time.Since(start))
    // 輸出：Result: 102334155, Time: ~500ms

    // 而且可以同時計算多個，利用多核心
    start = time.Now()
    ch := make(chan int, 4)

    for i := 0; i < 4; i++ {
        go func() {
            ch <- fibonacci(40)
        }()
    }

    for i := 0; i < 4; i++ {
        <-ch
    }
    fmt.Printf("4 parallel calculations, Time: %v\n", time.Since(start))
    // 輸出：4 parallel calculations, Time: ~500ms（在4核CPU上）
}
```

#### I/O 密集型任務

**Node.js 的優勢：**
```javascript
// Node.js 在 I/O 密集型任務上表現優秀
const fs = require('fs').promises;

async function readMultipleFiles() {
    const start = Date.now();

    // 並發讀取多個文件
    const files = await Promise.all([
        fs.readFile('file1.txt', 'utf8'),
        fs.readFile('file2.txt', 'utf8'),
        fs.readFile('file3.txt', 'utf8'),
    ]);

    console.log(`Read ${files.length} files in ${Date.now() - start}ms`);
}
```

**Go 同樣優秀：**
```go
package main

import (
    "fmt"
    "os"
    "sync"
    "time"
)

func readMultipleFiles() {
    start := time.Now()
    var wg sync.WaitGroup
    files := []string{"file1.txt", "file2.txt", "file3.txt"}

    for _, filename := range files {
        wg.Add(1)
        go func(name string) {
            defer wg.Done()
            os.ReadFile(name)
        }(filename)
    }

    wg.Wait()
    fmt.Printf("Read %d files in %v\n", len(files), time.Since(start))
}
```

### 3. 並發模型深度對比

#### Node.js：事件循環 + 回調/Promise

```javascript
// Node.js 並發模型
const http = require('http');

// 方式 1: 回調地獄
function getUserData(userId, callback) {
    http.get(`http://api.example.com/user/${userId}`, (res) => {
        let data = '';
        res.on('data', chunk => data += chunk);
        res.on('end', () => {
            const user = JSON.parse(data);
            // 需要再獲取用戶的訂單
            http.get(`http://api.example.com/orders/${user.id}`, (res2) => {
                let orders = '';
                res2.on('data', chunk => orders += chunk);
                res2.on('end', () => callback(null, {user, orders: JSON.parse(orders)}));
            });
        });
    });
}

// 方式 2: Promise/Async-Await
async function getUserDataModern(userId) {
    const userResponse = await fetch(`http://api.example.com/user/${userId}`);
    const user = await userResponse.json();

    const ordersResponse = await fetch(`http://api.example.com/orders/${user.id}`);
    const orders = await ordersResponse.json();

    return { user, orders };
}

// 並發執行多個請求
async function getMultipleUsers(userIds) {
    const users = await Promise.all(
        userIds.map(id => getUserDataModern(id))
    );
    return users;
}
```

#### Go：Goroutines + Channels

```go
package main

import (
    "encoding/json"
    "fmt"
    "io"
    "net/http"
    "sync"
)

type User struct {
    ID   int    `json:"id"`
    Name string `json:"name"`
}

type Order struct {
    ID     int `json:"id"`
    UserID int `json:"user_id"`
}

type UserData struct {
    User   User
    Orders []Order
}

// Go 的方式：清晰的順序執行
func getUserData(userId int) (*UserData, error) {
    // 獲取用戶資料
    resp, err := http.Get(fmt.Sprintf("http://api.example.com/user/%d", userId))
    if err != nil {
        return nil, err
    }
    defer resp.Body.Close()

    body, _ := io.ReadAll(resp.Body)
    var user User
    json.Unmarshal(body, &user)

    // 獲取訂單資料
    resp2, err := http.Get(fmt.Sprintf("http://api.example.com/orders/%d", user.ID))
    if err != nil {
        return nil, err
    }
    defer resp2.Body.Close()

    body2, _ := io.ReadAll(resp2.Body)
    var orders []Order
    json.Unmarshal(body2, &orders)

    return &UserData{User: user, Orders: orders}, nil
}

// 並發執行多個請求 - 使用 Goroutines
func getMultipleUsers(userIds []int) []*UserData {
    var wg sync.WaitGroup
    results := make([]*UserData, len(userIds))

    for i, id := range userIds {
        wg.Add(1)
        go func(index, userId int) {
            defer wg.Done()
            results[index], _ = getUserData(userId)
        }(i, id)
    }

    wg.Wait()
    return results
}

// 使用 Channels 的方式
func getMultipleUsersWithChannels(userIds []int) []*UserData {
    ch := make(chan *UserData, len(userIds))

    for _, id := range userIds {
        go func(userId int) {
            data, _ := getUserData(userId)
            ch <- data
        }(id)
    }

    results := make([]*UserData, 0, len(userIds))
    for i := 0; i < len(userIds); i++ {
        results = append(results, <-ch)
    }

    return results
}
```

### 4. 類型系統對比

#### Node.js：動態類型

```javascript
// JavaScript - 動態類型，靈活但容易出錯
function processUser(user) {
    // 沒有類型檢查，運行時才發現錯誤
    console.log(user.name.toUpperCase());
    // 如果 user.name 是 undefined，程序會崩潰
}

// TypeScript 改善了這個問題，但仍需要編譯步驟
interface User {
    id: number;
    name: string;
    email: string;
}

function processUserTS(user: User) {
    console.log(user.name.toUpperCase()); // 編譯時檢查
}
```

#### Go：靜態類型

```go
package main

import (
    "fmt"
    "strings"
)

// Go - 靜態類型，編譯時就能發現錯誤
type User struct {
    ID    int
    Name  string
    Email string
}

func processUser(user User) {
    fmt.Println(strings.ToUpper(user.Name))
    // 如果傳入的不是 User 類型，編譯就會失敗
}

func main() {
    user := User{
        ID:    1,
        Name:  "John",
        Email: "john@example.com",
    }

    processUser(user) // 類型安全

    // processUser("wrong type") // 編譯錯誤！
}
```

### 5. 錯誤處理對比

#### Node.js：Try-Catch + Error-First Callbacks

```javascript
// Node.js 錯誤處理
// 方式 1: Error-first callbacks
fs.readFile('file.txt', (err, data) => {
    if (err) {
        console.error('Error reading file:', err);
        return;
    }
    console.log(data);
});

// 方式 2: Try-Catch with Async/Await
async function readFile() {
    try {
        const data = await fs.promises.readFile('file.txt');
        console.log(data);
    } catch (err) {
        console.error('Error reading file:', err);
    }
}

// 問題：容易忘記處理錯誤
async function dangerousCode() {
    const data = await fs.promises.readFile('file.txt'); // 沒有 try-catch!
    // 如果文件不存在，會導致未捕獲的 Promise rejection
}
```

#### Go：顯式錯誤返回

```go
package main

import (
    "fmt"
    "os"
)

// Go 的錯誤處理 - 顯式且無法忽略
func readFile() {
    data, err := os.ReadFile("file.txt")
    if err != nil {
        fmt.Println("Error reading file:", err)
        return
    }
    fmt.Println(string(data))
}

// 如果你忘記處理錯誤，編譯器會警告
func dangerousCode() {
    data, _ := os.ReadFile("file.txt") // 使用 _ 忽略錯誤（不推薦）
    fmt.Println(string(data))
}

// 更好的做法：總是檢查錯誤
func safeCode() error {
    data, err := os.ReadFile("file.txt")
    if err != nil {
        return fmt.Errorf("failed to read file: %w", err)
    }
    fmt.Println(string(data))
    return nil
}
```

### 6. 部署與運維對比

#### Node.js 部署

```bash
# Node.js 部署流程
1. 確保服務器有正確版本的 Node.js
2. 上傳代碼和 package.json
3. 運行 npm install（需要網絡，耗時）
4. 可能需要編譯原生模組
5. 配置環境變量
6. 使用 PM2 或其他進程管理器運行

# 文件結構
my-app/
├── node_modules/     (可能有幾百MB)
├── src/
├── package.json
└── package-lock.json

# 啟動命令
pm2 start app.js
```

#### Go 部署

```bash
# Go 部署流程
1. 在本地編譯為目標平台的二進制文件
   GOOS=linux GOARCH=amd64 go build -o myapp
2. 上傳單個可執行文件
3. 運行即可！

# 文件結構
my-app       (單一文件，5-20MB)

# 啟動命令
./my-app

# 或使用 systemd
systemctl start myapp
```

**Go 部署示例：**

```go
// main.go - 完整的 Web 服務器
package main

import (
    "fmt"
    "log"
    "net/http"
)

func main() {
    http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintf(w, "Hello, World!")
    })

    log.Println("Server starting on :8080")
    log.Fatal(http.ListenAndServe(":8080", nil))
}

// 編譯：go build -o server
// 運行：./server
// 就這麼簡單！沒有依賴，沒有 node_modules
```

### 7. 生態系統對比

#### Node.js 生態系統

**優勢：**
- NPM 擁有超過 200 萬個包
- 前端開發的首選（React, Vue, Angular）
- 豐富的工具鏈（Webpack, Babel, ESLint）
- 大量的教程和社區支持

**劣勢：**
- 包質量參差不齊
- 依賴地獄（node_modules 過大）
- 版本碎片化嚴重
- 左墊問題（left-pad incident）

#### Go 生態系統

**優勢：**
- 標準庫非常強大（HTTP、JSON、加密等）
- 包管理簡單清晰（go.mod）
- 向後兼容性好
- 專注於雲原生和微服務

**劣勢：**
- 包的數量較少
- 某些領域（如前端）不適用
- Web 框架選擇較少（但標準庫已經很強）

```go
// Go 標準庫就能做很多事
package main

import (
    "encoding/json"
    "net/http"
    "log"
)

type Response struct {
    Message string `json:"message"`
    Status  int    `json:"status"`
}

func main() {
    // 不需要 Express！標準庫就夠了
    http.HandleFunc("/api/hello", func(w http.ResponseWriter, r *http.Request) {
        w.Header().Set("Content-Type", "application/json")
        json.NewEncoder(w).Encode(Response{
            Message: "Hello from Go!",
            Status:  200,
        })
    })

    log.Fatal(http.ListenAndServe(":8080", nil))
}
```

## 何時選擇 Go？何時選擇 Node.js？

### 選擇 Go 的場景

1. **微服務架構**
   - 快速啟動時間
   - 小的內存佔用
   - 容器化友好

2. **高並發服務**
   - WebSocket 服務器
   - 實時數據處理
   - API 網關

3. **系統工具**
   - CLI 工具
   - DevOps 工具
   - 網絡工具

4. **性能關鍵型應用**
   - 高頻交易系統
   - 遊戲服務器
   - 視頻處理

### 選擇 Node.js 的場景

1. **全棧 JavaScript 開發**
   - 前後端共享代碼
   - 團隊只懂 JavaScript

2. **快速原型開發**
   - MVP 開發
   - 快速迭代

3. **I/O 密集型應用**
   - 聊天應用
   - 實時協作工具

4. **前端工具鏈**
   - 構建工具
   - 開發服務器

## 實際案例對比

### 案例 1：簡單的 REST API

**Node.js + Express:**

```javascript
// Node.js - app.js
const express = require('express');
const app = express();

app.use(express.json());

let users = [
    { id: 1, name: 'Alice', email: 'alice@example.com' },
    { id: 2, name: 'Bob', email: 'bob@example.com' }
];

app.get('/users', (req, res) => {
    res.json(users);
});

app.get('/users/:id', (req, res) => {
    const user = users.find(u => u.id === parseInt(req.params.id));
    if (!user) return res.status(404).json({ error: 'User not found' });
    res.json(user);
});

app.post('/users', (req, res) => {
    const user = {
        id: users.length + 1,
        name: req.body.name,
        email: req.body.email
    };
    users.push(user);
    res.status(201).json(user);
});

app.listen(3000, () => {
    console.log('Server running on port 3000');
});

// 啟動：node app.js
// 依賴：需要 npm install express
```

**Go:**

```go
// Go - main.go
package main

import (
    "encoding/json"
    "fmt"
    "log"
    "net/http"
    "strconv"
    "strings"
)

type User struct {
    ID    int    `json:"id"`
    Name  string `json:"name"`
    Email string `json:"email"`
}

var users = []User{
    {ID: 1, Name: "Alice", Email: "alice@example.com"},
    {ID: 2, Name: "Bob", Email: "bob@example.com"},
}

func getUsers(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(users)
}

func getUser(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "application/json")

    // 提取 ID
    parts := strings.Split(r.URL.Path, "/")
    if len(parts) < 3 {
        http.Error(w, "Invalid request", http.StatusBadRequest)
        return
    }

    id, err := strconv.Atoi(parts[2])
    if err != nil {
        http.Error(w, "Invalid ID", http.StatusBadRequest)
        return
    }

    for _, user := range users {
        if user.ID == id {
            json.NewEncoder(w).Encode(user)
            return
        }
    }

    http.Error(w, "User not found", http.StatusNotFound)
}

func createUser(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "application/json")

    var user User
    if err := json.NewDecoder(r.Body).Decode(&user); err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }

    user.ID = len(users) + 1
    users = append(users, user)

    w.WriteHeader(http.StatusCreated)
    json.NewEncoder(w).Encode(user)
}

func usersHandler(w http.ResponseWriter, r *http.Request) {
    switch r.Method {
    case "GET":
        if r.URL.Path == "/users" {
            getUsers(w, r)
        } else {
            getUser(w, r)
        }
    case "POST":
        createUser(w, r)
    default:
        http.Error(w, "Method not allowed", http.StatusMethodNotAllowed)
    }
}

func main() {
    http.HandleFunc("/users", usersHandler)
    http.HandleFunc("/users/", usersHandler)

    fmt.Println("Server running on port 3000")
    log.Fatal(http.ListenAndServe(":3000", nil))
}

// 編譯：go build -o server
// 啟動：./server
// 依賴：無（使用標準庫）
```

## 重點總結

### Go 的主要優勢

1. **性能優越**
   - 編譯為原生機器碼
   - 高效的垃圾回收
   - 真正的並行執行

2. **並發模型簡單**
   - Goroutines 輕量級
   - Channels 通信清晰
   - 避免回調地獄

3. **部署簡單**
   - 單一二進制文件
   - 無運行時依賴
   - 跨平台編譯

4. **類型安全**
   - 編譯時捕獲錯誤
   - 代碼更可維護
   - 重構更安全

5. **快速啟動**
   - 適合微服務
   - 容器化友好
   - 雲原生首選

### Node.js 的主要優勢

1. **生態系統豐富**
   - NPM 包數量龐大
   - 工具鏈完善

2. **前端開發**
   - JavaScript 全棧
   - 代碼共享

3. **學習曲線平緩**
   - 語法靈活
   - 動態類型

4. **快速開發**
   - 原型開發快速
   - 熱重載方便

### 為什麼 Node.js 開發者應該學習 Go？

1. **職業發展**：Go 是雲原生和微服務領域的主流語言
2. **性能提升**：某些場景下 Go 性能是 Node.js 的 10 倍以上
3. **系統編程**：Go 讓你能開發更底層的系統工具
4. **並發編程**：學習不同的並發模型，擴展思維
5. **類型系統**：體驗靜態類型帶來的好處

## 練習題

### 基礎題

1. **對比分析**：列出你當前 Node.js 項目中，哪些部分適合用 Go 重寫？為什麼？

2. **性能測試**：
   - 用 Node.js 和 Go 分別實現一個計算斐波那契數列的 HTTP API
   - 使用 Apache Bench 或 wrk 測試性能
   - 對比並分析結果

3. **並發理解**：
   - 解釋 Node.js 的事件循環和 Go 的 Goroutines 有什麼本質區別？
   - 各自適合什麼場景？

### 進階題

4. **實戰項目**：
   - 用 Node.js 和 Go 分別實現一個簡單的文件上傳服務
   - 支持多文件同時上傳
   - 對比代碼量、性能、內存使用

5. **架構設計**：
   設計一個電商系統的微服務架構：
   - 哪些服務應該用 Go？
   - 哪些服務應該用 Node.js？
   - 給出你的理由

6. **錯誤處理**：
   - 對比 Node.js 和 Go 的錯誤處理機制
   - 實現一個包含多層函數調用的錯誤處理示例
   - 討論各自的優缺點

### 思考題

7. **類型系統**：
   - TypeScript 和 Go 的類型系統有什麼區別？
   - 為什麼 Go 不需要像 TypeScript 那樣的編譯步驟？

8. **生態系統**：
   - Go 的標準庫為什麼比 Node.js 強大？
   - 這對包生態系統有什麼影響？

9. **職業規劃**：
   - 作為 Node.js 開發者，學習 Go 能帶來什麼價值？
   - 你會如何在團隊中引入 Go？

## 下一章預告

下一章我們將學習如何安裝和配置 Go 開發環境，包括：
- Windows/Mac/Linux 系統的安裝步驟
- VS Code / GoLand 的配置
- 對比 nvm/node 的安裝體驗
- Go 工作空間的概念
- 第一個 Hello World 程序

現在，你應該對為什麼選擇 Go 有了清晰的認識。讓我們開始這段學習之旅吧！
