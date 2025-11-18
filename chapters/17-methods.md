# Chapter 17: 方法與接收器 (Methods & Receivers)

## 概述

在 Go 中，方法是與特定類型關聯的函數。方法通過接收器（receiver）與類型綁定，接收器可以是值接收器或指針接收器。這與 JavaScript 的類方法類似，但 Go 的實現更加靈活和明確。

## Node.js 與 Go 的對比

| 特性 | Node.js | Go |
|------|---------|-----|
| 方法定義 | 在 class 內部 | 在類型外部用接收器 |
| this 綁定 | `this` 關鍵字 | 接收器參數 |
| 值修改 | 總是可以修改 | 指針接收器才能修改 |
| 方法調用 | `obj.method()` | `obj.Method()` |
| 靜態方法 | `static method()` | 包級函數 |
| 繼承方法 | `extends` | 嵌入類型 |
| 方法重寫 | 支持 | 通過接口實現 |

## 詳細概念解釋

### 1. 方法的定義

方法是帶接收器的函數：

```go
func (接收器 類型) 方法名(參數) 返回值 {
    // 方法體
}
```

### 2. 值接收器 vs 指針接收器

**值接收器：**
- 接收器是類型的副本
- 不能修改原始值
- 適用於小型結構體或不需要修改的場景

**指針接收器：**
- 接收器是類型的指針
- 可以修改原始值
- 避免大結構體的複製
- 大多數情況下推薦使用

### 3. 接收器選擇規則

使用指針接收器的情況：
- 需要修改接收器
- 結構體較大，避免複製
- 保持一致性（如果有一個方法用指針，其他也用指針）

使用值接收器的情況：
- 結構體很小
- 不需要修改
- 類型是 map、slice 等引用類型

## 代碼示例

### 示例 1: 基本方法定義

**Go:**
```go
package main

import "fmt"

type Rectangle struct {
    Width  float64
    Height float64
}

// 值接收器方法
func (r Rectangle) Area() float64 {
    return r.Width * r.Height
}

// 值接收器方法
func (r Rectangle) Perimeter() float64 {
    return 2 * (r.Width + r.Height)
}

// 指針接收器方法
func (r *Rectangle) Scale(factor float64) {
    r.Width *= factor
    r.Height *= factor
}

// 值接收器嘗試修改（不會生效）
func (r Rectangle) ScaleWrong(factor float64) {
    r.Width *= factor
    r.Height *= factor
}

func main() {
    rect := Rectangle{Width: 10, Height: 5}

    // 調用值接收器方法
    fmt.Println("面積:", rect.Area())        // 50
    fmt.Println("周長:", rect.Perimeter())   // 30

    // 調用指針接收器方法
    fmt.Printf("原始: %+v\n", rect)
    rect.Scale(2)
    fmt.Printf("放大後: %+v\n", rect)  // {Width:20 Height:10}
    fmt.Println("新面積:", rect.Area()) // 200

    // 值接收器無法修改
    rect.ScaleWrong(2)
    fmt.Printf("ScaleWrong 後: %+v\n", rect) // 未改變

    // Go 會自動處理指針和值的轉換
    rectPtr := &Rectangle{Width: 8, Height: 4}
    fmt.Println("指針調用值方法:", rectPtr.Area()) // 32
    rectPtr.Scale(3)
    fmt.Printf("指針放大: %+v\n", rectPtr)
}
```

