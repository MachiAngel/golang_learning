# 第三章：第一個 Go 程序

## 章節概述

本章將帶領 Node.js 開發者編寫第一個 Go 程序。我們將對比 Node.js 和 Go 的 "Hello World"，深入理解 Go 的基本結構、package 系統、main 函數等核心概念。

## Hello World 對比

### Node.js 版本

```javascript
// hello.js
console.log("Hello, World!");

// 運行
// $ node hello.js
// Hello, World!
```

### Go 版本

```go
// hello.go
package main

import "fmt"

func main() {
    fmt.Println("Hello, World!")
}

// 運行
// $ go run hello.go
// Hello, World!
```

## 代碼結構詳解

### 1. Package 聲明

```go
package main
```

**Go 的 package 系統：**
- 每個 Go 文件都必須聲明 package
- `package main` 表示這是一個可執行程序
- 其他 package 名稱表示庫文件

**對比 Node.js:**

```javascript
// Node.js 沒有強制的 package 聲明
// 但有模組系統

// CommonJS
module.exports = { /* ... */ };

// ES Modules
export default { /* ... */ };
export const something = {};
```

#### Package 命名規則

```go
// 可執行程序
package main  // 必須有 main 函數

// 庫/包
package utils
package handlers
package models

// 包名通常是小寫，與目錄名一致
```

### 2. Import 導入

```go
import "fmt"

// 多個導入
import (
    "fmt"
    "os"
    "strings"
)
```

**對比 Node.js:**

```javascript
// CommonJS
const fs = require('fs');
const path = require('path');
const { readFile } = require('fs').promises;

// ES Modules
import fs from 'fs';
import path from 'path';
import { readFile } from 'fs/promises';
```

#### Import 路徑

```go
// 標準庫
import "fmt"
import "net/http"
import "encoding/json"

// 第三方庫（使用完整路徑）
import "github.com/gin-gonic/gin"
import "github.com/stretchr/testify/assert"

// 本地包（相對於 module root）
import "myproject/internal/handlers"
import "myproject/pkg/utils"
```

**對比 Node.js:**

```javascript
// 核心模組
const fs = require('fs');

// 第三方模組（從 node_modules）
const express = require('express');
const lodash = require('lodash');

// 本地模組（相對路徑）
const utils = require('./utils');
const handlers = require('../handlers');
```

#### Import 別名

```go
// Go 的 import 別名
import (
    f "fmt"           // 別名為 f
    _ "image/png"     // 僅執行初始化函數
    . "math"          // 導入到當前命名空間（不推薦）
)

func main() {
    f.Println("Using alias")  // 使用別名
    // Sqrt(4)                 // 使用 . 導入後可以直接調用
}
```

**對比 Node.js:**

```javascript
// Node.js 的 import 別名
import { readFile as read } from 'fs/promises';
import * as path from 'path';

// 或 CommonJS
const read = require('fs').promises.readFile;
```

### 3. Main 函數

```go
func main() {
    fmt.Println("Hello, World!")
}
```

**Go 的 main 函數特點：**
- 程序入口點
- 必須在 `package main` 中
- 不接受參數
- 不返回值
- 一個 package 只能有一個 main 函數

**對比 Node.js:**

```javascript
// Node.js 沒有強制的 main 函數
// 文件從上到下執行

console.log("This runs immediately");

function main() {
    console.log("This is just a regular function");
}

// 常見模式
if (require.main === module) {
    main();  // 僅在直接運行時執行
}
```

## 詳細示例

### 示例 1：基本輸出

**Node.js:**

```javascript
// output.js
console.log("Hello, World!");
console.log("Number:", 42);
console.log("Boolean:", true);
console.log("String:", "Go");

// 格式化輸出
const name = "Alice";
const age = 30;
console.log(`Name: ${name}, Age: ${age}`);

// 多種輸出方式
console.log("Log");
console.error("Error");
console.warn("Warning");
console.info("Info");
```

**Go:**

```go
// output.go
package main

import "fmt"

func main() {
    // 基本輸出
    fmt.Println("Hello, World!")
    fmt.Println("Number:", 42)
    fmt.Println("Boolean:", true)
    fmt.Println("String:", "Go")

    // 格式化輸出
    name := "Alice"
    age := 30
    fmt.Printf("Name: %s, Age: %d\n", name, age)

    // 或使用 Sprintf
    message := fmt.Sprintf("Name: %s, Age: %d", name, age)
    fmt.Println(message)

    // 不同的輸出函數
    fmt.Print("No newline")
    fmt.Println(" With newline")
    fmt.Printf("Formatted: %d\n", 42)
}
```

