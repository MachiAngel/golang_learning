# Chapter 16: 結構體 (Structs)

## 概述

結構體（struct）是 Go 中用於組合多個字段的複合數據類型，類似於 JavaScript 中的對象或類。但與 JavaScript 不同，Go 的結構體是值類型，且是靜態類型的。結構體是 Go 中實現面向對象編程的基礎。

## Node.js 與 Go 的對比

| 特性 | Node.js | Go |
|------|---------|-----|
| 定義類型 | `class` 或對象字面量 | `type ... struct` |
| 創建實例 | `new Class()` 或 `{}` | `Type{}` 或 `&Type{}` |
| 字段訪問 | `obj.field` | `s.field` |
| 繼承 | `extends` | 嵌入（embedding） |
| 私有字段 | `#field` 或約定 | 小寫字母開頭 |
| 公開字段 | 默認公開 | 大寫字母開頭 |
| 構造函數 | `constructor()` | 構造函數（約定） |
| 方法 | `class` 方法 | 獨立定義的方法 |

## 詳細概念解釋

### 1. 結構體的特性

- **值類型**：賦值和傳參會複製整個結構體
- **靜態類型**：字段類型在編譯時確定
- **零值**：未初始化的字段自動設為零值
- **可導出性**：大寫字母開頭的字段可被外部包訪問

### 2. 結構體 vs 類

Go 沒有類的概念，但通過結構體 + 方法可以實現類似的功能：

```
JavaScript Class          Go Struct + Methods
┌─────────────────┐      ┌─────────────────┐
│  class Person   │      │  type Person    │
│  {              │      │  struct {       │
│    name         │  ≈   │    Name string  │
│    age          │      │    Age  int     │
│    greet()      │      │  }              │
│  }              │      │  func (p Person)│
└─────────────────┘      │  Greet() {...}  │
                         └─────────────────┘
```

### 3. 結構體標籤（Tags）

結構體標籤用於為字段添加元數據，常用於 JSON 序列化、數據庫映射等：

```go
type User struct {
    Name  string `json:"name" db:"user_name"`
    Email string `json:"email,omitempty"`
}
```

## 代碼示例

### 示例 1: 定義和創建結構體

**Go:**
```go
package main

import "fmt"

// 定義結構體
type Person struct {
    Name string
    Age  int
    City string
}

// 嵌套結構體
type Address struct {
    Street  string
    City    string
    ZipCode string
}

type Employee struct {
    Name    string
    Age     int
    Address Address
}

func main() {
    // 方法 1: 字面量初始化（指定字段名）
    p1 := Person{
        Name: "Alice",
        Age:  30,
        City: "New York",
    }
    fmt.Printf("p1: %+v\n", p1)

    // 方法 2: 字面量初始化（按順序，不推薦）
    p2 := Person{"Bob", 25, "London"}
    fmt.Printf("p2: %+v\n", p2)

    // 方法 3: 聲明後賦值
    var p3 Person
    p3.Name = "Charlie"
    p3.Age = 35
    p3.City = "Paris"
    fmt.Printf("p3: %+v\n", p3)

    // 方法 4: new 創建（返回指針）
    p4 := new(Person)
    p4.Name = "David"
    p4.Age = 40
    fmt.Printf("p4: %+v\n", p4)

    // 方法 5: 創建指針
    p5 := &Person{
        Name: "Eve",
        Age:  28,
    }
    fmt.Printf("p5: %+v\n", p5)

    // 零值
    var p6 Person
    fmt.Printf("零值: %+v\n", p6) // {Name: Age:0 City:}

    // 嵌套結構體
    emp := Employee{
        Name: "Frank",
        Age:  32,
        Address: Address{
            Street:  "123 Main St",
            City:    "Boston",
            ZipCode: "02101",
        },
    }
    fmt.Printf("員工: %+v\n", emp)
    fmt.Println("城市:", emp.Address.City)
}
```

