# 第十一章：函數定義與調用

## 章節概述

本章將全面介紹 Go 的函數系統，包括函數聲明、多返回值、命名返回值、可變參數、匿名函數、閉包和 defer 語句。我們將深入對比 JavaScript 的 function 和 arrow function。

## 函數聲明

### 基本語法

**Go:**
```go
package main

import "fmt"

// func 函數名(參數列表) 返回類型 {
//     函數體
// }

func greet(name string) string {
    return "Hello, " + name
}

func main() {
    message := greet("Alice")
    fmt.Println(message)  // Hello, Alice
}
```

**對比 JavaScript:**
```javascript
// function 聲明
function greet(name) {
    return "Hello, " + name;
}

// arrow function
const greet = (name) => {
    return "Hello, " + name;
};

// 簡寫（單表達式）
const greet = name => "Hello, " + name;

let message = greet("Alice");
console.log(message);  // Hello, Alice
```

### 無參數無返回值

**Go:**
```go
package main

import "fmt"

func sayHello() {
    fmt.Println("Hello!")
}

func main() {
    sayHello()
}
```

**對比 JavaScript:**
```javascript
function sayHello() {
    console.log("Hello!");
}

// 或 arrow function
const sayHello = () => {
    console.log("Hello!");
};

sayHello();
```

### 多個參數

**Go:**
```go
package main

import "fmt"

func add(a int, b int) int {
    return a + b
}

// 相同類型可以簡寫
func multiply(a, b int) int {
    return a * b
}

// 不同類型
func describe(name string, age int, salary float64) string {
    return fmt.Sprintf("%s is %d years old, salary: %.2f", name, age, salary)
}

func main() {
    sum := add(10, 20)
    product := multiply(5, 6)
    desc := describe("Alice", 30, 50000.00)

    fmt.Println(sum, product, desc)
}
```

**對比 JavaScript:**
```javascript
function add(a, b) {
    return a + b;
}

const multiply = (a, b) => a * b;

function describe(name, age, salary) {
    return `${name} is ${age} years old, salary: ${salary.toFixed(2)}`;
}

// JavaScript 沒有類型聲明
// 參數可以是任何類型
```

## 多返回值

這是 Go 的重要特性！

### 基本多返回值

```go
package main

import "fmt"

// 返回多個值
func divide(a, b float64) (float64, error) {
    if b == 0 {
        return 0, fmt.Errorf("division by zero")
    }
    return a / b, nil
}

func main() {
    // 接收多個返回值
    result, err := divide(10, 2)
    if err != nil {
        fmt.Println("Error:", err)
        return
    }
    fmt.Println("Result:", result)  // 5

    // 忽略某個返回值
    result, _ = divide(10, 0)  // 忽略錯誤（不推薦）
}
```

**對比 JavaScript:**
```javascript
// JavaScript 沒有多返回值
// 需要返回對象或數組

function divide(a, b) {
    if (b === 0) {
        return { result: 0, error: new Error("division by zero") };
    }
    return { result: a / b, error: null };
}

let { result, error } = divide(10, 2);
if (error) {
    console.log("Error:", error);
} else {
    console.log("Result:", result);
}

// 或使用數組
function divideArray(a, b) {
    if (b === 0) {
        return [0, new Error("division by zero")];
    }
    return [a / b, null];
}

let [res, err] = divideArray(10, 2);

// 或使用 Promise/async-await
async function divideAsync(a, b) {
    if (b === 0) {
        throw new Error("division by zero");
    }
    return a / b;
}

try {
    result = await divideAsync(10, 2);
} catch (error) {
    console.log("Error:", error);
}
```

### 實際應用：錯誤處理

```go
package main

import (
    "fmt"
    "os"
)

func readFile(filename string) ([]byte, error) {
    data, err := os.ReadFile(filename)
    if err != nil {
        return nil, err
    }
    return data, nil
}

func main() {
    data, err := readFile("config.json")
    if err != nil {
        fmt.Println("Error:", err)
        return
    }
    fmt.Println("Data:", string(data))
}
```

### 命名返回值

```go
package main

import "fmt"

// 命名返回值
func divide(a, b float64) (result float64, err error) {
    if b == 0 {
        err = fmt.Errorf("division by zero")
        return  // 等價於 return result, err
    }
    result = a / b
    return  // 等價於 return result, err
}

func main() {
    result, err := divide(10, 2)
    if err != nil {
        fmt.Println("Error:", err)
        return
    }
    fmt.Println("Result:", result)
}
```

**命名返回值的優勢：**
1. 作為文檔（說明返回值的含義）
2. 自動初始化為零值
3. 可以使用裸 return

