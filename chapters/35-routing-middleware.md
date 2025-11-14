# Chapter 35: 路由與中間件

## 概述

雖然 Go 的標準庫提供了基本的路由功能，但在實際項目中我們通常需要更強大的路由和中間件系統。本章將介紹如何在標準庫基礎上構建路由和中間件模式，以及如何使用流行的路由庫。

## Node.js 與 Go 的對比

| 功能 | Node.js (Express) | Go (標準庫) | Go (第三方) |
|------|------------------|------------|-------------|
| 路徑參數 | `/users/:id` | 手動解析 | `/users/{id}` (gorilla/mux) |
| 中間件 | `app.use(middleware)` | 手動包裝 Handler | 鏈式調用 |
| 中間件順序 | 註冊順序 | 包裝順序 | 鏈順序 |
| 路由組 | `app.use('/api', router)` | 手動前綴 | `router.PathPrefix()` |
| 子路由 | `Router()` | 多個 ServeMux | 子路由器 |

## 詳細概念解釋

### 1. 中間件模式

中間件是一個包裝 HTTP Handler 的函數：

```go
type Middleware func(http.Handler) http.Handler
```

中間件可以：
- 在請求前執行邏輯
- 修改請求
- 決定是否繼續處理
- 在響應後執行邏輯
- 修改響應

### 2. 中間件鏈

多個中間件可以組合成鏈：

```go
handler = middleware1(middleware2(middleware3(handler)))
```

### 3. 路由參數

路由參數允許從 URL 中提取動態部分：
- `/users/123` -> `id = 123`
- `/posts/golang/comments` -> `category = golang`

## 實際代碼示例

### 示例 1: 基本中間件

**Go:**
```go
package main

import (
    "fmt"
    "log"
    "net/http"
    "time"
)

// 中間件類型
type Middleware func(http.Handler) http.Handler

// 日誌中間件
func LoggingMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        log.Printf("[%s] %s %s", r.Method, r.URL.Path, r.RemoteAddr)

        next.ServeHTTP(w, r)

        log.Printf("Completed in %v", time.Since(start))
    })
}

// 恢復中間件（捕獲 panic）
func RecoveryMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        defer func() {
            if err := recover(); err != nil {
                log.Printf("Panic recovered: %v", err)
                http.Error(w, "Internal Server Error", http.StatusInternalServerError)
            }
        }()

        next.ServeHTTP(w, r)
    })
}

// CORS 中間件
func CORSMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        w.Header().Set("Access-Control-Allow-Origin", "*")
        w.Header().Set("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE, OPTIONS")
        w.Header().Set("Access-Control-Allow-Headers", "Content-Type, Authorization")

        if r.Method == "OPTIONS" {
            w.WriteHeader(http.StatusOK)
            return
        }

        next.ServeHTTP(w, r)
    })
}

// 應用中間件鏈
func Chain(handler http.Handler, middlewares ...Middleware) http.Handler {
    for i := len(middlewares) - 1; i >= 0; i-- {
        handler = middlewares[i](handler)
    }
    return handler
}

func main() {
    // 業務處理器
    handler := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintf(w, "Hello, World!")
    })

    // 應用中間件
    finalHandler := Chain(handler,
        LoggingMiddleware,
        RecoveryMiddleware,
        CORSMiddleware,
    )

    http.Handle("/", finalHandler)

    log.Println("Server starting on :8080")
    log.Fatal(http.ListenAndServe(":8080", nil))
}
```

**Node.js (Express):**
```javascript
const express = require('express');
const cors = require('cors');
const app = express();

// 日誌中間件
app.use((req, res, next) => {
    const start = Date.now();
    console.log(`[${req.method}] ${req.url} ${req.ip}`);

    res.on('finish', () => {
        console.log(`Completed in ${Date.now() - start}ms`);
    });

    next();
});

// 錯誤處理中間件
app.use((err, req, res, next) => {
    console.error('Error:', err);
    res.status(500).send('Internal Server Error');
});

// CORS 中間件
app.use(cors());

// 業務路由
app.get('/', (req, res) => {
    res.send('Hello, World!');
});

app.listen(8080, () => {
    console.log('Server starting on :8080');
});
```

### 示例 2: 認證中間件

