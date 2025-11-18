# 第 62 章: 2025 年實際 Web 專案範例

## 概述

本章將展示一個完整的 2025 年 Go Web 應用程序，使用所有現代最佳實踐和特性。我們將構建一個**任務管理 API**（Task Management API），並與等效的 Node.js/Express 項目進行對比。

## 項目功能

- ✅ 用戶註冊和登入（JWT 認證）
- ✅ CRUD 操作（任務管理）
- ✅ 用戶權限控制
- ✅ PostgreSQL 數據庫
- ✅ Redis 緩存
- ✅ 結構化日誌（slog）
- ✅ 完整測試
- ✅ Docker 容器化
- ✅ 優雅關閉
- ✅ API 文檔

## 技術棧對比

### Go Stack (2025)

```
- Go 1.23+
- 標準庫 net/http (Go 1.22+ 路由)
- GORM (ORM)
- PostgreSQL
- Redis (go-redis)
- JWT (golang-jwt/jwt)
- log/slog (結構化日誌)
- testify (測試)
- Docker
```

### Node.js Stack (2025)

```
- Node.js 20+
- Express 4+
- TypeORM/Prisma
- PostgreSQL
- Redis (ioredis)
- JWT (jsonwebtoken)
- Winston/Pino (日誌)
- Jest (測試)
- Docker
```

## 項目結構

```
taskmaster/
├── cmd/
│   └── api/
│       └── main.go                 # 應用入口
├── internal/
│   ├── config/
│   │   └── config.go               # 配置管理
│   ├── domain/
│   │   ├── user.go                 # 用戶模型
│   │   └── task.go                 # 任務模型
│   ├── repository/
│   │   ├── user_repository.go      # 用戶數據訪問
│   │   └── task_repository.go      # 任務數據訪問
│   ├── service/
│   │   ├── auth_service.go         # 認證服務
│   │   ├── user_service.go         # 用戶服務
│   │   └── task_service.go         # 任務服務
│   ├── handler/
│   │   ├── auth_handler.go         # 認證處理器
│   │   ├── user_handler.go         # 用戶處理器
│   │   └── task_handler.go         # 任務處理器
│   ├── middleware/
│   │   ├── auth.go                 # JWT 中間件
│   │   ├── logging.go              # 日誌中間件
│   │   └── recovery.go             # 恢復中間件
│   └── database/
│       ├── postgres.go             # PostgreSQL 連接
│       └── redis.go                # Redis 連接
├── pkg/
│   ├── apperror/
│   │   └── error.go                # 自定義錯誤
│   └── jwt/
│       └── jwt.go                  # JWT 工具
├── migrations/
│   └── 001_initial.sql             # 數據庫遷移
├── docker/
│   ├── Dockerfile                  # Docker 配置
│   └── docker-compose.yml          # Docker Compose
├── configs/
│   └── config.yaml                 # 配置文件
├── tests/
│   ├── integration/
│   └── unit/
├── go.mod
├── go.sum
├── Makefile
└── README.md
```

### Node.js 對比結構

```
taskmaster-node/
├── src/
│   ├── config/
│   ├── models/                     # 對應 domain
│   ├── repositories/               # 對應 repository
│   ├── services/                   # 對應 service
│   ├── controllers/                # 對應 handler
│   ├── middlewares/                # 對應 middleware
│   └── database/
├── migrations/
├── docker/
├── tests/
├── package.json
└── tsconfig.json
```

## 完整代碼實現

### 1. 配置管理 (internal/config/config.go)

