# Chapter 56: 編譯與交叉編譯

## 概述

Go 的一大優勢是可以將代碼編譯成單一的靜態二進制文件，無需依賴運行時環境。本章將介紹 Go 的編譯過程、交叉編譯技術，以及與 Node.js 打包工具的對比。

## Go vs Node.js 部署對比

### Node.js 部署
```
挑戰：
- 需要 Node.js 運行時
- node_modules 依賴管理
- 平台相關的原生模塊
- 打包工具：pkg, nexe, webpack

優勢：
- 開發快速
- 熱重載
```

### Go 部署
```
優勢：
- 單一二進制文件
- 無運行時依賴
- 交叉編譯簡單
- 快速啟動

挑戰：
- 二進制文件較大（可優化）
- 無法動態修改代碼
```

## Go 編譯基礎

### 基本編譯命令

```bash
# 編譯當前目錄
go build

# 編譯指定文件
go build main.go

# 指定輸出文件名
go build -o myapp

# 編譯並運行
go run main.go

# 安裝到 $GOPATH/bin
go install
```

### 編譯選項

```bash
# 查看編譯過程
go build -v

# 查看編譯器執行的命令
go build -x

# 禁用編譯器優化和內聯
go build -gcflags="-N -l"

# 減小二進制文件大小
go build -ldflags="-s -w"

# 靜態編譯（不依賴動態庫）
CGO_ENABLED=0 go build
```

## 交叉編譯

Go 支持簡單的交叉編譯，通過設置環境變量即可編譯出不同平台的二進制文件。

### 基本交叉編譯

```bash
# 編譯 Linux 64 位
GOOS=linux GOARCH=amd64 go build -o myapp-linux-amd64

# 編譯 Windows 64 位
GOOS=windows GOARCH=amd64 go build -o myapp-windows-amd64.exe

# 編譯 macOS 64 位（Intel）
GOOS=darwin GOARCH=amd64 go build -o myapp-darwin-amd64

# 編譯 macOS ARM64（M1/M2）
GOOS=darwin GOARCH=arm64 go build -o myapp-darwin-arm64

# 編譯 Linux ARM64（樹莓派等）
GOOS=linux GOARCH=arm64 go build -o myapp-linux-arm64

# 編譯 Linux ARM（32位）
GOOS=linux GOARCH=arm GOARM=7 go build -o myapp-linux-arm
```

### 支持的平台

```bash
# 查看所有支持的平台
go tool dist list
```

常用平台：
```
GOOS         GOARCH
----         ------
linux        amd64, arm64, arm
darwin       amd64, arm64
windows      amd64, 386, arm64
freebsd      amd64, arm
openbsd      amd64, arm
android      amd64, arm64, arm
ios          arm64
```

## 實用編譯腳本

### 1. 多平台編譯 Makefile

```makefile
# Makefile
APP_NAME=myapp
VERSION=$(shell git describe --tags --always --dirty)
BUILD_TIME=$(shell date -u '+%Y-%m-%d_%H:%M:%S')
LDFLAGS=-ldflags "-s -w -X main.Version=$(VERSION) -X main.BuildTime=$(BUILD_TIME)"

.PHONY: build clean build-all

# 本地平台編譯
build:
	go build $(LDFLAGS) -o bin/$(APP_NAME) ./cmd/api

# 清理
clean:
	rm -rf bin/
	rm -rf dist/

# 所有平台編譯
build-all: clean
	mkdir -p dist

	# Linux AMD64
	GOOS=linux GOARCH=amd64 go build $(LDFLAGS) -o dist/$(APP_NAME)-linux-amd64 ./cmd/api

	# Linux ARM64
	GOOS=linux GOARCH=arm64 go build $(LDFLAGS) -o dist/$(APP_NAME)-linux-arm64 ./cmd/api

	# macOS AMD64
	GOOS=darwin GOARCH=amd64 go build $(LDFLAGS) -o dist/$(APP_NAME)-darwin-amd64 ./cmd/api

	# macOS ARM64
	GOOS=darwin GOARCH=arm64 go build $(LDFLAGS) -o dist/$(APP_NAME)-darwin-arm64 ./cmd/api

	# Windows AMD64
	GOOS=windows GOARCH=amd64 go build $(LDFLAGS) -o dist/$(APP_NAME)-windows-amd64.exe ./cmd/api

# 壓縮打包
package: build-all
	cd dist && tar -czf $(APP_NAME)-linux-amd64.tar.gz $(APP_NAME)-linux-amd64
	cd dist && tar -czf $(APP_NAME)-linux-arm64.tar.gz $(APP_NAME)-linux-arm64
	cd dist && tar -czf $(APP_NAME)-darwin-amd64.tar.gz $(APP_NAME)-darwin-amd64
	cd dist && tar -czf $(APP_NAME)-darwin-arm64.tar.gz $(APP_NAME)-darwin-arm64
	cd dist && zip $(APP_NAME)-windows-amd64.zip $(APP_NAME)-windows-amd64.exe

# 查看二進制文件信息
info:
	@echo "Binary size:"
	@ls -lh bin/$(APP_NAME)
	@echo "\nBuild info:"
	@go version -m bin/$(APP_NAME)
```

