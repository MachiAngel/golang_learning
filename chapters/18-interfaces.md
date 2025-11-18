# Chapter 18: 接口 (Interfaces)

## 概述

接口是 Go 中最強大的特性之一，它定義了一組方法簽名，任何類型只要實現了這些方法就自動實現了該接口（鴨子類型）。與 TypeScript 的接口不同，Go 的接口是隱式實現的，不需要顯式聲明。

## Node.js 與 Go 的對比

| 特性 | TypeScript Interface | Go Interface |
|------|---------------------|--------------|
| 實現方式 | 顯式實現 | 隱式實現（鴨子類型） |
| 類型檢查 | 編譯時 | 編譯時 |
| 方法定義 | 方法簽名 | 方法簽名 |
| 多重實現 | 支持 | 支持 |
| 組合 | extends | 嵌入 |
| 運行時檢查 | 類型斷言 | 類型斷言 |
| 空接口 | `any` | `interface{}` |

## 詳細概念解釋

### 1. 接口的定義

接口是一組方法簽名的集合：

```go
type 接口名 interface {
    方法名1(參數) 返回值
    方法名2(參數) 返回值
}
```

### 2. 隱式實現

Go 的接口是隱式實現的，只要類型實現了接口的所有方法，它就實現了該接口：

```go
type Writer interface {
    Write([]byte) (int, error)
}

// 只要有 Write 方法，就實現了 Writer 接口
type MyWriter struct{}

func (m MyWriter) Write(data []byte) (int, error) {
    // 實現
    return len(data), nil
}
```

### 3. 接口的優勢

- **解耦**：依賴抽象而非具體實現
- **多態**：同一接口的不同實現
- **可測試性**：容易 mock 和測試
- **靈活性**：運行時決定具體類型

## 代碼示例

### 示例 1: 基本接口定義和實現

**Go:**
```go
package main

import "fmt"

// 定義接口
type Speaker interface {
    Speak() string
}

type Mover interface {
    Move() string
}

// 定義類型
type Dog struct {
    Name string
}

func (d Dog) Speak() string {
    return "汪汪汪"
}

func (d Dog) Move() string {
    return "跑"
}

type Cat struct {
    Name string
}

func (c Cat) Speak() string {
    return "喵喵喵"
}

func (c Cat) Move() string {
    return "跳"
}

type Robot struct {
    Model string
}

func (r Robot) Speak() string {
    return "嗶嗶嗶"
}

// Robot 沒有實現 Mover 接口

// 接受接口作為參數
func MakeSpeak(s Speaker) {
    fmt.Println(s.Speak())
}

func MakeMove(m Mover) {
    fmt.Println(m.Move())
}

func main() {
    dog := Dog{Name: "旺財"}
    cat := Cat{Name: "小花"}
    robot := Robot{Model: "T-1000"}

    // 隱式實現接口
    MakeSpeak(dog)   // 汪汪汪
    MakeSpeak(cat)   // 喵喵喵
    MakeSpeak(robot) // 嗶嗶嗶

    MakeMove(dog) // 跑
    MakeMove(cat) // 跳
    // MakeMove(robot) // 編譯錯誤：Robot 沒有實現 Mover

    // 接口變量
    var s Speaker
    s = dog
    fmt.Println(s.Speak()) // 汪汪汪

    s = cat
    fmt.Println(s.Speak()) // 喵喵喵
}
```

