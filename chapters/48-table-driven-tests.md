# Chapter 48: 表格驅動測試 (Table-Driven Tests)

## 概述

表格驅動測試是 Go 社區廣泛採用的測試模式，通過定義測試用例表格來組織測試數據。這種方法簡潔、易於維護，並且能夠清晰地展示各種測試場景。

## Node.js vs Go 對比

### Jest each (Node.js)
```javascript
// Node.js - Jest
describe('Math operations', () => {
  test.each([
    { a: 1, b: 1, expected: 2 },
    { a: 2, b: 2, expected: 4 },
    { a: -1, b: 1, expected: 0 },
    { a: 0, b: 0, expected: 0 }
  ])('add($a, $b) should return $expected', ({ a, b, expected }) => {
    expect(add(a, b)).toBe(expected);
  });

  test.each([
    [1, 1, 2],
    [2, 2, 4],
    [-1, 1, 0],
    [0, 0, 0]
  ])('add(%i, %i) should return %i', (a, b, expected) => {
    expect(add(a, b)).toBe(expected);
  });
});
```

### Table-Driven Tests (Go)
```go
// Go - Table-Driven Tests
package math

import "testing"

func TestAdd(t *testing.T) {
    tests := []struct {
        name     string
        a        int
        b        int
        expected int
    }{
        {"positive numbers", 1, 1, 2},
        {"same numbers", 2, 2, 4},
        {"negative and positive", -1, 1, 0},
        {"zeros", 0, 0, 0},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            result := Add(tt.a, tt.b)
            if result != tt.expected {
                t.Errorf("Add(%d, %d) = %d; want %d",
                    tt.a, tt.b, result, tt.expected)
            }
        })
    }
}
```

## 1. 基礎表格驅動測試

### 簡單示例

```go
package strings

import "testing"

// 被測試的函數
func Reverse(s string) string {
    runes := []rune(s)
    for i, j := 0, len(runes)-1; i < j; i, j = i+1, j-1 {
        runes[i], runes[j] = runes[j], runes[i]
    }
    return string(runes)
}

// 表格驅動測試
func TestReverse(t *testing.T) {
    // 定義測試用例表格
    tests := []struct {
        name     string // 測試用例名稱
        input    string // 輸入
        expected string // 期望輸出
    }{
        {"empty string", "", ""},
        {"single character", "a", "a"},
        {"two characters", "ab", "ba"},
        {"palindrome", "aba", "aba"},
        {"normal string", "hello", "olleh"},
        {"unicode", "你好", "好你"},
        {"mixed", "a1b2c3", "3c2b1a"},
    }

    // 遍歷測試用例
    for _, tt := range tests {
        // 使用 t.Run 創建子測試
        t.Run(tt.name, func(t *testing.T) {
            result := Reverse(tt.input)
            if result != tt.expected {
                t.Errorf("Reverse(%q) = %q; want %q",
                    tt.input, result, tt.expected)
            }
        })
    }
}

// 運行特定測試：
// go test -run TestReverse/unicode
```

### 不使用子測試的版本

```go
func TestReverseSimple(t *testing.T) {
    tests := []struct {
        input    string
        expected string
    }{
        {"", ""},
        {"a", "a"},
        {"ab", "ba"},
        {"hello", "olleh"},
    }

    for i, tt := range tests {
        result := Reverse(tt.input)
        if result != tt.expected {
            t.Errorf("Test %d: Reverse(%q) = %q; want %q",
                i, tt.input, result, tt.expected)
        }
    }
}
```

**Node.js 對比：**
```javascript
// Jest
describe('Reverse', () => {
  test.each([
    { input: '', expected: '' },
    { input: 'a', expected: 'a' },
    { input: 'ab', expected: 'ba' },
    { input: 'hello', expected: 'olleh' }
  ])('reverse($input) should return $expected', ({ input, expected }) => {
    expect(reverse(input)).toBe(expected);
  });
});
```

## 2. 帶錯誤處理的表格測試

