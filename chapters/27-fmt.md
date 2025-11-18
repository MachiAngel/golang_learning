# Chapter 27: fmt 包格式化

## 概述

`fmt` 包是 Go 標準庫中最常用的包之一，提供了格式化 I/O 功能，類似於 C 的 printf 和 scanf。對於 Node.js 開發者來說，它相當於 `console.log()` 和模板字符串的強化版本，提供了更精確的格式化控制。

## Node.js 與 Go 的對比

| 功能 | Node.js | Go |
|------|---------|-----|
| 基本輸出 | `console.log()` | `fmt.Println()` |
| 格式化輸出 | `console.log(...)` | `fmt.Printf()` |
| 模板字符串 | `` `Hello ${name}` `` | `fmt.Sprintf("Hello %s", name)` |
| 字符串拼接 | `str1 + str2` | `fmt.Sprintf("%s%s", str1, str2)` |
| 錯誤輸出 | `console.error()` | `fmt.Fprintf(os.Stderr, ...)` |
| 類型檢查輸出 | `console.log(typeof x)` | `fmt.Printf("%T", x)` |

## 詳細概念解釋

### 1. fmt 包的主要函數

fmt 包提供了三組主要函數：

- **Print 系列**：直接輸出到標準輸出
  - `Print()` - 輸出不換行，參數間無空格
  - `Println()` - 輸出換行，參數間有空格
  - `Printf()` - 格式化輸出

- **Sprint 系列**：返回格式化的字符串
  - `Sprint()`
  - `Sprintln()`
  - `Sprintf()`

- **Fprint 系列**：輸出到指定的 io.Writer
  - `Fprint()`
  - `Fprintln()`
  - `Fprintf()`

### 2. 常用格式化動詞（Format Verbs）

| 動詞 | 說明 | Node.js 對應 |
|------|------|--------------|
| `%v` | 默認格式 | `${value}` |
| `%+v` | 帶字段名的結構體 | `JSON.stringify()` |
| `%#v` | Go 語法表示 | `util.inspect()` |
| `%T` | 類型 | `typeof` |
| `%t` | 布爾值 | `${bool}` |
| `%d` | 十進制整數 | `${num}` |
| `%b` | 二進制 | `num.toString(2)` |
| `%o` | 八進制 | `num.toString(8)` |
| `%x` | 十六進制（小寫） | `num.toString(16)` |
| `%X` | 十六進制（大寫） | `num.toString(16).toUpperCase()` |
| `%f` | 浮點數 | `num.toFixed()` |
| `%.2f` | 保留 2 位小數 | `num.toFixed(2)` |
| `%e` | 科學計數法 | `num.toExponential()` |
| `%s` | 字符串 | `${str}` |
| `%q` | 帶引號的字符串 | `JSON.stringify(str)` |
| `%p` | 指針地址 | - |
| `%%` | 百分號字面量 | `%` |

## 實際代碼示例

### 示例 1: 基本輸出

**Go:**
```go
package main

import "fmt"

func main() {
    name := "Alice"
    age := 30

    // 基本輸出
    fmt.Print("Hello ")
    fmt.Print("World\n")

    // 帶換行輸出
    fmt.Println("Hello", "World")

    // 格式化輸出
    fmt.Printf("Name: %s, Age: %d\n", name, age)

    // 輸出：
    // Hello World
    // Hello World
    // Name: Alice, Age: 30
}
```

**Node.js:**
```javascript
const name = "Alice";
const age = 30;

// 基本輸出
process.stdout.write("Hello ");
process.stdout.write("World\n");

// 帶換行輸出
console.log("Hello", "World");

// 格式化輸出（使用模板字符串）
console.log(`Name: ${name}, Age: ${age}`);

// 輸出：
// Hello World
// Hello World
// Name: Alice, Age: 30
```

### 示例 2: 字符串格式化

