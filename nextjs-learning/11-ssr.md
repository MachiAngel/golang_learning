# 第 11 章：SSR (Server-Side Rendering) 深入解析

## 概述

SSR（伺服器端渲染）在每次請求時都會在伺服器上重新渲染頁面。對於 Vue3 開發者來說，這類似於 Nuxt.js 的 SSR 模式，但 Next.js 讓 SSR 和 SSG 可以在同一個應用中共存。

## Vue3 SSR vs Next.js SSR

### Vue3 + Nuxt.js 的做法

```javascript
// Nuxt.js 2
export default {
  async asyncData({ params, $axios }) {
    const post = await $axios.$get(`/api/posts/${params.id}`)
    return { post }
  }
}

// Nuxt.js 3
<script setup>
const { data: post } = await useFetch(`/api/posts/${route.params.id}`)
</script>
```

### Next.js 的做法

```javascript
// pages/posts/[id].js
export async function getServerSideProps({ params }) {
  const response = await fetch(`https://api.example.com/posts/${params.id}`)
  const post = await response.json()

  return {
    props: { post }
  }
}

export default function PostPage({ post }) {
  return (
    <div>
      <h1>{post.title}</h1>
      <p>{post.content}</p>
    </div>
  )
}
```

## SSR 的運作流程

```
用戶請求頁面
  ↓
Next.js 伺服器接收請求
  ↓
執行 getServerSideProps
  ↓
獲取資料（API、資料庫等）
  ↓
渲染 React 組件為 HTML
  ↓
返回完整的 HTML 給用戶
  ↓
瀏覽器顯示 HTML
  ↓
下載並執行 JavaScript
  ↓
React Hydration（附加互動功能）
```

## getServerSideProps 完整解析

### 基本用法

```javascript
export async function getServerSideProps(context) {
  // 這段程式碼在每次請求時執行
  // 永遠在伺服器端執行，不會在用戶端執行

  const data = await fetchData()

  return {
    props: {
      data
    }
  }
}

export default function Page({ data }) {
  return <div>{JSON.stringify(data)}</div>
}
```

### context 參數詳解

```javascript
export async function getServerSideProps(context) {
  const {
    params,        // 動態路由參數
    req,          // HTTP 請求物件
    res,          // HTTP 回應物件
    query,        // URL query 參數
    preview,      // 預覽模式
    previewData,  // 預覽資料
    resolvedUrl,  // 請求的完整 URL
    locale,       // 當前語言
    locales,      // 所有可用語言
    defaultLocale // 預設語言
  } = context

  // 存取 cookies
  const cookies = req.cookies

  // 存取 headers
  const userAgent = req.headers['user-agent']

  // 存取 IP 地址
  const ip = req.headers['x-forwarded-for'] || req.connection.remoteAddress

  return {
    props: {
      userAgent,
      ip,
      query
    }
  }
}
```

### 完整的返回選項

```javascript
export async function getServerSideProps(context) {
  return {
    // 選項 1：返回 props
    props: {
      data: {}
    }

    // 選項 2：返回 404
    // notFound: true

    // 選項 3：重定向
    // redirect: {
    //   destination: '/login',
    //   permanent: false
    // }
  }
}
```

## 實際應用案例

### 案例 1：需要身份驗證的頁面

```javascript
// pages/dashboard.js
import { getSession } from 'next-auth/react'

export async function getServerSideProps(context) {
  // 檢查用戶是否登入
  const session = await getSession(context)

  if (!session) {
    return {
      redirect: {
        destination: '/login?callbackUrl=/dashboard',
        permanent: false
      }
    }
  }

  // 根據用戶資訊獲取資料
  const response = await fetch('https://api.example.com/user/dashboard', {
    headers: {
      'Authorization': `Bearer ${session.accessToken}`
    }
  })

  const data = await response.json()

  return {
    props: {
      user: session.user,
      data
    }
  }
}

