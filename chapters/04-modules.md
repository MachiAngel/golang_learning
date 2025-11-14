# 第四章：Go 工作空間與模組管理

## 章節概述

本章將深入講解 Go 的模組管理系統，對比 npm/package.json，幫助 Node.js 開發者理解 go.mod、go.sum 以及 Go 的依賴管理機制。

## 模組系統概覽

### Node.js 模組系統

```bash
# Node.js 項目結構
my-project/
├── node_modules/        # 依賴安裝目錄（可能幾百MB）
│   ├── express/
│   ├── lodash/
│   └── ...
├── package.json         # 項目配置
├── package-lock.json    # 鎖文件
└── src/
    └── index.js
```

```json
// package.json
{
  "name": "my-project",
  "version": "1.0.0",
  "main": "index.js",
  "dependencies": {
    "express": "^4.18.0",
    "lodash": "^4.17.21"
  },
  "devDependencies": {
    "jest": "^29.0.0"
  },
  "scripts": {
    "start": "node index.js",
    "test": "jest"
  }
}
```

### Go 模組系統

```bash
# Go 項目結構
my-project/
├── go.mod               # 模組定義
├── go.sum               # 鎖文件
└── main.go

# 依賴不在項目目錄！
# 全局緩存：~/go/pkg/mod/
```

```go
// go.mod
module github.com/username/my-project

go 1.21

require (
    github.com/gin-gonic/gin v1.9.1
    github.com/stretchr/testify v1.8.4
)

require (
    // 間接依賴（自動管理）
    github.com/bytedance/sonic v1.9.1 // indirect
    // ...
)
```

## 創建 Go 模組

### 初始化模組

**Node.js:**

```bash
# 初始化 npm 項目
$ npm init

# 或快速創建
$ npm init -y

# 生成 package.json
```

**Go:**

```bash
# 初始化 Go 模組
$ go mod init myproject

# 或使用完整路徑（推薦）
$ go mod init github.com/username/myproject

# 生成 go.mod 文件
```

### 初始 go.mod 文件

```go
module github.com/username/myproject

go 1.21
```

**詳解：**
- `module`: 模組路徑（類似 package.json 的 name）
- `go`: 使用的 Go 版本

### 模組命名最佳實踐

```bash
# 推薦：使用代碼託管路徑
go mod init github.com/username/project
go mod init gitlab.com/team/project
go mod init bitbucket.org/company/project

# 本地開發/學習可以用簡單名稱
go mod init hello
go mod init calculator
go mod init myapp

# 公司內部項目
go mod init company.com/team/project
```

**對比 Node.js:**

```json
{
  "name": "@username/project",     // npm scoped package
  "name": "project",                // 簡單名稱
  "name": "@company/internal-tool"  // 私有包
}
```

## 添加依賴

### 安裝依賴

**Node.js:**

```bash
# 安裝依賴
$ npm install express
$ npm install lodash@4.17.21    # 指定版本
$ npm install -D jest            # 開發依賴

# 批量安裝
$ npm install express lodash axios

# 安裝 package.json 中的所有依賴
$ npm install
```

**Go:**

```bash
# 方式 1：在代碼中 import，然後 go mod tidy
# main.go
import "github.com/gin-gonic/gin"

$ go mod tidy
# 自動下載並添加到 go.mod

# 方式 2：直接使用 go get
$ go get github.com/gin-gonic/gin
$ go get github.com/gin-gonic/gin@v1.9.1    # 指定版本
$ go get github.com/gin-gonic/gin@latest    # 最新版本

# 批量安裝（需要多次 go get）
$ go get github.com/gin-gonic/gin
$ go get github.com/gorilla/mux

# 安裝 go.mod 中的所有依賴
$ go mod download
```

### 實際示例

**Node.js 示例：**

```bash
# 創建項目
$ mkdir my-node-app && cd my-node-app
$ npm init -y

# 安裝依賴
$ npm install express

# package.json 自動更新
```

```json
{
  "name": "my-node-app",
  "dependencies": {
    "express": "^4.18.2"
  }
}
```

```javascript
// index.js
const express = require('express');
const app = express();

app.get('/', (req, res) => {
  res.send('Hello!');
});

app.listen(3000);
```

**Go 示例：**

