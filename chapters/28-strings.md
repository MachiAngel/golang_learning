# Chapter 28: strings 與 strconv

## 概述

`strings` 和 `strconv` 是 Go 標準庫中處理字符串的兩個核心包。`strings` 包提供字符串操作功能（類似 JavaScript 的 String 方法），而 `strconv` 包提供字符串和基本數據類型之間的轉換功能（類似 JavaScript 的 parseInt、parseFloat 等）。

## Node.js 與 Go 的對比

### strings 包對比

| 功能 | Node.js | Go |
|------|---------|-----|
| 包含子串 | `str.includes(substr)` | `strings.Contains(str, substr)` |
| 前綴檢查 | `str.startsWith(prefix)` | `strings.HasPrefix(str, prefix)` |
| 後綴檢查 | `str.endsWith(suffix)` | `strings.HasSuffix(str, suffix)` |
| 分割字符串 | `str.split(sep)` | `strings.Split(str, sep)` |
| 連接字符串 | `arr.join(sep)` | `strings.Join(arr, sep)` |
| 替換 | `str.replace(old, new)` | `strings.Replace(str, old, new, n)` |
| 全部替換 | `str.replaceAll(old, new)` | `strings.ReplaceAll(str, old, new)` |
| 去除空格 | `str.trim()` | `strings.TrimSpace(str)` |
| 轉大寫 | `str.toUpperCase()` | `strings.ToUpper(str)` |
| 轉小寫 | `str.toLowerCase()` | `strings.ToLower(str)` |
| 重複字符串 | `str.repeat(n)` | `strings.Repeat(str, n)` |
| 查找索引 | `str.indexOf(substr)` | `strings.Index(str, substr)` |

### strconv 包對比

| 功能 | Node.js | Go |
|------|---------|-----|
| 字符串轉整數 | `parseInt(str)` | `strconv.Atoi(str)` |
| 整數轉字符串 | `String(num)` | `strconv.Itoa(num)` |
| 字符串轉浮點 | `parseFloat(str)` | `strconv.ParseFloat(str, 64)` |
| 浮點轉字符串 | `num.toString()` | `strconv.FormatFloat(num, ...)` |
| 字符串轉布爾 | `str === "true"` | `strconv.ParseBool(str)` |
| 布爾轉字符串 | `String(bool)` | `strconv.FormatBool(bool)` |

## 詳細概念解釋

### 1. strings 包核心函數

#### 1.1 搜索和檢查
- `Contains(s, substr string) bool` - 檢查是否包含子串
- `HasPrefix(s, prefix string) bool` - 檢查前綴
- `HasSuffix(s, suffix string) bool` - 檢查後綴
- `Index(s, substr string) int` - 查找子串位置（-1 表示未找到）
- `Count(s, substr string) int` - 計算子串出現次數

#### 1.2 修改和轉換
- `ToUpper(s string) string` - 轉大寫
- `ToLower(s string) string` - 轉小寫
- `Title(s string) string` - 標題化（每個單詞首字母大寫）
- `Replace(s, old, new string, n int) string` - 替換（n 次）
- `ReplaceAll(s, old, new string) string` - 全部替換

#### 1.3 分割和連接
- `Split(s, sep string) []string` - 分割字符串
- `Join(elems []string, sep string) string` - 連接字符串
- `Fields(s string) []string` - 按空白字符分割

#### 1.4 修剪
- `TrimSpace(s string) string` - 去除首尾空白
- `Trim(s, cutset string) string` - 去除首尾指定字符
- `TrimLeft(s, cutset string) string` - 去除開頭指定字符
- `TrimRight(s, cutset string) string` - 去除結尾指定字符

### 2. strconv 包核心函數

#### 2.1 整數轉換
- `Atoi(s string) (int, error)` - 字符串轉 int
- `Itoa(i int) string` - int 轉字符串
- `ParseInt(s string, base int, bitSize int) (int64, error)` - 解析整數
- `FormatInt(i int64, base int) string` - 格式化整數

#### 2.2 浮點數轉換
- `ParseFloat(s string, bitSize int) (float64, error)` - 解析浮點數
- `FormatFloat(f float64, fmt byte, prec, bitSize int) string` - 格式化浮點數

#### 2.3 布爾值轉換
- `ParseBool(s string) (bool, error)` - 解析布爾值
- `FormatBool(b bool) string` - 格式化布爾值

