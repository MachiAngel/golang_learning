# 第 61 章: 2025 年 Go 寫作風格演進與最佳實踐

## 概述

就像 JavaScript/Node.js 經歷了從 callback hell → Promise → async/await 的演進，Go 語言也在不斷演化。本章將詳細介紹 Go 從 1.0 到 1.23+ 的寫作風格變化，特別是 2025 年的現代 Go 開發方式。

## Node.js 演進對比

### JavaScript/Node.js 的演進歷程

```javascript
// 2012-2014: Callback Hell
fs.readFile('file1.txt', (err, data1) => {
  if (err) throw err;
  fs.readFile('file2.txt', (err, data2) => {
    if (err) throw err;
    fs.writeFile('output.txt', data1 + data2, (err) => {
      if (err) throw err;
      console.log('Done!');
    });
  });
});

// 2015-2016: Promises
readFile('file1.txt')
  .then(data1 => readFile('file2.txt'))
  .then(data2 => writeFile('output.txt', data1 + data2))
  .then(() => console.log('Done!'))
  .catch(err => console.error(err));

// 2017-現在: Async/Await
async function processFiles() {
  try {
    const data1 = await readFile('file1.txt');
    const data2 = await readFile('file2.txt');
    await writeFile('output.txt', data1 + data2);
    console.log('Done!');
  } catch (err) {
    console.error(err);
  }
}

// 2020+: ES Modules, Optional Chaining, Nullish Coalescing
import { readFile } from 'fs/promises';
const config = userConfig?.timeout ?? 3000;
```

## Go 語言的演進歷程

### 1. 錯誤處理演進 (Go 1.0 → 1.13 → 1.23)

#### Go 1.0-1.12 (2012-2019): 基礎錯誤處理

```go
// 舊風格：簡單的錯誤返回
func ReadConfig(filename string) (*Config, error) {
    data, err := ioutil.ReadFile(filename)
    if err != nil {
        return nil, err  // 錯誤信息丟失上下文
    }

    var config Config
    err = json.Unmarshal(data, &config)
    if err != nil {
        return nil, err  // 無法知道是哪個文件解析失敗
    }

    return &config, nil
}
```

#### Go 1.13-1.19 (2019-2022): 錯誤包裝

```go
import (
    "errors"
    "fmt"
)

// 引入 %w 進行錯誤包裝
func ReadConfig(filename string) (*Config, error) {
    data, err := os.ReadFile(filename)
    if err != nil {
        return nil, fmt.Errorf("讀取配置文件失敗 %s: %w", filename, err)
    }

    var config Config
    err = json.Unmarshal(data, &config)
    if err != nil {
        return nil, fmt.Errorf("解析配置文件失敗 %s: %w", filename, err)
    }

    return &config, nil
}

// 使用 errors.Is 和 errors.As
func HandleError(err error) {
    if errors.Is(err, os.ErrNotExist) {
        log.Println("文件不存在")
    }

    var pathErr *os.PathError
    if errors.As(err, &pathErr) {
        log.Printf("路徑錯誤: %s", pathErr.Path)
    }
}
```

#### Go 1.20+ (2023-2025): errors.Join 和多錯誤處理

```go
// 2025 現代風格：處理多個錯誤
func ProcessMultipleFiles(files []string) error {
    var errs []error

    for _, file := range files {
        if err := processFile(file); err != nil {
            errs = append(errs, fmt.Errorf("處理 %s 失敗: %w", file, err))
        }
    }

    // Go 1.20+ 的 errors.Join
    return errors.Join(errs...)
}
```

### 2. 泛型革命 (Go 1.18+, 2022-2025)

這是 Go 最重大的變化之一，類似於 TypeScript 為 JavaScript 帶來的類型系統。

#### 泛型之前 (Go 1.0-1.17)

```go
// 需要為每種類型寫重複代碼
func MaxInt(a, b int) int {
    if a > b {
        return a
    }
    return b
}

func MaxFloat64(a, b float64) float64 {
    if a > b {
        return a
    }
    return b
}

// 或者使用 interface{} 失去類型安全
func Max(a, b interface{}) interface{} {
    // 需要類型斷言，容易出錯
}
```