**Node.js 對比:**
```javascript
class Rectangle {
    constructor(width, height) {
        this.width = width;
        this.height = height;
    }

    // 所有方法都可以修改 this
    area() {
        return this.width * this.height;
    }

    perimeter() {
        return 2 * (this.width + this.height);
    }

    scale(factor) {
        this.width *= factor;
        this.height *= factor;
    }
}

const rect = new Rectangle(10, 5);

// 調用方法
console.log("面積:", rect.area());        // 50
console.log("周長:", rect.perimeter());   // 30

// 修改對象
console.log("原始:", rect);
rect.scale(2);
console.log("放大後:", rect);  // { width: 20, height: 10 }
console.log("新面積:", rect.area()); // 200

// JavaScript 中 this 綁定可能有問題
const area = rect.area;
// console.log(area());  // 錯誤：this 未定義

// 需要綁定 this
const boundArea = rect.area.bind(rect);
console.log("綁定後:", boundArea()); // 200

// 或使用箭頭函數
class Rectangle2 {
    constructor(width, height) {
        this.width = width;
        this.height = height;

        // 箭頭函數自動綁定 this
        this.area = () => this.width * this.height;
    }
}

const rect2 = new Rectangle2(10, 5);
const area2 = rect2.area;
console.log("箭頭函數:", area2()); // 50
```

### 示例 2: 值接收器 vs 指針接收器的實際區別

**Go:**
```go
package main

import "fmt"

type Counter struct {
    Count int
}

// 值接收器：不會修改原始值
func (c Counter) IncrementValue() {
    c.Count++
    fmt.Println("值接收器內部:", c.Count)
}

// 指針接收器：會修改原始值
func (c *Counter) IncrementPointer() {
    c.Count++
    fmt.Println("指針接收器內部:", c.Count)
}

// 值接收器：返回新值
func (c Counter) Add(n int) Counter {
    c.Count += n
    return c
}

// 指針接收器：返回自身以支持鏈式調用
func (c *Counter) AddChain(n int) *Counter {
    c.Count += n
    return c
}

func main() {
    // 1. 值接收器測試
    counter1 := Counter{Count: 0}
    fmt.Println("初始值:", counter1.Count) // 0

    counter1.IncrementValue()
    fmt.Println("調用後:", counter1.Count) // 0 - 未改變

    // 2. 指針接收器測試
    counter2 := Counter{Count: 0}
    fmt.Println("\n初始值:", counter2.Count) // 0

    counter2.IncrementPointer()
    fmt.Println("調用後:", counter2.Count) // 1 - 已改變

    // 3. 值接收器返回新值
    counter3 := Counter{Count: 10}
    newCounter := counter3.Add(5)
    fmt.Println("\n原始:", counter3.Count)  // 10
    fmt.Println("新值:", newCounter.Count) // 15

    // 4. 鏈式調用
    counter4 := &Counter{Count: 0}
    counter4.AddChain(5).AddChain(10).AddChain(3)
    fmt.Println("\n鏈式調用後:", counter4.Count) // 18

    // 5. Go 的自動轉換
    counter5 := Counter{Count: 0}
    counter5.IncrementPointer() // Go 自動轉為 (&counter5).IncrementPointer()
    fmt.Println("\n自動轉換:", counter5.Count) // 1

    counter6 := &Counter{Count: 0}
    counter6.IncrementValue() // Go 自動轉為 (*counter6).IncrementValue()
    fmt.Println("自動轉換:", counter6.Count) // 0 - 仍然未改變
}
```

**Node.js 對比:**
```javascript
class Counter {
    constructor(count = 0) {
        this.count = count;
    }

    // JavaScript 中所有方法都可以修改 this
    increment() {
        this.count++;
        console.log("方法內部:", this.count);
    }

    // 返回新對象（不修改原對象）
    add(n) {
        return new Counter(this.count + n);
    }

    // 返回 this 支持鏈式調用
    addChain(n) {
        this.count += n;
        return this;
    }
}

// 1. 正常修改
const counter1 = new Counter(0);
console.log("初始值:", counter1.count); // 0

counter1.increment();
console.log("調用後:", counter1.count); // 1 - 已改變

// 2. 返回新對象
const counter2 = new Counter(10);
const newCounter = counter2.add(5);
console.log("\n原始:", counter2.count);  // 10
console.log("新值:", newCounter.count); // 15

// 3. 鏈式調用
const counter3 = new Counter(0);
counter3.addChain(5).addChain(10).addChain(3);
console.log("\n鏈式調用後:", counter3.count); // 18

// 如果要實現不可變對象，可以使用 Object.freeze
class ImmutableCounter {
    constructor(count = 0) {
        this.count = count;
        Object.freeze(this);
    }

    add(n) {
        return new ImmutableCounter(this.count + n);
    }
}

const immutable = new ImmutableCounter(10);
// immutable.count = 20; // 靜默失敗（嚴格模式下報錯）
console.log("\n不可變:", immutable.count); // 10
const result = immutable.add(5);
console.log("新對象:", result.count); // 15
```

