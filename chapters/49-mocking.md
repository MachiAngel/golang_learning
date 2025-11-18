# Chapter 49: Mock 與測試替身

## 概述

在單元測試中，我們經常需要隔離外部依賴（如數據庫、API、文件系統）。Go 通過接口實現依賴注入和 Mock，常用的 Mock 庫有 `gomock`、`testify/mock`。與 Node.js 的 Sinon 或 Jest.mock 相比，Go 的 Mock 更依賴接口設計。

## Node.js vs Go 對比

### Sinon/Jest (Node.js)
```javascript
// Node.js - Sinon
const sinon = require('sinon');

// Mock 函數
const mock = sinon.stub(userService, 'getUser');
mock.withArgs(1).returns({ id: 1, name: 'Alice' });

// Jest Mock
jest.mock('./userService');
userService.getUser.mockReturnValue({ id: 1, name: 'Alice' });

// Spy
const spy = sinon.spy(console, 'log');
console.log('test');
expect(spy.calledOnce).toBe(true);

// 恢復
mock.restore();
spy.restore();
```

### gomock/testify (Go)
```go
// Go - gomock
import (
    "github.com/golang/mock/gomock"
)

// 創建 Mock 控制器
ctrl := gomock.NewController(t)
defer ctrl.Finish()

// 創建 Mock 對象
mockService := NewMockUserService(ctrl)

// 設置期望
mockService.EXPECT().
    GetUser(1).
    Return(&User{ID: 1, Name: "Alice"}, nil)

// 使用 Mock
user, err := mockService.GetUser(1)
```

## 1. 手動 Mock（使用接口）

### 基礎 Mock

```go
// user.go
package user

type User struct {
    ID   int
    Name string
}

// 定義接口
type UserRepository interface {
    Get(id int) (*User, error)
    Create(user *User) error
    Update(user *User) error
    Delete(id int) error
}

type UserService struct {
    repo UserRepository
}

func NewUserService(repo UserRepository) *UserService {
    return &UserService{repo: repo}
}

func (s *UserService) GetUser(id int) (*User, error) {
    return s.repo.Get(id)
}

func (s *UserService) CreateUser(name string) (*User, error) {
    user := &User{Name: name}
    err := s.repo.Create(user)
    return user, err
}
```

```go
// user_test.go
package user

import (
    "errors"
    "testing"
)

// 手動實現 Mock
type MockUserRepository struct {
    users map[int]*User
    nextID int

    // 用於驗證調用
    getCalled    bool
    createCalled bool
    getID        int
    createUser   *User
}

func NewMockUserRepository() *MockUserRepository {
    return &MockUserRepository{
        users: make(map[int]*User),
        nextID: 1,
    }
}

func (m *MockUserRepository) Get(id int) (*User, error) {
    m.getCalled = true
    m.getID = id

    user, exists := m.users[id]
    if !exists {
        return nil, errors.New("user not found")
    }
    return user, nil
}

func (m *MockUserRepository) Create(user *User) error {
    m.createCalled = true
    m.createUser = user

    user.ID = m.nextID
    m.users[m.nextID] = user
    m.nextID++
    return nil
}

func (m *MockUserRepository) Update(user *User) error {
    m.users[user.ID] = user
    return nil
}

func (m *MockUserRepository) Delete(id int) error {
    delete(m.users, id)
    return nil
}

// 測試
func TestUserService_GetUser(t *testing.T) {
    // 創建 Mock
    mockRepo := NewMockUserRepository()
    mockRepo.users[1] = &User{ID: 1, Name: "Alice"}

    // 創建 Service
    service := NewUserService(mockRepo)

    // 執行測試
    user, err := service.GetUser(1)

    // 驗證結果
    if err != nil {
        t.Fatalf("unexpected error: %v", err)
    }

    if user.Name != "Alice" {
        t.Errorf("expected name Alice, got %s", user.Name)
    }

    // 驗證 Mock 被調用
    if !mockRepo.getCalled {
        t.Error("Get was not called")
    }

    if mockRepo.getID != 1 {
        t.Errorf("Get called with %d, expected 1", mockRepo.getID)
    }
}

func TestUserService_CreateUser(t *testing.T) {
    mockRepo := NewMockUserRepository()
    service := NewUserService(mockRepo)

    user, err := service.CreateUser("Bob")

    if err != nil {
        t.Fatalf("unexpected error: %v", err)
    }

    if user.ID != 1 {
        t.Errorf("expected ID 1, got %d", user.ID)
    }

    if !mockRepo.createCalled {
        t.Error("Create was not called")
    }
}
```

