# Chapter 30: json 包 - JSON 處理

## 概述

`encoding/json` 包是 Go 標準庫中處理 JSON 數據的核心包。對於 Node.js 開發者來說，它相當於內置的 `JSON.parse()` 和 `JSON.stringify()`，但提供了更多的類型安全和定制選項。

## Node.js 與 Go 的對比

| 功能 | Node.js | Go |
|------|---------|-----|
| 對象轉 JSON | `JSON.stringify(obj)` | `json.Marshal(obj)` |
| JSON 轉對象 | `JSON.parse(str)` | `json.Unmarshal(data, &obj)` |
| 格式化輸出 | `JSON.stringify(obj, null, 2)` | `json.MarshalIndent(obj, "", "  ")` |
| 自定義序列化 | `toJSON()` 方法 | 實現 `MarshalJSON()` 方法 |
| 自定義反序列化 | `JSON.parse(str, reviver)` | 實現 `UnmarshalJSON()` 方法 |
| 流式處理 | 手動實現 | `json.Encoder`/`json.Decoder` |

## 詳細概念解釋

### 1. Marshal（序列化）

將 Go 數據結構轉換為 JSON 字節數組：
- `json.Marshal(v any) ([]byte, error)` - 緊湊格式
- `json.MarshalIndent(v any, prefix, indent string) ([]byte, error)` - 格式化輸出

### 2. Unmarshal（反序列化）

將 JSON 字節數組轉換為 Go 數據結構：
- `json.Unmarshal(data []byte, v any) error`

### 3. 結構體標籤（Struct Tags）

使用結構體標籤控制 JSON 的序列化行為：
```go
type User struct {
    Name     string `json:"name"`               // 映射到 "name"
    Age      int    `json:"age,omitempty"`      // 為空時忽略
    Password string `json:"-"`                  // 忽略此字段
    Email    string `json:"email,omitempty"`    // 為空時忽略
}
```

常用標籤選項：
- `json:"fieldname"` - 自定義 JSON 字段名
- `json:",omitempty"` - 字段為零值時忽略
- `json:"-"` - 完全忽略此字段
- `json:",string"` - 將數字或布爾值作為字符串

### 4. Encoder/Decoder

用於流式處理 JSON：
- `json.NewEncoder(w io.Writer)` - 創建編碼器
- `json.NewDecoder(r io.Reader)` - 創建解碼器

## 實際代碼示例

### 示例 1: 基本 Marshal 和 Unmarshal

**Go:**
```go
package main

import (
    "encoding/json"
    "fmt"
)

type Person struct {
    Name string
    Age  int
    City string
}

func main() {
    // Marshal - Go 對象轉 JSON
    person := Person{
        Name: "Alice",
        Age:  30,
        City: "Tokyo",
    }

    jsonData, err := json.Marshal(person)
    if err != nil {
        fmt.Println("Marshal error:", err)
        return
    }

    fmt.Println("JSON:", string(jsonData))
    // 輸出: {"Name":"Alice","Age":30,"City":"Tokyo"}

    // Unmarshal - JSON 轉 Go 對象
    jsonString := `{"Name":"Bob","Age":25,"City":"New York"}`
    var person2 Person

    err = json.Unmarshal([]byte(jsonString), &person2)
    if err != nil {
        fmt.Println("Unmarshal error:", err)
        return
    }

    fmt.Printf("Person: %+v\n", person2)
    // 輸出: Person: {Name:Bob Age:25 City:New York}
}
```

**Node.js:**
```javascript
// 對象轉 JSON
const person = {
    name: "Alice",
    age: 30,
    city: "Tokyo"
};

const jsonString = JSON.stringify(person);
console.log("JSON:", jsonString);
// 輸出: {"name":"Alice","age":30,"city":"Tokyo"}

// JSON 轉對象
const jsonData = '{"name":"Bob","age":25,"city":"New York"}';
const person2 = JSON.parse(jsonData);

console.log("Person:", person2);
// 輸出: Person: { name: 'Bob', age: 25, city: 'New York' }
```

### 示例 2: 結構體標籤

