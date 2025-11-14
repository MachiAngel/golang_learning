# Chapter 20: 錯誤處理最佳實踐 (Error Handling Best Practices)

## 概述

Go 的錯誤處理機制簡潔明確，使用返回值而非異常。本章深入探討錯誤處理的最佳實踐，包括創建、包裝、檢查錯誤，以及如何設計良好的錯誤處理策略。

## Node.js 與 Go 的對比

| 特性 | Node.js | Go |
|------|---------|-----|
| 錯誤處理 | try/catch, 回調錯誤 | 返回值 error |
| 錯誤類型 | Error 對象 | error 接口 |
| 錯誤創建 | `new Error()` | `errors.New()`, `fmt.Errorf()` |
| 錯誤包裝 | 繼承 Error | `fmt.Errorf("%w", err)` |
| 錯誤檢查 | `instanceof` | `errors.Is()`, `errors.As()` |
| 堆棧追蹤 | 自動包含 | 需要第三方庫 |

## 詳細概念解釋

### 1. error 接口

```go
type error interface {
    Error() string
}
```

任何實現了 `Error() string` 方法的類型都是 error。

### 2. 錯誤處理哲學

- **顯式處理**：錯誤作為返回值，必須顯式處理
- **就近處理**：在錯誤發生處附近處理
- **向上傳遞**：無法處理時向上層傳遞
- **添加上下文**：傳遞時添加有用的上下文信息

### 3. 錯誤包裝（Go 1.13+）

使用 `%w` 包裝錯誤，保留原始錯誤信息：

```go
fmt.Errorf("operation failed: %w", err)
```

## 代碼示例

### 示例 1: 基本錯誤處理

**Go:**
```go
package main

import (
    "errors"
    "fmt"
)

// 創建簡單錯誤
func divide(a, b float64) (float64, error) {
    if b == 0 {
        return 0, errors.New("division by zero")
    }
    return a / b, nil
}

// 使用 fmt.Errorf 創建格式化錯誤
func divideWithContext(a, b float64) (float64, error) {
    if b == 0 {
        return 0, fmt.Errorf("cannot divide %f by zero", a)
    }
    return a / b, nil
}

func main() {
    // 正常情況
    result, err := divide(10, 2)
    if err != nil {
        fmt.Println("Error:", err)
        return
    }
    fmt.Println("Result:", result)

    // 錯誤情況
    result, err = divide(10, 0)
    if err != nil {
        fmt.Println("Error:", err)
        // 可以繼續處理或返回
    }

    // 帶上下文的錯誤
    result, err = divideWithContext(10, 0)
    if err != nil {
        fmt.Println("Error:", err)
    }
}
```

**Node.js 對比:**
```javascript
// 使用異常處理
function divide(a, b) {
    if (b === 0) {
        throw new Error("division by zero");
    }
    return a / b;
}

try {
    const result = divide(10, 2);
    console.log("Result:", result);
} catch (err) {
    console.log("Error:", err.message);
}

try {
    const result = divide(10, 0);
    console.log("Result:", result);
} catch (err) {
    console.log("Error:", err.message);
}

// 使用錯誤返回（Node.js 風格）
function divideCallback(a, b, callback) {
    if (b === 0) {
        callback(new Error("division by zero"), null);
        return;
    }
    callback(null, a / b);
}

divideCallback(10, 2, (err, result) => {
    if (err) {
        console.log("Error:", err.message);
        return;
    }
    console.log("Result:", result);
});

// 使用 Promise
function dividePromise(a, b) {
    return new Promise((resolve, reject) => {
        if (b === 0) {
            reject(new Error("division by zero"));
            return;
        }
        resolve(a / b);
    });
}

dividePromise(10, 2)
    .then(result => console.log("Result:", result))
    .catch(err => console.log("Error:", err.message));

// 使用 async/await
async function main() {
    try {
        const result = await dividePromise(10, 0);
        console.log("Result:", result);
    } catch (err) {
        console.log("Error:", err.message);
    }
}
```

### 示例 2: 自定義錯誤類型

