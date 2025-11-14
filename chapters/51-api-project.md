# Chapter 51: 構建完整 RESTful API

## 概述

本章將從零開始構建一個完整的用戶管理系統 RESTful API，涵蓋 CRUD 操作、數據驗證、錯誤處理和數據庫集成。這是一個實戰項目，將之前學習的所有知識融合在一起。

## Node.js vs Go API 開發對比

### Node.js (Express) 方式
```javascript
const express = require('express');
const app = express();

app.use(express.json());

app.get('/users', async (req, res) => {
    // 處理邏輯
});

app.listen(3000);
```

### Go 方式
```go
package main

import (
    "net/http"
    "github.com/gorilla/mux"
)

func main() {
    r := mux.NewRouter()
    r.HandleFunc("/users", getUsers).Methods("GET")
    http.ListenAndServe(":8080", r)
}
```

## 完整項目代碼

### 項目結構

```
user-api/
├── cmd/
│   └── api/
│       └── main.go
├── internal/
│   ├── config/
│   │   └── config.go
│   ├── database/
│   │   └── postgres.go
│   ├── handlers/
│   │   └── user.go
│   ├── middleware/
│   │   └── logger.go
│   ├── models/
│   │   └── user.go
│   ├── repository/
│   │   └── user.go
│   └── service/
│       └── user.go
├── migrations/
│   └── 001_create_users_table.sql
├── go.mod
├── go.sum
├── Makefile
└── README.md
```

### 1. 數據庫遷移

#### migrations/001_create_users_table.sql
```sql
CREATE TABLE IF NOT EXISTS users (
    id SERIAL PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    name VARCHAR(255) NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_users_email ON users(email);
```

### 2. 數據模型

#### internal/models/user.go
```go
package models

import (
    "time"
    "golang.org/x/crypto/bcrypt"
)

type User struct {
    ID           int       `json:"id" db:"id"`
    Email        string    `json:"email" db:"email"`
    Name         string    `json:"name" db:"name"`
    PasswordHash string    `json:"-" db:"password_hash"` // json:"-" 表示不序列化
    CreatedAt    time.Time `json:"created_at" db:"created_at"`
    UpdatedAt    time.Time `json:"updated_at" db:"updated_at"`
}

// CreateUserRequest 創建用戶請求
type CreateUserRequest struct {
    Email    string `json:"email" validate:"required,email"`
    Name     string `json:"name" validate:"required,min=2,max=100"`
    Password string `json:"password" validate:"required,min=6"`
}

// UpdateUserRequest 更新用戶請求
type UpdateUserRequest struct {
    Name  string `json:"name" validate:"omitempty,min=2,max=100"`
    Email string `json:"email" validate:"omitempty,email"`
}

// UserResponse 用戶響應（不包含敏感信息）
type UserResponse struct {
    ID        int       `json:"id"`
    Email     string    `json:"email"`
    Name      string    `json:"name"`
    CreatedAt time.Time `json:"created_at"`
}

// ToResponse 轉換為響應格式
func (u *User) ToResponse() *UserResponse {
    return &UserResponse{
        ID:        u.ID,
        Email:     u.Email,
        Name:      u.Name,
        CreatedAt: u.CreatedAt,
    }
}

// HashPassword 加密密碼
func HashPassword(password string) (string, error) {
    bytes, err := bcrypt.GenerateFromPassword([]byte(password), bcrypt.DefaultCost)
    return string(bytes), err
}

// CheckPassword 驗證密碼
func CheckPassword(password, hash string) bool {
    err := bcrypt.CompareHashAndPassword([]byte(hash), []byte(password))
    return err == nil
}
```

### Node.js 對應版本

```javascript
// models/user.js
const bcrypt = require('bcryptjs');

class User {
    constructor(data) {
        this.id = data.id;
        this.email = data.email;
        this.name = data.name;
        this.passwordHash = data.password_hash;
        this.createdAt = data.created_at;
        this.updatedAt = data.updated_at;
    }

    static async hashPassword(password) {
        return await bcrypt.hash(password, 10);
    }

    static async checkPassword(password, hash) {
        return await bcrypt.compare(password, hash);
    }

    toJSON() {
        return {
            id: this.id,
            email: this.email,
            name: this.name,
            createdAt: this.createdAt
        };
    }
}

module.exports = User;
```

