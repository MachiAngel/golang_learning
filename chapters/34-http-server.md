# Chapter 34: net/http 服務器

## 概述

`net/http` 是 Go 標準庫中用於構建 HTTP 服務器和客戶端的包。對於 Node.js 開發者來說，它相當於內置的 `http` 模塊，但提供了更簡潔的 API 和更好的性能。與 Express.js 不同，Go 的標準庫就足以構建生產級的 Web 服務。

## Node.js 與 Go 的對比

| 功能 | Node.js (Express) | Go (net/http) |
|------|------------------|---------------|
| 創建服務器 | `const app = express()` | `http.Server{}` |
| 路由註冊 | `app.get('/path', handler)` | `http.HandleFunc('/path', handler)` |
| 啟動服務器 | `app.listen(port)` | `http.ListenAndServe(addr, handler)` |
| 獲取參數 | `req.params`, `req.query` | 手動解析或使用 router |
| 發送響應 | `res.send()`, `res.json()` | `fmt.Fprintf(w, ...)`, `json.NewEncoder(w).Encode()` |
| 中間件 | `app.use(middleware)` | 手動包裝 handler |
| 靜態文件 | `express.static()` | `http.FileServer()` |
| 錯誤處理 | `app.use(errorHandler)` | 在 handler 中處理 |

## 詳細概念解釋

### 1. Handler 接口

Go 的 HTTP 處理基於 `Handler` 接口：

```go
type Handler interface {
    ServeHTTP(ResponseWriter, *Request)
}
```

任何實現了 `ServeHTTP` 方法的類型都可以作為 HTTP 處理器。

### 2. HandlerFunc

`HandlerFunc` 是一個函數類型，實現了 `Handler` 接口：

```go
type HandlerFunc func(ResponseWriter, *Request)
```

這讓我們可以直接使用函數作為處理器。

### 3. ServeMux

`ServeMux` 是 Go 的路由多路復用器（類似 Express 的 router）：
- 路由匹配基於最長前綴
- 支持 `/` 結尾的路徑作為子樹
- 不支持路徑參數（需要第三方庫）

### 4. Request 和 ResponseWriter

- `*http.Request` - 包含請求信息（方法、URL、Header、Body 等）
- `http.ResponseWriter` - 用於寫入響應的接口

## 實際代碼示例

### 示例 1: 最簡單的 HTTP 服務器

**Go:**
```go
package main

import (
    "fmt"
    "log"
    "net/http"
)

func helloHandler(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "Hello, World!")
}

func main() {
    // 註冊處理器
    http.HandleFunc("/", helloHandler)

    // 啟動服務器
    fmt.Println("Server starting on :8080")
    if err := http.ListenAndServe(":8080", nil); err != nil {
        log.Fatal(err)
    }
}
```

**Node.js (原生 http):**
```javascript
const http = require('http');

const server = http.createServer((req, res) => {
    if (req.url === '/') {
        res.writeHead(200, {'Content-Type': 'text/plain'});
        res.end('Hello, World!');
    }
});

server.listen(8080, () => {
    console.log('Server starting on :8080');
});
```

**Node.js (Express):**
```javascript
const express = require('express');
const app = express();

app.get('/', (req, res) => {
    res.send('Hello, World!');
});

app.listen(8080, () => {
    console.log('Server starting on :8080');
});
```

### 示例 2: 多個路由

**Go:**
```go
package main

import (
    "fmt"
    "log"
    "net/http"
)

func homeHandler(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "Welcome to Home Page")
}

func aboutHandler(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "About Us")
}

func contactHandler(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "Contact: info@example.com")
}

// 404 處理器
func notFoundHandler(w http.ResponseWriter, r *http.Request) {
    w.WriteHeader(http.StatusNotFound)
    fmt.Fprintf(w, "404 - Page Not Found")
}

func main() {
    // 註冊路由
    http.HandleFunc("/", homeHandler)
    http.HandleFunc("/about", aboutHandler)
    http.HandleFunc("/contact", contactHandler)

    // 注意：Go 的默認 ServeMux 會自動處理 404
    // 如果想自定義 404，需要使用自定義 ServeMux

    fmt.Println("Server starting on :8080")
    log.Fatal(http.ListenAndServe(":8080", nil))
}
```