**Node.js 對比:**
```javascript
// 方法 1: 使用 class
class Person {
    constructor(name, age, city) {
        this.name = name;
        this.age = age;
        this.city = city;
    }
}

const p1 = new Person("Alice", 30, "New York");
console.log("p1:", p1);

// 方法 2: 對象字面量
const p2 = {
    name: "Bob",
    age: 25,
    city: "London"
};
console.log("p2:", p2);

// 方法 3: 逐步賦值
const p3 = {};
p3.name = "Charlie";
p3.age = 35;
p3.city = "Paris";
console.log("p3:", p3);

// 方法 4: Object.create
const personProto = {
    greet() {
        console.log(`Hello, I'm ${this.name}`);
    }
};
const p4 = Object.create(personProto);
p4.name = "David";
p4.age = 40;
console.log("p4:", p4);

// 嵌套對象
const emp = {
    name: "Frank",
    age: 32,
    address: {
        street: "123 Main St",
        city: "Boston",
        zipCode: "02101"
    }
};
console.log("員工:", emp);
console.log("城市:", emp.address.city);

// TypeScript 接口（類型定義）
/*
interface Person {
    name: string;
    age: number;
    city: string;
}

interface Address {
    street: string;
    city: string;
    zipCode: string;
}

interface Employee {
    name: string;
    age: number;
    address: Address;
}
*/
```

### 示例 2: 結構體是值類型

**Go:**
```go
package main

import "fmt"

type Point struct {
    X, Y int
}

func main() {
    // 結構體是值類型
    p1 := Point{X: 1, Y: 2}
    p2 := p1  // 複製
    p2.X = 100

    fmt.Println("p1:", p1) // {1 2} - 未改變
    fmt.Println("p2:", p2) // {100 2}

    // 函數傳參也會複製
    modifyPoint(p1)
    fmt.Println("函數調用後 p1:", p1) // {1 2} - 未改變

    // 使用指針可以修改原值
    modifyPointPtr(&p1)
    fmt.Println("指針修改後 p1:", p1) // {999 2}

    // 指針賦值不會複製
    p3 := &Point{X: 3, Y: 4}
    p4 := p3
    p4.X = 200

    fmt.Println("p3:", p3) // &{200 4} - 被修改
    fmt.Println("p4:", p4) // &{200 4}
}

func modifyPoint(p Point) {
    p.X = 888
    fmt.Println("函數內部:", p) // {888 2}
}

func modifyPointPtr(p *Point) {
    p.X = 999
}
```

**Node.js 對比:**
```javascript
// JavaScript 對象是引用類型
const p1 = { x: 1, y: 2 };
const p2 = p1;  // 引用，不是複製
p2.x = 100;

console.log("p1:", p1); // { x: 100, y: 2 } - 被修改了
console.log("p2:", p2); // { x: 100, y: 2 }

// 函數傳參也是引用
function modifyPoint(p) {
    p.x = 888;
    console.log("函數內部:", p);
}

modifyPoint(p1);
console.log("函數調用後 p1:", p1); // { x: 888, y: 2 } - 被修改

// 如果需要複製，需要手動操作
const p3 = { x: 3, y: 4 };
const p4 = { ...p3 };  // 淺複製
p4.x = 200;

console.log("p3:", p3); // { x: 3, y: 4 } - 未改變
console.log("p4:", p4); // { x: 200, y: 4 }

// 深複製（針對嵌套對象）
const obj1 = { a: 1, b: { c: 2 } };
const obj2 = JSON.parse(JSON.stringify(obj1));
obj2.b.c = 999;

console.log("obj1:", obj1); // { a: 1, b: { c: 2 } }
console.log("obj2:", obj2); // { a: 1, b: { c: 999 } }
```

### 示例 3: 匿名結構體和嵌入

**Go:**
```go
package main

import "fmt"

type Address struct {
    City    string
    Country string
}