## 實際代碼示例

### 示例 1: 基本字符串操作

**Go:**
```go
package main

import (
    "fmt"
    "strings"
)

func main() {
    str := "Hello, Go World!"

    // 搜索和檢查
    fmt.Println(strings.Contains(str, "Go"))        // true
    fmt.Println(strings.HasPrefix(str, "Hello"))    // true
    fmt.Println(strings.HasSuffix(str, "World!"))   // true
    fmt.Println(strings.Index(str, "Go"))           // 7
    fmt.Println(strings.Count(str, "o"))            // 3

    // 大小寫轉換
    fmt.Println(strings.ToUpper(str))               // HELLO, GO WORLD!
    fmt.Println(strings.ToLower(str))               // hello, go world!
    fmt.Println(strings.Title("hello world"))       // Hello World

    // 替換
    fmt.Println(strings.Replace(str, "o", "0", 2))  // Hell0, G0 World!
    fmt.Println(strings.ReplaceAll(str, "o", "0"))  // Hell0, G0 W0rld!
}
```

**Node.js:**
```javascript
const str = "Hello, Go World!";

// 搜索和檢查
console.log(str.includes("Go"));                    // true
console.log(str.startsWith("Hello"));               // true
console.log(str.endsWith("World!"));                // true
console.log(str.indexOf("Go"));                     // 7
console.log((str.match(/o/g) || []).length);        // 3

// 大小寫轉換
console.log(str.toUpperCase());                     // HELLO, GO WORLD!
console.log(str.toLowerCase());                     // hello, go world!
console.log("hello world".replace(/\b\w/g, c => c.toUpperCase())); // Hello World

// 替換
let count = 0;
console.log(str.replace(/o/g, () => count++ < 2 ? "0" : "o")); // Hell0, G0 World!
console.log(str.replaceAll("o", "0"));              // Hell0, G0 W0rld!
```

### 示例 2: 字符串分割和連接

**Go:**
```go
package main

import (
    "fmt"
    "strings"
)

func main() {
    // Split - 分割字符串
    csv := "apple,banana,orange"
    fruits := strings.Split(csv, ",")
    fmt.Println(fruits) // [apple banana orange]

    // Join - 連接字符串
    joined := strings.Join(fruits, " | ")
    fmt.Println(joined) // apple | banana | orange

    // Fields - 按空白分割
    sentence := "Hello   World  Go"
    words := strings.Fields(sentence)
    fmt.Println(words) // [Hello World Go]

    // SplitN - 限制分割次數
    text := "a:b:c:d"
    parts := strings.SplitN(text, ":", 2)
    fmt.Println(parts) // [a b:c:d]

    // 實際應用：解析 CSV
    line := "Alice,30,Engineer"
    data := strings.Split(line, ",")
    name := data[0]
    age := data[1]
    job := data[2]
    fmt.Printf("Name: %s, Age: %s, Job: %s\n", name, age, job)
}
```

**Node.js:**
```javascript
// Split - 分割字符串
const csv = "apple,banana,orange";
const fruits = csv.split(",");
console.log(fruits); // ['apple', 'banana', 'orange']

// Join - 連接字符串
const joined = fruits.join(" | ");
console.log(joined); // apple | banana | orange

// 按空白分割
const sentence = "Hello   World  Go";
const words = sentence.split(/\s+/);
console.log(words); // ['Hello', 'World', 'Go']

// 限制分割次數
const text = "a:b:c:d";
const parts = text.split(":", 2);
console.log(parts); // ['a', 'b']

// 實際應用：解析 CSV
const line = "Alice,30,Engineer";
const data = line.split(",");
const [name, age, job] = data;
console.log(`Name: ${name}, Age: ${age}, Job: ${job}`);
```

### 示例 3: 字符串修剪

