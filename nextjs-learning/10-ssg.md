# 第 10 章：SSG (Static Site Generation) 深入解析

## 概述

SSG（靜態網站生成）是 Next.js 的預設渲染模式，也是效能最佳的選擇。與 Vue3 的傳統 SPA 不同，SSG 在構建時就生成完整的 HTML 檔案，不需要伺服器運行時處理。

## Vue3 vs Next.js SSG

### Vue3 的傳統做法

```javascript
// Vue3 SPA
// 1. 伺服器返回空的 HTML
// 2. 下載 JavaScript bundle
// 3. 執行 Vue 應用
// 4. 呼叫 API 獲取資料
// 5. 渲染頁面

// App.vue
export default {
  data() {
    return {
      posts: []
    }
  },
  async mounted() {
    // 在用戶端獲取資料
    const response = await fetch('/api/posts')
    this.posts = await response.json()
  }
}
```

**問題：**
- 首次載入慢（白屏時間長）
- SEO 不友善
- 需要等待 JavaScript 執行

### Next.js SSG 的做法

```javascript
// pages/posts.js
// 在構建時就生成完整的 HTML

export async function getStaticProps() {
  // 構建時執行，不在用戶端執行
  const response = await fetch('https://api.example.com/posts')
  const posts = await response.json()

  return {
    props: { posts }
  }
}

export default function PostsPage({ posts }) {
  // HTML 已經包含完整內容
  return (
    <div>
      <h1>文章列表</h1>
      {posts.map(post => (
        <article key={post.id}>
          <h2>{post.title}</h2>
          <p>{post.excerpt}</p>
        </article>
      ))}
    </div>
  )
}
```

**優點：**
- 瞬間載入（HTML 已經生成）
- 完美的 SEO
- 可以部署到 CDN
- 不需要伺服器

## SSG 的運作流程

### 構建時（Build Time）

```bash
# 執行構建
npm run build

# Next.js 會：
# 1. 執行所有頁面的 getStaticProps
# 2. 獲取資料
# 3. 渲染 React 組件為 HTML
# 4. 生成靜態 HTML 檔案到 .next 目錄
# 5. 準備好可以部署的檔案
```

### 運行時（Runtime）

```
用戶請求 /posts
  ↓
CDN 直接返回已生成的 HTML
  ↓
瀏覽器立即顯示內容
  ↓
React hydration（附加互動功能）
  ↓
頁面完全可互動
```

## getStaticProps 完整解析

### 基本用法

```javascript
// pages/index.js
export async function getStaticProps() {
  // 這段程式碼只在伺服器端執行
  // 永遠不會出現在用戶端的 bundle 中

  const response = await fetch('https://api.example.com/data')
  const data = await response.json()

  return {
    props: {
      data // 會作為 props 傳給組件
    }
  }
}

export default function HomePage({ data }) {
  return <div>{JSON.stringify(data)}</div>
}
```

### 完整的返回選項

```javascript
export async function getStaticProps(context) {
  return {
    // 必需：傳遞給組件的 props
    props: {
      data: {}
    },

    // 可選：重新驗證時間（秒）
    revalidate: 60, // ISR：每 60 秒重新生成一次

    // 可選：返回 404 頁面
    notFound: true,

    // 可選：重定向
    redirect: {
      destination: '/new-page',
      permanent: false
    }
  }
}
```

### context 參數詳解

```javascript
export async function getStaticProps(context) {
  console.log({
    params: context.params,       // 動態路由參數（如果有）
    preview: context.preview,     // 預覽模式
    previewData: context.previewData, // 預覽資料
    locale: context.locale,       // 當前語言（如果啟用 i18n）
    locales: context.locales,     // 所有可用語言
    defaultLocale: context.defaultLocale // 預設語言
  })

  return { props: {} }
}
```

## 實際應用案例

### 案例 1：部落格首頁（文章列表）

```javascript
// pages/blog/index.js
import Link from 'next/link'
import { getAllPosts } from '../../lib/api'

export async function getStaticProps() {
  // 從 Markdown 檔案、資料庫或 CMS 獲取文章
  const posts = await getAllPosts()

  return {
    props: {
      posts
    },
    revalidate: 3600 // 每小時重新生成一次
  }
}

export default function BlogIndex({ posts }) {
  return (
    <div className="container">
      <h1>部落格文章</h1>
      <div className="posts-grid">
        {posts.map(post => (
          <article key={post.slug} className="post-card">
            <Link href={`/blog/${post.slug}`}>
              <img src={post.coverImage} alt={post.title} />
              <h2>{post.title}</h2>
              <p>{post.excerpt}</p>
              <time>{new Date(post.date).toLocaleDateString('zh-TW')}</time>
            </Link>
          </article>
        ))}
      </div>
    </div>
  )
}
```