```go
package calculator

import (
    "errors"
    "testing"
)

var ErrDivisionByZero = errors.New("division by zero")

func Divide(a, b float64) (float64, error) {
    if b == 0 {
        return 0, ErrDivisionByZero
    }
    return a / b, nil
}

func TestDivide(t *testing.T) {
    tests := []struct {
        name      string
        a         float64
        b         float64
        want      float64
        wantErr   bool
        checkErr  func(error) bool // 可選：檢查特定錯誤
    }{
        {
            name:    "normal division",
            a:       10,
            b:       2,
            want:    5,
            wantErr: false,
        },
        {
            name:    "negative numbers",
            a:       -10,
            b:       2,
            want:    -5,
            wantErr: false,
        },
        {
            name:    "division by zero",
            a:       10,
            b:       0,
            want:    0,
            wantErr: true,
            checkErr: func(err error) bool {
                return errors.Is(err, ErrDivisionByZero)
            },
        },
        {
            name:    "zero divided by number",
            a:       0,
            b:       5,
            want:    0,
            wantErr: false,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got, err := Divide(tt.a, tt.b)

            // 檢查錯誤
            if tt.wantErr {
                if err == nil {
                    t.Errorf("Divide(%f, %f) expected error, got nil",
                        tt.a, tt.b)
                    return
                }
                // 如果提供了錯誤檢查函數
                if tt.checkErr != nil && !tt.checkErr(err) {
                    t.Errorf("Divide(%f, %f) got unexpected error: %v",
                        tt.a, tt.b, err)
                }
                return
            }

            // 不期望錯誤
            if err != nil {
                t.Errorf("Divide(%f, %f) unexpected error: %v",
                    tt.a, tt.b, err)
                return
            }

            // 檢查結果
            if got != tt.want {
                t.Errorf("Divide(%f, %f) = %f; want %f",
                    tt.a, tt.b, got, tt.want)
            }
        })
    }
}
```

**Node.js 對比：**
```javascript
describe('Divide', () => {
  test.each([
    { a: 10, b: 2, expected: 5, shouldThrow: false },
    { a: -10, b: 2, expected: -5, shouldThrow: false },
    { a: 10, b: 0, expected: null, shouldThrow: true },
    { a: 0, b: 5, expected: 0, shouldThrow: false }
  ])('divide($a, $b)', ({ a, b, expected, shouldThrow }) => {
    if (shouldThrow) {
      expect(() => divide(a, b)).toThrow('division by zero');
    } else {
      expect(divide(a, b)).toBe(expected);
    }
  });
});
```

## 3. 複雜數據類型測試

### 測試結構體

```go
package user

import (
    "reflect"
    "testing"
)

type User struct {
    ID    int
    Name  string
    Email string
    Age   int
}

func ValidateUser(u User) []string {
    var errors []string

    if u.Name == "" {
        errors = append(errors, "name is required")
    }

    if u.Email == "" {
        errors = append(errors, "email is required")
    }

    if u.Age < 0 || u.Age > 150 {
        errors = append(errors, "age must be between 0 and 150")
    }

    return errors
}

func TestValidateUser(t *testing.T) {
    tests := []struct {
        name          string
        user          User
        expectedErrors []string
    }{
        {
            name: "valid user",
            user: User{
                ID:    1,
                Name:  "Alice",
                Email: "alice@example.com",
                Age:   25,
            },
            expectedErrors: nil,
        },
        {
            name: "missing name",
            user: User{
                ID:    2,
                Email: "bob@example.com",
                Age:   30,
            },
            expectedErrors: []string{"name is required"},
        },
        {
            name: "missing email",
            user: User{
                ID:   3,
                Name: "Charlie",
                Age:  35,
            },
            expectedErrors: []string{"email is required"},
        },
        {
            name: "invalid age",
            user: User{
                ID:    4,
                Name:  "David",
                Email: "david@example.com",
                Age:   200,
            },
            expectedErrors: []string{"age must be between 0 and 150"},
        },
        {
            name: "multiple errors",
            user: User{
                ID:  5,
                Age: -1,
            },
            expectedErrors: []string{
                "name is required",
                "email is required",
                "age must be between 0 and 150",
            },
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            errors := ValidateUser(tt.user)

            if !reflect.DeepEqual(errors, tt.expectedErrors) {
                t.Errorf("ValidateUser() = %v; want %v",
                    errors, tt.expectedErrors)
            }
        })
    }
}
```

### 測試切片和映射