**Node.js (Express):**
```javascript
const express = require('express');
const app = express();

app.get('/', (req, res) => {
    res.send('Welcome to Home Page');
});

app.get('/about', (req, res) => {
    res.send('About Us');
});

app.get('/contact', (req, res) => {
    res.send('Contact: info@example.com');
});

// 404 處理器
app.use((req, res) => {
    res.status(404).send('404 - Page Not Found');
});

app.listen(8080, () => {
    console.log('Server starting on :8080');
});
```

### 示例 3: 處理不同 HTTP 方法

**Go:**
```go
package main

import (
    "fmt"
    "io"
    "log"
    "net/http"
)

func userHandler(w http.ResponseWriter, r *http.Request) {
    switch r.Method {
    case http.MethodGet:
        fmt.Fprintf(w, "GET: Get user information")

    case http.MethodPost:
        body, _ := io.ReadAll(r.Body)
        fmt.Fprintf(w, "POST: Create user with data: %s", string(body))

    case http.MethodPut:
        body, _ := io.ReadAll(r.Body)
        fmt.Fprintf(w, "PUT: Update user with data: %s", string(body))

    case http.MethodDelete:
        fmt.Fprintf(w, "DELETE: Delete user")

    default:
        w.WriteHeader(http.StatusMethodNotAllowed)
        fmt.Fprintf(w, "Method %s not allowed", r.Method)
    }
}

func main() {
    http.HandleFunc("/user", userHandler)

    fmt.Println("Server starting on :8080")
    log.Fatal(http.ListenAndServe(":8080", nil))
}
```

**Node.js (Express):**
```javascript
const express = require('express');
const app = express();

app.use(express.json());
app.use(express.text());

app.route('/user')
    .get((req, res) => {
        res.send('GET: Get user information');
    })
    .post((req, res) => {
        res.send(`POST: Create user with data: ${JSON.stringify(req.body)}`);
    })
    .put((req, res) => {
        res.send(`PUT: Update user with data: ${JSON.stringify(req.body)}`);
    })
    .delete((req, res) => {
        res.send('DELETE: Delete user');
    });

app.listen(8080, () => {
    console.log('Server starting on :8080');
});
```

### 示例 4: JSON 響應

**Go:**
```go
package main

import (
    "encoding/json"
    "log"
    "net/http"
)

type User struct {
    ID    int    `json:"id"`
    Name  string `json:"name"`
    Email string `json:"email"`
}

type Response struct {
    Success bool        `json:"success"`
    Data    interface{} `json:"data,omitempty"`
    Error   string      `json:"error,omitempty"`
}

func jsonResponse(w http.ResponseWriter, status int, data interface{}) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(status)
    json.NewEncoder(w).Encode(data)
}

func getUserHandler(w http.ResponseWriter, r *http.Request) {
    user := User{
        ID:    1,
        Name:  "Alice",
        Email: "alice@example.com",
    }

    response := Response{
        Success: true,
        Data:    user,
    }

    jsonResponse(w, http.StatusOK, response)
}

func createUserHandler(w http.ResponseWriter, r *http.Request) {
    if r.Method != http.MethodPost {
        response := Response{
            Success: false,
            Error:   "Method not allowed",
        }
        jsonResponse(w, http.StatusMethodNotAllowed, response)
        return
    }

    var user User
    if err := json.NewDecoder(r.Body).Decode(&user); err != nil {
        response := Response{
            Success: false,
            Error:   "Invalid JSON",
        }
        jsonResponse(w, http.StatusBadRequest, response)
        return
    }

    // 模擬創建用戶
    user.ID = 42

    response := Response{
        Success: true,
        Data:    user,
    }

    jsonResponse(w, http.StatusCreated, response)
}

func main() {
    http.HandleFunc("/api/user", getUserHandler)
    http.HandleFunc("/api/users", createUserHandler)

    log.Println("Server starting on :8080")
    log.Fatal(http.ListenAndServe(":8080", nil))
}
```