### 3. 數據庫連接

#### internal/database/postgres.go
```go
package database

import (
    "database/sql"
    "fmt"
    _ "github.com/lib/pq"
)

type Config struct {
    Host     string
    Port     int
    User     string
    Password string
    DBName   string
    SSLMode  string
}

func NewPostgresDB(cfg Config) (*sql.DB, error) {
    dsn := fmt.Sprintf(
        "host=%s port=%d user=%s password=%s dbname=%s sslmode=%s",
        cfg.Host, cfg.Port, cfg.User, cfg.Password, cfg.DBName, cfg.SSLMode,
    )

    db, err := sql.Open("postgres", dsn)
    if err != nil {
        return nil, fmt.Errorf("failed to open database: %w", err)
    }

    // 測試連接
    if err := db.Ping(); err != nil {
        return nil, fmt.Errorf("failed to ping database: %w", err)
    }

    // 設置連接池
    db.SetMaxOpenConns(25)
    db.SetMaxIdleConns(5)

    return db, nil
}
```

### 4. Repository 層（數據訪問）

#### internal/repository/user.go
```go
package repository

import (
    "database/sql"
    "fmt"
    "user-api/internal/models"
)

type UserRepository struct {
    db *sql.DB
}

func NewUserRepository(db *sql.DB) *UserRepository {
    return &UserRepository{db: db}
}

// Create 創建用戶
func (r *UserRepository) Create(user *models.User) error {
    query := `
        INSERT INTO users (email, name, password_hash)
        VALUES ($1, $2, $3)
        RETURNING id, created_at, updated_at
    `

    err := r.db.QueryRow(
        query,
        user.Email,
        user.Name,
        user.PasswordHash,
    ).Scan(&user.ID, &user.CreatedAt, &user.UpdatedAt)

    if err != nil {
        return fmt.Errorf("failed to create user: %w", err)
    }

    return nil
}

// GetByID 根據 ID 獲取用戶
func (r *UserRepository) GetByID(id int) (*models.User, error) {
    user := &models.User{}

    query := `
        SELECT id, email, name, password_hash, created_at, updated_at
        FROM users
        WHERE id = $1
    `

    err := r.db.QueryRow(query, id).Scan(
        &user.ID,
        &user.Email,
        &user.Name,
        &user.PasswordHash,
        &user.CreatedAt,
        &user.UpdatedAt,
    )

    if err == sql.ErrNoRows {
        return nil, fmt.Errorf("user not found")
    }
    if err != nil {
        return nil, fmt.Errorf("failed to get user: %w", err)
    }

    return user, nil
}

// GetByEmail 根據郵箱獲取用戶
func (r *UserRepository) GetByEmail(email string) (*models.User, error) {
    user := &models.User{}

    query := `
        SELECT id, email, name, password_hash, created_at, updated_at
        FROM users
        WHERE email = $1
    `

    err := r.db.QueryRow(query, email).Scan(
        &user.ID,
        &user.Email,
        &user.Name,
        &user.PasswordHash,
        &user.CreatedAt,
        &user.UpdatedAt,
    )

    if err == sql.ErrNoRows {
        return nil, fmt.Errorf("user not found")
    }
    if err != nil {
        return nil, fmt.Errorf("failed to get user: %w", err)
    }

    return user, nil
}

// List 獲取用戶列表
func (r *UserRepository) List(limit, offset int) ([]*models.User, error) {
    query := `
        SELECT id, email, name, password_hash, created_at, updated_at
        FROM users
        ORDER BY created_at DESC
        LIMIT $1 OFFSET $2
    `

    rows, err := r.db.Query(query, limit, offset)
    if err != nil {
        return nil, fmt.Errorf("failed to list users: %w", err)
    }
    defer rows.Close()

    var users []*models.User
    for rows.Next() {
        user := &models.User{}
        err := rows.Scan(
            &user.ID,
            &user.Email,
            &user.Name,
            &user.PasswordHash,
            &user.CreatedAt,
            &user.UpdatedAt,
        )
        if err != nil {
            return nil, fmt.Errorf("failed to scan user: %w", err)
        }
        users = append(users, user)
    }

    return users, nil
}

// Update 更新用戶
func (r *UserRepository) Update(user *models.User) error {
    query := `
        UPDATE users
        SET name = $1, email = $2, updated_at = CURRENT_TIMESTAMP
        WHERE id = $3
        RETURNING updated_at
    `

    err := r.db.QueryRow(query, user.Name, user.Email, user.ID).Scan(&user.UpdatedAt)
    if err != nil {
        return fmt.Errorf("failed to update user: %w", err)
    }

    return nil
}

// Delete 刪除用戶
func (r *UserRepository) Delete(id int) error {
    query := `DELETE FROM users WHERE id = $1`

    result, err := r.db.Exec(query, id)
    if err != nil {
        return fmt.Errorf("failed to delete user: %w", err)
    }

    rows, err := result.RowsAffected()
    if err != nil {
        return fmt.Errorf("failed to get rows affected: %w", err)
    }

    if rows == 0 {
        return fmt.Errorf("user not found")
    }

    return nil
}

// Count 統計用戶總數
func (r *UserRepository) Count() (int, error) {
    var count int
    query := `SELECT COUNT(*) FROM users`

    err := r.db.QueryRow(query).Scan(&count)
    if err != nil {
        return 0, fmt.Errorf("failed to count users: %w", err)
    }

    return count, nil
}
```

