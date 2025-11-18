# Chapter 38: Fiber 框架

## 概述

Fiber 是受 Express.js 啟發的 Go Web 框架，專為速度和零內存分配而設計。它建立在 fasthttp 之上，是 Go 中最快的 Web 框架之一。對於 Node.js 開發者來說，Fiber 的 API 幾乎與 Express 一模一樣。

## 安裝

```bash
go get -u github.com/gofiber/fiber/v2
```

## 為什麼選擇 Fiber？

- Express 風格的 API
- 極致的性能（基於 fasthttp）
- 零內存分配
- 豐富的中間件
- 易於學習（對 Express 開發者來說）

## 實際代碼示例

### 示例 1: Hello World

**Go (Fiber):**
```go
package main

import (
    "github.com/gofiber/fiber/v2"
    "log"
)

func main() {
    app := fiber.New()

    app.Get("/", func(c *fiber.Ctx) error {
        return c.SendString("Hello, World!")
    })

    app.Get("/json", func(c *fiber.Ctx) error {
        return c.JSON(fiber.Map{
            "message": "Hello, JSON!",
            "status":  "success",
        })
    })

    log.Fatal(app.Listen(":8080"))
}
```

**Node.js (Express):**
```javascript
const express = require('express');
const app = express();

app.get('/', (req, res) => {
    res.send('Hello, World!');
});

app.get('/json', (req, res) => {
    res.json({
        message: 'Hello, JSON!',
        status: 'success'
    });
});

app.listen(8080);
```

### 示例 2: 路由參數

```go
// 路徑參數（與 Express 完全相同）
app.Get("/users/:id", func(c *fiber.Ctx) error {
    id := c.Params("id")
    return c.JSON(fiber.Map{
        "user_id": id,
    })
})

// 可選參數
app.Get("/users/:id?", func(c *fiber.Ctx) error {
    id := c.Params("id", "default")
    return c.SendString("User ID: " + id)
})

// 查詢參數
app.Get("/search", func(c *fiber.Ctx) error {
    query := c.Query("q")
    page := c.Query("page", "1") // 帶默認值

    return c.JSON(fiber.Map{
        "query": query,
        "page":  page,
    })
})
```

### 示例 3: 請求體解析

```go
type User struct {
    Name  string `json:"name"`
    Email string `json:"email"`
}

// JSON body
app.Post("/users", func(c *fiber.Ctx) error {
    user := new(User)

    if err := c.BodyParser(user); err != nil {
        return c.Status(400).JSON(fiber.Map{
            "error": err.Error(),
        })
    }

    return c.Status(201).JSON(user)
})

// Form body
app.Post("/login", func(c *fiber.Ctx) error {
    username := c.FormValue("username")
    password := c.FormValue("password")

    return c.JSON(fiber.Map{
        "username": username,
    })
})
```

### 示例 4: 中間件

```go
package main

import (
    "log"
    "time"

    "github.com/gofiber/fiber/v2"
    "github.com/gofiber/fiber/v2/middleware/logger"
    "github.com/gofiber/fiber/v2/middleware/recover"
    "github.com/gofiber/fiber/v2/middleware/cors"
)

func main() {
    app := fiber.New()

    // 內置中間件
    app.Use(logger.New())
    app.Use(recover.New())
    app.Use(cors.New())

    // 自定義中間件
    app.Use(func(c *fiber.Ctx) error {
        start := time.Now()
        err := c.Next()
        duration := time.Since(start)

        log.Printf("[%s] %s - %v", c.Method(), c.Path(), duration)
        return err
    })

    // 認證中間件
    authMiddleware := func(c *fiber.Ctx) error {
        token := c.Get("Authorization")
        if token != "Bearer valid-token" {
            return c.Status(401).JSON(fiber.Map{
                "error": "Unauthorized",
            })
        }
        return c.Next()
    }

    // 路由
    app.Get("/public", func(c *fiber.Ctx) error {
        return c.SendString("Public route")
    })

    // 受保護的路由
    api := app.Group("/api", authMiddleware)
    api.Get("/profile", func(c *fiber.Ctx) error {
        return c.JSON(fiber.Map{
            "message": "Protected route",
        })
    })

    log.Fatal(app.Listen(":8080"))
}
```

### 示例 5: 路由組

```go
func main() {
    app := fiber.New()

    // API v1
    v1 := app.Group("/api/v1")
    v1.Get("/users", getUsersV1)
    v1.Post("/users", createUserV1)
    v1.Get("/users/:id", getUserV1)

    // API v2
    v2 := app.Group("/api/v2")
    v2.Get("/users", getUsersV2)
    v2.Post("/users", createUserV2)

    // Admin group
    admin := app.Group("/admin")
    admin.Use(authMiddleware)
    admin.Get("/dashboard", adminDashboard)
    admin.Get("/users", adminUsers)

    app.Listen(":8080")
}
```

### 示例 6: 文件上傳

