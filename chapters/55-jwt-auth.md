# Chapter 55: JWT 認證

## 概述

JWT (JSON Web Token) 是一種開放標準 (RFC 7519)，用於在各方之間安全地傳輸信息。本章將介紹如何在 Go 中實現基於 JWT 的認證系統，並與 Node.js 的 passport.js 和 jsonwebtoken 進行對比。

## JWT 基礎

### JWT 結構

JWT 由三部分組成，用點號（.）分隔：

```
Header.Payload.Signature
```

#### 示例 JWT
```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
```

#### 解碼後：

**Header**:
```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

**Payload**:
```json
{
  "sub": "1234567890",
  "name": "John Doe",
  "iat": 1516239022
}
```

**Signature**:
```
HMACSHA256(
  base64UrlEncode(header) + "." +
  base64UrlEncode(payload),
  secret
)
```

## Node.js vs Go JWT 對比

### Node.js (jsonwebtoken)
```javascript
const jwt = require('jsonwebtoken');

// 生成 token
const token = jwt.sign({ userId: 123 }, 'secret', { expiresIn: '1h' });

// 驗證 token
const decoded = jwt.verify(token, 'secret');
```

### Go (golang-jwt/jwt)
```go
import "github.com/golang-jwt/jwt/v5"

// 生成 token
token := jwt.NewWithClaims(jwt.SigningMethodHS256, jwt.MapClaims{
    "user_id": 123,
    "exp": time.Now().Add(time.Hour).Unix(),
})
tokenString, _ := token.SignedString([]byte("secret"))

// 驗證 token
token, _ := jwt.Parse(tokenString, func(token *jwt.Token) (interface{}, error) {
    return []byte("secret"), nil
})
```

## 完整的 JWT 認證系統

### 項目結構

```
jwt-auth-api/
├── cmd/
│   └── api/
│       └── main.go
├── internal/
│   ├── auth/
│   │   ├── jwt.go
│   │   ├── middleware.go
│   │   └── service.go
│   ├── handlers/
│   │   ├── auth.go
│   │   └── user.go
│   ├── models/
│   │   └── user.go
│   └── repository/
│       └── user.go
├── config/
│   └── config.go
└── go.mod
```

### 1. JWT 工具

#### internal/auth/jwt.go
```go
package auth

import (
    "fmt"
    "time"

    "github.com/golang-jwt/jwt/v5"
)

// Claims JWT 聲明
type Claims struct {
    UserID   int    `json:"user_id"`
    Username string `json:"username"`
    Email    string `json:"email"`
    jwt.RegisteredClaims
}

// JWTManager JWT 管理器
type JWTManager struct {
    secretKey     []byte
    tokenDuration time.Duration
}

// NewJWTManager 創建 JWT 管理器
func NewJWTManager(secretKey string, tokenDuration time.Duration) *JWTManager {
    return &JWTManager{
        secretKey:     []byte(secretKey),
        tokenDuration: tokenDuration,
    }
}

// GenerateToken 生成 token
func (m *JWTManager) GenerateToken(userID int, username, email string) (string, error) {
    claims := &Claims{
        UserID:   userID,
        Username: username,
        Email:    email,
        RegisteredClaims: jwt.RegisteredClaims{
            ExpiresAt: jwt.NewNumericDate(time.Now().Add(m.tokenDuration)),
            IssuedAt:  jwt.NewNumericDate(time.Now()),
            NotBefore: jwt.NewNumericDate(time.Now()),
            Issuer:    "jwt-auth-api",
            Subject:   fmt.Sprintf("%d", userID),
        },
    }

    token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
    return token.SignedString(m.secretKey)
}

