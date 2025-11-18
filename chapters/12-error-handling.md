# 第十二章：多返回值與錯誤處理

## 章節概述

本章將深入講解 Go 的錯誤處理機制，包括 error 接口、錯誤創建與處理、panic/recover，以及與 JavaScript 的 try-catch 機制的對比。

## 錯誤處理哲學

### JavaScript: 異常 (Exceptions)

```javascript
// JavaScript 使用 try-catch
function divide(a, b) {
    if (b === 0) {
        throw new Error("division by zero");
    }
    return a / b;
}

try {
    const result = divide(10, 0);
    console.log(result);
} catch (error) {
    console.error("Error:", error.message);
} finally {
    console.log("Cleanup");
}

// 或使用 Promise
function divideAsync(a, b) {
    return new Promise((resolve, reject) => {
        if (b === 0) {
            reject(new Error("division by zero"));
        } else {
            resolve(a / b);
        }
    });
}

divideAsync(10, 0)
    .then(result => console.log(result))
    .catch(error => console.error("Error:", error.message));

// async/await
async function main() {
    try {
        const result = await divideAsync(10, 0);
        console.log(result);
    } catch (error) {
        console.error("Error:", error.message);
    }
}
```

### Go: 顯式錯誤返回

```go
package main

import (
    "errors"
    "fmt"
)

// Go 使用多返回值返回錯誤
func divide(a, b float64) (float64, error) {
    if b == 0 {
        return 0, errors.New("division by zero")
    }
    return a / b, nil
}

func main() {
    result, err := divide(10, 0)
    if err != nil {
        fmt.Println("Error:", err)
        return
    }
    fmt.Println("Result:", result)
}
```

## error 接口

### error 是接口

```go
package main

import "fmt"

// error 是內置接口
// type error interface {
//     Error() string
// }

// 任何實現了 Error() string 的類型都是 error

type MyError struct {
    Code    int
    Message string
}

func (e *MyError) Error() string {
    return fmt.Sprintf("Error %d: %s", e.Code, e.Message)
}

func doSomething() error {
    return &MyError{
        Code:    404,
        Message: "Not Found",
    }
}

func main() {
    err := doSomething()
    if err != nil {
        fmt.Println(err)  // Error 404: Not Found

        // 類型斷言
        if myErr, ok := err.(*MyError); ok {
            fmt.Printf("Code: %d, Message: %s\n", myErr.Code, myErr.Message)
        }
    }
}
```

**對比 JavaScript:**
```javascript
// JavaScript Error 對象
class MyError extends Error {
    constructor(code, message) {
        super(message);
        this.code = code;
        this.name = "MyError";
    }
}

function doSomething() {
    throw new MyError(404, "Not Found");
}

try {
    doSomething();
} catch (error) {
    console.log(error.message);  // "Not Found"

    if (error instanceof MyError) {
        console.log(`Code: ${error.code}, Message: ${error.message}`);
    }
}
```

## 創建錯誤

### 方式 1: errors.New

```go
package main

import (
    "errors"
    "fmt"
)

func validate(age int) error {
    if age < 0 {
        return errors.New("age cannot be negative")
    }
    if age > 150 {
        return errors.New("age too large")
    }
    return nil
}

func main() {
    err := validate(-5)
    if err != nil {
        fmt.Println(err)  // age cannot be negative
    }
}
```

### 方式 2: fmt.Errorf

```go
package main

import "fmt"

func validateUser(name string, age int) error {
    if name == "" {
        return fmt.Errorf("name is empty")
    }
    if age < 18 {
        return fmt.Errorf("user %s is underage: %d", name, age)
    }
    return nil
}

func main() {
    err := validateUser("Alice", 15)
    if err != nil {
        fmt.Println(err)  // user Alice is underage: 15
    }
}
```

### 方式 3: 自定義錯誤類型

```go
package main

import "fmt"

// 驗證錯誤
type ValidationError struct {
    Field   string
    Message string
}

func (e *ValidationError) Error() string {
    return fmt.Sprintf("%s: %s", e.Field, e.Message)
}

// 網絡錯誤
type NetworkError struct {
    URL        string
    StatusCode int
}

func (e *NetworkError) Error() string {
    return fmt.Sprintf("network error: %s returned %d", e.URL, e.StatusCode)
}

func validateEmail(email string) error {
    if email == "" {
        return &ValidationError{
            Field:   "email",
            Message: "cannot be empty",
        }
    }
    // 更多驗證...
    return nil
}

func main() {
    err := validateEmail("")
    if err != nil {
        fmt.Println(err)  // email: cannot be empty

        // 類型檢查
        if valErr, ok := err.(*ValidationError); ok {
            fmt.Printf("Validation failed on field: %s\n", valErr.Field)
        }
    }
}
```

