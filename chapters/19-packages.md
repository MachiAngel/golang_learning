# Chapter 19: 包與模塊系統 (Packages & Modules)

## 概述

Go 的包（package）系統是代碼組織和重用的基礎。Go 使用模塊（module）來管理依賴，這與 Node.js 的 npm/yarn 類似但更簡潔。理解包和模塊系統對於構建可維護的 Go 應用至關重要。

## Node.js 與 Go 的對比

| 特性 | Node.js | Go |
|------|---------|-----|
| 包管理 | npm / yarn / pnpm | go modules |
| 配置文件 | `package.json` | `go.mod` |
| 鎖文件 | `package-lock.json` / `yarn.lock` | `go.sum` |
| 導入 | `require()` / `import` | `import` |
| 導出 | `module.exports` / `export` | 大寫字母開頭 |
| 本地包 | 相對路徑 | 模塊路徑 |
| 包註冊中心 | npmjs.com | pkg.go.dev |
| 版本管理 | semver | semver + 最小版本選擇 |

## 詳細概念解釋

### 1. 包 (Package)

包是 Go 代碼組織的基本單位：
- 一個目錄下的所有 `.go` 文件屬於同一個包
- 包名通常與目錄名相同（main 包除外）
- 每個 Go 文件開頭必須聲明包名

### 2. 模塊 (Module)

模塊是包的集合，由 `go.mod` 文件定義：
- 模塊路徑通常是代碼倉庫的 URL
- 管理依賴版本
- 支持語義化版本控制

### 3. 可見性規則

- **大寫字母開頭**：公開（exported），可被其他包訪問
- **小寫字母開頭**：私有（unexported），僅包內可訪問

### 4. 導入路徑

```go
import (
    "fmt"                           // 標準庫
    "github.com/user/repo/pkg"      // 第三方包
    "mymodule/internal/utils"       // 本地包
)
```

## 代碼示例

### 示例 1: 創建和使用包

**Go 項目結構:**
```
myproject/
├── go.mod
├── main.go
└── mathutil/
    ├── add.go
    └── multiply.go
```

**go.mod:**
```go
module myproject

go 1.21
```

**mathutil/add.go:**
```go
package mathutil

// Add 是公開函數（大寫開頭）
func Add(a, b int) int {
    return a + b
}

// addInternal 是私有函數（小寫開頭）
func addInternal(a, b int) int {
    return a + b
}
```

**mathutil/multiply.go:**
```go
package mathutil

func Multiply(a, b int) int {
    return a * b
}

// 包內部可以訪問私有函數
func MultiplyAndAdd(a, b, c int) int {
    result := Multiply(a, b)
    return addInternal(result, c)
}
```

**main.go:**
```go
package main

import (
    "fmt"
    "myproject/mathutil"
)

func main() {
    sum := mathutil.Add(5, 3)
    fmt.Println("5 + 3 =", sum)

    product := mathutil.Multiply(4, 6)
    fmt.Println("4 * 6 =", product)

    result := mathutil.MultiplyAndAdd(2, 3, 4)
    fmt.Println("2 * 3 + 4 =", result)

    // 編譯錯誤：無法訪問私有函數
    // mathutil.addInternal(1, 2)
}
```

**Node.js 對比:**

項目結構:
```
myproject/
├── package.json
├── index.js
└── mathutil/
    ├── add.js
    └── multiply.js
```

**package.json:**
```json
{
  "name": "myproject",
  "version": "1.0.0",
  "type": "module"
}
```

**mathutil/add.js:**
```javascript
// 導出公開函數
export function add(a, b) {
    return a + b;
}

// 不導出的是私有函數
function addInternal(a, b) {
    return a + b;
}

// 也可以使用 CommonJS
// module.exports = { add };
```

**mathutil/multiply.js:**
```javascript
export function multiply(a, b) {
    return a * b;
}

// 可以導入同一包中的其他模塊
import { add } from './add.js';

export function multiplyAndAdd(a, b, c) {
    const result = multiply(a, b);
    // 無法訪問 addInternal，因為它沒有被導出
    return result + c;
}
```

**index.js:**
```javascript
import { add } from './mathutil/add.js';
import { multiply, multiplyAndAdd } from './mathutil/multiply.js';

console.log("5 + 3 =", add(5, 3));
console.log("4 * 6 =", multiply(4, 6));
console.log("2 * 3 + 4 =", multiplyAndAdd(2, 3, 4));

// 無法訪問未導出的函數
// addInternal(1, 2);  // 錯誤
```

### 示例 2: 初始化和 init 函數