**Go:**
```go
package main

import (
    "fmt"
    "strings"
)

func main() {
    // TrimSpace - 去除首尾空白
    str1 := "  Hello World  \n\t"
    fmt.Printf("|%s|", strings.TrimSpace(str1)) // |Hello World|

    // Trim - 去除首尾指定字符
    str2 := "***Hello***"
    fmt.Println(strings.Trim(str2, "*")) // Hello

    // TrimLeft/TrimRight
    str3 := "###Title###"
    fmt.Println(strings.TrimLeft(str3, "#"))  // Title###
    fmt.Println(strings.TrimRight(str3, "#")) // ###Title

    // TrimPrefix/TrimSuffix
    url := "https://example.com"
    fmt.Println(strings.TrimPrefix(url, "https://")) // example.com

    filename := "document.pdf"
    fmt.Println(strings.TrimSuffix(filename, ".pdf")) // document

    // 實際應用：清理用戶輸入
    userInput := "  user@example.com  "
    email := strings.TrimSpace(userInput)
    fmt.Printf("Email: %s\n", email)
}
```

**Node.js:**
```javascript
// 去除首尾空白
const str1 = "  Hello World  \n\t";
console.log(`|${str1.trim()}|`); // |Hello World|

// 去除首尾指定字符（需要正則）
const str2 = "***Hello***";
console.log(str2.replace(/^\*+|\*+$/g, "")); // Hello

// 去除開頭/結尾指定字符
const str3 = "###Title###";
console.log(str3.replace(/^#+/, ""));  // Title###
console.log(str3.replace(/#+$/, ""));  // ###Title

// 去除前綴/後綴
const url = "https://example.com";
console.log(url.replace(/^https:\/\//, "")); // example.com

const filename = "document.pdf";
console.log(filename.replace(/\.pdf$/, "")); // document

// 實際應用：清理用戶輸入
const userInput = "  user@example.com  ";
const email = userInput.trim();
console.log(`Email: ${email}`);
```

### 示例 4: strconv - 字符串與數字轉換

**Go:**
```go
package main

import (
    "fmt"
    "strconv"
)

func main() {
    // 字符串轉整數
    str1 := "123"
    num1, err := strconv.Atoi(str1)
    if err != nil {
        fmt.Println("Error:", err)
    } else {
        fmt.Printf("Number: %d, Type: %T\n", num1, num1) // Number: 123, Type: int
    }

    // 整數轉字符串
    num2 := 456
    str2 := strconv.Itoa(num2)
    fmt.Printf("String: %s, Type: %T\n", str2, str2) // String: 456, Type: string

    // 字符串轉浮點數
    str3 := "3.14159"
    float1, err := strconv.ParseFloat(str3, 64)
    if err != nil {
        fmt.Println("Error:", err)
    } else {
        fmt.Printf("Float: %f\n", float1) // Float: 3.141590
    }

    // 浮點數轉字符串
    float2 := 2.71828
    str4 := strconv.FormatFloat(float2, 'f', 2, 64)
    fmt.Printf("String: %s\n", str4) // String: 2.72

    // 錯誤處理
    invalidStr := "abc"
    _, err = strconv.Atoi(invalidStr)
    if err != nil {
        fmt.Printf("Cannot convert '%s' to int: %v\n", invalidStr, err)
    }
}
```

**Node.js:**
```javascript
// 字符串轉整數
const str1 = "123";
const num1 = parseInt(str1, 10);
console.log(`Number: ${num1}, Type: ${typeof num1}`); // Number: 123, Type: number

// 整數轉字符串
const num2 = 456;
const str2 = String(num2); // 或 num2.toString()
console.log(`String: ${str2}, Type: ${typeof str2}`); // String: 456, Type: string

// 字符串轉浮點數
const str3 = "3.14159";
const float1 = parseFloat(str3);
console.log(`Float: ${float1}`); // Float: 3.14159

// 浮點數轉字符串
const float2 = 2.71828;
const str4 = float2.toFixed(2);
console.log(`String: ${str4}`); // String: 2.72

// 錯誤處理
const invalidStr = "abc";
const result = parseInt(invalidStr, 10);
if (isNaN(result)) {
    console.log(`Cannot convert '${invalidStr}' to int: NaN`);
}
```

### 示例 5: strconv - 進階轉換