**Node.js (TypeScript) 對比:**
```typescript
// TypeScript 接口
interface Speaker {
    speak(): string;
}

interface Mover {
    move(): string;
}

// 顯式實現接口
class Dog implements Speaker, Mover {
    name: string;

    constructor(name: string) {
        this.name = name;
    }

    speak(): string {
        return "汪汪汪";
    }

    move(): string {
        return "跑";
    }
}

class Cat implements Speaker, Mover {
    name: string;

    constructor(name: string) {
        this.name = name;
    }

    speak(): string {
        return "喵喵喵";
    }

    move(): string {
        return "跳";
    }
}

class Robot implements Speaker {
    model: string;

    constructor(model: string) {
        this.model = model;
    }

    speak(): string {
        return "嗶嗶嗶";
    }

    // Robot 沒有實現 move()
}

// 接受接口作為參數
function makeSpeak(s: Speaker): void {
    console.log(s.speak());
}

function makeMove(m: Mover): void {
    console.log(m.move());
}

const dog = new Dog("旺財");
const cat = new Cat("小花");
const robot = new Robot("T-1000");

makeSpeak(dog);   // 汪汪汪
makeSpeak(cat);   // 喵喵喵
makeSpeak(robot); // 嗶嗶嗶

makeMove(dog); // 跑
makeMove(cat); // 跳
// makeMove(robot); // TypeScript 編譯錯誤

// 接口變量
let s: Speaker;
s = dog;
console.log(s.speak()); // 汪汪汪

s = cat;
console.log(s.speak()); // 喵喵喵
```

**JavaScript (運行時鴨子類型):**
```javascript
// JavaScript 沒有編譯時類型檢查
// 但可以使用鴨子類型

function makeSpeak(s) {
    if (typeof s.speak === 'function') {
        console.log(s.speak());
    } else {
        throw new Error('對象沒有 speak 方法');
    }
}

function makeMove(m) {
    if (typeof m.move === 'function') {
        console.log(m.move());
    } else {
        throw new Error('對象沒有 move 方法');
    }
}

const dog = {
    name: "旺財",
    speak() { return "汪汪汪"; },
    move() { return "跑"; }
};

const cat = {
    name: "小花",
    speak() { return "喵喵喵"; },
    move() { return "跳"; }
};

const robot = {
    model: "T-1000",
    speak() { return "嗶嗶嗶"; }
    // 沒有 move 方法
};

makeSpeak(dog);   // 汪汪汪
makeSpeak(cat);   // 喵喵喵
makeSpeak(robot); // 嗶嗶嗶

makeMove(dog); // 跑
makeMove(cat); // 跳
// makeMove(robot); // 運行時錯誤
```

### 示例 2: 接口組合

**Go:**
```go
package main

import "fmt"

// 基本接口
type Reader interface {
    Read() string
}

type Writer interface {
    Write(string)
}

type Closer interface {
    Close() error
}

// 接口組合
type ReadWriter interface {
    Reader
    Writer
}

type ReadWriteCloser interface {
    Reader
    Writer
    Closer
}

// 實現
type File struct {
    content string
    closed  bool
}

func (f *File) Read() string {
    if f.closed {
        return ""
    }
    return f.content
}

func (f *File) Write(data string) {
    if !f.closed {
        f.content += data
    }
}

func (f *File) Close() error {
    f.closed = true
    fmt.Println("文件已關閉")
    return nil
}

// 使用接口
func ProcessReadWriter(rw ReadWriter) {
    data := rw.Read()
    fmt.Println("讀取:", data)
    rw.Write(" - 已處理")
}

func ProcessReadWriteCloser(rwc ReadWriteCloser) {
    data := rwc.Read()
    fmt.Println("讀取:", data)
    rwc.Write(" - 已處理")
    rwc.Close()
}

func main() {
    file := &File{content: "Hello"}

    // File 實現了所有接口
    var r Reader = file
    fmt.Println("Reader:", r.Read())

    var w Writer = file
    w.Write(" World")

    var rw ReadWriter = file
    fmt.Println("ReadWriter:", rw.Read())

    file2 := &File{content: "測試"}
    ProcessReadWriteCloser(file2)
}
```