**Node.js 對比：**
```javascript
// Jest
const userService = require('./userService');
const userRepo = require('./userRepository');

jest.mock('./userRepository');

test('gets user', async () => {
  userRepo.get.mockResolvedValue({ id: 1, name: 'Alice' });

  const user = await userService.getUser(1);

  expect(user.name).toBe('Alice');
  expect(userRepo.get).toHaveBeenCalledWith(1);
});
```

## 2. 使用 testify/mock

### 安裝

```bash
go get github.com/stretchr/testify/mock
go get github.com/stretchr/testify/assert
```

### 基礎用法

```go
// mock_repository.go
package user

import "github.com/stretchr/testify/mock"

type MockUserRepository struct {
    mock.Mock
}

func (m *MockUserRepository) Get(id int) (*User, error) {
    args := m.Called(id)
    if args.Get(0) == nil {
        return nil, args.Error(1)
    }
    return args.Get(0).(*User), args.Error(1)
}

func (m *MockUserRepository) Create(user *User) error {
    args := m.Called(user)
    return args.Error(0)
}

func (m *MockUserRepository) Update(user *User) error {
    args := m.Called(user)
    return args.Error(0)
}

func (m *MockUserRepository) Delete(id int) error {
    args := m.Called(id)
    return args.Error(0)
}
```

```go
// user_test.go
package user

import (
    "errors"
    "testing"

    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/mock"
)

func TestUserService_GetUser_Testify(t *testing.T) {
    // 創建 Mock
    mockRepo := new(MockUserRepository)

    // 設置期望
    expectedUser := &User{ID: 1, Name: "Alice"}
    mockRepo.On("Get", 1).Return(expectedUser, nil)

    // 創建 Service
    service := NewUserService(mockRepo)

    // 執行測試
    user, err := service.GetUser(1)

    // 斷言
    assert.NoError(t, err)
    assert.Equal(t, "Alice", user.Name)

    // 驗證所有期望都被調用
    mockRepo.AssertExpectations(t)
}

func TestUserService_GetUser_NotFound(t *testing.T) {
    mockRepo := new(MockUserRepository)
    mockRepo.On("Get", 999).Return(nil, errors.New("user not found"))

    service := NewUserService(mockRepo)
    user, err := service.GetUser(999)

    assert.Error(t, err)
    assert.Nil(t, user)
    mockRepo.AssertExpectations(t)
}

func TestUserService_CreateUser_Testify(t *testing.T) {
    mockRepo := new(MockUserRepository)

    // 使用 mock.Anything 匹配任意參數
    mockRepo.On("Create", mock.Anything).Return(nil)

    service := NewUserService(mockRepo)
    user, err := service.CreateUser("Bob")

    assert.NoError(t, err)
    assert.NotNil(t, user)
    mockRepo.AssertExpectations(t)

    // 驗證具體調用
    mockRepo.AssertCalled(t, "Create", mock.Anything)
}

func TestUserService_CreateUser_WithMatcher(t *testing.T) {
    mockRepo := new(MockUserRepository)

    // 使用自定義匹配器
    mockRepo.On("Create", mock.MatchedBy(func(u *User) bool {
        return u.Name == "Bob"
    })).Return(nil)

    service := NewUserService(mockRepo)
    user, err := service.CreateUser("Bob")

    assert.NoError(t, err)
    mockRepo.AssertExpectations(t)
}
```

### 高級用法

```go
package user

import (
    "testing"

    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/mock"
)

func TestAdvancedMocking(t *testing.T) {
    mockRepo := new(MockUserRepository)

    // 1. 多次調用返回不同結果
    mockRepo.On("Get", 1).Return(&User{ID: 1, Name: "Alice"}, nil).Once()
    mockRepo.On("Get", 1).Return(&User{ID: 1, Name: "Alice Updated"}, nil).Once()

    // 2. 返回函數結果
    mockRepo.On("Get", 2).Return(func(id int) (*User, error) {
        return &User{ID: id, Name: "Dynamic"}, nil
    })

    // 3. 延遲返回
    mockRepo.On("Get", 3).After(100 * time.Millisecond).Return(&User{ID: 3}, nil)

    // 4. 驗證調用次數
    mockRepo.On("Delete", 1).Return(nil).Times(3)

    // 5. 可能被調用（Maybe）
    mockRepo.On("Update", mock.Anything).Return(nil).Maybe()
}

func TestRunFunction(t *testing.T) {
    mockRepo := new(MockUserRepository)

    // 使用 Run 執行自定義邏輯
    mockRepo.On("Create", mock.Anything).Run(func(args mock.Arguments) {
        user := args.Get(0).(*User)
        user.ID = 100 // 修改傳入的參數
    }).Return(nil)

    service := NewUserService(mockRepo)
    user, _ := service.CreateUser("Test")

    assert.Equal(t, 100, user.ID)
}
```

