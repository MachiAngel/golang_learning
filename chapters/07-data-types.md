# 第七章：數據類型系統

## 章節概述

本章將全面介紹 Go 的數據類型系統，包括基本類型、複合類型和自定義類型。我們將深入對比 JavaScript 的類型系統，幫助你理解靜態類型語言的優勢。

## 類型系統對比

### JavaScript 的類型系統

```javascript
// JavaScript - 動態類型，7種基本類型 + Object
let num = 42;              // Number
let str = "hello";         // String
let bool = true;           // Boolean
let nothing = null;        // Null
let undef = undefined;     // Undefined
let sym = Symbol("id");    // Symbol
let big = 9007199254740991n; // BigInt

let obj = {};              // Object
let arr = [];              // Object (Array)
let func = () => {};       // Object (Function)

// 類型可以改變
num = "now a string";      // ✅ 允許
```

### Go 的類型系統

```go
// Go - 靜態類型，類型不可改變
var num int = 42
var str string = "hello"
var boolean bool = true

// num = "now a string"  // ❌ 編譯錯誤！

// Go 有更多細分的數字類型
var i8 int8 = 127          // -128 到 127
var i16 int16 = 32767
var i32 int32 = 2147483647
var i64 int64 = 9223372036854775807

var u8 uint8 = 255         // 0 到 255
var u16 uint16 = 65535
var u32 uint32 = 4294967295
var u64 uint64 = 18446744073709551615

var f32 float32 = 3.14
var f64 float64 = 3.141592653589793
```

## 基本類型

### 1. 整數類型

```go
package main

import "fmt"

func main() {
    // 有符號整數
    var i8 int8 = 127      // 8位：-128 到 127
    var i16 int16 = 32767  // 16位
    var i32 int32          // 32位（也叫 rune）
    var i64 int64          // 64位

    // 無符號整數
    var u8 uint8 = 255     // 8位：0 到 255（也叫 byte）
    var u16 uint16
    var u32 uint32
    var u64 uint64

    // 平台相關
    var i int              // 32 或 64 位（取決於平台）
    var u uint             // 32 或 64 位

    // 指針大小整數
    var uintptr uintptr    // 存儲指針的整數

    fmt.Printf("int8: %d, int16: %d\n", i8, i16)
    fmt.Printf("uint8: %d, uint16: %d\n", u8, u16)
}
```

**對比 JavaScript:**
```javascript
// JavaScript 只有一個 Number 類型
// 內部使用 64 位浮點數（IEEE 754）
let num = 42;              // 可以表示整數
let big = 9007199254740992; // 超過安全整數範圍

// 安全整數範圍
console.log(Number.MAX_SAFE_INTEGER);  // 9007199254740991
console.log(Number.MIN_SAFE_INTEGER);  // -9007199254740991

// BigInt（新增）
let bigInt = 9007199254740991n;
```

#### 整數溢出

```go
package main

import "fmt"

func main() {
    var i8 int8 = 127
    i8 = i8 + 1
    fmt.Println(i8)  // -128（溢出！）

    var u8 uint8 = 255
    u8 = u8 + 1
    fmt.Println(u8)  // 0（溢出！）
}
```

**JavaScript 對比:**
```javascript
// JavaScript Number 不會溢出，但會失去精度
let num = Number.MAX_SAFE_INTEGER;  // 9007199254740991
num = num + 1;  // 9007199254740992（正確）
num = num + 1;  // 9007199254740992（精度丟失！）
```

### 2. 浮點類型

```go
package main

import "fmt"

func main() {
    var f32 float32 = 3.14
    var f64 float64 = 3.141592653589793

    // 科學計數法
    var large float64 = 1.23e9   // 1,230,000,000
    var small float64 = 1.23e-9  // 0.00000000123

    fmt.Printf("float32: %f\n", f32)
    fmt.Printf("float64: %.15f\n", f64)
    fmt.Printf("large: %e, small: %e\n", large, small)

    // 精度問題（與 JavaScript 類似）
    fmt.Println(0.1 + 0.2)  // 0.30000000000000004
}
```

**對比 JavaScript:**
```javascript
let num = 3.14;            // 都是 Number（64位浮點）
let large = 1.23e9;
let small = 1.23e-9;

// 同樣的精度問題
console.log(0.1 + 0.2);    // 0.30000000000000004
```