type Person struct {
    Name string
    Age  int
    Address  // 嵌入（匿名字段）
}

func main() {
    // 1. 匿名結構體
    point := struct {
        X, Y int
    }{
        X: 10,
        Y: 20,
    }
    fmt.Printf("匿名結構體: %+v\n", point)

    // 常用於測試或一次性使用
    user := struct {
        Name  string
        Email string
    }{
        Name:  "Alice",
        Email: "alice@example.com",
    }
    fmt.Printf("用戶: %+v\n", user)

    // 2. 結構體嵌入（類似繼承）
    p := Person{
        Name: "Bob",
        Age:  30,
        Address: Address{
            City:    "New York",
            Country: "USA",
        },
    }

    // 可以直接訪問嵌入字段的字段
    fmt.Println("姓名:", p.Name)
    fmt.Println("城市:", p.City)    // 直接訪問
    fmt.Println("國家:", p.Country) // 直接訪問

    // 也可以通過類型名訪問
    fmt.Println("地址:", p.Address.City)

    // 3. 多重嵌入
    type Contact struct {
        Email string
        Phone string
    }

    type Employee struct {
        Person  // 嵌入 Person
        Contact // 嵌入 Contact
        Salary  int
    }

    emp := Employee{
        Person: Person{
            Name: "Charlie",
            Age:  35,
            Address: Address{
                City:    "London",
                Country: "UK",
            },
        },
        Contact: Contact{
            Email: "charlie@company.com",
            Phone: "123-456-7890",
        },
        Salary: 80000,
    }

    fmt.Printf("\n員工: %+v\n", emp)
    fmt.Println("姓名:", emp.Name)   // 來自 Person
    fmt.Println("郵箱:", emp.Email)  // 來自 Contact
    fmt.Println("城市:", emp.City)   // 來自 Person.Address
    fmt.Println("薪資:", emp.Salary)
}
```

**Node.js 對比:**
```javascript
// JavaScript 沒有直接的匿名類概念，但有匿名對象
const point = {
    x: 10,
    y: 20
};
console.log("匿名對象:", point);

const user = {
    name: "Alice",
    email: "alice@example.com"
};
console.log("用戶:", user);

// 類繼承（使用 extends）
class Address {
    constructor(city, country) {
        this.city = city;
        this.country = country;
    }
}

class Person {
    constructor(name, age, city, country) {
        this.name = name;
        this.age = age;
        this.address = new Address(city, country);
    }
}

const p = new Person("Bob", 30, "New York", "USA");
console.log("姓名:", p.name);
console.log("城市:", p.address.city);
console.log("國家:", p.address.country);

// 使用對象組合
const person = {
    name: "Bob",
    age: 30,
    ...{ city: "New York", country: "USA" }
};
console.log("組合對象:", person);

// 多重繼承（JavaScript 只支持單繼承，但可以用 mixin）
class Contact {
    constructor(email, phone) {
        this.email = email;
        this.phone = phone;
    }
}

// Mixin 模式
function mixin(target, ...sources) {
    Object.assign(target.prototype, ...sources);
}

class Employee extends Person {
    constructor(name, age, city, country, email, phone, salary) {
        super(name, age, city, country);
        this.email = email;
        this.phone = phone;
        this.salary = salary;
    }
}

const emp = new Employee(
    "Charlie", 35, "London", "UK",
    "charlie@company.com", "123-456-7890", 80000
);

console.log("\n員工:", emp);
console.log("姓名:", emp.name);
console.log("郵箱:", emp.email);
console.log("城市:", emp.address.city);
console.log("薪資:", emp.salary);

// 或使用對象組合
const employee = {
    name: "Charlie",
    age: 35,
    address: {
        city: "London",
        country: "UK"
    },
    contact: {
        email: "charlie@company.com",
        phone: "123-456-7890"
    },
    salary: 80000
};
```

### 示例 4: 結構體標籤（Tags）

**Go:**
```go
package main