```bash
# 創建項目
$ mkdir my-go-app && cd my-go-app
$ go mod init my-go-app

# 編寫代碼
$ cat > main.go << 'EOF'
package main

import (
    "github.com/gin-gonic/gin"
)

func main() {
    r := gin.Default()
    r.GET("/", func(c *gin.Context) {
        c.String(200, "Hello!")
    })
    r.Run(":3000")
}
EOF

# 下載依賴
$ go mod tidy

# go.mod 自動更新
```

```go
// go.mod（自動生成）
module my-go-app

go 1.21

require github.com/gin-gonic/gin v1.9.1

require (
    github.com/bytedance/sonic v1.9.1 // indirect
    github.com/chenzhuoyu/base64x v0.0.0-20221115062448-fe3a3abad311 // indirect
    // ... 更多間接依賴
)
```

## 依賴管理命令

### 常用命令對比

| 操作 | Node.js | Go |
|------|---------|-----|
| **初始化項目** | `npm init` | `go mod init` |
| **添加依賴** | `npm install pkg` | `go get pkg` |
| **安裝所有依賴** | `npm install` | `go mod download` |
| **更新依賴** | `npm update` | `go get -u pkg` |
| **刪除依賴** | `npm uninstall pkg` | `go mod tidy` |
| **清理依賴** | `npm prune` | `go mod tidy` |
| **查看依賴樹** | `npm list` | `go mod graph` |
| **查看過時依賴** | `npm outdated` | `go list -u -m all` |
| **審計安全性** | `npm audit` | `go list -json -m all` |

### Go 模組命令詳解

#### 1. go mod init

```bash
# 初始化模組
go mod init [模組路徑]

# 示例
go mod init github.com/username/project
go mod init myapp
```

#### 2. go mod tidy

```bash
# 最常用的命令！
# - 添加缺失的依賴
# - 刪除未使用的依賴
# - 更新 go.mod 和 go.sum

go mod tidy

# 指定 Go 版本
go mod tidy -go=1.21
```

**使用場景：**
```go
// main.go
package main

import (
    "github.com/gin-gonic/gin"
    // "github.com/unused/package"  // 註釋掉不用的
)

func main() {
    r := gin.Default()
    r.Run()
}
```

```bash
# 運行 go mod tidy
$ go mod tidy
# 自動：
# 1. 下載 gin 及其依賴
# 2. 刪除 unused/package
# 3. 更新 go.mod 和 go.sum
```

#### 3. go mod download

```bash
# 下載 go.mod 中的所有依賴到緩存
go mod download

# 下載特定模組
go mod download github.com/gin-gonic/gin

# 查看下載位置
go mod download -json
```

#### 4. go get

```bash
# 添加/更新依賴
go get github.com/gin-gonic/gin

# 安裝特定版本
go get github.com/gin-gonic/gin@v1.9.0
go get github.com/gin-gonic/gin@latest

# 更新到最新版本
go get -u github.com/gin-gonic/gin

# 更新所有依賴
go get -u ./...

# 降級版本
go get github.com/gin-gonic/gin@v1.8.0

# 安裝可執行工具
go install github.com/golangci/golangci-lint/cmd/golangci-lint@latest
```

#### 5. go mod graph

```bash
# 查看依賴圖
go mod graph

# 輸出示例：
# myapp github.com/gin-gonic/gin@v1.9.1
# github.com/gin-gonic/gin@v1.9.1 github.com/bytedance/sonic@v1.9.1
# github.com/gin-gonic/gin@v1.9.1 github.com/gin-contrib/sse@v0.1.0
# ...

# 查看特定包的依賴
go mod graph | grep gin
```

**對比 npm list:**
```bash
# Node.js
$ npm list
my-node-app@1.0.0
├── express@4.18.2
│   ├── accepts@1.3.8
│   ├── body-parser@1.20.1
│   └── ...
└── lodash@4.17.21
```

#### 6. go mod why

```bash
# 為什麼需要某個依賴？
go mod why github.com/bytedance/sonic

# 輸出：
# # github.com/bytedance/sonic
# myapp
# github.com/gin-gonic/gin
# github.com/bytedance/sonic
```

#### 7. go mod vendor

```bash
# 將依賴複製到 vendor/ 目錄（類似 node_modules）
go mod vendor

# 使用 vendor 目錄構建
go build -mod=vendor

# 目錄結構：
# myapp/
# ├── vendor/
# │   ├── github.com/
# │   │   └── gin-gonic/
# │   └── modules.txt
# ├── go.mod
# └── main.go
```