### 3. 布爾類型

```go
package main

import "fmt"

func main() {
    var isActive bool = true
    var isDeleted bool = false
    var defaultBool bool         // 零值為 false

    fmt.Println(isActive, isDeleted, defaultBool)

    // 邏輯運算
    result := true && false      // false
    result = true || false       // true
    result = !true               // false

    // ❌ 不能用數字或字符串作為布爾值
    // if 1 { }                  // 編譯錯誤
    // if "hello" { }            // 編譯錯誤

    // ✅ 必須是布爾表達式
    if len("hello") > 0 {
        fmt.Println("Not empty")
    }
}
```

**對比 JavaScript:**
```javascript
let isActive = true;
let isDeleted = false;

// JavaScript 有"真值"和"假值"
if (1) {}           // ✅ 1 是真值
if ("hello") {}     // ✅ 非空字符串是真值
if ([]) {}          // ✅ 空數組是真值
if ({}) {}          // ✅ 空對象是真值

// 假值：false, 0, "", null, undefined, NaN
if (0) {}           // ❌ 不執行
if ("") {}          // ❌ 不執行
if (null) {}        // ❌ 不執行
```

### 4. 字符串類型

```go
package main

import (
    "fmt"
    "strings"
)

func main() {
    // 字符串聲明
    var s1 string = "Hello, World!"
    s2 := "你好，世界！"

    // 多行字符串（使用反引號）
    s3 := `This is a
    multi-line
    string`

    // 字符串是不可變的
    // s1[0] = 'h'  // ❌ 編譯錯誤

    // 字符串長度（字節數，不是字符數！）
    fmt.Println(len(s1))   // 13
    fmt.Println(len(s2))   // 18（中文字符占多個字節）

    // 字符串拼接
    s4 := "Hello, " + "Go!"
    s5 := fmt.Sprintf("%s %s", "Hello", "Go")

    // 字符串操作
    fmt.Println(strings.ToUpper(s1))
    fmt.Println(strings.ToLower(s1))
    fmt.Println(strings.Contains(s1, "World"))
    fmt.Println(strings.Split("a,b,c", ","))
    fmt.Println(strings.Join([]string{"a", "b", "c"}, ","))
}
```

**對比 JavaScript:**
```javascript
// JavaScript 字符串
let s1 = "Hello, World!";
let s2 = '你好，世界！';

// 模板字符串
let s3 = `This is a
multi-line
string`;

let name = "Alice";
let greeting = `Hello, ${name}!`;  // 字符串插值

// 字符串也是不可變的
s1[0] = 'h';  // ❌ 無效（但不報錯）

// 長度（字符數）
console.log(s1.length);   // 13
console.log(s2.length);   // 6（正確計數字符）

// 字符串操作
console.log(s1.toUpperCase());
console.log(s1.toLowerCase());
console.log(s1.includes("World"));
console.log("a,b,c".split(","));
console.log(["a", "b", "c"].join(","));
```

#### Rune 和字符

```go
package main

import "fmt"

func main() {
    s := "Hello, 世界"

    // 遍歷字節
    for i := 0; i < len(s); i++ {
        fmt.Printf("%c ", s[i])
    }
    // 輸出：H e l l o ,   ä ¸ – ç • Œ（亂碼）

    // 遍歷 rune（字符）
    for _, r := range s {
        fmt.Printf("%c ", r)
    }
    // 輸出：H e l l o ,   世 界

    // rune 是 int32 的別名
    var r rune = '世'
    fmt.Printf("%c %d %U\n", r, r, r)
    // 輸出：世 19990 U+4E16
}
```

**對比 JavaScript:**
```javascript
let s = "Hello, 世界";

// JavaScript 正確處理 Unicode
for (let char of s) {
    console.log(char);  // H e l l o ,   世 界
}

// 字符編碼
console.log('世'.charCodeAt(0));  // 19990
```

## 複合類型

### 1. 數組（Array）

