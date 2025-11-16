# 第八章：資料獲取：getServerSideProps

## 8.1 什麼是 getServerSideProps？

`getServerSideProps` 是 Next.js 提供的資料獲取方法，用於在**每次請求時（Request Time）**在伺服器端獲取資料。

### 特點

- ✅ **伺服器端渲染（SSR）**：每次請求都在伺服器端執行
- ✅ **即時資料**：總是獲取最新的資料
- ✅ **SEO 友善**：搜尋引擎可以抓取完整內容
- ✅ **存取請求資訊**：可以使用 cookies、headers、查詢參數
- ⚠️ **速度較慢**：每次請求都需要執行，TTFB（Time To First Byte）較長

### getStaticProps vs getServerSideProps

| 特性 | getStaticProps | getServerSideProps |
|------|----------------|-------------------|
| 執行時機 | 建置時 | 每次請求時 |
| 速度 | 極快（靜態 HTML） | 較慢（需伺服器處理） |
| 資料更新 | 需重新建置或 ISR | 即時 |
| CDN 快取 | 可以 | 需額外配置 |
| 適用場景 | 靜態內容 | 動態內容 |

### Vue3/Nuxt3 對比

**Nuxt3（伺服器端獲取）**：
```vue
<script setup>
// useFetch 會在伺服器端執行
const { data } = await useFetch('/api/user', {
  server: true
})
</script>
```

**Next.js**：
```javascript
export async function getServerSideProps() {
  const res = await fetch('https://api.example.com/user')
  const user = await res.json()

  return {
    props: { user }
  }
}
```

## 8.2 基本用法

### 8.2.1 簡單範例

**pages/dashboard.js**：
```javascript
export async function getServerSideProps() {
  // 每次請求時執行
  const res = await fetch('https://api.example.com/user/stats')
  const stats = await res.json()

  return {
    props: {
      stats,
      timestamp: new Date().toISOString()
    }
  }
}

export default function Dashboard({ stats, timestamp }) {
  return (
    <div>
      <h1>儀表板</h1>
      <p>訪客數：{stats.visitors}</p>
      <p>最後更新：{timestamp}</p>
    </div>
  )
}
```

每次訪問此頁面，都會重新獲取最新資料。

### 8.2.2 執行時機

```javascript
export async function getServerSideProps() {
  console.log('這會在每次請求時執行')
  console.log('只在伺服器端執行，不會在瀏覽器執行')

  return {
    props: {
      serverTime: new Date().toISOString()
    }
  }
}
```

## 8.3 Context 參數

`getServerSideProps` 接收一個 context 物件，包含豐富的請求資訊。

```javascript
export async function getServerSideProps(context) {
  const {
    req,          // HTTP 請求物件
    res,          // HTTP 回應物件
    params,       // 動態路由參數
    query,        // 查詢字串參數
    preview,      // 預覽模式
    previewData,  // 預覽資料
    resolvedUrl,  // 完整 URL
    locale,       // 當前語系
    locales,      // 所有語系
    defaultLocale // 預設語系
  } = context

  return {
    props: {}
  }
}
```

### 8.3.1 req 和 res

```javascript
export async function getServerSideProps({ req, res }) {
  // 取得 cookies
  const cookies = req.cookies

  // 取得 headers
  const userAgent = req.headers['user-agent']

  // 設定 cache headers
  res.setHeader(
    'Cache-Control',
    'public, s-maxage=10, stale-while-revalidate=59'
  )

  return {
    props: {
      cookies,
      userAgent
    }
  }
}
```

### 8.3.2 params（動態路由）

```javascript
// pages/users/[id].js
export async function getServerSideProps({ params }) {
  const { id } = params

  const res = await fetch(`https://api.example.com/users/${id}`)
  const user = await res.json()

  return {
    props: { user }
  }
}

export default function UserProfile({ user }) {
  return (
    <div>
      <h1>{user.name}</h1>
      <p>{user.email}</p>
    </div>
  )
}
```

### 8.3.3 query（查詢參數）

```javascript
// pages/search.js
export async function getServerSideProps({ query }) {
  const { q, category, page = 1 } = query

  // URL: /search?q=nextjs&category=tutorial&page=2
  // q = "nextjs"
  // category = "tutorial"
  // page = "2"

  const results = await searchAPI(q, category, page)

  return {
    props: {
      results,
      query: { q, category, page }
    }
  }
}