**Go:**
```go
package main

import "fmt"

func main() {
    name := "Bob"
    score := 95.5

    // Sprintf - 返回格式化字符串
    message := fmt.Sprintf("Student %s scored %.1f", name, score)
    fmt.Println(message)

    // 字符串寬度控制
    fmt.Printf("|%10s|\n", "Hi")      // 右對齊，寬度 10
    fmt.Printf("|%-10s|\n", "Hi")     // 左對齊，寬度 10
    fmt.Printf("|%10.5s|\n", "Hello") // 寬度 10，最多 5 個字符

    // 輸出：
    // Student Bob scored 95.5
    // |        Hi|
    // |Hi        |
    // |     Hello|
}
```

**Node.js:**
```javascript
const name = "Bob";
const score = 95.5;

// 模板字符串
const message = `Student ${name} scored ${score.toFixed(1)}`;
console.log(message);

// 字符串寬度控制（需要手動實現）
const padStart = (str, width) => str.padStart(width);
const padEnd = (str, width) => str.padEnd(width);

console.log(`|${padStart("Hi", 10)}|`);
console.log(`|${padEnd("Hi", 10)}|`);
console.log(`|${padStart("Hello".substring(0, 5), 10)}|`);

// 輸出：
// Student Bob scored 95.5
// |        Hi|
// |Hi        |
// |     Hello|
```

### 示例 3: 數字格式化

**Go:**
```go
package main

import "fmt"

func main() {
    num := 255
    pi := 3.14159265

    // 不同進制
    fmt.Printf("十進制: %d\n", num)
    fmt.Printf("二進制: %b\n", num)
    fmt.Printf("八進制: %o\n", num)
    fmt.Printf("十六進制: %x\n", num)
    fmt.Printf("十六進制（大寫）: %X\n", num)

    // 浮點數格式化
    fmt.Printf("默認: %f\n", pi)
    fmt.Printf("保留 2 位小數: %.2f\n", pi)
    fmt.Printf("科學計數法: %e\n", pi)
    fmt.Printf("自動選擇: %g\n", pi)

    // 寬度和精度
    fmt.Printf("|%8.2f|\n", pi)  // 總寬度 8，小數點後 2 位
    fmt.Printf("|%08.2f|\n", pi) // 用 0 填充

    // 輸出：
    // 十進制: 255
    // 二進制: 11111111
    // 八進制: 377
    // 十六進制: ff
    // 十六進制（大寫）: FF
    // 默認: 3.141593
    // 保留 2 位小數: 3.14
    // 科學計數法: 3.141593e+00
    // 自動選擇: 3.14159265
    // |    3.14|
    // |00003.14|
}
```

**Node.js:**
```javascript
const num = 255;
const pi = 3.14159265;

// 不同進制
console.log(`十進制: ${num}`);
console.log(`二進制: ${num.toString(2)}`);
console.log(`八進制: ${num.toString(8)}`);
console.log(`十六進制: ${num.toString(16)}`);
console.log(`十六進制（大寫）: ${num.toString(16).toUpperCase()}`);

// 浮點數格式化
console.log(`默認: ${pi}`);
console.log(`保留 2 位小數: ${pi.toFixed(2)}`);
console.log(`科學計數法: ${pi.toExponential()}`);
console.log(`自動選擇: ${pi.toPrecision(9)}`);

// 寬度和精度（需要手動實現）
const formatted = pi.toFixed(2).padStart(8);
const paddedZero = pi.toFixed(2).padStart(8, '0');
console.log(`|${formatted}|`);
console.log(`|${paddedZero}|`);

// 輸出：
// 十進制: 255
// 二進制: 11111111
// 八進制: 377
// 十六進制: ff
// 十六進制（大寫）: FF
// 默認: 3.14159265
// 保留 2 位小數: 3.14
// 科學計數法: 3.14159265e+0
// 自動選擇: 3.14159265
// |    3.14|
// |00003.14|
```

### 示例 4: 結構體和類型輸出