```go
package collection

import (
    "reflect"
    "testing"
)

func Filter(nums []int, predicate func(int) bool) []int {
    var result []int
    for _, n := range nums {
        if predicate(n) {
            result = append(result, n)
        }
    }
    return result
}

func TestFilter(t *testing.T) {
    tests := []struct {
        name      string
        input     []int
        predicate func(int) bool
        expected  []int
    }{
        {
            name:      "filter even numbers",
            input:     []int{1, 2, 3, 4, 5, 6},
            predicate: func(n int) bool { return n%2 == 0 },
            expected:  []int{2, 4, 6},
        },
        {
            name:      "filter greater than 10",
            input:     []int{5, 10, 15, 20},
            predicate: func(n int) bool { return n > 10 },
            expected:  []int{15, 20},
        },
        {
            name:      "filter all",
            input:     []int{1, 2, 3},
            predicate: func(n int) bool { return true },
            expected:  []int{1, 2, 3},
        },
        {
            name:      "filter none",
            input:     []int{1, 2, 3},
            predicate: func(n int) bool { return false },
            expected:  nil,
        },
        {
            name:      "empty input",
            input:     []int{},
            predicate: func(n int) bool { return true },
            expected:  nil,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            result := Filter(tt.input, tt.predicate)

            if !reflect.DeepEqual(result, tt.expected) {
                t.Errorf("Filter() = %v; want %v", result, tt.expected)
            }
        })
    }
}
```

## 4. 高級表格測試模式

### 使用輔助函數

```go
package advanced

import (
    "testing"
)

// 定義測試輔助函數
func assertEqual(t *testing.T, got, want interface{}) {
    t.Helper()
    if !reflect.DeepEqual(got, want) {
        t.Errorf("got %v, want %v", got, want)
    }
}

func assertError(t *testing.T, err error, wantErr bool) {
    t.Helper()
    if (err != nil) != wantErr {
        t.Errorf("error = %v, wantErr %v", err, wantErr)
    }
}

func TestWithHelpers(t *testing.T) {
    tests := []struct {
        name    string
        input   int
        want    int
        wantErr bool
    }{
        {"positive", 5, 10, false},
        {"zero", 0, 0, false},
        {"negative", -5, 0, true},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got, err := process(tt.input)
            assertError(t, err, tt.wantErr)
            if !tt.wantErr {
                assertEqual(t, got, tt.want)
            }
        })
    }
}
```

### 並行測試

```go
package parallel

import (
    "testing"
    "time"
)

func TestParallel(t *testing.T) {
    tests := []struct {
        name  string
        input int
        want  int
    }{
        {"test1", 1, 2},
        {"test2", 2, 4},
        {"test3", 3, 6},
        {"test4", 4, 8},
    }

    for _, tt := range tests {
        tt := tt // 捕獲範圍變量
        t.Run(tt.name, func(t *testing.T) {
            t.Parallel() // 並行運行

            time.Sleep(100 * time.Millisecond) // 模擬耗時操作
            result := tt.input * 2

            if result != tt.want {
                t.Errorf("got %d, want %d", result, tt.want)
            }
        })
    }
}
```

### 生成測試數據

```go
package generator

import (
    "fmt"
    "testing"
)

func TestGenerated(t *testing.T) {
    // 生成測試用例
    var tests []struct {
        name  string
        input int
        want  int
    }

    for i := 0; i < 100; i++ {
        tests = append(tests, struct {
            name  string
            input int
            want  int
        }{
            name:  fmt.Sprintf("input_%d", i),
            input: i,
            want:  i * 2,
        })
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            result := double(tt.input)
            if result != tt.want {
                t.Errorf("double(%d) = %d; want %d",
                    tt.input, result, tt.want)
            }
        })
    }
}
```

## 5. 完整示例：HTTP Handler 測試