**Go:**
```go
package main

import (
    "fmt"
    "strconv"
)

func main() {
    // ParseInt - 指定進制
    binary := "1010"
    num1, _ := strconv.ParseInt(binary, 2, 64)
    fmt.Printf("Binary %s = Decimal %d\n", binary, num1) // Binary 1010 = Decimal 10

    hex := "FF"
    num2, _ := strconv.ParseInt(hex, 16, 64)
    fmt.Printf("Hex %s = Decimal %d\n", hex, num2) // Hex FF = Decimal 255

    // FormatInt - 轉換為不同進制
    num3 := int64(255)
    fmt.Printf("Decimal: %d\n", num3)
    fmt.Printf("Binary: %s\n", strconv.FormatInt(num3, 2))   // 11111111
    fmt.Printf("Octal: %s\n", strconv.FormatInt(num3, 8))    // 377
    fmt.Printf("Hex: %s\n", strconv.FormatInt(num3, 16))     // ff

    // ParseBool - 字符串轉布爾
    boolStrs := []string{"true", "false", "1", "0", "T", "F"}
    for _, s := range boolStrs {
        b, err := strconv.ParseBool(s)
        if err == nil {
            fmt.Printf("%s -> %t\n", s, b)
        }
    }

    // FormatFloat - 不同格式
    pi := 3.14159265359
    fmt.Println(strconv.FormatFloat(pi, 'f', 2, 64))  // 3.14 (固定小數點)
    fmt.Println(strconv.FormatFloat(pi, 'e', 2, 64))  // 3.14e+00 (科學計數法)
    fmt.Println(strconv.FormatFloat(pi, 'g', 5, 64))  // 3.1416 (自動選擇)
}
```

**Node.js:**
```javascript
// 指定進制解析
const binary = "1010";
const num1 = parseInt(binary, 2);
console.log(`Binary ${binary} = Decimal ${num1}`); // Binary 1010 = Decimal 10

const hex = "FF";
const num2 = parseInt(hex, 16);
console.log(`Hex ${hex} = Decimal ${num2}`); // Hex FF = Decimal 255

// 轉換為不同進制
const num3 = 255;
console.log(`Decimal: ${num3}`);
console.log(`Binary: ${num3.toString(2)}`);   // 11111111
console.log(`Octal: ${num3.toString(8)}`);    // 377
console.log(`Hex: ${num3.toString(16)}`);     // ff

// 字符串轉布爾
const boolStrs = ["true", "false", "1", "0", "T", "F"];
boolStrs.forEach(s => {
    const b = s === "true" || s === "1" || s === "T";
    if (["true", "false", "1", "0", "T", "F"].includes(s)) {
        console.log(`${s} -> ${b}`);
    }
});

// 不同格式的浮點數
const pi = 3.14159265359;
console.log(pi.toFixed(2));          // 3.14
console.log(pi.toExponential(2));    // 3.14e+0
console.log(pi.toPrecision(5));      // 3.1416
```

### 示例 6: 實際應用 - URL 處理

**Go:**
```go
package main

import (
    "fmt"
    "strings"
)

func parseURL(url string) map[string]string {
    result := make(map[string]string)

    // 移除協議
    url = strings.TrimPrefix(url, "http://")
    url = strings.TrimPrefix(url, "https://")

    // 分離路徑和查詢字符串
    parts := strings.SplitN(url, "?", 2)
    result["path"] = parts[0]

    if len(parts) > 1 {
        // 解析查詢參數
        params := strings.Split(parts[1], "&")
        for _, param := range params {
            kv := strings.SplitN(param, "=", 2)
            if len(kv) == 2 {
                result[kv[0]] = kv[1]
            }
        }
    }

    return result
}

func main() {
    url := "https://example.com/search?q=golang&page=2"
    parsed := parseURL(url)

    fmt.Println("Path:", parsed["path"])
    fmt.Println("Query q:", parsed["q"])
    fmt.Println("Query page:", parsed["page"])
}
```

**Node.js:**
```javascript
function parseURL(urlStr) {
    const url = new URL(urlStr);
    const result = {
        path: url.hostname + url.pathname
    };

    // 解析查詢參數
    url.searchParams.forEach((value, key) => {
        result[key] = value;
    });

    return result;
}

const url = "https://example.com/search?q=golang&page=2";
const parsed = parseURL(url);

console.log("Path:", parsed.path);
console.log("Query q:", parsed.q);
console.log("Query page:", parsed.page);
```

### 示例 7: 實際應用 - 數據驗證