### 2. Shell 腳本

```bash
#!/bin/bash
# build.sh

APP_NAME="myapp"
VERSION=$(git describe --tags --always --dirty)
BUILD_TIME=$(date -u '+%Y-%m-%d_%H:%M:%S')
PLATFORMS=("linux/amd64" "linux/arm64" "darwin/amd64" "darwin/arm64" "windows/amd64")

echo "Building $APP_NAME version $VERSION"

for PLATFORM in "${PLATFORMS[@]}"; do
    GOOS=${PLATFORM%/*}
    GOARCH=${PLATFORM#*/}
    OUTPUT_NAME="${APP_NAME}-${GOOS}-${GOARCH}"

    if [ "$GOOS" = "windows" ]; then
        OUTPUT_NAME="${OUTPUT_NAME}.exe"
    fi

    echo "Building for $GOOS/$GOARCH..."

    env GOOS=$GOOS GOARCH=$GOARCH go build \
        -ldflags "-s -w -X main.Version=$VERSION -X main.BuildTime=$BUILD_TIME" \
        -o "dist/$OUTPUT_NAME" \
        ./cmd/api

    if [ $? -ne 0 ]; then
        echo "Failed to build for $GOOS/$GOARCH"
        exit 1
    fi
done

echo "Build completed successfully!"
```

### 3. 嵌入版本信息

```go
// main.go
package main

import (
    "fmt"
    "runtime"
)

var (
    Version   string
    BuildTime string
    GitCommit string
)

func main() {
    printVersion()
    // ... 應用邏輯
}

func printVersion() {
    fmt.Printf("Version: %s\n", Version)
    fmt.Printf("Build Time: %s\n", BuildTime)
    fmt.Printf("Git Commit: %s\n", GitCommit)
    fmt.Printf("Go Version: %s\n", runtime.Version())
    fmt.Printf("OS/Arch: %s/%s\n", runtime.GOOS, runtime.GOARCH)
}
```

編譯時注入：
```bash
go build -ldflags "\
  -X main.Version=1.0.0 \
  -X main.BuildTime=$(date -u '+%Y-%m-%d_%H:%M:%S') \
  -X main.GitCommit=$(git rev-parse HEAD)"
```

## 減小二進制文件大小

### 1. 基本優化

```bash
# 去除調試信息和符號表
go build -ldflags="-s -w"

# -s: 去除符號表
# -w: 去除 DWARF 調試信息
```

### 2. 使用 UPX 壓縮

```bash
# 安裝 UPX
# macOS: brew install upx
# Linux: apt-get install upx-ucl

# 編譯
go build -ldflags="-s -w" -o myapp

# 壓縮
upx --best --lzma myapp

# 查看效果
ls -lh myapp
```

**注意**：UPX 壓縮後的二進制文件可能在某些系統上無法運行，且啟動時需要解壓。

### 3. 禁用 CGO

```bash
# CGO 會引入 C 庫依賴
CGO_ENABLED=0 go build
```

### 4. 大小對比

```bash
# 標準編譯
go build -o app1
ls -lh app1
# 約 10-15 MB

# 優化編譯
go build -ldflags="-s -w" -o app2
ls -lh app2
# 約 7-10 MB

# UPX 壓縮
upx --best app2
ls -lh app2
# 約 2-4 MB
```

## 條件編譯（Build Tags）

### 使用 Build Tags

```go
// +build linux

package main

import "fmt"

func platformSpecific() {
    fmt.Println("Running on Linux")
}
```

```go
// +build darwin

package main

import "fmt"

func platformSpecific() {
    fmt.Println("Running on macOS")
}
```

```go
// +build windows

package main

import "fmt"

func platformSpecific() {
    fmt.Println("Running on Windows")
}
```

### 編譯指定標籤

```bash
# 編譯時指定 tags
go build -tags prod

# 多個 tags
go build -tags "prod mysql"
```

### 文件名約定

Go 也支持通過文件名進行條件編譯：

```
app_linux.go      # 僅 Linux
app_darwin.go     # 僅 macOS
app_windows.go    # 僅 Windows
app_amd64.go      # 僅 AMD64
app_arm64.go      # 僅 ARM64
app_linux_amd64.go # Linux + AMD64
```

## 靜態編譯

### 完全靜態編譯

```bash
# 禁用 CGO，確保沒有動態鏈接
CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o myapp

# -a: 重新構建所有包
# -installsuffix: 在包安裝目錄後綴
```

### 驗證靜態編譯

```bash
# Linux
ldd myapp
# 應顯示 "not a dynamic executable"

# macOS
otool -L myapp

# 查看依賴的共享庫
readelf -d myapp  # Linux
```

## 嵌入文件（Go 1.16+）

### embed 包

```go
package main

import (
    _ "embed"
    "fmt"
)

//go:embed version.txt
var version string

//go:embed static/*
var static embed.FS

func main() {
    fmt.Println("Version:", version)

    // 讀取嵌入的文件
    data, _ := static.ReadFile("static/index.html")
    fmt.Println(string(data))
}
```

