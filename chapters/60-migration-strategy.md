# Chapter 60: 從 Node.js 遷移到 Go

## 概述

從 Node.js 遷移到 Go 是一個重要的技術決策。本章將介紹遷移策略、常見陷阱、最佳實踐，以及何時應該選擇 Go 而非 Node.js。

## 為什麼要遷移到 Go？

### Go 的優勢

1. **性能**
   - CPU 密集型任務快 10-100 倍
   - 內存占用更少
   - 並發處理能力強

2. **部署**
   - 單一二進制文件
   - 無運行時依賴
   - 交叉編譯簡單

3. **可維護性**
   - 靜態類型
   - 簡單的語法
   - 強大的工具鏈

4. **並發**
   - Goroutine 輕量級
   - Channel 通信模型
   - 更容易編寫並發程序

### Node.js 的優勢

1. **生態系統**
   - npm 包最豐富
   - 前端技術棧統一
   - 快速原型開發

2. **異步 I/O**
   - 事件循環模型成熟
   - 適合 I/O 密集型應用

3. **開發效率**
   - 動態類型（快速開發）
   - 熱重載
   - REPL

## 何時遷移到 Go

### 適合遷移的場景

1. **性能瓶頸**
   ```
   - CPU 密集型計算
   - 高並發需求
   - 內存限制
   - 響應時間要求嚴格
   ```

2. **微服務架構**
   ```
   - 服務數量多
   - 需要快速啟動
   - 資源利用率要求高
   ```

3. **系統工具**
   ```
   - CLI 工具
   - 網絡工具
   - DevOps 工具
   ```

### 不建議遷移的場景

1. **前端渲染**
   - Next.js、Nuxt.js 應用
   - SSR 應用

2. **快速原型**
   - MVP 階段
   - 頻繁變化的需求

3. **依賴 npm 生態**
   - 大量使用 npm 包
   - 沒有 Go 等價庫

## 遷移策略

### 策略 1: 漸進式遷移（推薦）

從性能瓶頸或獨立模塊開始，逐步遷移。

```
Node.js 應用
├── API Gateway (Node.js) ──────┐
├── User Service (Node.js)       │
├── Auth Service (Go) ←──────── 遷移 1
├── Payment Service (Node.js)    │
└── Analytics Service (Go) ←──── 遷移 2
```

**優點**：
- 風險低
- 可以驗證效果
- 團隊逐步適應

**步驟**：
1. 識別瓶頸模塊
2. 用 Go 重寫該模塊
3. 並行運行新舊版本
4. 逐步切換流量
5. 監控和驗證
6. 下線舊模塊

### 策略 2: 新功能用 Go

保持現有 Node.js 代碼，新功能用 Go 實現。

```
├── Legacy API (Node.js)
└── New API (Go)
    ├── v2 endpoints
    └── 新功能
```

**優點**：
- 無需重寫現有代碼
- 降低風險
- 逐步積累 Go 經驗

### 策略 3: 完全重寫

適合小型應用或需要重構的應用。

**注意事項**：
- 風險高
- 時間成本大
- 需要完整的測試覆蓋

## 遷移實戰

### 案例：遷移 Express API 到 Go

#### 原 Node.js 代碼

```javascript
// user.routes.js
const express = require('express');
const router = express.Router();
const userController = require('./user.controller');
const authMiddleware = require('./auth.middleware');

router.get('/users', authMiddleware, userController.getUsers);
router.get('/users/:id', authMiddleware, userController.getUser);
router.post('/users', authMiddleware, userController.createUser);
router.put('/users/:id', authMiddleware, userController.updateUser);
router.delete('/users/:id', authMiddleware, userController.deleteUser);

module.exports = router;
```

```javascript
// user.controller.js
const User = require('./user.model');

exports.getUsers = async (req, res) => {
    try {
        const users = await User.find();
        res.json(users);
    } catch (error) {
        res.status(500).json({ error: error.message });
    }
};

exports.getUser = async (req, res) => {
    try {
        const user = await User.findById(req.params.id);
        if (!user) {
            return res.status(404).json({ error: 'User not found' });
        }
        res.json(user);
    } catch (error) {
        res.status(500).json({ error: error.message });
    }
};

exports.createUser = async (req, res) => {
    try {
        const user = new User(req.body);
        await user.save();
        res.status(201).json(user);
    } catch (error) {
        res.status(400).json({ error: error.message });
    }
};
```