```go
package config

import (
    "fmt"
    "os"
    "strconv"
    "time"
)

type Config struct {
    Server   ServerConfig
    Database DatabaseConfig
    Redis    RedisConfig
    JWT      JWTConfig
}

type ServerConfig struct {
    Port            string
    ReadTimeout     time.Duration
    WriteTimeout    time.Duration
    ShutdownTimeout time.Duration
}

type DatabaseConfig struct {
    Host     string
    Port     string
    User     string
    Password string
    DBName   string
    SSLMode  string
}

type RedisConfig struct {
    Addr     string
    Password string
    DB       int
}

type JWTConfig struct {
    Secret          string
    AccessTokenTTL  time.Duration
    RefreshTokenTTL time.Duration
}

// Load 從環境變量加載配置
func Load() (*Config, error) {
    cfg := &Config{
        Server: ServerConfig{
            Port:            getEnv("SERVER_PORT", "8080"),
            ReadTimeout:     getDurationEnv("SERVER_READ_TIMEOUT", 10*time.Second),
            WriteTimeout:    getDurationEnv("SERVER_WRITE_TIMEOUT", 10*time.Second),
            ShutdownTimeout: getDurationEnv("SERVER_SHUTDOWN_TIMEOUT", 10*time.Second),
        },
        Database: DatabaseConfig{
            Host:     getEnv("DB_HOST", "localhost"),
            Port:     getEnv("DB_PORT", "5432"),
            User:     getEnv("DB_USER", "postgres"),
            Password: getEnv("DB_PASSWORD", ""),
            DBName:   getEnv("DB_NAME", "taskmaster"),
            SSLMode:  getEnv("DB_SSLMODE", "disable"),
        },
        Redis: RedisConfig{
            Addr:     getEnv("REDIS_ADDR", "localhost:6379"),
            Password: getEnv("REDIS_PASSWORD", ""),
            DB:       getIntEnv("REDIS_DB", 0),
        },
        JWT: JWTConfig{
            Secret:          getEnv("JWT_SECRET", "your-secret-key"),
            AccessTokenTTL:  getDurationEnv("JWT_ACCESS_TTL", 15*time.Minute),
            RefreshTokenTTL: getDurationEnv("JWT_REFRESH_TTL", 7*24*time.Hour),
        },
    }

    return cfg, nil
}

func (c *DatabaseConfig) DSN() string {
    return fmt.Sprintf(
        "host=%s port=%s user=%s password=%s dbname=%s sslmode=%s",
        c.Host, c.Port, c.User, c.Password, c.DBName, c.SSLMode,
    )
}

func getEnv(key, defaultValue string) string {
    if value := os.Getenv(key); value != "" {
        return value
    }
    return defaultValue
}

func getIntEnv(key string, defaultValue int) int {
    if value := os.Getenv(key); value != "" {
        if i, err := strconv.Atoi(value); err == nil {
            return i
        }
    }
    return defaultValue
}

func getDurationEnv(key string, defaultValue time.Duration) time.Duration {
    if value := os.Getenv(key); value != "" {
        if d, err := time.ParseDuration(value); err == nil {
            return d
        }
    }
    return defaultValue
}
```

**Node.js 對比**：

```typescript
// config/index.ts
import dotenv from 'dotenv';

dotenv.config();

export const config = {
    server: {
        port: process.env.SERVER_PORT || '8080',
        readTimeout: parseInt(process.env.SERVER_READ_TIMEOUT || '10000'),
        writeTimeout: parseInt(process.env.SERVER_WRITE_TIMEOUT || '10000'),
    },
    database: {
        host: process.env.DB_HOST || 'localhost',
        port: parseInt(process.env.DB_PORT || '5432'),
        user: process.env.DB_USER || 'postgres',
        password: process.env.DB_PASSWORD || '',
        database: process.env.DB_NAME || 'taskmaster',
    },
    redis: {
        host: process.env.REDIS_HOST || 'localhost',
        port: parseInt(process.env.REDIS_PORT || '6379'),
    },
    jwt: {
        secret: process.env.JWT_SECRET || 'your-secret-key',
        accessTTL: '15m',
        refreshTTL: '7d',
    },
};
```

### 2. 領域模型 (internal/domain/)

```go
// internal/domain/user.go
package domain

import (
    "time"
    "golang.org/x/crypto/bcrypt"
)

type User struct {
    ID        uint      `json:"id" gorm:"primaryKey"`
    Email     string    `json:"email" gorm:"uniqueIndex;not null"`
    Name      string    `json:"name" gorm:"not null"`
    Password  string    `json:"-" gorm:"not null"`  // 不返回到 JSON
    CreatedAt time.Time `json:"created_at"`
    UpdatedAt time.Time `json:"updated_at"`
    Tasks     []Task    `json:"tasks,omitempty" gorm:"foreignKey:UserID"`
}

// HashPassword 加密密碼
func (u *User) HashPassword() error {
    hashed, err := bcrypt.GenerateFromPassword([]byte(u.Password), bcrypt.DefaultCost)
    if err != nil {
        return err
    }
    u.Password = string(hashed)
    return nil
}

// CheckPassword 驗證密碼
func (u *User) CheckPassword(password string) bool {
    err := bcrypt.CompareHashAndPassword([]byte(u.Password), []byte(password))
    return err == nil
}

// internal/domain/task.go
package domain

import "time"

type TaskStatus string

const (
    TaskStatusTodo       TaskStatus = "todo"
    TaskStatusInProgress TaskStatus = "in_progress"
    TaskStatusDone       TaskStatus = "done"
)

type Task struct {
    ID          uint       `json:"id" gorm:"primaryKey"`
    Title       string     `json:"title" gorm:"not null"`
    Description string     `json:"description"`
    Status      TaskStatus `json:"status" gorm:"default:'todo'"`
    Priority    int        `json:"priority" gorm:"default:0"`
    DueDate     *time.Time `json:"due_date,omitempty"`
    UserID      uint       `json:"user_id" gorm:"not null;index"`
    User        *User      `json:"user,omitempty" gorm:"foreignKey:UserID"`
    CreatedAt   time.Time  `json:"created_at"`
    UpdatedAt   time.Time  `json:"updated_at"`
}

// 使用泛型的分頁響應
type PaginatedResponse[T any] struct {
    Data       []T   `json:"data"`
    Total      int64 `json:"total"`
    Page       int   `json:"page"`
    PageSize   int   `json:"page_size"`
    TotalPages int   `json:"total_pages"`
}
```