// VerifyToken 驗證 token
func (m *JWTManager) VerifyToken(tokenString string) (*Claims, error) {
    token, err := jwt.ParseWithClaims(
        tokenString,
        &Claims{},
        func(token *jwt.Token) (interface{}, error) {
            // 驗證簽名算法
            if _, ok := token.Method.(*jwt.SigningMethodHMAC); !ok {
                return nil, fmt.Errorf("unexpected signing method: %v", token.Header["alg"])
            }
            return m.secretKey, nil
        },
    )

    if err != nil {
        return nil, fmt.Errorf("invalid token: %w", err)
    }

    claims, ok := token.Claims.(*Claims)
    if !ok || !token.Valid {
        return nil, fmt.Errorf("invalid token claims")
    }

    return claims, nil
}

// RefreshToken 刷新 token
func (m *JWTManager) RefreshToken(tokenString string) (string, error) {
    claims, err := m.VerifyToken(tokenString)
    if err != nil {
        return "", err
    }

    // 如果 token 還有超過 30 分鐘才過期，不刷新
    if time.Until(claims.ExpiresAt.Time) > 30*time.Minute {
        return tokenString, nil
    }

    // 生成新 token
    return m.GenerateToken(claims.UserID, claims.Username, claims.Email)
}
```

### 2. 認證中間件

#### internal/auth/middleware.go
```go
package auth

import (
    "context"
    "net/http"
    "strings"
)

type contextKey string

const (
    // ContextKeyUserID 用戶 ID 上下文鍵
    ContextKeyUserID contextKey = "user_id"
    // ContextKeyUsername 用戶名上下文鍵
    ContextKeyUsername contextKey = "username"
)

// Middleware JWT 認證中間件
type Middleware struct {
    jwtManager *JWTManager
}

// NewMiddleware 創建認證中間件
func NewMiddleware(jwtManager *JWTManager) *Middleware {
    return &Middleware{jwtManager: jwtManager}
}

// Authenticate 認證中間件
func (m *Middleware) Authenticate(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // 從 Header 獲取 token
        authHeader := r.Header.Get("Authorization")
        if authHeader == "" {
            http.Error(w, "Missing authorization header", http.StatusUnauthorized)
            return
        }

        // Bearer token
        parts := strings.SplitN(authHeader, " ", 2)
        if len(parts) != 2 || parts[0] != "Bearer" {
            http.Error(w, "Invalid authorization header format", http.StatusUnauthorized)
            return
        }

        tokenString := parts[1]

        // 驗證 token
        claims, err := m.jwtManager.VerifyToken(tokenString)
        if err != nil {
            http.Error(w, "Invalid token", http.StatusUnauthorized)
            return
        }

        // 將用戶信息添加到上下文
        ctx := context.WithValue(r.Context(), ContextKeyUserID, claims.UserID)
        ctx = context.WithValue(ctx, ContextKeyUsername, claims.Username)

        // 調用下一個處理器
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}

// OptionalAuthenticate 可選認證中間件（不強制要求 token）
func (m *Middleware) OptionalAuthenticate(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        authHeader := r.Header.Get("Authorization")
        if authHeader == "" {
            next.ServeHTTP(w, r)
            return
        }

        parts := strings.SplitN(authHeader, " ", 2)
        if len(parts) == 2 && parts[0] == "Bearer" {
            claims, err := m.jwtManager.VerifyToken(parts[1])
            if err == nil {
                ctx := context.WithValue(r.Context(), ContextKeyUserID, claims.UserID)
                ctx = context.WithValue(ctx, ContextKeyUsername, claims.Username)
                r = r.WithContext(ctx)
            }
        }

        next.ServeHTTP(w, r)
    })
}

// GetUserID 從上下文獲取用戶 ID
func GetUserID(ctx context.Context) (int, bool) {
    userID, ok := ctx.Value(ContextKeyUserID).(int)
    return userID, ok
}

// GetUsername 從上下文獲取用戶名
func GetUsername(ctx context.Context) (string, bool) {
    username, ok := ctx.Value(ContextKeyUsername).(string)
    return username, ok
}
```

### 3. 認證服務

#### internal/auth/service.go
```go
package auth