#### 遷移到 Go

```go
// handlers/user.go
package handlers

import (
    "encoding/json"
    "net/http"
    "strconv"

    "myapp/internal/models"
    "myapp/internal/repository"

    "github.com/gorilla/mux"
)

type UserHandler struct {
    repo *repository.UserRepository
}

func NewUserHandler(repo *repository.UserRepository) *UserHandler {
    return &UserHandler{repo: repo}
}

func (h *UserHandler) GetUsers(w http.ResponseWriter, r *http.Request) {
    users, err := h.repo.FindAll()
    if err != nil {
        respondError(w, http.StatusInternalServerError, err.Error())
        return
    }

    respondJSON(w, http.StatusOK, users)
}

func (h *UserHandler) GetUser(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(r)
    id, err := strconv.Atoi(vars["id"])
    if err != nil {
        respondError(w, http.StatusBadRequest, "Invalid user ID")
        return
    }

    user, err := h.repo.FindByID(id)
    if err != nil {
        respondError(w, http.StatusNotFound, "User not found")
        return
    }

    respondJSON(w, http.StatusOK, user)
}

func (h *UserHandler) CreateUser(w http.ResponseWriter, r *http.Request) {
    var user models.User
    if err := json.NewDecoder(r.Body).Decode(&user); err != nil {
        respondError(w, http.StatusBadRequest, "Invalid request body")
        return
    }

    if err := h.repo.Create(&user); err != nil {
        respondError(w, http.StatusBadRequest, err.Error())
        return
    }

    respondJSON(w, http.StatusCreated, user)
}

func respondJSON(w http.ResponseWriter, status int, data interface{}) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(status)
    json.NewEncoder(w).Encode(data)
}

func respondError(w http.ResponseWriter, status int, message string) {
    respondJSON(w, status, map[string]string{"error": message})
}
```

```go
// main.go
package main

import (
    "log"
    "net/http"

    "myapp/internal/handlers"
    "myapp/internal/middleware"
    "myapp/internal/repository"

    "github.com/gorilla/mux"
)

func main() {
    // 初始化依賴
    repo := repository.NewUserRepository(db)
    userHandler := handlers.NewUserHandler(repo)

    // 設置路由
    r := mux.NewRouter()

    // 應用中間件
    r.Use(middleware.Logger)
    r.Use(middleware.Auth)

    // 用戶路由
    r.HandleFunc("/users", userHandler.GetUsers).Methods("GET")
    r.HandleFunc("/users/{id}", userHandler.GetUser).Methods("GET")
    r.HandleFunc("/users", userHandler.CreateUser).Methods("POST")
    r.HandleFunc("/users/{id}", userHandler.UpdateUser).Methods("PUT")
    r.HandleFunc("/users/{id}", userHandler.DeleteUser).Methods("DELETE")

    log.Fatal(http.ListenAndServe(":8080", r))
}
```

## 常見模式轉換

### 1. 異步處理

#### Node.js (async/await)
```javascript
async function processOrder(orderId) {
    const order = await fetchOrder(orderId);
    const user = await fetchUser(order.userId);
    const payment = await processPayment(order);
    await sendEmail(user.email, order);
    return { success: true };
}
```

#### Go (goroutine + channel)
```go
func processOrder(orderId int) (*Result, error) {
    orderChan := make(chan *Order)
    userChan := make(chan *User)
    errChan := make(chan error, 2)

    // 並行獲取訂單和用戶
    go func() {
        order, err := fetchOrder(orderId)
        if err != nil {
            errChan <- err
            return
        }
        orderChan <- order
    }()

    go func() {
        select {
        case order := <-orderChan:
            user, err := fetchUser(order.UserID)
            if err != nil {
                errChan <- err
                return
            }
            userChan <- user
        case err := <-errChan:
            return
        }
    }()

    // 等待結果
    var order *Order
    var user *User

    for i := 0; i < 2; i++ {
        select {
        case order = <-orderChan:
        case user = <-userChan:
        case err := <-errChan:
            return nil, err
        }
    }

    // 處理支付
    payment, err := processPayment(order)
    if err != nil {
        return nil, err
    }

    // 發送郵件
    go sendEmail(user.Email, order) // 異步

    return &Result{Success: true}, nil
}
```

### 2. 中間件