**Node.js 對比：**
```javascript
// Jest
test('multiple calls', () => {
  userRepo.get
    .mockReturnValueOnce({ id: 1, name: 'Alice' })
    .mockReturnValueOnce({ id: 1, name: 'Alice Updated' });

  expect(userRepo.get()).toEqual({ id: 1, name: 'Alice' });
  expect(userRepo.get()).toEqual({ id: 1, name: 'Alice Updated' });
});

test('custom implementation', () => {
  userRepo.get.mockImplementation((id) => {
    return { id, name: 'Dynamic' };
  });

  expect(userRepo.get(2)).toEqual({ id: 2, name: 'Dynamic' });
});

test('call count', () => {
  userRepo.delete.mockReturnValue(null);

  userRepo.delete(1);
  userRepo.delete(1);
  userRepo.delete(1);

  expect(userRepo.delete).toHaveBeenCalledTimes(3);
});
```

## 3. 使用 gomock

### 安裝與生成

```bash
# 安裝 gomock
go install github.com/golang/mock/mockgen@latest

# 生成 Mock（從源文件）
mockgen -source=user.go -destination=mock_user.go -package=user

# 生成 Mock（從接口）
mockgen -destination=mock_user.go -package=user github.com/user/repo UserRepository
```

### 定義接口

```go
// user_repository.go
package user

//go:generate mockgen -source=user_repository.go -destination=mock_user_repository.go -package=user

type UserRepository interface {
    Get(id int) (*User, error)
    Create(user *User) error
    List() ([]*User, error)
}
```

```bash
# 生成 Mock
go generate ./...
```

### 使用 gomock

```go
// user_test.go
package user

import (
    "errors"
    "testing"

    "github.com/golang/mock/gomock"
)

func TestUserService_GetUser_Gomock(t *testing.T) {
    // 創建控制器
    ctrl := gomock.NewController(t)
    defer ctrl.Finish()

    // 創建 Mock
    mockRepo := NewMockUserRepository(ctrl)

    // 設置期望
    expectedUser := &User{ID: 1, Name: "Alice"}
    mockRepo.EXPECT().
        Get(1).
        Return(expectedUser, nil)

    // 使用 Mock
    service := NewUserService(mockRepo)
    user, err := service.GetUser(1)

    // 驗證
    if err != nil {
        t.Fatalf("unexpected error: %v", err)
    }

    if user.Name != "Alice" {
        t.Errorf("expected Alice, got %s", user.Name)
    }
}

func TestUserService_Advanced_Gomock(t *testing.T) {
    ctrl := gomock.NewController(t)
    defer ctrl.Finish()

    mockRepo := NewMockUserRepository(ctrl)

    // 1. 任意參數
    mockRepo.EXPECT().
        Get(gomock.Any()).
        Return(&User{ID: 1}, nil)

    // 2. 多次調用
    mockRepo.EXPECT().
        Get(1).
        Return(&User{ID: 1, Name: "Alice"}, nil).
        Times(2)

    // 3. 至少/最多調用
    mockRepo.EXPECT().
        Get(2).
        Return(&User{ID: 2}, nil).
        MinTimes(1).
        MaxTimes(3)

    // 4. 任意次數
    mockRepo.EXPECT().
        Get(3).
        Return(&User{ID: 3}, nil).
        AnyTimes()

    // 5. 自定義匹配器
    mockRepo.EXPECT().
        Create(gomock.AssignableToTypeOf(&User{})).
        Return(nil)

    // 6. 執行自定義函數
    mockRepo.EXPECT().
        Create(gomock.Any()).
        DoAndReturn(func(user *User) error {
            user.ID = 100
            return nil
        })

    // 7. 調用順序
    gomock.InOrder(
        mockRepo.EXPECT().Get(1).Return(&User{}, nil),
        mockRepo.EXPECT().Get(2).Return(&User{}, nil),
    )
}

func TestUserService_Error_Gomock(t *testing.T) {
    ctrl := gomock.NewController(t)
    defer ctrl.Finish()

    mockRepo := NewMockUserRepository(ctrl)

    // 返回錯誤
    mockRepo.EXPECT().
        Get(999).
        Return(nil, errors.New("user not found"))

    service := NewUserService(mockRepo)
    user, err := service.GetUser(999)

    if err == nil {
        t.Error("expected error, got nil")
    }

    if user != nil {
        t.Error("expected nil user")
    }
}
```