```javascript
// Node.js 對比：JavaScript 本來就是動態類型
function max(a, b) {
    return a > b ? a : b;
}
```

#### 泛型之後 (Go 1.18+, 2025 推薦)

```go
// 現代 Go：使用泛型
func Max[T comparable](a, b T) T {
    if a > b {
        return a
    }
    return b
}

// 使用約束
type Number interface {
    int | int64 | float64
}

func Sum[T Number](numbers []T) T {
    var total T
    for _, n := range numbers {
        total += n
    }
    return total
}

// 泛型容器
type Stack[T any] struct {
    items []T
}

func (s *Stack[T]) Push(item T) {
    s.items = append(s.items, item)
}

func (s *Stack[T]) Pop() (T, bool) {
    if len(s.items) == 0 {
        var zero T
        return zero, false
    }
    item := s.items[len(s.items)-1]
    s.items = s.items[:len(s.items)-1]
    return item, true
}
```

```typescript
// TypeScript 對比
class Stack<T> {
    private items: T[] = [];

    push(item: T): void {
        this.items.push(item);
    }

    pop(): T | undefined {
        return this.items.pop();
    }
}
```

### 3. 標準庫現代化 (Go 1.21+)

#### slices 包 (Go 1.21+)

```go
import "slices"

// 2025 現代風格：使用 slices 包
func ModernSliceOperations() {
    numbers := []int{3, 1, 4, 1, 5, 9, 2, 6}

    // 排序
    slices.Sort(numbers)  // 使用泛型，不需要實現 sort.Interface

    // 查找
    found := slices.Contains(numbers, 5)
    index := slices.Index(numbers, 5)

    // 比較
    equal := slices.Equal(numbers, []int{1, 1, 2, 3, 4, 5, 6, 9})

    // 去重
    unique := slices.Compact(slices.Clone(numbers))

    // 刪除
    numbers = slices.Delete(numbers, 2, 4)  // 刪除索引 2-3

    // 插入
    numbers = slices.Insert(numbers, 1, 100, 200)
}
```

```go
// 舊風格 (Go 1.20 之前)
func OldSliceOperations() {
    numbers := []int{3, 1, 4, 1, 5, 9, 2, 6}

    // 排序需要使用 sort 包
    sort.Ints(numbers)

    // 查找需要手動實現
    found := false
    for _, n := range numbers {
        if n == 5 {
            found = true
            break
        }
    }

    // 刪除需要手動切片操作
    numbers = append(numbers[:2], numbers[4:]...)
}
```

```javascript
// JavaScript 對比：內建數組方法
const numbers = [3, 1, 4, 1, 5, 9, 2, 6];

numbers.sort((a, b) => a - b);
const found = numbers.includes(5);
const index = numbers.indexOf(5);
const unique = [...new Set(numbers)];
numbers.splice(2, 2);  // 刪除索引 2-3
```

#### maps 包 (Go 1.21+)

```go
import "maps"

// 2025 現代風格
func ModernMapOperations() {
    m1 := map[string]int{"a": 1, "b": 2}
    m2 := map[string]int{"c": 3, "d": 4}

    // 複製
    m3 := maps.Clone(m1)

    // 合併
    maps.Copy(m3, m2)  // m3 現在包含所有鍵值對

    // 比較
    equal := maps.Equal(m1, m2)

    // 刪除符合條件的項
    maps.DeleteFunc(m3, func(k string, v int) bool {
        return v > 2
    })
}
```

```go
// 舊風格
func OldMapOperations() {
    m1 := map[string]int{"a": 1, "b": 2}

    // 複製需要手動循環
    m3 := make(map[string]int)
    for k, v := range m1 {
        m3[k] = v
    }

    // 合併需要手動循環
    m2 := map[string]int{"c": 3, "d": 4}
    for k, v := range m2 {
        m3[k] = v
    }
}
```

### 4. 結構化日誌 (log/slog, Go 1.21+)

#### 舊風格日誌

```go
import "log"

// 舊風格：非結構化日誌
func OldLogging() {
    log.Printf("User %s logged in from IP %s", username, ip)
    log.Printf("Error: %v", err)
}
```