**Node.js (Express):**
```javascript
const express = require('express');
const app = express();

app.use(express.json());

app.get('/api/user', (req, res) => {
    const user = {
        id: 1,
        name: 'Alice',
        email: 'alice@example.com'
    };

    res.json({
        success: true,
        data: user
    });
});

app.post('/api/users', (req, res) => {
    const user = req.body;

    if (!user.name || !user.email) {
        return res.status(400).json({
            success: false,
            error: 'Invalid JSON'
        });
    }

    // 模擬創建用戶
    user.id = 42;

    res.status(201).json({
        success: true,
        data: user
    });
});

app.listen(8080, () => {
    console.log('Server starting on :8080');
});
```

### 示例 5: 查詢參數和表單數據

**Go:**
```go
package main

import (
    "fmt"
    "log"
    "net/http"
)

func searchHandler(w http.ResponseWriter, r *http.Request) {
    // 解析查詢參數
    query := r.URL.Query()

    // 獲取單個參數
    keyword := query.Get("q")
    page := query.Get("page")

    // 獲取多個值的參數
    tags := query["tag"] // []string

    fmt.Fprintf(w, "Search keyword: %s\n", keyword)
    fmt.Fprintf(w, "Page: %s\n", page)
    fmt.Fprintf(w, "Tags: %v\n", tags)
}

func loginHandler(w http.ResponseWriter, r *http.Request) {
    if r.Method == http.MethodGet {
        // 顯示登錄表單
        html := `
            <html>
                <body>
                    <form method="POST">
                        <input type="text" name="username" placeholder="Username">
                        <input type="password" name="password" placeholder="Password">
                        <button type="submit">Login</button>
                    </form>
                </body>
            </html>
        `
        w.Header().Set("Content-Type", "text/html")
        fmt.Fprint(w, html)
        return
    }

    if r.Method == http.MethodPost {
        // 解析表單數據
        if err := r.ParseForm(); err != nil {
            http.Error(w, "Parse form error", http.StatusBadRequest)
            return
        }

        username := r.FormValue("username")
        password := r.FormValue("password")

        fmt.Fprintf(w, "Username: %s\n", username)
        fmt.Fprintf(w, "Password: %s\n", password)
        return
    }

    http.Error(w, "Method not allowed", http.StatusMethodNotAllowed)
}

func main() {
    http.HandleFunc("/search", searchHandler)
    http.HandleFunc("/login", loginHandler)

    log.Println("Server starting on :8080")
    log.Fatal(http.ListenAndServe(":8080", nil))
}

// 訪問: http://localhost:8080/search?q=golang&page=1&tag=backend&tag=web
```

**Node.js (Express):**
```javascript
const express = require('express');
const app = express();

app.use(express.urlencoded({ extended: true }));
app.use(express.json());

app.get('/search', (req, res) => {
    // 獲取查詢參數
    const { q: keyword, page } = req.query;
    const tags = Array.isArray(req.query.tag) ? req.query.tag : [req.query.tag].filter(Boolean);

    res.send(`Search keyword: ${keyword}\nPage: ${page}\nTags: ${tags.join(', ')}`);
});

app.get('/login', (req, res) => {
    // 顯示登錄表單
    res.send(`
        <html>
            <body>
                <form method="POST" action="/login">
                    <input type="text" name="username" placeholder="Username">
                    <input type="password" name="password" placeholder="Password">
                    <button type="submit">Login</button>
                </form>
            </body>
        </html>
    `);
});

app.post('/login', (req, res) => {
    const { username, password } = req.body;
    res.send(`Username: ${username}\nPassword: ${password}`);
});

app.listen(8080, () => {
    console.log('Server starting on :8080');
});

// 訪問: http://localhost:8080/search?q=golang&page=1&tag=backend&tag=web
```

### 示例 6: 靜態文件服務

**Go:**
```go
package main

import (
    "log"
    "net/http"
)

func main() {
    // 方法 1: 使用 FileServer 服務整個目錄
    fs := http.FileServer(http.Dir("./static"))
    http.Handle("/static/", http.StripPrefix("/static/", fs))

    // 方法 2: 服務單個文件
    http.HandleFunc("/favicon.ico", func(w http.ResponseWriter, r *http.Request) {
        http.ServeFile(w, r, "./static/favicon.ico")
    })

    // 首頁
    http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        http.ServeFile(w, r, "./static/index.html")
    })

    log.Println("Server starting on :8080")
    log.Println("Static files: http://localhost:8080/static/")
    log.Fatal(http.ListenAndServe(":8080", nil))
}

// 目錄結構:
// .
// ├── main.go
// └── static/
//     ├── index.html
//     ├── favicon.ico
//     ├── css/
//     │   └── style.css
//     └── js/
//         └── app.js
```

