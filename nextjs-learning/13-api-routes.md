# 第 13 章：API Routes - 建立後端 API

## 概述

Next.js 的 API Routes 讓你可以在同一個專案中建立後端 API，不需要額外的伺服器。對於 Vue3 開發者來說，這類似於 Nuxt.js 的 Server Middleware，但 Next.js 提供了更簡潔和直觀的方式。

## Vue3 vs Next.js API 處理

### Vue3 的傳統做法

```javascript
// 需要另外建立 Express 伺服器
// server.js
const express = require('express')
const app = express()

app.get('/api/users', (req, res) => {
  res.json({ users: [] })
})

app.listen(3000)
```

```javascript
// Vue 組件中呼叫
export default {
  async mounted() {
    const response = await fetch('/api/users')
    this.users = await response.json()
  }
}
```

### Next.js 的做法

```javascript
// pages/api/users.js
export default function handler(req, res) {
  res.status(200).json({ users: [] })
}
```

```javascript
// React 組件中呼叫
export default function UsersPage() {
  const [users, setUsers] = useState([])

  useEffect(() => {
    fetch('/api/users')
      .then(r => r.json())
      .then(data => setUsers(data.users))
  }, [])

  return <div>{/* 渲染 users */}</div>
}
```

## API Routes 基礎

### 檔案結構

```
pages/
  api/
    hello.js          → /api/hello
    users/
      index.js        → /api/users
      [id].js         → /api/users/123
    posts/
      [postId]/
        comments.js   → /api/posts/123/comments
```

### 基本 API Handler

```javascript
// pages/api/hello.js
export default function handler(req, res) {
  // req: HTTP 請求物件
  // res: HTTP 回應物件

  res.status(200).json({
    message: 'Hello from Next.js API!'
  })
}
```

### 處理不同的 HTTP 方法

```javascript
// pages/api/users.js
export default function handler(req, res) {
  const { method } = req

  switch (method) {
    case 'GET':
      // 獲取用戶列表
      return res.status(200).json({
        users: [
          { id: 1, name: 'Alice' },
          { id: 2, name: 'Bob' }
        ]
      })

    case 'POST':
      // 創建新用戶
      const { name, email } = req.body
      return res.status(201).json({
        id: 3,
        name,
        email,
        message: '用戶已創建'
      })

    case 'PUT':
      // 更新用戶
      return res.status(200).json({
        message: '用戶已更新'
      })

    case 'DELETE':
      // 刪除用戶
      return res.status(200).json({
        message: '用戶已刪除'
      })

    default:
      // 不支援的方法
      res.setHeader('Allow', ['GET', 'POST', 'PUT', 'DELETE'])
      return res.status(405).json({
        error: `方法 ${method} 不被允許`
      })
  }
}
```

### 動態 API 路由

```javascript
// pages/api/users/[id].js
export default function handler(req, res) {
  const { id } = req.query
  const { method } = req

  switch (method) {
    case 'GET':
      // 獲取單個用戶
      return res.status(200).json({
        id,
        name: 'Alice',
        email: 'alice@example.com'
      })

    case 'PUT':
      // 更新用戶
      const { name, email } = req.body
      return res.status(200).json({
        id,
        name,
        email,
        message: '用戶已更新'
      })

    case 'DELETE':
      // 刪除用戶
      return res.status(200).json({
        message: `用戶 ${id} 已刪除`
      })

    default:
      res.setHeader('Allow', ['GET', 'PUT', 'DELETE'])
      return res.status(405).end(`方法 ${method} 不被允許`)
  }
}
```

## 實際應用案例

### 案例 1：用戶認證 API