#### Node.js (Express)
```javascript
function authMiddleware(req, res, next) {
    const token = req.headers.authorization;
    if (!token) {
        return res.status(401).json({ error: 'Unauthorized' });
    }

    try {
        const user = verifyToken(token);
        req.user = user;
        next();
    } catch (error) {
        res.status(401).json({ error: 'Invalid token' });
    }
}

app.use(authMiddleware);
```

#### Go
```go
func authMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        token := r.Header.Get("Authorization")
        if token == "" {
            http.Error(w, "Unauthorized", http.StatusUnauthorized)
            return
        }

        user, err := verifyToken(token)
        if err != nil {
            http.Error(w, "Invalid token", http.StatusUnauthorized)
            return
        }

        // 將用戶信息添加到上下文
        ctx := context.WithValue(r.Context(), "user", user)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}

// 使用
r.Use(authMiddleware)
```

### 3. 錯誤處理

#### Node.js (try-catch)
```javascript
async function getUserData(userId) {
    try {
        const user = await db.users.findById(userId);
        return user;
    } catch (error) {
        logger.error('Failed to get user', { userId, error });
        throw new Error('User not found');
    }
}
```

#### Go (error 返回值)
```go
func getUserData(userId int) (*User, error) {
    user, err := db.FindUserByID(userId)
    if err != nil {
        logger.Error("Failed to get user",
            zap.Int("userId", userId),
            zap.Error(err),
        )
        return nil, fmt.Errorf("user not found: %w", err)
    }

    return user, nil
}
```

### 4. 配置管理

#### Node.js (dotenv)
```javascript
require('dotenv').config();

const config = {
    port: process.env.PORT || 3000,
    database: {
        host: process.env.DB_HOST,
        port: process.env.DB_PORT,
    }
};
```

#### Go (環境變量 + Viper)
```go
type Config struct {
    Port     int
    Database DatabaseConfig
}

func LoadConfig() (*Config, error) {
    viper.SetEnvPrefix("APP")
    viper.AutomaticEnv()

    viper.SetDefault("port", 8080)

    var config Config
    config.Port = viper.GetInt("port")
    config.Database.Host = viper.GetString("database.host")

    return &config, nil
}
```

## 常見陷阱和解決方案

### 陷阱 1: 忽略錯誤處理

#### 錯誤做法
```go
result, _ := doSomething()  // 忽略錯誤
```

#### 正確做法
```go
result, err := doSomething()
if err != nil {
    return fmt.Errorf("failed to do something: %w", err)
}
```

### 陷阱 2: Goroutine 泄漏

#### 問題代碼
```go
func processData(data []string) {
    for _, item := range data {
        go process(item)  // goroutine 可能永不結束
    }
}
```

#### 正確做法
```go
func processData(data []string) error {
    var wg sync.WaitGroup
    errChan := make(chan error, len(data))

    for _, item := range data {
        wg.Add(1)
        go func(item string) {
            defer wg.Done()
            if err := process(item); err != nil {
                errChan <- err
            }
        }(item)
    }

    wg.Wait()
    close(errChan)

    // 檢查錯誤
    for err := range errChan {
        if err != nil {
            return err
        }
    }

    return nil
}
```

### 陷阱 3: 過度使用 Goroutine

Node.js 開發者可能會過度使用 goroutine。

#### 不必要的 goroutine
```go
func getUser(id int) (*User, error) {
    userChan := make(chan *User)
    go func() {
        user, _ := db.FindUser(id)
        userChan <- user
    }()
    return <-userChan, nil  // 不必要的複雜度
}
```

#### 簡單直接
```go
func getUser(id int) (*User, error) {
    return db.FindUser(id)
}
```

### 陷阱 4: 不理解指針

```go
// 錯誤：修改不會影響原值
func updateUser(user User) {
    user.Name = "Updated"
}

// 正確：使用指針
func updateUser(user *User) {
    user.Name = "Updated"
}
```

## 性能對比

### 實際測試結果

```
基準測試：處理 10000 個並發請求

Node.js (Express):
- 平均響應時間: 45ms
- 內存使用: 250MB
- CPU: 80%

Go (標準庫):
- 平均響應時間: 8ms
- 內存使用: 35MB
- CPU: 25%

結論：Go 性能約為 Node.js 的 5-6 倍
```

## 團隊轉型建議

### 1. 學習路徑

**第 1 週**：基礎語法
- 數據類型
- 控制流
- 函數和方法