import (
    "encoding/json"
    "fmt"
)

// 結構體標籤用於 JSON 序列化
type User struct {
    ID       int    `json:"id"`
    Name     string `json:"name"`
    Email    string `json:"email,omitempty"`     // omitempty: 零值時省略
    Password string `json:"-"`                    // -: 不序列化
    Age      int    `json:"age,omitempty"`
    IsActive bool   `json:"is_active"`
}

// 多種標籤組合
type Product struct {
    ID          int     `json:"id" db:"product_id" xml:"id"`
    Name        string  `json:"name" db:"product_name" xml:"name"`
    Price       float64 `json:"price" db:"price" xml:"price"`
    Description string  `json:"description,omitempty" db:"description"`
}

func main() {
    // 創建用戶
    user := User{
        ID:       1,
        Name:     "Alice",
        Email:    "alice@example.com",
        Password: "secret123",
        Age:      30,
        IsActive: true,
    }

    // 序列化為 JSON
    jsonData, err := json.Marshal(user)
    if err != nil {
        fmt.Println("錯誤:", err)
        return
    }
    fmt.Println("JSON:", string(jsonData))
    // {"id":1,"name":"Alice","email":"alice@example.com","age":30,"is_active":true}
    // 注意：Password 沒有被序列化

    // 零值字段測試
    user2 := User{
        ID:       2,
        Name:     "Bob",
        IsActive: false,
    }
    jsonData2, _ := json.Marshal(user2)
    fmt.Println("JSON2:", string(jsonData2))
    // {"id":2,"name":"Bob","is_active":false}
    // 注意：Email 和 Age 因為是零值且有 omitempty，所以被省略

    // 從 JSON 反序列化
    jsonStr := `{"id":3,"name":"Charlie","email":"charlie@example.com","age":25,"is_active":true}`
    var user3 User
    err = json.Unmarshal([]byte(jsonStr), &user3)
    if err != nil {
        fmt.Println("錯誤:", err)
        return
    }
    fmt.Printf("反序列化: %+v\n", user3)

    // 美化輸出
    prettyJSON, _ := json.MarshalIndent(user, "", "  ")
    fmt.Println("\n美化 JSON:")
    fmt.Println(string(prettyJSON))
}
```

**Node.js 對比:**
```javascript
// JavaScript 對象可以直接序列化為 JSON
const user = {
    id: 1,
    name: "Alice",
    email: "alice@example.com",
    password: "secret123",
    age: 30,
    isActive: true
};

// 序列化為 JSON
const jsonData = JSON.stringify(user);
console.log("JSON:", jsonData);

// 如果要排除某些字段，需要手動處理
const userWithoutPassword = { ...user };
delete userWithoutPassword.password;
const jsonDataNoPassword = JSON.stringify(userWithoutPassword);
console.log("無密碼 JSON:", jsonDataNoPassword);

// 或使用 replacer 函數
const jsonData2 = JSON.stringify(user, (key, value) => {
    if (key === 'password') return undefined;
    return value;
});
console.log("使用 replacer:", jsonData2);

// 美化輸出
const prettyJSON = JSON.stringify(user, null, 2);
console.log("\n美化 JSON:");
console.log(prettyJSON);

// 從 JSON 反序列化
const jsonStr = '{"id":3,"name":"Charlie","email":"charlie@example.com","age":25,"isActive":true}';
const user3 = JSON.parse(jsonStr);
console.log("反序列化:", user3);

// 處理零值字段（需要手動）
const user2 = {
    id: 2,
    name: "Bob",
    isActive: false
};

// 移除零值字段
const cleanUser = Object.fromEntries(
    Object.entries(user2).filter(([_, v]) => v !== '' && v !== 0 && v !== null && v !== undefined)
);
console.log("清理後:", JSON.stringify(cleanUser));