#### fmt 包的常用函數

```go
package main

import "fmt"

func main() {
    // Print: 不換行
    fmt.Print("Hello")
    fmt.Print(" World")  // 輸出：Hello World

    // Println: 自動換行
    fmt.Println("Hello")
    fmt.Println("World")
    // 輸出：
    // Hello
    // World

    // Printf: 格式化輸出
    name := "Alice"
    age := 30
    fmt.Printf("Name: %s, Age: %d\n", name, age)
    // 輸出：Name: Alice, Age: 30

    // Sprintf: 返回字符串
    str := fmt.Sprintf("Hello, %s!", name)
    fmt.Println(str)  // 輸出：Hello, Alice!
}
```

**格式化占位符：**

```go
package main

import "fmt"

func main() {
    // 常用格式化占位符
    fmt.Printf("%v\n", 42)           // 默認格式：42
    fmt.Printf("%d\n", 42)           // 十進制：42
    fmt.Printf("%b\n", 42)           // 二進制：101010
    fmt.Printf("%x\n", 42)           // 十六進制：2a
    fmt.Printf("%f\n", 3.14159)      // 浮點數：3.141590
    fmt.Printf("%.2f\n", 3.14159)    // 保留2位小數：3.14
    fmt.Printf("%s\n", "hello")      // 字符串：hello
    fmt.Printf("%q\n", "hello")      // 帶引號的字符串："hello"
    fmt.Printf("%t\n", true)         // 布爾值：true
    fmt.Printf("%T\n", 42)           // 類型：int
    fmt.Printf("%p\n", &age)         // 指針地址：0x...

    // 類似 JSON.stringify
    type Person struct {
        Name string
        Age  int
    }
    p := Person{"Alice", 30}
    fmt.Printf("%v\n", p)            // {Alice 30}
    fmt.Printf("%+v\n", p)           // {Name:Alice Age:30}
    fmt.Printf("%#v\n", p)           // main.Person{Name:"Alice", Age:30}
}
```

### 示例 2：命令行參數

**Node.js:**

```javascript
// args.js
// 命令行參數在 process.argv 中
console.log("All arguments:", process.argv);
// process.argv[0]: node 路徑
// process.argv[1]: 腳本路徑
// process.argv[2+]: 用戶參數

// 獲取用戶參數
const args = process.argv.slice(2);
console.log("User arguments:", args);

if (args.length > 0) {
    console.log(`Hello, ${args[0]}!`);
} else {
    console.log("Hello, World!");
}

// 運行：
// $ node args.js Alice Bob
// User arguments: [ 'Alice', 'Bob' ]
// Hello, Alice!
```

**Go:**

```go
// args.go
package main

import (
    "fmt"
    "os"
)

func main() {
    // 命令行參數在 os.Args 中
    fmt.Println("All arguments:", os.Args)
    // os.Args[0]: 程序名
    // os.Args[1+]: 用戶參數

    // 獲取用戶參數
    args := os.Args[1:]
    fmt.Println("User arguments:", args)

    if len(args) > 0 {
        fmt.Printf("Hello, %s!\n", args[0])
    } else {
        fmt.Println("Hello, World!")
    }
}

// 運行：
// $ go run args.go Alice Bob
// User arguments: [Alice Bob]
// Hello, Alice!
```

### 示例 3：讀取用戶輸入

**Node.js:**

```javascript
// input.js
const readline = require('readline');

const rl = readline.createInterface({
    input: process.stdin,
    output: process.stdout
});

rl.question('What is your name? ', (name) => {
    console.log(`Hello, ${name}!`);
    rl.close();
});

// 或使用 Promise
const question = (query) => {
    return new Promise((resolve) => {
        rl.question(query, resolve);
    });
};

(async () => {
    const name = await question('What is your name? ');
    console.log(`Hello, ${name}!`);
    rl.close();
})();
```

**Go:**

```go
// input.go
package main

import (
    "bufio"
    "fmt"
    "os"
)

func main() {
    // 創建讀取器
    reader := bufio.NewReader(os.Stdin)

    fmt.Print("What is your name? ")

    // 讀取一行
    name, _ := reader.ReadString('\n')

    // 去除換行符
    name = name[:len(name)-1]

    fmt.Printf("Hello, %s!\n", name)
}

// 更簡單的方式
func main2() {
    var name string

    fmt.Print("What is your name? ")
    fmt.Scanln(&name)  // 注意要傳指針

    fmt.Printf("Hello, %s!\n", name)
}
```