**Go:**
```go
package main

import (
    "fmt"
)

// 自定義錯誤類型
type ValidationError struct {
    Field   string
    Message string
}

func (e ValidationError) Error() string {
    return fmt.Sprintf("validation error on field '%s': %s", e.Field, e.Message)
}

// 另一個自定義錯誤
type NotFoundError struct {
    Resource string
    ID       int
}

func (e NotFoundError) Error() string {
    return fmt.Sprintf("%s with ID %d not found", e.Resource, e.ID)
}

// 使用自定義錯誤
func validateAge(age int) error {
    if age < 0 {
        return ValidationError{
            Field:   "age",
            Message: "must be non-negative",
        }
    }
    if age > 150 {
        return ValidationError{
            Field:   "age",
            Message: "must be less than 150",
        }
    }
    return nil
}

func findUser(id int) error {
    // 模擬未找到用戶
    if id != 1 {
        return NotFoundError{
            Resource: "User",
            ID:       id,
        }
    }
    return nil
}

func main() {
    // 驗證錯誤
    if err := validateAge(-5); err != nil {
        fmt.Println("Error:", err)

        // 類型斷言檢查具體錯誤類型
        if ve, ok := err.(ValidationError); ok {
            fmt.Printf("Field: %s, Message: %s\n", ve.Field, ve.Message)
        }
    }

    // NotFound 錯誤
    if err := findUser(99); err != nil {
        fmt.Println("Error:", err)

        if nfe, ok := err.(NotFoundError); ok {
            fmt.Printf("Resource: %s, ID: %d\n", nfe.Resource, nfe.ID)
        }
    }
}
```

**Node.js 對比:**
```javascript
// 自定義錯誤類
class ValidationError extends Error {
    constructor(field, message) {
        super(`validation error on field '${field}': ${message}`);
        this.name = 'ValidationError';
        this.field = field;
        this.validationMessage = message;
    }
}

class NotFoundError extends Error {
    constructor(resource, id) {
        super(`${resource} with ID ${id} not found`);
        this.name = 'NotFoundError';
        this.resource = resource;
        this.id = id;
    }
}

function validateAge(age) {
    if (age < 0) {
        throw new ValidationError('age', 'must be non-negative');
    }
    if (age > 150) {
        throw new ValidationError('age', 'must be less than 150');
    }
}

function findUser(id) {
    if (id !== 1) {
        throw new NotFoundError('User', id);
    }
}

// 使用
try {
    validateAge(-5);
} catch (err) {
    console.log("Error:", err.message);

    if (err instanceof ValidationError) {
        console.log("Field:", err.field);
        console.log("Message:", err.validationMessage);
    }
}

try {
    findUser(99);
} catch (err) {
    console.log("Error:", err.message);

    if (err instanceof NotFoundError) {
        console.log("Resource:", err.resource);
        console.log("ID:", err.id);
    }
}
```

### 示例 3: 錯誤包裝和解包（Go 1.13+）

**Go:**
```go
package main

import (
    "errors"
    "fmt"
)

var (
    ErrDatabase   = errors.New("database error")
    ErrNotFound   = errors.New("not found")
    ErrPermission = errors.New("permission denied")
)

// 數據庫層
func queryDatabase(id int) error {
    if id < 0 {
        return ErrDatabase
    }
    if id > 100 {
        return ErrNotFound
    }
    return nil
}

// 服務層：包裝錯誤並添加上下文
func getUserService(id int) error {
    err := queryDatabase(id)
    if err != nil {
        // 使用 %w 包裝錯誤
        return fmt.Errorf("getUserService: failed to query user %d: %w", id, err)
    }
    return nil
}

// 控制器層：繼續包裝
func handleGetUser(id int) error {
    err := getUserService(id)
    if err != nil {
        return fmt.Errorf("handleGetUser: %w", err)
    }
    return nil
}

func main() {
    err := handleGetUser(150)
    if err != nil {
        fmt.Println("Error:", err)

        // 使用 errors.Is 檢查錯誤鏈中是否包含特定錯誤
        if errors.Is(err, ErrNotFound) {
            fmt.Println("處理 NotFound 錯誤")
        }

        if errors.Is(err, ErrDatabase) {
            fmt.Println("處理 Database 錯誤")
        }
    }

    fmt.Println()

    // 測試其他錯誤
    err2 := handleGetUser(-1)
    if err2 != nil {
        fmt.Println("Error:", err2)

        if errors.Is(err2, ErrDatabase) {
            fmt.Println("檢測到數據庫錯誤")
        }
    }
}
```

