# 第二章：Go 環境安裝與配置

## 章節概述

本章將指導 Node.js 開發者如何安裝和配置 Go 開發環境。我們會對比 Node.js 的安裝流程（nvm/node），讓你快速上手 Go 的開發環境設置。

## Node.js vs Go 安裝對比

### 概念對比

| 對比項 | Node.js | Go |
|-------|---------|-----|
| **版本管理器** | nvm / n / fnm | gvm / g (較少使用) |
| **常見做法** | 使用 nvm 管理多版本 | 直接安裝官方版本 |
| **安裝包管理** | npm / yarn / pnpm | go get / go install |
| **全局工具位置** | ~/.nvm/versions/node/*/bin | ~/go/bin |
| **項目依賴** | node_modules/ | go.mod + 緩存 |
| **環境變量** | NODE_PATH (較少用) | GOPATH, GOROOT |

## 安裝 Go

### macOS 安裝

#### 方式 1：使用 Homebrew（推薦）

```bash
# 類似於使用 brew install node
brew install go

# 驗證安裝
go version
# 輸出：go version go1.21.5 darwin/amd64

# 查看 Go 環境信息
go env

# 重要的環境變量
go env GOROOT    # Go 安裝路徑，類似 Node.js 安裝目錄
go env GOPATH    # Go 工作空間，類似 npm 全局包路徑
```

**對比 Node.js:**
```bash
# Node.js with Homebrew
brew install node
node --version
npm --version
which node  # /usr/local/bin/node
```

#### 方式 2：官方安裝包

```bash
# 1. 從 https://go.dev/dl/ 下載 .pkg 文件
# 2. 雙擊安裝
# 3. 驗證安裝
go version

# Go 會自動設置環境變量
# 安裝位置：/usr/local/go
```

### Linux 安裝（Ubuntu/Debian）

```bash
# 方式 1：使用系統包管理器
sudo apt update
sudo apt install golang-go

# 方式 2：官方二進制（推薦，版本更新）
# 1. 下載最新版本
wget https://go.dev/dl/go1.21.5.linux-amd64.tar.gz

# 2. 解壓到 /usr/local
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf go1.21.5.linux-amd64.tar.gz

# 3. 設置環境變量（添加到 ~/.bashrc 或 ~/.zshrc）
echo 'export PATH=$PATH:/usr/local/go/bin' >> ~/.bashrc
echo 'export GOPATH=$HOME/go' >> ~/.bashrc
echo 'export PATH=$PATH:$GOPATH/bin' >> ~/.bashrc

# 4. 重新加載配置
source ~/.bashrc

# 5. 驗證
go version
```

**對比 Node.js (使用 nvm):**
```bash
# Node.js 安裝
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.0/install.sh | bash
source ~/.bashrc
nvm install 20
nvm use 20
node --version
```

### Windows 安裝

#### 使用 MSI 安裝包

```powershell
# 1. 從 https://go.dev/dl/ 下載 .msi 文件
# 2. 運行安裝程序
# 3. 安裝到 C:\Go (默認)
# 4. 環境變量會自動配置

# 驗證安裝（在 PowerShell 或 CMD 中）
go version

# 查看環境變量
go env GOROOT
go env GOPATH
# 默認 GOPATH: C:\Users\YourName\go
```

#### 使用 Chocolatey

```powershell
# 類似於 npm 的 Windows 包管理器
choco install golang

# 驗證
go version
```

**對比 Node.js:**
```powershell
# Node.js 使用官方安裝包
# 或使用 Chocolatey
choco install nodejs

# 或使用 nvm-windows
```

## 配置開發環境

### 理解 Go 的環境變量

```bash
# 查看所有 Go 環境變量
go env

# 重要的環境變量
```

#### 1. GOROOT
```bash
# GOROOT：Go 安裝目錄（類似 Node.js 安裝路徑）
go env GOROOT
# macOS/Linux: /usr/local/go
# Windows: C:\Go

# Node.js 對比
which node
# /usr/local/bin/node
```

#### 2. GOPATH
```bash
# GOPATH：Go 工作空間（類似 npm 全局包路徑）
go env GOPATH
# macOS/Linux: ~/go
# Windows: C:\Users\YourName\go

# GOPATH 目錄結構
# ~/go/
#   ├── bin/      # 可執行文件（類似 npm global bin）
#   ├── pkg/      # 編譯的包（緩存）
#   └── src/      # 源代碼（Go 1.11 前使用，現在較少用）

# Node.js 對比
npm config get prefix
# /usr/local (全局包位置)
# ~/.npm (緩存)
```

#### 3. GOMODCACHE
```bash
# Go modules 緩存（類似 ~/.npm）
go env GOMODCACHE
# ~/go/pkg/mod

# Node.js 對比
npm config get cache
# ~/.npm
```

### 配置 GOPATH 和 PATH

**Linux/macOS:**

```bash
# 編輯 ~/.bashrc 或 ~/.zshrc
vim ~/.bashrc

# 添加以下內容
export GOROOT=/usr/local/go
export GOPATH=$HOME/go
export PATH=$PATH:$GOROOT/bin:$GOPATH/bin

# 使配置生效
source ~/.bashrc

# 驗證
echo $GOPATH
echo $PATH | grep go
```

**Windows:**

```powershell
# 設置環境變量（PowerShell 管理員權限）
[System.Environment]::SetEnvironmentVariable('GOPATH', 'C:\Users\YourName\go', 'User')
[System.Environment]::SetEnvironmentVariable('Path', $env:Path + ';C:\Go\bin;C:\Users\YourName\go\bin', 'User')

# 或通過圖形界面：
# 控制面板 → 系統 → 高級系統設置 → 環境變量
```

**對比 Node.js 環境變量:**
```bash
# Node.js 較少需要手動配置環境變量
# nvm 會自動處理

# 查看 Node.js 相關路徑
echo $NVM_DIR
echo $PATH | grep node
```

## IDE 配置

### VS Code 配置（推薦）

#### 1. 安裝 Go 擴展

```bash
# 在 VS Code 中安裝 Go 擴展
# Extension ID: golang.go
```

**對比 Node.js:**
```bash
# Node.js 常用擴展
# - ESLint
# - Prettier
# - JavaScript (ES6) code snippets
```

#### 2. 安裝 Go 工具

```bash
# 打開 VS Code 命令面板 (Cmd/Ctrl + Shift + P)
# 運行：Go: Install/Update Tools
# 選擇所有工具並安裝

# 這些工具包括：
# - gopls: Go language server（類似 TypeScript language server）
# - dlv: Delve debugger（類似 Node.js debugger）
# - staticcheck: 靜態分析（類似 ESLint）
# - go-outline: 文檔大綱
# 等等...
```

#### 3. VS Code 配置文件

創建 `.vscode/settings.json`:

```json
{
  "go.useLanguageServer": true,
  "go.lintTool": "golangci-lint",
  "go.lintOnSave": "workspace",
  "go.formatTool": "goimports",
  "go.formatOnSave": true,
  "[go]": {
    "editor.formatOnSave": true,
    "editor.codeActionsOnSave": {
      "source.organizeImports": true
    }
  },
  "go.testFlags": ["-v"],
  "go.testTimeout": "30s",
  "go.coverOnSave": false,
  "go.coverageDecorator": {
    "type": "gutter"
  }
}
```

**對比 Node.js/TypeScript VS Code 配置:**
```json
{
  "editor.formatOnSave": true,
  "editor.defaultFormatter": "esbenp.prettier-vscode",
  "eslint.validate": ["javascript", "typescript"],
  "typescript.updateImportsOnFileMove.enabled": "always",
  "[javascript]": {
    "editor.defaultFormatter": "esbenp.prettier-vscode"
  }
}
```

#### 4. 調試配置

創建 `.vscode/launch.json`:

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Launch Package",
      "type": "go",
      "request": "launch",
      "mode": "debug",
      "program": "${fileDirname}"
    },
    {
      "name": "Launch File",
      "type": "go",
      "request": "launch",
      "mode": "debug",
      "program": "${file}"
    },
    {
      "name": "Attach to Process",
      "type": "go",
      "request": "attach",
      "mode": "local",
      "processId": 0
    }
  ]
}
```

**對比 Node.js 調試配置:**
```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "type": "node",
      "request": "launch",
      "name": "Launch Program",
      "skipFiles": ["<node_internals>/**"],
      "program": "${workspaceFolder}/app.js"
    }
  ]
}
```

### GoLand / IntelliJ IDEA

```bash
# GoLand 是 JetBrains 專為 Go 開發的 IDE
# 類似於 WebStorm 對於 JavaScript

# 優勢：
# - 開箱即用，無需配置
# - 強大的代碼分析
# - 優秀的重構工具
# - 內置數據庫工具

# 下載：https://www.jetbrains.com/go/
```

## 驗證安裝

### 創建第一個 Go 程序

```bash
# 1. 創建項目目錄
mkdir -p ~/projects/hello-go
cd ~/projects/hello-go

# 2. 初始化 Go module（類似 npm init）
go mod init hello-go
# 創建 go.mod 文件（類似 package.json）

# 3. 創建 main.go
cat > main.go << 'EOF'
package main

import "fmt"

func main() {
    fmt.Println("Hello, Go!")
}
EOF

# 4. 運行程序
go run main.go
# 輸出：Hello, Go!

# 5. 構建可執行文件
go build
# 生成 hello-go 可執行文件

# 6. 運行可執行文件
./hello-go
# 輸出：Hello, Go!
```

**對比 Node.js:**
```bash
# Node.js 等效操作
mkdir -p ~/projects/hello-node
cd ~/projects/hello-node

# 初始化項目
npm init -y

# 創建 index.js
cat > index.js << 'EOF'
console.log("Hello, Node.js!");
EOF

# 運行
node index.js

# Node.js 不需要構建步驟（直接解釋執行）
```

### 檢查環境配置

```bash
# 檢查 Go 版本
go version
# go version go1.21.5 darwin/amd64

# 檢查環境變量
go env GOROOT
go env GOPATH
go env GOMODCACHE

# 檢查已安裝的工具
ls $GOPATH/bin
# 應該看到：gopls, dlv, staticcheck 等

# 檢查 Go modules 是否啟用
go env GO111MODULE
# on（默認）

# 檢查 GOPROXY（Go 包代理，類似 npm registry）
go env GOPROXY
# https://proxy.golang.org,direct
```

## Go Modules 詳解

### Go Modules vs npm

```bash
# Node.js 項目初始化
npm init
# 創建 package.json

# Go 項目初始化
go mod init myproject
# 創建 go.mod
```

#### package.json vs go.mod

**package.json:**
```json
{
  "name": "myproject",
  "version": "1.0.0",
  "dependencies": {
    "express": "^4.18.0",
    "lodash": "^4.17.21"
  },
  "devDependencies": {
    "jest": "^29.0.0"
  }
}
```

**go.mod:**
```go
module myproject

go 1.21

require (
    github.com/gin-gonic/gin v1.9.1
    github.com/stretchr/testify v1.8.4
)

require (
    // 間接依賴（自動管理）
    github.com/bytedance/sonic v1.9.1 // indirect
    github.com/chenzhuoyu/base64x v0.0.0-20221115062448-fe3a3abad311 // indirect
    // ...
)
```

### 常用 Go 命令

```bash
# 添加依賴（類似 npm install）
go get github.com/gin-gonic/gin@latest
# 會更新 go.mod 和 go.sum

# 安裝所有依賴（類似 npm install）
go mod download

# 清理未使用的依賴（類似 npm prune）
go mod tidy

# 查看依賴樹（類似 npm list）
go mod graph

# 驗證依賴（檢查 go.sum）
go mod verify

# 更新依賴
go get -u github.com/gin-gonic/gin
go get -u ./...  # 更新所有依賴
```

**對比 Node.js 命令:**
```bash
# npm 命令對比
npm install express          # go get
npm install                  # go mod download
npm update                   # go get -u
npm list                     # go mod graph
npm prune                    # go mod tidy
```

### go.sum 文件

```bash
# go.sum（類似 package-lock.json）
# 記錄依賴的精確版本和校驗和
```

**go.sum 示例:**
```
github.com/gin-gonic/gin v1.9.1 h1:4idEAncQnU5cB7BeOkPtxjfCSye0AAm1R0RVIqJ+Jmg=
github.com/gin-gonic/gin v1.9.1/go.mod h1:hPrL7YrpYKXt5YId3A/Tnip5kqbEAP+KLuI3SUcPTeU=
```

**對比 package-lock.json:**
```json
{
  "name": "myproject",
  "lockfileVersion": 3,
  "packages": {
    "node_modules/express": {
      "version": "4.18.2",
      "resolved": "https://registry.npmjs.org/express/-/express-4.18.2.tgz",
      "integrity": "sha512-5/PsL6iGPdfQ/lKM1UuielYgv3BUoJfz1aUwU9vHZ+J7gyvwdQXFEBIEIaxeGf0GIcreATNyBExtalisDbuMqQ=="
    }
  }
}
```

## 配置 GOPROXY（重要！）

### 為什麼需要 GOPROXY

```bash
# 默認的 proxy.golang.org 在某些地區可能很慢
# 類似於 npm 使用淘寶鏡像

# Node.js 設置 npm 鏡像
npm config set registry https://registry.npmmirror.com

# Go 設置 GOPROXY
go env -w GOPROXY=https://goproxy.cn,direct
# 或
go env -w GOPROXY=https://goproxy.io,direct
```

### 常用的 GOPROXY

```bash
# 中國大陸推薦
go env -w GOPROXY=https://goproxy.cn,direct

# 全球
go env -w GOPROXY=https://proxy.golang.org,direct

# 多個代理（逗號分隔，失敗時嘗試下一個）
go env -w GOPROXY=https://goproxy.cn,https://goproxy.io,direct

# 查看當前設置
go env GOPROXY
```

### GOPRIVATE（私有倉庫）

```bash
# 設置私有倉庫（類似 npm 的 .npmrc）
go env -w GOPRIVATE=github.com/mycompany/*,gitlab.com/myteam/*

# 這些倉庫不會通過 GOPROXY 獲取
```

**對比 Node.js .npmrc:**
```bash
# .npmrc
registry=https://registry.npmmirror.com
@mycompany:registry=https://npm.mycompany.com
```

## 工作目錄結構對比

### Node.js 項目結構

```
my-node-project/
├── node_modules/      # 依賴（每個項目一份）
├── src/
│   ├── index.js
│   ├── routes/
│   └── utils/
├── test/
├── package.json       # 項目配置
├── package-lock.json  # 鎖文件
└── .gitignore
```

### Go 項目結構

```
my-go-project/
├── cmd/              # 可執行程序入口
│   └── server/
│       └── main.go
├── internal/         # 私有代碼（不能被外部 import）
│   ├── handlers/
│   └── utils/
├── pkg/              # 公共庫（可以被外部 import）
│   └── models/
├── go.mod            # 模組定義
├── go.sum            # 依賴鎖文件
└── .gitignore

# 依賴在全局緩存中：~/go/pkg/mod
```

### 推薦的 .gitignore

```bash
# .gitignore for Go
# 二進制文件
*.exe
*.exe~
*.dll
*.so
*.dylib

# 測試文件
*.test
*.out

# 依賴緩存（本地）
/vendor/

# Go workspace file
go.work

# IDE
.idea/
.vscode/
*.swp
*.swo

# OS
.DS_Store
Thumbs.db
```

**對比 Node.js .gitignore:**
```bash
# .gitignore for Node.js
node_modules/
npm-debug.log
.env
dist/
build/
.DS_Store
```

## 開發工具鏈

### 代碼格式化

```bash
# Go 內置格式化工具（類似 Prettier）
gofmt -w .          # 格式化所有文件
go fmt ./...        # 格式化所有包

# goimports（更智能，自動管理 import）
go install golang.org/x/tools/cmd/goimports@latest
goimports -w .

# Node.js 對比
npm install -g prettier
prettier --write .
```

### 代碼檢查

```bash
# 安裝 golangci-lint（集成多個 linter，類似 ESLint）
# macOS
brew install golangci-lint

# Linux
curl -sSfL https://raw.githubusercontent.com/golangci/golangci-lint/master/install.sh | sh -s -- -b $(go env GOPATH)/bin

# 運行檢查
golangci-lint run

# Node.js 對比
npm install -g eslint
eslint .
```

### 測試

```bash
# Go 內置測試工具（無需安裝 Jest/Mocha）
go test ./...              # 運行所有測試
go test -v ./...           # 詳細輸出
go test -cover ./...       # 顯示覆蓋率
go test -race ./...        # 競態檢測

# Node.js 對比
npm install -D jest
npm test
```

## 重點總結

### 安裝要點

1. **Go 安裝簡單**
   - 無需版本管理器（通常）
   - 官方安裝包即可
   - 自動配置環境變量

2. **重要環境變量**
   - `GOROOT`: Go 安裝目錄
   - `GOPATH`: 工作空間（~/go）
   - `GOPROXY`: 包代理（建議配置）

3. **Go Modules**
   - `go.mod`: 類似 package.json
   - `go.sum`: 類似 package-lock.json
   - 全局緩存依賴

### Node.js 開發者的轉變

| Node.js | Go | 說明 |
|---------|-----|------|
| `npm init` | `go mod init` | 初始化項目 |
| `npm install` | `go mod download` | 安裝依賴 |
| `npm install express` | `go get github.com/gin-gonic/gin` | 添加依賴 |
| `node index.js` | `go run main.go` | 運行程序 |
| 無 | `go build` | 構建二進制 |
| `npm test` | `go test ./...` | 運行測試 |
| Prettier | `gofmt` / `goimports` | 代碼格式化 |
| ESLint | `golangci-lint` | 代碼檢查 |

### IDE 推薦

1. **VS Code + Go 擴展**（免費，推薦）
2. **GoLand**（付費，功能最強）
3. **Vim/Neovim + vim-go**（極客選擇）

## 練習題

### 基礎題

1. **安裝驗證**
   - 在你的系統上安裝 Go
   - 驗證 `go version` 輸出
   - 檢查 `GOPATH` 和 `GOROOT`

2. **環境配置**
   - 配置 GOPROXY 為國內鏡像
   - 將 `$GOPATH/bin` 添加到 PATH
   - 驗證配置是否生效

3. **VS Code 配置**
   - 安裝 Go 擴展
   - 安裝所有推薦的 Go 工具
   - 創建一個簡單的 Go 文件並驗證語法高亮和自動完成

### 進階題

4. **對比練習**
   - 創建一個 Node.js 項目和一個 Go 項目
   - 對比項目結構和配置文件
   - 記錄兩者的異同

5. **工具鏈實踐**
   - 安裝 `golangci-lint`
   - 編寫一個有問題的 Go 代碼
   - 使用 linter 發現並修復問題

6. **Modules 練習**
   - 創建一個新的 Go module
   - 添加幾個第三方依賴
   - 檢查 `go.mod` 和 `go.sum` 的變化
   - 嘗試 `go mod tidy` 和 `go mod graph`

### 實戰題

7. **開發環境遷移**
   - 如果你有一個現有的 Node.js 開發環境
   - 設計一個方案在同一台機器上同時支持 Node.js 和 Go 開發
   - 避免環境變量衝突

8. **項目模板**
   - 創建一個 Go 項目模板
   - 包含合理的目錄結構
   - 配置好 VS Code 設置
   - 添加 Makefile 用於常用命令

9. **CI/CD 準備**
   - 研究如何在 GitHub Actions 中運行 Go
   - 對比 Node.js 的 CI 配置
   - 編寫一個簡單的 `.github/workflows/go.yml`

## 下一章預告

下一章我們將編寫第一個 Go 程序 "Hello World"，包括：
- package 和 main 函數的概念
- 對比 Node.js 的模組系統
- fmt 包的使用
- 編譯和運行的區別
- 命令行參數處理

準備好開始編寫 Go 代碼了嗎？讓我們繼續！
