# Chapter 46: 單元測試 (testing 包)

## 概述

Go 內置了強大的測試框架 `testing` 包，無需安裝第三方庫即可編寫測試。與 Node.js 的 Jest 或 Mocha 不同，Go 的測試更加簡潔和標準化。

## Node.js vs Go 對比

### Jest/Mocha (Node.js)
```javascript
// math.test.js
const { add, divide } = require('./math');

describe('Math functions', () => {
  test('add should return sum of two numbers', () => {
    expect(add(2, 3)).toBe(5);
    expect(add(-1, 1)).toBe(0);
  });

  test('divide should return quotient', () => {
    expect(divide(10, 2)).toBe(5);
  });

  test('divide by zero should throw error', () => {
    expect(() => divide(10, 0)).toThrow('division by zero');
  });

  beforeEach(() => {
    // 每個測試前執行
  });

  afterEach(() => {
    // 每個測試後執行
  });
});
```

### testing (Go)
```go
// math_test.go
package math

import "testing"

func TestAdd(t *testing.T) {
    result := Add(2, 3)
    if result != 5 {
        t.Errorf("Add(2, 3) = %d; want 5", result)
    }

    result = Add(-1, 1)
    if result != 0 {
        t.Errorf("Add(-1, 1) = %d; want 0", result)
    }
}

func TestDivide(t *testing.T) {
    result, err := Divide(10, 2)
    if err != nil {
        t.Fatalf("unexpected error: %v", err)
    }
    if result != 5 {
        t.Errorf("Divide(10, 2) = %f; want 5", result)
    }
}

func TestDivideByZero(t *testing.T) {
    _, err := Divide(10, 0)
    if err == nil {
        t.Error("expected error for division by zero")
    }
}
```

## 1. 基礎測試

### 測試文件命名

```bash
# 測試文件必須以 _test.go 結尾
math.go       # 源文件
math_test.go  # 測試文件
```

### 測試函數規則

```go
package calculator

import "testing"

// 1. 測試函數必須以 Test 開頭
// 2. 接受一個參數 *testing.T
// 3. 函數名：Test + 要測試的功能（駝峰命名）

func TestAdd(t *testing.T) {
    // t.Error/t.Errorf - 標記失敗但繼續執行
    // t.Fatal/t.Fatalf - 標記失敗並立即停止

    result := Add(2, 3)
    expected := 5

    if result != expected {
        t.Errorf("Add(2, 3) = %d; want %d", result, expected)
    }
}

func TestSubtract(t *testing.T) {
    result := Subtract(5, 3)
    if result != 2 {
        t.Fatalf("Subtract(5, 3) = %d; want 2", result)
        // Fatal 後的代碼不會執行
        t.Log("這行不會執行")
    }
}
```

### 運行測試

```bash
# 運行當前目錄的所有測試
go test

# 運行並顯示詳細輸出
go test -v

# 運行特定測試
go test -run TestAdd

# 運行匹配模式的測試
go test -run Test.*Add

# 運行特定包的測試
go test ./calculator

# 運行所有包的測試
go test ./...

# 顯示覆蓋率
go test -cover

# 生成覆蓋率報告
go test -coverprofile=coverage.out
go tool cover -html=coverage.out
```

**Node.js 對比：**
```bash
# Jest
npm test
npm test -- --verbose
npm test -- --testNamePattern="add"
npm test -- --coverage

# Mocha
npx mocha
npx mocha --grep "add"
npx mocha --reporter spec
```

## 2. 斷言方法

### testing.T 常用方法