// TypeScript 中的裝飾器可以實現類似標籤的功能
/*
import { Exclude, Expose } from 'class-transformer';

class User {
    @Expose() id: number;
    @Expose() name: string;
    @Expose() email?: string;
    @Exclude() password: string;
    @Expose() age?: number;
    @Expose({ name: 'is_active' }) isActive: boolean;
}
*/
```

### 示例 5: 構造函數模式

**Go:**
```go
package main

import (
    "fmt"
    "time"
)

type User struct {
    ID        int
    Name      string
    Email     string
    CreatedAt time.Time
}

// 構造函數（約定以 New 開頭）
func NewUser(name, email string) *User {
    return &User{
        ID:        generateID(),
        Name:      name,
        Email:     email,
        CreatedAt: time.Now(),
    }
}

// 帶驗證的構造函數
func NewUserWithValidation(name, email string) (*User, error) {
    if name == "" {
        return nil, fmt.Errorf("name cannot be empty")
    }
    if email == "" {
        return nil, fmt.Errorf("email cannot be empty")
    }

    return &User{
        ID:        generateID(),
        Name:      name,
        Email:     email,
        CreatedAt: time.Now(),
    }, nil
}

// 使用選項模式（Options Pattern）
type UserOption func(*User)

func WithEmail(email string) UserOption {
    return func(u *User) {
        u.Email = email
    }
}

func WithCreatedAt(t time.Time) UserOption {
    return func(u *User) {
        u.CreatedAt = t
    }
}

func NewUserWithOptions(name string, opts ...UserOption) *User {
    u := &User{
        ID:        generateID(),
        Name:      name,
        CreatedAt: time.Now(),
    }

    for _, opt := range opts {
        opt(u)
    }

    return u
}

var idCounter = 0

func generateID() int {
    idCounter++
    return idCounter
}

func main() {
    // 使用基本構造函數
    user1 := NewUser("Alice", "alice@example.com")
    fmt.Printf("user1: %+v\n", user1)

    // 使用帶驗證的構造函數
    user2, err := NewUserWithValidation("Bob", "bob@example.com")
    if err != nil {
        fmt.Println("錯誤:", err)
        return
    }
    fmt.Printf("user2: %+v\n", user2)

    // 驗證失敗的例子
    _, err = NewUserWithValidation("", "test@example.com")
    if err != nil {
        fmt.Println("驗證錯誤:", err)
    }

    // 使用選項模式
    user3 := NewUserWithOptions(
        "Charlie",
        WithEmail("charlie@example.com"),
        WithCreatedAt(time.Date(2024, 1, 1, 0, 0, 0, 0, time.UTC)),
    )
    fmt.Printf("user3: %+v\n", user3)

    // 只設置名字
    user4 := NewUserWithOptions("David")
    fmt.Printf("user4: %+v\n", user4)
}
```

**Node.js 對比:**
```javascript
// 使用 class 構造函數
class User {
    constructor(name, email) {
        this.id = User.generateID();
        this.name = name;
        this.email = email;
        this.createdAt = new Date();
    }

    static idCounter = 0;

    static generateID() {
        return ++User.idCounter;
    }

    // 靜態工廠方法
    static create(name, email) {
        return new User(name, email);
    }

    // 帶驗證的工廠方法
    static createWithValidation(name, email) {
        if (!name) {
            throw new Error("name cannot be empty");
        }
        if (!email) {
            throw new Error("email cannot be empty");
        }
        return new User(name, email);
    }
}

// 使用構造函數
const user1 = new User("Alice", "alice@example.com");
console.log("user1:", user1);

// 使用靜態工廠方法
const user2 = User.create("Bob", "bob@example.com");
console.log("user2:", user2);

// 帶驗證
try {
    const user3 = User.createWithValidation("", "test@example.com");
} catch (error) {
    console.log("驗證錯誤:", error.message);
}