```javascript
// pages/api/auth/login.js
import bcrypt from 'bcryptjs'
import jwt from 'jsonwebtoken'
import { getUserByEmail } from '../../../lib/db'

export default async function handler(req, res) {
  if (req.method !== 'POST') {
    return res.status(405).json({ error: '只允許 POST 請求' })
  }

  try {
    const { email, password } = req.body

    // 驗證輸入
    if (!email || !password) {
      return res.status(400).json({
        error: 'Email 和密碼為必填'
      })
    }

    // 從資料庫獲取用戶
    const user = await getUserByEmail(email)

    if (!user) {
      return res.status(401).json({
        error: 'Email 或密碼錯誤'
      })
    }

    // 驗證密碼
    const isValid = await bcrypt.compare(password, user.passwordHash)

    if (!isValid) {
      return res.status(401).json({
        error: 'Email 或密碼錯誤'
      })
    }

    // 生成 JWT
    const token = jwt.sign(
      {
        userId: user.id,
        email: user.email
      },
      process.env.JWT_SECRET,
      { expiresIn: '7d' }
    )

    // 設定 cookie
    res.setHeader(
      'Set-Cookie',
      `token=${token}; Path=/; HttpOnly; Secure; SameSite=Strict; Max-Age=${7 * 24 * 60 * 60}`
    )

    return res.status(200).json({
      user: {
        id: user.id,
        name: user.name,
        email: user.email
      },
      token
    })
  } catch (error) {
    console.error('登入錯誤：', error)
    return res.status(500).json({
      error: '伺服器錯誤，請稍後再試'
    })
  }
}
```

```javascript
// pages/api/auth/register.js
import bcrypt from 'bcryptjs'
import { createUser, getUserByEmail } from '../../../lib/db'

export default async function handler(req, res) {
  if (req.method !== 'POST') {
    return res.status(405).json({ error: '只允許 POST 請求' })
  }

  try {
    const { name, email, password } = req.body

    // 驗證輸入
    if (!name || !email || !password) {
      return res.status(400).json({
        error: '所有欄位為必填'
      })
    }

    if (password.length < 8) {
      return res.status(400).json({
        error: '密碼至少需要 8 個字元'
      })
    }

    // 檢查 email 是否已存在
    const existingUser = await getUserByEmail(email)

    if (existingUser) {
      return res.status(409).json({
        error: '此 Email 已被使用'
      })
    }

    // 雜湊密碼
    const passwordHash = await bcrypt.hash(password, 10)

    // 創建用戶
    const user = await createUser({
      name,
      email,
      passwordHash
    })

    return res.status(201).json({
      message: '註冊成功',
      user: {
        id: user.id,
        name: user.name,
        email: user.email
      }
    })
  } catch (error) {
    console.error('註冊錯誤：', error)
    return res.status(500).json({
      error: '伺服器錯誤，請稍後再試'
    })
  }
}
```

```javascript
// pages/api/auth/me.js
import jwt from 'jsonwebtoken'
import { getUserById } from '../../../lib/db'

export default async function handler(req, res) {
  if (req.method !== 'GET') {
    return res.status(405).json({ error: '只允許 GET 請求' })
  }

  try {
    // 從 cookie 或 Authorization header 獲取 token
    const token = req.cookies.token || req.headers.authorization?.replace('Bearer ', '')

    if (!token) {
      return res.status(401).json({
        error: '未提供驗證令牌'
      })
    }

    // 驗證 token
    const decoded = jwt.verify(token, process.env.JWT_SECRET)

    // 獲取用戶資訊
    const user = await getUserById(decoded.userId)

    if (!user) {
      return res.status(404).json({
        error: '用戶不存在'
      })
    }

    return res.status(200).json({
      user: {
        id: user.id,
        name: user.name,
        email: user.email
      }
    })
  } catch (error) {
    if (error.name === 'JsonWebTokenError') {
      return res.status(401).json({
        error: '無效的令牌'
      })
    }

    if (error.name === 'TokenExpiredError') {
      return res.status(401).json({
        error: '令牌已過期'
      })
    }

    console.error('驗證錯誤：', error)
    return res.status(500).json({
      error: '伺服器錯誤'
    })
  }
}
```

### 案例 2：CRUD API（以部落格文章為例）