### 案例 2：從 CMS 獲取資料

```javascript
// pages/index.js
import { fetchFromCMS } from '../lib/cms'

export async function getStaticProps() {
  try {
    // 從 Headless CMS（如 Contentful、Strapi）獲取資料
    const [hero, features, testimonials] = await Promise.all([
      fetchFromCMS('hero'),
      fetchFromCMS('features'),
      fetchFromCMS('testimonials')
    ])

    return {
      props: {
        hero,
        features,
        testimonials
      },
      revalidate: 300 // 每 5 分鐘重新驗證
    }
  } catch (error) {
    console.error('獲取 CMS 資料失敗：', error)

    // 如果獲取失敗，返回空資料或預設值
    return {
      props: {
        hero: null,
        features: [],
        testimonials: []
      },
      revalidate: 60 // 失敗時更頻繁地重試
    }
  }
}

export default function HomePage({ hero, features, testimonials }) {
  if (!hero) {
    return <div>資料載入中...</div>
  }

  return (
    <div>
      <section className="hero">
        <h1>{hero.title}</h1>
        <p>{hero.subtitle}</p>
      </section>

      <section className="features">
        {features.map(feature => (
          <div key={feature.id}>
            <h3>{feature.title}</h3>
            <p>{feature.description}</p>
          </div>
        ))}
      </section>

      <section className="testimonials">
        {testimonials.map(item => (
          <blockquote key={item.id}>
            <p>{item.quote}</p>
            <cite>— {item.author}</cite>
          </blockquote>
        ))}
      </section>
    </div>
  )
}
```

### 案例 3：電商產品列表

```javascript
// pages/products/index.js
export async function getStaticProps() {
  const response = await fetch('https://api.shop.com/products?limit=50')
  const products = await response.json()

  // 獲取分類資訊
  const categoriesResponse = await fetch('https://api.shop.com/categories')
  const categories = await categoriesResponse.json()

  return {
    props: {
      products,
      categories
    },
    revalidate: 60 // 價格可能變動，每分鐘更新
  }
}

export default function ProductsPage({ products, categories }) {
  const [selectedCategory, setSelectedCategory] = useState('all')

  const filteredProducts = selectedCategory === 'all'
    ? products
    : products.filter(p => p.categoryId === selectedCategory)

  return (
    <div>
      <aside>
        <h2>分類</h2>
        <button onClick={() => setSelectedCategory('all')}>全部</button>
        {categories.map(cat => (
          <button
            key={cat.id}
            onClick={() => setSelectedCategory(cat.id)}
          >
            {cat.name}
          </button>
        ))}
      </aside>

      <main>
        <div className="products-grid">
          {filteredProducts.map(product => (
            <div key={product.id} className="product-card">
              <img src={product.image} alt={product.name} />
              <h3>{product.name}</h3>
              <p>NT$ {product.price}</p>
              <button>加入購物車</button>
            </div>
          ))}
        </div>
      </main>
    </div>
  )
}
```

### 案例 4：從多個來源聚合資料

```javascript
// pages/dashboard.js
export async function getStaticProps() {
  // 並行獲取多個資料來源
  const [stats, recentOrders, topProducts] = await Promise.all([
    fetch('https://api.example.com/stats').then(r => r.json()),
    fetch('https://api.example.com/orders/recent').then(r => r.json()),
    fetch('https://api.example.com/products/top').then(r => r.json())
  ])

  return {
    props: {
      stats,
      recentOrders,
      topProducts,
      generatedAt: new Date().toISOString()
    },
    revalidate: 300 // 每 5 分鐘更新
  }
}

export default function Dashboard({ stats, recentOrders, topProducts, generatedAt }) {
  return (
    <div>
      <header>
        <h1>儀表板</h1>
        <p>最後更新：{new Date(generatedAt).toLocaleString('zh-TW')}</p>
      </header>

      <div className="stats-grid">
        <div className="stat-card">
          <h3>總銷售額</h3>
          <p>NT$ {stats.totalSales.toLocaleString()}</p>
        </div>
        <div className="stat-card">
          <h3>訂單數量</h3>
          <p>{stats.orderCount}</p>
        </div>
        <div className="stat-card">
          <h3>新客戶</h3>
          <p>{stats.newCustomers}</p>
        </div>
      </div>

      <section>
        <h2>最近訂單</h2>
        <table>
          <thead>
            <tr>
              <th>訂單編號</th>
              <th>客戶</th>
              <th>金額</th>
              <th>狀態</th>
            </tr>
          </thead>
          <tbody>
            {recentOrders.map(order => (
              <tr key={order.id}>
                <td>{order.id}</td>
                <td>{order.customerName}</td>
                <td>NT$ {order.total}</td>
                <td>{order.status}</td>
              </tr>
            ))}
          </tbody>
        </table>
      </section>

      <section>
        <h2>熱銷商品</h2>
        <ul>
          {topProducts.map(product => (
            <li key={product.id}>
              {product.name} - 銷量：{product.salesCount}
            </li>
          ))}
        </ul>
      </section>
    </div>
  )
}
```