import (
    "fmt"
    "jwt-auth-api/internal/models"
    "jwt-auth-api/internal/repository"

    "golang.org/x/crypto/bcrypt"
)

// Service 認證服務
type Service struct {
    userRepo   *repository.UserRepository
    jwtManager *JWTManager
}

// NewService 創建認證服務
func NewService(userRepo *repository.UserRepository, jwtManager *JWTManager) *Service {
    return &Service{
        userRepo:   userRepo,
        jwtManager: jwtManager,
    }
}

// Register 註冊用戶
func (s *Service) Register(email, username, password string) (*models.User, error) {
    // 檢查用戶是否已存在
    existingUser, _ := s.userRepo.GetByEmail(email)
    if existingUser != nil {
        return nil, fmt.Errorf("email already exists")
    }

    // 加密密碼
    hashedPassword, err := bcrypt.GenerateFromPassword([]byte(password), bcrypt.DefaultCost)
    if err != nil {
        return nil, fmt.Errorf("failed to hash password: %w", err)
    }

    // 創建用戶
    user := &models.User{
        Email:        email,
        Username:     username,
        PasswordHash: string(hashedPassword),
    }

    if err := s.userRepo.Create(user); err != nil {
        return nil, err
    }

    return user, nil
}

// Login 登錄
func (s *Service) Login(email, password string) (string, *models.User, error) {
    // 獲取用戶
    user, err := s.userRepo.GetByEmail(email)
    if err != nil {
        return "", nil, fmt.Errorf("invalid credentials")
    }

    // 驗證密碼
    if err := bcrypt.CompareHashAndPassword([]byte(user.PasswordHash), []byte(password)); err != nil {
        return "", nil, fmt.Errorf("invalid credentials")
    }

    // 生成 token
    token, err := s.jwtManager.GenerateToken(user.ID, user.Username, user.Email)
    if err != nil {
        return "", nil, fmt.Errorf("failed to generate token: %w", err)
    }

    return token, user, nil
}

// RefreshToken 刷新 token
func (s *Service) RefreshToken(tokenString string) (string, error) {
    return s.jwtManager.RefreshToken(tokenString)
}
```

### 4. 認證處理器

#### internal/handlers/auth.go
```go
package handlers

import (
    "encoding/json"
    "net/http"

    "jwt-auth-api/internal/auth"

    "github.com/go-playground/validator/v10"
)

// AuthHandler 認證處理器
type AuthHandler struct {
    authService *auth.Service
    validator   *validator.Validate
}

// NewAuthHandler 創建認證處理器
func NewAuthHandler(authService *auth.Service) *AuthHandler {
    return &AuthHandler{
        authService: authService,
        validator:   validator.New(),
    }
}

// RegisterRequest 註冊請求
type RegisterRequest struct {
    Email    string `json:"email" validate:"required,email"`
    Username string `json:"username" validate:"required,min=3,max=50"`
    Password string `json:"password" validate:"required,min=6"`
}

// LoginRequest 登錄請求
type LoginRequest struct {
    Email    string `json:"email" validate:"required,email"`
    Password string `json:"password" validate:"required"`
}

// RefreshRequest 刷新請求
type RefreshRequest struct {
    Token string `json:"token" validate:"required"`
}

// AuthResponse 認證響應
type AuthResponse struct {
    Token string      `json:"token"`
    User  interface{} `json:"user"`
}

// Register 註冊
func (h *AuthHandler) Register(w http.ResponseWriter, r *http.Request) {
    var req RegisterRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        respondError(w, http.StatusBadRequest, "Invalid request body")
        return
    }

    if err := h.validator.Struct(&req); err != nil {
        respondError(w, http.StatusBadRequest, err.Error())
        return
    }

    user, err := h.authService.Register(req.Email, req.Username, req.Password)
    if err != nil {
        respondError(w, http.StatusBadRequest, err.Error())
        return
    }

    // 自動登錄
    token, _, err := h.authService.Login(req.Email, req.Password)
    if err != nil {
        respondError(w, http.StatusInternalServerError, "Registration succeeded but login failed")
        return
    }

    respondJSON(w, http.StatusCreated, AuthResponse{
        Token: token,
        User: map[string]interface{}{
            "id":       user.ID,
            "email":    user.Email,
            "username": user.Username,
        },
    })
}