### 示例 3: 方法與函數的區別

**Go:**
```go
package main

import (
    "fmt"
    "math"
)

type Point struct {
    X, Y float64
}

// 方法：與類型綁定
func (p Point) Distance(other Point) float64 {
    dx := p.X - other.X
    dy := p.Y - other.Y
    return math.Sqrt(dx*dx + dy*dy)
}

// 函數：獨立存在
func Distance(p1, p2 Point) float64 {
    dx := p1.X - p2.X
    dy := p1.Y - p2.Y
    return math.Sqrt(dx*dx + dy*dy)
}

// 指針接收器方法
func (p *Point) Move(dx, dy float64) {
    p.X += dx
    p.Y += dy
}

// 函數版本
func Move(p *Point, dx, dy float64) {
    p.X += dx
    p.Y += dy
}

func main() {
    p1 := Point{X: 0, Y: 0}
    p2 := Point{X: 3, Y: 4}

    // 使用方法
    dist1 := p1.Distance(p2)
    fmt.Println("方法調用:", dist1) // 5

    // 使用函數
    dist2 := Distance(p1, p2)
    fmt.Println("函數調用:", dist2) // 5

    // 方法調用更自然
    p1.Move(1, 1)
    fmt.Printf("移動後: %+v\n", p1)

    // 函數調用需要傳遞指針
    Move(&p2, 1, 1)
    fmt.Printf("移動後: %+v\n", p2)

    // 方法可以鏈式調用
    type Point3D struct {
        X, Y, Z float64
    }

    func (p *Point3D) SetX(x float64) *Point3D {
        p.X = x
        return p
    }

    func (p *Point3D) SetY(y float64) *Point3D {
        p.Y = y
        return p
    }

    func (p *Point3D) SetZ(z float64) *Point3D {
        p.Z = z
        return p
    }

    p3d := &Point3D{}
    p3d.SetX(1).SetY(2).SetZ(3)
    fmt.Printf("3D 點: %+v\n", p3d)
}
```

**Node.js 對比:**
```javascript
class Point {
    constructor(x, y) {
        this.x = x;
        this.y = y;
    }

    // 方法
    distance(other) {
        const dx = this.x - other.x;
        const dy = this.y - other.y;
        return Math.sqrt(dx * dx + dy * dy);
    }

    move(dx, dy) {
        this.x += dx;
        this.y += dy;
    }

    // 鏈式調用
    setX(x) {
        this.x = x;
        return this;
    }

    setY(y) {
        this.y = y;
        return this;
    }

    // 靜態方法（類似 Go 的包級函數）
    static distance(p1, p2) {
        const dx = p1.x - p2.x;
        const dy = p1.y - p2.y;
        return Math.sqrt(dx * dx + dy * dy);
    }

    static move(p, dx, dy) {
        p.x += dx;
        p.y += dy;
    }
}

const p1 = new Point(0, 0);
const p2 = new Point(3, 4);

// 使用方法
const dist1 = p1.distance(p2);
console.log("方法調用:", dist1); // 5

// 使用靜態方法
const dist2 = Point.distance(p1, p2);
console.log("靜態方法:", dist2); // 5

// 修改對象
p1.move(1, 1);
console.log("移動後:", p1);

Point.move(p2, 1, 1);
console.log("移動後:", p2);

// 鏈式調用
p1.setX(10).setY(20);
console.log("鏈式調用後:", p1);
```