```go
// handler.go
package api

import (
    "encoding/json"
    "net/http"
    "strconv"
)

type UserHandler struct {
    users map[int]User
}

type User struct {
    ID   int    `json:"id"`
    Name string `json:"name"`
}

func (h *UserHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    switch r.Method {
    case http.MethodGet:
        h.getUser(w, r)
    case http.MethodPost:
        h.createUser(w, r)
    default:
        http.Error(w, "Method not allowed", http.StatusMethodNotAllowed)
    }
}

func (h *UserHandler) getUser(w http.ResponseWriter, r *http.Request) {
    idStr := r.URL.Query().Get("id")
    if idStr == "" {
        http.Error(w, "id required", http.StatusBadRequest)
        return
    }

    id, err := strconv.Atoi(idStr)
    if err != nil {
        http.Error(w, "invalid id", http.StatusBadRequest)
        return
    }

    user, exists := h.users[id]
    if !exists {
        http.Error(w, "user not found", http.StatusNotFound)
        return
    }

    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(user)
}

func (h *UserHandler) createUser(w http.ResponseWriter, r *http.Request) {
    var user User
    if err := json.NewDecoder(r.Body).Decode(&user); err != nil {
        http.Error(w, "invalid request body", http.StatusBadRequest)
        return
    }

    if user.Name == "" {
        http.Error(w, "name required", http.StatusBadRequest)
        return
    }

    user.ID = len(h.users) + 1
    h.users[user.ID] = user

    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusCreated)
    json.NewEncoder(w).Encode(user)
}
```

```go
// handler_test.go
package api

import (
    "bytes"
    "encoding/json"
    "net/http"
    "net/http/httptest"
    "testing"
)

func TestUserHandler(t *testing.T) {
    tests := []struct {
        name           string
        method         string
        path           string
        body           interface{}
        setupHandler   func() *UserHandler
        expectedStatus int
        expectedBody   interface{}
    }{
        {
            name:   "GET existing user",
            method: http.MethodGet,
            path:   "/?id=1",
            setupHandler: func() *UserHandler {
                return &UserHandler{
                    users: map[int]User{
                        1: {ID: 1, Name: "Alice"},
                    },
                }
            },
            expectedStatus: http.StatusOK,
            expectedBody:   User{ID: 1, Name: "Alice"},
        },
        {
            name:   "GET non-existing user",
            method: http.MethodGet,
            path:   "/?id=999",
            setupHandler: func() *UserHandler {
                return &UserHandler{users: make(map[int]User)}
            },
            expectedStatus: http.StatusNotFound,
        },
        {
            name:   "GET missing id",
            method: http.MethodGet,
            path:   "/",
            setupHandler: func() *UserHandler {
                return &UserHandler{users: make(map[int]User)}
            },
            expectedStatus: http.StatusBadRequest,
        },
        {
            name:   "GET invalid id",
            method: http.MethodGet,
            path:   "/?id=abc",
            setupHandler: func() *UserHandler {
                return &UserHandler{users: make(map[int]User)}
            },
            expectedStatus: http.StatusBadRequest,
        },
        {
            name:   "POST create user",
            method: http.MethodPost,
            path:   "/",
            body:   User{Name: "Bob"},
            setupHandler: func() *UserHandler {
                return &UserHandler{users: make(map[int]User)}
            },
            expectedStatus: http.StatusCreated,
            expectedBody:   User{ID: 1, Name: "Bob"},
        },
        {
            name:   "POST missing name",
            method: http.MethodPost,
            path:   "/",
            body:   User{},
            setupHandler: func() *UserHandler {
                return &UserHandler{users: make(map[int]User)}
            },
            expectedStatus: http.StatusBadRequest,
        },
        {
            name:   "invalid method",
            method: http.MethodPut,
            path:   "/",
            setupHandler: func() *UserHandler {
                return &UserHandler{users: make(map[int]User)}
            },
            expectedStatus: http.StatusMethodNotAllowed,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            // Setup
            handler := tt.setupHandler()

            // 創建請求
            var req *http.Request
            if tt.body != nil {
                bodyBytes, _ := json.Marshal(tt.body)
                req = httptest.NewRequest(tt.method, tt.path, bytes.NewBuffer(bodyBytes))
                req.Header.Set("Content-Type", "application/json")
            } else {
                req = httptest.NewRequest(tt.method, tt.path, nil)
            }

            // 創建響應記錄器
            rr := httptest.NewRecorder()

            // 執行請求
            handler.ServeHTTP(rr, req)

            // 檢查狀態碼
            if rr.Code != tt.expectedStatus {
                t.Errorf("status = %d; want %d", rr.Code, tt.expectedStatus)
            }

            // 檢查響應體（如果期望有響應體）
            if tt.expectedBody != nil {
                var got User
                json.NewDecoder(rr.Body).Decode(&got)

                expected := tt.expectedBody.(User)
                if got.ID != expected.ID || got.Name != expected.Name {
                    t.Errorf("body = %+v; want %+v", got, expected)
                }
            }
        })
    }
}
```

