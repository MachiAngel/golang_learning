# Chapter 36: Gin 框架

## 概述

Gin 是 Go 語言中最流行的 Web 框架之一，以其高性能和簡潔的 API 著稱。對於 Node.js 開發者來說，Gin 的使用體驗非常類似 Express.js，但提供了更好的性能和類型安全。

## Node.js 與 Go (Gin) 的對比

| 功能 | Express.js | Gin |
|------|-----------|-----|
| 創建應用 | `const app = express()` | `gin.Default()` |
| 路由定義 | `app.get('/path', handler)` | `router.GET("/path", handler)` |
| 路徑參數 | `/users/:id` | `/users/:id` |
| 查詢參數 | `req.query.key` | `c.Query("key")` |
| JSON 響應 | `res.json(data)` | `c.JSON(200, data)` |
| 中間件 | `app.use(middleware)` | `router.Use(middleware)` |
| 路由組 | `Router()` | `router.Group("/path")` |
| 參數綁定 | 手動解析 | `c.ShouldBind(&struct)` |

## 安裝

```bash
go get -u github.com/gin-gonic/gin
```

## 實際代碼示例

### 示例 1: Hello World

**Go (Gin):**
```go
package main

import (
    "github.com/gin-gonic/gin"
)

func main() {
    // 創建默認路由器（帶日誌和恢復中間件）
    r := gin.Default()

    // 定義路由
    r.GET("/", func(c *gin.Context) {
        c.String(200, "Hello, World!")
    })

    r.GET("/json", func(c *gin.Context) {
        c.JSON(200, gin.H{
            "message": "Hello, JSON!",
            "status":  "success",
        })
    })

    // 啟動服務器
    r.Run(":8080") // 相當於 http.ListenAndServe(":8080", r)
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

app.listen(8080, () => {
    console.log('Server running on :8080');
});
```

### 示例 2: 路徑參數和查詢參數

**Go (Gin):**
```go
package main

import (
    "github.com/gin-gonic/gin"
    "net/http"
)

func main() {
    r := gin.Default()

    // 路徑參數
    r.GET("/users/:id", func(c *gin.Context) {
        id := c.Param("id")
        c.JSON(http.StatusOK, gin.H{
            "user_id": id,
        })
    })

    // 通配符
    r.GET("/files/*filepath", func(c *gin.Context) {
        filepath := c.Param("filepath")
        c.JSON(http.StatusOK, gin.H{
            "filepath": filepath,
        })
    })

    // 查詢參數
    r.GET("/search", func(c *gin.Context) {
        query := c.Query("q")                   // 獲取單個參數
        page := c.DefaultQuery("page", "1")     // 帶默認值
        sort := c.QueryArray("sort")            // 獲取數組

        c.JSON(http.StatusOK, gin.H{
            "query": query,
            "page":  page,
            "sort":  sort,
        })
    })

    r.Run(":8080")
}
```

**Node.js (Express):**
```javascript
const express = require('express');
const app = express();

// 路徑參數
app.get('/users/:id', (req, res) => {
    res.json({
        user_id: req.params.id
    });
});

// 通配符
app.get('/files/*', (req, res) => {
    res.json({
        filepath: req.params[0]
    });
});

// 查詢參數
app.get('/search', (req, res) => {
    const query = req.query.q;
    const page = req.query.page || '1';
    const sort = Array.isArray(req.query.sort) ? req.query.sort : [req.query.sort];

    res.json({
        query,
        page,
        sort: sort.filter(Boolean)
    });
});

app.listen(8080);
```

### 示例 3: 請求體綁定和驗證

**Go (Gin):**
```go
package main

import (
    "github.com/gin-gonic/gin"
    "net/http"
)

type User struct {
    Name  string `json:"name" binding:"required"`
    Email string `json:"email" binding:"required,email"`
    Age   int    `json:"age" binding:"required,gte=18"`
}

type LoginForm struct {
    Username string `form:"username" binding:"required"`
    Password string `form:"password" binding:"required"`
}

func main() {
    r := gin.Default()

    // JSON 綁定
    r.POST("/users", func(c *gin.Context) {
        var user User

        // 綁定並驗證
        if err := c.ShouldBindJSON(&user); err != nil {
            c.JSON(http.StatusBadRequest, gin.H{
                "error": err.Error(),
            })
            return
        }

        c.JSON(http.StatusOK, gin.H{
            "message": "User created",
            "user":    user,
        })
    })

    // 表單綁定
    r.POST("/login", func(c *gin.Context) {
        var form LoginForm

        if err := c.ShouldBind(&form); err != nil {
            c.JSON(http.StatusBadRequest, gin.H{
                "error": err.Error(),
            })
            return
        }

        c.JSON(http.StatusOK, gin.H{
            "message":  "Login successful",
            "username": form.Username,
        })
    })

    // 查詢參數綁定
    r.GET("/filter", func(c *gin.Context) {
        var query struct {
            Page  int    `form:"page" binding:"required,gte=1"`
            Limit int    `form:"limit" binding:"required,gte=1,lte=100"`
            Sort  string `form:"sort" binding:"omitempty,oneof=asc desc"`
        }

        if err := c.ShouldBindQuery(&query); err != nil {
            c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
            return
        }

        c.JSON(http.StatusOK, query)
    })

    r.Run(":8080")
}
```