### 示例 4: 不同類型的方法

**Go:**
```go
package main

import (
    "fmt"
    "strings"
)

// 1. 自定義類型的方法
type MyInt int

func (i MyInt) IsEven() bool {
    return i%2 == 0
}

func (i MyInt) Double() MyInt {
    return i * 2
}

// 2. 字符串切片的方法
type StringSlice []string

func (s StringSlice) Join(sep string) string {
    return strings.Join(s, sep)
}

func (s StringSlice) Contains(str string) bool {
    for _, v := range s {
        if v == str {
            return true
        }
    }
    return false
}

// 3. Map 類型的方法
type StringIntMap map[string]int

func (m StringIntMap) Keys() []string {
    keys := make([]string, 0, len(m))
    for k := range m {
        keys = append(keys, k)
    }
    return keys
}

func (m StringIntMap) Values() []int {
    values := make([]int, 0, len(m))
    for _, v := range m {
        values = append(values, v)
    }
    return values
}

// 4. 函數類型的方法
type Handler func(string) string

func (h Handler) Apply(input string) string {
    return h(input)
}

func (h Handler) Chain(other Handler) Handler {
    return func(input string) string {
        return other(h(input))
    }
}

func main() {
    // 1. 整數類型方法
    var num MyInt = 10
    fmt.Println("是偶數:", num.IsEven())    // true
    fmt.Println("翻倍:", num.Double())      // 20

    // 2. 字符串切片方法
    fruits := StringSlice{"apple", "banana", "cherry"}
    fmt.Println("連接:", fruits.Join(", "))
    fmt.Println("包含 banana:", fruits.Contains("banana"))

    // 3. Map 方法
    scores := StringIntMap{
        "Alice": 95,
        "Bob":   87,
        "Charlie": 92,
    }
    fmt.Println("鍵:", scores.Keys())
    fmt.Println("值:", scores.Values())

    // 4. 函數類型方法
    upper := Handler(strings.ToUpper)
    prefix := Handler(func(s string) string {
        return "Hello, " + s
    })

    result := upper.Apply("world")
    fmt.Println("應用:", result) // WORLD

    combined := prefix.Chain(upper)
    fmt.Println("鏈式:", combined.Apply("world")) // HELLO, WORLD
}
```

**Node.js 對比:**
```javascript
// JavaScript 可以給原型添加方法

// 1. 擴展 Number（不推薦直接修改原生類型）
// Number.prototype.isEven = function() {
//     return this % 2 === 0;
// };

// 更好的方式：創建包裝類
class MyNumber {
    constructor(value) {
        this.value = value;
    }

    isEven() {
        return this.value % 2 === 0;
    }

    double() {
        return new MyNumber(this.value * 2);
    }
}

const num = new MyNumber(10);
console.log("是偶數:", num.isEven());    // true
console.log("翻倍:", num.double().value); // 20

// 2. 擴展 Array
class StringArray extends Array {
    join(sep) {
        return super.join(sep);
    }

    contains(str) {
        return this.includes(str);
    }
}

const fruits = new StringArray("apple", "banana", "cherry");
console.log("連接:", fruits.join(", "));
console.log("包含 banana:", fruits.contains("banana"));

// 3. Map 類擴展
class StringIntMap extends Map {
    keys() {
        return Array.from(super.keys());
    }

    values() {
        return Array.from(super.values());
    }
}

const scores = new StringIntMap([
    ["Alice", 95],
    ["Bob", 87],
    ["Charlie", 92]
]);
console.log("鍵:", scores.keys());
console.log("值:", scores.values());

// 4. 函數包裝
class Handler {
    constructor(fn) {
        this.fn = fn;
    }

    apply(input) {
        return this.fn(input);
    }

    chain(other) {
        return new Handler((input) => other.fn(this.fn(input)));
    }
}

const upper = new Handler(s => s.toUpperCase());
const prefix = new Handler(s => "Hello, " + s);

const result = upper.apply("world");
console.log("應用:", result); // WORLD

const combined = prefix.chain(upper);
console.log("鏈式:", combined.apply("world")); // HELLO, WORLD
```