```go
package example

import "testing"

func TestAssertions(t *testing.T) {
    // 1. Error - 標記失敗，繼續執行
    if 2+2 != 4 {
        t.Error("Math is broken!")
    }

    // 2. Errorf - 格式化錯誤消息
    result := 2 + 2
    if result != 4 {
        t.Errorf("expected %d, got %d", 4, result)
    }

    // 3. Fatal - 標記失敗，立即停止測試
    if true {
        t.Fatal("Critical failure")
    }

    // 4. Fatalf - 格式化並停止
    t.Fatalf("Critical: expected %v, got %v", "A", "B")

    // 5. Log - 記錄信息（僅在失敗或 -v 時顯示）
    t.Log("This is a log message")

    // 6. Logf - 格式化日誌
    t.Logf("Value is %d", 42)

    // 7. Fail - 標記失敗但不輸出消息
    t.Fail()

    // 8. FailNow - 立即失敗並停止
    t.FailNow()

    // 9. Failed - 檢查是否已失敗
    if t.Failed() {
        t.Log("Test has failed")
    }

    // 10. Skip - 跳過測試
    if testing.Short() {
        t.Skip("Skipping in short mode")
    }

    // 11. Skipf - 格式化並跳過
    t.Skipf("Skipping because: %s", "reason")

    // 12. Helper - 標記為輔助函數
    checkValue := func(t *testing.T, got, want int) {
        t.Helper() // 錯誤會指向調用處而非這裡
        if got != want {
            t.Errorf("got %d, want %d", got, want)
        }
    }

    checkValue(t, 5, 5)
}
```

### 自定義斷言輔助函數

```go
package testutil

import (
    "reflect"
    "testing"
)

// 相等斷言
func AssertEqual(t *testing.T, got, want interface{}) {
    t.Helper()
    if !reflect.DeepEqual(got, want) {
        t.Errorf("got %v, want %v", got, want)
    }
}

// 不相等斷言
func AssertNotEqual(t *testing.T, got, want interface{}) {
    t.Helper()
    if reflect.DeepEqual(got, want) {
        t.Errorf("got %v, should not equal %v", got, want)
    }
}

// Nil 斷言
func AssertNil(t *testing.T, got interface{}) {
    t.Helper()
    if got != nil {
        t.Errorf("expected nil, got %v", got)
    }
}

// 錯誤斷言
func AssertNoError(t *testing.T, err error) {
    t.Helper()
    if err != nil {
        t.Fatalf("unexpected error: %v", err)
    }
}

func AssertError(t *testing.T, err error) {
    t.Helper()
    if err == nil {
        t.Fatal("expected error, got nil")
    }
}

// 使用示例
func TestWithHelpers(t *testing.T) {
    AssertEqual(t, 2+2, 4)
    AssertNotEqual(t, "hello", "world")

    err := someFunction()
    AssertNoError(t, err)
}
```

**Node.js 對比：**
```javascript
// Jest
expect(result).toBe(expected);
expect(result).toEqual(expected);
expect(result).not.toBe(value);
expect(error).toBeNull();
expect(fn).toThrow();

// Chai
expect(result).to.equal(expected);
expect(result).to.not.equal(value);
expect(error).to.be.null;
expect(fn).to.throw();
```

## 3. 子測試 (Subtests)

```go
package calculator

import "testing"

func TestMath(t *testing.T) {
    // 使用 t.Run 創建子測試
    t.Run("Addition", func(t *testing.T) {
        result := Add(2, 3)
        if result != 5 {
            t.Errorf("Add(2, 3) = %d; want 5", result)
        }
    })

    t.Run("Subtraction", func(t *testing.T) {
        result := Subtract(5, 3)
        if result != 2 {
            t.Errorf("Subtract(5, 3) = %d; want 2", result)
        }
    })

    t.Run("Division", func(t *testing.T) {
        t.Run("Normal", func(t *testing.T) {
            result, err := Divide(10, 2)
            if err != nil {
                t.Fatalf("unexpected error: %v", err)
            }
            if result != 5 {
                t.Errorf("Divide(10, 2) = %f; want 5", result)
            }
        })

        t.Run("ByZero", func(t *testing.T) {
            _, err := Divide(10, 0)
            if err == nil {
                t.Error("expected error for division by zero")
            }
        })
    })
}

// 運行特定子測試
// go test -run TestMath/Division/ByZero
```