**Go:**
```go
package main

import (
    "encoding/json"
    "fmt"
)

type User struct {
    ID       int    `json:"id"`
    Username string `json:"username"`
    Email    string `json:"email,omitempty"`    // 為空時忽略
    Password string `json:"-"`                  // 永不序列化
    Age      int    `json:"age,omitempty"`      // 為 0 時忽略
    IsActive bool   `json:"is_active"`
}

func main() {
    user1 := User{
        ID:       1,
        Username: "alice",
        Email:    "alice@example.com",
        Password: "secret123",  // 不會被序列化
        Age:      30,
        IsActive: true,
    }

    jsonData1, _ := json.MarshalIndent(user1, "", "  ")
    fmt.Println("User with all fields:")
    fmt.Println(string(jsonData1))

    // 測試 omitempty
    user2 := User{
        ID:       2,
        Username: "bob",
        Password: "secret456",
        IsActive: false,
        // Email 和 Age 為空
    }

    jsonData2, _ := json.MarshalIndent(user2, "", "  ")
    fmt.Println("\nUser with omitted fields:")
    fmt.Println(string(jsonData2))
}
```

輸出：
```json
User with all fields:
{
  "id": 1,
  "username": "alice",
  "email": "alice@example.com",
  "age": 30,
  "is_active": true
}

User with omitted fields:
{
  "id": 2,
  "username": "bob",
  "is_active": false
}
```

**Node.js:**
```javascript
class User {
    constructor(id, username, email, password, age, isActive) {
        this.id = id;
        this.username = username;
        this.email = email;
        this.password = password;
        this.age = age;
        this.isActive = isActive;
    }

    // 自定義序列化
    toJSON() {
        const obj = {
            id: this.id,
            username: this.username,
            is_active: this.isActive
        };

        // 只在有值時包含
        if (this.email) obj.email = this.email;
        if (this.age) obj.age = this.age;
        // password 不包含

        return obj;
    }
}

const user1 = new User(1, "alice", "alice@example.com", "secret123", 30, true);
console.log("User with all fields:");
console.log(JSON.stringify(user1, null, 2));

const user2 = new User(2, "bob", "", "secret456", 0, false);
console.log("\nUser with omitted fields:");
console.log(JSON.stringify(user2, null, 2));
```

### 示例 3: 嵌套結構和數組

**Go:**
```go
package main

import (
    "encoding/json"
    "fmt"
)

type Address struct {
    Street  string `json:"street"`
    City    string `json:"city"`
    Country string `json:"country"`
}

type Person struct {
    Name      string   `json:"name"`
    Age       int      `json:"age"`
    Address   Address  `json:"address"`
    Hobbies   []string `json:"hobbies"`
    Languages []string `json:"languages,omitempty"`
}

func main() {
    person := Person{
        Name: "Alice",
        Age:  30,
        Address: Address{
            Street:  "123 Main St",
            City:    "Tokyo",
            Country: "Japan",
        },
        Hobbies: []string{"reading", "coding", "traveling"},
    }

    // Marshal 嵌套結構
    jsonData, _ := json.MarshalIndent(person, "", "  ")
    fmt.Println("JSON:")
    fmt.Println(string(jsonData))

    // Unmarshal 嵌套結構
    jsonString := `{
        "name": "Bob",
        "age": 25,
        "address": {
            "street": "456 Oak Ave",
            "city": "New York",
            "country": "USA"
        },
        "hobbies": ["gaming", "music"]
    }`

    var person2 Person
    json.Unmarshal([]byte(jsonString), &person2)
    fmt.Printf("\nParsed person: %+v\n", person2)
    fmt.Printf("Address: %+v\n", person2.Address)
}
```

**Node.js:**
```javascript
const person = {
    name: "Alice",
    age: 30,
    address: {
        street: "123 Main St",
        city: "Tokyo",
        country: "Japan"
    },
    hobbies: ["reading", "coding", "traveling"]
};

// Stringify 嵌套結構
const jsonString = JSON.stringify(person, null, 2);
console.log("JSON:");
console.log(jsonString);

// Parse 嵌套結構
const jsonData = `{
    "name": "Bob",
    "age": 25,
    "address": {
        "street": "456 Oak Ave",
        "city": "New York",
        "country": "USA"
    },
    "hobbies": ["gaming", "music"]
}`;

const person2 = JSON.parse(jsonData);
console.log("\nParsed person:", person2);
console.log("Address:", person2.address);
```

### 示例 4: 處理 map 和 interface{}