// Login 登錄
func (h *AuthHandler) Login(w http.ResponseWriter, r *http.Request) {
    var req LoginRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        respondError(w, http.StatusBadRequest, "Invalid request body")
        return
    }

    if err := h.validator.Struct(&req); err != nil {
        respondError(w, http.StatusBadRequest, err.Error())
        return
    }

    token, user, err := h.authService.Login(req.Email, req.Password)
    if err != nil {
        respondError(w, http.StatusUnauthorized, "Invalid credentials")
        return
    }

    respondJSON(w, http.StatusOK, AuthResponse{
        Token: token,
        User: map[string]interface{}{
            "id":       user.ID,
            "email":    user.Email,
            "username": user.Username,
        },
    })
}

// RefreshToken 刷新 token
func (h *AuthHandler) RefreshToken(w http.ResponseWriter, r *http.Request) {
    var req RefreshRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        respondError(w, http.StatusBadRequest, "Invalid request body")
        return
    }

    token, err := h.authService.RefreshToken(req.Token)
    if err != nil {
        respondError(w, http.StatusUnauthorized, "Invalid token")
        return
    }

    respondJSON(w, http.StatusOK, map[string]string{
        "token": token,
    })
}

// Me 獲取當前用戶信息
func (h *AuthHandler) Me(w http.ResponseWriter, r *http.Request) {
    userID, ok := auth.GetUserID(r.Context())
    if !ok {
        respondError(w, http.StatusUnauthorized, "Unauthorized")
        return
    }

    username, _ := auth.GetUsername(r.Context())

    respondJSON(w, http.StatusOK, map[string]interface{}{
        "id":       userID,
        "username": username,
    })
}

// 輔助函數
func respondJSON(w http.ResponseWriter, status int, data interface{}) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(status)
    json.NewEncoder(w).Encode(data)
}

func respondError(w http.ResponseWriter, status int, message string) {
    respondJSON(w, status, map[string]string{
        "error": message,
    })
}
```

### 5. 配置管理

#### config/config.go
```go
package config

import (
    "os"
    "time"
)

type Config struct {
    Server   ServerConfig
    Database DatabaseConfig
    JWT      JWTConfig
}

type ServerConfig struct {
    Port string
}

type DatabaseConfig struct {
    URL string
}

type JWTConfig struct {
    SecretKey     string
    TokenDuration time.Duration
}

func Load() *Config {
    return &Config{
        Server: ServerConfig{
            Port: getEnv("PORT", "8080"),
        },
        Database: DatabaseConfig{
            URL: getEnv("DATABASE_URL", "postgres://localhost/authdb?sslmode=disable"),
        },
        JWT: JWTConfig{
            SecretKey:     getEnv("JWT_SECRET", "your-secret-key-change-in-production"),
            TokenDuration: 24 * time.Hour,
        },
    }
}

func getEnv(key, defaultValue string) string {
    if value := os.Getenv(key); value != "" {
        return value
    }
    return defaultValue
}
```

### 6. 主程序

#### cmd/api/main.go
```go
package main

import (
    "database/sql"
    "log"
    "net/http"

    "jwt-auth-api/config"
    "jwt-auth-api/internal/auth"
    "jwt-auth-api/internal/handlers"
    "jwt-auth-api/internal/repository"

    "github.com/gorilla/mux"
    _ "github.com/lib/pq"
)