**何時使用 vendor：**
- 確保構建的可重現性
- 離線構建
- CI/CD 環境

#### 8. go list

```bash
# 列出所有依賴
go list -m all

# 列出可更新的依賴
go list -u -m all

# JSON 格式輸出
go list -json -m all

# 查看特定包的信息
go list -m github.com/gin-gonic/gin
```

## go.mod 文件詳解

### 基本結構

```go
module github.com/username/myproject

go 1.21

require (
    github.com/gin-gonic/gin v1.9.1
    github.com/stretchr/testify v1.8.4
)

require (
    github.com/bytedance/sonic v1.9.1 // indirect
    github.com/chenzhuoyu/base64x v0.0.0-20221115062448-fe3a3abad311 // indirect
)

replace github.com/old/package => github.com/new/package v1.0.0

exclude github.com/bad/package v1.2.3
```

### module 指令

```go
// 定義模組路徑
module github.com/username/myproject

// 這個路徑用於：
// 1. 其他項目導入此模組
// 2. 內部包的導入路徑
```

### go 指令

```go
// 指定 Go 版本
go 1.21

// 作用：
// 1. 指定最低 Go 版本要求
// 2. 啟用該版本的語言特性
```

### require 指令

```go
// 直接依賴
require (
    github.com/gin-gonic/gin v1.9.1
    github.com/gorilla/mux v1.8.0
)

// 間接依賴（自動管理）
require (
    github.com/bytedance/sonic v1.9.1 // indirect
)
```

**indirect 注釋：**
- 表示間接依賴（依賴的依賴）
- 由 `go mod tidy` 自動管理
- 類似 npm 的 dependencies vs peerDependencies

### replace 指令

```go
// 替換依賴（類似 npm 的 resolutions）
replace (
    // 使用本地路徑（開發時）
    github.com/mylib/package => ../mylib

    // 替換為其他版本
    github.com/old/package => github.com/new/package v1.0.0

    // 替換為 fork
    github.com/original/lib => github.com/myusername/lib v1.0.0
)
```

**使用場景：**
```go
// 開發時使用本地版本
replace github.com/mycompany/common => ../common

// 修復安全問題
replace github.com/vulnerable/lib => github.com/patched/lib v1.0.1

// 使用內部 fork
replace github.com/external/lib => github.com/company/internal-lib v2.0.0
```

**對比 Node.js:**
```json
{
  "resolutions": {
    "lodash": "4.17.21",
    "minimist": "1.2.6"
  }
}
```

### exclude 指令

```go
// 排除特定版本（罕見使用）
exclude (
    github.com/buggy/lib v1.2.3
    github.com/broken/package v0.1.0
)
```

## go.sum 文件

### 什麼是 go.sum？

```go
// go.sum 記錄依賴的校驗和
github.com/gin-gonic/gin v1.9.1 h1:4idEAncQnU5cB7BeOkPtxjfCSye0AAm1R0RVIqJ+Jmg=
github.com/gin-gonic/gin v1.9.1/go.mod h1:hPrL7YrpYKXt5YId3A/Tnip5kqbEAP+KLuI3SUcPTeU=
```

**作用：**
1. 確保依賴完整性
2. 防止惡意篡改
3. 鎖定精確版本

**對比 package-lock.json:**

```json
{
  "packages": {
    "node_modules/express": {
      "version": "4.18.2",
      "resolved": "https://registry.npmjs.org/express/-/express-4.18.2.tgz",
      "integrity": "sha512-5/PsL6iGPdfQ/lKM1UuielYgv3BUoJfz1aUwU9vHZ+J7gyvwdQXFEBIEIaxeGf0GIcreATNyBExtalisDbuMqQ=="
    }
  }
}
```

### 是否提交 go.sum？

```bash
# 是的！應該提交到 git
git add go.mod go.sum
git commit -m "Add dependencies"

# 類似於提交 package-lock.json
```

## 版本管理

### 語義化版本

```bash
# Go 使用語義化版本（Semantic Versioning）
# MAJOR.MINOR.PATCH

v1.9.1
# 1: 主版本（不兼容的 API 變更）
# 9: 次版本（向後兼容的功能）
# 1: 補丁版本（向後兼容的錯誤修復）
```

### 版本選擇