```go
package main

import "fmt"

func main() {
    // 固定長度數組
    var arr1 [3]int              // [0 0 0]
    arr2 := [3]int{1, 2, 3}
    arr3 := [...]int{1, 2, 3, 4} // 自動推導長度

    // 數組長度是類型的一部分
    // var arr4 [4]int = arr2    // ❌ 類型不匹配

    // 訪問元素
    fmt.Println(arr2[0])         // 1
    arr2[0] = 10
    fmt.Println(arr2)            // [10 2 3]

    // 數組長度
    fmt.Println(len(arr2))       // 3

    // 遍歷
    for i, v := range arr2 {
        fmt.Printf("index: %d, value: %d\n", i, v)
    }

    // 二維數組
    var matrix [2][3]int = [2][3]int{
        {1, 2, 3},
        {4, 5, 6},
    }
    fmt.Println(matrix)
}
```

**對比 JavaScript:**
```javascript
// JavaScript 數組（動態長度）
let arr1 = [];                 // 空數組
let arr2 = [1, 2, 3];
let arr3 = new Array(5);       // 長度為5的空數組

// 數組是動態的
arr2.push(4);                  // [1, 2, 3, 4]
arr2.pop();                    // [1, 2, 3]

// 訪問元素
console.log(arr2[0]);          // 1
arr2[0] = 10;
console.log(arr2);             // [10, 2, 3]

// 長度
console.log(arr2.length);      // 3

// 遍歷
arr2.forEach((v, i) => {
    console.log(`index: ${i}, value: ${v}`);
});

// 多維數組
let matrix = [
    [1, 2, 3],
    [4, 5, 6]
];
```

### 2. 切片（Slice）

這是 Go 中最常用的序列類型！

```go
package main

import "fmt"

func main() {
    // 切片聲明（類似 JavaScript 數組）
    var slice1 []int             // nil slice
    slice2 := []int{1, 2, 3}
    slice3 := make([]int, 3)     // [0 0 0]，長度為3
    slice4 := make([]int, 3, 5)  // 長度3，容量5

    // 添加元素
    slice2 = append(slice2, 4)   // [1 2 3 4]
    slice2 = append(slice2, 5, 6) // [1 2 3 4 5 6]

    // 切片操作
    sub := slice2[1:4]           // [2 3 4]（不包括索引4）
    fmt.Println(sub)

    // 長度和容量
    fmt.Println(len(slice2))     // 6
    fmt.Println(cap(slice2))     // 容量（底層數組大小）

    // 從數組創建切片
    arr := [5]int{1, 2, 3, 4, 5}
    slice5 := arr[1:4]           // [2 3 4]

    // 複製切片
    slice6 := make([]int, len(slice2))
    copy(slice6, slice2)
}
```

**對比 JavaScript:**
```javascript
// JavaScript 數組就像 Go 的 slice
let slice1 = [];
let slice2 = [1, 2, 3];
let slice3 = new Array(3).fill(0);  // [0, 0, 0]

// 添加元素
slice2.push(4);                // [1, 2, 3, 4]
slice2.push(5, 6);             // [1, 2, 3, 4, 5, 6]

// 切片操作
let sub = slice2.slice(1, 4);  // [2, 3, 4]

// 長度
console.log(slice2.length);    // 6

// 複製
let slice6 = [...slice2];      // 或 slice2.slice()
```

### 3. Map（映射）

```go
package main

import "fmt"

func main() {
    // Map 聲明
    var map1 map[string]int               // nil map（不能使用）
    map2 := make(map[string]int)          // 空 map
    map3 := map[string]int{
        "Alice": 30,
        "Bob":   25,
    }

    // 添加/修改
    map3["Charlie"] = 35
    map3["Alice"] = 31

    // 訪問
    age := map3["Alice"]                  // 31
    fmt.Println(age)

    // 檢查鍵是否存在
    age, exists := map3["David"]
    if exists {
        fmt.Println("David:", age)
    } else {
        fmt.Println("David not found")
    }

    // 刪除
    delete(map3, "Bob")

    // 遍歷
    for key, value := range map3 {
        fmt.Printf("%s: %d\n", key, value)
    }

    // 長度
    fmt.Println(len(map3))
}
```