export default function Dashboard({ user, data }) {
  return (
    <div>
      <h1>歡迎，{user.name}</h1>
      <div className="stats">
        <div>訂單數：{data.orderCount}</div>
        <div>總消費：NT$ {data.totalSpent}</div>
      </div>
    </div>
  )
}
```

### 案例 2：依賴 Query Parameters 的頁面

```javascript
// pages/search.js
export async function getServerSideProps({ query }) {
  const searchTerm = query.q || ''
  const page = parseInt(query.page) || 1
  const category = query.category || 'all'

  if (!searchTerm) {
    return {
      props: {
        results: [],
        searchTerm: '',
        page,
        category
      }
    }
  }

  // 執行搜尋
  const response = await fetch(
    `https://api.example.com/search?q=${encodeURIComponent(searchTerm)}&page=${page}&category=${category}`
  )
  const results = await response.json()

  return {
    props: {
      results,
      searchTerm,
      page,
      category,
      totalPages: results.totalPages
    }
  }
}

export default function SearchPage({ results, searchTerm, page, category, totalPages }) {
  return (
    <div>
      <h1>搜尋結果：{searchTerm}</h1>
      <p>分類：{category}</p>

      <div className="results">
        {results.items?.map(item => (
          <div key={item.id}>
            <h2>{item.title}</h2>
            <p>{item.description}</p>
          </div>
        ))}
      </div>

      <div className="pagination">
        {page > 1 && (
          <a href={`/search?q=${searchTerm}&page=${page - 1}&category=${category}`}>
            上一頁
          </a>
        )}
        <span>第 {page} 頁 / 共 {totalPages} 頁</span>
        {page < totalPages && (
          <a href={`/search?q=${searchTerm}&page=${page + 1}&category=${category}`}>
            下一頁
          </a>
        )}
      </div>
    </div>
  )
}
```

### 案例 3：個人化內容

```javascript
// pages/recommendations.js
import { parse } from 'cookie'

export async function getServerSideProps({ req }) {
  // 從 cookie 讀取用戶偏好
  const cookies = parse(req.headers.cookie || '')
  const userId = cookies.userId
  const preferences = cookies.preferences ? JSON.parse(cookies.preferences) : {}

  // 根據用戶 IP 取得地理位置
  const ip = req.headers['x-forwarded-for'] || req.connection.remoteAddress
  const geoResponse = await fetch(`https://api.ipapi.com/${ip}/json`)
  const geoData = await geoResponse.json()

  // 獲取個人化推薦
  const recommendationsResponse = await fetch('https://api.example.com/recommendations', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      userId,
      preferences,
      country: geoData.country,
      city: geoData.city
    })
  })

  const recommendations = await recommendationsResponse.json()

  return {
    props: {
      recommendations,
      location: {
        country: geoData.country,
        city: geoData.city
      }
    }
  }
}

export default function RecommendationsPage({ recommendations, location }) {
  return (
    <div>
      <h1>{location.city}, {location.country} 的推薦商品</h1>
      <div className="recommendations">
        {recommendations.map(item => (
          <div key={item.id} className="product-card">
            <img src={item.image} alt={item.name} />
            <h3>{item.name}</h3>
            <p>NT$ {item.price}</p>
          </div>
        ))}
      </div>
    </div>
  )
}
```

### 案例 4：即時資料

```javascript
// pages/stock/[symbol].js
export async function getServerSideProps({ params }) {
  const symbol = params.symbol.toUpperCase()

  // 獲取即時股價
  const response = await fetch(`https://api.stock.com/quote/${symbol}`)

  if (!response.ok) {
    return {
      notFound: true
    }
  }

  const stock = await response.json()

  // 獲取歷史資料
  const historyResponse = await fetch(`https://api.stock.com/history/${symbol}?days=30`)
  const history = await historyResponse.json()

  return {
    props: {
      stock,
      history,
      fetchedAt: new Date().toISOString()
    }
  }
}