**Node.js/TypeScript 對比**：

```typescript
// models/User.ts
import { Entity, Column, PrimaryGeneratedColumn, CreateDateColumn, UpdateDateColumn, OneToMany } from 'typeorm';
import bcrypt from 'bcrypt';
import { Task } from './Task';

@Entity('users')
export class User {
    @PrimaryGeneratedColumn()
    id: number;

    @Column({ unique: true })
    email: string;

    @Column()
    name: string;

    @Column({ select: false })
    password: string;

    @CreateDateColumn()
    createdAt: Date;

    @UpdateDateColumn()
    updatedAt: Date;

    @OneToMany(() => Task, task => task.user)
    tasks: Task[];

    async hashPassword(): Promise<void> {
        this.password = await bcrypt.hash(this.password, 10);
    }

    async checkPassword(password: string): Promise<boolean> {
        return bcrypt.compare(password, this.password);
    }
}
```

### 3. Repository 層 (internal/repository/)

```go
// internal/repository/user_repository.go
package repository

import (
    "context"
    "taskmaster/internal/domain"
    "gorm.io/gorm"
)

type UserRepository interface {
    Create(ctx context.Context, user *domain.User) error
    FindByID(ctx context.Context, id uint) (*domain.User, error)
    FindByEmail(ctx context.Context, email string) (*domain.User, error)
    Update(ctx context.Context, user *domain.User) error
    Delete(ctx context.Context, id uint) error
}

type userRepository struct {
    db *gorm.DB
}

func NewUserRepository(db *gorm.DB) UserRepository {
    return &userRepository{db: db}
}

func (r *userRepository) Create(ctx context.Context, user *domain.User) error {
    return r.db.WithContext(ctx).Create(user).Error
}

func (r *userRepository) FindByID(ctx context.Context, id uint) (*domain.User, error) {
    var user domain.User
    err := r.db.WithContext(ctx).First(&user, id).Error
    if err != nil {
        return nil, err
    }
    return &user, nil
}

func (r *userRepository) FindByEmail(ctx context.Context, email string) (*domain.User, error) {
    var user domain.User
    // 需要選擇 password 欄位用於認證
    err := r.db.WithContext(ctx).
        Select("*").
        Where("email = ?", email).
        First(&user).Error
    if err != nil {
        return nil, err
    }
    return &user, nil
}

func (r *userRepository) Update(ctx context.Context, user *domain.User) error {
    return r.db.WithContext(ctx).Save(user).Error
}

func (r *userRepository) Delete(ctx context.Context, id uint) error {
    return r.db.WithContext(ctx).Delete(&domain.User{}, id).Error
}

// internal/repository/task_repository.go
package repository

import (
    "context"
    "taskmaster/internal/domain"
    "gorm.io/gorm"
)

type TaskFilter struct {
    UserID *uint
    Status *domain.TaskStatus
    Page   int
    Limit  int
}

type TaskRepository interface {
    Create(ctx context.Context, task *domain.Task) error
    FindByID(ctx context.Context, id uint) (*domain.Task, error)
    FindAll(ctx context.Context, filter TaskFilter) ([]domain.Task, int64, error)
    Update(ctx context.Context, task *domain.Task) error
    Delete(ctx context.Context, id uint) error
}

type taskRepository struct {
    db *gorm.DB
}

func NewTaskRepository(db *gorm.DB) TaskRepository {
    return &taskRepository{db: db}
}

func (r *taskRepository) Create(ctx context.Context, task *domain.Task) error {
    return r.db.WithContext(ctx).Create(task).Error
}

func (r *taskRepository) FindByID(ctx context.Context, id uint) (*domain.Task, error) {
    var task domain.Task
    err := r.db.WithContext(ctx).Preload("User").First(&task, id).Error
    if err != nil {
        return nil, err
    }
    return &task, nil
}

func (r *taskRepository) FindAll(ctx context.Context, filter TaskFilter) ([]domain.Task, int64, error) {
    var tasks []domain.Task
    var total int64

    query := r.db.WithContext(ctx).Model(&domain.Task{})

    // 應用過濾器
    if filter.UserID != nil {
        query = query.Where("user_id = ?", *filter.UserID)
    }
    if filter.Status != nil {
        query = query.Where("status = ?", *filter.Status)
    }

    // 計算總數
    if err := query.Count(&total).Error; err != nil {
        return nil, 0, err
    }

    // 分頁
    offset := (filter.Page - 1) * filter.Limit
    err := query.
        Preload("User").
        Offset(offset).
        Limit(filter.Limit).
        Order("created_at DESC").
        Find(&tasks).Error

    return tasks, total, err
}

func (r *taskRepository) Update(ctx context.Context, task *domain.Task) error {
    return r.db.WithContext(ctx).Save(task).Error
}

func (r *taskRepository) Delete(ctx context.Context, id uint) error {
    return r.db.WithContext(ctx).Delete(&domain.Task{}, id).Error
}
```