**對比 JavaScript:**
```javascript
// JavaScript Object（類似 map）
let map1 = {};
let map3 = {
    "Alice": 30,
    "Bob": 25
};

// 添加/修改
map3["Charlie"] = 35;
map3.Alice = 31;  // 也可以用點號

// 訪問
let age = map3["Alice"];  // 31

// 檢查鍵
if ("David" in map3) {
    console.log("David:", map3.David);
} else {
    console.log("David not found");
}

// 刪除
delete map3.Bob;

// 遍歷
for (let key in map3) {
    console.log(`${key}: ${map3[key]}`);
}

// ES6 Map（更接近 Go 的 map）
let map4 = new Map();
map4.set("Alice", 30);
map4.set("Bob", 25);

console.log(map4.get("Alice"));  // 30
console.log(map4.has("David"));  // false
map4.delete("Bob");
map4.forEach((value, key) => {
    console.log(`${key}: ${value}`);
});
```

### 4. 結構體（Struct）

```go
package main

import "fmt"

// 定義結構體
type Person struct {
    Name string
    Age  int
    City string
}

func main() {
    // 創建結構體
    var p1 Person                     // 零值：{"" 0 ""}
    p2 := Person{"Alice", 30, "NYC"}
    p3 := Person{
        Name: "Bob",
        Age:  25,
        City: "LA",
    }

    // 訪問字段
    fmt.Println(p2.Name)
    p2.Age = 31

    // 匿名字段
    type Employee struct {
        Person                        // 嵌入 Person
        Department string
    }

    e := Employee{
        Person:     Person{"Charlie", 28, "SF"},
        Department: "Engineering",
    }

    // 可以直接訪問嵌入結構體的字段
    fmt.Println(e.Name)  // "Charlie"
    fmt.Println(e.Department)

    // 指針
    p4 := &Person{"David", 35, "Seattle"}
    fmt.Println(p4.Name)  // 自動解引用
}
```

**對比 JavaScript:**
```javascript
// JavaScript 對象（類似但更靈活）
let p1 = {};  // 空對象
let p2 = {
    name: "Alice",
    age: 30,
    city: "NYC"
};

// 訪問字段
console.log(p2.name);
p2.age = 31;

// 動態添加字段（Go 不允許）
p2.email = "alice@example.com";  // ✅ JavaScript 允許

// ES6 Class
class Person {
    constructor(name, age, city) {
        this.name = name;
        this.age = age;
        this.city = city;
    }
}

let p3 = new Person("Bob", 25, "LA");

// 繼承
class Employee extends Person {
    constructor(name, age, city, department) {
        super(name, age, city);
        this.department = department;
    }
}

let e = new Employee("Charlie", 28, "SF", "Engineering");
console.log(e.name);        // "Charlie"
console.log(e.department);  // "Engineering"
```

## 類型別名和自定義類型

### 類型別名

```go
package main

import "fmt"

// 類型別名（完全等價）
type MyInt = int

func main() {
    var a int = 10
    var b MyInt = 20

    // 可以直接相加（類型相同）
    c := a + b
    fmt.Println(c)  // 30
}
```

### 自定義類型

```go
package main

import "fmt"

// 自定義類型（新類型）
type Celsius float64
type Fahrenheit float64

func main() {
    var c Celsius = 100
    var f Fahrenheit = 212

    // ❌ 不能直接運算（不同類型）
    // temp := c + f  // 編譯錯誤

    // ✅ 需要轉換
    temp := c + Celsius(f)
    fmt.Println(temp)
}

// 為自定義類型添加方法
func (c Celsius) ToFahrenheit() Fahrenheit {
    return Fahrenheit(c*9/5 + 32)
}

func (f Fahrenheit) ToCelsius() Celsius {
    return Celsius((f - 32) * 5 / 9)
}
```

**對比 JavaScript/TypeScript:**
```javascript
// JavaScript 沒有類型別名
// TypeScript 有
type MyNumber = number;
type Celsius = number;
type Fahrenheit = number;

let c: Celsius = 100;
let f: Fahrenheit = 212;

// TypeScript 的類型別名只在編譯時存在
// 運行時都是 number，可以相加
let temp = c + f;  // ✅ TypeScript 允許（運行時都是 number）
```

## 類型轉換和斷言

### 類型轉換