**Go:**
```go
package main

import (
    "encoding/json"
    "fmt"
)

func main() {
    // Map 轉 JSON
    data := map[string]interface{}{
        "name":   "Alice",
        "age":    30,
        "active": true,
        "scores": []int{85, 90, 95},
    }

    jsonData, _ := json.MarshalIndent(data, "", "  ")
    fmt.Println("Map to JSON:")
    fmt.Println(string(jsonData))

    // JSON 轉 map
    jsonString := `{
        "product": "Laptop",
        "price": 999.99,
        "inStock": true,
        "tags": ["electronics", "computers"]
    }`

    var result map[string]interface{}
    json.Unmarshal([]byte(jsonString), &result)

    fmt.Println("\nJSON to map:")
    fmt.Printf("%+v\n", result)

    // 訪問嵌套數據（需要類型斷言）
    if tags, ok := result["tags"].([]interface{}); ok {
        fmt.Println("Tags:")
        for _, tag := range tags {
            fmt.Println("-", tag)
        }
    }

    // 類型斷言示例
    if price, ok := result["price"].(float64); ok {
        fmt.Printf("Price: $%.2f\n", price)
    }
}
```

**Node.js:**
```javascript
// 對象轉 JSON
const data = {
    name: "Alice",
    age: 30,
    active: true,
    scores: [85, 90, 95]
};

const jsonString = JSON.stringify(data, null, 2);
console.log("Object to JSON:");
console.log(jsonString);

// JSON 轉對象
const jsonData = `{
    "product": "Laptop",
    "price": 999.99,
    "inStock": true,
    "tags": ["electronics", "computers"]
}`;

const result = JSON.parse(jsonData);

console.log("\nJSON to object:");
console.log(result);

// 訪問嵌套數據（直接訪問）
if (Array.isArray(result.tags)) {
    console.log("Tags:");
    result.tags.forEach(tag => console.log("-", tag));
}

// 類型檢查
if (typeof result.price === 'number') {
    console.log(`Price: $${result.price.toFixed(2)}`);
}
```

### 示例 5: Encoder 和 Decoder（流式處理）

**Go:**
```go
package main

import (
    "encoding/json"
    "fmt"
    "os"
    "strings"
)

type Product struct {
    ID    int     `json:"id"`
    Name  string  `json:"name"`
    Price float64 `json:"price"`
}

func main() {
    // Encoder - 寫入到 Writer
    products := []Product{
        {ID: 1, Name: "Laptop", Price: 999.99},
        {ID: 2, Name: "Mouse", Price: 29.99},
        {ID: 3, Name: "Keyboard", Price: 79.99},
    }

    // 寫入文件
    file, _ := os.Create("products.json")
    defer file.Close()

    encoder := json.NewEncoder(file)
    encoder.SetIndent("", "  ")

    for _, product := range products {
        encoder.Encode(product)
    }

    fmt.Println("Encoded to products.json")

    // Decoder - 從 Reader 讀取
    jsonData := `
    {"id":1,"name":"Laptop","price":999.99}
    {"id":2,"name":"Mouse","price":29.99}
    {"id":3,"name":"Keyboard","price":79.99}
    `

    decoder := json.NewDecoder(strings.NewReader(jsonData))

    fmt.Println("\nDecoded products:")
    for decoder.More() {
        var product Product
        err := decoder.Decode(&product)
        if err != nil {
            fmt.Println("Error:", err)
            break
        }
        fmt.Printf("%+v\n", product)
    }
}
```

**Node.js:**
```javascript
const fs = require('fs');

const products = [
    {id: 1, name: "Laptop", price: 999.99},
    {id: 2, name: "Mouse", price: 29.99},
    {id: 3, name: "Keyboard", price: 79.99}
];

// 寫入文件
const writeStream = fs.createWriteStream('products.json');
products.forEach(product => {
    writeStream.write(JSON.stringify(product, null, 2) + '\n');
});
writeStream.end();

console.log("Encoded to products.json");

// 讀取流式 JSON（JSONL 格式）
const jsonData = `
{"id":1,"name":"Laptop","price":999.99}
{"id":2,"name":"Mouse","price":29.99}
{"id":3,"name":"Keyboard","price":79.99}
`;

console.log("\nDecoded products:");
jsonData.trim().split('\n').forEach(line => {
    if (line.trim()) {
        const product = JSON.parse(line);
        console.log(product);
    }
});
```

### 示例 6: 自定義 JSON 序列化