// 選項模式（使用對象參數）
class UserWithOptions {
    constructor(name, options = {}) {
        this.id = UserWithOptions.generateID();
        this.name = name;
        this.email = options.email || '';
        this.createdAt = options.createdAt || new Date();
    }

    static idCounter = 0;

    static generateID() {
        return ++UserWithOptions.idCounter;
    }
}

const user4 = new UserWithOptions("Charlie", {
    email: "charlie@example.com",
    createdAt: new Date("2024-01-01")
});
console.log("user4:", user4);

// 只設置名字
const user5 = new UserWithOptions("David");
console.log("user5:", user5);

// 建造者模式
class UserBuilder {
    constructor(name) {
        this.user = {
            id: ++UserBuilder.idCounter,
            name: name,
            createdAt: new Date()
        };
    }

    static idCounter = 0;

    withEmail(email) {
        this.user.email = email;
        return this;
    }

    withCreatedAt(date) {
        this.user.createdAt = date;
        return this;
    }

    build() {
        return this.user;
    }
}

const user6 = new UserBuilder("Eve")
    .withEmail("eve@example.com")
    .withCreatedAt(new Date("2024-01-01"))
    .build();
console.log("user6:", user6);
```

### 示例 6: 結構體比較

**Go:**
```go
package main

import "fmt"

type Point struct {
    X, Y int
}

type Person struct {
    Name string
    Age  int
}

type Container struct {
    Data []int  // 包含不可比較類型
}

func main() {
    // 1. 可比較的結構體
    p1 := Point{X: 1, Y: 2}
    p2 := Point{X: 1, Y: 2}
    p3 := Point{X: 3, Y: 4}

    fmt.Println("p1 == p2:", p1 == p2) // true
    fmt.Println("p1 == p3:", p1 == p3) // false

    // 2. 結構體比較的是所有字段
    person1 := Person{Name: "Alice", Age: 30}
    person2 := Person{Name: "Alice", Age: 30}
    person3 := Person{Name: "Bob", Age: 30}

    fmt.Println("person1 == person2:", person1 == person2) // true
    fmt.Println("person1 == person3:", person1 == person3) // false

    // 3. 包含不可比較字段的結構體不能比較
    // c1 := Container{Data: []int{1, 2, 3}}
    // c2 := Container{Data: []int{1, 2, 3}}
    // fmt.Println(c1 == c2)  // 編譯錯誤：invalid operation

    // 4. 指針比較的是地址
    pp1 := &Point{X: 1, Y: 2}
    pp2 := &Point{X: 1, Y: 2}
    pp3 := pp1

    fmt.Println("pp1 == pp2:", pp1 == pp2) // false (不同地址)
    fmt.Println("pp1 == pp3:", pp1 == pp3) // true (相同地址)
    fmt.Println("*pp1 == *pp2:", *pp1 == *pp2) // true (值相同)

    // 5. 自定義比較函數
    c1 := Container{Data: []int{1, 2, 3}}
    c2 := Container{Data: []int{1, 2, 3}}
    fmt.Println("容器相等:", equalContainers(c1, c2)) // true
}

func equalContainers(c1, c2 Container) bool {
    if len(c1.Data) != len(c2.Data) {
        return false
    }
    for i := range c1.Data {
        if c1.Data[i] != c2.Data[i] {
            return false
        }
    }
    return true
}
```

**Node.js 對比:**
```javascript
// JavaScript 對象比較的是引用，不是值
const p1 = { x: 1, y: 2 };
const p2 = { x: 1, y: 2 };
const p3 = p1;

console.log("p1 === p2:", p1 === p2); // false (不同引用)
console.log("p1 === p3:", p1 === p3); // true (相同引用)

// 深度比較需要自定義或使用庫
function deepEqual(obj1, obj2) {
    return JSON.stringify(obj1) === JSON.stringify(obj2);
}

console.log("深度相等:", deepEqual(p1, p2)); // true

// 使用 lodash
// const _ = require('lodash');
// console.log(_.isEqual(p1, p2)); // true