**TypeScript 對比:**
```typescript
// 基本接口
interface Reader {
    read(): string;
}

interface Writer {
    write(data: string): void;
}

interface Closer {
    close(): void;
}

// 接口組合
interface ReadWriter extends Reader, Writer {}

interface ReadWriteCloser extends Reader, Writer, Closer {}

// 實現
class File implements ReadWriteCloser {
    private content: string = "";
    private closed: boolean = false;

    constructor(content: string) {
        this.content = content;
    }

    read(): string {
        if (this.closed) return "";
        return this.content;
    }

    write(data: string): void {
        if (!this.closed) {
            this.content += data;
        }
    }

    close(): void {
        this.closed = true;
        console.log("文件已關閉");
    }
}

function processReadWriter(rw: ReadWriter): void {
    const data = rw.read();
    console.log("讀取:", data);
    rw.write(" - 已處理");
}

function processReadWriteCloser(rwc: ReadWriteCloser): void {
    const data = rwc.read();
    console.log("讀取:", data);
    rwc.write(" - 已處理");
    rwc.close();
}

const file = new File("Hello");

const r: Reader = file;
console.log("Reader:", r.read());

const w: Writer = file;
w.write(" World");

const rw: ReadWriter = file;
console.log("ReadWriter:", rw.read());

const file2 = new File("測試");
processReadWriteCloser(file2);
```

### 示例 3: 空接口和類型斷言

**Go:**
```go
package main

import "fmt"

func main() {
    // 空接口可以持有任何類型的值
    var any interface{}

    any = 42
    fmt.Printf("整數: %v, 類型: %T\n", any, any)

    any = "hello"
    fmt.Printf("字符串: %v, 類型: %T\n", any, any)

    any = true
    fmt.Printf("布爾: %v, 類型: %T\n", any, any)

    any = []int{1, 2, 3}
    fmt.Printf("切片: %v, 類型: %T\n", any, any)

    // 類型斷言（Type Assertion）
    var i interface{} = "hello"

    // 安全的類型斷言
    s, ok := i.(string)
    if ok {
        fmt.Println("字符串:", s)
    }

    // 不安全的類型斷言（類型不匹配會 panic）
    // n := i.(int) // panic!

    n, ok := i.(int)
    if ok {
        fmt.Println("整數:", n)
    } else {
        fmt.Println("不是整數")
    }

    // 類型 switch
    describe := func(i interface{}) {
        switch v := i.(type) {
        case int:
            fmt.Printf("整數: %d\n", v)
        case string:
            fmt.Printf("字符串: %s (長度 %d)\n", v, len(v))
        case bool:
            fmt.Printf("布爾: %t\n", v)
        case []int:
            fmt.Printf("整數切片: %v (長度 %d)\n", v, len(v))
        default:
            fmt.Printf("未知類型: %T\n", v)
        }
    }

    describe(42)
    describe("hello")
    describe(true)
    describe([]int{1, 2, 3})
    describe(3.14)

    // 實際應用：處理不同類型
    values := []interface{}{
        42,
        "hello",
        true,
        3.14,
        []int{1, 2, 3},
    }

    for _, v := range values {
        describe(v)
    }
}
```

**TypeScript 對比:**
```typescript
// TypeScript 使用聯合類型和類型守衛

// any 類型（類似空接口）
let any: any;

any = 42;
console.log("整數:", any, "類型:", typeof any);

any = "hello";
console.log("字符串:", any, "類型:", typeof any);

any = true;
console.log("布爾:", any, "類型:", typeof any);

any = [1, 2, 3];
console.log("數組:", any, "類型:", typeof any);

// 類型守衛
let i: unknown = "hello";  // unknown 比 any 更安全

if (typeof i === "string") {
    console.log("字符串:", i);
}

if (typeof i === "number") {
    console.log("整數:", i);
} else {
    console.log("不是整數");
}

// 類型守衛函數
function isString(value: unknown): value is string {
    return typeof value === "string";
}

function isNumber(value: unknown): value is number {
    return typeof value === "number";
}

function describe(i: unknown): void {
    if (typeof i === "number") {
        console.log(`整數: ${i}`);
    } else if (typeof i === "string") {
        console.log(`字符串: ${i} (長度 ${i.length})`);
    } else if (typeof i === "boolean") {
        console.log(`布爾: ${i}`);
    } else if (Array.isArray(i)) {
        console.log(`數組: ${i} (長度 ${i.length})`);
    } else {
        console.log(`未知類型: ${typeof i}`);
    }
}

describe(42);
describe("hello");
describe(true);
describe([1, 2, 3]);
describe(3.14);

// 聯合類型（更類型安全）
type Value = number | string | boolean | number[];

const values: Value[] = [
    42,
    "hello",
    true,
    [1, 2, 3]
];

for (const v of values) {
    describe(v);
}
```