### 4. Service 層 (internal/service/)

```go
// internal/service/auth_service.go
package service

import (
    "context"
    "errors"
    "taskmaster/internal/domain"
    "taskmaster/internal/repository"
    "taskmaster/pkg/jwt"
)

var (
    ErrInvalidCredentials = errors.New("無效的憑證")
    ErrEmailAlreadyExists = errors.New("郵箱已存在")
)

type AuthService interface {
    Register(ctx context.Context, email, name, password string) (*domain.User, error)
    Login(ctx context.Context, email, password string) (string, string, error)
}

type authService struct {
    userRepo repository.UserRepository
    jwtMgr   *jwt.Manager
}

func NewAuthService(userRepo repository.UserRepository, jwtMgr *jwt.Manager) AuthService {
    return &authService{
        userRepo: userRepo,
        jwtMgr:   jwtMgr,
    }
}

func (s *authService) Register(ctx context.Context, email, name, password string) (*domain.User, error) {
    // 檢查郵箱是否已存在
    existing, err := s.userRepo.FindByEmail(ctx, email)
    if err == nil && existing != nil {
        return nil, ErrEmailAlreadyExists
    }

    // 創建用戶
    user := &domain.User{
        Email:    email,
        Name:     name,
        Password: password,
    }

    if err := user.HashPassword(); err != nil {
        return nil, err
    }

    if err := s.userRepo.Create(ctx, user); err != nil {
        return nil, err
    }

    // 清除密碼
    user.Password = ""
    return user, nil
}

func (s *authService) Login(ctx context.Context, email, password string) (string, string, error) {
    // 查找用戶
    user, err := s.userRepo.FindByEmail(ctx, email)
    if err != nil {
        return "", "", ErrInvalidCredentials
    }

    // 驗證密碼
    if !user.CheckPassword(password) {
        return "", "", ErrInvalidCredentials
    }

    // 生成 token
    accessToken, err := s.jwtMgr.GenerateAccessToken(user.ID, user.Email)
    if err != nil {
        return "", "", err
    }

    refreshToken, err := s.jwtMgr.GenerateRefreshToken(user.ID)
    if err != nil {
        return "", "", err
    }

    return accessToken, refreshToken, nil
}

// internal/service/task_service.go
package service

import (
    "context"
    "errors"
    "math"
    "taskmaster/internal/domain"
    "taskmaster/internal/repository"
)

var (
    ErrTaskNotFound   = errors.New("任務不存在")
    ErrUnauthorized   = errors.New("無權限")
)

type TaskService interface {
    Create(ctx context.Context, userID uint, title, description string, priority int) (*domain.Task, error)
    GetByID(ctx context.Context, userID, taskID uint) (*domain.Task, error)
    List(ctx context.Context, userID uint, status *domain.TaskStatus, page, limit int) (*domain.PaginatedResponse[domain.Task], error)
    Update(ctx context.Context, userID, taskID uint, updates map[string]interface{}) (*domain.Task, error)
    Delete(ctx context.Context, userID, taskID uint) error
}

type taskService struct {
    taskRepo repository.TaskRepository
}

func NewTaskService(taskRepo repository.TaskRepository) TaskService {
    return &taskService{taskRepo: taskRepo}
}

func (s *taskService) Create(ctx context.Context, userID uint, title, description string, priority int) (*domain.Task, error) {
    task := &domain.Task{
        Title:       title,
        Description: description,
        Priority:    priority,
        UserID:      userID,
        Status:      domain.TaskStatusTodo,
    }

    if err := s.taskRepo.Create(ctx, task); err != nil {
        return nil, err
    }

    return task, nil
}

func (s *taskService) GetByID(ctx context.Context, userID, taskID uint) (*domain.Task, error) {
    task, err := s.taskRepo.FindByID(ctx, taskID)
    if err != nil {
        return nil, ErrTaskNotFound
    }

    if task.UserID != userID {
        return nil, ErrUnauthorized
    }

    return task, nil
}

func (s *taskService) List(ctx context.Context, userID uint, status *domain.TaskStatus, page, limit int) (*domain.PaginatedResponse[domain.Task], error) {
    if page < 1 {
        page = 1
    }
    if limit < 1 || limit > 100 {
        limit = 10
    }

    filter := repository.TaskFilter{
        UserID: &userID,
        Status: status,
        Page:   page,
        Limit:  limit,
    }

    tasks, total, err := s.taskRepo.FindAll(ctx, filter)
    if err != nil {
        return nil, err
    }

    totalPages := int(math.Ceil(float64(total) / float64(limit)))

    return &domain.PaginatedResponse[domain.Task]{
        Data:       tasks,
        Total:      total,
        Page:       page,
        PageSize:   limit,
        TotalPages: totalPages,
    }, nil
}

func (s *taskService) Update(ctx context.Context, userID, taskID uint, updates map[string]interface{}) (*domain.Task, error) {
    task, err := s.GetByID(ctx, userID, taskID)
    if err != nil {
        return nil, err
    }

    // 應用更新
    if title, ok := updates["title"].(string); ok {
        task.Title = title
    }
    if description, ok := updates["description"].(string); ok {
        task.Description = description
    }
    if status, ok := updates["status"].(string); ok {
        task.Status = domain.TaskStatus(status)
    }
    if priority, ok := updates["priority"].(float64); ok {
        task.Priority = int(priority)
    }

    if err := s.taskRepo.Update(ctx, task); err != nil {
        return nil, err
    }

    return task, nil
}

func (s *taskService) Delete(ctx context.Context, userID, taskID uint) error {
    task, err := s.GetByID(ctx, userID, taskID)
    if err != nil {
        return err
    }

    if task.UserID != userID {
        return ErrUnauthorized
    }

    return s.taskRepo.Delete(ctx, taskID)
}
```