**Node.js (Express):**
```javascript
const express = require('express');
const path = require('path');
const app = express();

// 服務靜態文件
app.use('/static', express.static('static'));

// 服務單個文件
app.get('/favicon.ico', (req, res) => {
    res.sendFile(path.join(__dirname, 'static', 'favicon.ico'));
});

// 首頁
app.get('/', (req, res) => {
    res.sendFile(path.join(__dirname, 'static', 'index.html'));
});

app.listen(8080, () => {
    console.log('Server starting on :8080');
    console.log('Static files: http://localhost:8080/static/');
});

// 目錄結構相同
```

### 示例 7: 自定義 ServeMux 和 404 處理

**Go:**
```go
package main

import (
    "fmt"
    "log"
    "net/http"
)

type customHandler struct {
    mux *http.ServeMux
}

func (h *customHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    // 記錄請求
    log.Printf("%s %s", r.Method, r.URL.Path)

    // 創建自定義 ResponseWriter 來捕獲狀態碼
    rw := &responseWriter{ResponseWriter: w, statusCode: http.StatusOK}

    // 調用實際的處理器
    h.mux.ServeHTTP(rw, r)

    // 如果是 404，使用自定義處理
    if rw.statusCode == http.StatusNotFound {
        w.Header().Set("Content-Type", "text/html")
        fmt.Fprintf(w, "<h1>404 - Page Not Found</h1><p>The page %s does not exist.</p>", r.URL.Path)
    }
}

type responseWriter struct {
    http.ResponseWriter
    statusCode int
}

func (rw *responseWriter) WriteHeader(code int) {
    rw.statusCode = code
    rw.ResponseWriter.WriteHeader(code)
}

func main() {
    mux := http.NewServeMux()

    // 註冊路由
    mux.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintf(w, "Home Page")
    })

    mux.HandleFunc("/about", func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintf(w, "About Page")
    })

    // 包裝 mux
    handler := &customHandler{mux: mux}

    log.Println("Server starting on :8080")
    log.Fatal(http.ListenAndServe(":8080", handler))
}
```

**Node.js (Express):**
```javascript
const express = require('express');
const app = express();

// 請求日誌中間件
app.use((req, res, next) => {
    console.log(`${req.method} ${req.url}`);
    next();
});

// 路由
app.get('/', (req, res) => {
    res.send('Home Page');
});

app.get('/about', (req, res) => {
    res.send('About Page');
});

// 自定義 404 處理
app.use((req, res) => {
    res.status(404).send(`
        <h1>404 - Page Not Found</h1>
        <p>The page ${req.url} does not exist.</p>
    `);
});

app.listen(8080, () => {
    console.log('Server starting on :8080');
});
```

### 示例 8: 完整的 Web 服務器