## 4. HTTP Mock

### 使用 httptest

```go
// client.go
package client

import (
    "encoding/json"
    "net/http"
)

type User struct {
    ID   int    `json:"id"`
    Name string `json:"name"`
}

type UserClient struct {
    baseURL string
    client  *http.Client
}

func NewUserClient(baseURL string) *UserClient {
    return &UserClient{
        baseURL: baseURL,
        client:  &http.Client{},
    }
}

func (c *UserClient) GetUser(id int) (*User, error) {
    resp, err := c.client.Get(fmt.Sprintf("%s/users/%d", c.baseURL, id))
    if err != nil {
        return nil, err
    }
    defer resp.Body.Close()

    if resp.StatusCode != http.StatusOK {
        return nil, fmt.Errorf("unexpected status: %d", resp.StatusCode)
    }

    var user User
    if err := json.NewDecoder(resp.Body).Decode(&user); err != nil {
        return nil, err
    }

    return &user, nil
}
```

```go
// client_test.go
package client

import (
    "encoding/json"
    "net/http"
    "net/http/httptest"
    "testing"
)

func TestUserClient_GetUser(t *testing.T) {
    // 創建測試服務器
    server := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // 驗證請求
        if r.Method != http.MethodGet {
            t.Errorf("expected GET, got %s", r.Method)
        }

        if r.URL.Path != "/users/1" {
            t.Errorf("expected /users/1, got %s", r.URL.Path)
        }

        // 返回模擬響應
        user := User{ID: 1, Name: "Alice"}
        w.Header().Set("Content-Type", "application/json")
        json.NewEncoder(w).Encode(user)
    }))
    defer server.Close()

    // 使用測試服務器 URL
    client := NewUserClient(server.URL)

    // 執行測試
    user, err := client.GetUser(1)

    if err != nil {
        t.Fatalf("unexpected error: %v", err)
    }

    if user.Name != "Alice" {
        t.Errorf("expected Alice, got %s", user.Name)
    }
}

func TestUserClient_GetUser_NotFound(t *testing.T) {
    server := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        w.WriteHeader(http.StatusNotFound)
    }))
    defer server.Close()

    client := NewUserClient(server.URL)
    user, err := client.GetUser(999)

    if err == nil {
        t.Error("expected error, got nil")
    }

    if user != nil {
        t.Error("expected nil user")
    }
}

func TestUserClient_TableDriven(t *testing.T) {
    tests := []struct {
        name           string
        userID         int
        handler        http.HandlerFunc
        expectedUser   *User
        expectedError  bool
    }{
        {
            name:   "success",
            userID: 1,
            handler: func(w http.ResponseWriter, r *http.Request) {
                json.NewEncoder(w).Encode(User{ID: 1, Name: "Alice"})
            },
            expectedUser: &User{ID: 1, Name: "Alice"},
            expectedError: false,
        },
        {
            name:   "not found",
            userID: 999,
            handler: func(w http.ResponseWriter, r *http.Request) {
                w.WriteHeader(http.StatusNotFound)
            },
            expectedUser: nil,
            expectedError: true,
        },
        {
            name:   "invalid json",
            userID: 2,
            handler: func(w http.ResponseWriter, r *http.Request) {
                w.Write([]byte("invalid json"))
            },
            expectedUser: nil,
            expectedError: true,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            server := httptest.NewServer(tt.handler)
            defer server.Close()

            client := NewUserClient(server.URL)
            user, err := client.GetUser(tt.userID)

            if tt.expectedError {
                if err == nil {
                    t.Error("expected error, got nil")
                }
            } else {
                if err != nil {
                    t.Fatalf("unexpected error: %v", err)
                }

                if user.ID != tt.expectedUser.ID || user.Name != tt.expectedUser.Name {
                    t.Errorf("got %+v, want %+v", user, tt.expectedUser)
                }
            }
        })
    }
}
```