### 示例 5: 嵌入類型的方法提升

**Go:**
```go
package main

import "fmt"

type Person struct {
    Name string
    Age  int
}

func (p Person) Introduce() {
    fmt.Printf("我是 %s，%d 歲\n", p.Name, p.Age)
}

func (p *Person) HaveBirthday() {
    p.Age++
    fmt.Printf("%s 現在 %d 歲了\n", p.Name, p.Age)
}

// 嵌入 Person
type Employee struct {
    Person  // 匿名字段
    Company string
    Salary  int
}

// Employee 自己的方法
func (e Employee) Work() {
    fmt.Printf("%s 在 %s 工作\n", e.Name, e.Company)
}

// 重寫 Introduce 方法
func (e Employee) Introduce() {
    fmt.Printf("我是 %s，%d 歲，在 %s 工作\n", e.Name, e.Age, e.Company)
}

func main() {
    // 創建員工
    emp := Employee{
        Person: Person{
            Name: "Alice",
            Age:  30,
        },
        Company: "Tech Corp",
        Salary:  80000,
    }

    // 調用提升的方法
    // emp.Introduce() 會調用 Employee 的 Introduce
    emp.Introduce()  // 我是 Alice，30 歲，在 Tech Corp 工作

    // 調用嵌入類型的原始方法
    emp.Person.Introduce()  // 我是 Alice，30 歲

    // 調用自己的方法
    emp.Work()  // Alice 在 Tech Corp 工作

    // 指針方法也會被提升
    emp.HaveBirthday()  // Alice 現在 31 歲了
    fmt.Println("年齡:", emp.Age)  // 31

    // 方法集
    fmt.Println("\n=== 方法集演示 ===")

    // 值類型可以調用值接收器和指針接收器的方法
    p := Person{Name: "Bob", Age: 25}
    p.Introduce()
    p.HaveBirthday()  // Go 自動轉換為 (&p).HaveBirthday()

    // 指針類型也可以調用所有方法
    pPtr := &Person{Name: "Charlie", Age: 35}
    pPtr.Introduce()   // Go 自動轉換為 (*pPtr).Introduce()
    pPtr.HaveBirthday()
}
```

**Node.js 對比:**
```javascript
class Person {
    constructor(name, age) {
        this.name = name;
        this.age = age;
    }

    introduce() {
        console.log(`我是 ${this.name}，${this.age} 歲`);
    }

    haveBirthday() {
        this.age++;
        console.log(`${this.name} 現在 ${this.age} 歲了`);
    }
}

// 繼承
class Employee extends Person {
    constructor(name, age, company, salary) {
        super(name, age);
        this.company = company;
        this.salary = salary;
    }

    work() {
        console.log(`${this.name} 在 ${this.company} 工作`);
    }

    // 重寫方法
    introduce() {
        console.log(`我是 ${this.name}，${this.age} 歲，在 ${this.company} 工作`);
    }
}

// 創建員工
const emp = new Employee("Alice", 30, "Tech Corp", 80000);

// 調用重寫的方法
emp.introduce();  // 我是 Alice，30 歲，在 Tech Corp 工作

// 調用父類方法需要使用 super
class Employee2 extends Person {
    constructor(name, age, company, salary) {
        super(name, age);
        this.company = company;
        this.salary = salary;
    }

    introduce() {
        super.introduce();  // 調用父類方法
        console.log(`在 ${this.company} 工作`);
    }
}

const emp2 = new Employee2("Bob", 25, "StartUp Inc", 60000);
emp2.introduce();

// 調用自己的方法
emp.work();

// 修改狀態
emp.haveBirthday();
console.log("年齡:", emp.age);

// 使用組合而非繼承
class EmployeeComposition {
    constructor(person, company, salary) {
        this.person = person;
        this.company = company;
        this.salary = salary;
    }

    introduce() {
        console.log(`我是 ${this.person.name}，${this.person.age} 歲，在 ${this.company} 工作`);
    }

    work() {
        console.log(`${this.person.name} 在 ${this.company} 工作`);
    }

    // 委託給 person
    haveBirthday() {
        this.person.haveBirthday();
    }
}

const charlie = new Person("Charlie", 35);
const emp3 = new EmployeeComposition(charlie, "Big Corp", 100000);
emp3.introduce();
emp3.haveBirthday();
```