**Go:**
```go
package main

import (
    "encoding/json"
    "log"
    "net/http"
    "time"
)

type Server struct {
    mux *http.ServeMux
}

func NewServer() *Server {
    s := &Server{
        mux: http.NewServeMux(),
    }
    s.routes()
    return s
}

func (s *Server) routes() {
    // API 路由
    s.mux.HandleFunc("/api/health", s.healthHandler)
    s.mux.HandleFunc("/api/time", s.timeHandler)
    s.mux.HandleFunc("/api/echo", s.echoHandler)

    // 靜態文件
    fs := http.FileServer(http.Dir("./public"))
    s.mux.Handle("/", fs)
}

func (s *Server) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    // 日誌中間件
    start := time.Now()
    log.Printf("[%s] %s %s", r.Method, r.URL.Path, r.RemoteAddr)

    // CORS 中間件
    w.Header().Set("Access-Control-Allow-Origin", "*")
    w.Header().Set("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE, OPTIONS")
    w.Header().Set("Access-Control-Allow-Headers", "Content-Type")

    if r.Method == "OPTIONS" {
        w.WriteHeader(http.StatusOK)
        return
    }

    // 調用實際處理器
    s.mux.ServeHTTP(w, r)

    // 記錄響應時間
    log.Printf("Completed in %v", time.Since(start))
}

func (s *Server) healthHandler(w http.ResponseWriter, r *http.Request) {
    response := map[string]interface{}{
        "status": "healthy",
        "time":   time.Now().Format(time.RFC3339),
    }
    s.jsonResponse(w, http.StatusOK, response)
}

func (s *Server) timeHandler(w http.ResponseWriter, r *http.Request) {
    response := map[string]interface{}{
        "time": time.Now().Unix(),
    }
    s.jsonResponse(w, http.StatusOK, response)
}

func (s *Server) echoHandler(w http.ResponseWriter, r *http.Request) {
    if r.Method != http.MethodPost {
        http.Error(w, "Method not allowed", http.StatusMethodNotAllowed)
        return
    }

    var data map[string]interface{}
    if err := json.NewDecoder(r.Body).Decode(&data); err != nil {
        http.Error(w, "Invalid JSON", http.StatusBadRequest)
        return
    }

    s.jsonResponse(w, http.StatusOK, data)
}

func (s *Server) jsonResponse(w http.ResponseWriter, status int, data interface{}) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(status)
    json.NewEncoder(w).Encode(data)
}

func main() {
    server := NewServer()

    httpServer := &http.Server{
        Addr:         ":8080",
        Handler:      server,
        ReadTimeout:  15 * time.Second,
        WriteTimeout: 15 * time.Second,
        IdleTimeout:  60 * time.Second,
    }

    log.Println("Server starting on :8080")
    log.Fatal(httpServer.ListenAndServe())
}
```

**Node.js (Express):**
```javascript
const express = require('express');
const cors = require('cors');
const app = express();

// 中間件
app.use(express.json());
app.use(cors());

// 日誌中間件
app.use((req, res, next) => {
    const start = Date.now();
    console.log(`[${req.method}] ${req.url} ${req.ip}`);

    res.on('finish', () => {
        const duration = Date.now() - start;
        console.log(`Completed in ${duration}ms`);
    });

    next();
});

// API 路由
app.get('/api/health', (req, res) => {
    res.json({
        status: 'healthy',
        time: new Date().toISOString()
    });
});

app.get('/api/time', (req, res) => {
    res.json({
        time: Math.floor(Date.now() / 1000)
    });
});

app.post('/api/echo', (req, res) => {
    res.json(req.body);
});

// 靜態文件
app.use(express.static('public'));

// 啟動服務器
const server = app.listen(8080, () => {
    console.log('Server starting on :8080');
});

// 設置超時
server.timeout = 15000;
server.keepAliveTimeout = 60000;
```

## 重點總結

1. **Handler 模式**
   - `http.Handler` 接口是核心
   - `http.HandlerFunc` 讓函數成為處理器
   - 任何實現 `ServeHTTP` 的類型都可以是處理器

2. **路由註冊**
   - `http.HandleFunc()` - 註冊函數處理器
   - `http.Handle()` - 註冊 Handler 接口
   - `http.ServeMux` - 路由多路復用器

3. **請求處理**
   - `r.Method` - HTTP 方法
   - `r.URL.Query()` - 查詢參數
   - `r.FormValue()` - 表單數據
   - `json.NewDecoder(r.Body)` - JSON 數據

4. **響應處理**
   - `w.WriteHeader()` - 設置狀態碼
   - `w.Header().Set()` - 設置響應頭
   - `fmt.Fprintf(w, ...)` - 寫入文本
   - `json.NewEncoder(w).Encode()` - 寫入 JSON

5. **與 Express 的主要差異**
   - Go 需要手動處理不同 HTTP 方法
   - Go 沒有內置的路徑參數解析
   - Go 的中間件需要手動實現
   - Go 的錯誤處理更明確
   - Go 標準庫就足以構建生產級服務

6. **最佳實踐**
   - 使用自定義 ServeMux 而不是默認的
   - 總是設置超時
   - 實現請求日誌
   - 統一錯誤處理
   - 使用結構體組織路由

## 練習題

### 練習 1: RESTful API
創建簡單的 RESTful API，支持 CRUD 操作。

### 練習 2: 中間件鏈
實現中間件鏈，包括日誌、認證和 CORS。