**Node.js 對比：**
```javascript
// Jest + nock
const nock = require('nock');
const { getUserClient } = require('./client');

test('gets user', async () => {
  nock('http://api.example.com')
    .get('/users/1')
    .reply(200, { id: 1, name: 'Alice' });

  const client = getUserClient('http://api.example.com');
  const user = await client.getUser(1);

  expect(user.name).toBe('Alice');
});

test('handles 404', async () => {
  nock('http://api.example.com')
    .get('/users/999')
    .reply(404);

  const client = getUserClient('http://api.example.com');
  await expect(client.getUser(999)).rejects.toThrow();
});
```

## 5. 完整示例：服務層測試

```go
// service.go
package service

import (
    "errors"
    "fmt"
)

var (
    ErrUserNotFound = errors.New("user not found")
    ErrInvalidEmail = errors.New("invalid email")
)

type User struct {
    ID    int
    Name  string
    Email string
}

type UserRepository interface {
    Get(id int) (*User, error)
    GetByEmail(email string) (*User, error)
    Create(user *User) error
    Update(user *User) error
}

type EmailService interface {
    SendWelcome(email string) error
}

type UserService struct {
    repo  UserRepository
    email EmailService
}

func NewUserService(repo UserRepository, email EmailService) *UserService {
    return &UserService{
        repo:  repo,
        email: email,
    }
}

func (s *UserService) RegisterUser(name, email string) (*User, error) {
    // 驗證郵箱
    if !isValidEmail(email) {
        return nil, ErrInvalidEmail
    }

    // 檢查郵箱是否已存在
    existing, _ := s.repo.GetByEmail(email)
    if existing != nil {
        return nil, errors.New("email already exists")
    }

    // 創建用戶
    user := &User{Name: name, Email: email}
    if err := s.repo.Create(user); err != nil {
        return nil, err
    }

    // 發送歡迎郵件
    if err := s.email.SendWelcome(email); err != nil {
        // 記錄錯誤但不影響註冊
        fmt.Printf("failed to send welcome email: %v\n", err)
    }

    return user, nil
}

func (s *UserService) UpdateUser(id int, name, email string) error {
    user, err := s.repo.Get(id)
    if err != nil {
        return err
    }

    if !isValidEmail(email) {
        return ErrInvalidEmail
    }

    user.Name = name
    user.Email = email

    return s.repo.Update(user)
}

func isValidEmail(email string) bool {
    return len(email) > 0 && strings.Contains(email, "@")
}
```

