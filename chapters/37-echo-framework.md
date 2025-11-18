# Chapter 37: Echo 框架

## 概述

Echo 是另一個流行的 Go Web 框架，以其高性能、可擴展性和簡潔的 API 著稱。Echo 的設計哲學是提供最小化但完整的功能集。

## 安裝

```bash
go get -u github.com/labstack/echo/v4
```

## 快速對比

| 功能 | Express | Gin | Echo |
|------|---------|-----|------|
| 性能 | 中等 | 高 | 高 |
| 中間件 | 豐富 | 豐富 | 豐富 |
| 文檔 | 優秀 | 良好 | 優秀 |
| 社區 | 最大 | 大 | 中等 |

## 實際代碼示例

### 示例 1: Hello World

```go
package main

import (
    "net/http"

    "github.com/labstack/echo/v4"
    "github.com/labstack/echo/v4/middleware"
)

func main() {
    e := echo.New()

    // 中間件
    e.Use(middleware.Logger())
    e.Use(middleware.Recover())

    // 路由
    e.GET("/", func(c echo.Context) error {
        return c.String(http.StatusOK, "Hello, World!")
    })

    e.GET("/json", func(c echo.Context) error {
        return c.JSON(http.StatusOK, map[string]string{
            "message": "Hello, JSON!",
        })
    })

    e.Logger.Fatal(e.Start(":8080"))
}
```

### 示例 2: 路由參數

```go
// 路徑參數
e.GET("/users/:id", func(c echo.Context) error {
    id := c.Param("id")
    return c.JSON(http.StatusOK, map[string]string{
        "user_id": id,
    })
})

// 查詢參數
e.GET("/search", func(c echo.Context) error {
    query := c.QueryParam("q")
    page := c.QueryParam("page")

    return c.JSON(http.StatusOK, map[string]string{
        "query": query,
        "page":  page,
    })
})
```

### 示例 3: 請求綁定

```go
type User struct {
    Name  string `json:"name" validate:"required"`
    Email string `json:"email" validate:"required,email"`
}

e.POST("/users", func(c echo.Context) error {
    user := new(User)

    if err := c.Bind(user); err != nil {
        return err
    }

    if err := c.Validate(user); err != nil {
        return err
    }

    return c.JSON(http.StatusCreated, user)
})
```

### 示例 4: 中間件

```go
// 自定義中間件
func CustomMiddleware(next echo.HandlerFunc) echo.HandlerFunc {
    return func(c echo.Context) error {
        // 前置處理
        log.Println("Before request")

        // 調用下一個處理器
        err := next(c)

        // 後置處理
        log.Println("After request")

        return err
    }
}

// 使用中間件
e.Use(CustomMiddleware)

// 路由組中間件
g := e.Group("/admin")
g.Use(AuthMiddleware)
```

### 示例 5: 路由組

```go
// API v1
v1 := e.Group("/api/v1")
v1.GET("/users", getUsersV1)
v1.POST("/users", createUserV1)

// API v2
v2 := e.Group("/api/v2")
v2.GET("/users", getUsersV2)
v2.POST("/users", createUserV2)

// Admin 組（需要認證）
admin := e.Group("/admin")
admin.Use(middleware.BasicAuth(func(username, password string, c echo.Context) (bool, error) {
    return username == "admin" && password == "secret", nil
}))
admin.GET("/dashboard", adminDashboard)
```

### 示例 6: 文件上傳

```go
e.POST("/upload", func(c echo.Context) error {
    file, err := c.FormFile("file")
    if err != nil {
        return err
    }

    src, err := file.Open()
    if err != nil {
        return err
    }
    defer src.Close()

    dst, err := os.Create(filepath.Join("uploads", file.Filename))
    if err != nil {
        return err
    }
    defer dst.Close()

    if _, err = io.Copy(dst, src); err != nil {
        return err
    }

    return c.JSON(http.StatusOK, map[string]interface{}{
        "filename": file.Filename,
        "size":     file.Size,
    })
})
```

## Echo vs Gin

| 特性 | Gin | Echo |
|------|-----|------|
| 性能 | 稍快 | 很快 |
| API 設計 | 更接近 Express | 更符合 Go 慣例 |
| 錯誤處理 | 手動 | 統一錯誤處理 |
| 中間件 | 豐富 | 更豐富 |
| 學習曲線 | 平緩 | 稍陡 |

## 重點總結

1. Echo 提供簡潔而強大的 API
2. 內置豐富的中間件
3. 統一的錯誤處理機制
4. 優秀的性能表現
5. 完善的文檔和示例

## 練習題

1. 創建完整的 RESTful API
2. 實現 JWT 認證系統
3. 添加請求驗證和錯誤處理
4. 實現文件上傳和靜態文件服務
5. 創建 WebSocket 實時通訊
6. 實現 API 版本控制
7. 添加請求限流和緩存
8. 構建完整的 CRUD 應用