export default function Search({ results, query }) {
  return (
    <div>
      <h1>搜尋：{query.q}</h1>
      <p>分類：{query.category}</p>
      <p>第 {query.page} 頁</p>
      <ul>
        {results.map(item => (
          <li key={item.id}>{item.title}</li>
        ))}
      </ul>
    </div>
  )
}
```

## 8.4 返回值選項

### 8.4.1 props

```javascript
export async function getServerSideProps() {
  const data = await fetchData()

  return {
    props: {
      data,
      timestamp: Date.now()
    }
  }
}
```

### 8.4.2 notFound

返回 404 頁面。

```javascript
export async function getServerSideProps({ params }) {
  const product = await getProduct(params.id)

  if (!product) {
    return {
      notFound: true  // 顯示 404 頁面
    }
  }

  return {
    props: { product }
  }
}
```

### 8.4.3 redirect

重定向到其他頁面。

```javascript
export async function getServerSideProps({ req }) {
  const session = await getSession(req)

  // 未登入，重定向到登入頁
  if (!session) {
    return {
      redirect: {
        destination: '/login',
        permanent: false
      }
    }
  }

  return {
    props: { session }
  }
}
```

**永久重定向（301）**：
```javascript
return {
  redirect: {
    destination: '/new-url',
    permanent: true  // HTTP 301
  }
}
```

**臨時重定向（307）**：
```javascript
return {
  redirect: {
    destination: '/temporary-url',
    permanent: false  // HTTP 307
  }
}
```

## 8.5 認證與授權

### 8.5.1 檢查登入狀態

```javascript
import { getSession } from 'next-auth/react'

export async function getServerSideProps(context) {
  const session = await getSession(context)

  if (!session) {
    return {
      redirect: {
        destination: '/login',
        permanent: false
      }
    }
  }

  return {
    props: { session }
  }
}

export default function ProtectedPage({ session }) {
  return (
    <div>
      <h1>歡迎，{session.user.name}</h1>
    </div>
  )
}
```

### 8.5.2 基於角色的存取控制

```javascript
export async function getServerSideProps({ req }) {
  const session = await getSession({ req })

  // 未登入
  if (!session) {
    return {
      redirect: {
        destination: '/login',
        permanent: false
      }
    }
  }

  // 權限不足
  if (session.user.role !== 'admin') {
    return {
      redirect: {
        destination: '/unauthorized',
        permanent: false
      }
    }
  }

  // 獲取管理員資料
  const adminData = await getAdminData()

  return {
    props: {
      session,
      adminData
    }
  }
}

export default function AdminDashboard({ session, adminData }) {
  return (
    <div>
      <h1>管理員儀表板</h1>
      <p>歡迎，{session.user.name}</p>
    </div>
  )
}
```

### 8.5.3 使用 Cookies

```javascript
export async function getServerSideProps({ req, res }) {
  const token = req.cookies.authToken

  if (!token) {
    return {
      redirect: {
        destination: '/login',
        permanent: false
      }
    }
  }

  // 驗證 token
  const user = await verifyToken(token)

  if (!user) {
    // 清除無效的 cookie
    res.setHeader('Set-Cookie', 'authToken=; Max-Age=0; Path=/')

    return {
      redirect: {
        destination: '/login',
        permanent: false
      }
    }
  }

  return {
    props: { user }
  }
}
```

## 8.6 實戰範例

### 8.6.1 使用者儀表板

**pages/dashboard.js**：
```javascript
import { getSession } from 'next-auth/react'

export async function getServerSideProps({ req }) {
  const session = await getSession({ req })

  if (!session) {
    return {
      redirect: {
        destination: '/login',
        permanent: false
      }
    }
  }

  // 獲取使用者資料
  const [userStats, recentActivity] = await Promise.all([
    getUserStats(session.user.id),
    getRecentActivity(session.user.id)
  ])

  return {
    props: {
      session,
      userStats,
      recentActivity
    }
  }
}

export default function Dashboard({ session, userStats, recentActivity }) {
  return (
    <div>
      <h1>歡迎，{session.user.name}</h1>

      <div>
        <h2>統計資料</h2>
        <p>文章數：{userStats.postCount}</p>
        <p>追蹤者：{userStats.followers}</p>
      </div>

      <div>
        <h2>最近活動</h2>
        <ul>
          {recentActivity.map(activity => (
            <li key={activity.id}>{activity.description}</li>
          ))}
        </ul>
      </div>
    </div>
  )
}
```

### 8.6.2 搜尋結果頁

**pages/search.js**：
```javascript
export async function getServerSideProps({ query }) {
  const { q, page = '1', sort = 'relevance' } = query

  // 沒有搜尋關鍵字，重定向到首頁
  if (!q) {
    return {
      redirect: {
        destination: '/',
        permanent: false
      }
    }
  }

  const results = await searchAPI({
    query: q,
    page: parseInt(page),
    sort
  })

  return {
    props: {
      results,
      searchQuery: q,
      currentPage: parseInt(page),
      totalPages: results.totalPages
    }
  }
}

