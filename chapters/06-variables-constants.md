# 第六章：變量與常量聲明

## 章節概述

本章將深入講解 Go 的變量和常量聲明，對比 JavaScript 的 var、let、const，幫助 Node.js 開發者理解 Go 的聲明方式、類型推導、零值概念和作用域規則。

## 變量聲明對比

### JavaScript 的變量聲明

```javascript
// JavaScript - 三種聲明方式
var name = "Alice";        // 函數作用域，可重新賦值
let age = 30;              // 塊作用域，可重新賦值
const PI = 3.14;           // 塊作用域，不可重新賦值

// 可以不初始化
let x;                     // undefined
console.log(x);            // undefined

// const 必須初始化
const y;                   // ❌ SyntaxError
```

### Go 的變量聲明

```go
// Go - 多種聲明方式
var name string = "Alice"  // 完整聲明
var age = 30               // 類型推導
city := "New York"         // 短變量聲明（最常用）

// 變量必須初始化或有零值
var x int                  // 0（零值）
fmt.Println(x)             // 0

// 常量
const PI = 3.14
```

## 變量聲明方式

### 方式 1：var 聲明（帶類型）

```go
package main

import "fmt"

func main() {
    // 聲明並初始化
    var name string = "Alice"
    var age int = 30
    var isActive bool = true

    fmt.Println(name, age, isActive)
}
```

**對應 JavaScript:**
```javascript
let name = "Alice";        // 但 JS 沒有類型聲明
let age = 30;
let isActive = true;
```

### 方式 2：var 聲明（類型推導）

```go
package main

import "fmt"

func main() {
    // Go 編譯器自動推導類型
    var name = "Alice"      // string
    var age = 30            // int
    var price = 19.99       // float64
    var isActive = true     // bool

    // 查看類型
    fmt.Printf("%T\n", name)     // string
    fmt.Printf("%T\n", age)      // int
    fmt.Printf("%T\n", price)    // float64
    fmt.Printf("%T\n", isActive) // bool
}
```

**對應 JavaScript:**
```javascript
let name = "Alice";        // 運行時動態類型
let age = 30;
let price = 19.99;
let isActive = true;

// JavaScript 的類型是動態的
console.log(typeof name);      // "string"
console.log(typeof age);       // "number"
console.log(typeof price);     // "number"
console.log(typeof isActive);  // "boolean"
```

### 方式 3：短變量聲明（:=，最常用）

```go
package main

import "fmt"

func main() {
    // 短變量聲明：最簡潔的方式
    name := "Alice"
    age := 30
    price := 19.99
    isActive := true

    fmt.Println(name, age, price, isActive)
}
```

**重要限制：**
```go
package main

import "fmt"

// ❌ := 只能在函數內使用
// name := "Alice"  // 語法錯誤！

// ✅ 包級別必須使用 var
var globalName = "Global"

func main() {
    // ✅ := 在函數內可以使用
    localName := "Local"
    fmt.Println(globalName, localName)
}
```

**對應 JavaScript:**
```javascript
// JavaScript 沒有這個限制
let globalName = "Global";    // 全局

function main() {
    let localName = "Local";  // 局部
}
```

### 方式 4：批量聲明

```go
package main

import "fmt"

func main() {
    // 批量聲明（同類型）
    var x, y, z int = 1, 2, 3

    // 批量聲明（不同類型）
    var (
        name   string = "Alice"
        age    int    = 30
        salary float64 = 50000.0
    )

    // 批量聲明（類型推導）
    var (
        city    = "New York"
        country = "USA"
        zip     = 10001
    )

    // 短變量批量聲明
    a, b, c := 1, 2, 3
    name2, age2 := "Bob", 25

    fmt.Println(x, y, z)
    fmt.Println(name, age, salary)
    fmt.Println(city, country, zip)
    fmt.Println(a, b, c)
    fmt.Println(name2, age2)
}
```

**對應 JavaScript:**
```javascript
// JavaScript 批量聲明
let x = 1, y = 2, z = 3;

// 或使用解構
let [a, b, c] = [1, 2, 3];
let {name, age} = {name: "Alice", age: 30};
```

## 零值（Zero Value）

這是 Go 的重要概念！

### Go 的零值

```go
package main

import "fmt"

func main() {
    // 聲明但不初始化，會有默認零值
    var i int           // 0
    var f float64       // 0.0
    var b bool          // false
    var s string        // ""（空字符串）

    fmt.Printf("int: %d\n", i)        // 0
    fmt.Printf("float64: %f\n", f)    // 0.000000
    fmt.Printf("bool: %t\n", b)       // false
    fmt.Printf("string: %q\n", s)     // ""

    // 複合類型的零值
    var arr [3]int      // [0 0 0]
    var ptr *int        // nil
    var slice []int     // nil
    var m map[string]int // nil

    fmt.Printf("array: %v\n", arr)    // [0 0 0]
    fmt.Printf("pointer: %v\n", ptr)  // <nil>
    fmt.Printf("slice: %v\n", slice)  // []
    fmt.Printf("map: %v\n", m)        // map[]
}
```