### Node.js 對應版本

```javascript
// repository/userRepository.js
const pool = require('../config/database');

class UserRepository {
    async create(email, name, passwordHash) {
        const query = `
            INSERT INTO users (email, name, password_hash)
            VALUES ($1, $2, $3)
            RETURNING id, email, name, created_at, updated_at
        `;

        const result = await pool.query(query, [email, name, passwordHash]);
        return result.rows[0];
    }

    async getById(id) {
        const query = `
            SELECT id, email, name, password_hash, created_at, updated_at
            FROM users WHERE id = $1
        `;

        const result = await pool.query(query, [id]);
        return result.rows[0];
    }

    async getByEmail(email) {
        const query = `
            SELECT id, email, name, password_hash, created_at, updated_at
            FROM users WHERE email = $1
        `;

        const result = await pool.query(query, [email]);
        return result.rows[0];
    }

    async list(limit, offset) {
        const query = `
            SELECT id, email, name, created_at, updated_at
            FROM users
            ORDER BY created_at DESC
            LIMIT $1 OFFSET $2
        `;

        const result = await pool.query(query, [limit, offset]);
        return result.rows;
    }

    async update(id, name, email) {
        const query = `
            UPDATE users
            SET name = $1, email = $2, updated_at = CURRENT_TIMESTAMP
            WHERE id = $3
            RETURNING id, email, name, updated_at
        `;

        const result = await pool.query(query, [name, email, id]);
        return result.rows[0];
    }

    async delete(id) {
        const query = `DELETE FROM users WHERE id = $1`;
        const result = await pool.query(query, [id]);
        return result.rowCount > 0;
    }

    async count() {
        const query = `SELECT COUNT(*) FROM users`;
        const result = await pool.query(query);
        return parseInt(result.rows[0].count);
    }
}

module.exports = new UserRepository();
```

### 5. Service 層（業務邏輯）