```javascript
// pages/api/posts/index.js
import { getPosts, createPost } from '../../../lib/db'

export default async function handler(req, res) {
  const { method } = req

  switch (method) {
    case 'GET':
      try {
        const { page = 1, limit = 10, category } = req.query

        const posts = await getPosts({
          page: parseInt(page),
          limit: parseInt(limit),
          category
        })

        return res.status(200).json(posts)
      } catch (error) {
        console.error('獲取文章失敗：', error)
        return res.status(500).json({ error: '伺服器錯誤' })
      }

    case 'POST':
      try {
        const { title, content, categoryId } = req.body

        // 驗證輸入
        if (!title || !content) {
          return res.status(400).json({
            error: '標題和內容為必填'
          })
        }

        // 創建文章
        const post = await createPost({
          title,
          content,
          categoryId,
          authorId: req.user?.id // 從中介軟體獲取
        })

        return res.status(201).json(post)
      } catch (error) {
        console.error('創建文章失敗：', error)
        return res.status(500).json({ error: '伺服器錯誤' })
      }

    default:
      res.setHeader('Allow', ['GET', 'POST'])
      return res.status(405).end(`方法 ${method} 不被允許`)
  }
}
```

```javascript
// pages/api/posts/[id].js
import { getPostById, updatePost, deletePost } from '../../../lib/db'

export default async function handler(req, res) {
  const { id } = req.query
  const { method } = req

  switch (method) {
    case 'GET':
      try {
        const post = await getPostById(id)

        if (!post) {
          return res.status(404).json({ error: '文章不存在' })
        }

        return res.status(200).json(post)
      } catch (error) {
        console.error('獲取文章失敗：', error)
        return res.status(500).json({ error: '伺服器錯誤' })
      }

    case 'PUT':
      try {
        const { title, content, categoryId } = req.body

        const updatedPost = await updatePost(id, {
          title,
          content,
          categoryId
        })

        return res.status(200).json(updatedPost)
      } catch (error) {
        console.error('更新文章失敗：', error)
        return res.status(500).json({ error: '伺服器錯誤' })
      }

    case 'DELETE':
      try {
        await deletePost(id)

        return res.status(200).json({
          message: '文章已刪除'
        })
      } catch (error) {
        console.error('刪除文章失敗：', error)
        return res.status(500).json({ error: '伺服器錯誤' })
      }

    default:
      res.setHeader('Allow', ['GET', 'PUT', 'DELETE'])
      return res.status(405).end(`方法 ${method} 不被允許`)
  }
}
```

### 案例 3：檔案上傳

```javascript
// pages/api/upload.js
import formidable from 'formidable'
import fs from 'fs'
import path from 'path'

// 禁用 Next.js 的 body parser
export const config = {
  api: {
    bodyParser: false
  }
}

export default async function handler(req, res) {
  if (req.method !== 'POST') {
    return res.status(405).json({ error: '只允許 POST 請求' })
  }

  const uploadDir = path.join(process.cwd(), 'public', 'uploads')

  // 確保上傳目錄存在
  if (!fs.existsSync(uploadDir)) {
    fs.mkdirSync(uploadDir, { recursive: true })
  }

  const form = formidable({
    uploadDir,
    keepExtensions: true,
    maxFileSize: 10 * 1024 * 1024 // 10MB
  })

  form.parse(req, (err, fields, files) => {
    if (err) {
      console.error('上傳錯誤：', err)
      return res.status(500).json({ error: '檔案上傳失敗' })
    }

    const file = files.file

    // 生成可存取的 URL
    const fileName = path.basename(file.filepath)
    const fileUrl = `/uploads/${fileName}`

    return res.status(200).json({
      message: '檔案上傳成功',
      url: fileUrl,
      fileName: file.originalFilename,
      size: file.size
    })
  })
}
```

### 案例 4：第三方 API 代理