**注意：**
```go
func example() (x int, y int) {
    x = 10
    y = 20
    return  // 自動返回 x, y

    // 也可以顯式返回不同的值
    // return 100, 200  // 返回 100, 200 而不是 x, y
}
```

## 可變參數

### 基本用法

```go
package main

import "fmt"

// 可變參數（類似 JavaScript 的 ...args）
func sum(numbers ...int) int {
    total := 0
    for _, num := range numbers {
        total += num
    }
    return total
}

func main() {
    fmt.Println(sum(1, 2, 3))           // 6
    fmt.Println(sum(1, 2, 3, 4, 5))     // 15
    fmt.Println(sum())                  // 0

    // 展開切片
    nums := []int{1, 2, 3, 4, 5}
    fmt.Println(sum(nums...))           // 15
}
```

**對比 JavaScript:**
```javascript
// JavaScript rest 參數
function sum(...numbers) {
    return numbers.reduce((a, b) => a + b, 0);
}

console.log(sum(1, 2, 3));        // 6
console.log(sum(1, 2, 3, 4, 5));  // 15
console.log(sum());                // 0

// 展開數組
const nums = [1, 2, 3, 4, 5];
console.log(sum(...nums));         // 15
```

### 混合參數

```go
package main

import "fmt"

// 可變參數必須是最後一個參數
func greet(greeting string, names ...string) {
    for _, name := range names {
        fmt.Printf("%s, %s!\n", greeting, name)
    }
}

func main() {
    greet("Hello", "Alice", "Bob", "Charlie")
    // Hello, Alice!
    // Hello, Bob!
    // Hello, Charlie!
}
```

**對比 JavaScript:**
```javascript
function greet(greeting, ...names) {
    names.forEach(name => {
        console.log(`${greeting}, ${name}!`);
    });
}

greet("Hello", "Alice", "Bob", "Charlie");
```

### Printf 風格函數

```go
package main

import "fmt"

func printf(format string, args ...interface{}) {
    fmt.Printf(format, args...)
}

func main() {
    printf("Name: %s, Age: %d, Salary: %.2f\n", "Alice", 30, 50000.00)
}
```

## 匿名函數和閉包

### 匿名函數

```go
package main

import "fmt"

func main() {
    // 匿名函數
    func() {
        fmt.Println("Anonymous function")
    }()  // 立即執行

    // 賦值給變量
    greet := func(name string) string {
        return "Hello, " + name
    }

    fmt.Println(greet("Alice"))

    // 作為參數傳遞
    execute(func() {
        fmt.Println("Callback executed")
    })
}

func execute(fn func()) {
    fn()
}
```

**對比 JavaScript:**
```javascript
// 立即執行函數表達式（IIFE）
(function() {
    console.log("Anonymous function");
})();

// 賦值給變量
const greet = function(name) {
    return "Hello, " + name;
};

// 或 arrow function
const greet = (name) => "Hello, " + name;

console.log(greet("Alice"));

// 回調函數
function execute(fn) {
    fn();
}

execute(function() {
    console.log("Callback executed");
});

// 或
execute(() => console.log("Callback executed"));
```

### 閉包

```go
package main

import "fmt"

func counter() func() int {
    count := 0
    return func() int {
        count++
        return count
    }
}

func main() {
    c1 := counter()
    fmt.Println(c1())  // 1
    fmt.Println(c1())  // 2
    fmt.Println(c1())  // 3

    c2 := counter()
    fmt.Println(c2())  // 1（新的計數器）
}
```

**對比 JavaScript:**
```javascript
function counter() {
    let count = 0;
    return function() {
        count++;
        return count;
    };
}

const c1 = counter();
console.log(c1());  // 1
console.log(c1());  // 2
console.log(c1());  // 3

const c2 = counter();
console.log(c2());  // 1
```

### 閉包陷阱

```go
package main

import "fmt"

func main() {
    funcs := make([]func(), 3)

    // ❌ 錯誤：所有函數都會打印 3
    for i := 0; i < 3; i++ {
        funcs[i] = func() {
            fmt.Println(i)  // 捕獲的是 i 的引用
        }
    }

    for _, f := range funcs {
        f()  // 3 3 3
    }

    // ✅ 正確：創建新變量
    for i := 0; i < 3; i++ {
        i := i  // 創建新變量
        funcs[i] = func() {
            fmt.Println(i)
        }
    }

    for _, f := range funcs {
        f()  // 0 1 2
    }

    // ✅ 或通過參數傳遞
    for i := 0; i < 3; i++ {
        funcs[i] = func(n int) func() {
            return func() {
                fmt.Println(n)
            }
        }(i)
    }
}
```