## notFound 和 redirect 的使用

### 返回 404

```javascript
export async function getStaticProps({ params }) {
  const post = await fetchPost(params.id)

  // 如果資料不存在，顯示 404 頁面
  if (!post) {
    return {
      notFound: true
    }
  }

  return {
    props: { post }
  }
}
```

### 重定向

```javascript
export async function getStaticProps() {
  const maintenanceMode = await checkMaintenanceMode()

  // 如果網站在維護中，重定向到維護頁面
  if (maintenanceMode) {
    return {
      redirect: {
        destination: '/maintenance',
        permanent: false // 臨時重定向（307）
      }
    }
  }

  return {
    props: { data: {} }
  }
}
```

```javascript
// 永久重定向範例
export async function getStaticProps() {
  return {
    redirect: {
      destination: '/new-home',
      permanent: true // 永久重定向（308），對 SEO 友善
    }
  }
}
```

## 從檔案系統讀取資料

### 讀取 Markdown 檔案

```javascript
// lib/posts.js
import fs from 'fs'
import path from 'path'
import matter from 'gray-matter'

const postsDirectory = path.join(process.cwd(), 'posts')

export function getAllPosts() {
  const fileNames = fs.readdirSync(postsDirectory)

  const posts = fileNames.map(fileName => {
    const slug = fileName.replace(/\.md$/, '')
    const fullPath = path.join(postsDirectory, fileName)
    const fileContents = fs.readFileSync(fullPath, 'utf8')

    // 解析 Markdown frontmatter
    const { data, content } = matter(fileContents)

    return {
      slug,
      title: data.title,
      date: data.date,
      excerpt: data.excerpt,
      content
    }
  })

  // 按日期排序
  return posts.sort((a, b) => (a.date > b.date ? -1 : 1))
}
```

```javascript
// pages/blog/index.js
import { getAllPosts } from '../../lib/posts'

export async function getStaticProps() {
  const posts = getAllPosts()

  return {
    props: { posts }
  }
}

export default function BlogIndex({ posts }) {
  return (
    <div>
      {posts.map(post => (
        <article key={post.slug}>
          <h2>{post.title}</h2>
          <time>{post.date}</time>
          <p>{post.excerpt}</p>
        </article>
      ))}
    </div>
  )
}
```

## SSG 的限制與解決方案

### 限制 1：資料不是即時的

```javascript
// ❌ 問題：使用者資料需要即時更新
export async function getStaticProps() {
  const user = await fetchUser() // 在構建時獲取
  // 用戶資料可能已經過期
  return { props: { user } }
}

// ✅ 解決方案 1：使用 ISR
export async function getStaticProps() {
  const user = await fetchUser()
  return {
    props: { user },
    revalidate: 10 // 每 10 秒重新驗證
  }
}

// ✅ 解決方案 2：客戶端獲取即時資料
export default function UserProfile({ initialUser }) {
  const [user, setUser] = useState(initialUser)

  useEffect(() => {
    // 在客戶端獲取最新資料
    fetch('/api/user')
      .then(r => r.json())
      .then(data => setUser(data))
  }, [])

  return <div>{user.name}</div>
}
```

### 限制 2：無法存取請求資訊

```javascript
// ❌ getStaticProps 無法存取：
// - cookies
// - headers
// - query parameters（除了動態路由參數）

// ✅ 如果需要這些資訊，使用 getServerSideProps
export async function getServerSideProps({ req, query }) {
  const userAgent = req.headers['user-agent']
  const searchTerm = query.q

  return { props: { userAgent, searchTerm } }
}
```

### 限制 3：構建時間長（頁面多時）