export default function StockPage({ stock, history, fetchedAt }) {
  return (
    <div>
      <h1>{stock.symbol} - {stock.name}</h1>
      <div className="stock-info">
        <div className="price">
          <span className="current">NT$ {stock.price}</span>
          <span className={stock.change > 0 ? 'positive' : 'negative'}>
            {stock.change > 0 ? '+' : ''}{stock.change} ({stock.changePercent}%)
          </span>
        </div>
        <p className="timestamp">
          更新時間：{new Date(fetchedAt).toLocaleString('zh-TW')}
        </p>
      </div>

      <div className="chart">
        <h2>30 天走勢</h2>
        {/* 渲染圖表 */}
      </div>

      <button onClick={() => window.location.reload()}>
        重新整理
      </button>
    </div>
  )
}
```

### 案例 5：從資料庫直接查詢

```javascript
// pages/admin/users.js
import { PrismaClient } from '@prisma/client'

const prisma = new PrismaClient()

export async function getServerSideProps({ query }) {
  const page = parseInt(query.page) || 1
  const limit = 20
  const skip = (page - 1) * limit

  // 直接查詢資料庫
  const [users, totalCount] = await Promise.all([
    prisma.user.findMany({
      skip,
      take: limit,
      orderBy: { createdAt: 'desc' },
      select: {
        id: true,
        name: true,
        email: true,
        createdAt: true,
        _count: {
          select: { orders: true }
        }
      }
    }),
    prisma.user.count()
  ])

  // 轉換 Date 物件為字串（以便序列化）
  const serializedUsers = users.map(user => ({
    ...user,
    createdAt: user.createdAt.toISOString()
  }))

  return {
    props: {
      users: serializedUsers,
      totalPages: Math.ceil(totalCount / limit),
      currentPage: page
    }
  }
}

export default function UsersPage({ users, totalPages, currentPage }) {
  return (
    <div>
      <h1>用戶管理</h1>
      <table>
        <thead>
          <tr>
            <th>ID</th>
            <th>姓名</th>
            <th>Email</th>
            <th>訂單數</th>
            <th>註冊時間</th>
          </tr>
        </thead>
        <tbody>
          {users.map(user => (
            <tr key={user.id}>
              <td>{user.id}</td>
              <td>{user.name}</td>
              <td>{user.email}</td>
              <td>{user._count.orders}</td>
              <td>{new Date(user.createdAt).toLocaleDateString('zh-TW')}</td>
            </tr>
          ))}
        </tbody>
      </table>

      <div className="pagination">
        {/* 分頁導航 */}
      </div>
    </div>
  )
}
```

## 存取請求和回應

### 讀取 Headers

```javascript
export async function getServerSideProps({ req }) {
  const userAgent = req.headers['user-agent']
  const acceptLanguage = req.headers['accept-language']
  const referer = req.headers['referer']

  return {
    props: {
      userAgent,
      acceptLanguage,
      referer
    }
  }
}
```

### 讀取和設定 Cookies

```javascript
import { parse, serialize } from 'cookie'