**Node.js (Express + express-validator):**
```javascript
const express = require('express');
const { body, query, validationResult } = require('express-validator');
const app = express();

app.use(express.json());
app.use(express.urlencoded({ extended: true }));

// JSON 驗證
app.post('/users',
    body('name').notEmpty(),
    body('email').isEmail(),
    body('age').isInt({ min: 18 }),
    (req, res) => {
        const errors = validationResult(req);
        if (!errors.isEmpty()) {
            return res.status(400).json({ errors: errors.array() });
        }

        res.json({
            message: 'User created',
            user: req.body
        });
    }
);

// 表單驗證
app.post('/login',
    body('username').notEmpty(),
    body('password').notEmpty(),
    (req, res) => {
        const errors = validationResult(req);
        if (!errors.isEmpty()) {
            return res.status(400).json({ errors: errors.array() });
        }

        res.json({
            message: 'Login successful',
            username: req.body.username
        });
    }
);

app.listen(8080);
```

### 示例 4: 中間件

**Go (Gin):**
```go
package main

import (
    "log"
    "time"

    "github.com/gin-gonic/gin"
)

// 自定義日誌中間件
func Logger() gin.HandlerFunc {
    return func(c *gin.Context) {
        start := time.Now()
        path := c.Request.URL.Path

        // 處理請求
        c.Next()

        // 請求完成後
        duration := time.Since(start)
        status := c.Writer.Status()

        log.Printf("[%s] %s %d %v", c.Request.Method, path, status, duration)
    }
}

// 認證中間件
func AuthRequired() gin.HandlerFunc {
    return func(c *gin.Context) {
        token := c.GetHeader("Authorization")

        if token != "Bearer valid-token" {
            c.JSON(401, gin.H{"error": "Unauthorized"})
            c.Abort() // 停止後續處理
            return
        }

        // 存儲用戶信息
        c.Set("user_id", "123")
        c.Next()
    }
}

func main() {
    r := gin.New() // 不帶默認中間件

    // 全局中間件
    r.Use(Logger())
    r.Use(gin.Recovery())

    // 公開路由
    r.GET("/public", func(c *gin.Context) {
        c.JSON(200, gin.H{"message": "Public endpoint"})
    })

    // 受保護路由組
    protected := r.Group("/api")
    protected.Use(AuthRequired())
    {
        protected.GET("/profile", func(c *gin.Context) {
            userID := c.GetString("user_id")
            c.JSON(200, gin.H{
                "user_id": userID,
                "message": "Protected endpoint",
            })
        })
    }

    r.Run(":8080")
}
```

**Node.js (Express):**
```javascript
const express = require('express');
const app = express();

// 自定義日誌中間件
const logger = (req, res, next) => {
    const start = Date.now();

    res.on('finish', () => {
        const duration = Date.now() - start;
        console.log(`[${req.method}] ${req.url} ${res.statusCode} ${duration}ms`);
    });

    next();
};

// 認證中間件
const authRequired = (req, res, next) => {
    const token = req.headers.authorization;

    if (token !== 'Bearer valid-token') {
        return res.status(401).json({ error: 'Unauthorized' });
    }

    req.userId = '123';
    next();
};

// 全局中間件
app.use(logger);

// 公開路由
app.get('/public', (req, res) => {
    res.json({ message: 'Public endpoint' });
});

// 受保護路由組
const protected = express.Router();
protected.use(authRequired);

protected.get('/profile', (req, res) => {
    res.json({
        user_id: req.userId,
        message: 'Protected endpoint'
    });
});

app.use('/api', protected);

app.listen(8080);
```

### 示例 5: 文件上傳

**Go (Gin):**
```go
package main

import (
    "fmt"
    "path/filepath"

    "github.com/gin-gonic/gin"
)

func main() {
    r := gin.Default()

    // 單文件上傳
    r.POST("/upload", func(c *gin.Context) {
        file, err := c.FormFile("file")
        if err != nil {
            c.JSON(400, gin.H{"error": err.Error()})
            return
        }

        // 保存文件
        filename := filepath.Base(file.Filename)
        dst := "./uploads/" + filename

        if err := c.SaveUploadedFile(file, dst); err != nil {
            c.JSON(500, gin.H{"error": err.Error()})
            return
        }

        c.JSON(200, gin.H{
            "message":  "File uploaded",
            "filename": filename,
            "size":     file.Size,
        })
    })

    // 多文件上傳
    r.POST("/uploads", func(c *gin.Context) {
        form, _ := c.MultipartForm()
        files := form.File["files"]

        var uploadedFiles []string
        for _, file := range files {
            filename := filepath.Base(file.Filename)
            dst := "./uploads/" + filename
            c.SaveUploadedFile(file, dst)
            uploadedFiles = append(uploadedFiles, filename)
        }

        c.JSON(200, gin.H{
            "message": fmt.Sprintf("%d files uploaded", len(files)),
            "files":   uploadedFiles,
        })
    })

    r.Run(":8080")
}
```