### 示例 6: 方法表達式和方法值

**Go:**
```go
package main

import "fmt"

type Point struct {
    X, Y int
}

func (p Point) Add(other Point) Point {
    return Point{X: p.X + other.X, Y: p.Y + other.Y}
}

func (p *Point) Scale(factor int) {
    p.X *= factor
    p.Y *= factor
}

func main() {
    p1 := Point{X: 1, Y: 2}
    p2 := Point{X: 3, Y: 4}

    // 1. 正常方法調用
    p3 := p1.Add(p2)
    fmt.Printf("正常調用: %+v\n", p3)

    // 2. 方法值（Method Value）
    // 綁定了接收器的方法
    addMethod := p1.Add
    p4 := addMethod(p2)
    fmt.Printf("方法值: %+v\n", p4)

    // 3. 方法表達式（Method Expression）
    // 未綁定接收器，需要顯式傳遞
    addExpr := Point.Add
    p5 := addExpr(p1, p2)
    fmt.Printf("方法表達式: %+v\n", p5)

    // 4. 指針方法的方法值
    scaleMethod := p1.Scale
    scaleMethod(2)
    fmt.Printf("縮放後: %+v\n", p1) // {2 4}

    // 5. 指針方法的方法表達式
    scaleExpr := (*Point).Scale
    scaleExpr(&p1, 3)
    fmt.Printf("再次縮放: %+v\n", p1) // {6 12}

    // 6. 方法作為函數參數
    applyOp := func(p1, p2 Point, op func(Point, Point) Point) Point {
        return op(p1, p2)
    }

    result := applyOp(
        Point{X: 5, Y: 6},
        Point{X: 1, Y: 2},
        Point.Add,
    )
    fmt.Printf("函數參數: %+v\n", result)
}
```

**Node.js 對比:**
```javascript
class Point {
    constructor(x, y) {
        this.x = x;
        this.y = y;
    }

    add(other) {
        return new Point(this.x + other.x, this.y + other.y);
    }

    scale(factor) {
        this.x *= factor;
        this.y *= factor;
    }
}

const p1 = new Point(1, 2);
const p2 = new Point(3, 4);

// 1. 正常方法調用
const p3 = p1.add(p2);
console.log("正常調用:", p3);

// 2. 方法引用（需要 bind）
const addMethod = p1.add.bind(p1);
const p4 = addMethod(p2);
console.log("綁定方法:", p4);

// 3. 方法作為函數（失去 this 綁定）
const addUnbound = p1.add;
// const p5 = addUnbound(p2);  // 錯誤：this 是 undefined

// 需要使用 call 或 apply
const p5 = addUnbound.call(p1, p2);
console.log("使用 call:", p5);

// 4. 箭頭函數自動綁定
class Point2 {
    constructor(x, y) {
        this.x = x;
        this.y = y;

        this.add = (other) => {
            return new Point2(this.x + other.x, this.y + other.y);
        };
    }
}

const p6 = new Point2(1, 2);
const p7 = new Point2(3, 4);
const addArrow = p6.add;  // 自動綁定
const p8 = addArrow(p7);
console.log("箭頭函數:", p8);

// 5. 方法作為回調
const points = [
    new Point(1, 2),
    new Point(3, 4),
    new Point(5, 6)
];

// 需要綁定 this
const origin = new Point(0, 0);
const results = points.map(p => origin.add(p));
console.log("map 結果:", results);

// 6. 方法作為函數參數
function applyOp(p1, p2, op) {
    return op.call(p1, p2);
}

const result = applyOp(
    new Point(5, 6),
    new Point(1, 2),
    Point.prototype.add
);
console.log("函數參數:", result);
```