**Go:**
```go
package main

import (
    "context"
    "fmt"
    "log"
    "net/http"
    "strings"
)

type contextKey string

const userContextKey contextKey = "user"

// 簡單的 Token 認證中間件
func AuthMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // 獲取 Authorization header
        auth := r.Header.Get("Authorization")
        if auth == "" {
            http.Error(w, "Unauthorized", http.StatusUnauthorized)
            return
        }

        // 解析 Bearer token
        parts := strings.Split(auth, " ")
        if len(parts) != 2 || parts[0] != "Bearer" {
            http.Error(w, "Invalid authorization header", http.StatusUnauthorized)
            return
        }

        token := parts[1]

        // 驗證 token（簡化示例）
        if token != "valid-token-123" {
            http.Error(w, "Invalid token", http.StatusUnauthorized)
            return
        }

        // 將用戶信息存入 context
        user := map[string]string{
            "id":   "123",
            "name": "Alice",
        }
        ctx := context.WithValue(r.Context(), userContextKey, user)
        r = r.WithContext(ctx)

        next.ServeHTTP(w, r)
    })
}

// 從 context 獲取用戶信息
func GetUser(r *http.Request) map[string]string {
    user, ok := r.Context().Value(userContextKey).(map[string]string)
    if !ok {
        return nil
    }
    return user
}

func protectedHandler(w http.ResponseWriter, r *http.Request) {
    user := GetUser(r)
    fmt.Fprintf(w, "Hello, %s! Your ID is %s", user["name"], user["id"])
}

func main() {
    // 公開端點
    http.HandleFunc("/public", func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintf(w, "This is public")
    })

    // 受保護端點
    protectedHandler := http.HandlerFunc(protectedHandler)
    http.Handle("/protected", AuthMiddleware(protectedHandler))

    log.Println("Server starting on :8080")
    log.Println("Public: curl http://localhost:8080/public")
    log.Println("Protected: curl -H 'Authorization: Bearer valid-token-123' http://localhost:8080/protected")
    log.Fatal(http.ListenAndServe(":8080", nil))
}
```

**Node.js (Express):**
```javascript
const express = require('express');
const app = express();

// 認證中間件
const authMiddleware = (req, res, next) => {
    const auth = req.headers.authorization;

    if (!auth) {
        return res.status(401).send('Unauthorized');
    }

    const parts = auth.split(' ');
    if (parts.length !== 2 || parts[0] !== 'Bearer') {
        return res.status(401).send('Invalid authorization header');
    }

    const token = parts[1];

    if (token !== 'valid-token-123') {
        return res.status(401).send('Invalid token');
    }

    // 將用戶信息存入 req
    req.user = {
        id: '123',
        name: 'Alice'
    };

    next();
};

// 公開端點
app.get('/public', (req, res) => {
    res.send('This is public');
});

// 受保護端點
app.get('/protected', authMiddleware, (req, res) => {
    res.send(`Hello, ${req.user.name}! Your ID is ${req.user.id}`);
});

app.listen(8080, () => {
    console.log('Server starting on :8080');
});
```

### 示例 3: 使用 gorilla/mux 進行路由

**Go:**
```go
package main

import (
    "encoding/json"
    "log"
    "net/http"

    "github.com/gorilla/mux"
)

type Product struct {
    ID    string `json:"id"`
    Name  string `json:"name"`
    Price float64 `json:"price"`
}

var products = []Product{
    {ID: "1", Name: "Laptop", Price: 999.99},
    {ID: "2", Name: "Mouse", Price: 29.99},
}

func getProductsHandler(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(products)
}

func getProductHandler(w http.ResponseWriter, r *http.Request) {
    // 獲取路徑參數
    vars := mux.Vars(r)
    id := vars["id"]

    for _, product := range products {
        if product.ID == id {
            w.Header().Set("Content-Type", "application/json")
            json.NewEncoder(w).Encode(product)
            return
        }
    }

    http.Error(w, "Product not found", http.StatusNotFound)
}

func createProductHandler(w http.ResponseWriter, r *http.Request) {
    var product Product
    if err := json.NewDecoder(r.Body).Decode(&product); err != nil {
        http.Error(w, "Invalid JSON", http.StatusBadRequest)
        return
    }

    products = append(products, product)

    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusCreated)
    json.NewEncoder(w).Encode(product)
}

func searchProductsHandler(w http.ResponseWriter, r *http.Request) {
    // 獲取查詢參數
    query := r.URL.Query()
    name := query.Get("name")

    var results []Product
    for _, product := range products {
        if name == "" || product.Name == name {
            results = append(results, product)
        }
    }

    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(results)
}

func main() {
    r := mux.NewRouter()

    // API 路由
    api := r.PathPrefix("/api").Subrouter()

    // 產品路由
    api.HandleFunc("/products", getProductsHandler).Methods("GET")
    api.HandleFunc("/products", createProductHandler).Methods("POST")
    api.HandleFunc("/products/{id}", getProductHandler).Methods("GET")
    api.HandleFunc("/products/search", searchProductsHandler).Methods("GET")

    // 應用中間件
    r.Use(loggingMiddleware)

    log.Println("Server starting on :8080")
    log.Fatal(http.ListenAndServe(":8080", r))
}

func loggingMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        log.Printf("[%s] %s", r.Method, r.URL.Path)
        next.ServeHTTP(w, r)
    })
}

// 安裝: go get -u github.com/gorilla/mux
```