**Go:**
```go
// config/config.go
package config

import "fmt"

var (
    AppName    string
    AppVersion string
)

// init 函數在包被導入時自動執行
// 可以有多個 init 函數，按順序執行
func init() {
    fmt.Println("初始化 config 包...")
    AppName = "MyApp"
    AppVersion = "1.0.0"
}

func init() {
    fmt.Println("執行第二個 init...")
    // 進行其他初始化
}

func GetInfo() string {
    return fmt.Sprintf("%s v%s", AppName, AppVersion)
}
```

**main.go:**
```go
package main

import (
    "fmt"
    "myproject/config"
)

func init() {
    fmt.Println("初始化 main 包...")
}

func main() {
    fmt.Println("執行 main 函數...")
    fmt.Println(config.GetInfo())
}

// 輸出順序：
// 初始化 config 包...
// 執行第二個 init...
// 初始化 main 包...
// 執行 main 函數...
// MyApp v1.0.0
```

**Node.js 對比:**
```javascript
// config/config.js
let appName = "";
let appVersion = "";

// 模塊級代碼在首次導入時執行
console.log("初始化 config 模塊...");
appName = "MyApp";
appVersion = "1.0.0";

// 可以使用 IIFE 模擬多個初始化
(function() {
    console.log("執行第二個初始化...");
})();

export function getInfo() {
    return `${appName} v${appVersion}`;
}

export { appName, appVersion };
```

**index.js:**
```javascript
console.log("初始化 main 模塊...");

import { getInfo } from './config/config.js';

console.log("執行 main 函數...");
console.log(getInfo());

// 輸出順序：
// 初始化 config 模塊...
// 執行第二個初始化...
// 初始化 main 模塊...
// 執行 main 函數...
// MyApp v1.0.0
```

### 示例 3: 導入別名和空白導入

**Go:**
```go
package main

import (
    "fmt"

    // 導入別名
    m "myproject/mathutil"

    // 點導入（不推薦，會污染命名空間）
    . "strings"

    // 空白導入（僅執行 init 函數）
    _ "myproject/config"

    // 多個包使用同名時必須使用別名
    cryptorand "crypto/rand"
    mathrand "math/rand"
)

func main() {
    // 使用別名
    sum := m.Add(1, 2)
    fmt.Println("Sum:", sum)

    // 點導入後可以直接使用函數
    result := ToUpper("hello")
    fmt.Println("Upper:", result)

    // 空白導入的包不能直接使用
    // config.GetInfo()  // 錯誤

    // 使用重命名的包
    _ = cryptorand.Reader
    _ = mathrand.Int()
}
```

**Node.js 對比:**
```javascript
// ES6 模塊

// 導入別名
import * as m from './mathutil/add.js';

// 導入特定函數並重命名
import { add as addition } from './mathutil/add.js';

// 僅執行副作用（類似空白導入）
import './config/config.js';

// 導入同名函數時重命名
import { random as cryptoRandom } from 'crypto';
import { random as mathRandom } from './mathutil/random.js';

console.log("Sum:", m.add(1, 2));
console.log("Addition:", addition(3, 4));

// CommonJS
const m = require('./mathutil/add.js');
const { add: addition } = require('./mathutil/add.js');
require('./config/config.js');  // 僅執行
```

### 示例 4: 內部包 (Internal Packages)

**Go:**

項目結構:
```
myproject/
├── go.mod
├── main.go
├── internal/
│   └── auth/
│       └── auth.go
└── pkg/
    └── api/
        └── api.go
```

**internal/auth/auth.go:**
```go
package auth

// internal 包只能被父目錄及其子目錄導入
// 其他項目無法導入此包

func Authenticate(token string) bool {
    // 認證邏輯
    return token == "secret"
}
```

**pkg/api/api.go:**
```go
package api

import (
    "myproject/internal/auth"  // 可以導入，因為在同一模塊內
)

func HandleRequest(token string) string {
    if auth.Authenticate(token) {
        return "Authenticated"
    }
    return "Unauthorized"
}
```

**main.go:**
```go
package main

import (
    "fmt"
    "myproject/internal/auth"  // 可以導入
    "myproject/pkg/api"
)

func main() {
    fmt.Println(auth.Authenticate("secret"))
    fmt.Println(api.HandleRequest("secret"))
}
```

**Node.js 對比:**

Node.js 沒有內建的 internal 機制，但可以通過約定實現：

