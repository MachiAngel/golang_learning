# 第十三章：指針基礎

## 章節概述

本章將從 Node.js 工程師的視角介紹 Go 的指針。我們會對比 JavaScript 的引用類型，幫助你理解指針的概念、用法和常見陷阱。

## 什麼是指針？

### JavaScript 的引用類型

```javascript
// JavaScript 中的基本類型（值傳遞）
let a = 10;
let b = a;
b = 20;
console.log(a);  // 10（a 沒有改變）

// JavaScript 中的引用類型（對象、數組）
let obj1 = { value: 10 };
let obj2 = obj1;  // obj2 指向同一個對象
obj2.value = 20;
console.log(obj1.value);  // 20（obj1 也改變了）

// 數組也是引用類型
let arr1 = [1, 2, 3];
let arr2 = arr1;
arr2.push(4);
console.log(arr1);  // [1, 2, 3, 4]
```

### Go 的指針

```go
package main

import "fmt"

func main() {
    // 值類型（默認）
    a := 10
    b := a
    b = 20
    fmt.Println(a)  // 10（a 沒有改變）

    // 指針（顯式引用）
    x := 10
    ptr := &x       // & 取地址
    fmt.Println(ptr)   // 0xc000018030（內存地址）
    fmt.Println(*ptr)  // 10（* 解引用，獲取值）

    *ptr = 20       // 通過指針修改值
    fmt.Println(x)  // 20（x 改變了）
}
```

**關鍵區別：**
- JavaScript：對象/數組自動是引用
- Go：需要顯式使用指針（`&` 和 `*`）

## 指針運算符

### & 取地址運算符

```go
package main

import "fmt"

func main() {
    x := 42
    ptr := &x  // 獲取 x 的內存地址

    fmt.Printf("x 的值: %d\n", x)           // 42
    fmt.Printf("x 的地址: %p\n", &x)        // 0xc000018030
    fmt.Printf("ptr 的值: %p\n", ptr)       // 0xc000018030（相同）
    fmt.Printf("ptr 指向的值: %d\n", *ptr)  // 42
}
```

### * 解引用運算符

```go
package main

import "fmt"

func main() {
    x := 42
    ptr := &x

    // * 用於兩個地方：
    // 1. 聲明指針類型
    var p *int = &x

    // 2. 解引用（獲取指針指向的值）
    value := *ptr

    fmt.Println(value)  // 42

    // 修改指針指向的值
    *ptr = 100
    fmt.Println(x)  // 100
}
```

**對比 JavaScript（沒有直接等價物）：**
```javascript
// JavaScript 沒有指針運算符
// 對象引用是自動的

let obj = { value: 42 };
let ref = obj;  // ref 引用 obj（自動）

ref.value = 100;
console.log(obj.value);  // 100
```

## 值傳遞 vs 指針傳遞

### Go 默認是值傳遞

```go
package main

import "fmt"

func modifyValue(x int) {
    x = 100  // 修改的是副本
}

func modifyPointer(ptr *int) {
    *ptr = 100  // 修改的是原始值
}

func main() {
    a := 42

    modifyValue(a)
    fmt.Println(a)  // 42（沒有改變）

    modifyPointer(&a)
    fmt.Println(a)  // 100（改變了）
}
```

**對比 JavaScript:**
```javascript
// 基本類型：值傳遞
function modifyValue(x) {
    x = 100;  // 修改的是副本
}

let a = 42;
modifyValue(a);
console.log(a);  // 42（沒有改變）

// 對象：引用傳遞（自動）
function modifyObject(obj) {
    obj.value = 100;  // 修改原始對象
}

let o = { value: 42 };
modifyObject(o);
console.log(o.value);  // 100（改變了）

// 但重新賦值不影響原對象
function reassignObject(obj) {
    obj = { value: 100 };  // 只改變局部變量
}

reassignObject(o);
console.log(o.value);  // 100（沒有改變）
```

### 結構體的值傳遞和指針傳遞

```go
package main

import "fmt"

type Person struct {
    Name string
    Age  int
}

// 值接收者（複製整個結構體）
func (p Person) BirthdayValue() {
    p.Age++  // 修改的是副本
}

// 指針接收者（引用原始結構體）
func (p *Person) BirthdayPointer() {
    p.Age++  // 修改原始值
}

func main() {
    person := Person{Name: "Alice", Age: 30}

    person.BirthdayValue()
    fmt.Println(person.Age)  // 30（沒有改變）

    person.BirthdayPointer()
    fmt.Println(person.Age)  // 31（改變了）

    // Go 會自動處理指針解引用
    // 以下兩種寫法等價：
    person.BirthdayPointer()
    (&person).BirthdayPointer()
}
```