### 練習 3: 文件上傳
實現文件上傳處理器。

### 練習 4: WebSocket
使用 gorilla/websocket 實現 WebSocket 服務器。

### 練習 5: 模板渲染
使用 html/template 渲染 HTML 頁面。

### 練習 6: 會話管理
實現簡單的會話管理（使用 cookie）。

### 練習 7: 速率限制
實現 API 速率限制中間件。

### 練習 8: 優雅關閉
實現支持優雅關閉的 HTTP 服務器。

## 參考答案

### 練習 1 答案
```go
package main

import (
    "encoding/json"
    "log"
    "net/http"
    "strconv"
    "strings"
    "sync"
)

type Item struct {
    ID   int    `json:"id"`
    Name string `json:"name"`
}

type Store struct {
    sync.RWMutex
    items  map[int]Item
    nextID int
}

func NewStore() *Store {
    return &Store{
        items:  make(map[int]Item),
        nextID: 1,
    }
}

func (s *Store) itemsHandler(w http.ResponseWriter, r *http.Request) {
    switch r.Method {
    case http.MethodGet:
        s.getAllItems(w, r)
    case http.MethodPost:
        s.createItem(w, r)
    default:
        http.Error(w, "Method not allowed", http.StatusMethodNotAllowed)
    }
}

func (s *Store) itemHandler(w http.ResponseWriter, r *http.Request) {
    // 提取 ID
    parts := strings.Split(r.URL.Path, "/")
    if len(parts) < 4 {
        http.Error(w, "Invalid URL", http.StatusBadRequest)
        return
    }

    id, err := strconv.Atoi(parts[3])
    if err != nil {
        http.Error(w, "Invalid ID", http.StatusBadRequest)
        return
    }

    switch r.Method {
    case http.MethodGet:
        s.getItem(w, r, id)
    case http.MethodPut:
        s.updateItem(w, r, id)
    case http.MethodDelete:
        s.deleteItem(w, r, id)
    default:
        http.Error(w, "Method not allowed", http.StatusMethodNotAllowed)
    }
}

func (s *Store) getAllItems(w http.ResponseWriter, r *http.Request) {
    s.RLock()
    defer s.RUnlock()

    items := make([]Item, 0, len(s.items))
    for _, item := range s.items {
        items = append(items, item)
    }

    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(items)
}

func (s *Store) createItem(w http.ResponseWriter, r *http.Request) {
    var item Item
    if err := json.NewDecoder(r.Body).Decode(&item); err != nil {
        http.Error(w, "Invalid JSON", http.StatusBadRequest)
        return
    }

    s.Lock()
    item.ID = s.nextID
    s.nextID++
    s.items[item.ID] = item
    s.Unlock()

    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusCreated)
    json.NewEncoder(w).Encode(item)
}

func (s *Store) getItem(w http.ResponseWriter, r *http.Request, id int) {
    s.RLock()
    item, exists := s.items[id]
    s.RUnlock()

    if !exists {
        http.Error(w, "Item not found", http.StatusNotFound)
        return
    }

    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(item)
}

func (s *Store) updateItem(w http.ResponseWriter, r *http.Request, id int) {
    var item Item
    if err := json.NewDecoder(r.Body).Decode(&item); err != nil {
        http.Error(w, "Invalid JSON", http.StatusBadRequest)
        return
    }

    s.Lock()
    if _, exists := s.items[id]; !exists {
        s.Unlock()
        http.Error(w, "Item not found", http.StatusNotFound)
        return
    }

    item.ID = id
    s.items[id] = item
    s.Unlock()

    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(item)
}

func (s *Store) deleteItem(w http.ResponseWriter, r *http.Request, id int) {
    s.Lock()
    if _, exists := s.items[id]; !exists {
        s.Unlock()
        http.Error(w, "Item not found", http.StatusNotFound)
        return
    }

    delete(s.items, id)
    s.Unlock()

    w.WriteHeader(http.StatusNoContent)
}

func main() {
    store := NewStore()

    http.HandleFunc("/api/items", store.itemsHandler)
    http.HandleFunc("/api/items/", store.itemHandler)

    log.Println("Server starting on :8080")
    log.Fatal(http.ListenAndServe(":8080", nil))
}
```