export default function Search({ results, searchQuery, currentPage, totalPages }) {
  const router = useRouter()

  const handlePageChange = (newPage) => {
    router.push({
      pathname: '/search',
      query: { q: searchQuery, page: newPage }
    })
  }

  return (
    <div>
      <h1>搜尋：{searchQuery}</h1>
      <p>找到 {results.total} 個結果</p>

      <ul>
        {results.items.map(item => (
          <li key={item.id}>
            <h3>{item.title}</h3>
            <p>{item.description}</p>
          </li>
        ))}
      </ul>

      {/* 分頁 */}
      <div>
        {currentPage > 1 && (
          <button onClick={() => handlePageChange(currentPage - 1)}>
            上一頁
          </button>
        )}
        <span>第 {currentPage} / {totalPages} 頁</span>
        {currentPage < totalPages && (
          <button onClick={() => handlePageChange(currentPage + 1)}>
            下一頁
          </button>
        )}
      </div>
    </div>
  )
}
```

### 8.6.3 產品詳情頁（即時庫存）

**pages/products/[id].js**：
```javascript
export async function getServerSideProps({ params, req }) {
  const { id } = params

  try {
    // 平行獲取產品資訊和庫存
    const [product, inventory] = await Promise.all([
      getProduct(id),
      getInventory(id)
    ])

    if (!product) {
      return { notFound: true }
    }

    // 記錄瀏覽
    await trackView(id, req.headers['user-agent'])

    return {
      props: {
        product,
        inventory,
        timestamp: new Date().toISOString()
      }
    }
  } catch (error) {
    console.error('Error fetching product:', error)

    return {
      props: {
        error: '無法載入產品資訊'
      }
    }
  }
}

export default function Product({ product, inventory, error }) {
  if (error) {
    return <div>錯誤：{error}</div>
  }

  return (
    <div>
      <h1>{product.name}</h1>
      <p>${product.price}</p>

      <div>
        {inventory.stock > 0 ? (
          <>
            <p>庫存：{inventory.stock} 件</p>
            <button>加入購物車</button>
          </>
        ) : (
          <p>目前缺貨</p>
        )}
      </div>

      <div dangerouslySetInnerHTML={{ __html: product.description }} />
    </div>
  )
}
```

## 8.7 快取策略

### 8.7.1 設定 Cache-Control Headers

```javascript
export async function getServerSideProps({ res }) {
  // 快取 10 秒，stale-while-revalidate 59 秒
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

**Cache-Control 選項**：
- `public`：允許 CDN 快取
- `private`：只允許瀏覽器快取
- `s-maxage=N`：CDN 快取 N 秒
- `max-age=N`：瀏覽器快取 N 秒
- `stale-while-revalidate=N`：過期後 N 秒內返回舊內容，同時重新驗證

### 8.7.2 不快取

```javascript
export async function getServerSideProps({ res }) {
  res.setHeader(
    'Cache-Control',
    'no-store, no-cache, must-revalidate, proxy-revalidate'
  )

  return {
    props: {}
  }
}
```

## 8.8 效能優化

### 8.8.1 平行資料獲取

```javascript
export async function getServerSideProps({ params }) {
  // ❌ 序列執行（慢）
  const user = await getUser(params.id)
  const posts = await getUserPosts(params.id)
  const followers = await getFollowers(params.id)

  // ✅ 平行執行（快）
  const [user, posts, followers] = await Promise.all([
    getUser(params.id),
    getUserPosts(params.id),
    getFollowers(params.id)
  ])

  return {
    props: { user, posts, followers }
  }
}
```

### 8.8.2 早期返回

```javascript
export async function getServerSideProps({ params }) {
  const user = await getUser(params.id)

  // 如果使用者不存在，立即返回，不繼續獲取其他資料
  if (!user) {
    return { notFound: true }
  }

  const posts = await getUserPosts(params.id)

  return {
    props: { user, posts }
  }
}
```

### 8.8.3 限制資料大小

```javascript
export async function getServerSideProps() {
  const allPosts = await getAllPosts()

  // 只返回前 10 筆，並移除不必要的欄位
  const posts = allPosts.slice(0, 10).map(post => ({
    id: post.id,
    title: post.title,
    excerpt: post.excerpt,
    date: post.date
    // 不包含 content（太大）
  }))

  return {
    props: { posts }
  }
}
```

## 8.9 錯誤處理

### 8.9.1 Try-Catch

```javascript
export async function getServerSideProps({ params }) {
  try {
    const res = await fetch(`https://api.example.com/posts/${params.id}`)

    if (!res.ok) {
      if (res.status === 404) {
        return { notFound: true }
      }
      throw new Error(`API error: ${res.status}`)
    }

    const post = await res.json()

    return {
      props: { post }
    }
  } catch (error) {
    console.error('Error in getServerSideProps:', error)

    return {
      props: {
        error: '無法載入資料，請稍後再試'
      }
    }
  }
}