## 重點總結

### 方法的關鍵概念

1. **方法 = 函數 + 接收器**：方法通過接收器與類型關聯
2. **值接收器 vs 指針接收器**：決定能否修改原值
3. **自動轉換**：Go 會自動在值和指針間轉換
4. **方法提升**：嵌入類型的方法會被提升到外層類型

### 選擇接收器類型的規則

使用**指針接收器**：
- 需要修改接收器
- 結構體較大
- 保持一致性

使用**值接收器**：
- 結構體很小（幾個基本類型字段）
- 不需要修改
- 接收器是 map、slice 等

### Go vs JavaScript 方法對比

| 特性 | Go | JavaScript |
|------|-----|------------|
| 定義位置 | 類型外部 | 類內部 |
| 接收器 | 顯式接收器參數 | 隱式 this |
| 值/引用 | 明確區分 | 都是引用 |
| this 綁定 | 無此問題 | 需要注意綁定 |
| 修改控制 | 接收器類型決定 | 總是可以修改 |

### 最佳實踐

1. **一致性**：同一類型的方法應全用值接收器或全用指針接收器
2. **默認指針**：不確定時優先使用指針接收器
3. **語義清晰**：方法名應清楚表達操作意圖
4. **返回 this**：支持鏈式調用時返回接收器指針
5. **小方法**：保持方法簡潔，復雜邏輯拆分為多個方法

## 練習題

### 練習 1: 銀行賬戶
創建 `BankAccount` 結構體，實現以下方法：
- `Deposit(amount)`: 存款
- `Withdraw(amount) error`: 取款（餘額不足返回錯誤）
- `Balance()`: 查詢餘額
- `Transfer(to *BankAccount, amount) error`: 轉賬

### 練習 2: 溫度轉換
創建 `Temperature` 類型，實現：
- `ToCelsius()`: 轉為攝氏度
- `ToFahrenheit()`: 轉為華氏度
- `ToKelvin()`: 轉為開爾文

### 練習 3: 鏈表操作
實現單鏈表，包含方法：
- `Append(value)`: 追加節點
- `Prepend(value)`: 前置節點
- `Delete(value) bool`: 刪除節點
- `Find(value) bool`: 查找節點
- `Reverse()`: 反轉鏈表

### 練習 4: 圖形接口
定義多種圖形（圓形、矩形、三角形），每種圖形實現：
- `Area()`: 計算面積
- `Perimeter()`: 計算周長
- `Scale(factor)`: 縮放

### 練習 5: 計數器集合
實現 `CounterSet` 類型（基於 map），提供方法：
- `Increment(key)`: 增加計數
- `Decrement(key)`: 減少計數
- `Get(key) int`: 獲取計數
- `MostCommon(n) []string`: 返回前 n 個最常見的鍵

### 答案提示

**練習 1:**
```go
type BankAccount struct {
    balance float64
}

func (b *BankAccount) Deposit(amount float64) {
    b.balance += amount
}

func (b *BankAccount) Withdraw(amount float64) error {
    if b.balance < amount {
        return fmt.Errorf("餘額不足")
    }
    b.balance -= amount
    return nil
}

func (b BankAccount) Balance() float64 {
    return b.balance
}

func (b *BankAccount) Transfer(to *BankAccount, amount float64) error {
    if err := b.Withdraw(amount); err != nil {
        return err
    }
    to.Deposit(amount)
    return nil
}
```