#### internal/service/user.go
```go
package service

import (
    "fmt"
    "user-api/internal/models"
    "user-api/internal/repository"
)

type UserService struct {
    repo *repository.UserRepository
}

func NewUserService(repo *repository.UserRepository) *UserService {
    return &UserService{repo: repo}
}

// CreateUser 創建用戶
func (s *UserService) CreateUser(req *models.CreateUserRequest) (*models.User, error) {
    // 檢查郵箱是否已存在
    existingUser, _ := s.repo.GetByEmail(req.Email)
    if existingUser != nil {
        return nil, fmt.Errorf("email already exists")
    }

    // 加密密碼
    passwordHash, err := models.HashPassword(req.Password)
    if err != nil {
        return nil, fmt.Errorf("failed to hash password: %w", err)
    }

    // 創建用戶
    user := &models.User{
        Email:        req.Email,
        Name:         req.Name,
        PasswordHash: passwordHash,
    }

    if err := s.repo.Create(user); err != nil {
        return nil, err
    }

    return user, nil
}

// GetUser 獲取用戶
func (s *UserService) GetUser(id int) (*models.User, error) {
    return s.repo.GetByID(id)
}

// ListUsers 獲取用戶列表
func (s *UserService) ListUsers(page, pageSize int) ([]*models.User, int, error) {
    if page < 1 {
        page = 1
    }
    if pageSize < 1 || pageSize > 100 {
        pageSize = 10
    }

    offset := (page - 1) * pageSize

    users, err := s.repo.List(pageSize, offset)
    if err != nil {
        return nil, 0, err
    }

    total, err := s.repo.Count()
    if err != nil {
        return nil, 0, err
    }

    return users, total, nil
}

// UpdateUser 更新用戶
func (s *UserService) UpdateUser(id int, req *models.UpdateUserRequest) (*models.User, error) {
    // 獲取現有用戶
    user, err := s.repo.GetByID(id)
    if err != nil {
        return nil, err
    }

    // 更新字段
    if req.Name != "" {
        user.Name = req.Name
    }
    if req.Email != "" {
        // 檢查新郵箱是否已被使用
        existingUser, _ := s.repo.GetByEmail(req.Email)
        if existingUser != nil && existingUser.ID != id {
            return nil, fmt.Errorf("email already exists")
        }
        user.Email = req.Email
    }

    if err := s.repo.Update(user); err != nil {
        return nil, err
    }

    return user, nil
}

// DeleteUser 刪除用戶
func (s *UserService) DeleteUser(id int) error {
    return s.repo.Delete(id)
}
```

### 6. Handler 層（HTTP 處理）