export default function Post({ post, error }) {
  if (error) {
    return <div>錯誤：{error}</div>
  }

  return <h1>{post.title}</h1>
}
```

### 8.9.2 Timeout 處理

```javascript
async function fetchWithTimeout(url, timeout = 5000) {
  const controller = new AbortController()
  const id = setTimeout(() => controller.abort(), timeout)

  try {
    const response = await fetch(url, {
      signal: controller.signal
    })
    clearTimeout(id)
    return response
  } catch (error) {
    clearTimeout(id)
    throw error
  }
}

export async function getServerSideProps() {
  try {
    const res = await fetchWithTimeout('https://api.example.com/data', 3000)
    const data = await res.json()

    return { props: { data } }
  } catch (error) {
    if (error.name === 'AbortError') {
      return {
        props: {
          error: '請求逾時，請重試'
        }
      }
    }
    throw error
  }
}
```

## 8.10 何時使用 getServerSideProps？

### ✅ 適合的場景

- 需要即時資料（股票價格、庫存）
- 使用者特定內容（儀表板、個人資料）
- 依賴請求資訊（cookies、headers）
- SEO 重要且資料經常變化
- 需要認證/授權

### ❌ 不適合的場景

- 資料不常變化（使用 `getStaticProps` + ISR）
- 不需要 SEO（使用客戶端獲取）
- 對速度要求極高（使用 `getStaticProps`）
- 公開的靜態內容

### 🤔 選擇指南

```
需要 SEO？
├─ 是 → 資料經常變化？
│        ├─ 是 → getServerSideProps
│        └─ 否 → getStaticProps + ISR
└─ 否 → 客戶端獲取（useEffect, SWR）
```

## 8.11 與其他方法對比

### getServerSideProps vs getStaticProps

```javascript
// getStaticProps：建置時執行一次
export async function getStaticProps() {
  const posts = await getPosts()
  return {
    props: { posts },
    revalidate: 60  // ISR
  }
}

// getServerSideProps：每次請求都執行
export async function getServerSideProps() {
  const posts = await getPosts()
  return {
    props: { posts }
  }
}
```

### getServerSideProps vs 客戶端獲取

```javascript
// getServerSideProps：伺服器端，SEO 友善
export async function getServerSideProps() {
  const data = await fetchData()
  return { props: { data } }
}

// 客戶端獲取：瀏覽器端，不利 SEO
export default function Page() {
  const [data, setData] = useState(null)

  useEffect(() => {
    fetchData().then(setData)
  }, [])

  return <div>{data}</div>
}
```

## 8.12 本章小結

- `getServerSideProps` 在每次請求時執行
- 適合需要即時資料或認證的頁面
- 可以存取請求資訊（cookies、headers、query）
- 支援重定向和返回 404
- 可以設定 Cache-Control headers 優化效能
- 相比 `getStaticProps` 速度較慢，但資料更即時

## 總結：資料獲取方法選擇

| 方法 | 執行時機 | 快取 | 使用場景 |
|------|---------|------|---------|
| `getStaticProps` | 建置時 | ✅ | 部落格、文檔 |
| `getStaticProps` + ISR | 建置時 + 定期 | ✅ | 新聞網站 |
| `getServerSideProps` | 每次請求 | ⚠️ | 儀表板、認證頁面 |
| 客戶端獲取 | 客戶端 | ❌ | 不需 SEO 的動態內容 |

## 練習題

1. 創建一個需要登入的儀表板頁面
2. 實作一個搜尋結果頁，支援分頁和排序
3. 創建產品詳情頁，顯示即時庫存
4. 實作基於角色的存取控制
5. 設定適當的 Cache-Control headers
6. 處理 API 請求失敗的情況
7. 比較 `getServerSideProps` 和 `getStaticProps` 的效能差異