### 示例 4: 接口的實際應用 - 多態

**Go:**
```go
package main

import (
    "fmt"
    "math"
)

// 定義 Shape 接口
type Shape interface {
    Area() float64
    Perimeter() float64
}

// 圓形
type Circle struct {
    Radius float64
}

func (c Circle) Area() float64 {
    return math.Pi * c.Radius * c.Radius
}

func (c Circle) Perimeter() float64 {
    return 2 * math.Pi * c.Radius
}

// 矩形
type Rectangle struct {
    Width  float64
    Height float64
}

func (r Rectangle) Area() float64 {
    return r.Width * r.Height
}

func (r Rectangle) Perimeter() float64 {
    return 2 * (r.Width + r.Height)
}

// 三角形
type Triangle struct {
    A, B, C float64 // 三邊長
}

func (t Triangle) Area() float64 {
    // 使用海倫公式
    s := (t.A + t.B + t.C) / 2
    return math.Sqrt(s * (s - t.A) * (s - t.B) * (s - t.C))
}

func (t Triangle) Perimeter() float64 {
    return t.A + t.B + t.C
}

// 使用接口的函數
func PrintShapeInfo(s Shape) {
    fmt.Printf("類型: %T\n", s)
    fmt.Printf("面積: %.2f\n", s.Area())
    fmt.Printf("周長: %.2f\n", s.Perimeter())
    fmt.Println()
}

func TotalArea(shapes []Shape) float64 {
    total := 0.0
    for _, shape := range shapes {
        total += shape.Area()
    }
    return total
}

func main() {
    circle := Circle{Radius: 5}
    rect := Rectangle{Width: 4, Height: 6}
    triangle := Triangle{A: 3, B: 4, C: 5}

    // 多態：不同類型但都實現了 Shape 接口
    PrintShapeInfo(circle)
    PrintShapeInfo(rect)
    PrintShapeInfo(triangle)

    // 接口切片
    shapes := []Shape{
        Circle{Radius: 3},
        Rectangle{Width: 4, Height: 5},
        Triangle{A: 3, B: 4, C: 5},
        Circle{Radius: 7},
    }

    fmt.Printf("總面積: %.2f\n", TotalArea(shapes))

    // 遍歷並處理
    for i, shape := range shapes {
        fmt.Printf("形狀 %d - 面積: %.2f\n", i+1, shape.Area())
    }
}
```

**TypeScript 對比:**
```typescript
// 定義 Shape 接口
interface Shape {
    area(): number;
    perimeter(): number;
}

// 圓形
class Circle implements Shape {
    constructor(public radius: number) {}

    area(): number {
        return Math.PI * this.radius * this.radius;
    }

    perimeter(): number {
        return 2 * Math.PI * this.radius;
    }
}

// 矩形
class Rectangle implements Shape {
    constructor(public width: number, public height: number) {}

    area(): number {
        return this.width * this.height;
    }

    perimeter(): number {
        return 2 * (this.width + this.height);
    }
}

// 三角形
class Triangle implements Shape {
    constructor(public a: number, public b: number, public c: number) {}

    area(): number {
        const s = (this.a + this.b + this.c) / 2;
        return Math.sqrt(s * (s - this.a) * (s - this.b) * (s - this.c));
    }

    perimeter(): number {
        return this.a + this.b + this.c;
    }
}

// 使用接口的函數
function printShapeInfo(s: Shape): void {
    console.log("類型:", s.constructor.name);
    console.log("面積:", s.area().toFixed(2));
    console.log("周長:", s.perimeter().toFixed(2));
    console.log();
}

function totalArea(shapes: Shape[]): number {
    return shapes.reduce((total, shape) => total + shape.area(), 0);
}

const circle = new Circle(5);
const rect = new Rectangle(4, 6);
const triangle = new Triangle(3, 4, 5);

// 多態
printShapeInfo(circle);
printShapeInfo(rect);
printShapeInfo(triangle);

// 接口數組
const shapes: Shape[] = [
    new Circle(3),
    new Rectangle(4, 5),
    new Triangle(3, 4, 5),
    new Circle(7)
];

console.log("總面積:", totalArea(shapes).toFixed(2));

shapes.forEach((shape, i) => {
    console.log(`形狀 ${i + 1} - 面積: ${shape.area().toFixed(2)}`);
});
```

