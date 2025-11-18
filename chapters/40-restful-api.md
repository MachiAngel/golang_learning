# Chapter 40: RESTful API 實戰

## 概述

本章將通過一個完整的示例項目，展示如何使用 Go 構建生產級的 RESTful API。我們將使用 Gin 框架，並展示如何與 Node.js/Express 的實現進行對比。

## 項目需求

構建一個博客 API，包括：
- 用戶管理（註冊、登錄、認證）
- 文章管理（CRUD）
- 評論功能
- 分頁和篩選
- 錯誤處理
- 日誌記錄
- API 文檔

## 項目結構

```
blog-api/
├── main.go
├── config/
│   └── config.go
├── models/
│   ├── user.go
│   ├── post.go
│   └── comment.go
├── controllers/
│   ├── auth.go
│   ├── post.go
│   └── comment.go
├── middleware/
│   ├── auth.go
│   ├── logger.go
│   └── error.go
├── routes/
│   └── routes.go
├── database/
│   └── db.go
├── utils/
│   ├── jwt.go
│   └── response.go
└── docs/
    └── api.md
```

## 完整實現

### 1. 主文件 (main.go)

**Go (Gin):**
```go
package main

import (
    "log"

    "github.com/gin-gonic/gin"
    "blog-api/config"
    "blog-api/database"
    "blog-api/routes"
)

func main() {
    // 加載配置
    cfg := config.Load()

    // 連接數據庫
    database.Connect(cfg.DatabaseURL)

    // 創建路由器
    r := gin.Default()

    // 設置路由
    routes.Setup(r)

    // 啟動服務器
    log.Printf("Server starting on %s", cfg.Port)
    if err := r.Run(cfg.Port); err != nil {
        log.Fatal("Failed to start server:", err)
    }
}
```

**Node.js (Express):**
```javascript
const express = require('express');
const config = require('./config');
const database = require('./database');
const routes = require('./routes');

const app = express();

// 中間件
app.use(express.json());

// 連接數據庫
database.connect(config.databaseURL);

// 設置路由
routes.setup(app);

// 啟動服務器
app.listen(config.port, () => {
    console.log(`Server starting on ${config.port}`);
});
```

### 2. 數據模型 (models/user.go)

**Go:**
```go
package models

import (
    "time"
    "golang.org/x/crypto/bcrypt"
)

type User struct {
    ID        uint      `json:"id" gorm:"primaryKey"`
    Username  string    `json:"username" gorm:"unique;not null"`
    Email     string    `json:"email" gorm:"unique;not null"`
    Password  string    `json:"-" gorm:"not null"`
    CreatedAt time.Time `json:"created_at"`
    UpdatedAt time.Time `json:"updated_at"`
    Posts     []Post    `json:"posts,omitempty" gorm:"foreignKey:UserID"`
}

type RegisterInput struct {
    Username string `json:"username" binding:"required,min=3,max=20"`
    Email    string `json:"email" binding:"required,email"`
    Password string `json:"password" binding:"required,min=6"`
}

type LoginInput struct {
    Email    string `json:"email" binding:"required,email"`
    Password string `json:"password" binding:"required"`
}

func (u *User) HashPassword(password string) error {
    hashedPassword, err := bcrypt.GenerateFromPassword([]byte(password), bcrypt.DefaultCost)
    if err != nil {
        return err
    }
    u.Password = string(hashedPassword)
    return nil
}

func (u *User) CheckPassword(password string) bool {
    err := bcrypt.CompareHashAndPassword([]byte(u.Password), []byte(password))
    return err == nil
}
```

**Node.js:**
```javascript
const bcrypt = require('bcrypt');

class User {
    constructor(data) {
        this.id = data.id;
        this.username = data.username;
        this.email = data.email;
        this.password = data.password;
        this.createdAt = data.createdAt;
        this.updatedAt = data.updatedAt;
    }

    async hashPassword(password) {
        this.password = await bcrypt.hash(password, 10);
    }

    async checkPassword(password) {
        return await bcrypt.compare(password, this.password);
    }

    toJSON() {
        const obj = { ...this };
        delete obj.password;
        return obj;
    }
}

module.exports = User;
```

### 3. 認證控制器 (controllers/auth.go)