export async function getServerSideProps({ req, res }) {
  // 讀取 cookies
  const cookies = parse(req.headers.cookie || '')
  const userId = cookies.userId

  // 設定 cookie
  res.setHeader(
    'Set-Cookie',
    serialize('lastVisit', new Date().toISOString(), {
      path: '/',
      maxAge: 60 * 60 * 24 * 7 // 7 天
    })
  )

  return {
    props: {
      userId
    }
  }
}
```

### 設定自訂 Headers

```javascript
export async function getServerSideProps({ res }) {
  // 設定快取 headers
  res.setHeader(
    'Cache-Control',
    'public, s-maxage=10, stale-while-revalidate=59'
  )

  // 設定其他 headers
  res.setHeader('X-Custom-Header', 'value')

  return {
    props: {}
  }
}
```

## 錯誤處理

### 404 頁面

```javascript
export async function getServerSideProps({ params }) {
  const post = await fetchPost(params.id)

  if (!post) {
    return {
      notFound: true // 顯示 404 頁面
    }
  }

  return {
    props: { post }
  }
}
```

### 重定向

```javascript
export async function getServerSideProps({ req, res }) {
  const session = await getSession({ req })

  // 未登入用戶重定向到登入頁
  if (!session) {
    return {
      redirect: {
        destination: '/login',
        permanent: false // 307 臨時重定向
      }
    }
  }

  // 永久重定向範例
  if (shouldRedirect) {
    return {
      redirect: {
        destination: '/new-url',
        permanent: true // 308 永久重定向
      }
    }
  }

  return {
    props: {}
  }
}
```

### Try-Catch 錯誤處理

```javascript
export async function getServerSideProps({ params }) {
  try {
    const response = await fetch(`https://api.example.com/posts/${params.id}`)

    if (!response.ok) {
      if (response.status === 404) {
        return { notFound: true }
      }
      throw new Error(`API 錯誤：${response.status}`)
    }

    const post = await response.json()

    return {
      props: { post }
    }
  } catch (error) {
    console.error('獲取資料失敗：', error)

    // 返回錯誤狀態
    return {
      props: {
        error: {
          message: '無法載入文章，請稍後再試',
          code: 'FETCH_ERROR'
        }
      }
    }
  }
}

export default function PostPage({ post, error }) {
  if (error) {
    return (
      <div className="error">
        <h1>發生錯誤</h1>
        <p>{error.message}</p>
      </div>
    )
  }

  return <div>{post.title}</div>
}
```

## SSR 效能優化

### 1. 快取策略

```javascript
export async function getServerSideProps({ res }) {
  // 設定快取 headers
  res.setHeader(
    'Cache-Control',
    'public, s-maxage=10, stale-while-revalidate=59'
  )

  const data = await fetchData()

  return {
    props: { data }
  }
}
```

**解釋：**
- `s-maxage=10`：CDN 快取 10 秒
- `stale-while-revalidate=59`：過期後的 59 秒內仍返回舊資料，同時在背景更新

### 2. 並行資料獲取

```javascript
export async function getServerSideProps() {
  // ❌ 不好的做法：序列執行
  const user = await fetchUser()
  const posts = await fetchPosts()
  const comments = await fetchComments()

  // ✅ 好的做法：並行執行
  const [user, posts, comments] = await Promise.all([
    fetchUser(),
    fetchPosts(),
    fetchComments()
  ])

  return {
    props: { user, posts, comments }
  }
}
```

### 3. 資料庫連線池

```javascript
// lib/db.js
import { Pool } from 'pg'

// 使用連線池，避免每次都建立新連線
const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
  max: 20
})

export default pool
```

```javascript
// pages/api/users.js
import pool from '../../lib/db'