**對比 JavaScript:**
```javascript
class Person {
    constructor(name, age) {
        this.name = name;
        this.age = age;
    }

    birthday() {
        this.age++;  // 總是修改原始對象
    }
}

let person = new Person("Alice", 30);
person.birthday();
console.log(person.age);  // 31

// JavaScript 的類方法總是操作引用
// 無法選擇值傳遞還是引用傳遞
```

## 何時使用指針

### 1. 修改函數參數

```go
package main

import "fmt"

func swap(a, b *int) {
    *a, *b = *b, *a
}

func main() {
    x, y := 1, 2
    fmt.Println(x, y)  // 1 2

    swap(&x, &y)
    fmt.Println(x, y)  // 2 1
}
```

**對比 JavaScript:**
```javascript
// JavaScript 需要返回新值
function swap(a, b) {
    return [b, a];
}

let x = 1, y = 2;
[x, y] = swap(x, y);
console.log(x, y);  // 2 1

// 或使用對象（引用傳遞）
function swapObj(obj) {
    [obj.a, obj.b] = [obj.b, obj.a];
}

let o = { a: 1, b: 2 };
swapObj(o);
console.log(o.a, o.b);  // 2 1
```

### 2. 避免大結構體的複製

```go
package main

import "fmt"

type LargeStruct struct {
    Data [10000]int
}

// ❌ 不好：複製整個結構體
func processValue(s LargeStruct) {
    // 10000 個 int 被複製
    fmt.Println(s.Data[0])
}

// ✅ 好：只複製指針
func processPointer(s *LargeStruct) {
    // 只複製一個指針（8 字節）
    fmt.Println(s.Data[0])
}

func main() {
    large := LargeStruct{}

    processValue(large)   // 慢
    processPointer(&large) // 快
}
```

**對比 JavaScript:**
```javascript
// JavaScript 對象總是引用傳遞
// 不會複製大對象

let large = {
    data: new Array(10000).fill(0)
};

function process(obj) {
    console.log(obj.data[0]);  // 傳遞引用，不複製
}

process(large);
```

### 3. 實現可選值（nil）

```go
package main

import "fmt"

type Config struct {
    Host     string
    Port     *int  // 可選的端口
    Timeout  *int  // 可選的超時
}

func main() {
    port := 8080

    config := Config{
        Host: "localhost",
        Port: &port,  // 有值
        // Timeout 為 nil（未設置）
    }

    if config.Port != nil {
        fmt.Println("Port:", *config.Port)
    }

    if config.Timeout != nil {
        fmt.Println("Timeout:", *config.Timeout)
    } else {
        fmt.Println("Timeout not set")
    }
}
```

**對比 JavaScript:**
```javascript
// JavaScript 使用 null/undefined
const config = {
    host: "localhost",
    port: 8080,
    timeout: null  // 或 undefined
};

if (config.port != null) {
    console.log("Port:", config.port);
}

if (config.timeout != null) {
    console.log("Timeout:", config.timeout);
} else {
    console.log("Timeout not set");
}
```

## 指針的零值

### nil 指針

```go
package main

import "fmt"

func main() {
    var ptr *int
    fmt.Println(ptr)  // <nil>

    if ptr == nil {
        fmt.Println("Pointer is nil")
    }

    // ❌ 解引用 nil 指針會 panic
    // fmt.Println(*ptr)  // panic: runtime error

    // ✅ 使用前檢查
    if ptr != nil {
        fmt.Println(*ptr)
    }

    // 初始化指針
    x := 42
    ptr = &x
    fmt.Println(*ptr)  // 42
}
```

**對比 JavaScript:**
```javascript
let ptr = null;
console.log(ptr);  // null

if (ptr === null) {
    console.log("Pointer is null");
}

// ❌ 訪問 null 的屬性會錯誤
// console.log(ptr.value);  // TypeError

// ✅ 使用前檢查
if (ptr !== null) {
    console.log(ptr.value);
}

let obj = { value: 42 };
ptr = obj;
console.log(ptr.value);  // 42
```

## 指針陷阱

### 陷阱 1：循環中的指針

```go
package main

import "fmt"

func main() {
    numbers := []int{1, 2, 3}
    var ptrs []*int

    // ❌ 錯誤：所有指針都指向同一個變量
    for _, num := range numbers {
        ptrs = append(ptrs, &num)  // num 是同一個變量
    }

    for _, ptr := range ptrs {
        fmt.Println(*ptr)  // 3 3 3（都是最後一個值）
    }

    // ✅ 正確：創建新變量
    for _, num := range numbers {
        num := num  // 創建新變量
        ptrs = append(ptrs, &num)
    }

    for _, ptr := range ptrs {
        fmt.Println(*ptr)  // 1 2 3
    }

    // ✅ 或使用索引
    for i := range numbers {
        ptrs = append(ptrs, &numbers[i])
    }
}
```