**對應 JavaScript:**
```javascript
// JavaScript 未初始化的變量是 undefined
let i;
let f;
let b;
let s;

console.log(i);  // undefined
console.log(f);  // undefined
console.log(b);  // undefined
console.log(s);  // undefined

// JavaScript 沒有"零值"的概念
// 需要手動初始化
let num = 0;
let str = "";
let bool = false;
```

### 零值的好處

```go
package main

import "fmt"

type Counter struct {
    count int  // 零值為 0
}

func main() {
    var c Counter
    // count 已經是 0，無需顯式初始化
    fmt.Println(c.count)  // 0

    c.count++
    fmt.Println(c.count)  // 1
}
```

**對應 JavaScript:**
```javascript
class Counter {
    constructor() {
        this.count = 0;  // 必須顯式初始化
    }
}

let c = new Counter();
console.log(c.count);  // 0
```

## 常量

### Go 的常量

```go
package main

import "fmt"

func main() {
    // 常量聲明
    const PI = 3.14159
    const AppName = "MyApp"
    const MaxSize = 100

    // 常量不能重新賦值
    // PI = 3.14  // ❌ 編譯錯誤！

    // 批量常量聲明
    const (
        StatusOK    = 200
        StatusError = 500
        StatusNotFound = 404
    )

    // 類型化常量
    const TypedPI float64 = 3.14159

    fmt.Println(PI, AppName, MaxSize)
    fmt.Println(StatusOK, StatusError, StatusNotFound)
}
```

**對應 JavaScript:**
```javascript
// JavaScript const
const PI = 3.14159;
const APP_NAME = "MyApp";
const MAX_SIZE = 100;

// const 不能重新賦值
// PI = 3.14;  // ❌ TypeError

// 但對象屬性可以修改！
const obj = {name: "Alice"};
obj.name = "Bob";  // ✅ 允許！
obj.age = 30;      // ✅ 允許！

// Go 沒有這個問題，常量是真正不可變的
```

### iota（常量生成器）

這是 Go 特有的強大功能！

```go
package main

import "fmt"

func main() {
    // iota：自動遞增的常量
    const (
        Sunday    = iota  // 0
        Monday            // 1
        Tuesday           // 2
        Wednesday         // 3
        Thursday          // 4
        Friday            // 5
        Saturday          // 6
    )

    fmt.Println(Sunday, Monday, Tuesday)  // 0 1 2

    // 從 1 開始
    const (
        January = iota + 1  // 1
        February            // 2
        March               // 3
        April               // 4
    )

    fmt.Println(January, February, March)  // 1 2 3

    // 跳過值
    const (
        _ = iota      // 0（忽略）
        KB = 1 << (10 * iota)  // 1 << 10 = 1024
        MB                      // 1 << 20 = 1048576
        GB                      // 1 << 30 = 1073741824
    )

    fmt.Println(KB, MB, GB)  // 1024 1048576 1073741824

    // 位掩碼
    const (
        Read = 1 << iota   // 1 << 0 = 1
        Write              // 1 << 1 = 2
        Execute            // 1 << 2 = 4
    )

    fmt.Println(Read, Write, Execute)  // 1 2 4
}
```

**對應 JavaScript:**
```javascript
// JavaScript 沒有 iota，需要手動定義
const Sunday = 0;
const Monday = 1;
const Tuesday = 2;
// ...

// 或使用對象
const Weekdays = {
    Sunday: 0,
    Monday: 1,
    Tuesday: 2,
    Wednesday: 3,
    Thursday: 4,
    Friday: 5,
    Saturday: 6
};

// 或使用 enum（TypeScript）
enum Weekday {
    Sunday,    // 0
    Monday,    // 1
    Tuesday,   // 2
    // ...
}
```

## 類型轉換

### Go 的類型轉換（顯式）

```go
package main

import "fmt"

func main() {
    // Go 不支持隱式類型轉換！
    var i int = 42
    var f float64 = float64(i)  // 必須顯式轉換
    var u uint = uint(i)

    fmt.Println(i, f, u)  // 42 42 42

    // ❌ 不能隱式轉換
    // var x float64 = i  // 編譯錯誤！

    // 字符串轉換
    var num int = 65
    var str string = string(num)  // "A"（ASCII）
    fmt.Println(str)

    // 整數和字符串之間的轉換（使用 strconv）
    import "strconv"

    var numStr string = strconv.Itoa(42)      // "42"
    var numInt, _ = strconv.Atoi("42")        // 42

    fmt.Println(numStr, numInt)
}
```