// 數組比較
const arr1 = [1, 2, 3];
const arr2 = [1, 2, 3];
console.log("arr1 === arr2:", arr1 === arr2); // false

// 需要手動比較
function arrayEqual(a1, a2) {
    if (a1.length !== a2.length) return false;
    return a1.every((val, idx) => val === a2[idx]);
}

console.log("數組相等:", arrayEqual(arr1, arr2)); // true

// 包含數組的對象
const c1 = { data: [1, 2, 3] };
const c2 = { data: [1, 2, 3] };

console.log("深度相等 (包含數組):", deepEqual(c1, c2)); // true
```

## 重點總結

### 結構體的關鍵特性

1. **值類型**：賦值和傳參會複製
2. **靜態類型**：字段類型在編譯時確定
3. **零值初始化**：未初始化的字段自動為零值
4. **可導出性**：大寫開頭的字段可被外部訪問

### 結構體 vs JavaScript 對象/類

| 特性 | Go Struct | JS Object | JS Class |
|------|-----------|-----------|----------|
| 類型 | 值類型 | 引用類型 | 引用類型 |
| 定義 | `type T struct{}` | `{}` | `class T {}` |
| 繼承 | 嵌入 | 原型鏈 | `extends` |
| 方法 | 外部定義 | 對象內 | 類內定義 |
| 私有性 | 小寫字母 | `#` 或約定 | `#` 或約定 |

### 最佳實踐

1. **使用構造函數**：用 `NewXxx` 函數初始化複雜結構體
2. **返回指針**：大結構體使用指針避免複製
3. **字段可導出性**：合理控制字段的訪問權限
4. **使用標籤**：JSON、數據庫等場景充分利用標籤
5. **優先組合**：使用嵌入實現組合優於繼承

### 常用模式

```go
// 定義結構體
type User struct {
    ID   int
    Name string
}

// 構造函數
func NewUser(name string) *User {
    return &User{
        ID:   generateID(),
        Name: name,
    }
}

// 選項模式
type Option func(*User)

func WithID(id int) Option {
    return func(u *User) { u.ID = id }
}

// 方法將在下一章詳細介紹
```

## 練習題

### 練習 1: 圖書管理
定義 `Book` 結構體，包含書名、作者、ISBN、價格字段。實現以下功能：
1. 構造函數 `NewBook`
2. JSON 序列化（使用合適的標籤）
3. 字符串表示方法

### 練習 2: 矩形操作
定義 `Rectangle` 結構體，包含寬度和高度。實現：
1. 計算面積的函數
2. 計算周長的函數
3. 判斷兩個矩形是否相等的函數

### 練習 3: 學生成績
定義以下結構體：
- `Course`：課程名、學分
- `Grade`：課程、分數
- `Student`：姓名、學號、成績列表

實現計算學生 GPA 的函數。

### 練習 4: 員工層級
定義 `Employee` 結構體，包含姓名、職位、薪資。使用嵌入實現 `Manager` 結構體，添加管理的員工列表。

### 練習 5: 配置管理
設計一個配置系統，使用結構體標籤支持從 JSON 文件加載配置，實現默認值處理。

### 答案提示

**練習 1:**
```go
type Book struct {
    Title  string  `json:"title"`
    Author string  `json:"author"`
    ISBN   string  `json:"isbn"`
    Price  float64 `json:"price"`
}

func NewBook(title, author, isbn string, price float64) *Book {
    return &Book{
        Title:  title,
        Author: author,
        ISBN:   isbn,
        Price:  price,
    }
}

func (b Book) String() string {
    return fmt.Sprintf("%s by %s (ISBN: %s) - $%.2f",
        b.Title, b.Author, b.ISBN, b.Price)
}
```

**練習 2:**
```go
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

func (r Rectangle) Equal(other Rectangle) bool {
    return r.Width == other.Width && r.Height == other.Height
}
```