**對比 JavaScript:**
```javascript
class ValidationError extends Error {
    constructor(field, message) {
        super(`${field}: ${message}`);
        this.field = field;
        this.name = "ValidationError";
    }
}

class NetworkError extends Error {
    constructor(url, statusCode) {
        super(`network error: ${url} returned ${statusCode}`);
        this.url = url;
        this.statusCode = statusCode;
        this.name = "NetworkError";
    }
}

function validateEmail(email) {
    if (!email) {
        throw new ValidationError("email", "cannot be empty");
    }
}

try {
    validateEmail("");
} catch (error) {
    console.log(error.message);

    if (error instanceof ValidationError) {
        console.log(`Validation failed on field: ${error.field}`);
    }
}
```

## 錯誤處理模式

### 模式 1: 提前返回

```go
package main

import (
    "fmt"
    "os"
)

func processFile(filename string) error {
    // 讀取文件
    data, err := os.ReadFile(filename)
    if err != nil {
        return fmt.Errorf("failed to read file: %w", err)
    }

    // 解析數據
    result, err := parse(data)
    if err != nil {
        return fmt.Errorf("failed to parse data: %w", err)
    }

    // 保存結果
    if err := save(result); err != nil {
        return fmt.Errorf("failed to save result: %w", err)
    }

    return nil
}

func parse(data []byte) (interface{}, error) {
    // 解析邏輯
    return nil, nil
}

func save(data interface{}) error {
    // 保存邏輯
    return nil
}

func main() {
    if err := processFile("data.txt"); err != nil {
        fmt.Println("Error:", err)
    }
}
```

**對比 JavaScript:**
```javascript
const fs = require('fs').promises;

async function processFile(filename) {
    try {
        // 讀取文件
        const data = await fs.readFile(filename);

        // 解析數據
        const result = await parse(data);

        // 保存結果
        await save(result);
    } catch (error) {
        throw new Error(`Failed to process file: ${error.message}`);
    }
}

async function main() {
    try {
        await processFile("data.txt");
    } catch (error) {
        console.error("Error:", error.message);
    }
}
```

### 模式 2: 錯誤包裝

```go
package main

import (
    "errors"
    "fmt"
)

func level3() error {
    return errors.New("original error")
}

func level2() error {
    err := level3()
    if err != nil {
        return fmt.Errorf("level2: %w", err)  // %w 包裝錯誤
    }
    return nil
}

func level1() error {
    err := level2()
    if err != nil {
        return fmt.Errorf("level1: %w", err)
    }
    return nil
}

func main() {
    err := level1()
    if err != nil {
        fmt.Println(err)
        // level1: level2: original error

        // 解包錯誤
        var originalErr error
        if errors.Is(err, originalErr) {
            fmt.Println("Found original error")
        }

        // 獲取原始錯誤
        fmt.Println(errors.Unwrap(err))  // level2: original error
    }
}
```

### 模式 3: 錯誤檢查

```go
package main

import (
    "errors"
    "fmt"
    "os"
)

var ErrNotFound = errors.New("not found")
var ErrPermissionDenied = errors.New("permission denied")

func loadData(id int) (string, error) {
    if id < 0 {
        return "", ErrNotFound
    }
    if id > 100 {
        return "", ErrPermissionDenied
    }
    return "data", nil
}

func main() {
    data, err := loadData(-1)
    if err != nil {
        // 檢查特定錯誤
        if errors.Is(err, ErrNotFound) {
            fmt.Println("Data not found")
        } else if errors.Is(err, ErrPermissionDenied) {
            fmt.Println("Permission denied")
        } else {
            fmt.Println("Unknown error:", err)
        }
        return
    }

    fmt.Println("Data:", data)

    // 檢查特定類型的錯誤
    _, err = os.Open("nonexistent.txt")
    if err != nil {
        var pathErr *os.PathError
        if errors.As(err, &pathErr) {
            fmt.Printf("Path error: %s\n", pathErr.Path)
        }
    }
}
```