**對比 JavaScript:**
```javascript
// JavaScript 有相同的問題
let funcs = [];

// ❌ 錯誤（var 有問題）
for (var i = 0; i < 3; i++) {
    funcs[i] = function() {
        console.log(i);
    };
}

funcs.forEach(f => f());  // 3 3 3

// ✅ 使用 let（塊作用域）
for (let i = 0; i < 3; i++) {
    funcs[i] = function() {
        console.log(i);
    };
}

funcs.forEach(f => f());  // 0 1 2

// ✅ 或使用 IIFE
for (var i = 0; i < 3; i++) {
    funcs[i] = (function(n) {
        return function() {
            console.log(n);
        };
    })(i);
}
```

## 函數作為值

### 函數類型

```go
package main

import "fmt"

// 定義函數類型
type MathOp func(int, int) int

func add(a, b int) int {
    return a + b
}

func multiply(a, b int) int {
    return a * b
}

func calculate(a, b int, op MathOp) int {
    return op(a, b)
}

func main() {
    result := calculate(10, 20, add)
    fmt.Println(result)  // 30

    result = calculate(10, 20, multiply)
    fmt.Println(result)  // 200

    // 使用匿名函數
    result = calculate(10, 20, func(a, b int) int {
        return a - b
    })
    fmt.Println(result)  // -10
}
```

**對比 JavaScript:**
```javascript
function add(a, b) {
    return a + b;
}

function multiply(a, b) {
    return a * b;
}

function calculate(a, b, op) {
    return op(a, b);
}

let result = calculate(10, 20, add);
console.log(result);  // 30

result = calculate(10, 20, multiply);
console.log(result);  // 200

result = calculate(10, 20, (a, b) => a - b);
console.log(result);  // -10
```

### 函數數組/Map

```go
package main

import "fmt"

func main() {
    // 函數數組
    operations := []func(int, int) int{
        func(a, b int) int { return a + b },
        func(a, b int) int { return a - b },
        func(a, b int) int { return a * b },
    }

    for i, op := range operations {
        fmt.Printf("Op %d: %d\n", i, op(10, 5))
    }

    // 函數 map
    ops := map[string]func(int, int) int{
        "add":      func(a, b int) int { return a + b },
        "subtract": func(a, b int) int { return a - b },
        "multiply": func(a, b int) int { return a * b },
    }

    fmt.Println(ops["add"](10, 5))       // 15
    fmt.Println(ops["multiply"](10, 5))  // 50
}
```

**對比 JavaScript:**
```javascript
const operations = [
    (a, b) => a + b,
    (a, b) => a - b,
    (a, b) => a * b
];

operations.forEach((op, i) => {
    console.log(`Op ${i}: ${op(10, 5)}`);
});

const ops = {
    add: (a, b) => a + b,
    subtract: (a, b) => a - b,
    multiply: (a, b) => a * b
};

console.log(ops.add(10, 5));       // 15
console.log(ops.multiply(10, 5));  // 50
```

## defer 語句

這是 Go 特有的強大功能！

### 基本用法

```go
package main

import "fmt"

func main() {
    defer fmt.Println("World")  // 延遲執行
    fmt.Println("Hello")

    // 輸出：
    // Hello
    // World
}
```

**defer 在函數返回前執行，用於清理資源。**

### 多個 defer

```go
package main

import "fmt"

func main() {
    defer fmt.Println("1")
    defer fmt.Println("2")
    defer fmt.Println("3")

    fmt.Println("Start")

    // 輸出（defer 以相反順序執行）：
    // Start
    // 3
    // 2
    // 1
}
```

### 實際應用：資源清理

```go
package main

import (
    "fmt"
    "os"
)

func readFile(filename string) error {
    file, err := os.Open(filename)
    if err != nil {
        return err
    }
    defer file.Close()  // 確保文件關閉

    // 讀取文件...
    // 無論如何返回，file.Close() 都會執行

    return nil
}

func main() {
    // 互斥鎖示例
    // mu.Lock()
    // defer mu.Unlock()

    // 數據庫連接示例
    // db.Begin()
    // defer db.Rollback()  // 或 Commit

    readFile("data.txt")
}
```

**對比 JavaScript:**
```javascript
// JavaScript 沒有 defer
// 使用 try-finally

const fs = require('fs').promises;

async function readFile(filename) {
    let file;
    try {
        file = await fs.open(filename);
        // 讀取文件...
    } finally {
        if (file) {
            await file.close();  // 確保關閉
        }
    }
}

// 或使用 using（新提案）
// await using file = await fs.open(filename);
// 自動清理
```