### 5. JWT 工具 (pkg/jwt/jwt.go)

```go
package jwt

import (
    "errors"
    "time"
    "github.com/golang-jwt/jwt/v5"
)

var (
    ErrInvalidToken = errors.New("無效的 token")
    ErrExpiredToken = errors.New("token 已過期")
)

type Manager struct {
    secret          []byte
    accessTokenTTL  time.Duration
    refreshTokenTTL time.Duration
}

type Claims struct {
    UserID uint   `json:"user_id"`
    Email  string `json:"email,omitempty"`
    jwt.RegisteredClaims
}

func NewManager(secret string, accessTTL, refreshTTL time.Duration) *Manager {
    return &Manager{
        secret:          []byte(secret),
        accessTokenTTL:  accessTTL,
        refreshTokenTTL: refreshTTL,
    }
}

func (m *Manager) GenerateAccessToken(userID uint, email string) (string, error) {
    claims := &Claims{
        UserID: userID,
        Email:  email,
        RegisteredClaims: jwt.RegisteredClaims{
            ExpiresAt: jwt.NewNumericDate(time.Now().Add(m.accessTokenTTL)),
            IssuedAt:  jwt.NewNumericDate(time.Now()),
        },
    }

    token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
    return token.SignedString(m.secret)
}

func (m *Manager) GenerateRefreshToken(userID uint) (string, error) {
    claims := &Claims{
        UserID: userID,
        RegisteredClaims: jwt.RegisteredClaims{
            ExpiresAt: jwt.NewNumericDate(time.Now().Add(m.refreshTokenTTL)),
            IssuedAt:  jwt.NewNumericDate(time.Now()),
        },
    }

    token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
    return token.SignedString(m.secret)
}

func (m *Manager) ValidateToken(tokenString string) (*Claims, error) {
    token, err := jwt.ParseWithClaims(tokenString, &Claims{}, func(token *jwt.Token) (interface{}, error) {
        if _, ok := token.Method.(*jwt.SigningMethodHMAC); !ok {
            return nil, ErrInvalidToken
        }
        return m.secret, nil
    })

    if err != nil {
        if errors.Is(err, jwt.ErrTokenExpired) {
            return nil, ErrExpiredToken
        }
        return nil, ErrInvalidToken
    }

    claims, ok := token.Claims.(*Claims)
    if !ok || !token.Valid {
        return nil, ErrInvalidToken
    }

    return claims, nil
}
```

### 6. 中間件 (internal/middleware/)

```go
// internal/middleware/auth.go
package middleware

import (
    "context"
    "log/slog"
    "net/http"
    "strings"
    "taskmaster/pkg/jwt"
)

type contextKey string

const UserIDKey contextKey = "user_id"

func Auth(jwtMgr *jwt.Manager) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            authHeader := r.Header.Get("Authorization")
            if authHeader == "" {
                http.Error(w, "未提供認證 token", http.StatusUnauthorized)
                return
            }

            parts := strings.Split(authHeader, " ")
            if len(parts) != 2 || parts[0] != "Bearer" {
                http.Error(w, "無效的認證格式", http.StatusUnauthorized)
                return
            }

            claims, err := jwtMgr.ValidateToken(parts[1])
            if err != nil {
                slog.Error("token 驗證失敗", slog.Any("error", err))
                http.Error(w, "無效的 token", http.StatusUnauthorized)
                return
            }

            // 將用戶 ID 存入 context
            ctx := context.WithValue(r.Context(), UserIDKey, claims.UserID)
            next.ServeHTTP(w, r.WithContext(ctx))
        })
    }
}

// GetUserID 從 context 獲取用戶 ID
func GetUserID(ctx context.Context) (uint, bool) {
    userID, ok := ctx.Value(UserIDKey).(uint)
    return userID, ok
}

// internal/middleware/logging.go
package middleware

import (
    "log/slog"
    "net/http"
    "time"
)

type responseWriter struct {
    http.ResponseWriter
    status int
    size   int
}

func (rw *responseWriter) WriteHeader(status int) {
    rw.status = status
    rw.ResponseWriter.WriteHeader(status)
}

func (rw *responseWriter) Write(b []byte) (int, error) {
    size, err := rw.ResponseWriter.Write(b)
    rw.size += size
    return size, err
}

func Logging(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()

        rw := &responseWriter{
            ResponseWriter: w,
            status:         http.StatusOK,
        }

        next.ServeHTTP(rw, r)

        duration := time.Since(start)

        slog.Info("HTTP 請求",
            slog.String("method", r.Method),
            slog.String("path", r.URL.Path),
            slog.Int("status", rw.status),
            slog.Int("size", rw.size),
            slog.Duration("duration", duration),
            slog.String("ip", r.RemoteAddr),
        )
    })
}

// internal/middleware/recovery.go
package middleware

import (
    "log/slog"
    "net/http"
    "runtime/debug"
)

func Recovery(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        defer func() {
            if err := recover(); err != nil {
                slog.Error("panic 恢復",
                    slog.Any("error", err),
                    slog.String("stack", string(debug.Stack())),
                )

                http.Error(w, "Internal Server Error", http.StatusInternalServerError)
            }
        }()

        next.ServeHTTP(w, r)
    })
}
```

