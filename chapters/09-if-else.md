# 第九章：流程控制 - if/else

## 章節概述

本章將深入講解 Go 的 if/else 條件語句，重點介紹 Go 特有的 if 初始化語句、作用域規則，以及與 JavaScript 的對比。

## if 語句基礎

### 基本語法

**Go:**
```go
package main

import "fmt"

func main() {
    score := 85

    // 基本 if
    if score >= 60 {
        fmt.Println("Pass")
    }

    // 注意：條件不需要括號！
    // 但大括號是必須的，即使只有一行

    // ❌ 錯誤寫法
    // if (score >= 60)  // 不需要括號（但可以有）
    // if score >= 60    // ❌ 缺少大括號
    //     fmt.Println("Pass")
}
```

**對比 JavaScript:**
```javascript
let score = 85;

// JavaScript 需要括號
if (score >= 60) {
    console.log("Pass");
}

// ✅ 單行可以省略大括號（但不推薦）
if (score >= 60)
    console.log("Pass");

// ✅ 括號是必須的
// if score >= 60 { }  // ❌ 語法錯誤
```

### if-else

**Go:**
```go
package main

import "fmt"

func main() {
    score := 50

    if score >= 60 {
        fmt.Println("Pass")
    } else {
        fmt.Println("Fail")
    }

    // 注意：else 必須與 if 的右大括號同行
    // ❌ 錯誤寫法
    // if score >= 60 {
    //     fmt.Println("Pass")
    // }
    // else {  // 語法錯誤！
    //     fmt.Println("Fail")
    // }
}
```

**對比 JavaScript:**
```javascript
let score = 50;

if (score >= 60) {
    console.log("Pass");
} else {
    console.log("Fail");
}

// ✅ JavaScript 允許 else 另起一行
if (score >= 60) {
    console.log("Pass");
}
else {  // ✅ 允許
    console.log("Fail");
}
```

### if-else if-else

**Go:**
```go
package main

import "fmt"

func main() {
    score := 85

    if score >= 90 {
        fmt.Println("A")
    } else if score >= 80 {
        fmt.Println("B")
    } else if score >= 70 {
        fmt.Println("C")
    } else if score >= 60 {
        fmt.Println("D")
    } else {
        fmt.Println("F")
    }
}
```

**對比 JavaScript:**
```javascript
let score = 85;

if (score >= 90) {
    console.log("A");
} else if (score >= 80) {
    console.log("B");
} else if (score >= 70) {
    console.log("C");
} else if (score >= 60) {
    console.log("D");
} else {
    console.log("F");
}

// 或使用 switch（更清晰）
// 或使用三元運算符（Go 沒有）
```

## if 初始化語句

這是 Go 的特色功能！

### 基本用法

```go
package main

import "fmt"

func main() {
    // if 可以包含初始化語句
    if score := 85; score >= 60 {
        fmt.Println("Pass, score:", score)
        // score 只在 if 作用域內可見
    }

    // fmt.Println(score)  // ❌ score 不存在

    // 實用示例：錯誤處理
    if err := doSomething(); err != nil {
        fmt.Println("Error:", err)
        return
    }
    // err 在這裡不可見
}

func doSomething() error {
    return nil
}
```

**對比 JavaScript:**
```javascript
// JavaScript 沒有 if 初始化語句

// 需要先聲明
let score = 85;
if (score >= 60) {
    console.log("Pass, score:", score);
}
// score 在外部仍然可見

// 使用塊作用域模擬
{
    let score = 85;
    if (score >= 60) {
        console.log("Pass, score:", score);
    }
}
// score 在這裡不可見
```

### 實際應用場景

#### 1. 錯誤處理

```go
package main

import (
    "fmt"
    "os"
)

func main() {
    // 讀取文件
    if data, err := os.ReadFile("config.json"); err != nil {
        fmt.Println("Error reading file:", err)
    } else {
        fmt.Println("File content:", string(data))
        // data 和 err 在 else 中也可見
    }
    // data 和 err 在這裡不可見

    // 不使用初始化語句的寫法（作用域更大）
    data, err := os.ReadFile("config.json")
    if err != nil {
        fmt.Println("Error:", err)
    }
    fmt.Println(string(data))  // data 仍然可見
}
```