**Go:**
```go
package main

import "fmt"

type Person struct {
    Name string
    Age  int
    City string
}

func main() {
    p := Person{Name: "Alice", Age: 30, City: "Tokyo"}

    // 默認格式
    fmt.Printf("%%v: %v\n", p)

    // 帶字段名
    fmt.Printf("%%+v: %+v\n", p)

    // Go 語法表示
    fmt.Printf("%%#v: %#v\n", p)

    // 類型
    fmt.Printf("%%T: %T\n", p)

    // 輸出：
    // %v: {Alice 30 Tokyo}
    // %+v: {Name:Alice Age:30 City:Tokyo}
    // %#v: main.Person{Name:"Alice", Age:30, City:"Tokyo"}
    // %T: main.Person
}
```

**Node.js:**
```javascript
class Person {
    constructor(name, age, city) {
        this.name = name;
        this.age = age;
        this.city = city;
    }
}

const p = new Person("Alice", 30, "Tokyo");

// 默認格式
console.log("默認:", p);

// JSON 格式（類似 %+v）
console.log("JSON:", JSON.stringify(p));

// util.inspect（類似 %#v）
const util = require('util');
console.log("inspect:", util.inspect(p, { showHidden: false, depth: null }));

// 類型
console.log("type:", typeof p, p.constructor.name);

// 輸出：
// 默認: Person { name: 'Alice', age: 30, city: 'Tokyo' }
// JSON: {"name":"Alice","age":30,"city":"Tokyo"}
// inspect: Person { name: 'Alice', age: 30, city: 'Tokyo' }
// type: object Person
```

### 示例 5: 錯誤輸出和文件輸出

**Go:**
```go
package main

import (
    "fmt"
    "os"
)

func main() {
    // 輸出到標準錯誤
    fmt.Fprintln(os.Stderr, "This is an error message")

    // 創建文件
    file, err := os.Create("output.txt")
    if err != nil {
        fmt.Fprintf(os.Stderr, "Error creating file: %v\n", err)
        return
    }
    defer file.Close()

    // 輸出到文件
    fmt.Fprintln(file, "Hello, File!")
    fmt.Fprintf(file, "Number: %d\n", 42)
}
```

**Node.js:**
```javascript
const fs = require('fs');

// 輸出到標準錯誤
console.error("This is an error message");

// 創建文件
try {
    const file = fs.createWriteStream('output.txt');

    // 輸出到文件
    file.write("Hello, File!\n");
    file.write(`Number: ${42}\n`);

    file.end();
} catch (err) {
    console.error(`Error creating file: ${err.message}`);
}
```

### 示例 6: 實用的格式化技巧

**Go:**
```go
package main

import "fmt"

func main() {
    // 1. 表格式輸出
    fmt.Println("ID\tName\t\tAge")
    fmt.Println("--\t----\t\t---")
    fmt.Printf("%d\t%s\t\t%d\n", 1, "Alice", 30)
    fmt.Printf("%d\t%s\t\t%d\n", 2, "Bob", 25)

    // 2. 對齊的表格
    fmt.Printf("%-5d %-10s %5d\n", 1, "Alice", 30)
    fmt.Printf("%-5d %-10s %5d\n", 2, "Bob", 25)

    // 3. 百分比輸出
    progress := 0.756
    fmt.Printf("Progress: %.1f%%\n", progress*100)

    // 4. 貨幣格式
    price := 1234.56
    fmt.Printf("Price: $%.2f\n", price)

    // 5. 調試信息
    data := map[string]int{"a": 1, "b": 2}
    fmt.Printf("Debug: %#v\n", data)
}
```