### 7. Handler 層 (internal/handler/)

```go
// internal/handler/auth_handler.go
package handler

import (
    "encoding/json"
    "log/slog"
    "net/http"
    "taskmaster/internal/service"
)

type AuthHandler struct {
    authService service.AuthService
}

func NewAuthHandler(authService service.AuthService) *AuthHandler {
    return &AuthHandler{authService: authService}
}

type RegisterRequest struct {
    Email    string `json:"email"`
    Name     string `json:"name"`
    Password string `json:"password"`
}

type LoginRequest struct {
    Email    string `json:"email"`
    Password string `json:"password"`
}

type AuthResponse struct {
    AccessToken  string `json:"access_token"`
    RefreshToken string `json:"refresh_token"`
    User         interface{} `json:"user,omitempty"`
}

func (h *AuthHandler) Register(w http.ResponseWriter, r *http.Request) {
    var req RegisterRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        respondError(w, http.StatusBadRequest, "無效的請求數據")
        return
    }

    user, err := h.authService.Register(r.Context(), req.Email, req.Name, req.Password)
    if err != nil {
        if err == service.ErrEmailAlreadyExists {
            respondError(w, http.StatusConflict, err.Error())
            return
        }
        slog.Error("註冊失敗", slog.Any("error", err))
        respondError(w, http.StatusInternalServerError, "註冊失敗")
        return
    }

    respondJSON(w, http.StatusCreated, user)
}

func (h *AuthHandler) Login(w http.ResponseWriter, r *http.Request) {
    var req LoginRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        respondError(w, http.StatusBadRequest, "無效的請求數據")
        return
    }

    accessToken, refreshToken, err := h.authService.Login(r.Context(), req.Email, req.Password)
    if err != nil {
        if err == service.ErrInvalidCredentials {
            respondError(w, http.StatusUnauthorized, err.Error())
            return
        }
        slog.Error("登入失敗", slog.Any("error", err))
        respondError(w, http.StatusInternalServerError, "登入失敗")
        return
    }

    respondJSON(w, http.StatusOK, AuthResponse{
        AccessToken:  accessToken,
        RefreshToken: refreshToken,
    })
}

// internal/handler/task_handler.go (部分代碼)
package handler

import (
    "encoding/json"
    "net/http"
    "strconv"
    "taskmaster/internal/middleware"
    "taskmaster/internal/service"
    "taskmaster/internal/domain"
)

type TaskHandler struct {
    taskService service.TaskService
}

func NewTaskHandler(taskService service.TaskService) *TaskHandler {
    return &TaskHandler{taskService: taskService}
}

type CreateTaskRequest struct {
    Title       string `json:"title"`
    Description string `json:"description"`
    Priority    int    `json:"priority"`
}

func (h *TaskHandler) Create(w http.ResponseWriter, r *http.Request) {
    userID, ok := middleware.GetUserID(r.Context())
    if !ok {
        respondError(w, http.StatusUnauthorized, "未認證")
        return
    }

    var req CreateTaskRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        respondError(w, http.StatusBadRequest, "無效的請求數據")
        return
    }

    task, err := h.taskService.Create(r.Context(), userID, req.Title, req.Description, req.Priority)
    if err != nil {
        respondError(w, http.StatusInternalServerError, "創建任務失敗")
        return
    }

    respondJSON(w, http.StatusCreated, task)
}

func (h *TaskHandler) List(w http.ResponseWriter, r *http.Request) {
    userID, ok := middleware.GetUserID(r.Context())
    if !ok {
        respondError(w, http.StatusUnauthorized, "未認證")
        return
    }

    // 獲取查詢參數
    page, _ := strconv.Atoi(r.URL.Query().Get("page"))
    limit, _ := strconv.Atoi(r.URL.Query().Get("limit"))
    statusStr := r.URL.Query().Get("status")

    var status *domain.TaskStatus
    if statusStr != "" {
        s := domain.TaskStatus(statusStr)
        status = &s
    }

    result, err := h.taskService.List(r.Context(), userID, status, page, limit)
    if err != nil {
        respondError(w, http.StatusInternalServerError, "獲取任務列表失敗")
        return
    }

    respondJSON(w, http.StatusOK, result)
}

// 通用響應函數
func respondJSON(w http.ResponseWriter, status int, data interface{}) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(status)
    json.NewEncoder(w).Encode(data)
}

func respondError(w http.ResponseWriter, status int, message string) {
    respondJSON(w, status, map[string]string{"error": message})
}
```