**Go:**
```go
package main

import (
    "encoding/json"
    "fmt"
    "time"
)

type Event struct {
    Name      string    `json:"name"`
    Timestamp time.Time `json:"-"`
    TimeStr   string    `json:"time"`
}

// MarshalJSON 自定義序列化
func (e Event) MarshalJSON() ([]byte, error) {
    type Alias Event
    return json.Marshal(&struct {
        *Alias
        TimeStr string `json:"time"`
    }{
        Alias:   (*Alias)(&e),
        TimeStr: e.Timestamp.Format("2006-01-02 15:04:05"),
    })
}

// UnmarshalJSON 自定義反序列化
func (e *Event) UnmarshalJSON(data []byte) error {
    type Alias Event
    aux := &struct {
        TimeStr string `json:"time"`
        *Alias
    }{
        Alias: (*Alias)(e),
    }

    if err := json.Unmarshal(data, &aux); err != nil {
        return err
    }

    t, err := time.Parse("2006-01-02 15:04:05", aux.TimeStr)
    if err != nil {
        return err
    }

    e.Timestamp = t
    return nil
}

func main() {
    event := Event{
        Name:      "Meeting",
        Timestamp: time.Now(),
    }

    // Marshal with custom format
    jsonData, _ := json.MarshalIndent(event, "", "  ")
    fmt.Println("Marshaled:")
    fmt.Println(string(jsonData))

    // Unmarshal with custom format
    jsonString := `{"name":"Conference","time":"2024-12-25 10:00:00"}`
    var event2 Event
    json.Unmarshal([]byte(jsonString), &event2)

    fmt.Printf("\nUnmarshaled: %+v\n", event2)
    fmt.Println("Timestamp:", event2.Timestamp)
}
```

**Node.js:**
```javascript
class Event {
    constructor(name, timestamp) {
        this.name = name;
        this.timestamp = timestamp;
    }

    // 自定義序列化
    toJSON() {
        return {
            name: this.name,
            time: this.formatTime(this.timestamp)
        };
    }

    formatTime(date) {
        const year = date.getFullYear();
        const month = String(date.getMonth() + 1).padStart(2, '0');
        const day = String(date.getDate()).padStart(2, '0');
        const hours = String(date.getHours()).padStart(2, '0');
        const minutes = String(date.getMinutes()).padStart(2, '0');
        const seconds = String(date.getSeconds()).padStart(2, '0');
        return `${year}-${month}-${day} ${hours}:${minutes}:${seconds}`;
    }

    // 自定義反序列化（靜態方法）
    static fromJSON(json) {
        const obj = typeof json === 'string' ? JSON.parse(json) : json;
        const timestamp = new Date(obj.time.replace(' ', 'T'));
        return new Event(obj.name, timestamp);
    }
}

// Marshal
const event = new Event("Meeting", new Date());
const jsonString = JSON.stringify(event, null, 2);
console.log("Marshaled:");
console.log(jsonString);

// Unmarshal
const jsonData = '{"name":"Conference","time":"2024-12-25 10:00:00"}';
const event2 = Event.fromJSON(jsonData);
console.log("\nUnmarshaled:", event2);
console.log("Timestamp:", event2.timestamp);
```

### 示例 7: 錯誤處理和驗證

**Go:**
```go
package main

import (
    "encoding/json"
    "fmt"
)

type Config struct {
    Host string `json:"host"`
    Port int    `json:"port"`
}

func parseConfig(jsonData string) (*Config, error) {
    var config Config
    err := json.Unmarshal([]byte(jsonData), &config)
    if err != nil {
        return nil, fmt.Errorf("invalid JSON: %w", err)
    }

    // 驗證
    if config.Host == "" {
        return nil, fmt.Errorf("host is required")
    }
    if config.Port <= 0 || config.Port > 65535 {
        return nil, fmt.Errorf("invalid port: %d", config.Port)
    }

    return &config, nil
}

func main() {
    // 有效的 JSON
    validJSON := `{"host":"localhost","port":8080}`
    if config, err := parseConfig(validJSON); err == nil {
        fmt.Printf("Valid config: %+v\n", config)
    }

    // 無效的 JSON
    invalidJSON := `{"host":"localhost","port":"invalid"}`
    if _, err := parseConfig(invalidJSON); err != nil {
        fmt.Printf("Error: %v\n", err)
    }

    // 缺少必要字段
    missingField := `{"port":8080}`
    if _, err := parseConfig(missingField); err != nil {
        fmt.Printf("Error: %v\n", err)
    }

    // 無效的端口
    invalidPort := `{"host":"localhost","port":99999}`
    if _, err := parseConfig(invalidPort); err != nil {
        fmt.Printf("Error: %v\n", err)
    }
}
```