```go
app.Post("/upload", func(c *fiber.Ctx) error {
    // 單文件上傳
    file, err := c.FormFile("file")
    if err != nil {
        return err
    }

    // 保存文件
    err = c.SaveFile(file, "./uploads/"+file.Filename)
    if err != nil {
        return err
    }

    return c.JSON(fiber.Map{
        "filename": file.Filename,
        "size":     file.Size,
    })
})

// 多文件上傳
app.Post("/uploads", func(c *fiber.Ctx) error {
    form, err := c.MultipartForm()
    if err != nil {
        return err
    }

    files := form.File["files"]

    for _, file := range files {
        c.SaveFile(file, "./uploads/"+file.Filename)
    }

    return c.JSON(fiber.Map{
        "count": len(files),
    })
})
```

### 示例 7: 靜態文件服務

```go
app.Static("/", "./public")

// 自定義配置
app.Static("/static", "./public", fiber.Static{
    Compress:      true,
    ByteRange:     true,
    Browse:        true,
    Index:         "index.html",
    CacheDuration: 10 * time.Second,
})
```

### 示例 8: 完整的 CRUD API

```go
package main

import (
    "log"
    "strconv"

    "github.com/gofiber/fiber/v2"
)

type Product struct {
    ID    int     `json:"id"`
    Name  string  `json:"name"`
    Price float64 `json:"price"`
}

var products = []Product{
    {ID: 1, Name: "Laptop", Price: 999.99},
    {ID: 2, Name: "Mouse", Price: 29.99},
}
var nextID = 3

func main() {
    app := fiber.New(fiber.Config{
        ErrorHandler: customErrorHandler,
    })

    api := app.Group("/api/products")

    api.Get("/", getProducts)
    api.Get("/:id", getProduct)
    api.Post("/", createProduct)
    api.Put("/:id", updateProduct)
    api.Delete("/:id", deleteProduct)

    log.Fatal(app.Listen(":8080"))
}

func getProducts(c *fiber.Ctx) error {
    return c.JSON(products)
}

func getProduct(c *fiber.Ctx) error {
    id, _ := strconv.Atoi(c.Params("id"))

    for _, p := range products {
        if p.ID == id {
            return c.JSON(p)
        }
    }

    return c.Status(404).JSON(fiber.Map{
        "error": "Product not found",
    })
}

func createProduct(c *fiber.Ctx) error {
    product := new(Product)

    if err := c.BodyParser(product); err != nil {
        return c.Status(400).JSON(fiber.Map{
            "error": err.Error(),
        })
    }

    product.ID = nextID
    nextID++
    products = append(products, *product)

    return c.Status(201).JSON(product)
}

func updateProduct(c *fiber.Ctx) error {
    id, _ := strconv.Atoi(c.Params("id"))
    product := new(Product)

    if err := c.BodyParser(product); err != nil {
        return c.Status(400).JSON(fiber.Map{
            "error": err.Error(),
        })
    }

    for i, p := range products {
        if p.ID == id {
            product.ID = id
            products[i] = *product
            return c.JSON(product)
        }
    }

    return c.Status(404).JSON(fiber.Map{
        "error": "Product not found",
    })
}

func deleteProduct(c *fiber.Ctx) error {
    id, _ := strconv.Atoi(c.Params("id"))

    for i, p := range products {
        if p.ID == id {
            products = append(products[:i], products[i+1:]...)
            return c.SendStatus(204)
        }
    }

    return c.Status(404).JSON(fiber.Map{
        "error": "Product not found",
    })
}

func customErrorHandler(c *fiber.Ctx, err error) error {
    code := fiber.StatusInternalServerError

    if e, ok := err.(*fiber.Error); ok {
        code = e.Code
    }

    return c.Status(code).JSON(fiber.Map{
        "error": err.Error(),
    })
}
```

## Fiber vs Express API 對比

| 功能 | Express | Fiber |
|------|---------|-------|
| 創建應用 | `const app = express()` | `app := fiber.New()` |
| GET 路由 | `app.get('/path', handler)` | `app.Get("/path", handler)` |
| POST 路由 | `app.post('/path', handler)` | `app.Post("/path", handler)` |
| 路徑參數 | `req.params.id` | `c.Params("id")` |
| 查詢參數 | `req.query.page` | `c.Query("page")` |
| JSON 響應 | `res.json(data)` | `c.JSON(data)` |
| 狀態碼 | `res.status(200)` | `c.Status(200)` |
| 中間件 | `app.use(middleware)` | `app.Use(middleware)` |
| 路由組 | `Router()` | `app.Group("/path")` |

## 重點總結

1. **Fiber 的優勢**
   - Express 風格的 API
   - 極致性能（比標準庫快 10 倍以上）
   - 零內存分配
   - 易於學習

2. **適用場景**
   - 高性能 API
   - 微服務
   - 實時應用
   - Express 開發者遷移到 Go

3. **注意事項**
   - 基於 fasthttp，不是標準 net/http
   - 不能直接使用標準庫的某些功能
   - 需要使用 Fiber 特定的中間件

4. **與其他框架對比**
   - 比 Gin 更接近 Express
   - 性能優於 Gin 和 Echo
   - 社區相對較小

## 練習題

1. 創建完整的 RESTful API
2. 實現 JWT 認證
3. 添加文件上傳功能
4. 實現實時聊天室（WebSocket）
5. 創建 API 限流中間件
6. 實現緩存機制
7. 構建完整的博客系統
8. 添加 API 文檔（Swagger）
