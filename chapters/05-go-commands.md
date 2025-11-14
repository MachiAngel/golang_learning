# 第五章：Go 命令行工具詳解

## 章節概述

本章將詳細介紹 Go 的命令行工具，包括 `go run`、`go build`、`go test` 等核心命令。我們將對比 npm scripts 和 Node.js 工作流程，幫助你快速掌握 Go 的開發工具鏈。

## Go 命令概覽

### Node.js vs Go 命令對比

| 任務 | Node.js | Go |
|------|---------|-----|
| **運行程序** | `node app.js` | `go run main.go` |
| **構建** | `webpack build` (需配置) | `go build` |
| **測試** | `npm test` (需配置 jest/mocha) | `go test` |
| **格式化** | `prettier --write .` | `go fmt` |
| **代碼檢查** | `eslint .` | `go vet` |
| **依賴管理** | `npm install` | `go mod download` |
| **清理** | `rm -rf node_modules` | `go clean` |
| **文檔** | JSDoc + 工具 | `go doc` |

## go run - 運行程序

### 基本用法

```bash
# Node.js
$ node index.js

# Go
$ go run main.go
```

### 詳細示例

```go
// hello.go
package main

import "fmt"

func main() {
    fmt.Println("Hello, Go!")
}
```

```bash
# 運行單個文件
$ go run hello.go
Hello, Go!

# 運行多個文件
$ go run main.go utils.go helpers.go

# 運行整個包（當前目錄）
$ go run .

# 運行指定包
$ go run ./cmd/server
```

**go run 工作原理：**
```bash
# go run 實際上是：
# 1. 編譯代碼為臨時可執行文件
# 2. 運行可執行文件
# 3. 刪除臨時文件

# 類似於：
# go build -o /tmp/go-build123/main && /tmp/go-build123/main
```

**對比 Node.js:**
```bash
# Node.js 直接解釋執行
$ node app.js

# 或使用 npm scripts
$ npm start
```

```json
// package.json
{
  "scripts": {
    "start": "node app.js",
    "dev": "nodemon app.js"
  }
}
```

### 傳遞參數

```bash
# Node.js
$ node app.js arg1 arg2

# Go
$ go run main.go arg1 arg2
```

```go
// args.go
package main

import (
    "fmt"
    "os"
)

func main() {
    // os.Args[0] 是程序名稱
    // os.Args[1:] 是用戶參數
    if len(os.Args) > 1 {
        fmt.Println("Arguments:", os.Args[1:])
    }
}
```

```bash
$ go run args.go hello world
Arguments: [hello world]
```

### 設置環境變量

```bash
# Node.js
$ PORT=8080 node app.js
$ NODE_ENV=production node app.js

# Go
$ PORT=8080 go run main.go
$ ENV=production go run main.go
```

```go
// env.go
package main

import (
    "fmt"
    "os"
)

func main() {
    port := os.Getenv("PORT")
    if port == "" {
        port = "3000"
    }
    fmt.Println("Starting on port:", port)
}
```

## go build - 構建可執行文件

### 基本用法

```bash
# 構建當前目錄
$ go build

# 構建指定文件
$ go build main.go

# 指定輸出文件名
$ go build -o myapp

# 構建到指定目錄
$ go build -o bin/myapp
```

**對比 Node.js/Webpack:**
```bash
# Node.js 通常不需要構建（解釋型）
# 但如果使用 TypeScript 或打包工具：

# TypeScript
$ tsc

# Webpack
$ webpack --mode production

# Rollup
$ rollup -c
```

### 構建選項

```bash
# 顯示構建過程
$ go build -v

# 顯示編譯器命令
$ go build -x

# 禁用優化（便於調試）
$ go build -gcflags="-N -l"

# 減小二進制文件大小
$ go build -ldflags="-s -w"
# -s: 去除符號表
# -w: 去除 DWARF 調試信息

# 組合使用
$ go build -v -ldflags="-s -w" -o myapp
```

### 交叉編譯

這是 Go 的超強特性！

```bash
# 為 Linux 構建（在 Mac/Windows 上）
$ GOOS=linux GOARCH=amd64 go build -o myapp-linux

# 為 Windows 構建（在 Mac/Linux 上）
$ GOOS=windows GOARCH=amd64 go build -o myapp.exe

# 為 Mac 構建（在 Linux/Windows 上）
$ GOOS=darwin GOARCH=amd64 go build -o myapp-mac

# ARM 架構（樹莓派）
$ GOOS=linux GOARCH=arm go build -o myapp-arm

# ARM64 (Apple Silicon)
$ GOOS=darwin GOARCH=arm64 go build -o myapp-m1

# 查看所有支持的平台
$ go tool dist list
```