func main() {
    // 加載配置
    cfg := config.Load()

    // 連接數據庫
    db, err := sql.Open("postgres", cfg.Database.URL)
    if err != nil {
        log.Fatal(err)
    }
    defer db.Close()

    // 初始化依賴
    jwtManager := auth.NewJWTManager(cfg.JWT.SecretKey, cfg.JWT.TokenDuration)
    userRepo := repository.NewUserRepository(db)
    authService := auth.NewService(userRepo, jwtManager)
    authHandler := handlers.NewAuthHandler(authService)

    // 創建中間件
    authMiddleware := auth.NewMiddleware(jwtManager)

    // 設置路由
    r := mux.NewRouter()

    // 公開路由
    r.HandleFunc("/api/auth/register", authHandler.Register).Methods("POST")
    r.HandleFunc("/api/auth/login", authHandler.Login).Methods("POST")
    r.HandleFunc("/api/auth/refresh", authHandler.RefreshToken).Methods("POST")

    // 受保護路由
    protected := r.PathPrefix("/api").Subrouter()
    protected.Use(authMiddleware.Authenticate)
    protected.HandleFunc("/auth/me", authHandler.Me).Methods("GET")

    // 健康檢查
    r.HandleFunc("/health", func(w http.ResponseWriter, r *http.Request) {
        w.WriteHeader(http.StatusOK)
        w.Write([]byte("OK"))
    }).Methods("GET")

    // 啟動服務器
    addr := ":" + cfg.Server.Port
    log.Printf("Server starting on %s", addr)
    if err := http.ListenAndServe(addr, r); err != nil {
        log.Fatal(err)
    }
}
```

## 高級功能

### 1. Refresh Token（雙 Token 機制）

```go
type TokenPair struct {
    AccessToken  string `json:"access_token"`
    RefreshToken string `json:"refresh_token"`
}

func (m *JWTManager) GenerateTokenPair(userID int, username, email string) (*TokenPair, error) {
    // Access Token（短期，15 分鐘）
    accessToken, err := m.generateToken(userID, username, email, 15*time.Minute, "access")
    if err != nil {
        return nil, err
    }

    // Refresh Token（長期，7 天）
    refreshToken, err := m.generateToken(userID, username, email, 7*24*time.Hour, "refresh")
    if err != nil {
        return nil, err
    }

    return &TokenPair{
        AccessToken:  accessToken,
        RefreshToken: refreshToken,
    }, nil
}

func (m *JWTManager) generateToken(userID int, username, email string, duration time.Duration, tokenType string) (string, error) {
    claims := &Claims{
        UserID:    userID,
        Username:  username,
        Email:     email,
        TokenType: tokenType,
        RegisteredClaims: jwt.RegisteredClaims{
            ExpiresAt: jwt.NewNumericDate(time.Now().Add(duration)),
            IssuedAt:  jwt.NewNumericDate(time.Now()),
        },
    }

    token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
    return token.SignedString(m.secretKey)
}
```

### 2. Token 黑名單（登出功能）

```go
package auth

import (
    "sync"
    "time"
)

// TokenBlacklist Token 黑名單
type TokenBlacklist struct {
    tokens map[string]time.Time
    mu     sync.RWMutex
}

func NewTokenBlacklist() *TokenBlacklist {
    bl := &TokenBlacklist{
        tokens: make(map[string]time.Time),
    }

    // 定期清理過期的 token
    go bl.cleanup()

    return bl
}

// Add 添加到黑名單
func (bl *TokenBlacklist) Add(token string, expiry time.Time) {
    bl.mu.Lock()
    defer bl.mu.Unlock()
    bl.tokens[token] = expiry
}

// IsBlacklisted 檢查是否在黑名單
func (bl *TokenBlacklist) IsBlacklisted(token string) bool {
    bl.mu.RLock()
    defer bl.mu.RUnlock()

    expiry, exists := bl.tokens[token]
    if !exists {
        return false
    }

    // 如果已過期，從黑名單移除
    if time.Now().After(expiry) {
        delete(bl.tokens, token)
        return false
    }

    return true
}