```bash
# 最新版本
go get github.com/gin-gonic/gin@latest

# 特定版本
go get github.com/gin-gonic/gin@v1.9.1

# 特定 commit
go get github.com/gin-gonic/gin@abc1234

# 特定分支
go get github.com/gin-gonic/gin@master

# 版本範圍（通過 >= 和 < 自動選擇）
# Go 會自動選擇最新的兼容版本
```

**對比 npm 版本規則：**

```json
{
  "dependencies": {
    "express": "^4.18.0",    // 允許 4.x.x（<5.0.0）
    "lodash": "~4.17.21",    // 允許 4.17.x
    "axios": "1.3.4",        // 精確版本
    "chalk": "latest"        // 最新版本
  }
}
```

### Go 版本策略

```go
// go.mod 中的版本是精確的
require github.com/gin-gonic/gin v1.9.1

// 更新次版本和補丁（安全）
go get -u=patch github.com/gin-gonic/gin  // 只更新補丁

// 更新次版本
go get -u github.com/gin-gonic/gin        // 更新次版本和補丁

// 更新主版本（需要顯式指定）
go get github.com/gin-gonic/gin/v2
```

## 內部包和路徑

### 導入路徑結構

```go
// go.mod
module github.com/username/myproject

// 目錄結構
myproject/
├── go.mod
├── main.go
├── internal/
│   ├── handlers/
│   │   └── user.go
│   └── models/
│       └── user.go
└── pkg/
    └── utils/
        └── string.go
```

```go
// main.go
package main

import (
    // 標準庫
    "fmt"
    "net/http"

    // 內部包（基於 module 路徑）
    "github.com/username/myproject/internal/handlers"
    "github.com/username/myproject/pkg/utils"

    // 第三方包
    "github.com/gin-gonic/gin"
)

func main() {
    r := gin.Default()
    r.GET("/user", handlers.GetUser)
    r.Run()
}
```

**對比 Node.js:**

```javascript
// Node.js
const http = require('http');                    // 核心模組
const express = require('express');              // 第三方
const handlers = require('./internal/handlers'); // 本地模組
const utils = require('./pkg/utils');           // 本地模組
```

### internal 目錄

```bash
# internal/ 是特殊目錄
# 只能被父目錄和兄弟目錄導入
# 外部項目無法導入

myproject/
├── internal/
│   └── auth/       # 只能被 myproject 使用
└── pkg/
    └── utils/      # 可以被任何項目使用
```

**示例：**

```go
// ✅ 可以（同一項目）
import "github.com/username/myproject/internal/auth"

// ❌ 不可以（外部項目）
// other-project 無法導入
import "github.com/username/myproject/internal/auth"
```

**對比 Node.js:**
```javascript
// Node.js 沒有內置的私有包機制
// 通常通過命名約定
// _private/
// .internal/
```

## 工作空間（Workspaces）

### Go 1.18+ 工作空間

```bash
# 類似 npm/yarn workspaces
# 用於同時開發多個相關模組

# 創建工作空間
go work init ./module1 ./module2

# 添加模組
go work use ./module3
```

**go.work 文件：**

```go
go 1.21

use (
    ./module1
    ./module2
    ./module3
)

replace github.com/example/lib => ./local-lib
```

**目錄結構：**

```bash
workspace/
├── go.work           # 工作空間配置
├── module1/
│   ├── go.mod
│   └── main.go
├── module2/
│   ├── go.mod
│   └── main.go
└── local-lib/
    ├── go.mod
    └── lib.go
```

**對比 Node.js workspaces:**

```json
// package.json
{
  "name": "my-monorepo",
  "workspaces": [
    "packages/*",
    "apps/*"
  ]
}
```

## 實戰示例

### 示例 1：創建可重用的庫

```bash
# 創建庫項目
mkdir mathlib && cd mathlib
go mod init github.com/username/mathlib
```

```go
// mathlib.go
package mathlib

// Add 兩數相加（公開函數）
func Add(a, b int) int {
    return a + b
}

// multiply 兩數相乘（私有函數）
func multiply(a, b int) int {
    return a * b
}

// Power 計算冪
func Power(base, exp int) int {
    result := 1
    for i := 0; i < exp; i++ {
        result = multiply(result, base)
    }
    return result
}
```

```bash
# 推送到 GitHub
git init
git add .
git commit -m "Initial commit"
git tag v1.0.0
git push origin main --tags
```