### 8. 主程序 (cmd/api/main.go)

```go
package main

import (
    "context"
    "log/slog"
    "net/http"
    "os"
    "os/signal"
    "syscall"
    "time"

    "taskmaster/internal/config"
    "taskmaster/internal/handler"
    "taskmaster/internal/middleware"
    "taskmaster/internal/repository"
    "taskmaster/internal/service"
    "taskmaster/pkg/jwt"

    "gorm.io/driver/postgres"
    "gorm.io/gorm"
    "taskmaster/internal/domain"
)

func main() {
    // 設置結構化日誌
    logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
        Level:     slog.LevelInfo,
        AddSource: true,
    }))
    slog.SetDefault(logger)

    // 加載配置
    cfg, err := config.Load()
    if err != nil {
        slog.Error("加載配置失敗", slog.Any("error", err))
        os.Exit(1)
    }

    // 連接數據庫
    db, err := gorm.Open(postgres.Open(cfg.Database.DSN()), &gorm.Config{})
    if err != nil {
        slog.Error("連接數據庫失敗", slog.Any("error", err))
        os.Exit(1)
    }

    // 自動遷移
    if err := db.AutoMigrate(&domain.User{}, &domain.Task{}); err != nil {
        slog.Error("數據庫遷移失敗", slog.Any("error", err))
        os.Exit(1)
    }

    // 初始化依賴
    jwtManager := jwt.NewManager(
        cfg.JWT.Secret,
        cfg.JWT.AccessTokenTTL,
        cfg.JWT.RefreshTokenTTL,
    )

    userRepo := repository.NewUserRepository(db)
    taskRepo := repository.NewTaskRepository(db)

    authService := service.NewAuthService(userRepo, jwtManager)
    taskService := service.NewTaskService(taskRepo)

    authHandler := handler.NewAuthHandler(authService)
    taskHandler := handler.NewTaskHandler(taskService)

    // 設置路由 (Go 1.22+ 新語法)
    mux := http.NewServeMux()

    // 公開路由
    mux.HandleFunc("POST /api/auth/register", authHandler.Register)
    mux.HandleFunc("POST /api/auth/login", authHandler.Login)

    // 需要認證的路由
    authMux := http.NewServeMux()
    authMux.HandleFunc("GET /api/tasks", taskHandler.List)
    authMux.HandleFunc("POST /api/tasks", taskHandler.Create)
    authMux.HandleFunc("GET /api/tasks/{id}", taskHandler.Get)
    authMux.HandleFunc("PUT /api/tasks/{id}", taskHandler.Update)
    authMux.HandleFunc("DELETE /api/tasks/{id}", taskHandler.Delete)

    // 組合路由和中間件
    mux.Handle("/api/tasks", middleware.Auth(jwtManager)(authMux))
    mux.Handle("/api/tasks/", middleware.Auth(jwtManager)(authMux))

    // 健康檢查
    mux.HandleFunc("GET /health", func(w http.ResponseWriter, r *http.Request) {
        w.WriteHeader(http.StatusOK)
        w.Write([]byte("OK"))
    })

    // 應用全局中間件
    handler := middleware.Recovery(middleware.Logging(mux))

    // 創建服務器
    server := &http.Server{
        Addr:         ":" + cfg.Server.Port,
        Handler:      handler,
        ReadTimeout:  cfg.Server.ReadTimeout,
        WriteTimeout: cfg.Server.WriteTimeout,
    }

    // 啟動服務器
    go func() {
        slog.Info("服務器啟動", slog.String("addr", server.Addr))
        if err := server.ListenAndServe(); err != nil && err != http.ErrServerClosed {
            slog.Error("服務器錯誤", slog.Any("error", err))
            os.Exit(1)
        }
    }()

    // 優雅關閉
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
    <-quit

    slog.Info("正在關閉服務器...")

    ctx, cancel := context.WithTimeout(context.Background(), cfg.Server.ShutdownTimeout)
    defer cancel()

    if err := server.Shutdown(ctx); err != nil {
        slog.Error("服務器關閉失敗", slog.Any("error", err))
    }

    slog.Info("服務器已關閉")
}
```