**Go:**
```go
package controllers

import (
    "net/http"

    "github.com/gin-gonic/gin"
    "blog-api/database"
    "blog-api/models"
    "blog-api/utils"
)

func Register(c *gin.Context) {
    var input models.RegisterInput

    if err := c.ShouldBindJSON(&input); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }

    user := models.User{
        Username: input.Username,
        Email:    input.Email,
    }

    if err := user.HashPassword(input.Password); err != nil {
        c.JSON(http.StatusInternalServerError, gin.H{"error": "Failed to hash password"})
        return
    }

    if err := database.DB.Create(&user).Error; err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": "Username or email already exists"})
        return
    }

    token, err := utils.GenerateToken(user.ID)
    if err != nil {
        c.JSON(http.StatusInternalServerError, gin.H{"error": "Failed to generate token"})
        return
    }

    c.JSON(http.StatusCreated, gin.H{
        "user":  user,
        "token": token,
    })
}

func Login(c *gin.Context) {
    var input models.LoginInput

    if err := c.ShouldBindJSON(&input); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }

    var user models.User
    if err := database.DB.Where("email = ?", input.Email).First(&user).Error; err != nil {
        c.JSON(http.StatusUnauthorized, gin.H{"error": "Invalid credentials"})
        return
    }

    if !user.CheckPassword(input.Password) {
        c.JSON(http.StatusUnauthorized, gin.H{"error": "Invalid credentials"})
        return
    }

    token, err := utils.GenerateToken(user.ID)
    if err != nil {
        c.JSON(http.StatusInternalServerError, gin.H{"error": "Failed to generate token"})
        return
    }

    c.JSON(http.StatusOK, gin.H{
        "user":  user,
        "token": token,
    })
}

func GetProfile(c *gin.Context) {
    userID := c.GetUint("userID")

    var user models.User
    if err := database.DB.Preload("Posts").First(&user, userID).Error; err != nil {
        c.JSON(http.StatusNotFound, gin.H{"error": "User not found"})
        return
    }

    c.JSON(http.StatusOK, user)
}
```

**Node.js:**
```javascript
const User = require('../models/user');
const database = require('../database');
const jwt = require('../utils/jwt');

exports.register = async (req, res) => {
    try {
        const { username, email, password } = req.body;

        const user = new User({ username, email });
        await user.hashPassword(password);

        const savedUser = await database.users.create(user);

        const token = jwt.generateToken(savedUser.id);

        res.status(201).json({
            user: savedUser,
            token
        });
    } catch (error) {
        res.status(400).json({ error: error.message });
    }
};

exports.login = async (req, res) => {
    try {
        const { email, password } = req.body;

        const user = await database.users.findByEmail(email);
        if (!user || !(await user.checkPassword(password))) {
            return res.status(401).json({ error: 'Invalid credentials' });
        }

        const token = jwt.generateToken(user.id);

        res.json({
            user,
            token
        });
    } catch (error) {
        res.status(400).json({ error: error.message });
    }
};

exports.getProfile = async (req, res) => {
    try {
        const user = await database.users.findById(req.userId, {
            include: ['posts']
        });

        res.json(user);
    } catch (error) {
        res.status(404).json({ error: 'User not found' });
    }
};
```

### 4. 文章控制器 (controllers/post.go)