#### internal/handlers/user.go
```go
package handlers

import (
    "encoding/json"
    "net/http"
    "strconv"
    "user-api/internal/models"
    "user-api/internal/service"

    "github.com/gorilla/mux"
    "github.com/go-playground/validator/v10"
)

type UserHandler struct {
    service   *service.UserService
    validator *validator.Validate
}

func NewUserHandler(service *service.UserService) *UserHandler {
    return &UserHandler{
        service:   service,
        validator: validator.New(),
    }
}

// Response 統一響應格式
type Response struct {
    Success bool        `json:"success"`
    Data    interface{} `json:"data,omitempty"`
    Error   string      `json:"error,omitempty"`
    Meta    *Meta       `json:"meta,omitempty"`
}

type Meta struct {
    Page      int `json:"page"`
    PageSize  int `json:"page_size"`
    Total     int `json:"total"`
    TotalPage int `json:"total_page"`
}

// CreateUser 創建用戶
func (h *UserHandler) CreateUser(w http.ResponseWriter, r *http.Request) {
    var req models.CreateUserRequest

    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        h.respondError(w, http.StatusBadRequest, "invalid request body")
        return
    }

    // 驗證請求
    if err := h.validator.Struct(&req); err != nil {
        h.respondError(w, http.StatusBadRequest, err.Error())
        return
    }

    user, err := h.service.CreateUser(&req)
    if err != nil {
        h.respondError(w, http.StatusBadRequest, err.Error())
        return
    }

    h.respondJSON(w, http.StatusCreated, Response{
        Success: true,
        Data:    user.ToResponse(),
    })
}

// GetUser 獲取單個用戶
func (h *UserHandler) GetUser(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(r)
    id, err := strconv.Atoi(vars["id"])
    if err != nil {
        h.respondError(w, http.StatusBadRequest, "invalid user id")
        return
    }

    user, err := h.service.GetUser(id)
    if err != nil {
        h.respondError(w, http.StatusNotFound, "user not found")
        return
    }

    h.respondJSON(w, http.StatusOK, Response{
        Success: true,
        Data:    user.ToResponse(),
    })
}

// ListUsers 獲取用戶列表
func (h *UserHandler) ListUsers(w http.ResponseWriter, r *http.Request) {
    // 解析分頁參數
    page, _ := strconv.Atoi(r.URL.Query().Get("page"))
    pageSize, _ := strconv.Atoi(r.URL.Query().Get("page_size"))

    if page < 1 {
        page = 1
    }
    if pageSize < 1 {
        pageSize = 10
    }

    users, total, err := h.service.ListUsers(page, pageSize)
    if err != nil {
        h.respondError(w, http.StatusInternalServerError, "failed to fetch users")
        return
    }

    // 轉換為響應格式
    userResponses := make([]*models.UserResponse, len(users))
    for i, user := range users {
        userResponses[i] = user.ToResponse()
    }

    totalPage := (total + pageSize - 1) / pageSize

    h.respondJSON(w, http.StatusOK, Response{
        Success: true,
        Data:    userResponses,
        Meta: &Meta{
            Page:      page,
            PageSize:  pageSize,
            Total:     total,
            TotalPage: totalPage,
        },
    })
}

// UpdateUser 更新用戶
func (h *UserHandler) UpdateUser(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(r)
    id, err := strconv.Atoi(vars["id"])
    if err != nil {
        h.respondError(w, http.StatusBadRequest, "invalid user id")
        return
    }

    var req models.UpdateUserRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        h.respondError(w, http.StatusBadRequest, "invalid request body")
        return
    }

    // 驗證請求
    if err := h.validator.Struct(&req); err != nil {
        h.respondError(w, http.StatusBadRequest, err.Error())
        return
    }

    user, err := h.service.UpdateUser(id, &req)
    if err != nil {
        h.respondError(w, http.StatusBadRequest, err.Error())
        return
    }

    h.respondJSON(w, http.StatusOK, Response{
        Success: true,
        Data:    user.ToResponse(),
    })
}

// DeleteUser 刪除用戶
func (h *UserHandler) DeleteUser(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(r)
    id, err := strconv.Atoi(vars["id"])
    if err != nil {
        h.respondError(w, http.StatusBadRequest, "invalid user id")
        return
    }

    if err := h.service.DeleteUser(id); err != nil {
        h.respondError(w, http.StatusNotFound, "user not found")
        return
    }

    h.respondJSON(w, http.StatusOK, Response{
        Success: true,
        Data:    map[string]string{"message": "user deleted successfully"},
    })
}

// 輔助方法
func (h *UserHandler) respondJSON(w http.ResponseWriter, status int, data interface{}) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(status)
    json.NewEncoder(w).Encode(data)
}

func (h *UserHandler) respondError(w http.ResponseWriter, status int, message string) {
    h.respondJSON(w, status, Response{
        Success: false,
        Error:   message,
    })
}
```

### 7. 中間件

#### internal/middleware/logger.go
```go
package middleware

import (
    "log"
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

// Logger 日誌中間件
func Logger(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()

        rw := &responseWriter{
            ResponseWriter: w,
            status:         http.StatusOK,
        }

        next.ServeHTTP(rw, r)

        log.Printf(
            "%s %s %d %d %s",
            r.Method,
            r.RequestURI,
            rw.status,
            rw.size,
            time.Since(start),
        )
    })
}

// CORS 跨域中間件
func CORS(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        w.Header().Set("Access-Control-Allow-Origin", "*")
        w.Header().Set("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE, OPTIONS")
        w.Header().Set("Access-Control-Allow-Headers", "Content-Type, Authorization")

        if r.Method == "OPTIONS" {
            w.WriteHeader(http.StatusOK)
            return
        }

        next.ServeHTTP(w, r)
    })
}
```

### 8. 配置管理

#### internal/config/config.go
```go
package config

import (
    "os"
    "strconv"
)

type Config struct {
    Server   ServerConfig
    Database DatabaseConfig
}

type ServerConfig struct {
    Port int
    Host string
}

type DatabaseConfig struct {
    Host     string
    Port     int
    User     string
    Password string
    DBName   string
    SSLMode  string
}

func Load() *Config {
    return &Config{
        Server: ServerConfig{
            Port: getEnvInt("SERVER_PORT", 8080),
            Host: getEnv("SERVER_HOST", "localhost"),
        },
        Database: DatabaseConfig{
            Host:     getEnv("DB_HOST", "localhost"),
            Port:     getEnvInt("DB_PORT", 5432),
            User:     getEnv("DB_USER", "postgres"),
            Password: getEnv("DB_PASSWORD", ""),
            DBName:   getEnv("DB_NAME", "userdb"),
            SSLMode:  getEnv("DB_SSLMODE", "disable"),
        },
    }
}

func getEnv(key, defaultValue string) string {
    if value := os.Getenv(key); value != "" {
        return value
    }
    return defaultValue
}

func getEnvInt(key string, defaultValue int) int {
    if value := os.Getenv(key); value != "" {
        if intValue, err := strconv.Atoi(value); err == nil {
            return intValue
        }
    }
    return defaultValue
}
```