### 示例 5: 接口用於解耦和測試

**Go:**
```go
package main

import "fmt"

// 定義接口（抽象）
type Database interface {
    Save(data string) error
    Load(id string) (string, error)
}

// 真實實現
type MySQLDatabase struct {
    connection string
}

func (db MySQLDatabase) Save(data string) error {
    fmt.Printf("保存到 MySQL: %s\n", data)
    return nil
}

func (db MySQLDatabase) Load(id string) (string, error) {
    fmt.Printf("從 MySQL 加載: %s\n", id)
    return "MySQL 數據", nil
}

// Mock 實現（用於測試）
type MockDatabase struct {
    data map[string]string
}

func NewMockDatabase() *MockDatabase {
    return &MockDatabase{
        data: make(map[string]string),
    }
}

func (db *MockDatabase) Save(data string) error {
    db.data["test"] = data
    fmt.Printf("保存到 Mock: %s\n", data)
    return nil
}

func (db *MockDatabase) Load(id string) (string, error) {
    fmt.Printf("從 Mock 加載: %s\n", id)
    if data, ok := db.data[id]; ok {
        return data, nil
    }
    return "", fmt.Errorf("未找到")
}

// 業務邏輯依賴接口而非具體實現
type UserService struct {
    db Database  // 依賴接口
}

func NewUserService(db Database) *UserService {
    return &UserService{db: db}
}

func (s *UserService) CreateUser(name string) error {
    return s.db.Save(name)
}

func (s *UserService) GetUser(id string) (string, error) {
    return s.db.Load(id)
}

func main() {
    // 生產環境：使用真實數據庫
    prodDB := MySQLDatabase{connection: "prod"}
    prodService := NewUserService(prodDB)
    prodService.CreateUser("Alice")
    prodService.GetUser("123")

    fmt.Println()

    // 測試環境：使用 Mock
    mockDB := NewMockDatabase()
    testService := NewUserService(mockDB)
    testService.CreateUser("Bob")
    testService.GetUser("test")

    // 輕鬆切換實現，無需修改業務邏輯
}
```

**TypeScript 對比:**
```typescript
// 定義接口
interface Database {
    save(data: string): Promise<void>;
    load(id: string): Promise<string>;
}

// 真實實現
class MySQLDatabase implements Database {
    constructor(private connection: string) {}

    async save(data: string): Promise<void> {
        console.log(`保存到 MySQL: ${data}`);
    }

    async load(id: string): Promise<string> {
        console.log(`從 MySQL 加載: ${id}`);
        return "MySQL 數據";
    }
}

// Mock 實現
class MockDatabase implements Database {
    private data: Map<string, string> = new Map();

    async save(data: string): Promise<void> {
        this.data.set("test", data);
        console.log(`保存到 Mock: ${data}`);
    }

    async load(id: string): Promise<string> {
        console.log(`從 Mock 加載: ${id}`);
        const data = this.data.get(id);
        if (data) {
            return data;
        }
        throw new Error("未找到");
    }
}

// 業務邏輯依賴接口
class UserService {
    constructor(private db: Database) {}

    async createUser(name: string): Promise<void> {
        await this.db.save(name);
    }

    async getUser(id: string): Promise<string> {
        return await this.db.load(id);
    }
}

// 使用
async function main() {
    // 生產環境
    const prodDB = new MySQLDatabase("prod");
    const prodService = new UserService(prodDB);
    await prodService.createUser("Alice");
    await prodService.getUser("123");

    console.log();

    // 測試環境
    const mockDB = new MockDatabase();
    const testService = new UserService(mockDB);
    await testService.createUser("Bob");
    await testService.getUser("test");
}

main();
```