**Node.js 對比：**
```javascript
// Jest
describe('Math', () => {
  describe('Addition', () => {
    test('adds two numbers', () => {
      expect(add(2, 3)).toBe(5);
    });
  });

  describe('Division', () => {
    test('divides normally', () => {
      expect(divide(10, 2)).toBe(5);
    });

    test('throws on division by zero', () => {
      expect(() => divide(10, 0)).toThrow();
    });
  });
});

// 運行特定測試
// npm test -- --testNamePattern="Division"
```

## 4. Setup 和 Teardown

### 測試前後操作

```go
package database

import (
    "testing"
)

// TestMain - 在所有測試前後執行
func TestMain(m *testing.M) {
    // 全局 Setup
    setup()

    // 運行所有測試
    code := m.Run()

    // 全局 Teardown
    teardown()

    // 退出
    os.Exit(code)
}

func setup() {
    // 初始化數據庫連接、創建測試表等
    fmt.Println("Global setup")
}

func teardown() {
    // 清理資源、刪除測試數據等
    fmt.Println("Global teardown")
}

// 每個測試的 Setup/Teardown
func TestUserRepository(t *testing.T) {
    // Setup
    db := setupTestDB(t)
    defer cleanupTestDB(t, db) // Teardown

    t.Run("Create", func(t *testing.T) {
        // 測試代碼
    })

    t.Run("Read", func(t *testing.T) {
        // 測試代碼
    })
}

func setupTestDB(t *testing.T) *sql.DB {
    t.Helper()
    db, err := sql.Open("sqlite3", ":memory:")
    if err != nil {
        t.Fatalf("failed to open db: %v", err)
    }

    // 創建表結構
    _, err = db.Exec(`CREATE TABLE users (id INTEGER PRIMARY KEY, name TEXT)`)
    if err != nil {
        t.Fatalf("failed to create table: %v", err)
    }

    return db
}

func cleanupTestDB(t *testing.T, db *sql.DB) {
    t.Helper()
    if err := db.Close(); err != nil {
        t.Errorf("failed to close db: %v", err)
    }
}
```

**Node.js 對比：**
```javascript
// Jest
beforeAll(() => {
  // 所有測試前執行一次
  setup();
});

afterAll(() => {
  // 所有測試後執行一次
  teardown();
});

beforeEach(() => {
  // 每個測試前執行
  db = setupTestDB();
});

afterEach(() => {
  // 每個測試後執行
  cleanupTestDB(db);
});

test('creates user', () => {
  // 測試代碼
});
```

## 5. 測試覆蓋率

```go
// calculator.go
package calculator

func Add(a, b int) int {
    return a + b
}

func Subtract(a, b int) int {
    return a - b
}

func Multiply(a, b int) int {
    if a == 0 || b == 0 {
        return 0
    }
    return a * b
}

// calculator_test.go
package calculator

import "testing"

func TestAdd(t *testing.T) {
    if Add(2, 3) != 5 {
        t.Error("Add failed")
    }
}

func TestSubtract(t *testing.T) {
    if Subtract(5, 3) != 2 {
        t.Error("Subtract failed")
    }
}

// Multiply 沒有測試！
```

```bash
# 查看覆蓋率
go test -cover
# PASS
# coverage: 66.7% of statements

# 生成詳細報告
go test -coverprofile=coverage.out
go tool cover -html=coverage.out

# 按函數查看覆蓋率
go test -coverprofile=coverage.out -covermode=count
go tool cover -func=coverage.out
# calculator.go:3:  Add       100.0%
# calculator.go:7:  Subtract  100.0%
# calculator.go:11: Multiply  0.0%
# total:            66.7%
```