#### 現代結構化日誌 (2025 推薦)

```go
import "log/slog"

// 2025 現代風格：結構化日誌
func ModernLogging() {
    // 設置日誌格式（JSON 或文本）
    logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
        Level: slog.LevelInfo,
        AddSource: true,
    }))

    // 結構化日誌
    logger.Info("用戶登入",
        slog.String("username", username),
        slog.String("ip", ip),
        slog.Int("attempt", 3),
    )

    // 錯誤日誌
    logger.Error("數據庫連接失敗",
        slog.String("database", "users"),
        slog.Any("error", err),
    )

    // 帶上下文的日誌
    logger = logger.With(
        slog.String("service", "auth"),
        slog.String("version", "1.0.0"),
    )

    // 所有日誌都會包含 service 和 version
    logger.Info("處理請求")
}
```

```javascript
// Node.js 對比：Winston, Pino 等結構化日誌庫
import winston from 'winston';

const logger = winston.createLogger({
    format: winston.format.json(),
    transports: [new winston.transports.Console()]
});

logger.info('用戶登入', {
    username: username,
    ip: ip,
    attempt: 3
});
```

### 5. HTTP 路由增強 (Go 1.22+)

#### 舊風格 (需要第三方路由庫)

```go
// Go 1.21 之前需要使用 gorilla/mux 或 chi
import "github.com/gorilla/mux"

func OldRouting() {
    r := mux.NewRouter()
    r.HandleFunc("/users/{id}", getUser).Methods("GET")
    r.HandleFunc("/users/{id}", updateUser).Methods("PUT")
    r.HandleFunc("/users/{id}", deleteUser).Methods("DELETE")

    http.ListenAndServe(":8080", r)
}
```

#### 現代風格 (Go 1.22+, 2025 推薦)

```go
// Go 1.22+ 內建路由方法和路徑參數
func ModernRouting() {
    mux := http.NewServeMux()

    // HTTP 方法路由
    mux.HandleFunc("GET /users/{id}", getUser)
    mux.HandleFunc("PUT /users/{id}", updateUser)
    mux.HandleFunc("DELETE /users/{id}", deleteUser)

    // 路徑參數提取
    http.ListenAndServe(":8080", mux)
}

func getUser(w http.ResponseWriter, r *http.Request) {
    id := r.PathValue("id")  // Go 1.22+ 新方法
    // ...
}
```

```javascript
// Express.js 對比：一直都有這個功能
app.get('/users/:id', getUser);
app.put('/users/:id', updateUser);
app.delete('/users/:id', deleteUser);

function getUser(req, res) {
    const id = req.params.id;
    // ...
}
```

### 6. Context 增強使用 (Go 1.21+)

```go
import "context"

// 2025 現代風格：WithoutCancel, WithDeadlineCause
func ModernContextUsage() {
    // 創建可取消的上下文
    ctx, cancel := context.WithCancelCause(context.Background())
    defer cancel(nil)

    go func() {
        time.Sleep(5 * time.Second)
        cancel(errors.New("操作超時"))
    }()

    if err := longRunningOperation(ctx); err != nil {
        // Go 1.20+ 可以獲取取消原因
        cause := context.Cause(ctx)
        log.Printf("操作失敗: %v (原因: %v)", err, cause)
    }
}

// WithoutCancel: 繼承值但不會被取消
func SpawnBackgroundTask(ctx context.Context) {
    // 即使父上下文取消，這個任務也會繼續
    bgCtx := context.WithoutCancel(ctx)

    go func() {
        // 使用父上下文的值，但不受其取消影響
        processBackgroundJob(bgCtx)
    }()
}
```

### 7. 現代項目結構 (2025 最佳實踐)

#### 舊風格項目結構 (2020 之前)

```
myapp/
├── main.go
├── handlers.go
├── models.go
├── db.go
└── utils.go
```

#### 現代項目結構 (2025 推薦)