```javascript
// ❌ 生成 10000 個頁面會很慢
export async function getStaticPaths() {
  const products = await fetchAllProducts() // 10000+
  return {
    paths: products.map(p => ({ params: { id: p.id } })),
    fallback: false
  }
}

// ✅ 只預先生成重要頁面
export async function getStaticPaths() {
  const popularProducts = await fetchPopularProducts(100)
  return {
    paths: popularProducts.map(p => ({ params: { id: p.id } })),
    fallback: 'blocking' // 其他頁面按需生成
  }
}
```

## SSG 的最佳實踐

### 1. 使用環境變數分離開發和生產資料

```javascript
// .env.local
API_URL=http://localhost:3000/api

// .env.production
API_URL=https://api.production.com
```

```javascript
// pages/index.js
export async function getStaticProps() {
  const apiUrl = process.env.API_URL
  const response = await fetch(`${apiUrl}/posts`)
  const posts = await response.json()

  return {
    props: { posts }
  }
}
```

### 2. 錯誤處理

```javascript
export async function getStaticProps() {
  try {
    const response = await fetch('https://api.example.com/data')

    if (!response.ok) {
      throw new Error(`API 返回 ${response.status}`)
    }

    const data = await response.json()

    return {
      props: { data }
    }
  } catch (error) {
    console.error('獲取資料失敗：', error)

    // 返回預設資料或空資料
    return {
      props: {
        data: null,
        error: error.message
      },
      revalidate: 60 // 失敗時更頻繁重試
    }
  }
}
```

### 3. 資料快取

```javascript
// lib/cache.js
const cache = new Map()

export async function fetchWithCache(url, ttl = 60000) {
  const cached = cache.get(url)

  if (cached && Date.now() - cached.timestamp < ttl) {
    return cached.data
  }

  const response = await fetch(url)
  const data = await response.json()

  cache.set(url, {
    data,
    timestamp: Date.now()
  })

  return data
}
```

### 4. 使用 TypeScript

```typescript
// types/index.ts
export interface Post {
  id: number
  title: string
  content: string
  date: string
}

// pages/posts.tsx
import { GetStaticProps } from 'next'
import { Post } from '../types'

interface PostsPageProps {
  posts: Post[]
}

export const getStaticProps: GetStaticProps<PostsPageProps> = async () => {
  const response = await fetch('https://api.example.com/posts')
  const posts: Post[] = await response.json()

  return {
    props: { posts }
  }
}

export default function PostsPage({ posts }: PostsPageProps) {
  return (
    <div>
      {posts.map(post => (
        <article key={post.id}>
          <h2>{post.title}</h2>
        </article>
      ))}
    </div>
  )
}
```

## SSG vs 其他渲染模式比較

| 特性 | SSG | SSR | CSR (Vue3 SPA) |
|------|-----|-----|----------------|
| 執行時機 | 構建時 | 每次請求 | 用戶端 |
| 速度 | 最快 | 中等 | 慢（首次） |
| SEO | 完美 | 完美 | 需要額外配置 |
| 即時性 | 差（可用 ISR 改善） | 完美 | 完美 |
| 伺服器需求 | 不需要 | 需要 | 不需要 |
| 成本 | 最低（CDN） | 高 | 低 |
| 適用場景 | 內容網站、部落格 | 個人化內容 | 互動式應用 |

## 何時使用 SSG

### ✅ 適合使用 SSG 的場景

1. **部落格和文章網站**
   - 內容不常變動
   - SEO 很重要
   - 需要快速載入

2. **產品展示網站**
   - 產品資訊相對穩定
   - 可以搭配 ISR 定期更新

3. **文件網站**
   - 內容來自 Markdown 或 CMS
   - 需要搜尋引擎索引

4. **電商產品列表**
   - 搭配 ISR 更新價格和庫存
   - 產品頁面可以預先生成

5. **行銷頁面（Landing Pages）**
   - 效能至關重要
   - 內容由行銷團隊管理

### ❌ 不適合使用 SSG 的場景

1. **高度個人化的內容**
   - 每個用戶看到的內容不同
   - 需要根據用戶權限顯示

2. **即時資料**
   - 股票價格
   - 即時聊天
   - 即時通知

3. **需要請求資訊的頁面**
   - 依賴 cookies
   - 依賴 query parameters
   - 需要 IP 地址等資訊

## 小結

SSG 是 Next.js 的核心優勢之一，為 Vue3 開發者提供了一個全新的思考方式：不是在用戶請求時處理，而是提前準備好一切。這種模式在效能、SEO 和成本上都有巨大優勢，特別適合內容型網站。下一章我們將學習 SSR，看看如何處理需要即時資料的場景。
