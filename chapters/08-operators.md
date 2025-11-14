# 第八章：運算符與表達式

## 章節概述

本章將全面介紹 Go 的運算符系統，包括算術、比較、邏輯和位運算符。我們會對比 JavaScript，特別關注兩者的差異，如三元運算符的缺失、++/-- 的限制等。

## 運算符概覽

### 運算符對比表

| 類別 | Go | JavaScript | 差異 |
|------|-----|------------|------|
| **算術** | `+`, `-`, `*`, `/`, `%` | 相同 | Go 除法有整數除法 |
| **遞增/遞減** | `++`, `--` | 相同 | Go 只支持後綴 |
| **比較** | `==`, `!=`, `<`, `>`, `<=`, `>=` | 相同 + `===`, `!==` | Go 沒有嚴格相等 |
| **邏輯** | `&&`, `\|\|`, `!` | 相同 | - |
| **位運算** | `&`, `\|`, `^`, `<<`, `>>`, `&^` | 相同（除 `&^`）| Go 有位清除 |
| **賦值** | `=`, `+=`, `-=`, ... | 相同 | - |
| **指針** | `*`, `&` | 無 | Go 特有 |
| **三元** | 無 | `? :` | Go 沒有三元運算符 |

## 算術運算符

### 基本算術

```go
package main

import "fmt"

func main() {
    a, b := 10, 3

    fmt.Println(a + b)  // 13
    fmt.Println(a - b)  // 7
    fmt.Println(a * b)  // 30
    fmt.Println(a / b)  // 3（整數除法！）
    fmt.Println(a % b)  // 1（取模）

    // 浮點除法
    var x, y float64 = 10, 3
    fmt.Println(x / y)  // 3.3333333333333335
}
```

**對比 JavaScript:**
```javascript
let a = 10, b = 3;

console.log(a + b);  // 13
console.log(a - b);  // 7
console.log(a * b);  // 30
console.log(a / b);  // 3.3333333333333335（總是浮點除法）
console.log(a % b);  // 1

// JavaScript 沒有整數除法
// 需要手動處理
console.log(Math.floor(a / b));  // 3
console.log(~~(a / b));           // 3（位運算技巧）
```

### 整數除法 vs 浮點除法

```go
package main

import "fmt"

func main() {
    // 整數除法
    fmt.Println(10 / 3)      // 3（截斷）
    fmt.Println(-10 / 3)     // -3（向零截斷）

    // 浮點除法
    fmt.Println(10.0 / 3.0)  // 3.3333333333333335
    fmt.Println(float64(10) / float64(3))  // 3.3333333333333335

    // 混合類型會編譯錯誤
    // fmt.Println(10 / 3.0)  // ❌ 類型不匹配
}
```

**對比 JavaScript:**
```javascript
// JavaScript 只有一種除法
console.log(10 / 3);     // 3.3333333333333335
console.log(-10 / 3);    // -3.3333333333333335

// 混合類型自動轉換
console.log(10 / 3.0);   // 3.3333333333333335（自動轉換）
```

### 字符串拼接

```go
package main

import "fmt"

func main() {
    s1 := "Hello"
    s2 := "World"

    // 使用 + 拼接
    s3 := s1 + ", " + s2 + "!"
    fmt.Println(s3)  // "Hello, World!"

    // ❌ 不能用 + 拼接字符串和數字
    // result := "Answer: " + 42  // 編譯錯誤

    // ✅ 需要轉換
    import "strconv"
    result := "Answer: " + strconv.Itoa(42)
    fmt.Println(result)  // "Answer: 42"

    // 或使用 fmt.Sprintf
    result2 := fmt.Sprintf("Answer: %d", 42)
    fmt.Println(result2)  // "Answer: 42"
}
```

**對比 JavaScript:**
```javascript
let s1 = "Hello";
let s2 = "World";

// + 拼接
let s3 = s1 + ", " + s2 + "!";
console.log(s3);  // "Hello, World!"

// ✅ 可以直接拼接字符串和數字（隱式轉換）
let result = "Answer: " + 42;
console.log(result);  // "Answer: 42"

// 模板字符串
let result2 = `Answer: ${42}`;
console.log(result2);  // "Answer: 42"
```