**一次構建多平台：**
```bash
#!/bin/bash
# build-all.sh

platforms=("linux/amd64" "darwin/amd64" "windows/amd64")

for platform in "${platforms[@]}"
do
    platform_split=(${platform//\// })
    GOOS=${platform_split[0]}
    GOARCH=${platform_split[1]}
    output_name='myapp-'$GOOS'-'$GOARCH
    if [ $GOOS = "windows" ]; then
        output_name+='.exe'
    fi

    echo "Building for $GOOS/$GOARCH..."
    GOOS=$GOOS GOARCH=$GOARCH go build -o $output_name
done
```

**對比 Node.js:**
```bash
# Node.js 使用 pkg 進行打包
$ npm install -g pkg
$ pkg index.js

# 但生成的文件通常很大（包含整個 Node.js 運行時）
# 而且不如 Go 交叉編譯方便
```

### 構建標籤（Build Tags）

```go
// +build linux

package main

import "fmt"

func main() {
    fmt.Println("Running on Linux")
}
```

```bash
# 只在 Linux 上編譯
$ GOOS=linux go build
```

或使用新語法（Go 1.17+）:
```go
//go:build linux

package main
```

**條件編譯示例：**

```go
// config_dev.go
//go:build dev

package config

const APIEndpoint = "http://localhost:8080"
```

```go
// config_prod.go
//go:build prod

package config

const APIEndpoint = "https://api.production.com"
```

```bash
# 開發環境構建
$ go build -tags dev

# 生產環境構建
$ go build -tags prod
```

## go test - 運行測試

### 基本用法

```bash
# 測試當前包
$ go test

# 測試所有子包
$ go test ./...

# 詳細輸出
$ go test -v

# 顯示覆蓋率
$ go test -cover

# 生成覆蓋率報告
$ go test -coverprofile=coverage.out
$ go tool cover -html=coverage.out
```

**對比 Node.js:**
```bash
# Node.js 需要安裝測試框架
$ npm install -D jest

# package.json
{
  "scripts": {
    "test": "jest",
    "test:watch": "jest --watch",
    "test:coverage": "jest --coverage"
  }
}

$ npm test
```

### 測試示例

**Go 測試：**

```go
// math.go
package math

func Add(a, b int) int {
    return a + b
}

func Multiply(a, b int) int {
    return a * b
}
```

```go
// math_test.go
package math

import "testing"

func TestAdd(t *testing.T) {
    result := Add(2, 3)
    expected := 5

    if result != expected {
        t.Errorf("Add(2, 3) = %d; want %d", result, expected)
    }
}

func TestMultiply(t *testing.T) {
    tests := []struct {
        a, b     int
        expected int
    }{
        {2, 3, 6},
        {0, 5, 0},
        {-2, 3, -6},
    }

    for _, tt := range tests {
        result := Multiply(tt.a, tt.b)
        if result != tt.expected {
            t.Errorf("Multiply(%d, %d) = %d; want %d",
                tt.a, tt.b, result, tt.expected)
        }
    }
}
```

```bash
$ go test -v
=== RUN   TestAdd
--- PASS: TestAdd (0.00s)
=== RUN   TestMultiply
--- PASS: TestMultiply (0.00s)
PASS
ok      math    0.002s
```

**對比 Node.js/Jest:**

```javascript
// math.js
function add(a, b) {
    return a + b;
}

function multiply(a, b) {
    return a * b;
}

module.exports = { add, multiply };
```

```javascript
// math.test.js
const { add, multiply } = require('./math');

describe('Math functions', () => {
    test('adds 2 + 3 to equal 5', () => {
        expect(add(2, 3)).toBe(5);
    });

    test.each([
        [2, 3, 6],
        [0, 5, 0],
        [-2, 3, -6],
    ])('multiplies %i * %i to equal %i', (a, b, expected) => {
        expect(multiply(a, b)).toBe(expected);
    });
});
```

### 基準測試（Benchmark）

Go 內置基準測試！

```go
// math_test.go
func BenchmarkAdd(b *testing.B) {
    for i := 0; i < b.N; i++ {
        Add(2, 3)
    }
}

func BenchmarkMultiply(b *testing.B) {
    for i := 0; i < b.N; i++ {
        Multiply(2, 3)
    }
}
```

```bash
# 運行基準測試
$ go test -bench=.
BenchmarkAdd-8          1000000000           0.25 ns/op
BenchmarkMultiply-8     1000000000           0.26 ns/op

# 顯示內存分配
$ go test -bench=. -benchmem
BenchmarkAdd-8          1000000000           0.25 ns/op        0 B/op          0 allocs/op
```