**Node.js 對比:**
```javascript
class DatabaseError extends Error {
    constructor(message) {
        super(message);
        this.name = 'DatabaseError';
    }
}

class NotFoundError extends Error {
    constructor(message) {
        super(message);
        this.name = 'NotFoundError';
    }
}

// 數據庫層
function queryDatabase(id) {
    if (id < 0) {
        throw new DatabaseError("database error");
    }
    if (id > 100) {
        throw new NotFoundError("not found");
    }
}

// 服務層：包裝錯誤
function getUserService(id) {
    try {
        queryDatabase(id);
    } catch (err) {
        // 創建新錯誤並保留原始錯誤
        const newErr = new Error(`getUserService: failed to query user ${id}: ${err.message}`);
        newErr.cause = err;  // Node.js 原生支持
        throw newErr;
    }
}

// 控制器層
function handleGetUser(id) {
    try {
        getUserService(id);
    } catch (err) {
        const newErr = new Error(`handleGetUser: ${err.message}`);
        newErr.cause = err;
        throw newErr;
    }
}

// 使用
try {
    handleGetUser(150);
} catch (err) {
    console.log("Error:", err.message);

    // 檢查錯誤鏈
    let currentErr = err;
    while (currentErr) {
        if (currentErr instanceof NotFoundError) {
            console.log("處理 NotFound 錯誤");
            break;
        }
        currentErr = currentErr.cause;
    }
}

console.log();

try {
    handleGetUser(-1);
} catch (err) {
    console.log("Error:", err.message);

    let currentErr = err;
    while (currentErr) {
        if (currentErr instanceof DatabaseError) {
            console.log("檢測到數據庫錯誤");
            break;
        }
        currentErr = currentErr.cause;
    }
}
```

### 示例 4: errors.As 使用

**Go:**
```go
package main

import (
    "errors"
    "fmt"
)

type TemporaryError struct {
    Msg string
}

func (e TemporaryError) Error() string {
    return e.Msg
}

func (e TemporaryError) Temporary() bool {
    return true
}

type PermanentError struct {
    Msg string
}

func (e PermanentError) Error() string {
    return e.Msg
}

func operation() error {
    // 返回臨時錯誤
    return fmt.Errorf("operation failed: %w", TemporaryError{Msg: "network timeout"})
}

func main() {
    err := operation()
    if err != nil {
        fmt.Println("Error:", err)

        // 使用 errors.As 提取特定類型的錯誤
        var tempErr TemporaryError
        if errors.As(err, &tempErr) {
            fmt.Println("這是臨時錯誤，可以重試")
            fmt.Println("Temporary:", tempErr.Temporary())
        }

        var permErr PermanentError
        if errors.As(err, &permErr) {
            fmt.Println("這是永久錯誤")
        } else {
            fmt.Println("不是永久錯誤")
        }
    }
}
```

**Node.js 對比:**
```javascript
class TemporaryError extends Error {
    constructor(message) {
        super(message);
        this.name = 'TemporaryError';
    }

    isTemporary() {
        return true;
    }
}

class PermanentError extends Error {
    constructor(message) {
        super(message);
        this.name = 'PermanentError';
    }
}

function operation() {
    const tempErr = new TemporaryError("network timeout");
    const err = new Error("operation failed");
    err.cause = tempErr;
    throw err;
}

try {
    operation();
} catch (err) {
    console.log("Error:", err.message);

    // 查找特定類型錯誤
    let currentErr = err;
    let foundTemp = false;

    while (currentErr) {
        if (currentErr instanceof TemporaryError) {
            console.log("這是臨時錯誤，可以重試");
            console.log("Temporary:", currentErr.isTemporary());
            foundTemp = true;
            break;
        }
        currentErr = currentErr.cause;
    }

    if (!foundTemp) {
        currentErr = err;
        while (currentErr) {
            if (currentErr instanceof PermanentError) {
                console.log("這是永久錯誤");
                break;
            }
            currentErr = currentErr.cause;
        }
        if (!currentErr) {
            console.log("不是永久錯誤");
        }
    }
}
```

### 示例 5: 錯誤處理模式

