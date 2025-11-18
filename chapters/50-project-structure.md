# Chapter 50: 項目結構最佳實踐

## 概述

良好的項目結構是可維護代碼的基礎。本章將介紹 Go 項目的標準目錄結構，並與 Node.js 項目結構進行對比，幫助您快速理解 Go 生態系統的組織方式。

## Node.js vs Go 項目結構對比

### Node.js 典型結構
```
my-node-app/
├── node_modules/      # 依賴包
├── src/
│   ├── controllers/   # 控制器
│   ├── models/        # 數據模型
│   ├── routes/        # 路由
│   ├── middleware/    # 中間件
│   ├── services/      # 業務邏輯
│   ├── utils/         # 工具函數
│   └── app.js         # 入口文件
├── tests/             # 測試文件
├── package.json       # 依賴配置
├── .env               # 環境變量
└── README.md
```

### Go 標準項目結構
```
my-go-app/
├── cmd/               # 主應用程序入口
│   └── api/
│       └── main.go    # API 服務器入口
├── internal/          # 私有代碼（不可被其他項目導入）
│   ├── handlers/      # HTTP 處理器
│   ├── models/        # 數據模型
│   ├── repository/    # 數據訪問層
│   ├── service/       # 業務邏輯
│   └── middleware/    # 中間件
├── pkg/               # 公共庫（可被其他項目導入）
│   └── utils/         # 工具函數
├── api/               # API 定義（OpenAPI, Protocol Buffers）
├── configs/           # 配置文件
├── scripts/           # 構建和部署腳本
├── test/              # 額外的測試數據和工具
├── vendor/            # 依賴包（可選）
├── go.mod             # 依賴管理
├── go.sum             # 依賴校驗
├── Makefile           # 構建命令
└── README.md
```

## 詳細概念解釋

### 1. cmd/ 目錄

存放應用程序的主入口點。每個子目錄代表一個可執行程序。