```
myapp/
├── cmd/
│   └── api/
│       └── main.go          # 應用入口
├── internal/                # 私有代碼
│   ├── domain/              # 領域模型
│   │   ├── user.go
│   │   └── product.go
│   ├── handler/             # HTTP handlers
│   │   ├── user_handler.go
│   │   └── product_handler.go
│   ├── repository/          # 數據訪問層
│   │   ├── user_repo.go
│   │   └── product_repo.go
│   ├── service/             # 業務邏輯
│   │   ├── user_service.go
│   │   └── product_service.go
│   └── middleware/          # 中間件
│       ├── auth.go
│       └── logging.go
├── pkg/                     # 可被外部使用的庫
│   └── validator/
│       └── validator.go
├── api/                     # API 定義
│   └── openapi.yaml
├── configs/                 # 配置文件
│   └── config.yaml
├── migrations/              # 數據庫遷移
│   └── 001_initial.sql
├── scripts/                 # 腳本
│   └── seed.sh
├── go.mod
├── go.sum
├── Makefile
└── README.md
```

```
// Node.js 現代結構對比
myapp/
├── src/
│   ├── controllers/
│   ├── models/
│   ├── services/
│   ├── middleware/
│   └── routes/
├── config/
├── migrations/
├── package.json
└── tsconfig.json
```

### 8. 錯誤處理模式演進

#### 舊模式

```go
func OldErrorHandling() error {
    result, err := doSomething()
    if err != nil {
        return err
    }

    data, err := doAnotherThing(result)
    if err != nil {
        return err
    }

    err = saveData(data)
    if err != nil {
        return err
    }

    return nil
}
```

#### 現代模式 (2025)

```go
// 使用自定義錯誤類型
type AppError struct {
    Code    string
    Message string
    Err     error
}

func (e *AppError) Error() string {
    return fmt.Sprintf("%s: %s: %v", e.Code, e.Message, e.Err)
}

func (e *AppError) Unwrap() error {
    return e.Err
}

// 錯誤處理函數
func ModernErrorHandling() error {
    result, err := doSomething()
    if err != nil {
        return &AppError{
            Code:    "DO_SOMETHING_FAILED",
            Message: "執行某操作失敗",
            Err:     err,
        }
    }

    data, err := doAnotherThing(result)
    if err != nil {
        return &AppError{
            Code:    "PROCESS_FAILED",
            Message: "處理數據失敗",
            Err:     err,
        }
    }

    if err := saveData(data); err != nil {
        return &AppError{
            Code:    "SAVE_FAILED",
            Message: "保存數據失敗",
            Err:     err,
        }
    }

    return nil
}

// HTTP 處理中的錯誤
func HandleRequest(w http.ResponseWriter, r *http.Request) {
    if err := ModernErrorHandling(); err != nil {
        var appErr *AppError
        if errors.As(err, &appErr) {
            w.WriteHeader(http.StatusInternalServerError)
            json.NewEncoder(w).Encode(map[string]string{
                "error": appErr.Code,
                "message": appErr.Message,
            })
            return
        }

        w.WriteHeader(http.StatusInternalServerError)
        json.NewEncoder(w).Encode(map[string]string{
            "error": "INTERNAL_ERROR",
        })
    }
}
```

### 9. 測試風格演進

#### 舊風格測試

```go
func TestOldStyle(t *testing.T) {
    result := Add(2, 3)
    if result != 5 {
        t.Errorf("Expected 5, got %d", result)
    }
}
```

#### 現代測試風格 (2025)

```go
// 使用表格驅動測試 + testify
import (
    "testing"
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"
)

func TestModernStyle(t *testing.T) {
    tests := []struct {
        name     string
        a, b     int
        expected int
    }{
        {"positive numbers", 2, 3, 5},
        {"negative numbers", -2, -3, -5},
        {"mixed numbers", -2, 3, 1},
        {"zero", 0, 0, 0},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            // 並行測試
            t.Parallel()

            result := Add(tt.a, tt.b)
            assert.Equal(t, tt.expected, result)
        })
    }
}

// 使用 testify 的 suite
type UserServiceTestSuite struct {
    suite.Suite
    db      *sql.DB
    service *UserService
}

func (s *UserServiceTestSuite) SetupTest() {
    // 每個測試前設置
    s.db = setupTestDB()
    s.service = NewUserService(s.db)
}

func (s *UserServiceTestSuite) TearDownTest() {
    // 每個測試後清理
    s.db.Close()
}

func (s *UserServiceTestSuite) TestCreateUser() {
    user, err := s.service.Create("test@example.com")
    require.NoError(s.T(), err)
    assert.NotEmpty(s.T(), user.ID)
    assert.Equal(s.T(), "test@example.com", user.Email)
}

func TestUserServiceSuite(t *testing.T) {
    suite.Run(t, new(UserServiceTestSuite))
}
```