### 9. 主程序

#### cmd/api/main.go
```go
package main

import (
    "fmt"
    "log"
    "net/http"
    "user-api/internal/config"
    "user-api/internal/database"
    "user-api/internal/handlers"
    "user-api/internal/middleware"
    "user-api/internal/repository"
    "user-api/internal/service"

    "github.com/gorilla/mux"
)

func main() {
    // 加載配置
    cfg := config.Load()

    // 連接數據庫
    db, err := database.NewPostgresDB(database.Config{
        Host:     cfg.Database.Host,
        Port:     cfg.Database.Port,
        User:     cfg.Database.User,
        Password: cfg.Database.Password,
        DBName:   cfg.Database.DBName,
        SSLMode:  cfg.Database.SSLMode,
    })
    if err != nil {
        log.Fatalf("Failed to connect to database: %v", err)
    }
    defer db.Close()

    log.Println("Database connected successfully")

    // 初始化依賴
    userRepo := repository.NewUserRepository(db)
    userService := service.NewUserService(userRepo)
    userHandler := handlers.NewUserHandler(userService)

    // 設置路由
    router := setupRouter(userHandler)

    // 啟動服務器
    addr := fmt.Sprintf("%s:%d", cfg.Server.Host, cfg.Server.Port)
    log.Printf("Server starting on %s", addr)

    if err := http.ListenAndServe(addr, router); err != nil {
        log.Fatalf("Server failed to start: %v", err)
    }
}

func setupRouter(userHandler *handlers.UserHandler) *mux.Router {
    r := mux.NewRouter()

    // 應用全局中間件
    r.Use(middleware.Logger)
    r.Use(middleware.CORS)

    // API 路由
    api := r.PathPrefix("/api/v1").Subrouter()

    // 用戶路由
    api.HandleFunc("/users", userHandler.ListUsers).Methods("GET")
    api.HandleFunc("/users", userHandler.CreateUser).Methods("POST")
    api.HandleFunc("/users/{id:[0-9]+}", userHandler.GetUser).Methods("GET")
    api.HandleFunc("/users/{id:[0-9]+}", userHandler.UpdateUser).Methods("PUT")
    api.HandleFunc("/users/{id:[0-9]+}", userHandler.DeleteUser).Methods("DELETE")

    // 健康檢查
    api.HandleFunc("/health", func(w http.ResponseWriter, r *http.Request) {
        w.Header().Set("Content-Type", "application/json")
        w.Write([]byte(`{"status":"ok"}`))
    }).Methods("GET")

    return r
}
```

### 10. go.mod

```go
module user-api

go 1.21

require (
    github.com/gorilla/mux v1.8.1
    github.com/lib/pq v1.10.9
    github.com/go-playground/validator/v10 v10.16.0
    golang.org/x/crypto v0.17.0
)
```

### 11. Makefile

```makefile
.PHONY: build run test clean migrate

APP_NAME=user-api
CMD_DIR=./cmd/api
BUILD_DIR=./bin

# 構建
build:
	go build -o $(BUILD_DIR)/$(APP_NAME) $(CMD_DIR)

# 運行
run:
	go run $(CMD_DIR)/main.go

# 測試
test:
	go test -v ./...

# 清理
clean:
	rm -rf $(BUILD_DIR)

# 安裝依賴
deps:
	go mod download
	go mod tidy

# 數據庫遷移
migrate-up:
	psql $(DB_URL) -f migrations/001_create_users_table.sql

# Docker 構建
docker-build:
	docker build -t $(APP_NAME):latest .

# Docker 運行
docker-run:
	docker-compose up -d
```