## 遞增和遞減運算符

### Go 的限制

```go
package main

import "fmt"

func main() {
    i := 1

    // ✅ 後綴（語句，不是表達式）
    i++
    fmt.Println(i)  // 2

    i--
    fmt.Println(i)  // 1

    // ❌ 不支持前綴
    // ++i  // 語法錯誤
    // --i  // 語法錯誤

    // ❌ 不能作為表達式
    // j := i++     // 語法錯誤
    // arr[i++]     // 語法錯誤
    // fmt.Println(i++)  // 語法錯誤

    // ✅ 必須獨立成行
    i++
    j := i
    fmt.Println(j)  // 2
}
```

**對比 JavaScript:**
```javascript
let i = 1;

// ✅ 前綴和後綴都支持
i++;
console.log(i);  // 2

++i;
console.log(i);  // 3

// ✅ 可以作為表達式
let j = i++;
console.log(j);  // 3（舊值）
console.log(i);  // 4（新值）

let k = ++i;
console.log(k);  // 5（新值）
console.log(i);  // 5

// ✅ 可以在任何地方使用
let arr = [1, 2, 3];
console.log(arr[i++]);  // arr[5]（可能 undefined）
```

## 比較運算符

### 基本比較

```go
package main

import "fmt"

func main() {
    a, b := 5, 10

    fmt.Println(a == b)   // false
    fmt.Println(a != b)   // true
    fmt.Println(a < b)    // true
    fmt.Println(a > b)    // false
    fmt.Println(a <= b)   // true
    fmt.Println(a >= b)   // false

    // 字符串比較（字典順序）
    fmt.Println("apple" < "banana")  // true
    fmt.Println("apple" == "apple")  // true
}
```

**對比 JavaScript:**
```javascript
let a = 5, b = 10;

console.log(a == b);   // false
console.log(a != b);   // true
console.log(a < b);    // true
console.log(a > b);    // false
console.log(a <= b);   // true
console.log(a >= b);   // false

// JavaScript 有嚴格相等
console.log(5 == "5");   // true（類型轉換）
console.log(5 === "5");  // false（嚴格相等）
console.log(5 !== "5");  // true

// Go 沒有這個問題（靜態類型）
// fmt.Println(5 == "5")  // 編譯錯誤！
```

### 類型必須相同

```go
package main

func main() {
    var a int = 5
    var b int64 = 5

    // ❌ 不同類型不能比較
    // fmt.Println(a == b)  // 編譯錯誤

    // ✅ 需要轉換
    fmt.Println(int64(a) == b)  // true
}
```

**對比 JavaScript:**
```javascript
// JavaScript 自動類型轉換
console.log(5 == "5");      // true
console.log(0 == false);    // true
console.log("" == false);   // true
console.log(null == undefined);  // true

// 這可能導致意外行為
console.log([] == ![]);     // true（！）
```

## 邏輯運算符

### 基本邏輯

```go
package main

import "fmt"

func main() {
    t, f := true, false

    fmt.Println(t && f)  // false（AND）
    fmt.Println(t || f)  // true（OR）
    fmt.Println(!t)      // false（NOT）

    // 短路求值
    x := 0
    if x != 0 && 10/x > 1 {  // 不會除以零
        fmt.Println("OK")
    }

    // ❌ 邏輯運算符只能用於布爾值
    // if 1 && 2 { }  // 編譯錯誤
}
```

**對比 JavaScript:**
```javascript
let t = true, f = false;

console.log(t && f);  // false
console.log(t || f);  // true
console.log(!t);      // false

// 短路求值
let x = 0;
if (x !== 0 && 10/x > 1) {
    console.log("OK");
}

// ✅ JavaScript 允許非布爾值
console.log(1 && 2);        // 2
console.log(0 || "default"); // "default"
console.log(!0);            // true

// 常用模式
let value = input || "default";  // 默認值
let result = condition && doSomething();  // 條件執行
```

### Go 的 if 初始化語句

```go
package main

import "fmt"

func main() {
    // if 可以包含初始化語句
    if x := 10; x > 5 {
        fmt.Println("x is greater than 5")
        // x 只在這個作用域內
    }
    // fmt.Println(x)  // ❌ x 不存在

    // 實用示例
    if err := doSomething(); err != nil {
        fmt.Println("Error:", err)
    }
}
```