### Web 應用嵌入靜態文件

```go
package main

import (
    "embed"
    "net/http"
)

//go:embed web/dist/*
var webFiles embed.FS

func main() {
    // 提供靜態文件服務
    http.Handle("/", http.FileServer(http.FS(webFiles)))
    http.ListenAndServe(":8080", nil)
}
```

## Node.js 打包對比

### pkg（Node.js）

```bash
# 安裝 pkg
npm install -g pkg

# 打包
pkg package.json

# 多平台打包
pkg -t node18-linux-x64,node18-macos-x64,node18-win-x64 app.js
```

**package.json**:
```json
{
  "bin": "app.js",
  "pkg": {
    "assets": ["views/**/*", "public/**/*"],
    "targets": ["node18-linux-x64", "node18-macos-x64", "node18-win-x64"]
  }
}
```

### nexe（Node.js）

```bash
# 安裝 nexe
npm install -g nexe

# 打包
nexe app.js -o myapp

# 多平台
nexe app.js -t linux-x64-18.0.0
nexe app.js -t mac-x64-18.0.0
nexe app.js -t windows-x64-18.0.0
```

### 對比

| 特性 | Go | pkg/nexe |
|------|-----|----------|
| 編譯速度 | 快 | 慢 |
| 文件大小 | 7-15 MB | 30-50 MB |
| 啟動速度 | 極快 | 慢 |
| 配置複雜度 | 簡單 | 中等 |
| 原生模塊支持 | N/A | 困難 |

## CI/CD 集成

### GitHub Actions

```yaml
# .github/workflows/build.yml
name: Build

on:
  push:
    tags:
      - 'v*'

jobs:
  build:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        goos: [linux, darwin, windows]
        goarch: [amd64, arm64]
        exclude:
          - goos: windows
            goarch: arm64

    steps:
      - uses: actions/checkout@v3

      - name: Set up Go
        uses: actions/setup-go@v4
        with:
          go-version: '1.21'

      - name: Build
        env:
          GOOS: ${{ matrix.goos }}
          GOARCH: ${{ matrix.goarch }}
        run: |
          OUTPUT_NAME=myapp-${{ matrix.goos }}-${{ matrix.goarch }}
          if [ "$GOOS" = "windows" ]; then
            OUTPUT_NAME="${OUTPUT_NAME}.exe"
          fi
          go build -ldflags="-s -w" -o dist/$OUTPUT_NAME ./cmd/api

      - name: Upload artifacts
        uses: actions/upload-artifact@v3
        with:
          name: binaries
          path: dist/*

      - name: Create Release
        uses: softprops/action-gh-release@v1
        if: startsWith(github.ref, 'refs/tags/')
        with:
          files: dist/*
```

### GitLab CI

```yaml
# .gitlab-ci.yml
stages:
  - build
  - release

build:
  stage: build
  image: golang:1.21
  script:
    - make build-all
  artifacts:
    paths:
      - dist/
    expire_in: 1 week

release:
  stage: release
  image: golang:1.21
  only:
    - tags
  script:
    - make package
  artifacts:
    paths:
      - dist/*.tar.gz
      - dist/*.zip
```

## 重點總結

### Go 編譯優勢

1. **單一二進制**
   - 無運行時依賴
   - 部署簡單
   - 版本管理容易

2. **交叉編譯**
   - 簡單的環境變量設置
   - 支持多平台
   - 無需專用工具

3. **性能優異**
   - 靜態編譯
   - 快速啟動
   - 低內存占用

### 最佳實踐

1. **版本管理**
   - 嵌入版本信息
   - 使用 Git tags
   - 記錄構建時間

2. **大小優化**
   - 使用 `-ldflags="-s -w"`
   - 考慮 UPX 壓縮
   - 禁用不需要的功能

3. **自動化**
   - 使用 Makefile
   - CI/CD 集成
   - 自動化發布

4. **測試**
   - 多平台測試
   - 驗證靜態鏈接
   - 檢查文件大小

## 練習題

### 練習 1: 創建多平台構建腳本

編寫一個腳本，為以下平台構建二進制文件：
- Linux (amd64, arm64)
- macOS (amd64, arm64)
- Windows (amd64)

並計算每個平台的文件大小。

### 練習 2: 嵌入版本信息

修改應用程序，嵌入以下信息：
- 版本號
- Git commit hash
- 構建時間
- 構建者

### 練習 3: 優化文件大小

對比以下編譯方式的文件大小：
1. 標準編譯
2. 去除調試信息
3. 靜態編譯
4. UPX 壓縮

### 練習 4: 條件編譯

實現不同環境的配置：
- 開發環境（dev）
- 測試環境（test）
- 生產環境（prod）

### 練習 5: 嵌入靜態文件

創建一個 Web 應用，將前端文件嵌入到二進制文件中。

---

下一章：[Chapter 57: Docker 容器化](57-docker.md)