### 10. 並發模式現代化

#### 舊風格

```go
func OldConcurrency() {
    done := make(chan bool)

    go func() {
        // 做一些工作
        time.Sleep(time.Second)
        done <- true
    }()

    <-done
}
```

#### 現代風格 (2025)

```go
import (
    "context"
    "golang.org/x/sync/errgroup"
)

// 使用 errgroup 進行錯誤處理
func ModernConcurrency(ctx context.Context, urls []string) error {
    g, ctx := errgroup.WithContext(ctx)

    // 限制並發數
    g.SetLimit(10)

    for _, url := range urls {
        url := url  // Go 1.22+ 不需要這行了！
        g.Go(func() error {
            return fetchURL(ctx, url)
        })
    }

    // 等待所有 goroutine 完成
    return g.Wait()
}

// 使用 context 進行超時控制
func ModernTimeout() error {
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()

    return ModernConcurrency(ctx, urls)
}
```

## 2025 年 Go 最佳實踐總結

### ✅ 推薦使用

1. **泛型** - 使用泛型減少重複代碼
2. **slices/maps 包** - 使用標準庫新增的工具函數
3. **log/slog** - 使用結構化日誌
4. **錯誤包裝** - 使用 `fmt.Errorf` 的 `%w` 和 `errors.Join`
5. **Go 1.22+ 路由** - 標準庫足夠強大，減少依賴
6. **errgroup** - 處理並發錯誤
7. **testify** - 增強測試體驗
8. **清晰的項目結構** - 使用 cmd/internal/pkg 結構

### ❌ 避免使用

1. **ioutil 包** - 已廢棄，使用 os 和 io 包替代
2. **interface{}** - 能用泛型就用泛型
3. **手動 slice/map 操作** - 使用 slices/maps 包
4. **非結構化日誌** - 使用 slog
5. **忽略錯誤上下文** - 總是包裝錯誤

## 完整現代 Web API 示例 (2025 風格)