## 位運算符

### 基本位運算

```go
package main

import "fmt"

func main() {
    a, b := 12, 10  // 1100, 1010（二進制）

    fmt.Printf("%b\n", a)      // 1100
    fmt.Printf("%b\n", b)      // 1010

    fmt.Printf("%b\n", a & b)  // 1000（AND：8）
    fmt.Printf("%b\n", a | b)  // 1110（OR：14）
    fmt.Printf("%b\n", a ^ b)  // 0110（XOR：6）
    fmt.Printf("%b\n", a &^ b) // 0100（AND NOT：4，Go 特有！）

    // 移位
    fmt.Printf("%b\n", a << 1) // 11000（左移：24）
    fmt.Printf("%b\n", a >> 1) // 110（右移：6）

    // 取反（一元運算符）
    fmt.Printf("%b\n", ^a)     // ...11110011（按位取反）
}
```

**對比 JavaScript:**
```javascript
let a = 12, b = 10;  // 1100, 1010

console.log(a.toString(2));      // "1100"
console.log(b.toString(2));      // "1010"

console.log((a & b).toString(2));  // "1000"（8）
console.log((a | b).toString(2));  // "1110"（14）
console.log((a ^ b).toString(2));  // "110"（6）
console.log((~a).toString(2));     // "-1101"（取反）

// 移位
console.log((a << 1).toString(2)); // "11000"（24）
console.log((a >> 1).toString(2)); // "110"（6）
console.log((a >>> 1).toString(2)); // "110"（無符號右移）

// ❌ JavaScript 沒有 &^（位清除）
// 需要組合實現
console.log((a & ~b).toString(2)); // "100"（4）
```

### 位運算實際應用

```go
package main

import "fmt"

const (
    Read = 1 << iota  // 1（0001）
    Write             // 2（0010）
    Execute           // 4（0100）
    Admin             // 8（1000）
)

func main() {
    // 組合權限
    permissions := Read | Write  // 3（0011）

    // 檢查權限
    canRead := permissions&Read != 0
    canWrite := permissions&Write != 0
    canExecute := permissions&Execute != 0

    fmt.Println(canRead, canWrite, canExecute)  // true true false

    // 添加權限
    permissions |= Execute  // 7（0111）

    // 移除權限
    permissions &^= Write   // 5（0101）

    // 切換權限
    permissions ^= Read     // 4（0100）
}
```

**對比 JavaScript:**
```javascript
const Permissions = {
    READ: 1 << 0,    // 1
    WRITE: 1 << 1,   // 2
    EXECUTE: 1 << 2, // 4
    ADMIN: 1 << 3    // 8
};

let permissions = Permissions.READ | Permissions.WRITE;

// 檢查
let canRead = (permissions & Permissions.READ) !== 0;
let canWrite = (permissions & Permissions.WRITE) !== 0;

console.log(canRead, canWrite);  // true true

// 添加
permissions |= Permissions.EXECUTE;

// 移除（沒有 &^，用 & ~）
permissions &= ~Permissions.WRITE;

// 切換
permissions ^= Permissions.READ;
```

## 賦值運算符

### 複合賦值

```go
package main

import "fmt"

func main() {
    x := 10

    x += 5   // x = x + 5（15）
    x -= 3   // x = x - 3（12）
    x *= 2   // x = x * 2（24）
    x /= 4   // x = x / 4（6）
    x %= 5   // x = x % 5（1）

    // 位運算複合賦值
    x = 12
    x &= 10  // x = x & 10
    x |= 5   // x = x | 5
    x ^= 3   // x = x ^ 3
    x <<= 1  // x = x << 1
    x >>= 1  // x = x >> 1

    fmt.Println(x)
}
```

**與 JavaScript 完全相同。**

## 三元運算符的替代

### Go 沒有三元運算符