export async function getServerSideProps() {
  const { rows } = await pool.query('SELECT * FROM users LIMIT 10')

  return {
    props: { users: rows }
  }
}
```

### 4. 減少傳輸資料量

```javascript
export async function getServerSideProps() {
  const response = await fetch('https://api.example.com/users')
  const users = await response.json()

  // ❌ 不好的做法：傳遞完整資料
  // return { props: { users } }

  // ✅ 好的做法：只傳遞必要欄位
  const simplifiedUsers = users.map(user => ({
    id: user.id,
    name: user.name,
    email: user.email
    // 移除不需要的欄位
  }))

  return {
    props: { users: simplifiedUsers }
  }
}
```

## SSR vs SSG vs CSR 比較

| 特性 | SSR | SSG | CSR (Vue3 SPA) |
|------|-----|-----|----------------|
| 執行時機 | 每次請求 | 構建時 | 用戶端 |
| 資料新鮮度 | 即時 | 靜態（可用 ISR） | 即時 |
| 首次載入速度 | 快 | 最快 | 慢 |
| 伺服器負載 | 高 | 無 | 無 |
| 可存取請求資訊 | ✅ | ❌ | ❌ |
| SEO | ✅ | ✅ | ⚠️ |
| 成本 | 高 | 低 | 低 |
| 個人化內容 | ✅ | ❌ | ✅ |

## 何時使用 SSR

### ✅ 適合使用 SSR 的場景

1. **需要身份驗證的頁面**
   - 用戶個人儀表板
   - 帳戶設定頁面
   - 管理後台

2. **個人化內容**
   - 根據地理位置顯示不同內容
   - 根據用戶偏好推薦
   - 根據用戶權限顯示功能

3. **即時資料**
   - 股票價格
   - 即時庫存
   - 運動比分

4. **依賴請求資訊**
   - 需要讀取 cookies
   - 需要讀取 query parameters
   - 需要讀取 headers

5. **搜尋結果頁面**
   - 根據搜尋詞顯示結果
   - 動態篩選和排序

### ❌ 不適合使用 SSR 的場景

1. **靜態內容**
   - 部落格文章（用 SSG）
   - 產品介紹頁面（用 SSG）
   - 文件頁面（用 SSG）

2. **高流量頁面**
   - 首頁（用 SSG + ISR）
   - 熱門產品頁面（用 SSG + ISR）

3. **不需要 SEO 的頁面**
   - 後台管理介面（用 CSR）
   - 互動式工具（用 CSR）

## 與 Vue3 的比較總結

```javascript
// Vue3 (Nuxt.js) 的做法
export default {
  async asyncData({ $axios, params }) {
    const post = await $axios.$get(`/api/posts/${params.id}`)
    return { post }
  }
}

// Next.js 的做法
export async function getServerSideProps({ params }) {
  const response = await fetch(`https://api.example.com/posts/${params.id}`)
  const post = await response.json()
  return { props: { post } }
}
```

**主要差異：**
- Next.js：函式更明確，易於理解
- Next.js：可以在同一個 app 中混用 SSR、SSG、CSR
- Next.js：TypeScript 支援更好
- 兩者：核心概念相同，都是在伺服器端獲取資料並渲染

## 最佳實踐

### 1. 序列化問題

```javascript
// ❌ 錯誤：Date 物件無法序列化
export async function getServerSideProps() {
  return {
    props: {
      date: new Date() // 會報錯！
    }
  }
}

// ✅ 正確：轉換為字串
export async function getServerSideProps() {
  return {
    props: {
      date: new Date().toISOString()
    }
  }
}
```

### 2. 避免在組件中使用 window

```javascript
// ❌ 錯誤：SSR 時 window 不存在
export default function Page() {
  const width = window.innerWidth // 會報錯！

  return <div>{width}</div>
}

// ✅ 正確：使用 useEffect
export default function Page() {
  const [width, setWidth] = useState(0)

  useEffect(() => {
    setWidth(window.innerWidth)
  }, [])

  return <div>{width}</div>
}
```

### 3. 使用 TypeScript

```typescript
import { GetServerSideProps } from 'next'

interface PageProps {
  user: {
    id: number
    name: string
  }
}

export const getServerSideProps: GetServerSideProps<PageProps> = async ({ params }) => {
  const user = await fetchUser(params.id)

  return {
    props: { user }
  }
}

export default function Page({ user }: PageProps) {
  return <h1>{user.name}</h1>
}
```

## 小結

SSR 讓 Next.js 能夠處理需要即時資料和個人化內容的場景。對於有 Vue3 經驗的開發者，SSR 的概念應該很熟悉，但 Next.js 提供了更靈活的方式，讓你可以在同一個應用中混用 SSR、SSG 和 CSR，根據每個頁面的需求選擇最適合的渲染模式。