```javascript
// 項目結構
// myproject/
// ├── package.json
// ├── index.js
// ├── internal/
// │   └── auth/
// │       └── auth.js
// └── pkg/
//     └── api/
//         └── api.js

// internal/auth/auth.js
// 約定：internal 目錄下的模塊不應被外部使用
export function authenticate(token) {
    return token === "secret";
}

// pkg/api/api.js
import { authenticate } from '../../internal/auth/auth.js';

export function handleRequest(token) {
    if (authenticate(token)) {
        return "Authenticated";
    }
    return "Unauthorized";
}

// index.js
import { authenticate } from './internal/auth/auth.js';
import { handleRequest } from './pkg/api/api.js';

console.log(authenticate("secret"));
console.log(handleRequest("secret"));

// 注意：Node.js 不會強制執行 internal 約定
// 外部包仍然可以導入 internal 目錄下的模塊
```

### 示例 5: 使用第三方包

**Go:**

```bash
# 初始化模塊
go mod init myproject

# 添加依賴
go get github.com/gin-gonic/gin@v1.9.1

# 更新依賴
go get -u github.com/gin-gonic/gin

# 整理依賴（移除未使用的）
go mod tidy

# 查看依賴
go list -m all

# 下載依賴到本地緩存
go mod download
```

**go.mod:**
```go
module myproject

go 1.21

require github.com/gin-gonic/gin v1.9.1

require (
    // 間接依賴
    github.com/gin-contrib/sse v0.1.0 // indirect
    github.com/go-playground/validator/v10 v10.14.0 // indirect
    // ... 其他依賴
)
```

**main.go:**
```go
package main

import (
    "github.com/gin-gonic/gin"
    "net/http"
)

func main() {
    r := gin.Default()

    r.GET("/ping", func(c *gin.Context) {
        c.JSON(http.StatusOK, gin.H{
            "message": "pong",
        })
    })

    r.Run(":8080")
}
```

**Node.js 對比:**

```bash
# 初始化項目
npm init -y

# 添加依賴
npm install express@4.18.2

# 添加開發依賴
npm install --save-dev nodemon

# 更新依賴
npm update express

# 移除依賴
npm uninstall express

# 查看依賴樹
npm list

# 清理並重裝
rm -rf node_modules package-lock.json
npm install
```

**package.json:**
```json
{
  "name": "myproject",
  "version": "1.0.0",
  "type": "module",
  "dependencies": {
    "express": "^4.18.2"
  },
  "devDependencies": {
    "nodemon": "^3.0.1"
  }
}
```

**index.js:**
```javascript
import express from 'express';

const app = express();

app.get('/ping', (req, res) => {
    res.json({ message: 'pong' });
});

app.listen(8080, () => {
    console.log('Server running on port 8080');
});
```

### 示例 6: 替換和本地開發

**Go:**

```bash
# 使用本地版本替換遠程依賴
go mod edit -replace github.com/user/pkg=../local-pkg

# 從文件系統導入
go mod edit -replace github.com/user/pkg=/absolute/path/to/pkg
```

**go.mod:**
```go
module myproject

go 1.21

require github.com/user/pkg v1.2.3

// 替換為本地版本
replace github.com/user/pkg => ../local-pkg

// 或指定版本
replace github.com/user/pkg v1.2.3 => github.com/fork/pkg v1.3.0
```

**Node.js 對比:**

```json
{
  "dependencies": {
    "my-package": "file:../local-package"
  }
}
```

或使用 npm link:
```bash
# 在依賴包目錄
cd ../local-package
npm link

# 在項目目錄
cd myproject
npm link my-package
```

或使用工作區（Workspaces）:
```json
{
  "name": "myproject",
  "workspaces": [
    "packages/*"
  ]
}
```

### 示例 7: 包文檔

**Go:**

```go
// Package mathutil 提供數學工具函數。
//
// 這個包包含常用的數學運算函數，
// 適用於日常計算需求。
package mathutil

// Add 將兩個整數相加並返回結果。
//
// 參數：
//   a: 第一個加數
//   b: 第二個加數
//
// 返回值：
//   兩數之和
//
// 示例：
//   result := Add(5, 3)  // result = 8
func Add(a, b int) int {
    return a + b
}

// Multiply 將兩個整數相乘。
// 此函數對於大數可能溢出。
func Multiply(a, b int) int {
    return a * b
}
```

查看文檔:
```bash
# 在瀏覽器中查看
go doc -http=:6060

# 命令行查看
go doc mathutil
go doc mathutil.Add

# 生成文檔網站
godoc -http=:8080
```

**Node.js 對比:**