// cleanup 定期清理過期 token
func (bl *TokenBlacklist) cleanup() {
    ticker := time.NewTicker(1 * time.Hour)
    defer ticker.Stop()

    for range ticker.C {
        bl.mu.Lock()
        now := time.Now()
        for token, expiry := range bl.tokens {
            if now.After(expiry) {
                delete(bl.tokens, token)
            }
        }
        bl.mu.Unlock()
    }
}
```

### 3. 權限控制

```go
type Role string

const (
    RoleUser  Role = "user"
    RoleAdmin Role = "admin"
)

type Claims struct {
    UserID   int    `json:"user_id"`
    Username string `json:"username"`
    Email    string `json:"email"`
    Role     Role   `json:"role"`
    jwt.RegisteredClaims
}

// RequireRole 要求特定角色的中間件
func (m *Middleware) RequireRole(role Role) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            // 先進行認證
            authHeader := r.Header.Get("Authorization")
            if authHeader == "" {
                http.Error(w, "Unauthorized", http.StatusUnauthorized)
                return
            }

            parts := strings.SplitN(authHeader, " ", 2)
            if len(parts) != 2 {
                http.Error(w, "Invalid token", http.StatusUnauthorized)
                return
            }

            claims, err := m.jwtManager.VerifyToken(parts[1])
            if err != nil {
                http.Error(w, "Invalid token", http.StatusUnauthorized)
                return
            }

            // 檢查角色
            if claims.Role != role && claims.Role != RoleAdmin {
                http.Error(w, "Forbidden", http.StatusForbidden)
                return
            }

            ctx := context.WithValue(r.Context(), ContextKeyUserID, claims.UserID)
            next.ServeHTTP(w, r.WithContext(ctx))
        })
    }
}
```

## Node.js 對應版本

### 使用 jsonwebtoken

```javascript
const jwt = require('jsonwebtoken');
const bcrypt = require('bcryptjs');

// 生成 token
function generateToken(user) {
    return jwt.sign(
        {
            userId: user.id,
            username: user.username,
            email: user.email
        },
        process.env.JWT_SECRET,
        { expiresIn: '24h' }
    );
}

// 驗證 token
function verifyToken(token) {
    return jwt.verify(token, process.env.JWT_SECRET);
}

// 認證中間件
function authMiddleware(req, res, next) {
    const authHeader = req.headers.authorization;

    if (!authHeader) {
        return res.status(401).json({ error: 'Missing authorization header' });
    }

    const [bearer, token] = authHeader.split(' ');

    if (bearer !== 'Bearer' || !token) {
        return res.status(401).json({ error: 'Invalid authorization format' });
    }

    try {
        const decoded = verifyToken(token);
        req.user = decoded;
        next();
    } catch (error) {
        return res.status(401).json({ error: 'Invalid token' });
    }
}

// 註冊
app.post('/api/auth/register', async (req, res) => {
    const { email, username, password } = req.body;

    // 檢查用戶是否存在
    const existingUser = await User.findOne({ email });
    if (existingUser) {
        return res.status(400).json({ error: 'Email already exists' });
    }

    // 加密密碼
    const hashedPassword = await bcrypt.hash(password, 10);

    // 創建用戶
    const user = await User.create({
        email,
        username,
        password: hashedPassword
    });

    // 生成 token
    const token = generateToken(user);

    res.status(201).json({
        token,
        user: {
            id: user.id,
            email: user.email,
            username: user.username
        }
    });
});

// 登錄
app.post('/api/auth/login', async (req, res) => {
    const { email, password } = req.body;

    // 查找用戶
    const user = await User.findOne({ email });
    if (!user) {
        return res.status(401).json({ error: 'Invalid credentials' });
    }

    // 驗證密碼
    const isValid = await bcrypt.compare(password, user.password);
    if (!isValid) {
        return res.status(401).json({ error: 'Invalid credentials' });
    }

    // 生成 token
    const token = generateToken(user);

    res.json({
        token,
        user: {
            id: user.id,
            email: user.email,
            username: user.username
        }
    });
});