**對應 JavaScript:**
```javascript
// JavaScript 支持隱式類型轉換（有時很危險）
let i = 42;
let f = i;           // 自動轉換
let s = i + "";      // "42"（數字轉字符串）
let n = "42" - 0;    // 42（字符串轉數字）

// 顯式轉換
let num = 42;
let str = String(num);     // "42"
let parsed = parseInt("42"); // 42

// 隱式轉換的問題
console.log("5" + 3);      // "53"（字符串拼接）
console.log("5" - 3);      // 2（數字減法）
console.log([] + []);      // ""（空字符串）
console.log({} + []);      // "[object Object]"
```

## 作用域

### Go 的作用域規則

```go
package main

import "fmt"

// 包級別變量（package scope）
var globalVar = "global"

func main() {
    // 函數級別變量
    var funcVar = "function"

    // 塊級作用域
    if true {
        var blockVar = "block"
        fmt.Println(globalVar)  // ✅ 可訪問
        fmt.Println(funcVar)    // ✅ 可訪問
        fmt.Println(blockVar)   // ✅ 可訪問
    }

    // fmt.Println(blockVar)    // ❌ 不可訪問

    // for 循環作用域
    for i := 0; i < 3; i++ {
        fmt.Println(i)  // ✅ 可訪問
    }
    // fmt.Println(i)  // ❌ 不可訪問

    // 短變量聲明的陷阱
    var x = 1
    if true {
        x := 2  // 這是新變量！不是重新賦值
        fmt.Println(x)  // 2
    }
    fmt.Println(x)  // 1（外層的 x 沒變）
}
```

**對應 JavaScript:**
```javascript
// 全局作用域
var globalVar = "global";  // 或 window.globalVar

function main() {
    // 函數作用域
    var funcVar = "function";

    // 塊作用域（let/const）
    if (true) {
        let blockVar = "block";
        console.log(globalVar);  // ✅ 可訪問
        console.log(funcVar);    // ✅ 可訪問
        console.log(blockVar);   // ✅ 可訪問
    }

    // console.log(blockVar);    // ❌ ReferenceError

    // var 的問題：沒有塊作用域
    if (true) {
        var varInBlock = "var";
    }
    console.log(varInBlock);  // ✅ 可訪問（var 沒有塊作用域）

    // let/const 的塊作用域
    let x = 1;
    if (true) {
        let x = 2;  // 新變量（遮蔽外層）
        console.log(x);  // 2
    }
    console.log(x);  // 1
}
```

### 變量遮蔽（Shadowing）

```go
package main

import "fmt"

var x = "global"

func main() {
    fmt.Println(x)  // "global"

    x := "local"    // 遮蔽全局變量
    fmt.Println(x)  // "local"

    {
        x := "block"  // 遮蔽函數變量
        fmt.Println(x)  // "block"
    }

    fmt.Println(x)  // "local"
}

func other() {
    fmt.Println(x)  // "global"（不受 main 影響）
}
```

**對應 JavaScript:**
```javascript
let x = "global";

function main() {
    console.log(x);  // "global"

    let x = "local";  // 遮蔽
    console.log(x);   // "local"

    {
        let x = "block";
        console.log(x);  // "block"
    }

    console.log(x);  // "local"
}

function other() {
    console.log(x);  // "global"
}
```

## 變量的可見性

### 導出（Exported）vs 未導出

```go
// mypackage/mypackage.go
package mypackage

// 大寫開頭：導出（公開）
var PublicVar = "public"
const PublicConst = 100

// 小寫開頭：未導出（私有）
var privateVar = "private"
const privateConst = 200

func PublicFunc() string {
    return "public function"
}

func privateFunc() string {
    return "private function"
}
```

```go
// main.go
package main

import (
    "fmt"
    "myproject/mypackage"
)

func main() {
    // ✅ 可以訪問導出的變量/函數
    fmt.Println(mypackage.PublicVar)
    fmt.Println(mypackage.PublicConst)
    fmt.Println(mypackage.PublicFunc())

    // ❌ 不能訪問未導出的變量/函數
    // fmt.Println(mypackage.privateVar)    // 編譯錯誤
    // fmt.Println(mypackage.privateFunc()) // 編譯錯誤
}
```

**對應 JavaScript:**
```javascript
// mymodule.js
const publicVar = "public";
const privateVar = "private";

function publicFunc() {
    return "public function";
}

function privateFunc() {
    return "private function";
}

// 顯式導出
module.exports = {
    publicVar,
    publicFunc
    // privateVar 和 privateFunc 沒有導出
};

// 或 ES Modules
export { publicVar, publicFunc };
// privateVar 和 privateFunc 保持私有
```

## 實戰示例