```javascript
/**
 * 數學工具模塊
 *
 * 提供常用的數學運算函數
 * @module mathutil
 */

/**
 * 將兩個數字相加
 *
 * @param {number} a - 第一個加數
 * @param {number} b - 第二個加數
 * @returns {number} 兩數之和
 *
 * @example
 * const result = add(5, 3);  // result = 8
 */
export function add(a, b) {
    return a + b;
}

/**
 * 將兩個數字相乘
 *
 * @param {number} a - 第一個乘數
 * @param {number} b - 第二個乘數
 * @returns {number} 兩數之積
 */
export function multiply(a, b) {
    return a * b;
}
```

使用 JSDoc 生成文檔:
```bash
npm install -g jsdoc
jsdoc mathutil.js -d docs
```

## 重點總結

### 包系統對比

| 概念 | Go | Node.js |
|------|-----|---------|
| 包定義 | 目錄 + package 聲明 | package.json |
| 導入語法 | `import "path"` | `import` / `require` |
| 導出 | 大寫開頭 | `export` / `module.exports` |
| 可見性 | 大小寫控制 | 手動導出控制 |
| 內部包 | `internal/` 目錄 | 約定（無強制） |

### 模塊系統對比

| 功能 | Go Modules | npm |
|------|-----------|-----|
| 配置文件 | go.mod | package.json |
| 鎖文件 | go.sum | package-lock.json |
| 版本策略 | 最小版本選擇 | semver + 鎖文件 |
| 緩存 | $GOPATH/pkg/mod | node_modules |
| 鏡像 | GOPROXY | npm registry |

### 最佳實踐

1. **包命名**：
   - 使用小寫字母
   - 簡短且描述性
   - 避免下劃線和駝峰

2. **包組織**：
   - 一個目錄一個包
   - 相關功能組織在一起
   - 使用 internal/ 保護內部實現

3. **導入規則**：
   - 標準庫 → 第三方 → 本地包
   - 分組導入，空行分隔
   - 避免循環導入

4. **版本管理**：
   - 使用語義化版本
   - v2+ 需要在導入路徑中包含版本號
   - 定期運行 `go mod tidy`

## 練習題

### 練習 1: 創建工具包
創建一個 `stringutil` 包，實現：
- `Reverse(s string) string`: 反轉字符串
- `IsPalindrome(s string) bool`: 判斷是否回文
- `WordCount(s string) int`: 統計單詞數

### 練習 2: 配置包
創建一個 `config` 包，支持：
- 從環境變量加載配置
- 從 JSON 文件加載配置
- 提供默認值
- 使用 init 函數初始化

### 練習 3: 日誌包
實現一個簡單的日誌包：
- 支持不同日誌級別（Debug, Info, Warn, Error）
- 可配置輸出目標（控制台、文件）
- 包含時間戳和日誌級別

### 練習 4: 數據驗證包
創建 `validator` 包：
- 驗證郵箱格式
- 驗證手機號
- 驗證密碼強度
- 使用內部包隱藏驗證規則

### 練習 5: HTTP 客戶端包
封裝一個 HTTP 客戶端包：
- 支持 GET、POST 請求
- 自動處理 JSON
- 添加超時控制
- 提供重試機制

### 答案提示

**練習 1:**
```go
// stringutil/stringutil.go
package stringutil

import "strings"

func Reverse(s string) string {
    runes := []rune(s)
    for i, j := 0, len(runes)-1; i < j; i, j = i+1, j-1 {
        runes[i], runes[j] = runes[j], runes[i]
    }
    return string(runes)
}

func IsPalindrome(s string) bool {
    s = strings.ToLower(s)
    return s == Reverse(s)
}

func WordCount(s string) int {
    return len(strings.Fields(s))
}
```

**練習 3:**
```go
// logger/logger.go
package logger

import (
    "fmt"
    "log"
    "os"
    "time"
)

type Level int

const (
    DEBUG Level = iota
    INFO
    WARN
    ERROR
)

var (
    currentLevel = INFO
    logger       = log.New(os.Stdout, "", 0)
)

func SetLevel(level Level) {
    currentLevel = level
}

func log(level Level, format string, args ...interface{}) {
    if level >= currentLevel {
        timestamp := time.Now().Format("2006-01-02 15:04:05")
        levelStr := [...]string{"DEBUG", "INFO", "WARN", "ERROR"}[level]
        message := fmt.Sprintf(format, args...)
        logger.Printf("[%s] %s: %s", timestamp, levelStr, message)
    }
}

func Debug(format string, args ...interface{}) {
    log(DEBUG, format, args...)
}

func Info(format string, args ...interface{}) {
    log(INFO, format, args...)
}

func Warn(format string, args ...interface{}) {
    log(WARN, format, args...)
}

func Error(format string, args ...interface{}) {
    log(ERROR, format, args...)
}
```