**為什麼使用 cmd/**：
- 清晰地標識應用程序入口
- 支持多個可執行文件（如 API 服務器、CLI 工具、Worker）
- 保持 main.go 簡潔，只負責初始化和啟動

### 2. internal/ 目錄

存放私有應用程序代碼。這是 Go 語言的特殊目錄，編譯器會阻止其他項目導入這裡的包。

**Node.js 對比**：
- Node.js 沒有強制的私有代碼機制
- 通常通過約定或不發布到 npm 來保持私有
- Go 通過編譯器強制執行

### 3. pkg/ 目錄

存放可以被外部應用程序使用的庫代碼。

**注意**：
- 如果您的項目不打算被其他項目導入，可能不需要這個目錄
- 近年來有爭議，有些團隊避免使用

### 4. api/ 目錄

存放 API 定義文件：
- OpenAPI/Swagger 規範
- Protocol Buffer 定義
- JSON Schema 文件

## 實際代碼示例

### 示例 1: 簡單的 Web 應用結構

#### 目錄結構
```
simple-api/
├── cmd/
│   └── api/
│       └── main.go
├── internal/
│   ├── config/
│   │   └── config.go
│   ├── handlers/
│   │   └── user.go
│   ├── models/
│   │   └── user.go
│   └── repository/
│       └── user.go
├── go.mod
└── README.md
```

#### cmd/api/main.go
```go
package main

import (
    "log"
    "net/http"
    "simple-api/internal/config"
    "simple-api/internal/handlers"
)

func main() {
    // 加載配置
    cfg := config.Load()

    // 初始化處理器
    userHandler := handlers.NewUserHandler()

    // 設置路由
    http.HandleFunc("/users", userHandler.List)
    http.HandleFunc("/users/", userHandler.Get)

    // 啟動服務器
    log.Printf("Server starting on %s", cfg.ServerAddress)
    if err := http.ListenAndServe(cfg.ServerAddress, nil); err != nil {
        log.Fatal(err)
    }
}
```

#### internal/config/config.go
```go
package config

import "os"

type Config struct {
    ServerAddress string
    DatabaseURL   string
}

func Load() *Config {
    return &Config{
        ServerAddress: getEnv("SERVER_ADDRESS", ":8080"),
        DatabaseURL:   getEnv("DATABASE_URL", "localhost:5432"),
    }
}

func getEnv(key, defaultValue string) string {
    if value := os.Getenv(key); value != "" {
        return value
    }
    return defaultValue
}
```

#### internal/models/user.go
```go
package models

import "time"

type User struct {
    ID        int       `json:"id"`
    Name      string    `json:"name"`
    Email     string    `json:"email"`
    CreatedAt time.Time `json:"created_at"`
}
```

#### internal/handlers/user.go
```go
package handlers

import (
    "encoding/json"
    "net/http"
    "simple-api/internal/models"
    "strconv"
    "strings"
)

type UserHandler struct {
    // 這裡可以注入依賴，如 repository
}

func NewUserHandler() *UserHandler {
    return &UserHandler{}
}

func (h *UserHandler) List(w http.ResponseWriter, r *http.Request) {
    users := []models.User{
        {ID: 1, Name: "Alice", Email: "alice@example.com"},
        {ID: 2, Name: "Bob", Email: "bob@example.com"},
    }

    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(users)
}

func (h *UserHandler) Get(w http.ResponseWriter, r *http.Request) {
    // 從 URL 提取 ID: /users/123
    idStr := strings.TrimPrefix(r.URL.Path, "/users/")
    id, err := strconv.Atoi(idStr)
    if err != nil {
        http.Error(w, "Invalid ID", http.StatusBadRequest)
        return
    }

    user := models.User{
        ID:    id,
        Name:  "Alice",
        Email: "alice@example.com",
    }

    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(user)
}
```

### Node.js 對應版本

#### package.json
```json
{
  "name": "simple-api",
  "version": "1.0.0",
  "main": "src/app.js",
  "scripts": {
    "start": "node src/app.js"
  },
  "dependencies": {
    "express": "^4.18.0"
  }
}
```

#### src/app.js
```javascript
const express = require('express');
const userRoutes = require('./routes/user');
const config = require('./config/config');

const app = express();

app.use(express.json());
app.use('/users', userRoutes);

app.listen(config.port, () => {
    console.log(`Server running on port ${config.port}`);
});
```

#### src/config/config.js
```javascript
module.exports = {
    port: process.env.PORT || 3000,
    databaseUrl: process.env.DATABASE_URL || 'localhost:5432'
};
```

#### src/routes/user.js
```javascript
const express = require('express');
const router = express.Router();
const userController = require('../controllers/user');

router.get('/', userController.list);
router.get('/:id', userController.get);

module.exports = router;
```

#### src/controllers/user.js
```javascript
const User = require('../models/user');

exports.list = (req, res) => {
    const users = [
        { id: 1, name: 'Alice', email: 'alice@example.com' },
        { id: 2, name: 'Bob', email: 'bob@example.com' }
    ];
    res.json(users);
};

exports.get = (req, res) => {
    const id = parseInt(req.params.id);
    const user = { id, name: 'Alice', email: 'alice@example.com' };
    res.json(user);
};
```

### 示例 2: 複雜項目結構（包含多個服務）

```
complex-app/
├── cmd/
│   ├── api/              # REST API 服務
│   │   └── main.go
│   ├── worker/           # 後台 Worker
│   │   └── main.go
│   └── cli/              # 命令行工具
│       └── main.go
├── internal/
│   ├── api/              # API 相關代碼
│   │   ├── handlers/
│   │   ├── middleware/
│   │   └── routes/
│   ├── domain/           # 領域模型
│   │   ├── user/
│   │   └── product/
│   ├── infrastructure/   # 基礎設施
│   │   ├── database/
│   │   ├── cache/
│   │   └── queue/
│   └── shared/           # 共享代碼
│       ├── config/
│       └── logger/
├── pkg/
│   └── validator/        # 可複用的驗證工具
├── api/
│   ├── openapi.yaml      # API 文檔
│   └── proto/            # gRPC 定義
├── configs/
│   ├── config.yaml
│   └── config.prod.yaml
├── migrations/           # 數據庫遷移
├── scripts/
│   ├── build.sh
│   └── deploy.sh
├── test/
│   ├── integration/
│   └── fixtures/
├── Makefile
├── Dockerfile
├── go.mod
└── go.sum
```

### 示例 3: Makefile（構建工具）

Go 項目通常使用 Makefile 來管理構建、測試等任務。

#### Makefile
```makefile
.PHONY: build run test clean

# 變量定義
APP_NAME=myapp
CMD_DIR=./cmd/api
BUILD_DIR=./bin

# 構建
build:
	go build -o $(BUILD_DIR)/$(APP_NAME) $(CMD_DIR)

# 運行
run:
	go run $(CMD_DIR)/main.go

# 測試
test:
	go test -v ./...

# 測試覆蓋率
coverage:
	go test -coverprofile=coverage.out ./...
	go tool cover -html=coverage.out

# 清理
clean:
	rm -rf $(BUILD_DIR)
	rm -f coverage.out

# 安裝依賴
deps:
	go mod download
	go mod verify

# 代碼檢查
lint:
	golangci-lint run

# 格式化代碼
fmt:
	go fmt ./...

# 多平台構建
build-all:
	GOOS=linux GOARCH=amd64 go build -o $(BUILD_DIR)/$(APP_NAME)-linux-amd64 $(CMD_DIR)
	GOOS=darwin GOARCH=amd64 go build -o $(BUILD_DIR)/$(APP_NAME)-darwin-amd64 $(CMD_DIR)
	GOOS=windows GOARCH=amd64 go build -o $(BUILD_DIR)/$(APP_NAME)-windows-amd64.exe $(CMD_DIR)
```

**Node.js 對應**：
```json
{
  "scripts": {
    "build": "tsc",
    "start": "node dist/app.js",
    "dev": "nodemon src/app.js",
    "test": "jest",
    "test:coverage": "jest --coverage",
    "lint": "eslint .",
    "format": "prettier --write ."
  }
}
```

### 示例 4: 依賴注入模式

#### internal/app/app.go（應用程序容器）
```go
package app

import (
    "database/sql"
    "simple-api/internal/handlers"
    "simple-api/internal/repository"
    "simple-api/internal/service"
)

// App 包含應用程序的所有依賴
type App struct {
    DB           *sql.DB
    UserRepo     *repository.UserRepository
    UserService  *service.UserService
    UserHandler  *handlers.UserHandler
}

// NewApp 創建並初始化應用程序
func NewApp(db *sql.DB) *App {
    // 按依賴順序初始化
    userRepo := repository.NewUserRepository(db)
    userService := service.NewUserService(userRepo)
    userHandler := handlers.NewUserHandler(userService)

    return &App{
        DB:          db,
        UserRepo:    userRepo,
        UserService: userService,
        UserHandler: userHandler,
    }
}

// Close 清理資源
func (a *App) Close() error {
    return a.DB.Close()
}
```

#### cmd/api/main.go（使用依賴注入）
```go
package main

import (
    "database/sql"
    "log"
    "net/http"
    "simple-api/internal/app"

    _ "github.com/lib/pq"
)

func main() {
    // 初始化數據庫
    db, err := sql.Open("postgres", "your-connection-string")
    if err != nil {
        log.Fatal(err)
    }

    // 創建應用程序實例
    application := app.NewApp(db)
    defer application.Close()

    // 設置路由
    mux := http.NewServeMux()
    mux.HandleFunc("/users", application.UserHandler.List)
    mux.HandleFunc("/users/", application.UserHandler.Get)

    // 啟動服務器
    log.Println("Server starting on :8080")
    if err := http.ListenAndServe(":8080", mux); err != nil {
        log.Fatal(err)
    }
}
```

## 重點總結

### Go 項目結構關鍵點

1. **cmd/ 是入口點**
   - 每個可執行程序一個子目錄
   - main.go 應該簡潔，只負責組裝和啟動

2. **internal/ 保護私有代碼**
   - 編譯器強制執行
   - 防止外部項目導入

3. **pkg/ 存放可複用代碼**
   - 僅當需要被外部項目使用時
   - 有爭議，謹慎使用

4. **分層架構**
   - handlers（控制器層）
   - service（業務邏輯層）
   - repository（數據訪問層）
   - models（數據模型）

5. **依賴管理**
   - go.mod 和 go.sum（類似 package.json 和 package-lock.json）
   - 不要提交 vendor/ 到版本控制（類似 node_modules）

### 與 Node.js 的主要差異

| 特性 | Node.js | Go |
|------|---------|-----|
| 入口點 | src/app.js 或 index.js | cmd/xxx/main.go |
| 依賴安裝位置 | node_modules/ | $GOPATH/pkg 或模塊緩存 |
| 私有代碼 | 約定俗成 | internal/ 強制執行 |
| 構建工具 | npm scripts | Makefile, go build |
| 配置文件 | package.json | go.mod |

### 最佳實踐

1. **保持 main.go 簡潔**
   ```go
   // 好的做法
   func main() {
       app := setupApp()
       app.Run()
   }

   // 避免在 main 中編寫大量業務邏輯
   ```

2. **使用分層架構**
   - 清晰的職責分離
   - 便於測試和維護

3. **依賴注入**
   - 通過構造函數注入依賴
   - 避免全局變量

4. **配置管理**
   - 使用環境變量或配置文件
   - 不要硬編碼配置

## 練習題

### 練習 1: 創建基本項目結構

創建一個包含以下功能的項目結構：
- RESTful API 服務器（cmd/api）
- 數據導入工具（cmd/importer）
- 用戶和產品兩個領域模型

要求：
- 使用標準的 Go 項目結構
- 實現依賴注入
- 編寫 Makefile

### 練習 2: 重構 Node.js 項目

將以下 Node.js 項目結構轉換為 Go 結構：

```
node-app/
├── src/
│   ├── controllers/
│   │   ├── auth.js
│   │   └── user.js
│   ├── models/
│   │   └── user.js
│   ├── services/
│   │   └── email.js
│   └── app.js
└── package.json
```

### 練習 3: Makefile 編寫

為您的 Go 項目編寫 Makefile，包含：
- build（構建）
- run（運行）
- test（測試）
- clean（清理）
- docker-build（Docker 構建）

### 練習 4: 配置管理

實現一個配置管理包，支持：
- 從環境變量讀取
- 從 YAML 文件讀取
- 設置默認值
- 驗證必需的配置項

提示：
```go
type Config struct {
    Server   ServerConfig   `yaml:"server"`
    Database DatabaseConfig `yaml:"database"`
}

type ServerConfig struct {
    Port int    `yaml:"port"`
    Host string `yaml:"host"`
}

type DatabaseConfig struct {
    URL      string `yaml:"url"`
    MaxConns int    `yaml:"max_conns"`
}
```

### 練習 5: 分層架構實現

實現一個完整的用戶管理模塊，包含：
1. Model 層：User 結構體
2. Repository 層：數據訪問接口和實現
3. Service 層：業務邏輯
4. Handler 層：HTTP 處理

要求：
- 使用接口定義契約
- 實現依賴注入
- 每層職責清晰

### 參考答案（練習 1）

```
my-project/
├── cmd/
│   ├── api/
│   │   └── main.go
│   └── importer/
│       └── main.go
├── internal/
│   ├── domain/
│   │   ├── user/
│   │   │   ├── model.go
│   │   │   ├── repository.go
│   │   │   └── service.go
│   │   └── product/
│   │       ├── model.go
│   │       ├── repository.go
│   │       └── service.go
│   ├── api/
│   │   └── handlers/
│   │       ├── user.go
│   │       └── product.go
│   └── infrastructure/
│       └── database/
│           └── postgres.go
├── Makefile
└── go.mod
```

**Makefile**:
```makefile
.PHONY: build-api build-importer run-api test clean

build-api:
	go build -o bin/api ./cmd/api

build-importer:
	go build -o bin/importer ./cmd/importer

run-api:
	go run ./cmd/api

test:
	go test -v ./...

clean:
	rm -rf bin/
```

---

下一章：[Chapter 51: 構建完整 RESTful API](51-api-project.md)