**對比 JavaScript:**
```javascript
class NotFoundError extends Error {
    constructor(message) {
        super(message);
        this.name = "NotFoundError";
    }
}

class PermissionDeniedError extends Error {
    constructor(message) {
        super(message);
        this.name = "PermissionDeniedError";
    }
}

function loadData(id) {
    if (id < 0) {
        throw new NotFoundError("Data not found");
    }
    if (id > 100) {
        throw new PermissionDeniedError("Permission denied");
    }
    return "data";
}

try {
    const data = loadData(-1);
    console.log("Data:", data);
} catch (error) {
    if (error instanceof NotFoundError) {
        console.log("Data not found");
    } else if (error instanceof PermissionDeniedError) {
        console.log("Permission denied");
    } else {
        console.log("Unknown error:", error.message);
    }
}
```

## panic 和 recover

### panic（類似 throw）

```go
package main

import "fmt"

func riskyOperation() {
    panic("something went wrong")  // 拋出 panic
}

func main() {
    fmt.Println("Start")
    // riskyOperation()  // 會導致程序崩潰
    fmt.Println("End")  // 不會執行
}
```

**何時使用 panic：**
- 程序無法繼續運行的情況
- 初始化失敗
- 不應該發生的錯誤（編程錯誤）

**不應該使用 panic：**
- 正常的錯誤處理（使用 error）
- 可預見的錯誤
- 用戶輸入錯誤

### recover（類似 catch）

```go
package main

import "fmt"

func riskyOperation() {
    panic("something went wrong")
}

func safeCall() {
    defer func() {
        if r := recover(); r != nil {
            fmt.Println("Recovered from:", r)
        }
    }()

    riskyOperation()
    fmt.Println("This won't execute")
}

func main() {
    fmt.Println("Start")
    safeCall()
    fmt.Println("End")  // 會執行

    // 輸出：
    // Start
    // Recovered from: something went wrong
    // End
}
```

**對比 JavaScript:**
```javascript
function riskyOperation() {
    throw new Error("something went wrong");
}

function safeCall() {
    try {
        riskyOperation();
        console.log("This won't execute");
    } catch (error) {
        console.log("Caught:", error.message);
    } finally {
        console.log("Cleanup");
    }
}

console.log("Start");
safeCall();
console.log("End");

// 輸出：
// Start
// Caught: something went wrong
// Cleanup
// End
```

### panic/recover 的限制

```go
package main

import "fmt"

func main() {
    // ❌ recover 只能在 defer 中使用
    // r := recover()  // 不起作用

    // ✅ 正確用法
    defer func() {
        if r := recover(); r != nil {
            fmt.Println("Recovered:", r)
        }
    }()

    panic("test")
}
```

### 實際應用：HTTP 服務器

```go
package main

import (
    "fmt"
    "net/http"
)

func handler(w http.ResponseWriter, r *http.Request) {
    defer func() {
        if r := recover(); r != nil {
            // 記錄錯誤
            fmt.Printf("Panic: %v\n", r)

            // 返回 500 錯誤
            http.Error(w, "Internal Server Error", 500)
        }
    }()

    // 可能 panic 的代碼
    riskyOperation()

    w.Write([]byte("OK"))
}

func riskyOperation() {
    // 某些操作...
}

func main() {
    http.HandleFunc("/", handler)
    // http.ListenAndServe(":8080", nil)
}
```

**對比 Express.js:**
```javascript
const express = require('express');
const app = express();

// 錯誤處理中間件
app.use((err, req, res, next) => {
    console.error('Error:', err);
    res.status(500).send('Internal Server Error');
});

app.get('/', (req, res, next) => {
    try {
        riskyOperation();
        res.send('OK');
    } catch (error) {
        next(error);  // 傳遞給錯誤處理中間件
    }
});

function riskyOperation() {
    throw new Error("something went wrong");
}

// app.listen(8080);
```

## 錯誤處理最佳實踐

### 1. 總是檢查錯誤

```go
// ❌ 不好
data, _ := os.ReadFile("file.txt")

// ✅ 好
data, err := os.ReadFile("file.txt")
if err != nil {
    return fmt.Errorf("failed to read file: %w", err)
}
```

### 2. 提供上下文