**Go:**
```go
package controllers

import (
    "net/http"
    "strconv"

    "github.com/gin-gonic/gin"
    "blog-api/database"
    "blog-api/models"
)

func GetPosts(c *gin.Context) {
    page, _ := strconv.Atoi(c.DefaultQuery("page", "1"))
    limit, _ := strconv.Atoi(c.DefaultQuery("limit", "10"))
    offset := (page - 1) * limit

    var posts []models.Post
    var total int64

    query := database.DB.Model(&models.Post{}).Preload("User")

    // 篩選條件
    if userID := c.Query("user_id"); userID != "" {
        query = query.Where("user_id = ?", userID)
    }

    query.Count(&total)
    query.Limit(limit).Offset(offset).Order("created_at DESC").Find(&posts)

    c.JSON(http.StatusOK, gin.H{
        "posts": posts,
        "pagination": gin.H{
            "page":  page,
            "limit": limit,
            "total": total,
        },
    })
}

func GetPost(c *gin.Context) {
    id := c.Param("id")

    var post models.Post
    if err := database.DB.Preload("User").Preload("Comments.User").First(&post, id).Error; err != nil {
        c.JSON(http.StatusNotFound, gin.H{"error": "Post not found"})
        return
    }

    c.JSON(http.StatusOK, post)
}

func CreatePost(c *gin.Context) {
    var input struct {
        Title   string `json:"title" binding:"required"`
        Content string `json:"content" binding:"required"`
    }

    if err := c.ShouldBindJSON(&input); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }

    post := models.Post{
        Title:   input.Title,
        Content: input.Content,
        UserID:  c.GetUint("userID"),
    }

    if err := database.DB.Create(&post).Error; err != nil {
        c.JSON(http.StatusInternalServerError, gin.H{"error": "Failed to create post"})
        return
    }

    database.DB.Preload("User").First(&post, post.ID)

    c.JSON(http.StatusCreated, post)
}

func UpdatePost(c *gin.Context) {
    id := c.Param("id")
    userID := c.GetUint("userID")

    var post models.Post
    if err := database.DB.First(&post, id).Error; err != nil {
        c.JSON(http.StatusNotFound, gin.H{"error": "Post not found"})
        return
    }

    if post.UserID != userID {
        c.JSON(http.StatusForbidden, gin.H{"error": "You can only update your own posts"})
        return
    }

    var input struct {
        Title   string `json:"title"`
        Content string `json:"content"`
    }

    if err := c.ShouldBindJSON(&input); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }

    database.DB.Model(&post).Updates(models.Post{
        Title:   input.Title,
        Content: input.Content,
    })

    c.JSON(http.StatusOK, post)
}

func DeletePost(c *gin.Context) {
    id := c.Param("id")
    userID := c.GetUint("userID")

    var post models.Post
    if err := database.DB.First(&post, id).Error; err != nil {
        c.JSON(http.StatusNotFound, gin.H{"error": "Post not found"})
        return
    }

    if post.UserID != userID {
        c.JSON(http.StatusForbidden, gin.H{"error": "You can only delete your own posts"})
        return
    }

    database.DB.Delete(&post)

    c.Status(http.StatusNoContent)
}
```

### 5. 認證中間件 (middleware/auth.go)

**Go:**
```go
package middleware

import (
    "net/http"
    "strings"

    "github.com/gin-gonic/gin"
    "blog-api/utils"
)

func AuthRequired() gin.HandlerFunc {
    return func(c *gin.Context) {
        authHeader := c.GetHeader("Authorization")

        if authHeader == "" {
            c.JSON(http.StatusUnauthorized, gin.H{"error": "Authorization header required"})
            c.Abort()
            return
        }

        parts := strings.Split(authHeader, " ")
        if len(parts) != 2 || parts[0] != "Bearer" {
            c.JSON(http.StatusUnauthorized, gin.H{"error": "Invalid authorization header format"})
            c.Abort()
            return
        }

        token := parts[1]
        userID, err := utils.ValidateToken(token)
        if err != nil {
            c.JSON(http.StatusUnauthorized, gin.H{"error": "Invalid token"})
            c.Abort()
            return
        }

        c.Set("userID", userID)
        c.Next()
    }
}
```

**Node.js:**
```javascript
const jwt = require('../utils/jwt');

exports.authRequired = (req, res, next) => {
    const authHeader = req.headers.authorization;

    if (!authHeader) {
        return res.status(401).json({ error: 'Authorization header required' });
    }

    const parts = authHeader.split(' ');
    if (parts.length !== 2 || parts[0] !== 'Bearer') {
        return res.status(401).json({ error: 'Invalid authorization header format' });
    }

    const token = parts[1];
    try {
        const userId = jwt.validateToken(token);
        req.userId = userId;
        next();
    } catch (error) {
        res.status(401).json({ error: 'Invalid token' });
    }
};
```

### 6. 路由設置 (routes/routes.go)