**Node.js (Express):**
```javascript
const express = require('express');
const app = express();

app.use(express.json());

let products = [
    {id: '1', name: 'Laptop', price: 999.99},
    {id: '2', name: 'Mouse', price: 29.99}
];

// 日誌中間件
app.use((req, res, next) => {
    console.log(`[${req.method}] ${req.url}`);
    next();
});

// API 路由
const api = express.Router();

api.get('/products', (req, res) => {
    res.json(products);
});

api.get('/products/search', (req, res) => {
    const { name } = req.query;
    const results = name
        ? products.filter(p => p.name === name)
        : products;
    res.json(results);
});

api.get('/products/:id', (req, res) => {
    const product = products.find(p => p.id === req.params.id);
    if (!product) {
        return res.status(404).send('Product not found');
    }
    res.json(product);
});

api.post('/products', (req, res) => {
    const product = req.body;
    products.push(product);
    res.status(201).json(product);
});

app.use('/api', api);

app.listen(8080, () => {
    console.log('Server starting on :8080');
});
```

### 示例 4: 路由組和版本控制

**Go:**
```go
package main

import (
    "encoding/json"
    "log"
    "net/http"

    "github.com/gorilla/mux"
)

func main() {
    r := mux.NewRouter()

    // API v1
    v1 := r.PathPrefix("/api/v1").Subrouter()
    v1.HandleFunc("/users", getUsersV1).Methods("GET")
    v1.HandleFunc("/users/{id}", getUserV1).Methods("GET")

    // API v2
    v2 := r.PathPrefix("/api/v2").Subrouter()
    v2.HandleFunc("/users", getUsersV2).Methods("GET")
    v2.HandleFunc("/users/{id}", getUserV2).Methods("GET")

    // Admin 路由組（需要認證）
    admin := r.PathPrefix("/admin").Subrouter()
    admin.Use(adminAuthMiddleware)
    admin.HandleFunc("/dashboard", adminDashboard).Methods("GET")
    admin.HandleFunc("/users", adminUsers).Methods("GET")

    // Public 路由
    r.HandleFunc("/", homeHandler).Methods("GET")
    r.HandleFunc("/health", healthHandler).Methods("GET")

    // 全局中間件
    r.Use(loggingMiddleware)
    r.Use(corsMiddleware)

    log.Println("Server starting on :8080")
    log.Fatal(http.ListenAndServe(":8080", r))
}

// V1 處理器
func getUsersV1(w http.ResponseWriter, r *http.Request) {
    users := []map[string]string{
        {"id": "1", "name": "Alice"},
        {"id": "2", "name": "Bob"},
    }
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(map[string]interface{}{
        "version": "v1",
        "users":   users,
    })
}

func getUserV1(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(r)
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(map[string]interface{}{
        "version": "v1",
        "user":    map[string]string{"id": vars["id"], "name": "Alice"},
    })
}

// V2 處理器（帶更多信息）
func getUsersV2(w http.ResponseWriter, r *http.Request) {
    users := []map[string]interface{}{
        {"id": "1", "name": "Alice", "email": "alice@example.com"},
        {"id": "2", "name": "Bob", "email": "bob@example.com"},
    }
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(map[string]interface{}{
        "version": "v2",
        "users":   users,
    })
}

func getUserV2(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(r)
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(map[string]interface{}{
        "version": "v2",
        "user": map[string]interface{}{
            "id":    vars["id"],
            "name":  "Alice",
            "email": "alice@example.com",
        },
    })
}

// Admin 處理器
func adminDashboard(w http.ResponseWriter, r *http.Request) {
    w.Write([]byte("Admin Dashboard"))
}

func adminUsers(w http.ResponseWriter, r *http.Request) {
    w.Write([]byte("Admin Users Management"))
}

// Public 處理器
func homeHandler(w http.ResponseWriter, r *http.Request) {
    w.Write([]byte("Welcome to API"))
}

func healthHandler(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(map[string]string{"status": "healthy"})
}

// 中間件
func loggingMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        log.Printf("[%s] %s", r.Method, r.URL.Path)
        next.ServeHTTP(w, r)
    })
}

func corsMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        w.Header().Set("Access-Control-Allow-Origin", "*")
        next.ServeHTTP(w, r)
    })
}

func adminAuthMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // 簡化的認證檢查
        auth := r.Header.Get("Authorization")
        if auth != "Bearer admin-token" {
            http.Error(w, "Unauthorized", http.StatusUnauthorized)
            return
        }
        next.ServeHTTP(w, r)
    })
}
```