**Node.js 對比：**
```bash
# Jest
npm test -- --coverage

# 查看詳細報告
npm test -- --coverage --coverageReporters=html
open coverage/index.html

# 配置覆蓋率閾值
# package.json
{
  "jest": {
    "coverageThreshold": {
      "global": {
        "branches": 80,
        "functions": 80,
        "lines": 80,
        "statements": 80
      }
    }
  }
}
```

## 6. 完整示例：用戶服務測試

```go
// user.go
package user

import (
    "errors"
    "regexp"
)

var (
    ErrInvalidEmail = errors.New("invalid email")
    ErrUserNotFound = errors.New("user not found")
)

type User struct {
    ID    int
    Name  string
    Email string
    Age   int
}

type UserService struct {
    users map[int]*User
    nextID int
}

func NewUserService() *UserService {
    return &UserService{
        users: make(map[int]*User),
        nextID: 1,
    }
}

func (s *UserService) Create(name, email string, age int) (*User, error) {
    if !isValidEmail(email) {
        return nil, ErrInvalidEmail
    }

    user := &User{
        ID:    s.nextID,
        Name:  name,
        Email: email,
        Age:   age,
    }

    s.users[s.nextID] = user
    s.nextID++

    return user, nil
}

func (s *UserService) Get(id int) (*User, error) {
    user, exists := s.users[id]
    if !exists {
        return nil, ErrUserNotFound
    }
    return user, nil
}

func (s *UserService) Update(id int, name, email string, age int) error {
    user, err := s.Get(id)
    if err != nil {
        return err
    }

    if !isValidEmail(email) {
        return ErrInvalidEmail
    }

    user.Name = name
    user.Email = email
    user.Age = age

    return nil
}

func (s *UserService) Delete(id int) error {
    if _, exists := s.users[id]; !exists {
        return ErrUserNotFound
    }
    delete(s.users, id)
    return nil
}

func (s *UserService) List() []*User {
    users := make([]*User, 0, len(s.users))
    for _, user := range s.users {
        users = append(users, user)
    }
    return users
}

func isValidEmail(email string) bool {
    regex := regexp.MustCompile(`^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$`)
    return regex.MatchString(email)
}
```