**Go:**
```go
package main

import (
    "errors"
    "fmt"
)

// 模式 1: 早返回（Early Return）
func processData(data string) error {
    if data == "" {
        return errors.New("data is empty")
    }

    if len(data) > 100 {
        return errors.New("data too long")
    }

    // 處理邏輯...
    fmt.Println("Processing:", data)
    return nil
}

// 模式 2: 錯誤聚合
type MultiError struct {
    Errors []error
}

func (m MultiError) Error() string {
    return fmt.Sprintf("multiple errors occurred: %v", m.Errors)
}

func validateUser(name string, age int, email string) error {
    var errs []error

    if name == "" {
        errs = append(errs, errors.New("name is required"))
    }

    if age < 18 {
        errs = append(errs, errors.New("age must be at least 18"))
    }

    if email == "" {
        errs = append(errs, errors.New("email is required"))
    }

    if len(errs) > 0 {
        return MultiError{Errors: errs}
    }

    return nil
}

// 模式 3: 默認值處理
func getConfigValue(key string) (string, error) {
    // 模擬配置查找
    configs := map[string]string{
        "host": "localhost",
    }

    value, ok := configs[key]
    if !ok {
        return "", fmt.Errorf("config key '%s' not found", key)
    }

    return value, nil
}

func getConfigWithDefault(key, defaultValue string) string {
    value, err := getConfigValue(key)
    if err != nil {
        return defaultValue
    }
    return value
}

// 模式 4: 重試邏輯
func retryOperation(maxRetries int, operation func() error) error {
    var err error
    for i := 0; i < maxRetries; i++ {
        err = operation()
        if err == nil {
            return nil
        }

        // 檢查是否是臨時錯誤
        var tempErr interface{ Temporary() bool }
        if !errors.As(err, &tempErr) || !tempErr.Temporary() {
            return err  // 不是臨時錯誤，立即返回
        }

        fmt.Printf("Attempt %d failed: %v, retrying...\n", i+1, err)
    }

    return fmt.Errorf("operation failed after %d attempts: %w", maxRetries, err)
}

func main() {
    // 模式 1
    fmt.Println("=== Early Return ===")
    if err := processData(""); err != nil {
        fmt.Println("Error:", err)
    }

    // 模式 2
    fmt.Println("\n=== Error Aggregation ===")
    if err := validateUser("", 15, ""); err != nil {
        fmt.Println("Error:", err)
        if me, ok := err.(MultiError); ok {
            for i, e := range me.Errors {
                fmt.Printf("  Error %d: %v\n", i+1, e)
            }
        }
    }

    // 模式 3
    fmt.Println("\n=== Default Value ===")
    host := getConfigWithDefault("host", "0.0.0.0")
    fmt.Println("Host:", host)

    port := getConfigWithDefault("port", "8080")
    fmt.Println("Port:", port)

    // 模式 4
    fmt.Println("\n=== Retry Logic ===")
    attempt := 0
    err := retryOperation(3, func() error {
        attempt++
        if attempt < 3 {
            return TemporaryError{Msg: "network error"}
        }
        return nil
    })
    if err != nil {
        fmt.Println("Final error:", err)
    } else {
        fmt.Println("Operation succeeded")
    }
}

type TemporaryError struct {
    Msg string
}

func (e TemporaryError) Error() string {
    return e.Msg
}

func (e TemporaryError) Temporary() bool {
    return true
}
```