**對比 JavaScript:**
```javascript
// JavaScript 沒有這個問題（let 有塊作用域）
let numbers = [1, 2, 3];
let refs = [];

for (let num of numbers) {
    refs.push({ value: num });  // 每次創建新對象
}

refs.forEach(ref => console.log(ref.value));  // 1 2 3

// var 有類似問題
for (var i = 0; i < 3; i++) {
    setTimeout(() => console.log(i), 100);  // 3 3 3
}

// let 解決了這個問題
for (let i = 0; i < 3; i++) {
    setTimeout(() => console.log(i), 100);  // 0 1 2
}
```

### 陷阱 2：返回局部變量的指針

```go
package main

import "fmt"

// ✅ Go 允許返回局部變量的指針（會逃逸到堆）
func createPointer() *int {
    x := 42
    return &x  // 安全：x 會分配在堆上
}

func main() {
    ptr := createPointer()
    fmt.Println(*ptr)  // 42
}
```

**C/C++ 中這是危險的，但 Go 會自動處理（逃逸分析）。**

**對比 JavaScript:**
```javascript
// JavaScript 沒有這個問題（總是引用）
function createObject() {
    let obj = { value: 42 };
    return obj;  // 總是安全的
}

let ref = createObject();
console.log(ref.value);  // 42
```

### 陷阱 3：nil 指針解引用

```go
package main

import "fmt"

type Person struct {
    Name string
}

func (p *Person) Greet() {
    // ✅ 即使 p 是 nil，方法也可以被調用
    if p == nil {
        fmt.Println("Hello from nil")
        return
    }
    fmt.Println("Hello,", p.Name)
}

func main() {
    var p *Person  // nil
    p.Greet()      // Hello from nil（不會 panic）

    // ❌ 但訪問字段會 panic
    // fmt.Println(p.Name)  // panic
}
```

## 指針和切片/Map

### 切片是引用類型

```go
package main

import "fmt"

func modifySlice(s []int) {
    s[0] = 100  // 修改的是原始切片的元素
}

func reassignSlice(s []int) {
    s = []int{100, 200}  // 只改變局部變量
}

func main() {
    slice := []int{1, 2, 3}

    modifySlice(slice)
    fmt.Println(slice)  // [100 2 3]（改變了）

    reassignSlice(slice)
    fmt.Println(slice)  // [100 2 3]（沒有改變）

    // 需要指針才能重新賦值
    reassignSlicePointer(&slice)
    fmt.Println(slice)  // [100 200]
}

func reassignSlicePointer(s *[]int) {
    *s = []int{100, 200}
}
```

**對比 JavaScript:**
```javascript
function modifyArray(arr) {
    arr[0] = 100;  // 修改原數組
}

function reassignArray(arr) {
    arr = [100, 200];  // 只改變局部變量
}

let array = [1, 2, 3];

modifyArray(array);
console.log(array);  // [100, 2, 3]

reassignArray(array);
console.log(array);  // [100, 2, 3]

// JavaScript 沒有指針，無法重新賦值外部變量
// 需要返回新值
function reassignArrayReturn(arr) {
    return [100, 200];
}

array = reassignArrayReturn(array);
console.log(array);  // [100, 200]
```

### Map 也是引用類型

```go
package main

import "fmt"

func modifyMap(m map[string]int) {
    m["key"] = 100  // 修改原始 map
}

func main() {
    myMap := map[string]int{"key": 1}

    modifyMap(myMap)
    fmt.Println(myMap)  // map[key:100]
}
```

## 實際應用示例

### 示例 1：構建器模式

```go
package main

import "fmt"

type Config struct {
    Host    string
    Port    int
    Timeout int
}

func NewConfig() *Config {
    return &Config{
        Host:    "localhost",
        Port:    8080,
        Timeout: 30,
    }
}

func (c *Config) WithHost(host string) *Config {
    c.Host = host
    return c  // 返回指針，支持鏈式調用
}

func (c *Config) WithPort(port int) *Config {
    c.Port = port
    return c
}

func main() {
    config := NewConfig().
        WithHost("example.com").
        WithPort(9090)

    fmt.Printf("%+v\n", config)
    // &{Host:example.com Port:9090 Timeout:30}
}
```

**對比 JavaScript:**
```javascript
class Config {
    constructor() {
        this.host = "localhost";
        this.port = 8080;
        this.timeout = 30;
    }

    withHost(host) {
        this.host = host;
        return this;  // 鏈式調用
    }

    withPort(port) {
        this.port = port;
        return this;
    }
}

const config = new Config()
    .withHost("example.com")
    .withPort(9090);

console.log(config);
// Config { host: 'example.com', port: 9090, timeout: 30 }
```

### 示例 2：鏈表