### 示例 6: 常用標準庫接口

**Go:**
```go
package main

import (
    "fmt"
    "io"
    "strings"
)

// 1. io.Reader 和 io.Writer
func DemoIOInterfaces() {
    fmt.Println("=== io.Reader / io.Writer ===")

    // strings.Reader 實現了 io.Reader
    reader := strings.NewReader("Hello, Go!")

    // 讀取數據
    buf := make([]byte, 5)
    n, err := reader.Read(buf)
    if err != nil && err != io.EOF {
        panic(err)
    }
    fmt.Printf("讀取了 %d 字節: %s\n", n, buf[:n])

    // strings.Builder 實現了 io.Writer
    var builder strings.Builder
    builder.WriteString("Hello")
    builder.WriteString(", ")
    builder.WriteString("World!")
    fmt.Println("Builder:", builder.String())
}

// 2. fmt.Stringer 接口
type Person struct {
    Name string
    Age  int
}

func (p Person) String() string {
    return fmt.Sprintf("%s (%d歲)", p.Name, p.Age)
}

func DemoStringer() {
    fmt.Println("\n=== fmt.Stringer ===")

    p := Person{Name: "Alice", Age: 30}
    fmt.Println("Person:", p)  // 自動調用 String()
    fmt.Printf("格式化: %v\n", p)
    fmt.Printf("詳細: %+v\n", p)
}

// 3. error 接口
type MyError struct {
    Code    int
    Message string
}

func (e MyError) Error() string {
    return fmt.Sprintf("錯誤 %d: %s", e.Code, e.Message)
}

func DemoError() {
    fmt.Println("\n=== error 接口 ===")

    err := MyError{Code: 404, Message: "未找到"}
    fmt.Println("錯誤:", err)

    // 可以作為 error 使用
    var e error = err
    fmt.Println("作為 error:", e)
}

// 4. sort.Interface
type ByLength []string

func (s ByLength) Len() int           { return len(s) }
func (s ByLength) Swap(i, j int)      { s[i], s[j] = s[j], s[i] }
func (s ByLength) Less(i, j int) bool { return len(s[i]) < len(s[j]) }

func main() {
    DemoIOInterfaces()
    DemoStringer()
    DemoError()
}
```

**Node.js 對比:**
```javascript
// Node.js 中的類似概念

// 1. Readable / Writable Stream
const { Readable, Writable } = require('stream');

function demoStreams() {
    console.log("=== Streams ===");

    // Readable Stream
    const readable = Readable.from(["Hello", ", ", "Go!"]);
    readable.on('data', (chunk) => {
        console.log("讀取:", chunk.toString());
    });

    // Writable Stream
    const writable = new Writable({
        write(chunk, encoding, callback) {
            console.log("寫入:", chunk.toString());
            callback();
        }
    });

    writable.write("Hello");
    writable.write(", ");
    writable.write("World!");
    writable.end();
}

// 2. toString() 方法
class Person {
    constructor(name, age) {
        this.name = name;
        this.age = age;
    }

    toString() {
        return `${this.name} (${this.age}歲)`;
    }

    // Symbol.toStringTag 自定義類型字符串
    get [Symbol.toStringTag]() {
        return 'Person';
    }
}

function demoToString() {
    console.log("\n=== toString ===");

    const p = new Person("Alice", 30);
    console.log("Person:", p.toString());
    console.log("自動:", String(p));
    console.log("類型:", Object.prototype.toString.call(p));
}

// 3. Error 類
class MyError extends Error {
    constructor(code, message) {
        super(message);
        this.code = code;
        this.name = 'MyError';
    }

    toString() {
        return `錯誤 ${this.code}: ${this.message}`;
    }
}

function demoError() {
    console.log("\n=== Error ===");

    const err = new MyError(404, "未找到");
    console.log("錯誤:", err.toString());
    console.log("作為 Error:", err.message);
}

demoStreams();
demoToString();
demoError();
```