### defer 和返回值

```go
package main

import "fmt"

func example() (result int) {
    defer func() {
        result++  // 可以修改命名返回值
    }()

    return 5  // 返回 5，但 defer 會修改為 6
}

func main() {
    fmt.Println(example())  // 6
}
```

### defer 陷阱

```go
package main

import "fmt"

func main() {
    // ❌ 在循環中使用 defer（可能內存洩漏）
    for i := 0; i < 5; i++ {
        file, _ := os.Open(fmt.Sprintf("file%d.txt", i))
        defer file.Close()  // 所有文件在函數結束時才關閉！
    }

    // ✅ 使用函數包裝
    for i := 0; i < 5; i++ {
        func() {
            file, _ := os.Open(fmt.Sprintf("file%d.txt", i))
            defer file.Close()  // 每次迭代結束時關閉
        }()
    }
}
```

## 遞歸

```go
package main

import "fmt"

func factorial(n int) int {
    if n <= 1 {
        return 1
    }
    return n * factorial(n-1)
}

func fibonacci(n int) int {
    if n <= 1 {
        return n
    }
    return fibonacci(n-1) + fibonacci(n-2)
}

func main() {
    fmt.Println(factorial(5))  // 120
    fmt.Println(fibonacci(10)) // 55
}
```

**與 JavaScript 完全相同。**

## 方法（Methods）

雖然不是函數，但值得一提：

```go
package main

import "fmt"

type Rectangle struct {
    Width, Height float64
}

// 為 Rectangle 定義方法
func (r Rectangle) Area() float64 {
    return r.Width * r.Height
}

// 指針接收者（可以修改）
func (r *Rectangle) Scale(factor float64) {
    r.Width *= factor
    r.Height *= factor
}

func main() {
    rect := Rectangle{Width: 10, Height: 5}

    fmt.Println(rect.Area())  // 50

    rect.Scale(2)
    fmt.Println(rect.Area())  // 200
}
```

**對比 JavaScript:**
```javascript
class Rectangle {
    constructor(width, height) {
        this.width = width;
        this.height = height;
    }

    area() {
        return this.width * this.height;
    }

    scale(factor) {
        this.width *= factor;
        this.height *= factor;
    }
}

const rect = new Rectangle(10, 5);
console.log(rect.area());  // 50

rect.scale(2);
console.log(rect.area());  // 200
```

## 重點總結

### Go 函數特點

1. **多返回值**（核心特性）
2. **命名返回值**（自動初始化）
3. **defer**（資源清理）
4. **值傳遞**（默認）
5. **一等公民**（可以作為值）

### 函數簽名對比

| Go | JavaScript |
|----|------------|
| `func add(a, b int) int` | `function add(a, b)` |
| `func() (int, error)` | `function() { return [int, error] }` |
| `func(s ...string)` | `function(...s)` |
| `defer cleanup()` | `try-finally` |

### 最佳實踐

1. **返回錯誤而不是 panic**
2. **使用 defer 清理資源**
3. **命名返回值作為文檔**
4. **小函數，單一職責**
5. **避免過深的嵌套**

## 練習題

### 基礎題

1. **函數基礎**
   - 實現加減乘除函數
   - 使用多返回值處理除零
   - 對比 Go 和 JavaScript

2. **可變參數**
   - 實現 max/min 函數
   - 接受任意數量的整數
   - 處理空參數情況

3. **閉包練習**
   - 實現計數器
   - 實現私有變量
   - 理解閉包的作用域

### 進階題

4. **高階函數**
   - 實現 map/filter/reduce
   - 接受函數作為參數
   - 返回函數

5. **defer 應用**
   - 實現文件操作函數
   - 使用 defer 確保資源釋放
   - 處理多個資源

6. **遞歸優化**
   - 實現尾遞歸優化
   - 對比遞歸和迭代
   - 測試性能差異

### 實戰題

7. **錯誤處理鏈**
   - 實現多層函數調用
   - 正確傳遞錯誤
   - 添加上下文信息

8. **函數式編程**
   - 實現柯里化
   - 實現組合函數
   - 對比 JavaScript 實現

9. **資源管理**
   - 實現數據庫連接池
   - 使用 defer 管理連接
   - 處理錯誤和清理

## 下一章預告

下一章我們將學習 Go 的錯誤處理，包括：
- error 接口
- 錯誤創建和處理
- 對比 try-catch
- panic 和 recover
- 錯誤包裝和上下文

準備好學習 Go 的錯誤處理了嗎？