```go
// user_test.go
package user

import (
    "testing"
)

func TestUserService(t *testing.T) {
    t.Run("Create", func(t *testing.T) {
        service := NewUserService()

        t.Run("ValidUser", func(t *testing.T) {
            user, err := service.Create("Alice", "alice@example.com", 25)

            if err != nil {
                t.Fatalf("unexpected error: %v", err)
            }

            if user.ID != 1 {
                t.Errorf("expected ID 1, got %d", user.ID)
            }

            if user.Name != "Alice" {
                t.Errorf("expected name Alice, got %s", user.Name)
            }

            if user.Email != "alice@example.com" {
                t.Errorf("expected email alice@example.com, got %s", user.Email)
            }

            if user.Age != 25 {
                t.Errorf("expected age 25, got %d", user.Age)
            }
        })

        t.Run("InvalidEmail", func(t *testing.T) {
            _, err := service.Create("Bob", "invalid-email", 30)

            if err != ErrInvalidEmail {
                t.Errorf("expected ErrInvalidEmail, got %v", err)
            }
        })
    })

    t.Run("Get", func(t *testing.T) {
        service := NewUserService()
        created, _ := service.Create("Alice", "alice@example.com", 25)

        t.Run("ExistingUser", func(t *testing.T) {
            user, err := service.Get(created.ID)

            if err != nil {
                t.Fatalf("unexpected error: %v", err)
            }

            if user.ID != created.ID {
                t.Errorf("expected ID %d, got %d", created.ID, user.ID)
            }
        })

        t.Run("NonExistingUser", func(t *testing.T) {
            _, err := service.Get(999)

            if err != ErrUserNotFound {
                t.Errorf("expected ErrUserNotFound, got %v", err)
            }
        })
    })

    t.Run("Update", func(t *testing.T) {
        service := NewUserService()
        user, _ := service.Create("Alice", "alice@example.com", 25)

        t.Run("ValidUpdate", func(t *testing.T) {
            err := service.Update(user.ID, "Alice Smith", "alice.smith@example.com", 26)

            if err != nil {
                t.Fatalf("unexpected error: %v", err)
            }

            updated, _ := service.Get(user.ID)
            if updated.Name != "Alice Smith" {
                t.Errorf("expected name Alice Smith, got %s", updated.Name)
            }

            if updated.Age != 26 {
                t.Errorf("expected age 26, got %d", updated.Age)
            }
        })

        t.Run("InvalidEmail", func(t *testing.T) {
            err := service.Update(user.ID, "Alice", "invalid", 25)

            if err != ErrInvalidEmail {
                t.Errorf("expected ErrInvalidEmail, got %v", err)
            }
        })

        t.Run("NonExistingUser", func(t *testing.T) {
            err := service.Update(999, "Test", "test@example.com", 20)

            if err != ErrUserNotFound {
                t.Errorf("expected ErrUserNotFound, got %v", err)
            }
        })
    })

    t.Run("Delete", func(t *testing.T) {
        service := NewUserService()
        user, _ := service.Create("Alice", "alice@example.com", 25)

        t.Run("ExistingUser", func(t *testing.T) {
            err := service.Delete(user.ID)

            if err != nil {
                t.Fatalf("unexpected error: %v", err)
            }

            _, err = service.Get(user.ID)
            if err != ErrUserNotFound {
                t.Error("user should be deleted")
            }
        })

        t.Run("NonExistingUser", func(t *testing.T) {
            err := service.Delete(999)

            if err != ErrUserNotFound {
                t.Errorf("expected ErrUserNotFound, got %v", err)
            }
        })
    })

    t.Run("List", func(t *testing.T) {
        service := NewUserService()
        service.Create("Alice", "alice@example.com", 25)
        service.Create("Bob", "bob@example.com", 30)
        service.Create("Charlie", "charlie@example.com", 35)

        users := service.List()

        if len(users) != 3 {
            t.Errorf("expected 3 users, got %d", len(users))
        }
    })
}

func TestEmailValidation(t *testing.T) {
    tests := []struct {
        email string
        valid bool
    }{
        {"alice@example.com", true},
        {"bob.smith@company.co.uk", true},
        {"test+tag@domain.com", true},
        {"invalid", false},
        {"@example.com", false},
        {"user@", false},
        {"", false},
    }

    for _, tt := range tests {
        t.Run(tt.email, func(t *testing.T) {
            result := isValidEmail(tt.email)
            if result != tt.valid {
                t.Errorf("isValidEmail(%q) = %v; want %v", tt.email, result, tt.valid)
            }
        })
    }
}
```

## 重點總結

### Go vs Jest/Mocha

| 特性 | Go testing | Jest/Mocha |
|------|------------|------------|
| **安裝** | 內置 | 需要安裝 |
| **斷言** | 手動比較 | expect/assert |
| **子測試** | t.Run | describe/it |
| **Setup** | TestMain/defer | beforeEach/afterEach |
| **覆蓋率** | go test -cover | --coverage |
| **運行** | go test | npm test |

### 最佳實踐

1. **測試文件命名**：`xxx_test.go`
2. **測試函數命名**：`TestXxx`
3. **使用子測試**組織相關測試
4. **使用 t.Helper()**標記輔助函數
5. **檢查覆蓋率**：目標 80% 以上
6. **使用 TestMain** 進行全局 setup/teardown

## 練習題

1. **基礎**：為一個字符串工具函數編寫測試
2. **中級**：為一個 Stack 數據結構編寫完整測試
3. **高級**：為一個 HTTP Handler 編寫測試（使用 httptest）
4. **實戰**：為一個完整的 CRUD 服務編寫測試，包含邊界情況

## 下一步

下一章將學習基準測試（Benchmarking），了解如何測試和優化代碼性能。