### 示例 1：配置管理

**Go:**
```go
package config

// 配置常量
const (
    AppName    = "MyApp"
    AppVersion = "1.0.0"
    MaxRetries = 3
)

// 環境配置
var (
    Debug      bool
    ServerPort int
    DBHost     string
)

// 初始化
func init() {
    Debug = true
    ServerPort = 8080
    DBHost = "localhost"
}
```

**JavaScript:**
```javascript
// config.js
const APP_NAME = "MyApp";
const APP_VERSION = "1.0.0";
const MAX_RETRIES = 3;

let debug = true;
let serverPort = 8080;
let dbHost = "localhost";

module.exports = {
    APP_NAME,
    APP_VERSION,
    MAX_RETRIES,
    debug,
    serverPort,
    dbHost
};
```

### 示例 2：狀態碼管理

**Go:**
```go
package http

const (
    StatusOK                  = 200
    StatusCreated             = 201
    StatusBadRequest          = 400
    StatusUnauthorized        = 401
    StatusNotFound            = 404
    StatusInternalServerError = 500
)

// 使用 iota
const (
    LevelDebug = iota  // 0
    LevelInfo          // 1
    LevelWarn          // 2
    LevelError         // 3
)
```

**JavaScript:**
```javascript
// http.js
const StatusCode = {
    OK: 200,
    CREATED: 201,
    BAD_REQUEST: 400,
    UNAUTHORIZED: 401,
    NOT_FOUND: 404,
    INTERNAL_SERVER_ERROR: 500
};

const LogLevel = {
    DEBUG: 0,
    INFO: 1,
    WARN: 2,
    ERROR: 3
};
```

## 重點總結

### 變量聲明方式

| 方式 | 語法 | 使用場景 |
|------|------|----------|
| **var 完整** | `var name string = "value"` | 明確類型 |
| **var 推導** | `var name = "value"` | 包級別變量 |
| **短聲明** | `name := "value"` | 函數內（最常用）|
| **批量聲明** | `var (...)` | 多個相關變量 |

### 常量

1. **const 聲明** - 真正不可變
2. **iota** - 自動遞增（Go 特有）
3. **類型化/非類型化** - 靈活性

### 零值

| 類型 | 零值 |
|------|------|
| `int`, `float64` | `0` |
| `bool` | `false` |
| `string` | `""` |
| `pointer`, `slice`, `map` | `nil` |

### Go vs JavaScript

| 特性 | Go | JavaScript |
|------|-----|------------|
| **類型** | 靜態類型 | 動態類型 |
| **類型轉換** | 顯式 | 隱式（危險）|
| **零值** | 有默認值 | undefined |
| **常量** | 真正不可變 | 對象可變 |
| **作用域** | 塊作用域 | var 函數作用域，let/const 塊作用域 |
| **可見性** | 大小寫控制 | 手動 export |

### 最佳實踐

1. **優先使用 :=** （函數內）
2. **使用有意義的變量名**
3. **利用零值**（無需顯式初始化為 0）
4. **常量用於不變的值**
5. **使用 iota 定義枚舉**

## 練習題

### 基礎題

1. **變量聲明**
   - 用三種方式聲明變量
   - 打印變量類型
   - 觀察零值

2. **常量練習**
   - 定義一週的七天（使用 iota）
   - 定義文件權限（使用位掩碼）
   - 定義 HTTP 狀態碼

3. **作用域**
   - 創建全局、函數、塊作用域變量
   - 測試變量遮蔽
   - 理解 := 和 = 的區別

### 進階題

4. **類型轉換**
   - 實現 int 和 string 的互相轉換
   - 處理轉換錯誤
   - 對比 JavaScript 的轉換行為

5. **配置系統**
   - 設計一個配置包
   - 包含常量和變量
   - 實現環境切換（dev/prod）

6. **枚舉實現**
   - 使用 iota 實現顏色枚舉
   - 實現 String() 方法
   - 對比 TypeScript enum

### 實戰題

7. **狀態機**
   - 使用 iota 定義狀態
   - 實現狀態轉換
   - 添加狀態驗證

8. **類型安全**
   - 對比 Go 和 JavaScript 的類型安全
   - 創建會在 JS 中出錯但在 Go 中編譯失敗的示例
   - 理解靜態類型的優勢

9. **重構練習**
   - 將一個 JavaScript 代碼轉換為 Go
   - 處理動態類型到靜態類型的轉換
   - 記錄遇到的問題和解決方案

## 下一章預告

下一章我們將學習 Go 的數據類型系統，包括：
- 基本類型（int, float, string, bool）
- 複合類型（array, slice, map, struct）
- 對比 JavaScript 的類型系統
- 類型別名和自定義類型
- 類型斷言和反射

準備好深入了解 Go 的類型系統了嗎？