**Node.js:**
```javascript
function parseConfig(jsonData) {
    let config;
    try {
        config = JSON.parse(jsonData);
    } catch (err) {
        throw new Error(`Invalid JSON: ${err.message}`);
    }

    // 驗證
    if (!config.host) {
        throw new Error('host is required');
    }
    if (typeof config.port !== 'number' || config.port <= 0 || config.port > 65535) {
        throw new Error(`invalid port: ${config.port}`);
    }

    return config;
}

// 有效的 JSON
try {
    const validJSON = '{"host":"localhost","port":8080}';
    const config = parseConfig(validJSON);
    console.log("Valid config:", config);
} catch (err) {
    console.error("Error:", err.message);
}

// 無效的 JSON
try {
    const invalidJSON = '{"host":"localhost","port":"invalid"}';
    parseConfig(invalidJSON);
} catch (err) {
    console.error("Error:", err.message);
}

// 缺少必要字段
try {
    const missingField = '{"port":8080}';
    parseConfig(missingField);
} catch (err) {
    console.error("Error:", err.message);
}

// 無效的端口
try {
    const invalidPort = '{"host":"localhost","port":99999}';
    parseConfig(invalidPort);
} catch (err) {
    console.error("Error:", err.message);
}
```

## 重點總結

1. **基本操作**
   - `json.Marshal()` - Go 對象轉 JSON（類似 JSON.stringify）
   - `json.Unmarshal()` - JSON 轉 Go 對象（類似 JSON.parse）
   - `json.MarshalIndent()` - 格式化輸出

2. **結構體標籤**
   - `json:"name"` - 自定義字段名
   - `json:",omitempty"` - 零值時省略
   - `json:"-"` - 忽略字段
   - 必須導出字段（首字母大寫）才能序列化

3. **流式處理**
   - `json.Encoder` - 寫入到 io.Writer
   - `json.Decoder` - 從 io.Reader 讀取
   - 適合處理大文件或網絡流

4. **自定義序列化**
   - 實現 `MarshalJSON()` 方法
   - 實現 `UnmarshalJSON()` 方法

5. **與 Node.js 的主要差異**
   - Go 需要定義結構體，Node.js 更靈活
   - Go 使用結構體標籤控制序列化
   - Go 的錯誤處理更明確
   - Go 支持類型安全

## 練習題

### 練習 1: 用戶配置
創建 User 結構體，實現 JSON 序列化，密碼字段不應該被序列化。

### 練習 2: API 響應
設計 API 響應結構體，包含 code, message, data 字段。

### 練習 3: 時間格式化
創建帶時間字段的結構體，自定義時間格式為 "YYYY-MM-DD"。

### 練習 4: 批量處理
讀取 JSON 文件（包含數組），解析並處理每個元素。

### 練習 5: 動態 JSON
處理未知結構的 JSON，使用 map[string]interface{}。

### 練習 6: JSON 轉換
編寫函數將駝峰命名的 JSON 轉換為蛇形命名。

### 練習 7: 數據驗證
創建 JSON 驗證器，檢查必要字段和類型。

### 練習 8: 配置文件
實現配置文件讀取和寫入功能，支持 JSON 格式。

## 參考答案

### 練習 1 答案
```go
package main

import (
    "encoding/json"
    "fmt"
)

type User struct {
    ID       int    `json:"id"`
    Username string `json:"username"`
    Email    string `json:"email"`
    Password string `json:"-"` // 不序列化
}

func main() {
    user := User{
        ID:       1,
        Username: "alice",
        Email:    "alice@example.com",
        Password: "secret123",
    }

    jsonData, _ := json.MarshalIndent(user, "", "  ")
    fmt.Println(string(jsonData))
}
```

### 練習 2 答案
```go
package main

import (
    "encoding/json"
    "fmt"
)

type APIResponse struct {
    Code    int         `json:"code"`
    Message string      `json:"message"`
    Data    interface{} `json:"data,omitempty"`
}

func NewSuccessResponse(data interface{}) APIResponse {
    return APIResponse{
        Code:    200,
        Message: "Success",
        Data:    data,
    }
}

func NewErrorResponse(code int, message string) APIResponse {
    return APIResponse{
        Code:    code,
        Message: message,
    }
}

func main() {
    // 成功響應
    successResp := NewSuccessResponse(map[string]string{
        "name": "Alice",
        "role": "Admin",
    })
    jsonData1, _ := json.MarshalIndent(successResp, "", "  ")
    fmt.Println("Success response:")
    fmt.Println(string(jsonData1))

    // 錯誤響應
    errorResp := NewErrorResponse(404, "Not Found")
    jsonData2, _ := json.MarshalIndent(errorResp, "", "  ")
    fmt.Println("\nError response:")
    fmt.Println(string(jsonData2))
}
```