```javascript
// pages/api/weather.js
export default async function handler(req, res) {
  const { city } = req.query

  if (!city) {
    return res.status(400).json({
      error: '請提供城市名稱'
    })
  }

  try {
    // 呼叫第三方天氣 API
    const response = await fetch(
      `https://api.openweathermap.org/data/2.5/weather?q=${city}&appid=${process.env.WEATHER_API_KEY}&units=metric&lang=zh_tw`
    )

    if (!response.ok) {
      return res.status(response.status).json({
        error: '無法獲取天氣資訊'
      })
    }

    const data = await response.json()

    // 轉換並返回資料
    return res.status(200).json({
      city: data.name,
      temperature: data.main.temp,
      description: data.weather[0].description,
      humidity: data.main.humidity,
      windSpeed: data.wind.speed
    })
  } catch (error) {
    console.error('天氣 API 錯誤：', error)
    return res.status(500).json({
      error: '伺服器錯誤'
    })
  }
}
```

### 案例 5：資料庫整合（Prisma）

```javascript
// pages/api/products/index.js
import { PrismaClient } from '@prisma/client'

const prisma = new PrismaClient()

export default async function handler(req, res) {
  const { method } = req

  switch (method) {
    case 'GET':
      try {
        const { category, minPrice, maxPrice, search } = req.query

        // 建立查詢條件
        const where = {}

        if (category) {
          where.categoryId = parseInt(category)
        }

        if (minPrice || maxPrice) {
          where.price = {}
          if (minPrice) where.price.gte = parseFloat(minPrice)
          if (maxPrice) where.price.lte = parseFloat(maxPrice)
        }

        if (search) {
          where.OR = [
            { name: { contains: search, mode: 'insensitive' } },
            { description: { contains: search, mode: 'insensitive' } }
          ]
        }

        // 查詢產品
        const products = await prisma.product.findMany({
          where,
          include: {
            category: true
          },
          orderBy: {
            createdAt: 'desc'
          }
        })

        return res.status(200).json({ products })
      } catch (error) {
        console.error('查詢產品失敗：', error)
        return res.status(500).json({ error: '伺服器錯誤' })
      }

    case 'POST':
      try {
        const { name, description, price, stock, categoryId } = req.body

        const product = await prisma.product.create({
          data: {
            name,
            description,
            price: parseFloat(price),
            stock: parseInt(stock),
            categoryId: parseInt(categoryId)
          }
        })

        return res.status(201).json(product)
      } catch (error) {
        console.error('創建產品失敗：', error)
        return res.status(500).json({ error: '伺服器錯誤' })
      }

    default:
      res.setHeader('Allow', ['GET', 'POST'])
      return res.status(405).end(`方法 ${method} 不被允許`)
  }
}
```

## 中介軟體（Middleware）模式

### 建立認證中介軟體

```javascript
// lib/middleware/auth.js
import jwt from 'jsonwebtoken'
import { getUserById } from '../db'

export function authMiddleware(handler) {
  return async (req, res) => {
    try {
      const token = req.cookies.token || req.headers.authorization?.replace('Bearer ', '')

      if (!token) {
        return res.status(401).json({ error: '未授權' })
      }

      const decoded = jwt.verify(token, process.env.JWT_SECRET)
      const user = await getUserById(decoded.userId)

      if (!user) {
        return res.status(401).json({ error: '用戶不存在' })
      }

      // 將用戶資訊附加到 req
      req.user = user

      return handler(req, res)
    } catch (error) {
      return res.status(401).json({ error: '無效的令牌' })
    }
  }
}
```

### 使用中介軟體

```javascript
// pages/api/protected.js
import { authMiddleware } from '../../lib/middleware/auth'

async function handler(req, res) {
  // req.user 已經由中介軟體設定
  return res.status(200).json({
    message: '這是受保護的資料',
    user: req.user
  })
}

export default authMiddleware(handler)
```

### 組合多個中介軟體

```javascript
// lib/middleware/compose.js
export function compose(...middlewares) {
  return (handler) => {
    return middlewares.reduceRight(
      (next, middleware) => middleware(next),
      handler
    )
  }
}
```

```javascript
// pages/api/admin/users.js
import { compose } from '../../../lib/middleware/compose'
import { authMiddleware } from '../../../lib/middleware/auth'
import { adminMiddleware } from '../../../lib/middleware/admin'