**對比 JavaScript:**
```javascript
const fs = require('fs').promises;

// JavaScript 需要 try-catch 或 .then().catch()
try {
    const data = await fs.readFile("config.json");
    console.log("File content:", data.toString());
} catch (err) {
    console.log("Error reading file:", err);
}
// data 在這裡不可見（塊作用域）

// 或使用 Promise
fs.readFile("config.json")
    .then(data => {
        console.log("File content:", data.toString());
    })
    .catch(err => {
        console.log("Error:", err);
    });
```

#### 2. Map 查找

```go
package main

import "fmt"

func main() {
    ages := map[string]int{
        "Alice": 30,
        "Bob":   25,
    }

    // 檢查鍵是否存在
    if age, ok := ages["Alice"]; ok {
        fmt.Println("Alice's age:", age)
    } else {
        fmt.Println("Alice not found")
    }

    // 不使用初始化語句
    age, ok := ages["Charlie"]
    if ok {
        fmt.Println("Charlie's age:", age)
    } else {
        fmt.Println("Charlie not found")
    }
}
```

**對比 JavaScript:**
```javascript
const ages = {
    "Alice": 30,
    "Bob": 25
};

// JavaScript 檢查鍵
if ("Alice" in ages) {
    console.log("Alice's age:", ages["Alice"]);
}

// 或使用 hasOwnProperty
if (ages.hasOwnProperty("Alice")) {
    console.log("Alice's age:", ages.Alice);
}

// ES6 Map
const agesMap = new Map([
    ["Alice", 30],
    ["Bob", 25]
]);

if (agesMap.has("Alice")) {
    console.log("Alice's age:", agesMap.get("Alice"));
}
```

#### 3. 類型斷言

```go
package main

import "fmt"

func main() {
    var i interface{} = "hello"

    // 類型斷言
    if str, ok := i.(string); ok {
        fmt.Println("String:", str)
    } else {
        fmt.Println("Not a string")
    }

    // 不安全的方式（可能 panic）
    // str := i.(string)  // 如果不是 string 會 panic
}
```

**對比 JavaScript:**
```javascript
let i = "hello";

// JavaScript 運行時類型檢查
if (typeof i === "string") {
    console.log("String:", i);
} else {
    console.log("Not a string");
}

// instanceof 檢查
if (i instanceof String) {
    // ...
}
```

## 條件表達式

### 布爾值

```go
package main

import "fmt"

func main() {
    isActive := true

    // ✅ 直接使用布爾值
    if isActive {
        fmt.Println("Active")
    }

    // ✅ 取反
    if !isActive {
        fmt.Println("Inactive")
    }

    // ❌ 不能用非布爾值
    // if 1 { }           // 編譯錯誤
    // if "hello" { }     // 編譯錯誤
    // if nil { }         // 編譯錯誤
}
```

**對比 JavaScript:**
```javascript
let isActive = true;

// ✅ 布爾值
if (isActive) {
    console.log("Active");
}

// ✅ JavaScript 允許真值/假值
if (1) {
    console.log("Truthy");
}

if ("hello") {
    console.log("Truthy");
}

// 假值：false, 0, "", null, undefined, NaN
if ("") {
    console.log("Never executes");
}
```

### 比較運算符

```go
package main

import "fmt"

func main() {
    x, y := 10, 20

    if x < y {
        fmt.Println("x is less than y")
    }

    if x != y {
        fmt.Println("x is not equal to y")
    }

    // 字符串比較
    if "apple" < "banana" {
        fmt.Println("Lexicographic order")
    }

    // ❌ 不同類型不能比較
    // if x < "20" { }  // 編譯錯誤
}
```

**對比 JavaScript:**
```javascript
let x = 10, y = 20;

if (x < y) {
    console.log("x is less than y");
}

// ✅ JavaScript 允許不同類型比較（隱式轉換）
if (x < "20") {
    console.log("String converted to number");
}

// 但可能有意外行為
console.log(10 < "9");   // false
console.log(10 < "11");  // true
console.log("10" < "9"); // true（字符串比較）
```