**Node.js 對比：**
需要額外的庫（如 benchmark.js）

### 示例測試（Example Tests）

```go
// math_test.go
func ExampleAdd() {
    result := Add(2, 3)
    fmt.Println(result)
    // Output: 5
}

func ExampleMultiply() {
    result := Multiply(4, 5)
    fmt.Println(result)
    // Output: 20
}
```

這些示例會：
1. 作為測試運行
2. 出現在文檔中

## go fmt - 格式化代碼

```bash
# 格式化當前文件
$ go fmt main.go

# 格式化當前包
$ go fmt

# 格式化所有包
$ go fmt ./...

# 使用 gofmt（更多選項）
$ gofmt -w main.go        # 寫入文件
$ gofmt -d main.go        # 顯示差異
$ gofmt -s -w main.go     # 簡化代碼
```

**對比 Prettier:**
```bash
# Prettier
$ npm install -D prettier
$ prettier --write .

# 配置文件 .prettierrc
{
  "semi": true,
  "singleQuote": true
}
```

**Go 的優勢：**
- 內置，無需配置
- 社區統一標準
- 沒有爭論空間

## go vet - 靜態分析

```bash
# 檢查可疑代碼
$ go vet

# 檢查所有包
$ go vet ./...
```

**示例：**

```go
// bad.go
package main

import "fmt"

func main() {
    // Printf 格式錯誤
    name := "Alice"
    fmt.Printf("Hello, %d", name)  // ❌ %d 用於整數，不是字符串
}
```

```bash
$ go vet bad.go
# bad.go:7: Printf format %d has arg name of wrong type string
```

**對比 ESLint:**
```bash
# ESLint
$ npm install -D eslint
$ eslint .

# 配置文件 .eslintrc.js
module.exports = {
    rules: {
        'no-unused-vars': 'error'
    }
};
```

## go doc - 查看文檔

```bash
# 查看包文檔
$ go doc fmt

# 查看函數文檔
$ go doc fmt.Println

# 查看本地包
$ go doc ./mypackage

# 啟動本地文檔服務器
$ go doc -http=:6060
# 訪問 http://localhost:6060
```

**對比 Node.js:**
```bash
# Node.js
$ npm docs package-name     # 打開網頁
```

## go clean - 清理構建文件

```bash
# 清理構建產物
$ go clean

# 清理測試緩存
$ go clean -testcache

# 清理模組緩存
$ go clean -modcache

# 全部清理
$ go clean -cache -testcache -modcache
```

**對比 Node.js:**
```bash
# Node.js
$ rm -rf node_modules
$ npm cache clean --force
```

## go install - 安裝工具

```bash
# 安裝可執行文件到 $GOPATH/bin
$ go install github.com/golangci/golangci-lint/cmd/golangci-lint@latest

# 安裝到自定義位置
$ GOBIN=/usr/local/bin go install github.com/tool/cmd@latest
```

**對比 npm:**
```bash
# npm
$ npm install -g typescript
$ npm install -g nodemon
```

## 開發工作流程

### Node.js 典型工作流

```json
// package.json
{
  "scripts": {
    "start": "node app.js",
    "dev": "nodemon app.js",
    "build": "webpack --mode production",
    "test": "jest",
    "lint": "eslint .",
    "format": "prettier --write ."
  }
}
```

```bash
$ npm run dev      # 開發
$ npm test         # 測試
$ npm run lint     # 檢查
$ npm run build    # 構建
```

### Go 典型工作流

**方式 1：直接使用命令**

```bash
# 開發（運行）
$ go run .

# 測試
$ go test ./...

# 格式化
$ go fmt ./...

# 檢查
$ go vet ./...

# 構建
$ go build -o bin/myapp
```

**方式 2：使用 Makefile**

```makefile
# Makefile
.PHONY: run build test clean

run:
	go run .

build:
	go build -ldflags="-s -w" -o bin/myapp

test:
	go test -v -cover ./...

fmt:
	go fmt ./...

vet:
	go vet ./...

lint:
	golangci-lint run

clean:
	go clean
	rm -rf bin/

all: fmt vet test build
```

```bash
$ make run
$ make test
$ make build
```

**方式 3：使用 Task（現代替代方案）**

```yaml
# Taskfile.yml
version: '3'

tasks:
  run:
    cmds:
      - go run .

  build:
    cmds:
      - go build -ldflags="-s -w" -o bin/myapp

  test:
    cmds:
      - go test -v -cover ./...

  lint:
    cmds:
      - go fmt ./...
      - go vet ./...
      - golangci-lint run

  clean:
    cmds:
      - go clean
      - rm -rf bin/
```