async function handler(req, res) {
  // 只有認證的管理員可以存取
  return res.status(200).json({ users: [] })
}

export default compose(authMiddleware, adminMiddleware)(handler)
```

## 錯誤處理最佳實踐

### 統一的錯誤處理

```javascript
// lib/api-error.js
export class ApiError extends Error {
  constructor(statusCode, message) {
    super(message)
    this.statusCode = statusCode
  }
}

export function errorHandler(handler) {
  return async (req, res) => {
    try {
      await handler(req, res)
    } catch (error) {
      console.error('API 錯誤：', error)

      if (error instanceof ApiError) {
        return res.status(error.statusCode).json({
          error: error.message
        })
      }

      // 未知錯誤
      return res.status(500).json({
        error: '伺服器錯誤，請稍後再試'
      })
    }
  }
}
```

```javascript
// pages/api/posts/[id].js
import { errorHandler, ApiError } from '../../../lib/api-error'
import { getPostById } from '../../../lib/db'

async function handler(req, res) {
  const { id } = req.query

  const post = await getPostById(id)

  if (!post) {
    throw new ApiError(404, '文章不存在')
  }

  return res.status(200).json(post)
}

export default errorHandler(handler)
```

## CORS 設定

```javascript
// pages/api/public-data.js
function setCorsHeaders(res) {
  res.setHeader('Access-Control-Allow-Origin', '*')
  res.setHeader('Access-Control-Allow-Methods', 'GET, POST, PUT, DELETE, OPTIONS')
  res.setHeader('Access-Control-Allow-Headers', 'Content-Type, Authorization')
}

export default function handler(req, res) {
  setCorsHeaders(res)

  if (req.method === 'OPTIONS') {
    return res.status(200).end()
  }

  return res.status(200).json({ data: 'public data' })
}
```

## 速率限制

```javascript
// lib/rate-limit.js
const rateLimit = new Map()

export function rateLimiter({ limit = 10, window = 60000 }) {
  return (handler) => {
    return async (req, res) => {
      const ip = req.headers['x-forwarded-for'] || req.connection.remoteAddress
      const now = Date.now()
      const userLimit = rateLimit.get(ip) || { count: 0, resetTime: now + window }

      if (now > userLimit.resetTime) {
        userLimit.count = 0
        userLimit.resetTime = now + window
      }

      userLimit.count++

      if (userLimit.count > limit) {
        return res.status(429).json({
          error: '請求過於頻繁，請稍後再試'
        })
      }

      rateLimit.set(ip, userLimit)

      return handler(req, res)
    }
  }
}
```

```javascript
// pages/api/login.js
import { rateLimiter } from '../../lib/rate-limit'

async function handler(req, res) {
  // 登入邏輯
}

export default rateLimiter({ limit: 5, window: 60000 })(handler)
```

## 與 Vue3 的比較總結

| 功能 | Vue3 (純前端) | Nuxt.js | Next.js API Routes |
|------|---------------|---------|-------------------|
| API 建立 | 需要額外伺服器 | Server Middleware | 內建支援 |
| 檔案結構 | 獨立專案 | `/server` 目錄 | `/pages/api` 目錄 |
| 路由 | 手動設定 | 基於檔案 | 基於檔案 |
| 部署 | 需要 Node 伺服器 | 需要 Node 伺服器 | Serverless 或 Node |

## 最佳實踐

1. **使用 TypeScript**
2. **統一的錯誤處理**
3. **使用中介軟體模式**
4. **驗證所有輸入**
5. **適當的速率限制**
6. **安全的認證機制**
7. **記錄日誌**
8. **使用環境變數管理敏感資訊**

## 小結

Next.js API Routes 提供了一個簡單而強大的方式來建立後端 API，讓你可以在同一個專案中處理前端和後端邏輯。對於 Vue3 開發者來說，這大大簡化了全端開發的流程，不需要維護獨立的後端專案。