```go
// ❌ 不好
if err != nil {
    return err  // 丟失上下文
}

// ✅ 好
if err != nil {
    return fmt.Errorf("failed to process user %s: %w", userName, err)
}
```

### 3. 使用哨兵錯誤

```go
package main

import "errors"

// 定義常見錯誤
var (
    ErrNotFound         = errors.New("not found")
    ErrInvalidInput     = errors.New("invalid input")
    ErrUnauthorized     = errors.New("unauthorized")
    ErrInternalError    = errors.New("internal error")
)

func getUser(id int) (User, error) {
    if id < 0 {
        return User{}, ErrInvalidInput
    }
    // ...
    return User{}, ErrNotFound
}

type User struct{}
```

### 4. 自定義錯誤類型

```go
package main

import "fmt"

type APIError struct {
    StatusCode int
    Message    string
    Err        error
}

func (e *APIError) Error() string {
    return fmt.Sprintf("API error %d: %s", e.StatusCode, e.Message)
}

func (e *APIError) Unwrap() error {
    return e.Err
}

func callAPI() error {
    return &APIError{
        StatusCode: 404,
        Message:    "User not found",
    }
}
```

### 5. 錯誤日誌

```go
package main

import (
    "fmt"
    "log"
)

func processData() error {
    err := doSomething()
    if err != nil {
        // 記錄詳細錯誤
        log.Printf("Error in processData: %v", err)

        // 返回簡化的錯誤給用戶
        return fmt.Errorf("failed to process data")
    }
    return nil
}

func doSomething() error {
    return nil
}
```

## 重點總結

### 錯誤處理對比

| 特性 | Go | JavaScript |
|------|-----|------------|
| **機制** | 返回值 | 異常 |
| **檢查** | 顯式 if err != nil | try-catch |
| **傳播** | return err | throw |
| **包裝** | fmt.Errorf("%w") | Error cause chain |
| **緊急情況** | panic/recover | throw/catch |

### Go 錯誤處理優勢

1. **顯式**：強制檢查錯誤
2. **清晰**：錯誤處理邏輯明確
3. **性能**：無異常拋出開銷
4. **類型安全**：編譯時檢查

### JavaScript 異常優勢

1. **簡潔**：不需要每次檢查
2. **自動傳播**：向上冒泡
3. **統一處理**：catch 捕獲所有
4. **堆棧跟蹤**：自動記錄

### 最佳實踐

1. **優先使用 error 返回**
2. **總是檢查錯誤**
3. **添加上下文信息**
4. **使用哨兵錯誤**
5. **panic 只用於不可恢復的錯誤**
6. **在 HTTP handler 中 recover**

## 練習題

### 基礎題

1. **錯誤創建**
   - 使用三種方式創建錯誤
   - 理解 errors.New、fmt.Errorf、自定義類型
   - 對比 JavaScript Error

2. **錯誤檢查**
   - 實現文件讀取函數
   - 正確處理所有可能的錯誤
   - 提供有用的錯誤消息

3. **錯誤傳播**
   - 創建多層函數調用
   - 正確傳播錯誤
   - 添加上下文信息

### 進階題

4. **自定義錯誤類型**
   - 實現 ValidationError
   - 包含字段名和錯誤消息
   - 實現 Unwrap 方法

5. **錯誤包裝**
   - 使用 %w 包裝錯誤
   - 使用 errors.Is 和 errors.As
   - 理解錯誤鏈

6. **panic/recover**
   - 實現安全的 API handler
   - 捕獲 panic 並記錄
   - 返回適當的 HTTP 錯誤

### 實戰題

7. **錯誤處理策略**
   - 設計一個完整的錯誤處理系統
   - 包括日誌、監控、用戶反饋
   - 對比 JavaScript 的錯誤處理

8. **重構練習**
   - 將 try-catch 代碼改為 Go 風格
   - 處理所有邊界情況
   - 改善錯誤消息

9. **錯誤測試**
   - 為錯誤處理編寫測試
   - 測試所有錯誤路徑
   - 驗證錯誤類型和消息

## 下一章預告

下一章我們將學習 Go 的指針基礎，包括：
- 指針概念（*, &）
- 值傳遞 vs 指針傳遞
- Node.js 工程師視角的理解
- 指針的常見陷阱
- 何時使用指針

這是本系列最後一章！準備好理解 Go 的指針了嗎？