**Go:**
```go
package main

import (
    "fmt"
    "strconv"
    "strings"
)

func validateEmail(email string) bool {
    email = strings.TrimSpace(email)
    return strings.Contains(email, "@") && strings.Contains(email, ".")
}

func validateAge(ageStr string) (int, error) {
    age, err := strconv.Atoi(ageStr)
    if err != nil {
        return 0, fmt.Errorf("invalid age format")
    }
    if age < 0 || age > 150 {
        return 0, fmt.Errorf("age out of range")
    }
    return age, nil
}

func validatePhone(phone string) string {
    // 移除所有非數字字符
    cleaned := strings.Map(func(r rune) rune {
        if r >= '0' && r <= '9' {
            return r
        }
        return -1
    }, phone)
    return cleaned
}

func main() {
    // 驗證郵箱
    email := "  user@example.com  "
    if validateEmail(email) {
        fmt.Printf("Valid email: %s\n", strings.TrimSpace(email))
    }

    // 驗證年齡
    ageStr := "25"
    if age, err := validateAge(ageStr); err == nil {
        fmt.Printf("Valid age: %d\n", age)
    } else {
        fmt.Printf("Error: %v\n", err)
    }

    // 驗證電話
    phone := "+1 (555) 123-4567"
    cleaned := validatePhone(phone)
    fmt.Printf("Cleaned phone: %s\n", cleaned) // 15551234567
}
```

**Node.js:**
```javascript
function validateEmail(email) {
    email = email.trim();
    return email.includes("@") && email.includes(".");
}

function validateAge(ageStr) {
    const age = parseInt(ageStr, 10);
    if (isNaN(age)) {
        throw new Error("invalid age format");
    }
    if (age < 0 || age > 150) {
        throw new Error("age out of range");
    }
    return age;
}

function validatePhone(phone) {
    // 移除所有非數字字符
    return phone.replace(/\D/g, "");
}

// 驗證郵箱
const email = "  user@example.com  ";
if (validateEmail(email)) {
    console.log(`Valid email: ${email.trim()}`);
}

// 驗證年齡
const ageStr = "25";
try {
    const age = validateAge(ageStr);
    console.log(`Valid age: ${age}`);
} catch (err) {
    console.log(`Error: ${err.message}`);
}

// 驗證電話
const phone = "+1 (555) 123-4567";
const cleaned = validatePhone(phone);
console.log(`Cleaned phone: ${cleaned}`); // 15551234567
```

### 示例 8: strings.Builder - 高效字符串構建

**Go:**
```go
package main

import (
    "fmt"
    "strings"
)

func main() {
    // 方法 1: 使用 + 操作符（低效，不推薦用於循環）
    str1 := ""
    for i := 0; i < 5; i++ {
        str1 += fmt.Sprintf("Line %d\n", i)
    }
    fmt.Println("Method 1:\n" + str1)

    // 方法 2: 使用 strings.Builder（高效，推薦）
    var builder strings.Builder
    for i := 0; i < 5; i++ {
        builder.WriteString(fmt.Sprintf("Line %d\n", i))
    }
    str2 := builder.String()
    fmt.Println("Method 2:\n" + str2)

    // Builder 的其他方法
    var b strings.Builder
    b.WriteString("Hello")
    b.WriteRune(' ')
    b.WriteByte('G')
    b.WriteString("o")
    fmt.Println(b.String()) // Hello Go
}
```

**Node.js:**
```javascript
// 方法 1: 使用 + 操作符
let str1 = "";
for (let i = 0; i < 5; i++) {
    str1 += `Line ${i}\n`;
}
console.log("Method 1:\n" + str1);

// 方法 2: 使用數組（更高效）
const arr = [];
for (let i = 0; i < 5; i++) {
    arr.push(`Line ${i}`);
}
const str2 = arr.join("\n") + "\n";
console.log("Method 2:\n" + str2);

// 方法 3: 使用模板字符串
const lines = Array.from({length: 5}, (_, i) => `Line ${i}`);
const str3 = lines.join("\n") + "\n";
console.log("Method 3:\n" + str3);
```

## 重點總結

### strings 包
1. **不可變性**: Go 的字符串是不可變的，所有操作都返回新字符串
2. **常用函數**:
   - 搜索: `Contains`, `HasPrefix`, `HasSuffix`, `Index`
   - 修改: `Replace`, `ToUpper`, `ToLower`
   - 分割/連接: `Split`, `Join`, `Fields`
   - 修剪: `TrimSpace`, `Trim`, `TrimPrefix`, `TrimSuffix`