**第 2-3 週**：核心概念
- Goroutine 和 Channel
- 接口
- 錯誤處理

**第 4 週**：實戰項目
- 構建簡單 REST API
- 數據庫集成
- 測試

**第 5-6 週**：高級主題
- 性能優化
- 最佳實踐
- 生產部署

### 2. 代碼審查重點

- 錯誤處理是否完整
- Goroutine 是否正確使用
- 資源是否正確釋放
- 是否遵循 Go 慣例

### 3. 工具鏈

```bash
# 必備工具
go install golang.org/x/tools/gopls@latest        # LSP
go install github.com/golangci/golangci-lint@latest  # Linter
go install github.com/cosmtrek/air@latest         # 熱重載

# 推薦 IDE
- VS Code + Go 插件
- GoLand
- Vim + vim-go
```

## 決策樹：是否遷移到 Go

```
開始
  │
  ├─ 性能是瓶頸？
  │   └─ 是 → 考慮遷移
  │   └─ 否 → 繼續
  │
  ├─ 需要高並發？
  │   └─ 是 → 考慮遷移
  │   └─ 否 → 繼續
  │
  ├─ 部署複雜？
  │   └─ 是 → 考慮遷移
  │   └─ 否 → 繼續
  │
  ├─ 團隊願意學習？
  │   └─ 否 → 保持 Node.js
  │   └─ 是 → 繼續
  │
  ├─ 有時間遷移？
  │   └─ 否 → 保持 Node.js
  │   └─ 是 → 開始遷移
```

## 重點總結

### 遷移決策要點

1. **性能需求**
   - Go：CPU 密集型、高並發
   - Node.js：I/O 密集型、快速開發

2. **團隊因素**
   - 學習曲線
   - 現有技能
   - 招聘難度

3. **生態系統**
   - Go：系統工具、微服務
   - Node.js：Web 應用、前端工具

### 遷移最佳實踐

1. **漸進式遷移**
   - 從獨立模塊開始
   - 並行運行驗證
   - 逐步擴大範圍

2. **充分測試**
   - 單元測試
   - 集成測試
   - 性能測試

3. **監控和日誌**
   - 遷移前後對比
   - 實時監控
   - 問題追蹤

## 練習題

### 練習 1: 遷移 Express 中間件

將以下 Express 中間件遷移到 Go：
```javascript
function rateLimiter(req, res, next) {
    // 實現速率限制
}
```

### 練習 2: 遷移異步邏輯

將以下 Node.js 異步代碼遷移到 Go：
```javascript
async function fetchAllData() {
    const [users, posts, comments] = await Promise.all([
        fetchUsers(),
        fetchPosts(),
        fetchComments()
    ]);
    return { users, posts, comments };
}
```

### 練習 3: 性能對比

實現相同的 API，對比 Node.js 和 Go 的性能：
- 響應時間
- 內存使用
- CPU 使用

### 練習 4: 遷移計劃

為現有 Node.js 項目制定遷移計劃：
1. 識別模塊
2. 評估優先級
3. 制定時間表
4. 風險評估

### 練習 5: 混合架構

設計一個混合架構：
- API Gateway (Node.js)
- 核心服務 (Go)
- 如何通信？
- 如何部署？

---

## 結語

恭喜您完成 Go 學習指南！您已經掌握了：

1. **Go 基礎**（第 1-20 章）
2. **Go 進階**（第 21-40 章）
3. **Go 實戰**（第 41-50 章）
4. **生產部署**（第 51-60 章）

### 後續學習建議

1. **深入源碼**
   - 閱讀 Go 標準庫源碼
   - 學習優秀開源項目

2. **實戰項目**
   - 構建完整應用
   - 參與開源貢獻

3. **持續學習**
   - 關注 Go 官方博客
   - 參加 Go 社區活動
   - 閱讀 Go 相關書籍

### 推薦資源

**官方資源**：
- [Go 官方文檔](https://golang.org/doc/)
- [Go 博客](https://blog.golang.org/)
- [Go Wiki](https://github.com/golang/go/wiki)

**書籍**：
- "The Go Programming Language"
- "Concurrency in Go"
- "Go in Action"

**社區**：
- [Go Forum](https://forum.golangbridge.org/)
- [Gophers Slack](https://gophers.slack.com/)
- [Reddit r/golang](https://reddit.com/r/golang)

祝您的 Go 之旅順利！