**Node.js (Express + multer):**
```javascript
const express = require('express');
const multer = require('multer');
const path = require('path');
const app = express();

const storage = multer.diskStorage({
    destination: './uploads/',
    filename: (req, file, cb) => {
        cb(null, path.basename(file.originalname));
    }
});

const upload = multer({ storage });

// 單文件上傳
app.post('/upload', upload.single('file'), (req, res) => {
    res.json({
        message: 'File uploaded',
        filename: req.file.filename,
        size: req.file.size
    });
});

// 多文件上傳
app.post('/uploads', upload.array('files'), (req, res) => {
    const filenames = req.files.map(f => f.filename);
    res.json({
        message: `${req.files.length} files uploaded`,
        files: filenames
    });
});

app.listen(8080);
```

### 示例 6: 完整的 CRUD API

**Go (Gin):**
```go
package main

import (
    "github.com/gin-gonic/gin"
    "net/http"
    "strconv"
)

type Product struct {
    ID    int     `json:"id"`
    Name  string  `json:"name" binding:"required"`
    Price float64 `json:"price" binding:"required,gt=0"`
}

var products = []Product{
    {ID: 1, Name: "Laptop", Price: 999.99},
    {ID: 2, Name: "Mouse", Price: 29.99},
}
var nextID = 3

func main() {
    r := gin.Default()

    api := r.Group("/api/products")
    {
        api.GET("", getProducts)
        api.GET("/:id", getProduct)
        api.POST("", createProduct)
        api.PUT("/:id", updateProduct)
        api.DELETE("/:id", deleteProduct)
    }

    r.Run(":8080")
}

func getProducts(c *gin.Context) {
    c.JSON(http.StatusOK, products)
}

func getProduct(c *gin.Context) {
    id, _ := strconv.Atoi(c.Param("id"))

    for _, p := range products {
        if p.ID == id {
            c.JSON(http.StatusOK, p)
            return
        }
    }

    c.JSON(http.StatusNotFound, gin.H{"error": "Product not found"})
}

func createProduct(c *gin.Context) {
    var product Product

    if err := c.ShouldBindJSON(&product); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }

    product.ID = nextID
    nextID++
    products = append(products, product)

    c.JSON(http.StatusCreated, product)
}

func updateProduct(c *gin.Context) {
    id, _ := strconv.Atoi(c.Param("id"))
    var product Product

    if err := c.ShouldBindJSON(&product); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }

    for i, p := range products {
        if p.ID == id {
            product.ID = id
            products[i] = product
            c.JSON(http.StatusOK, product)
            return
        }
    }

    c.JSON(http.StatusNotFound, gin.H{"error": "Product not found"})
}

func deleteProduct(c *gin.Context) {
    id, _ := strconv.Atoi(c.Param("id"))

    for i, p := range products {
        if p.ID == id {
            products = append(products[:i], products[i+1:]...)
            c.Status(http.StatusNoContent)
            return
        }
    }

    c.JSON(http.StatusNotFound, gin.H{"error": "Product not found"})
}
```

## 重點總結

1. **Gin 的優勢**
   - 高性能（基於 httprouter）
   - Express 風格的 API
   - 內置參數綁定和驗證
   - 豐富的中間件支持

2. **核心功能**
   - `gin.Default()` - 帶默認中間件
   - `gin.New()` - 空路由器
   - `c.JSON()` - JSON 響應
   - `c.ShouldBind()` - 參數綁定

3. **路由**
   - 支持路徑參數 `:id`
   - 支持通配符 `*path`
   - 路由組 `Group()`
   - 中間件 `Use()`

4. **參數處理**
   - `c.Param()` - 路徑參數
   - `c.Query()` - 查詢參數
   - `c.PostForm()` - 表單數據
   - `c.ShouldBindJSON()` - JSON 綁定

5. **與 Express 的相似性**
   - API 設計相似
   - 中間件概念相同
   - 路由組織相似
   - 學習曲線平緩

## 練習題

1. 實現完整的用戶認證系統（註冊、登錄、JWT）
2. 創建帶分頁和篩選的產品列表 API
3. 實現文件上傳和圖片處理功能
4. 創建 WebSocket 實時聊天室
5. 實現 API 速率限制和緩存
6. 構建完整的博客 API（文章、評論、標籤）
7. 實現多租戶 API（基於子域名或 header）
8. 創建支持國際化的 API