```go
package main

import "fmt"

func main() {
    var i int = 42
    var f float64 = float64(i)    // 顯式轉換
    var u uint = uint(f)

    fmt.Println(i, f, u)

    // 字符串轉換
    import "strconv"

    // int to string
    s := strconv.Itoa(42)         // "42"

    // string to int
    n, err := strconv.Atoi("42")
    if err != nil {
        fmt.Println("Error:", err)
    }
    fmt.Println(n)  // 42
}
```

### 接口類型斷言

```go
package main

import "fmt"

func main() {
    var i interface{} = "hello"

    // 類型斷言
    s, ok := i.(string)
    if ok {
        fmt.Println("String:", s)
    }

    // 如果斷言失敗且不檢查 ok，會 panic
    // n := i.(int)  // panic!

    // 類型 switch
    switch v := i.(type) {
    case string:
        fmt.Println("String:", v)
    case int:
        fmt.Println("Int:", v)
    default:
        fmt.Println("Unknown type")
    }
}
```

**對比 JavaScript:**
```javascript
let i = "hello";

// JavaScript 運行時類型檢查
if (typeof i === "string") {
    console.log("String:", i);
}

// instanceof 檢查
if (i instanceof String) {
    // ...
}

// TypeScript 類型守衛
function processValue(value: string | number) {
    if (typeof value === "string") {
        console.log(value.toUpperCase());
    } else {
        console.log(value.toFixed(2));
    }
}
```

## 重點總結

### 基本類型

| Go | JavaScript | 說明 |
|----|------------|------|
| `int`, `int8`, `int16`, `int32`, `int64` | `Number` | Go 有多種整數類型 |
| `uint`, `uint8`, ... | `Number` | Go 有無符號整數 |
| `float32`, `float64` | `Number` | Go 區分精度 |
| `bool` | `Boolean` | Go 不允許真值/假值 |
| `string` | `String` | 類似，但 Go 是 UTF-8 |
| `rune` | - | Unicode 字符（int32）|
| `byte` | - | uint8 別名 |

### 複合類型

| Go | JavaScript | 說明 |
|----|------------|------|
| `[n]T` 數組 | `Array` | Go 數組長度固定 |
| `[]T` 切片 | `Array` | Go 切片類似 JS 數組 |
| `map[K]V` | `Object`/`Map` | 鍵值對 |
| `struct` | `Object`/`Class` | 結構化數據 |

### 關鍵差異

1. **靜態 vs 動態**
   - Go：編譯時確定類型
   - JS：運行時確定類型

2. **類型轉換**
   - Go：必須顯式轉換
   - JS：隱式轉換（危險）

3. **零值**
   - Go：每種類型有零值
   - JS：undefined

4. **不可變性**
   - Go：字符串、數組值傳遞
   - JS：字符串不可變，數組可變

## 練習題

### 基礎題

1. **類型探索**
   - 創建所有基本類型的變量
   - 打印零值
   - 測試類型轉換

2. **數組和切片**
   - 創建數組和切片
   - 對比行為差異
   - 練習切片操作

3. **Map 操作**
   - 創建並操作 map
   - 處理不存在的鍵
   - 遍歷 map

### 進階題

4. **結構體設計**
   - 設計一個 User 結構體
   - 包含嵌套結構體
   - 實現方法

5. **類型轉換**
   - 實現溫度轉換（Celsius/Fahrenheit）
   - 處理字符串和數字轉換
   - 錯誤處理

6. **對比實現**
   - 用 Go 和 JavaScript 實現相同的數據結構
   - 對比類型安全性
   - 記錄差異

### 實戰題

7. **數據驗證**
   - 創建一個驗證系統
   - 使用類型斷言
   - 處理多種數據類型

8. **JSON 處理**
   - 結構體和 JSON 互轉
   - 處理嵌套結構
   - 對比 JavaScript 的 JSON 處理

9. **類型系統優勢**
   - 找出 JavaScript 中容易出錯但 Go 能在編譯時發現的案例
   - 實現類型安全的配置系統
   - 記錄靜態類型的好處

## 下一章預告

下一章我們將學習 Go 的運算符與表達式，包括：
- 算術、比較、邏輯運算符
- 位運算符
- 三元運算符的替代方案
- 運算符優先級
- 對比 JavaScript 的運算符

準備好學習 Go 的運算符了嗎？