**Node.js (Express):**
```javascript
const express = require('express');
const app = express();

app.use(express.json());

// 全局中間件
app.use((req, res, next) => {
    console.log(`[${req.method}] ${req.url}`);
    next();
});

app.use((req, res, next) => {
    res.header('Access-Control-Allow-Origin', '*');
    next();
});

// Admin 認證中間件
const adminAuth = (req, res, next) => {
    if (req.headers.authorization !== 'Bearer admin-token') {
        return res.status(401).send('Unauthorized');
    }
    next();
};

// API v1
const v1 = express.Router();
v1.get('/users', (req, res) => {
    res.json({
        version: 'v1',
        users: [
            {id: '1', name: 'Alice'},
            {id: '2', name: 'Bob'}
        ]
    });
});

v1.get('/users/:id', (req, res) => {
    res.json({
        version: 'v1',
        user: {id: req.params.id, name: 'Alice'}
    });
});

// API v2
const v2 = express.Router();
v2.get('/users', (req, res) => {
    res.json({
        version: 'v2',
        users: [
            {id: '1', name: 'Alice', email: 'alice@example.com'},
            {id: '2', name: 'Bob', email: 'bob@example.com'}
        ]
    });
});

v2.get('/users/:id', (req, res) => {
    res.json({
        version: 'v2',
        user: {id: req.params.id, name: 'Alice', email: 'alice@example.com'}
    });
});

// Admin 路由
const admin = express.Router();
admin.use(adminAuth);
admin.get('/dashboard', (req, res) => res.send('Admin Dashboard'));
admin.get('/users', (req, res) => res.send('Admin Users Management'));

// 掛載路由
app.use('/api/v1', v1);
app.use('/api/v2', v2);
app.use('/admin', admin);

// Public 路由
app.get('/', (req, res) => res.send('Welcome to API'));
app.get('/health', (req, res) => res.json({status: 'healthy'}));

app.listen(8080, () => {
    console.log('Server starting on :8080');
});
```

## 重點總結

1. **中間件模式**
   - 中間件是包裝 HTTP Handler 的函數
   - 可以在請求前後執行邏輯
   - 使用 Chain 函數組合多個中間件

2. **常用中間件**
   - 日誌記錄
   - 錯誤恢復
   - CORS 處理
   - 認證授權
   - 速率限制

3. **路由參數**
   - 標準庫需要手動解析
   - gorilla/mux 支持 `{id}` 語法
   - chi 支持 `{id}` 語法

4. **路由組織**
   - 使用 PathPrefix 創建子路由
   - 使用不同版本 API
   - 為不同區域應用不同中間件

5. **Context 使用**
   - 在中間件間傳遞數據
   - 存儲用戶信息
   - 存儲請求 ID

6. **與 Express 的差異**
   - Go 需要手動實現中間件模式
   - Go 的中間件是函數包裝
   - Express 的中間件更像事件鏈

## 練習題

### 練習 1: 速率限制中間件
實現基於 IP 的速率限制中間件。

### 練習 2: JWT 認證
實現 JWT Token 認證中間件。

### 練習 3: 請求 ID
實現為每個請求生成唯一 ID 的中間件。

### 練習 4: 響應時間
實現記錄每個請求響應時間的中間件。

### 練習 5: 請求體大小限制
實現限制請求體大小的中間件。

### 練習 6: IP 白名單
實現 IP 白名單訪問控制中間件。

### 練習 7: 緩存中間件
實現簡單的響應緩存中間件。

### 練習 8: 路由分組
使用 gorilla/mux 創建複雜的路由結構（包含版本、認證等）。