**Node.js 對比：**
```javascript
// Jest + supertest
const request = require('supertest');
const app = require('./app');

describe('User API', () => {
  test.each([
    {
      name: 'GET existing user',
      method: 'get',
      path: '/users/1',
      expectedStatus: 200,
      expectedBody: { id: 1, name: 'Alice' }
    },
    {
      name: 'GET non-existing user',
      method: 'get',
      path: '/users/999',
      expectedStatus: 404
    },
    {
      name: 'POST create user',
      method: 'post',
      path: '/users',
      body: { name: 'Bob' },
      expectedStatus: 201,
      expectedBody: { id: 2, name: 'Bob' }
    }
  ])('$name', async ({ method, path, body, expectedStatus, expectedBody }) => {
    const res = await request(app)[method](path).send(body);

    expect(res.status).toBe(expectedStatus);
    if (expectedBody) {
      expect(res.body).toEqual(expectedBody);
    }
  });
});
```

## 6. 最佳實踐

### 測試用例組織

```go
// 好的做法：使用有意義的測試名稱
tests := []struct {
    name     string
    input    int
    expected int
}{
    {"zero", 0, 0},
    {"positive", 5, 10},
    {"negative", -5, -10},
}

// 不好的做法：使用索引
tests := []struct {
    input    int
    expected int
}{
    {0, 0},
    {5, 10},
    {-5, -10},
}
```

### 邊界情況覆蓋

```go
func TestEdgeCases(t *testing.T) {
    tests := []struct {
        name  string
        input string
        want  string
    }{
        {"empty string", "", ""},
        {"single char", "a", "A"},
        {"whitespace", "   ", "   "},
        {"special chars", "!@#", "!@#"},
        {"unicode", "你好", "你好"},
        {"max length", strings.Repeat("a", 1000), strings.Repeat("A", 1000)},
        {"nil-like", "\x00", "\x00"},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got := process(tt.input)
            if got != tt.want {
                t.Errorf("got %q, want %q", got, tt.want)
            }
        })
    }
}
```

### 使用常量和變量

```go
func TestConstants(t *testing.T) {
    const (
        validEmail   = "test@example.com"
        invalidEmail = "invalid-email"
        emptyString  = ""
    )

    tests := []struct {
        name  string
        email string
        valid bool
    }{
        {"valid email", validEmail, true},
        {"invalid email", invalidEmail, false},
        {"empty email", emptyString, false},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            result := isValidEmail(tt.email)
            if result != tt.valid {
                t.Errorf("isValidEmail(%q) = %v; want %v",
                    tt.email, result, tt.valid)
            }
        })
    }
}
```

## 重點總結

### 表格驅動測試優勢

1. **易於添加測試用例**：只需添加一行
2. **清晰展示所有場景**：一目了然
3. **減少代碼重複**：測試邏輯只寫一次
4. **便於維護**：修改測試邏輯只需改一處
5. **可讀性高**：測試數據與邏輯分離

### 最佳實踐

1. **總是使用 t.Run** 創建子測試
2. **提供有意義的測試名稱**
3. **覆蓋邊界情況**（空值、零值、極值）
4. **測試錯誤情況**
5. **使用輔助函數** 簡化斷言
6. **考慮並行測試** 提高速度

### 表格結構設計

```go
// 推薦的表格結構
tests := []struct {
    name     string    // 測試用例名稱
    input    InputType // 輸入參數
    want     WantType  // 期望輸出
    wantErr  bool      // 是否期望錯誤
}{
    // 測試用例...
}
```

## 練習題

1. **基礎**：為字符串處理函數編寫表格驅動測試
2. **中級**：為 JSON 序列化/反序列化編寫表格驅動測試
3. **高級**：為複雜業務邏輯編寫表格驅動測試，包含多種錯誤情況
4. **實戰**：為一個完整的 REST API 編寫表格驅動測試

## 下一步

下一章將學習 Mock 和測試替身，了解如何隔離依賴進行單元測試。