### 示例 4：多文件程序

**Node.js:**

```javascript
// utils.js
function greet(name) {
    return `Hello, ${name}!`;
}

function add(a, b) {
    return a + b;
}

module.exports = { greet, add };

// main.js
const { greet, add } = require('./utils');

console.log(greet("Alice"));
console.log("2 + 3 =", add(2, 3));
```

**Go:**

```go
// utils.go
package main

func greet(name string) string {
    return "Hello, " + name + "!"
}

func add(a, b int) int {
    return a + b
}

// main.go
package main

import "fmt"

func main() {
    fmt.Println(greet("Alice"))
    fmt.Println("2 + 3 =", add(2, 3))
}

// 運行：go run main.go utils.go
// 或：go run .
```

**注意：**
- Go 的同一個 package 下的所有文件共享命名空間
- 不需要顯式 import
- 大寫開頭的函數/變量是公開的（exported）
- 小寫開頭的是私有的（package 內部）

### 示例 5：完整的 Hello World 變體

**多語言問候：**

**Node.js:**

```javascript
// greetings.js
const greetings = {
    en: "Hello",
    es: "Hola",
    fr: "Bonjour",
    de: "Guten Tag",
    zh: "你好",
    ja: "こんにちは"
};

function greet(name, lang = 'en') {
    const greeting = greetings[lang] || greetings['en'];
    return `${greeting}, ${name}!`;
}

// 命令行使用
if (require.main === module) {
    const [name = 'World', lang = 'en'] = process.argv.slice(2);
    console.log(greet(name, lang));
}

module.exports = { greet };

// 運行：
// $ node greetings.js Alice zh
// 你好, Alice!
```

**Go:**

```go
// greetings.go
package main

import (
    "fmt"
    "os"
)

var greetings = map[string]string{
    "en": "Hello",
    "es": "Hola",
    "fr": "Bonjour",
    "de": "Guten Tag",
    "zh": "你好",
    "ja": "こんにちは",
}

func greet(name, lang string) string {
    greeting, ok := greetings[lang]
    if !ok {
        greeting = greetings["en"]
    }
    return fmt.Sprintf("%s, %s!", greeting, name)
}

func main() {
    name := "World"
    lang := "en"

    args := os.Args[1:]
    if len(args) > 0 {
        name = args[0]
    }
    if len(args) > 1 {
        lang = args[1]
    }

    fmt.Println(greet(name, lang))
}

// 運行：
// $ go run greetings.go Alice zh
// 你好, Alice!
```

## 編譯 vs 解釋執行

### Node.js: JIT 編譯 + 解釋執行

```bash
# Node.js 直接運行源碼
$ node hello.js
# V8 引擎會 JIT 編譯 JavaScript 為機器碼

# 文件結構
hello.js          (源碼)
# 沒有編譯產物，運行時編譯

# 優點：快速迭代，無需編譯步驟
# 缺點：需要 Node.js 運行時，啟動較慢
```

### Go: 提前編譯（AOT）

```bash
# 方式 1：直接運行（內部編譯為臨時文件）
$ go run hello.go
# Go 編譯為臨時可執行文件並運行

# 方式 2：構建可執行文件
$ go build hello.go
# 生成 hello 可執行文件（Unix）或 hello.exe（Windows）

$ ./hello
Hello, World!

# 文件結構
hello.go          (源碼)
hello             (可執行文件，5-10MB)

# 優點：
# - 無需運行時
# - 啟動極快
# - 性能最優

# 缺點：
# - 需要編譯步驟
# - 可執行文件較大（包含完整運行時）
```

### 編譯選項

```bash
# 基本編譯
go build main.go

# 指定輸出文件名
go build -o myapp main.go

# 交叉編譯（編譯為其他平台）
GOOS=linux GOARCH=amd64 go build -o myapp-linux main.go
GOOS=windows GOARCH=amd64 go build -o myapp.exe main.go
GOOS=darwin GOARCH=amd64 go build -o myapp-mac main.go

# 減小可執行文件大小
go build -ldflags="-s -w" main.go
# -s: 去除符號表
# -w: 去除 DWARF 調試信息

# 顯示編譯詳情
go build -v main.go

# 編譯當前目錄的所有 .go 文件
go build .
```

**對比 Node.js 打包：**