```go
// service_test.go
package service

import (
    "errors"
    "testing"

    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/mock"
)

type MockUserRepository struct {
    mock.Mock
}

func (m *MockUserRepository) Get(id int) (*User, error) {
    args := m.Called(id)
    if args.Get(0) == nil {
        return nil, args.Error(1)
    }
    return args.Get(0).(*User), args.Error(1)
}

func (m *MockUserRepository) GetByEmail(email string) (*User, error) {
    args := m.Called(email)
    if args.Get(0) == nil {
        return nil, args.Error(1)
    }
    return args.Get(0).(*User), args.Error(1)
}

func (m *MockUserRepository) Create(user *User) error {
    args := m.Called(user)
    return args.Error(0)
}

func (m *MockUserRepository) Update(user *User) error {
    args := m.Called(user)
    return args.Error(0)
}

type MockEmailService struct {
    mock.Mock
}

func (m *MockEmailService) SendWelcome(email string) error {
    args := m.Called(email)
    return args.Error(0)
}

func TestUserService_RegisterUser(t *testing.T) {
    tests := []struct {
        name          string
        userName      string
        userEmail     string
        setupMocks    func(*MockUserRepository, *MockEmailService)
        expectedError bool
    }{
        {
            name:      "successful registration",
            userName:  "Alice",
            userEmail: "alice@example.com",
            setupMocks: func(repo *MockUserRepository, email *MockEmailService) {
                repo.On("GetByEmail", "alice@example.com").Return(nil, ErrUserNotFound)
                repo.On("Create", mock.AnythingOfType("*service.User")).Return(nil)
                email.On("SendWelcome", "alice@example.com").Return(nil)
            },
            expectedError: false,
        },
        {
            name:      "invalid email",
            userName:  "Bob",
            userEmail: "invalid",
            setupMocks: func(repo *MockUserRepository, email *MockEmailService) {
                // 不會調用任何方法
            },
            expectedError: true,
        },
        {
            name:      "email already exists",
            userName:  "Charlie",
            userEmail: "existing@example.com",
            setupMocks: func(repo *MockUserRepository, email *MockEmailService) {
                existing := &User{ID: 1, Email: "existing@example.com"}
                repo.On("GetByEmail", "existing@example.com").Return(existing, nil)
            },
            expectedError: true,
        },
        {
            name:      "email service failure",
            userName:  "David",
            userEmail: "david@example.com",
            setupMocks: func(repo *MockUserRepository, email *MockEmailService) {
                repo.On("GetByEmail", "david@example.com").Return(nil, ErrUserNotFound)
                repo.On("Create", mock.AnythingOfType("*service.User")).Return(nil)
                email.On("SendWelcome", "david@example.com").Return(errors.New("smtp error"))
            },
            expectedError: false, // 郵件失敗不影響註冊
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            // 創建 Mocks
            mockRepo := new(MockUserRepository)
            mockEmail := new(MockEmailService)

            // 設置 Mocks
            tt.setupMocks(mockRepo, mockEmail)

            // 創建 Service
            service := NewUserService(mockRepo, mockEmail)

            // 執行測試
            user, err := service.RegisterUser(tt.userName, tt.userEmail)

            // 驗證結果
            if tt.expectedError {
                assert.Error(t, err)
                assert.Nil(t, user)
            } else {
                assert.NoError(t, err)
                assert.NotNil(t, user)
                assert.Equal(t, tt.userName, user.Name)
                assert.Equal(t, tt.userEmail, user.Email)
            }

            // 驗證 Mock 調用
            mockRepo.AssertExpectations(t)
            mockEmail.AssertExpectations(t)
        })
    }
}

func TestUserService_UpdateUser(t *testing.T) {
    mockRepo := new(MockUserRepository)
    mockEmail := new(MockEmailService)

    existingUser := &User{ID: 1, Name: "Alice", Email: "alice@example.com"}
    mockRepo.On("Get", 1).Return(existingUser, nil)
    mockRepo.On("Update", mock.MatchedBy(func(u *User) bool {
        return u.ID == 1 && u.Name == "Alice Updated"
    })).Return(nil)

    service := NewUserService(mockRepo, mockEmail)
    err := service.UpdateUser(1, "Alice Updated", "alice.new@example.com")

    assert.NoError(t, err)
    mockRepo.AssertExpectations(t)
}
```

## 重點總結

### Mock 策略選擇

| 場景 | 推薦方案 | 原因 |
|------|---------|------|
| **簡單接口** | 手動 Mock | 靈活、易理解 |
| **複雜接口** | gomock | 類型安全、自動生成 |
| **快速開發** | testify/mock | 語法簡潔 |
| **HTTP 測試** | httptest | 標準庫支持 |

### 最佳實踐

1. **優先使用接口**設計依賴
2. **依賴注入**而不是全局變量
3. **Mock 外部依賴**（數據庫、API、文件系統）
4. **不要 Mock 被測試的代碼**
5. **驗證 Mock 調用**（AssertExpectations）
6. **使用表格驅動測試**組織多場景

### Go vs Node.js Mock

| 特性 | Go | Node.js |
|------|------|---------|
| **Mock 方式** | 接口 + Mock 實現 | 函數替換 |
| **類型安全** | 編譯時檢查 | 運行時檢查 |
| **生成工具** | mockgen | N/A |
| **驗證** | AssertExpectations | expect().toHaveBeenCalled() |

## 練習題

1. **基礎**：為一個簡單的服務編寫手動 Mock 測試
2. **中級**：使用 testify/mock 測試一個有多個依賴的服務
3. **高級**：使用 gomock 測試複雜業務邏輯
4. **實戰**：為一個完整的 Web 服務編寫 Mock 測試，包括數據庫、緩存、外部 API

## 總結

恭喜完成 Go 學習指南的 Part 7-8！你現在已經掌握：
- 數據庫操作（SQL、GORM、PostgreSQL、MongoDB、Redis）
- 測試技術（單元測試、基準測試、表格驅動測試、Mock）

這些知識將幫助你構建生產級的 Go 應用程序。繼續練習，祝學習順利！