**Node.js:**
```javascript
// 1. 表格式輸出
console.log("ID\tName\t\tAge");
console.log("--\t----\t\t---");
console.log(`${1}\t${"Alice"}\t\t${30}`);
console.log(`${2}\t${"Bob"}\t\t${25}`);

// 2. 對齊的表格
console.log(`${String(1).padEnd(5)} ${"Alice".padEnd(10)} ${String(30).padStart(5)}`);
console.log(`${String(2).padEnd(5)} ${"Bob".padEnd(10)} ${String(25).padStart(5)}`);

// 3. 百分比輸出
const progress = 0.756;
console.log(`Progress: ${(progress * 100).toFixed(1)}%`);

// 4. 貨幣格式
const price = 1234.56;
console.log(`Price: $${price.toFixed(2)}`);

// 5. 調試信息
const data = {a: 1, b: 2};
console.log("Debug:", JSON.stringify(data));
```

### 示例 7: Scan 系列 - 輸入函數

**Go:**
```go
package main

import (
    "bufio"
    "fmt"
    "os"
    "strings"
)

func main() {
    // 示例 1: 簡單輸入
    var name string
    var age int

    fmt.Print("Enter your name: ")
    fmt.Scan(&name)

    fmt.Print("Enter your age: ")
    fmt.Scan(&age)

    fmt.Printf("Hello %s, you are %d years old\n", name, age)

    // 示例 2: 讀取整行（包含空格）
    reader := bufio.NewReader(os.Stdin)
    fmt.Print("Enter your full name: ")
    fullName, _ := reader.ReadString('\n')
    fullName = strings.TrimSpace(fullName)
    fmt.Printf("Your full name is: %s\n", fullName)

    // 示例 3: Scanf 格式化輸入
    var day, month, year int
    fmt.Print("Enter date (DD/MM/YYYY): ")
    fmt.Scanf("%d/%d/%d", &day, &month, &year)
    fmt.Printf("Date: %02d/%02d/%04d\n", day, month, year)
}
```

**Node.js:**
```javascript
const readline = require('readline');

const rl = readline.createInterface({
    input: process.stdin,
    output: process.stdout
});

// 示例 1: 簡單輸入
rl.question('Enter your name: ', (name) => {
    rl.question('Enter your age: ', (age) => {
        console.log(`Hello ${name}, you are ${age} years old`);

        // 示例 2: 讀取整行
        rl.question('Enter your full name: ', (fullName) => {
            console.log(`Your full name is: ${fullName.trim()}`);

            // 示例 3: 格式化輸入
            rl.question('Enter date (DD/MM/YYYY): ', (dateStr) => {
                const [day, month, year] = dateStr.split('/').map(Number);
                console.log(`Date: ${String(day).padStart(2, '0')}/${String(month).padStart(2, '0')}/${String(year).padStart(4, '0')}`);
                rl.close();
            });
        });
    });
});
```

## 重點總結

1. **fmt 包是 Go 中最常用的包**
   - `Println()` - 日常輸出（類似 console.log）
   - `Printf()` - 格式化輸出（類似模板字符串，但更強大）
   - `Sprintf()` - 格式化字符串（返回字符串而不是輸出）

2. **格式化動詞提供精確控制**
   - `%v` - 通用格式，適用於任何類型
   - `%d` - 整數
   - `%f` - 浮點數
   - `%s` - 字符串
   - `%t` - 布爾值
   - `%T` - 類型信息

3. **寬度和精度控制**
   - `%8d` - 最小寬度 8
   - `%08d` - 用 0 填充
   - `%.2f` - 保留 2 位小數
   - `%8.2f` - 寬度 8，保留 2 位小數

4. **相比 Node.js 的優勢**
   - 更精確的數字格式化
   - 內置類型檢查輸出
   - 統一的格式化語法
   - 更好的性能

5. **Node.js 開發者注意事項**
   - Go 不支持模板字符串，必須使用格式化動詞
   - 格式化參數順序很重要
   - `%v` 可以處理大多數情況，但特定類型使用特定動詞更好

## 練習題

### 練習 1: 基本格式化
編寫一個程序，輸出以下信息（使用適當的格式化）：
```
Name: Alice
Age: 30
Height: 1.68m
Is Student: false
```

**提示**: 使用 `%s`、`%d`、`%.2f`、`%t`