// 受保護路由
app.get('/api/auth/me', authMiddleware, (req, res) => {
    res.json(req.user);
});
```

### 使用 Passport.js

```javascript
const passport = require('passport');
const JwtStrategy = require('passport-jwt').Strategy;
const ExtractJwt = require('passport-jwt').ExtractJwt;

// 配置 JWT 策略
const options = {
    jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
    secretOrKey: process.env.JWT_SECRET
};

passport.use(new JwtStrategy(options, async (payload, done) => {
    try {
        const user = await User.findById(payload.userId);
        if (user) {
            return done(null, user);
        }
        return done(null, false);
    } catch (error) {
        return done(error, false);
    }
}));

// 使用中間件
app.get('/api/auth/me',
    passport.authenticate('jwt', { session: false }),
    (req, res) => {
        res.json(req.user);
    }
);
```

## API 測試示例

### 使用 curl

```bash
# 註冊
curl -X POST http://localhost:8080/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{
    "email": "alice@example.com",
    "username": "alice",
    "password": "secret123"
  }'

# 登錄
curl -X POST http://localhost:8080/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{
    "email": "alice@example.com",
    "password": "secret123"
  }'

# 訪問受保護路由
curl http://localhost:8080/api/auth/me \
  -H "Authorization: Bearer YOUR_TOKEN_HERE"

# 刷新 token
curl -X POST http://localhost:8080/api/auth/refresh \
  -H "Content-Type: application/json" \
  -d '{"token": "YOUR_TOKEN_HERE"}'
```

## 重點總結

### JWT 核心概念

1. **無狀態**
   - 服務器不需要存儲 session
   - 適合分布式系統

2. **自包含**
   - Token 包含所有必要信息
   - 減少數據庫查詢

3. **安全性**
   - 使用強密鑰
   - HTTPS 傳輸
   - 設置合理的過期時間

### Go vs Node.js JWT 對比

| 特性 | Go (golang-jwt) | Node.js (jsonwebtoken) |
|------|-----------------|------------------------|
| 性能 | 高 | 中 |
| 類型安全 | 強類型 | 弱類型 |
| 生態系統 | 簡潔 | 豐富（Passport.js） |
| 中間件 | 手動實現 | 現成方案多 |

### 最佳實踐

1. **Token 安全**
   - 使用環境變量存儲密鑰
   - 密鑰長度至少 256 位
   - 定期輪換密鑰

2. **過期時間**
   - Access Token: 15 分鐘 - 1 小時
   - Refresh Token: 7 天 - 30 天

3. **存儲**
   - 不要在 LocalStorage 存儲敏感 token
   - 使用 HttpOnly Cookie（Web）
   - 使用安全存儲（Mobile）

4. **刷新策略**
   - 使用 Refresh Token
   - 實現靜默刷新

## 練習題

### 練習 1: 實現郵箱驗證

註冊後發送驗證郵件，用戶點擊鏈接後才能激活賬號。

### 練習 2: 實現 OAuth2

集成 Google 或 GitHub OAuth2 登錄。

### 練習 3: 多設備管理

實現設備管理功能，用戶可以查看和撤銷登錄設備。

### 練習 4: 密碼重置

實現忘記密碼功能，通過郵件發送重置鏈接。

### 練習 5: 實現 RBAC

基於角色的訪問控制，支持多種角色和權限。

```go
type Permission string

const (
    PermissionRead   Permission = "read"
    PermissionWrite  Permission = "write"
    PermissionDelete Permission = "delete"
)

type Role struct {
    Name        string
    Permissions []Permission
}
```

---

下一章：[Chapter 56: 編譯與交叉編譯](56-compilation.md)