```go
package main

import (
    "context"
    "encoding/json"
    "log/slog"
    "net/http"
    "os"
    "os/signal"
    "time"
)

// 領域模型
type User struct {
    ID    string `json:"id"`
    Email string `json:"email"`
    Name  string `json:"name"`
}

// 自定義錯誤
type AppError struct {
    Code    int
    Message string
    Err     error
}

func (e *AppError) Error() string {
    return e.Message
}

// 使用泛型的響應包裝
type Response[T any] struct {
    Data  T      `json:"data,omitempty"`
    Error string `json:"error,omitempty"`
}

// 主函數
func main() {
    // 設置結構化日誌
    logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
        Level: slog.LevelInfo,
    }))
    slog.SetDefault(logger)

    // 創建路由 (Go 1.22+)
    mux := http.NewServeMux()
    mux.HandleFunc("GET /users/{id}", getUser)
    mux.HandleFunc("POST /users", createUser)
    mux.HandleFunc("PUT /users/{id}", updateUser)
    mux.HandleFunc("DELETE /users/{id}", deleteUser)

    // 包裝中間件
    handler := loggingMiddleware(mux)

    // 創建服務器
    server := &http.Server{
        Addr:         ":8080",
        Handler:      handler,
        ReadTimeout:  10 * time.Second,
        WriteTimeout: 10 * time.Second,
    }

    // 優雅關閉
    go func() {
        slog.Info("服務器啟動", slog.String("addr", server.Addr))
        if err := server.ListenAndServe(); err != http.ErrServerClosed {
            slog.Error("服務器錯誤", slog.Any("error", err))
            os.Exit(1)
        }
    }()

    // 等待中斷信號
    stop := make(chan os.Signal, 1)
    signal.Notify(stop, os.Interrupt)
    <-stop

    // 優雅關閉
    slog.Info("正在關閉服務器...")
    ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()

    if err := server.Shutdown(ctx); err != nil {
        slog.Error("服務器關閉失敗", slog.Any("error", err))
    }
    slog.Info("服務器已關閉")
}

// 日誌中間件
func loggingMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()

        next.ServeHTTP(w, r)

        slog.Info("請求處理完成",
            slog.String("method", r.Method),
            slog.String("path", r.URL.Path),
            slog.Duration("duration", time.Since(start)),
        )
    })
}

// Handler 函數
func getUser(w http.ResponseWriter, r *http.Request) {
    id := r.PathValue("id")  // Go 1.22+

    slog.Info("獲取用戶", slog.String("id", id))

    // 模擬數據庫查詢
    user := User{
        ID:    id,
        Email: "user@example.com",
        Name:  "Test User",
    }

    writeJSON(w, http.StatusOK, Response[User]{Data: user})
}

func createUser(w http.ResponseWriter, r *http.Request) {
    var user User
    if err := json.NewDecoder(r.Body).Decode(&user); err != nil {
        writeJSON(w, http.StatusBadRequest, Response[any]{
            Error: "無效的請求數據",
        })
        return
    }

    slog.Info("創建用戶",
        slog.String("email", user.Email),
        slog.String("name", user.Name),
    )

    user.ID = "generated-id"
    writeJSON(w, http.StatusCreated, Response[User]{Data: user})
}

func updateUser(w http.ResponseWriter, r *http.Request) {
    id := r.PathValue("id")

    var user User
    if err := json.NewDecoder(r.Body).Decode(&user); err != nil {
        writeJSON(w, http.StatusBadRequest, Response[any]{
            Error: "無效的請求數據",
        })
        return
    }

    user.ID = id
    slog.Info("更新用戶", slog.String("id", id))

    writeJSON(w, http.StatusOK, Response[User]{Data: user})
}

func deleteUser(w http.ResponseWriter, r *http.Request) {
    id := r.PathValue("id")

    slog.Info("刪除用戶", slog.String("id", id))

    w.WriteHeader(http.StatusNoContent)
}

// 泛型 JSON 響應函數
func writeJSON[T any](w http.ResponseWriter, status int, data T) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(status)
    json.NewEncoder(w).Encode(data)
}
```

## Node.js vs Go 演進對比總結

| 特性 | JavaScript/Node.js | Go |
|------|-------------------|-----|
| **異步處理** | Callback → Promise → async/await | Goroutine + Channel (一直保持) |
| **類型系統** | JavaScript → TypeScript | 無泛型 → 泛型 (Go 1.18+) |
| **模組系統** | CommonJS → ES Modules | Go modules (一直改進) |
| **錯誤處理** | try/catch (一直保持) | 簡單返回 → 錯誤包裝 → errors.Join |
| **日誌** | console.log → Winston/Pino | log → log/slog |
| **HTTP 路由** | Express (第三方) | 弱 → 增強 (Go 1.22+) |
| **測試** | 各種框架競爭 | testing + testify |
| **包管理** | npm → yarn → pnpm | go get → go modules |

## 重點總結

1. **Go 1.18 (泛型)** 是最大的語言變化，類似 TypeScript 對 JavaScript 的影響
2. **Go 1.21 (slices/maps/slog)** 讓標準庫更現代化
3. **Go 1.22 (HTTP 路由增強)** 減少了對第三方路由庫的依賴
4. **錯誤處理** 持續改進，從簡單返回到包裝再到多錯誤處理
5. **項目結構** 趨向標準化 (cmd/internal/pkg)
6. **測試** 更加結構化和易用

## 練習

1. 將一個舊的 Go 項目升級到使用泛型
2. 使用 slices/maps 包重構現有代碼
3. 將日誌系統遷移到 log/slog
4. 使用 Go 1.22+ 的新路由特性重寫 HTTP 服務器
5. 實現一個使用所有現代特性的完整 Web API

## 下一章預告

下一章我們將看到一個完整的 2025 年實際 Web 專案範例，展示所有這些現代特性如何在真實項目中協同工作。