### 練習 2: 表格輸出
創建一個程序，輸出學生成績表：
```
ID    Name        Math  English  Average
1     Alice       95    88       91.50
2     Bob         87    92       89.50
3     Charlie     92    85       88.50
```

**提示**: 使用 `%-5d`、`%-12s`、`%5d`、`%7.2f`

### 練習 3: 數字進制轉換
編寫一個程序，接收一個十進制數字，輸出它的二進制、八進制和十六進制表示。

**提示**: 使用 `%b`、`%o`、`%x`

### 練習 4: 貨幣格式化
創建一個函數 `formatCurrency(amount float64) string`，將數字格式化為貨幣格式（例如：1234.56 -> "$1,234.56"）

**提示**: 使用 `%.2f` 和字符串處理

### 練習 5: 調試輸出
創建一個結構體 `Product`，包含字段：ID (int)、Name (string)、Price (float64)、InStock (bool)。
使用不同的格式化動詞輸出這個結構體。

**提示**: 嘗試 `%v`、`%+v`、`%#v`

### 練習 6: 用戶輸入
編寫一個程序，要求用戶輸入姓名和年齡，然後輸出問候語。

**提示**: 使用 `fmt.Scan()` 或 `bufio.Reader`

### 練習 7: 進度條
創建一個函數，顯示一個文本進度條：
```
Progress: [=========>          ] 45%
```

**提示**: 使用 `fmt.Printf()` 和字符串重複

### 練習 8: 日誌格式化
創建一個簡單的日誌函數，輸出格式如：
```
[2025-01-15 14:30:45] INFO: Server started on port 8080
[2025-01-15 14:30:46] ERROR: Connection failed
```

**提示**: 結合 `time` 包和 `fmt.Sprintf()`

## 參考答案

### 練習 1 答案
```go
package main

import "fmt"

func main() {
    name := "Alice"
    age := 30
    height := 1.68
    isStudent := false

    fmt.Printf("Name: %s\n", name)
    fmt.Printf("Age: %d\n", age)
    fmt.Printf("Height: %.2fm\n", height)
    fmt.Printf("Is Student: %t\n", isStudent)
}
```

### 練習 2 答案
```go
package main

import "fmt"

type Student struct {
    ID      int
    Name    string
    Math    int
    English int
}

func main() {
    students := []Student{
        {1, "Alice", 95, 88},
        {2, "Bob", 87, 92},
        {3, "Charlie", 92, 85},
    }

    fmt.Printf("%-5s %-12s %-5s %-8s %s\n", "ID", "Name", "Math", "English", "Average")
    for _, s := range students {
        avg := float64(s.Math+s.English) / 2
        fmt.Printf("%-5d %-12s %-5d %-8d %.2f\n", s.ID, s.Name, s.Math, s.English, avg)
    }
}
```

### 練習 3 答案
```go
package main

import "fmt"

func main() {
    var num int
    fmt.Print("Enter a decimal number: ")
    fmt.Scan(&num)

    fmt.Printf("Decimal: %d\n", num)
    fmt.Printf("Binary: %b\n", num)
    fmt.Printf("Octal: %o\n", num)
    fmt.Printf("Hexadecimal: %x\n", num)
}
```

### 練習 4 答案
```go
package main

import (
    "fmt"
    "strings"
)

func formatCurrency(amount float64) string {
    // 格式化為兩位小數
    formatted := fmt.Sprintf("%.2f", amount)

    // 分離整數和小數部分
    parts := strings.Split(formatted, ".")
    intPart := parts[0]
    decPart := parts[1]

    // 添加千位分隔符
    var result []rune
    for i, digit := range intPart {
        if i > 0 && (len(intPart)-i)%3 == 0 {
            result = append(result, ',')
        }
        result = append(result, digit)
    }

    return fmt.Sprintf("$%s.%s", string(result), decPart)
}

func main() {
    fmt.Println(formatCurrency(1234.56))    // $1,234.56
    fmt.Println(formatCurrency(1234567.89)) // $1,234,567.89
}
```