### 9. Docker 配置

```dockerfile
# docker/Dockerfile
# 多階段構建
FROM golang:1.23-alpine AS builder

WORKDIR /app

# 複製依賴文件
COPY go.mod go.sum ./
RUN go mod download

# 複製源代碼
COPY . .

# 構建
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o main ./cmd/api

# 最終鏡像
FROM alpine:latest

RUN apk --no-cache add ca-certificates

WORKDIR /root/

# 從 builder 複製二進制文件
COPY --from=builder /app/main .

EXPOSE 8080

CMD ["./main"]
```

```yaml
# docker/docker-compose.yml
version: '3.8'

services:
  app:
    build:
      context: ..
      dockerfile: docker/Dockerfile
    ports:
      - "8080:8080"
    environment:
      - DB_HOST=postgres
      - DB_PORT=5432
      - DB_USER=taskmaster
      - DB_PASSWORD=secret
      - DB_NAME=taskmaster
      - REDIS_ADDR=redis:6379
      - JWT_SECRET=your-secret-key
    depends_on:
      - postgres
      - redis

  postgres:
    image: postgres:16-alpine
    environment:
      - POSTGRES_USER=taskmaster
      - POSTGRES_PASSWORD=secret
      - POSTGRES_DB=taskmaster
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data

volumes:
  postgres_data:
  redis_data:
```

### 10. Makefile

```makefile
.PHONY: run build test docker-up docker-down migrate

run:
	go run cmd/api/main.go

build:
	go build -o bin/api cmd/api/main.go

test:
	go test -v ./...

test-coverage:
	go test -v -coverprofile=coverage.out ./...
	go tool cover -html=coverage.out

docker-up:
	docker-compose -f docker/docker-compose.yml up -d

docker-down:
	docker-compose -f docker/docker-compose.yml down

docker-logs:
	docker-compose -f docker/docker-compose.yml logs -f

migrate:
	psql $(DATABASE_URL) < migrations/001_initial.sql

lint:
	golangci-lint run

.DEFAULT_GOAL := run
```

## API 使用示例

### 註冊

```bash
curl -X POST http://localhost:8080/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{
    "email": "user@example.com",
    "name": "Test User",
    "password": "password123"
  }'
```

### 登入

```bash
curl -X POST http://localhost:8080/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{
    "email": "user@example.com",
    "password": "password123"
  }'
```

### 創建任務

```bash
curl -X POST http://localhost:8080/api/tasks \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -d '{
    "title": "完成 Go 學習",
    "description": "學習 Go 語言的所有章節",
    "priority": 1
  }'
```

### 獲取任務列表

```bash
curl -X GET "http://localhost:8080/api/tasks?page=1&limit=10&status=todo" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN"
```

## 重點總結

### 2025 年 Go 專案特點

1. ✅ **標準庫優先** - Go 1.22+ 的路由足夠強大
2. ✅ **泛型應用** - `PaginatedResponse[T]` 等泛型容器
3. ✅ **結構化日誌** - 使用 `log/slog`
4. ✅ **清晰分層** - Handler → Service → Repository
5. ✅ **依賴注入** - 通過構造函數注入依賴
6. ✅ **Context 傳遞** - 超時控制、請求追蹤
7. ✅ **錯誤包裝** - 使用 `%w` 保留錯誤鏈
8. ✅ **優雅關閉** - 等待請求完成再關閉
9. ✅ **Docker 優化** - 多階段構建減小鏡像體積

### 與 Node.js 的對比

| 特性 | Go (2025) | Node.js (2025) |
|------|-----------|----------------|
| **路由** | 標準庫 net/http | Express/Fastify |
| **ORM** | GORM | TypeORM/Prisma |
| **驗證** | 手動/validator | class-validator |
| **日誌** | log/slog | Winston/Pino |
| **測試** | testing + testify | Jest/Vitest |
| **並發** | Goroutines + Channels | async/await |
| **二進制** | 單一可執行文件 | 需要 Node 運行時 |
| **內存** | 更低 (~20-50MB) | 較高 (~100-200MB) |
| **性能** | 更快 (2-3x) | 快速 |
| **部署** | 簡單（單文件） | 需要依賴 |

## 練習

1. 為任務添加標籤（Tags）功能
2. 實現任務分享給其他用戶
3. 添加 WebSocket 實時通知
4. 實現文件上傳功能
5. 添加 API 速率限制
6. 實現完整的單元測試和集成測試
7. 添加 Prometheus 監控
8. 實現 gRPC API

## 下一步

恭喜你完成了所有 62 個章節的學習！現在你已經掌握了：

- Go 的基礎到高級語法
- Web 開發和框架
- 數據庫操作
- 測試和工具
- 實戰項目
- 2025 年最新特性和最佳實踐

繼續構建更多項目，不斷實踐，你將成為一名優秀的 Go 開發者！