```bash
# Node.js 使用 pkg 或 nexe 打包為可執行文件
npm install -g pkg

pkg index.js
# 生成 index-linux, index-macos, index-win.exe

# 但文件通常很大（~50MB，包含 Node.js 運行時）
```

## Init 函數

```go
package main

import "fmt"

// init 函數在 main 之前自動執行
// 類似於 Node.js 的模組頂層代碼
func init() {
    fmt.Println("init() called")
}

func main() {
    fmt.Println("main() called")
}

// 輸出：
// init() called
// main() called
```

**對比 Node.js:**

```javascript
// Node.js 模組頂層代碼自動執行
console.log("Module loaded");  // 類似 init()

function main() {
    console.log("main() called");
}

main();

// 輸出：
// Module loaded
// main() called
```

### 多個 init 函數

```go
package main

import "fmt"

func init() {
    fmt.Println("init 1")
}

func init() {
    fmt.Println("init 2")
}

func init() {
    fmt.Println("init 3")
}

func main() {
    fmt.Println("main")
}

// 輸出（按順序執行）：
// init 1
// init 2
// init 3
// main
```

## 重點總結

### Go 程序結構

1. **Package 聲明**
   - 每個文件必須聲明 package
   - `package main` 表示可執行程序
   - 其他名稱表示庫

2. **Import 語句**
   - 導入需要的包
   - 未使用的 import 會導致編譯錯誤
   - 使用 goimports 自動管理

3. **Main 函數**
   - 程序入口點
   - 必須在 package main 中
   - 不接受參數，不返回值

4. **Init 函數**
   - 在 main 之前執行
   - 用於初始化
   - 可以有多個

### Node.js vs Go 對比

| 特性 | Node.js | Go |
|------|---------|-----|
| **入口點** | 文件頂層 | main() 函數 |
| **模組系統** | CommonJS/ES Modules | package + import |
| **執行方式** | 解釋執行 | 編譯執行 |
| **輸出函數** | console.log | fmt.Println |
| **初始化** | 模組頂層代碼 | init() 函數 |
| **命令行參數** | process.argv | os.Args |
| **標準輸入** | readline | bufio.Reader |

### 最佳實踐

1. **文件命名**
   - 使用小寫和下劃線：`user_handler.go`
   - 測試文件：`user_handler_test.go`

2. **Package 命名**
   - 簡短、小寫、單數
   - 與目錄名一致

3. **代碼組織**
   - 一個目錄一個 package
   - 相關功能放在同一 package

4. **可見性**
   - 大寫開頭：公開（Exported）
   - 小寫開頭：私有（package 內部）

## 練習題

### 基礎題

1. **Hello World 變體**
   - 編寫一個程序，輸出你的名字和年齡
   - 使用 `fmt.Printf` 格式化輸出
   - 嘗試不同的格式化占位符

2. **命令行參數**
   - 編寫一個程序接受兩個命令行參數
   - 輸出問候語，如：`Hello, Alice and Bob!`
   - 如果沒有參數，輸出默認問候

3. **用戶輸入**
   - 編寫一個程序詢問用戶姓名和年齡
   - 輸出：`Hello, [name]! You are [age] years old.`

### 進階題

4. **多語言問候**
   - 實現支持 5 種語言的問候程序
   - 通過命令行參數指定語言
   - 如果語言不支持，使用英文

5. **簡單計算器**
   - 接受三個命令行參數：數字1、運算符、數字2
   - 支持 +、-、*、/ 運算
   - 輸出計算結果

6. **多文件組織**
   - 創建 `utils.go` 包含輔助函數
   - 創建 `main.go` 調用這些函數
   - 使用 `go run .` 運行

### 實戰題

7. **交互式程序**
   - 創建一個簡單的命令行菜單
   - 提供多個選項（如：1. 打招呼 2. 退出）
   - 循環接受用戶輸入直到退出

8. **格式化輸出練習**
   - 創建一個結構體表示 Person（姓名、年齡、城市）
   - 使用不同的 fmt 占位符輸出
   - 嘗試 `%v`、`%+v`、`%#v` 的區別

9. **對比實現**
   - 用 Node.js 和 Go 分別實現相同功能的程序
   - 對比代碼量、執行速度、可執行文件大小
   - 記錄你的觀察

## 下一章預告

下一章我們將學習 Go 的模組管理系統，包括：
- go.mod 文件詳解
- 依賴管理（go get, go mod tidy）
- 對比 package.json 和 npm
- 工作空間和模組結構
- 版本管理和語義化版本

現在你已經會寫 Go 程序了，讓我們深入了解如何管理項目依賴！