### 邏輯運算符

```go
package main

import "fmt"

func main() {
    age := 25
    hasLicense := true

    // AND
    if age >= 18 && hasLicense {
        fmt.Println("Can drive")
    }

    // OR
    if age < 18 || !hasLicense {
        fmt.Println("Cannot drive")
    }

    // 複雜條件
    score := 85
    attendance := 90
    if score >= 60 && attendance >= 80 || score >= 90 {
        fmt.Println("Pass")
    }

    // 推薦：使用括號明確優先級
    if (score >= 60 && attendance >= 80) || score >= 90 {
        fmt.Println("Pass")
    }
}
```

**對比 JavaScript:**
```javascript
let age = 25;
let hasLicense = true;

// AND
if (age >= 18 && hasLicense) {
    console.log("Can drive");
}

// OR
if (age < 18 || !hasLicense) {
    console.log("Cannot drive");
}

// JavaScript 還可以利用短路求值
let name = userName || "Guest";  // 默認值
let value = isValid && getValue();  // 條件執行
```

## 常見模式

### 1. 提前返回（Early Return）

```go
package main

import "fmt"

func divide(a, b float64) (float64, error) {
    // 提前返回錯誤
    if b == 0 {
        return 0, fmt.Errorf("division by zero")
    }

    // 正常邏輯
    return a / b, nil
}

func processUser(user *User) error {
    if user == nil {
        return fmt.Errorf("user is nil")
    }

    if user.Name == "" {
        return fmt.Errorf("name is empty")
    }

    if user.Age < 0 {
        return fmt.Errorf("age is negative")
    }

    // 所有驗證通過，執行邏輯
    // ...
    return nil
}

type User struct {
    Name string
    Age  int
}
```

**對比 JavaScript:**
```javascript
function divide(a, b) {
    // 提前返回
    if (b === 0) {
        throw new Error("division by zero");
    }

    return a / b;
}

function processUser(user) {
    if (!user) {
        throw new Error("user is null");
    }

    if (!user.name) {
        throw new Error("name is empty");
    }

    if (user.age < 0) {
        throw new Error("age is negative");
    }

    // 執行邏輯
}
```

### 2. 守衛子句（Guard Clauses）

```go
package main

import "fmt"

func getDiscount(price float64, isMember bool, age int) float64 {
    // 守衛子句
    if price <= 0 {
        return 0
    }

    discount := 0.0

    if isMember {
        discount += 0.1  // 10% 會員折扣
    }

    if age < 18 {
        discount += 0.05  // 5% 學生折扣
    } else if age >= 65 {
        discount += 0.15  // 15% 長者折扣
    }

    return price * discount
}
```

### 3. 鏈式條件

```go
package main

import "fmt"

func validateInput(input string) error {
    if input == "" {
        return fmt.Errorf("input is empty")
    }

    if len(input) < 3 {
        return fmt.Errorf("input too short")
    }

    if len(input) > 50 {
        return fmt.Errorf("input too long")
    }

    // 所有驗證通過
    return nil
}

func main() {
    if err := validateInput("hi"); err != nil {
        fmt.Println("Validation error:", err)
    }
}
```

**對比 JavaScript:**
```javascript
function validateInput(input) {
    if (!input) {
        throw new Error("input is empty");
    }

    if (input.length < 3) {
        throw new Error("input too short");
    }

    if (input.length > 50) {
        throw new Error("input too long");
    }
}

try {
    validateInput("hi");
} catch (err) {
    console.log("Validation error:", err.message);
}
```

## 作用域

### if 塊作用域

```go
package main

import "fmt"

func main() {
    x := 1

    if true {
        x := 2  // 新變量（遮蔽外部 x）
        fmt.Println(x)  // 2
    }

    fmt.Println(x)  // 1（外部 x 未改變）

    // 修改外部變量
    if true {
        x = 3  // 賦值（不是聲明）
        fmt.Println(x)  // 3
    }

    fmt.Println(x)  // 3（外部 x 被修改）
}
```