### 示例 2：使用自己的庫

```bash
# 創建新項目
mkdir calculator && cd calculator
go mod init calculator
```

```go
// main.go
package main

import (
    "fmt"
    "github.com/username/mathlib"
)

func main() {
    sum := mathlib.Add(10, 20)
    fmt.Println("Sum:", sum)

    power := mathlib.Power(2, 10)
    fmt.Println("Power:", power)
}
```

```bash
# 下載依賴
go mod tidy

# 運行
go run main.go
```

### 示例 3：使用 replace 進行本地開發

```bash
# 目錄結構
~/projects/
├── mathlib/       # 庫項目
└── calculator/    # 使用庫的項目
```

```go
// calculator/go.mod
module calculator

go 1.21

require github.com/username/mathlib v1.0.0

// 使用本地版本進行開發
replace github.com/username/mathlib => ../mathlib
```

```bash
# 現在可以直接修改 mathlib 並立即在 calculator 中看到效果
# 無需 push 和 update
```

**對比 Node.js:**

```json
// package.json
{
  "dependencies": {
    "mathlib": "file:../mathlib"
  }
}
```

或使用 npm link:
```bash
cd mathlib
npm link

cd calculator
npm link mathlib
```

## 重點總結

### Go Modules 特點

1. **全局緩存**
   - 依賴存儲在 `~/go/pkg/mod`
   - 不是每個項目一份
   - 節省磁盤空間

2. **語義化版本**
   - 嚴格遵循 SemVer
   - 主版本在導入路徑中

3. **go.mod vs package.json**
   - 更簡單，自動生成
   - 明確區分直接和間接依賴

4. **go.sum 確保完整性**
   - 加密校驗和
   - 防止篡改

5. **內置工具鏈**
   - 無需額外安裝 yarn/pnpm
   - `go mod` 命令已包含所有功能

### 常用命令速查

```bash
# 創建項目
go mod init [module-path]

# 添加依賴
go get package@version

# 更新依賴
go get -u package

# 整理依賴
go mod tidy

# 下載依賴
go mod download

# 查看依賴
go list -m all
go mod graph

# 本地開發
replace package => /local/path
```

### 最佳實踐

1. **模組命名**
   - 使用完整的託管路徑
   - 便於他人導入

2. **及時 go mod tidy**
   - 添加新 import 後運行
   - 刪除代碼後運行
   - 保持 go.mod 整潔

3. **提交 go.sum**
   - 確保團隊一致性
   - CI/CD 可重現

4. **使用 internal/**
   - 私有代碼放 internal
   - 防止意外導入

## 練習題

### 基礎題

1. **創建第一個模組**
   - 創建一個 "greeting" 模組
   - 實現 Hello() 和 Goodbye() 函數
   - 編寫 main.go 測試

2. **依賴管理**
   - 創建項目並添加 gin 依賴
   - 觀察 go.mod 和 go.sum 的變化
   - 嘗試更新到最新版本

3. **清理依賴**
   - 添加幾個依賴
   - 刪除部分 import
   - 使用 go mod tidy 清理

### 進階題

4. **創建可重用庫**
   - 創建一個字符串工具庫
   - 發布到 GitHub
   - 在另一個項目中使用

5. **本地開發流程**
   - 創建兩個項目：庫和應用
   - 使用 replace 指令本地開發
   - 測試完成後發布

6. **依賴分析**
   - 使用 go mod graph 查看依賴樹
   - 找出某個包為何被引入（go mod why）
   - 分析間接依賴

### 實戰題

7. **模組升級**
   - 創建模組 v1.0.0
   - 做不兼容變更，發布 v2.0.0
   - 學習如何在 import 路徑中包含主版本號

8. **Monorepo 設置**
   - 使用 go.work 創建工作空間
   - 包含多個相關模組
   - 測試模組間的依賴

9. **對比實踐**
   - 創建相同功能的 Node.js 和 Go 項目
   - 對比依賴管理的差異
   - 記錄 node_modules vs go 緩存的大小

## 下一章預告

下一章我們將學習 Go 的命令行工具，包括：
- go run, go build, go test 詳解
- 編譯選項和交叉編譯
- 對比 npm scripts
- 開發工作流程
- 調試和性能分析工具

準備好深入了解 Go 工具鏈了嗎？