```go
package main

import "fmt"

func main() {
    score := 85

    // ❌ Go 沒有三元運算符
    // result := score >= 60 ? "Pass" : "Fail"  // 語法錯誤

    // ✅ 使用 if-else
    var result string
    if score >= 60 {
        result = "Pass"
    } else {
        result = "Fail"
    }
    fmt.Println(result)

    // ✅ 或使用函數
    result = func() string {
        if score >= 60 {
            return "Pass"
        }
        return "Fail"
    }()

    // ✅ 或定義輔助函數
    result = ternary(score >= 60, "Pass", "Fail")
}

func ternary(condition bool, trueVal, falseVal string) string {
    if condition {
        return trueVal
    }
    return falseVal
}
```

**對比 JavaScript:**
```javascript
let score = 85;

// ✅ JavaScript 有三元運算符
let result = score >= 60 ? "Pass" : "Fail";
console.log(result);  // "Pass"

// 嵌套三元（不推薦）
let grade = score >= 90 ? "A" :
            score >= 80 ? "B" :
            score >= 70 ? "C" :
            score >= 60 ? "D" : "F";
```

## 運算符優先級

### Go 的優先級

```go
package main

import "fmt"

func main() {
    // 從高到低：
    // 1. *  /  %  <<  >>  &  &^
    // 2. +  -  |  ^
    // 3. ==  !=  <  <=  >  >=
    // 4. &&
    // 5. ||

    result := 2 + 3 * 4      // 14（不是 20）
    result = (2 + 3) * 4     // 20

    condition := true || false && false  // true（&& 優先於 ||）
    condition = (true || false) && false // false

    fmt.Println(result, condition)
}
```

**與 JavaScript 類似，但建議使用括號明確優先級。**

## 指針運算符

這是 Go 特有的！

```go
package main

import "fmt"

func main() {
    x := 42

    // & 取地址
    ptr := &x
    fmt.Printf("Address: %p\n", ptr)

    // * 解引用
    fmt.Println(*ptr)  // 42

    // 修改值
    *ptr = 100
    fmt.Println(x)     // 100

    // 零值指針
    var p *int
    fmt.Println(p)     // <nil>

    // 不能解引用 nil
    // fmt.Println(*p)  // panic!
}
```

**JavaScript 沒有指針（引用類型除外）。**

## 重點總結

### 主要差異

1. **整數除法**
   - Go：`10 / 3 = 3`
   - JS：`10 / 3 = 3.333...`

2. **++/--**
   - Go：只支持後綴，不是表達式
   - JS：前綴後綴都支持，是表達式

3. **三元運算符**
   - Go：沒有，使用 if-else
   - JS：有 `? :`

4. **類型轉換**
   - Go：必須顯式
   - JS：隱式轉換

5. **指針**
   - Go：有 `*` 和 `&`
   - JS：沒有（但有引用）

### 最佳實踐

1. **使用括號明確優先級**
2. **避免複雜的位運算**（除非必要）
3. **用 if-else 代替三元運算符**
4. **注意整數除法**
5. **小心指針的 nil**

## 練習題

### 基礎題

1. **算術運算**
   - 實現整數除法和浮點除法
   - 處理除以零
   - 對比 Go 和 JavaScript

2. **位運算**
   - 實現權限系統
   - 使用位掩碼
   - 練習位運算

3. **運算符優先級**
   - 不使用括號寫表達式
   - 添加括號改變結果
   - 理解優先級

### 進階題

4. **三元運算符替代**
   - 實現通用的 ternary 函數
   - 支持不同類型
   - 對比性能

5. **類型安全**
   - 找出 JavaScript 隱式轉換的陷阱
   - 用 Go 重寫
   - 理解靜態類型的好處

6. **指針運算**
   - 實現 swap 函數
   - 理解值傳遞 vs 指針傳遞
   - 對比 JavaScript 的引用

### 實戰題

7. **計算器**
   - 實現四則運算
   - 處理整數和浮點數
   - 添加錯誤處理

8. **位操作工具**
   - 實現位操作函數集
   - 包括設置、清除、切換位
   - 應用到實際場景

9. **對比分析**
   - 列出 Go 和 JavaScript 運算符的所有差異
   - 創建對照表
   - 記錄踩坑經驗

## 下一章預告

下一章我們將學習 Go 的流程控制（if/else），包括：
- if 語句的特殊語法
- if 初始化語句
- else if 和 else
- 對比 JavaScript 的條件語句
- 錯誤處理模式

準備好學習 Go 的流程控制了嗎？