## 重點總結

### 接口的關鍵概念

1. **隱式實現**：無需顯式聲明，只要實現了所有方法即可
2. **鴨子類型**："如果它走起來像鴨子，叫起來像鴨子，那它就是鴨子"
3. **多態**：同一接口的不同實現
4. **解耦**：依賴抽象而非具體實現

### Go vs TypeScript 接口對比

| 特性 | Go Interface | TypeScript Interface |
|------|--------------|---------------------|
| 實現方式 | 隱式（鴨子類型） | 顯式（implements） |
| 檢查時機 | 編譯時 + 運行時 | 編譯時（會被擦除） |
| 運行時存在 | 是 | 否（類型擦除） |
| 空接口 | `interface{}` | `any` / `unknown` |
| 類型斷言 | `v.(Type)` | `as Type` / 類型守衛 |

### 最佳實踐

1. **小接口**：接口應該小而專注（1-3個方法）
2. **接受接口，返回具體**：函數參數用接口，返回值用具體類型
3. **定義在使用處**：在需要的包中定義接口，而非實現處
4. **命名規範**：單方法接口通常以 -er 結尾（如 Reader, Writer）
5. **避免過度抽象**：不要為了接口而接口

### 常用標準庫接口

```go
// 輸入輸出
io.Reader      // Read([]byte) (int, error)
io.Writer      // Write([]byte) (int, error)
io.Closer      // Close() error

// 格式化
fmt.Stringer   // String() string

// 錯誤
error          // Error() string

// 排序
sort.Interface // Len(), Less(), Swap()
```

## 練習題

### 練習 1: 動物園管理
定義 `Animal` 接口，包含 `Speak()` 和 `Move()` 方法。實現至少三種動物，並編寫函數模擬動物園中動物的活動。

### 練習 2: 存儲系統
設計 `Storage` 接口，包含 `Set(key, value)` 和 `Get(key)` 方法。實現：
- 內存存儲
- 文件存儲（使用 map 模擬）
- 緩存存儲（帶過期時間）

### 練習 3: 序列化器
定義 `Serializer` 接口，包含 `Marshal()` 和 `Unmarshal()` 方法。實現：
- JSON 序列化器
- CSV 序列化器

### 練習 4: 日誌系統
設計 `Logger` 接口，支持不同日誌級別。實現：
- 控制台日誌
- 文件日誌
- 多目標日誌（同時寫入多個目標）

### 練習 5: 支付系統
定義 `PaymentMethod` 接口，實現：
- 信用卡支付
- PayPal 支付
- 加密貨幣支付

編寫 `PaymentProcessor` 處理不同支付方式。

### 答案提示

**練習 1:**
```go
type Animal interface {
    Speak() string
    Move() string
}

type Dog struct{ Name string }
func (d Dog) Speak() string { return "汪汪" }
func (d Dog) Move() string  { return "跑" }

type Bird struct{ Name string }
func (b Bird) Speak() string { return "啾啾" }
func (b Bird) Move() string  { return "飛" }

func SimulateZoo(animals []Animal) {
    for _, animal := range animals {
        fmt.Printf("%s: %s\n", animal.Speak(), animal.Move())
    }
}
```

**練習 2:**
```go
type Storage interface {
    Set(key, value string) error
    Get(key string) (string, error)
}

type MemoryStorage struct {
    data map[string]string
}

func (m *MemoryStorage) Set(key, value string) error {
    m.data[key] = value
    return nil
}

func (m *MemoryStorage) Get(key string) (string, error) {
    if val, ok := m.data[key]; ok {
        return val, nil
    }
    return "", fmt.Errorf("key not found")
}
```