**Node.js 對比:**
```javascript
// 模式 1: 早返回
function processData(data) {
    if (!data) {
        throw new Error("data is empty");
    }

    if (data.length > 100) {
        throw new Error("data too long");
    }

    console.log("Processing:", data);
}

// 模式 2: 錯誤聚合
class MultiError extends Error {
    constructor(errors) {
        super(`multiple errors occurred: ${errors.map(e => e.message).join(', ')}`);
        this.name = 'MultiError';
        this.errors = errors;
    }
}

function validateUser(name, age, email) {
    const errors = [];

    if (!name) {
        errors.push(new Error("name is required"));
    }

    if (age < 18) {
        errors.push(new Error("age must be at least 18"));
    }

    if (!email) {
        errors.push(new Error("email is required"));
    }

    if (errors.length > 0) {
        throw new MultiError(errors);
    }
}

// 模式 3: 默認值處理
function getConfigValue(key) {
    const configs = {
        host: 'localhost'
    };

    if (!(key in configs)) {
        throw new Error(`config key '${key}' not found`);
    }

    return configs[key];
}

function getConfigWithDefault(key, defaultValue) {
    try {
        return getConfigValue(key);
    } catch (err) {
        return defaultValue;
    }
}

// 模式 4: 重試邏輯
class TemporaryError extends Error {
    constructor(message) {
        super(message);
        this.name = 'TemporaryError';
    }
    isTemporary() {
        return true;
    }
}

async function retryOperation(maxRetries, operation) {
    let lastErr;

    for (let i = 0; i < maxRetries; i++) {
        try {
            await operation();
            return;  // 成功
        } catch (err) {
            lastErr = err;

            // 檢查是否是臨時錯誤
            if (!(err instanceof TemporaryError)) {
                throw err;  // 不是臨時錯誤，立即拋出
            }

            console.log(`Attempt ${i + 1} failed: ${err.message}, retrying...`);
        }
    }

    throw new Error(`operation failed after ${maxRetries} attempts: ${lastErr.message}`);
}

// 使用示例
console.log("=== Early Return ===");
try {
    processData("");
} catch (err) {
    console.log("Error:", err.message);
}

console.log("\n=== Error Aggregation ===");
try {
    validateUser("", 15, "");
} catch (err) {
    console.log("Error:", err.message);
    if (err instanceof MultiError) {
        err.errors.forEach((e, i) => {
            console.log(`  Error ${i + 1}: ${e.message}`);
        });
    }
}

console.log("\n=== Default Value ===");
const host = getConfigWithDefault("host", "0.0.0.0");
console.log("Host:", host);

const port = getConfigWithDefault("port", "8080");
console.log("Port:", port);

console.log("\n=== Retry Logic ===");
let attempt = 0;
retryOperation(3, async () => {
    attempt++;
    if (attempt < 3) {
        throw new TemporaryError("network error");
    }
})
    .then(() => console.log("Operation succeeded"))
    .catch(err => console.log("Final error:", err.message));
```

## 重點總結

### 錯誤處理對比

| 特性 | Go | Node.js |
|------|-----|---------|
| 機制 | 返回值 | 異常 / 回調 / Promise |
| 檢查 | 顯式 if err != nil | try/catch |
| 包裝 | fmt.Errorf("%w") | err.cause |
| 類型檢查 | errors.Is/As | instanceof |
| 堆棧 | 需要庫 | 自動包含 |

### 最佳實踐

1. **總是處理錯誤**：不要忽略錯誤返回值
2. **添加上下文**：包裝錯誤時添加有用信息
3. **使用哨兵錯誤**：定義包級錯誤變量供外部檢查
4. **自定義錯誤**：複雜場景使用自定義錯誤類型
5. **早返回**：發現錯誤立即返回，避免深度嵌套

### 常用模式

```go
// 基本檢查
if err != nil {
    return err
}

// 添加上下文
if err != nil {
    return fmt.Errorf("operation failed: %w", err)
}

// 檢查特定錯誤
if errors.Is(err, ErrNotFound) {
    // 處理
}

// 提取錯誤類型
var valErr ValidationError
if errors.As(err, &valErr) {
    // 使用 valErr
}
```

## 練習題

### 練習 1: 文件操作錯誤處理
實現文件讀寫函數，處理各種錯誤情況（文件不存在、權限不足等）。

### 練習 2: API 錯誤響應
設計 HTTP API 的錯誤響應系統，包含錯誤碼、消息、詳情。

### 練習 3: 重試機制
實現一個通用的重試函數，支持指數退避和最大重試次數。

### 練習 4: 錯誤日誌
創建錯誤日誌系統，記錄錯誤類型、時間、上下文。

### 練習 5: 錯誤恢復
實現一個錯誤恢復機制，在特定錯誤發生時嘗試自動修復。

### 答案提示

**練習 3:**
```go
func RetryWithBackoff(maxRetries int, initialDelay time.Duration, operation func() error) error {
    delay := initialDelay

    for i := 0; i < maxRetries; i++ {
        err := operation()
        if err == nil {
            return nil
        }

        var tempErr interface{ Temporary() bool }
        if !errors.As(err, &tempErr) || !tempErr.Temporary() {
            return err
        }

        if i < maxRetries-1 {
            time.Sleep(delay)
            delay *= 2  // 指數退避
        }
    }

    return fmt.Errorf("operation failed after %d attempts", maxRetries)
}
```