**對比 JavaScript:**
```javascript
let x = 1;

if (true) {
    let x = 2;  // 新變量（遮蔽）
    console.log(x);  // 2
}

console.log(x);  // 1

// 修改外部變量
if (true) {
    x = 3;  // 賦值
    console.log(x);  // 3
}

console.log(x);  // 3
```

### else 作用域

```go
package main

import "fmt"

func main() {
    if x := 1; x > 0 {
        fmt.Println("Positive:", x)
        // x 可見
    } else {
        fmt.Println("Non-positive:", x)
        // x 在 else 中也可見！
    }

    // x 在這裡不可見
}
```

## switch vs if-else

有時 switch 更清晰：

```go
package main

import "fmt"

func main() {
    score := 85

    // 使用 if-else
    var grade string
    if score >= 90 {
        grade = "A"
    } else if score >= 80 {
        grade = "B"
    } else if score >= 70 {
        grade = "C"
    } else if score >= 60 {
        grade = "D"
    } else {
        grade = "F"
    }

    // 使用 switch（更清晰）
    switch {
    case score >= 90:
        grade = "A"
    case score >= 80:
        grade = "B"
    case score >= 70:
        grade = "C"
    case score >= 60:
        grade = "D"
    default:
        grade = "F"
    }

    fmt.Println("Grade:", grade)
}
```

## 重點總結

### Go if 語句特點

1. **條件不需要括號**（但可以有）
2. **大括號必須有**（即使單行）
3. **else 必須與 if 的 } 同行**
4. **可以有初始化語句**
5. **只接受布爾值**（不接受真值/假值）

### Go vs JavaScript

| 特性 | Go | JavaScript |
|------|-----|------------|
| **條件括號** | 可選（不推薦） | 必須 |
| **大括號** | 必須 | 單行可選 |
| **else 位置** | 必須同行 | 可另起一行 |
| **初始化語句** | ✅ 支持 | ❌ 不支持 |
| **真值/假值** | ❌ 不支持 | ✅ 支持 |
| **三元運算符** | ❌ 沒有 | ✅ 有 |

### 最佳實踐

1. **使用 if 初始化語句**縮小作用域
2. **提前返回**減少嵌套
3. **使用守衛子句**處理錯誤
4. **條件複雜時使用 switch**
5. **避免過深嵌套**（超過3層考慮重構）

## 練習題

### 基礎題

1. **基本條件**
   - 編寫檢查數字正負零的函數
   - 實現成績評級系統
   - 對比 Go 和 JavaScript 實現

2. **if 初始化語句**
   - 使用 if 初始化讀取 map
   - 實現文件讀取並處理錯誤
   - 理解作用域差異

3. **邏輯運算符**
   - 實現年齡驗證（18-65）
   - 組合多個條件
   - 使用括號明確優先級

### 進階題

4. **錯誤處理模式**
   - 實現多層驗證函數
   - 使用提前返回
   - 對比 try-catch 方式

5. **重構練習**
   - 將深層嵌套 if 改為提前返回
   - 使用守衛子句
   - 改善代碼可讀性

6. **作用域練習**
   - 創建變量遮蔽示例
   - 理解 if 初始化變量的生命週期
   - 對比 JavaScript 的塊作用域

### 實戰題

7. **表單驗證**
   - 實現用戶註冊驗證
   - 檢查用戶名、密碼、郵箱
   - 提供友好的錯誤消息

8. **權限檢查**
   - 實現多層權限系統
   - 組合角色和權限檢查
   - 使用守衛子句

9. **對比分析**
   - 用 Go 和 JavaScript 實現相同邏輯
   - 對比代碼可讀性
   - 記錄 if 初始化語句的優勢

## 下一章預告

下一章我們將學習 Go 的循環（for 循環），包括：
- for 循環的各種形式
- range 遍歷
- 對比 for/while/forEach
- break 和 continue
- 標籤和嵌套循環

準備好學習 Go 的循環了嗎？