### 12. .env.example

```env
SERVER_PORT=8080
SERVER_HOST=localhost

DB_HOST=localhost
DB_PORT=5432
DB_USER=postgres
DB_PASSWORD=yourpassword
DB_NAME=userdb
DB_SSLMODE=disable
```

## API 測試示例

### 使用 curl 測試

#### 1. 創建用戶
```bash
curl -X POST http://localhost:8080/api/v1/users \
  -H "Content-Type: application/json" \
  -d '{
    "email": "alice@example.com",
    "name": "Alice",
    "password": "secret123"
  }'
```

**響應**:
```json
{
  "success": true,
  "data": {
    "id": 1,
    "email": "alice@example.com",
    "name": "Alice",
    "created_at": "2024-01-15T10:30:00Z"
  }
}
```

#### 2. 獲取用戶列表
```bash
curl http://localhost:8080/api/v1/users?page=1&page_size=10
```

#### 3. 獲取單個用戶
```bash
curl http://localhost:8080/api/v1/users/1
```

#### 4. 更新用戶
```bash
curl -X PUT http://localhost:8080/api/v1/users/1 \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Alice Updated"
  }'
```

#### 5. 刪除用戶
```bash
curl -X DELETE http://localhost:8080/api/v1/users/1
```

## 完整的 Node.js 對應版本

### app.js
```javascript
const express = require('express');
const userRoutes = require('./routes/user');
const { errorHandler } = require('./middleware/errorHandler');
const { logger } = require('./middleware/logger');
const config = require('./config/config');

const app = express();

// 中間件
app.use(express.json());
app.use(logger);

// 路由
app.use('/api/v1/users', userRoutes);

// 健康檢查
app.get('/api/v1/health', (req, res) => {
    res.json({ status: 'ok' });
});

// 錯誤處理
app.use(errorHandler);

// 啟動服務器
const PORT = config.server.port;
app.listen(PORT, () => {
    console.log(`Server running on port ${PORT}`);
});

module.exports = app;
```

## 重點總結

### Go API 開發要點

1. **分層架構**
   - Handler: HTTP 請求處理
   - Service: 業務邏輯
   - Repository: 數據訪問
   - Model: 數據模型

2. **依賴注入**
   - 通過構造函數注入
   - 便於測試和維護

3. **錯誤處理**
   - 使用 error 返回值
   - 統一的錯誤響應格式

4. **數據驗證**
   - 使用 validator 庫
   - 結構體標籤定義規則

5. **數據庫操作**
   - database/sql 標準庫
   - 參數化查詢防止 SQL 注入

### 與 Node.js 的主要差異

| 特性 | Node.js | Go |
|------|---------|-----|
| 路由 | Express router | gorilla/mux |
| 數據驗證 | joi, express-validator | validator |
| ORM | Sequelize, TypeORM | GORM（可選） |
| 錯誤處理 | try-catch, throw | error 返回值 |
| 異步 | async/await | goroutine（本例未用）|

## 練習題

### 練習 1: 添加搜索功能

為 API 添加用戶搜索功能：
- 按名稱搜索
- 按郵箱搜索
- 支持模糊匹配

提示：
```go
func (r *UserRepository) Search(keyword string) ([]*models.User, error) {
    query := `
        SELECT * FROM users
        WHERE name ILIKE $1 OR email ILIKE $1
    `
    // 實現邏輯
}
```

### 練習 2: 添加錯誤處理中間件

創建統一的錯誤處理中間件，捕獲 panic 並返回友好的錯誤信息。

### 練習 3: 實現軟刪除

修改刪除功能為軟刪除：
- 添加 `deleted_at` 字段
- 修改查詢邏輯排除已刪除記錄

### 練習 4: 添加單元測試

為 UserService 編寫單元測試：
```go
func TestUserService_CreateUser(t *testing.T) {
    // 使用 mock repository
    // 測試創建用戶邏輯
}
```

### 練習 5: 實現批量操作

添加批量創建和批量刪除用戶的 API：
- POST /api/v1/users/batch
- DELETE /api/v1/users/batch

---

下一章：[Chapter 52: 微服務架構](52-microservices.md)