```bash
$ task run
$ task test
$ task build
```

## 性能分析

### CPU 分析

```bash
# 生成 CPU profile
$ go test -cpuprofile=cpu.out -bench=.

# 分析 profile
$ go tool pprof cpu.out

# Web 界面
$ go tool pprof -http=:8080 cpu.out
```

### 內存分析

```bash
# 生成內存 profile
$ go test -memprofile=mem.out -bench=.

# 分析
$ go tool pprof mem.out
```

### 競態檢測

```bash
# 檢測數據競態
$ go test -race

# 運行時檢測
$ go run -race main.go

# 構建帶競態檢測的版本
$ go build -race
```

**對比 Node.js:**
```bash
# Node.js 性能分析
$ node --prof app.js
$ node --prof-process isolate-*.log

# 或使用 clinic
$ npm install -g clinic
$ clinic doctor -- node app.js
```

## 實用技巧

### 1. 監聽文件變化（類似 nodemon）

使用 Air：
```bash
# 安裝
$ go install github.com/cosmtrek/air@latest

# 運行
$ air

# .air.toml 配置文件
root = "."
tmp_dir = "tmp"

[build]
cmd = "go build -o ./tmp/main ."
bin = "tmp/main"
include_ext = ["go", "tpl", "tmpl", "html"]
exclude_dir = ["assets", "tmp", "vendor"]
```

### 2. 生成代碼

```bash
# 使用 go generate
// main.go
//go:generate go run generate_config.go

$ go generate
```

### 3. 查看依賴圖

```bash
# 可視化依賴
$ go mod graph | grep mypackage
```

### 4. 下載依賴以供離線使用

```bash
# 下載並 vendor
$ go mod vendor

# 使用 vendor 構建
$ go build -mod=vendor
```

## 重點總結

### 核心命令

1. **go run** - 快速運行（開發時）
2. **go build** - 構建可執行文件（部署）
3. **go test** - 運行測試（內置）
4. **go fmt** - 格式化代碼（統一風格）
5. **go vet** - 靜態分析（發現問題）

### Go vs Node.js 工具鏈對比

| 功能 | Node.js | Go | Go 優勢 |
|------|---------|-----|---------|
| **運行** | node | go run | 無 |
| **構建** | webpack/rollup | go build | 內置 |
| **測試** | jest/mocha | go test | 內置 |
| **格式化** | prettier | go fmt | 內置，統一標準 |
| **檢查** | eslint | go vet | 內置 |
| **基準測試** | benchmark.js | go test -bench | 內置 |
| **交叉編譯** | pkg | GOOS/GOARCH | 簡單快速 |

### 最佳實踐

1. **開發時使用 go run**
   - 快速迭代
   - 自動編譯

2. **部署前使用 go build**
   - 優化選項
   - 交叉編譯

3. **持續使用 go fmt**
   - 保持代碼風格統一
   - 配置編輯器自動格式化

4. **頻繁運行 go test**
   - TDD 開發
   - 確保質量

5. **使用 Makefile/Task**
   - 簡化常用命令
   - 團隊協作

## 練習題

### 基礎題

1. **命令練習**
   - 創建一個簡單的 Go 程序
   - 分別使用 go run 和 go build 運行
   - 對比執行時間和產物

2. **交叉編譯**
   - 為 Linux、Windows、Mac 構建同一個程序
   - 檢查文件大小
   - 在對應平台上測試

3. **測試入門**
   - 編寫一個函數及其測試
   - 使用 go test -v 運行
   - 查看覆蓋率報告

### 進階題

4. **優化構建**
   - 使用不同的 ldflags 構建
   - 對比文件大小
   - 測試性能差異

5. **Makefile 配置**
   - 創建 Makefile
   - 包含 run、build、test、clean 目標
   - 添加變量和條件

6. **基準測試**
   - 實現兩種算法
   - 編寫基準測試對比性能
   - 分析結果

### 實戰題

7. **CI/CD 腳本**
   - 編寫構建腳本
   - 包含測試、檢查、構建步驟
   - 生成多平台二進制文件

8. **性能分析**
   - 編寫一個有性能問題的程序
   - 使用 pprof 分析
   - 優化並對比

9. **開發環境設置**
   - 配置編輯器自動 go fmt
   - 設置 git pre-commit hook 運行測試
   - 創建完整的開發工作流

## 下一章預告

Part 2 開始！下一章我們將學習 Go 的變量和常量，包括：
- var、const、:= 的區別
- 對比 JavaScript 的 let/const/var
- 類型推導
- 零值概念
- 作用域規則

準備好深入學習 Go 語法了嗎？