3. **Builder**: 需要頻繁拼接字符串時使用 `strings.Builder`

### strconv 包
1. **基本轉換**:
   - `Atoi` / `Itoa` - 字符串和整數互轉（最常用）
   - `ParseFloat` / `FormatFloat` - 字符串和浮點數互轉
   - `ParseBool` / `FormatBool` - 字符串和布爾值互轉

2. **進階轉換**:
   - `ParseInt` - 支持不同進制和位大小
   - `FormatInt` - 格式化為不同進制

3. **錯誤處理**: 轉換函數返回 `(value, error)`，必須檢查錯誤

### 與 Node.js 的主要差異
1. **函數式 vs 方法**: Go 使用包級函數（如 `strings.ToUpper(s)`），Node.js 使用方法（如 `s.toUpperCase()`）
2. **錯誤處理**: Go 顯式返回錯誤，Node.js 通常返回特殊值（如 NaN）
3. **不可變性**: 兩者的字符串都是不可變的
4. **性能**: Go 的 `strings.Builder` 類似於 Node.js 的數組 join 方法

## 練習題

### 練習 1: 字符串統計
編寫函數統計字符串中單詞數量、字符數量和行數。

### 練習 2: 字符串反轉
編寫函數反轉字符串（支持 Unicode）。

### 練習 3: 駝峰轉換
實現以下轉換函數：
- `toSnakeCase("HelloWorld")` -> `"hello_world"`
- `toCamelCase("hello_world")` -> `"helloWorld"`
- `toPascalCase("hello_world")` -> `"HelloWorld"`

### 練習 4: CSV 解析
編寫簡單的 CSV 解析器，將 CSV 字符串轉換為二維數組。

### 練習 5: 數字格式化
實現函數將數字格式化為千位分隔格式：
- `formatNumber(1234567)` -> `"1,234,567"`

### 練習 6: 密碼驗證
編寫密碼驗證函數，要求：
- 至少 8 個字符
- 至少一個大寫字母
- 至少一個小寫字母
- 至少一個數字

### 練習 7: URL Slug 生成
將標題轉換為 URL slug：
- `"Hello World!"` -> `"hello-world"`
- `"Go 語言教程"` -> `"go-tutorial"`

### 練習 8: 模板替換
實現簡單的模板引擎：
```go
template := "Hello {{name}}, you are {{age}} years old"
data := map[string]string{"name": "Alice", "age": "30"}
// 結果: "Hello Alice, you are 30 years old"
```

## 參考答案

### 練習 1 答案
```go
package main

import (
    "fmt"
    "strings"
)

type TextStats struct {
    Words      int
    Characters int
    Lines      int
}

func analyzeText(text string) TextStats {
    stats := TextStats{
        Lines:      strings.Count(text, "\n") + 1,
        Characters: len(text),
    }

    words := strings.Fields(text)
    stats.Words = len(words)

    return stats
}

func main() {
    text := "Hello World\nThis is Go\nString processing"
    stats := analyzeText(text)

    fmt.Printf("Words: %d\n", stats.Words)
    fmt.Printf("Characters: %d\n", stats.Characters)
    fmt.Printf("Lines: %d\n", stats.Lines)
}
```

### 練習 3 答案
```go
package main

import (
    "fmt"
    "strings"
    "unicode"
)

func toSnakeCase(s string) string {
    var result strings.Builder
    for i, r := range s {
        if unicode.IsUpper(r) {
            if i > 0 {
                result.WriteRune('_')
            }
            result.WriteRune(unicode.ToLower(r))
        } else {
            result.WriteRune(r)
        }
    }
    return result.String()
}

func toCamelCase(s string) string {
    words := strings.Split(s, "_")
    var result strings.Builder
    for i, word := range words {
        if i == 0 {
            result.WriteString(strings.ToLower(word))
        } else {
            result.WriteString(strings.Title(word))
        }
    }
    return result.String()
}

func toPascalCase(s string) string {
    words := strings.Split(s, "_")
    var result strings.Builder
    for _, word := range words {
        result.WriteString(strings.Title(word))
    }
    return result.String()
}

func main() {
    fmt.Println(toSnakeCase("HelloWorld"))      // hello_world
    fmt.Println(toCamelCase("hello_world"))     // helloWorld
    fmt.Println(toPascalCase("hello_world"))    // HelloWorld
}
```