**Go:**
```go
package routes

import (
    "github.com/gin-gonic/gin"
    "blog-api/controllers"
    "blog-api/middleware"
)

func Setup(r *gin.Engine) {
    // 公開路由
    api := r.Group("/api")
    {
        // 認證
        auth := api.Group("/auth")
        {
            auth.POST("/register", controllers.Register)
            auth.POST("/login", controllers.Login)
        }

        // 公開的文章列表
        api.GET("/posts", controllers.GetPosts)
        api.GET("/posts/:id", controllers.GetPost)
    }

    // 需要認證的路由
    protected := api.Group("")
    protected.Use(middleware.AuthRequired())
    {
        // 用戶
        protected.GET("/profile", controllers.GetProfile)

        // 文章
        protected.POST("/posts", controllers.CreatePost)
        protected.PUT("/posts/:id", controllers.UpdatePost)
        protected.DELETE("/posts/:id", controllers.DeletePost)

        // 評論
        protected.POST("/posts/:id/comments", controllers.CreateComment)
        protected.DELETE("/comments/:id", controllers.DeleteComment)
    }
}
```

### 7. JWT 工具 (utils/jwt.go)

**Go:**
```go
package utils

import (
    "errors"
    "time"

    "github.com/golang-jwt/jwt/v5"
)

var jwtSecret = []byte("your-secret-key")

type Claims struct {
    UserID uint `json:"user_id"`
    jwt.RegisteredClaims
}

func GenerateToken(userID uint) (string, error) {
    claims := Claims{
        UserID: userID,
        RegisteredClaims: jwt.RegisteredClaims{
            ExpiresAt: jwt.NewNumericDate(time.Now().Add(24 * time.Hour)),
            IssuedAt:  jwt.NewNumericDate(time.Now()),
        },
    }

    token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
    return token.SignedString(jwtSecret)
}

func ValidateToken(tokenString string) (uint, error) {
    token, err := jwt.ParseWithClaims(tokenString, &Claims{}, func(token *jwt.Token) (interface{}, error) {
        return jwtSecret, nil
    })

    if err != nil {
        return 0, err
    }

    if claims, ok := token.Claims.(*Claims); ok && token.Valid {
        return claims.UserID, nil
    }

    return 0, errors.New("invalid token")
}
```

## API 端點總結

| 方法 | 端點 | 描述 | 認證 |
|------|------|------|------|
| POST | /api/auth/register | 註冊 | ❌ |
| POST | /api/auth/login | 登錄 | ❌ |
| GET | /api/profile | 獲取個人資料 | ✅ |
| GET | /api/posts | 獲取文章列表 | ❌ |
| GET | /api/posts/:id | 獲取文章詳情 | ❌ |
| POST | /api/posts | 創建文章 | ✅ |
| PUT | /api/posts/:id | 更新文章 | ✅ |
| DELETE | /api/posts/:id | 刪除文章 | ✅ |
| POST | /api/posts/:id/comments | 創建評論 | ✅ |
| DELETE | /api/comments/:id | 刪除評論 | ✅ |

## 測試 API

### 使用 curl

```bash
# 註冊
curl -X POST http://localhost:8080/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{"username":"alice","email":"alice@example.com","password":"password123"}'

# 登錄
curl -X POST http://localhost:8080/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"alice@example.com","password":"password123"}'

# 創建文章
curl -X POST http://localhost:8080/api/posts \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -d '{"title":"My First Post","content":"This is my first blog post!"}'

# 獲取文章列表
curl http://localhost:8080/api/posts?page=1&limit=10
```

## 重點總結

1. **RESTful 設計原則**
   - 使用 HTTP 方法表示操作
   - 使用 HTTP 狀態碼
   - 資源為名詞，操作為動詞
   - 無狀態

2. **項目結構**
   - 清晰的目錄組織
   - 分層架構
   - 關注點分離

3. **安全性**
   - 密碼哈希
   - JWT 認證
   - 輸入驗證
   - 錯誤處理

4. **最佳實踐**
   - 參數綁定和驗證
   - 統一的響應格式
   - 中間件使用
   - 錯誤處理

## 練習題

1. 添加文章標籤功能
2. 實現文章搜索功能
3. 添加文章點贊功能
4. 實現用戶關注功能
5. 添加圖片上傳功能
6. 實現評論回覆功能
7. 添加文章分類功能
8. 實現 API 限流