```go
package main

import "fmt"

type Node struct {
    Value int
    Next  *Node  // 指向下一個節點
}

func main() {
    // 創建鏈表：1 -> 2 -> 3
    head := &Node{Value: 1}
    head.Next = &Node{Value: 2}
    head.Next.Next = &Node{Value: 3}

    // 遍歷鏈表
    current := head
    for current != nil {
        fmt.Println(current.Value)
        current = current.Next
    }
}
```

**對比 JavaScript:**
```javascript
class Node {
    constructor(value) {
        this.value = value;
        this.next = null;
    }
}

// 創建鏈表：1 -> 2 -> 3
let head = new Node(1);
head.next = new Node(2);
head.next.next = new Node(3);

// 遍歷鏈表
let current = head;
while (current !== null) {
    console.log(current.value);
    current = current.next;
}
```

## new 函數

```go
package main

import "fmt"

func main() {
    // new 分配內存並返回指針
    ptr := new(int)
    fmt.Println(ptr)   // 0xc000018030
    fmt.Println(*ptr)  // 0（零值）

    *ptr = 42
    fmt.Println(*ptr)  // 42

    // 等價於
    var x int
    ptr2 := &x

    // 結構體
    p := new(Person)
    fmt.Println(p)  // &{ 0}（零值）

    p.Name = "Alice"
    p.Age = 30
}

type Person struct {
    Name string
    Age  int
}
```

**對比 JavaScript:**
```javascript
// JavaScript 沒有 new 用於分配內存
// new 用於創建對象

class Person {
    constructor(name, age) {
        this.name = name || "";
        this.age = age || 0;
    }
}

let p = new Person();
console.log(p);  // Person { name: '', age: 0 }

p.name = "Alice";
p.age = 30;
```

## 重點總結

### 指針概念

1. **& 運算符**：取地址
2. *** 運算符**：解引用（聲明和使用）
3. **nil**：指針的零值
4. **值傳遞**：Go 默認行為
5. **指針傳遞**：顯式使用 `&`

### Go vs JavaScript

| 特性 | Go | JavaScript |
|------|-----|------------|
| **基本類型** | 值傳遞 | 值傳遞 |
| **結構體/對象** | 值傳遞（默認） | 引用傳遞 |
| **數組** | 值傳遞 | 引用傳遞 |
| **切片/Map** | 引用特性 | 引用傳遞 |
| **顯式指針** | ✅ 有（&, *） | ❌ 無 |
| **nil 檢查** | 必須 | null/undefined 檢查 |

### 何時使用指針

1. **需要修改參數**
2. **避免大結構體複製**
3. **實現可選值（nil）**
4. **實現數據結構**（鏈表、樹）
5. **方法需要修改接收者**

### 最佳實踐

1. **小結構體**：值傳遞
2. **大結構體**：指針傳遞
3. **需要修改**：指針接收者
4. **只讀操作**：值接收者
5. **總是檢查 nil**

## 練習題

### 基礎題

1. **指針基礎**
   - 創建變量並獲取其地址
   - 通過指針修改值
   - 理解 & 和 * 的用法

2. **函數參數**
   - 實現 swap 函數（交換兩個變量）
   - 對比值傳遞和指針傳遞
   - 理解參數傳遞機制

3. **nil 檢查**
   - 創建 nil 指針
   - 安全地使用指針
   - 處理 nil 情況

### 進階題

4. **方法接收者**
   - 實現值接收者和指針接收者
   - 理解兩者的區別
   - 選擇合適的接收者類型

5. **循環陷阱**
   - 重現循環中的指針陷阱
   - 實現正確的解決方案
   - 理解變量作用域

6. **性能對比**
   - 對比大結構體的值傳遞和指針傳遞
   - 測試性能差異
   - 理解內存分配

### 實戰題

7. **數據結構**
   - 實現鏈表
   - 實現二叉樹
   - 使用指針連接節點

8. **構建器模式**
   - 實現鏈式調用的配置器
   - 使用指針返回
   - 對比 JavaScript 實現

9. **對比分析**
   - 列出 Go 指針和 JavaScript 引用的差異
   - 創建對照示例
   - 記錄學習心得

## 總結

恭喜你完成了 Part 1-2 的所有章節！你已經學習了：

### Part 1: 入門基礎
- 為什麼選擇 Go
- 環境安裝與配置
- Hello World
- 模組管理
- Go 命令行工具

### Part 2: 基本語法
- 變量與常量
- 數據類型
- 運算符
- if/else 流程控制
- 循環
- 函數
- 錯誤處理
- 指針

## 下一步

繼續學習 Part 3-5：
- Part 3: 複合類型（數組、切片、Map、結構體）
- Part 4: 並發編程（Goroutines、Channels）
- Part 5: 高級特性（接口、泛型、反射）

祝你在 Go 語言學習之旅中順利前進！
