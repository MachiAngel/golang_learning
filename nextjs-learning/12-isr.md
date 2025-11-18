# 第 12 章：ISR (Incremental Static Regeneration)

## 概述

ISR（增量靜態再生）是 Next.js 獨有的創新功能，結合了 SSG 的效能優勢和 SSR 的資料新鮮度。它讓你可以在不重新構建整個網站的情況下，更新靜態頁面的內容。對於 Vue3 開發者來說，這是一個全新的概念，Vue 生態系統中沒有直接對應的功能。

## ISR 的核心概念

### 傳統 SSG 的問題

```javascript
// 傳統 SSG
export async function getStaticProps() {
  const posts = await fetchPosts()

  return {
    props: { posts }
  }
}
```

**問題：**
- 新文章發布後，需要重新構建整個網站
- 內容更新後，舊的靜態頁面仍然存在
- 無法處理頻繁變動的資料（如價格、庫存）

### ISR 的解決方案

```javascript
// ISR
export async function getStaticProps() {
  const posts = await fetchPosts()

  return {
    props: { posts },
    revalidate: 60 // 每 60 秒重新驗證一次
  }
}
```

**優點：**
- 保持 SSG 的效能優勢
- 自動更新內容，無需重新構建
- 只重新生成需要更新的頁面

## ISR 的運作流程

### 首次請求

```
用戶請求頁面
  ↓
頁面不存在（或已過期）
  ↓
伺服器執行 getStaticProps
  ↓
生成 HTML
  ↓
儲存到快取
  ↓
返回給用戶
```

### 後續請求（在 revalidate 時間內）

```
用戶請求頁面
  ↓
返回快取的 HTML（瞬間載入）
```

### 後續請求（超過 revalidate 時間）

```
用戶 A 請求頁面
  ↓
返回舊的快取 HTML（仍然很快）
  ↓
同時在背景觸發重新生成
  ↓
執行 getStaticProps
  ↓
生成新的 HTML
  ↓
更新快取
  ↓
用戶 B 的請求會得到新的 HTML
```

## 基本使用

### 簡單的 ISR 範例

```javascript
// pages/posts/index.js
export async function getStaticProps() {
  const response = await fetch('https://api.example.com/posts')
  const posts = await response.json()

  return {
    props: {
      posts,
      generatedAt: new Date().toISOString()
    },
    revalidate: 10 // 每 10 秒重新驗證
  }
}

export default function PostsPage({ posts, generatedAt }) {
  return (
    <div>
      <h1>文章列表</h1>
      <p className="timestamp">
        頁面生成時間：{new Date(generatedAt).toLocaleString('zh-TW')}
      </p>
      <div>
        {posts.map(post => (
          <article key={post.id}>
            <h2>{post.title}</h2>
            <p>{post.excerpt}</p>
          </article>
        ))}
      </div>
    </div>
  )
}
```

### 動態路由的 ISR

```javascript
// pages/posts/[id].js
export async function getStaticPaths() {
  // 只預先生成部分頁面
  const response = await fetch('https://api.example.com/posts/popular')
  const popularPosts = await response.json()

  const paths = popularPosts.map(post => ({
    params: { id: post.id.toString() }
  }))

  return {
    paths,
    fallback: 'blocking' // 其他頁面會在請求時生成
  }
}

export async function getStaticProps({ params }) {
  const response = await fetch(`https://api.example.com/posts/${params.id}`)
  const post = await response.json()

  if (!post) {
    return {
      notFound: true,
      revalidate: 60 // 即使是 404，也會重新驗證
    }
  }

  return {
    props: {
      post,
      generatedAt: new Date().toISOString()
    },
    revalidate: 60 // 每分鐘重新驗證
  }
}

export default function PostPage({ post, generatedAt }) {
  return (
    <article>
      <h1>{post.title}</h1>
      <p className="meta">
        發布時間：{new Date(post.publishedAt).toLocaleDateString('zh-TW')}
        <br />
        頁面更新：{new Date(generatedAt).toLocaleString('zh-TW')}
      </p>
      <div dangerouslySetInnerHTML={{ __html: post.content }} />
    </article>
  )
}
```

## 實際應用案例

### 案例 1：新聞網站

```javascript
// pages/news/[slug].js
export async function getStaticPaths() {
  // 只預先生成今日的新聞
  const today = new Date().toISOString().split('T')[0]
  const response = await fetch(`https://api.news.com/articles?date=${today}`)
  const articles = await response.json()

  const paths = articles.map(article => ({
    params: { slug: article.slug }
  }))

  return {
    paths,
    fallback: 'blocking'
  }
}

export async function getStaticProps({ params }) {
  const response = await fetch(`https://api.news.com/articles/${params.slug}`)

  if (!response.ok) {
    return {
      notFound: true,
      revalidate: 60
    }
  }

  const article = await response.json()

  // 獲取相關文章
  const relatedResponse = await fetch(
    `https://api.news.com/articles/${params.slug}/related`
  )
  const relatedArticles = await relatedResponse.json()

  return {
    props: {
      article,
      relatedArticles,
      generatedAt: new Date().toISOString()
    },
    revalidate: 30 // 新聞每 30 秒更新
  }
}

export default function NewsArticle({ article, relatedArticles, generatedAt }) {
  return (
    <div>
      <article>
        <header>
          <h1>{article.title}</h1>
          <p className="byline">
            {article.author} - {new Date(article.publishedAt).toLocaleString('zh-TW')}
          </p>
          {article.updatedAt && (
            <p className="update-notice">
              更新於：{new Date(article.updatedAt).toLocaleString('zh-TW')}
            </p>
          )}
        </header>

        <img src={article.coverImage} alt={article.title} />

        <div dangerouslySetInnerHTML={{ __html: article.content }} />

        <footer>
          <p className="page-gen">
            頁面生成：{new Date(generatedAt).toLocaleString('zh-TW')}
          </p>
        </footer>
      </article>

      <aside>
        <h2>相關新聞</h2>
        {relatedArticles.map(related => (
          <a key={related.slug} href={`/news/${related.slug}`}>
            <h3>{related.title}</h3>
          </a>
        ))}
      </aside>
    </div>
  )
}
```

### 案例 2：電商產品頁面

```javascript
// pages/products/[id].js
export async function getStaticPaths() {
  // 只預先生成熱門產品
  const response = await fetch('https://api.shop.com/products/bestsellers?limit=100')
  const products = await response.json()

  return {
    paths: products.map(p => ({ params: { id: p.id.toString() } })),
    fallback: 'blocking'
  }
}

export async function getStaticProps({ params }) {
  try {
    const [productRes, reviewsRes] = await Promise.all([
      fetch(`https://api.shop.com/products/${params.id}`),
      fetch(`https://api.shop.com/products/${params.id}/reviews?limit=5`)
    ])

    if (!productRes.ok) {
      return { notFound: true, revalidate: 60 }
    }

    const product = await productRes.json()
    const reviews = await reviewsRes.json()

    return {
      props: {
        product,
        reviews,
        lastUpdated: new Date().toISOString()
      },
      revalidate: 60 // 價格和庫存每分鐘更新
    }
  } catch (error) {
    console.error('獲取產品資料失敗：', error)
    return { notFound: true, revalidate: 10 }
  }
}

export default function ProductPage({ product, reviews, lastUpdated }) {
  return (
    <div className="product-page">
      <div className="product-info">
        <img src={product.image} alt={product.name} />
        <div>
          <h1>{product.name}</h1>
          <p className="price">NT$ {product.price.toLocaleString()}</p>

          {product.stock > 0 ? (
            <p className="stock in-stock">庫存：{product.stock} 件</p>
          ) : (
            <p className="stock out-of-stock">已售完</p>
          )}

          <button disabled={product.stock === 0}>
            {product.stock > 0 ? '加入購物車' : '已售完'}
          </button>

          <p className="last-update">
            價格更新：{new Date(lastUpdated).toLocaleString('zh-TW')}
          </p>
        </div>
      </div>

      <div className="product-description">
        <h2>商品描述</h2>
        <div dangerouslySetInnerHTML={{ __html: product.description }} />
      </div>

      <div className="reviews">
        <h2>客戶評價</h2>
        {reviews.map(review => (
          <div key={review.id} className="review">
            <div className="rating">★ {review.rating}/5</div>
            <p>{review.comment}</p>
            <cite>— {review.author}</cite>
          </div>
        ))}
      </div>
    </div>
  )
}
```

### 案例 3：體育比分（頻繁更新）

```javascript
// pages/matches/[id].js
export async function getStaticPaths() {
  // 只預先生成今日比賽
  const response = await fetch('https://api.sports.com/matches/today')
  const matches = await response.json()

  return {
    paths: matches.map(m => ({ params: { id: m.id.toString() } })),
    fallback: 'blocking'
  }
}

export async function getStaticProps({ params }) {
  const response = await fetch(`https://api.sports.com/matches/${params.id}`)

  if (!response.ok) {
    return { notFound: true, revalidate: 10 }
  }

  const match = await response.json()

  return {
    props: {
      match,
      updatedAt: new Date().toISOString()
    },
    revalidate: 10 // 比賽進行中每 10 秒更新
  }
}

export default function MatchPage({ match, updatedAt }) {
  // 如果比賽已結束，可以在客戶端停止重新載入
  const isLive = match.status === 'live'

  useEffect(() => {
    if (isLive) {
      // 比賽進行中，每 10 秒重新載入頁面
      const interval = setInterval(() => {
        window.location.reload()
      }, 10000)

      return () => clearInterval(interval)
    }
  }, [isLive])

  return (
    <div className="match-page">
      <div className="match-header">
        <h1>{match.homeTeam} vs {match.awayTeam}</h1>
        <span className={`status ${match.status}`}>
          {match.status === 'live' ? '直播中' : '已結束'}
        </span>
      </div>

      <div className="scoreboard">
        <div className="team">
          <img src={match.homeTeam.logo} alt={match.homeTeam.name} />
          <h2>{match.homeTeam.name}</h2>
          <span className="score">{match.homeScore}</span>
        </div>

        <div className="separator">-</div>

        <div className="team">
          <img src={match.awayTeam.logo} alt={match.awayTeam.name} />
          <h2>{match.awayTeam.name}</h2>
          <span className="score">{match.awayScore}</span>
        </div>
      </div>

      <div className="match-info">
        <p>比賽時間：{new Date(match.startTime).toLocaleString('zh-TW')}</p>
        <p>最後更新：{new Date(updatedAt).toLocaleString('zh-TW')}</p>
        {isLive && <p className="live-notice">⚡ 頁面每 10 秒自動更新</p>}
      </div>

      <div className="timeline">
        <h2>比賽紀錄</h2>
        {match.events.map(event => (
          <div key={event.id} className="event">
            <span className="time">{event.minute}'</span>
            <span className="description">{event.description}</span>
          </div>
        ))}
      </div>
    </div>
  )
}
```

### 案例 4：部落格與評論系統

```javascript
// pages/blog/[slug].js
export async function getStaticPaths() {
  const posts = await getAllPostSlugs()

  return {
    paths: posts.map(slug => ({ params: { slug } })),
    fallback: 'blocking'
  }
}

export async function getStaticProps({ params }) {
  const post = await getPostBySlug(params.slug)

  if (!post) {
    return { notFound: true, revalidate: 60 }
  }

  // 獲取最新評論
  const comments = await fetch(
    `https://api.example.com/posts/${post.id}/comments?limit=10`
  ).then(r => r.json())

  return {
    props: {
      post,
      comments,
      commentCount: comments.length,
      generatedAt: new Date().toISOString()
    },
    revalidate: 300 // 評論每 5 分鐘更新
  }
}

export default function BlogPost({ post, comments, commentCount, generatedAt }) {
  const [clientComments, setClientComments] = useState(comments)

  // 提交新評論後重新獲取
  const handleCommentSubmit = async (comment) => {
    await fetch('/api/comments', {
      method: 'POST',
      body: JSON.stringify(comment)
    })

    // 重新獲取評論
    const updated = await fetch(`/api/posts/${post.id}/comments`).then(r => r.json())
    setClientComments(updated)
  }

  return (
    <article>
      <h1>{post.title}</h1>
      <div dangerouslySetInnerHTML={{ __html: post.content }} />

      <section className="comments">
        <h2>評論 ({commentCount})</h2>

        <CommentForm onSubmit={handleCommentSubmit} />

        <div className="comment-list">
          {clientComments.map(comment => (
            <div key={comment.id} className="comment">
              <strong>{comment.author}</strong>
              <p>{comment.content}</p>
              <time>{new Date(comment.createdAt).toLocaleString('zh-TW')}</time>
            </div>
          ))}
        </div>

        <p className="meta">
          頁面生成：{new Date(generatedAt).toLocaleString('zh-TW')}
        </p>
      </section>
    </article>
  )
}
```

## On-Demand Revalidation（按需重新驗證）

### 使用場景

當內容更新時（如 CMS 發布新文章），不想等待 `revalidate` 時間，想立即更新頁面。

### 實作方式

```javascript
// pages/api/revalidate.js
export default async function handler(req, res) {
  // 驗證請求來源（防止未授權的重新驗證）
  const secret = req.query.secret

  if (secret !== process.env.REVALIDATE_SECRET) {
    return res.status(401).json({ message: '未授權' })
  }

  try {
    const path = req.query.path

    // 重新驗證指定路徑
    await res.revalidate(path)

    return res.json({
      revalidated: true,
      path
    })
  } catch (error) {
    return res.status(500).json({
      message: '重新驗證失敗',
      error: error.message
    })
  }
}
```

### 從 CMS Webhook 觸發

```javascript
// CMS (Contentful/Strapi) Webhook 設定
// Webhook URL: https://your-site.com/api/revalidate?secret=YOUR_SECRET&path=/blog/post-slug

// pages/api/revalidate.js
export default async function handler(req, res) {
  // 驗證 secret
  if (req.query.secret !== process.env.REVALIDATE_SECRET) {
    return res.status(401).json({ message: '未授權' })
  }

  try {
    // 從 webhook payload 獲取資訊
    const { slug, type } = req.body

    // 根據內容類型重新驗證不同頁面
    if (type === 'post') {
      await res.revalidate(`/blog/${slug}`)
      await res.revalidate('/blog') // 也更新列表頁
    } else if (type === 'product') {
      await res.revalidate(`/products/${slug}`)
      await res.revalidate('/products')
    }

    return res.json({ revalidated: true })
  } catch (error) {
    return res.status(500).json({ error: error.message })
  }
}
```

### 在管理後台手動觸發

```javascript
// components/AdminPanel.js
export default function AdminPanel() {
  const [revalidating, setRevalidating] = useState(false)
  const [message, setMessage] = useState('')

  const handleRevalidate = async (path) => {
    setRevalidating(true)
    setMessage('')

    try {
      const response = await fetch(
        `/api/revalidate?secret=${process.env.NEXT_PUBLIC_REVALIDATE_SECRET}&path=${path}`,
        { method: 'POST' }
      )

      const data = await response.json()

      if (data.revalidated) {
        setMessage(`✓ 頁面 ${path} 已重新生成`)
      } else {
        setMessage(`✗ 重新生成失敗`)
      }
    } catch (error) {
      setMessage(`✗ 錯誤：${error.message}`)
    } finally {
      setRevalidating(false)
    }
  }

  return (
    <div className="admin-panel">
      <h2>頁面管理</h2>

      <div>
        <button
          onClick={() => handleRevalidate('/blog/latest-post')}
          disabled={revalidating}
        >
          重新生成最新文章
        </button>

        <button
          onClick={() => handleRevalidate('/products/featured')}
          disabled={revalidating}
        >
          重新生成特色產品
        </button>
      </div>

      {message && <p className="message">{message}</p>}
    </div>
  )
}
```

## revalidate 時間的選擇策略

### 根據內容更新頻率

```javascript
// 極少變動的內容（公司介紹、政策頁面）
export async function getStaticProps() {
  return {
    props: { ... },
    revalidate: 86400 // 24 小時
  }
}

// 偶爾變動的內容（部落格文章）
export async function getStaticProps() {
  return {
    props: { ... },
    revalidate: 3600 // 1 小時
  }
}

// 經常變動的內容（新聞、評論）
export async function getStaticProps() {
  return {
    props: { ... },
    revalidate: 60 // 1 分鐘
  }
}

// 即時變動的內容（股價、比分）
export async function getStaticProps() {
  return {
    props: { ... },
    revalidate: 10 // 10 秒
  }
}
```

### 根據流量模式

```javascript
// 高流量頁面：較長的 revalidate 時間
export async function getStaticProps() {
  return {
    props: { ... },
    revalidate: 300 // 5 分鐘，減少伺服器負載
  }
}

// 低流量頁面：較短的 revalidate 時間
export async function getStaticProps() {
  return {
    props: { ... },
    revalidate: 30 // 30 秒，保持內容新鮮
  }
}
```

## ISR 的限制與注意事項

### 1. 第一個用戶看到的是舊內容

```
頁面過期後：
用戶 A 請求 → 得到舊內容（同時觸發重新生成）
用戶 B 請求 → 得到新內容
```

**解決方案：** 如果需要立即更新，使用 On-Demand Revalidation。

### 2. 無法保證即時性

```javascript
// ❌ 不適合：需要絕對即時的資料
export async function getStaticProps() {
  const stockPrice = await fetchStockPrice() // 股價必須即時
  return {
    props: { stockPrice },
    revalidate: 1 // 即使 1 秒也可能過時
  }
}

// ✅ 使用 SSR 或客戶端獲取
export async function getServerSideProps() {
  const stockPrice = await fetchStockPrice()
  return { props: { stockPrice } }
}
```

### 3. 錯誤處理

```javascript
export async function getStaticProps() {
  try {
    const data = await fetchData()
    return {
      props: { data },
      revalidate: 60
    }
  } catch (error) {
    console.error('獲取資料失敗：', error)

    // 返回舊資料或預設值，但縮短 revalidate 時間
    return {
      props: {
        data: null,
        error: true
      },
      revalidate: 10 // 失敗時更頻繁重試
    }
  }
}
```

## ISR vs SSG vs SSR 比較

| 特性 | ISR | SSG | SSR |
|------|-----|-----|-----|
| 效能 | 優秀（接近 SSG） | 最佳 | 良好 |
| 資料新鮮度 | 定期更新 | 靜態 | 即時 |
| 構建時間 | 短（可增量） | 長（全部生成） | 不適用 |
| 伺服器負載 | 低 | 無 | 高 |
| 內容更新 | 自動 | 需重新構建 | 每次請求 |
| 適用場景 | 大部分內容型網站 | 完全靜態的網站 | 個人化/即時內容 |

## 最佳實踐

### 1. 合理設定 revalidate 時間

```javascript
// ✅ 根據實際需求設定
const REVALIDATE_TIMES = {
  STATIC_CONTENT: 86400,    // 24 小時
  BLOG_POST: 3600,          // 1 小時
  NEWS: 60,                 // 1 分鐘
  LIVE_CONTENT: 10          // 10 秒
}

export async function getStaticProps() {
  const post = await fetchPost()

  return {
    props: { post },
    revalidate: REVALIDATE_TIMES.BLOG_POST
  }
}
```

### 2. 結合 fallback 使用

```javascript
export async function getStaticPaths() {
  const popularPosts = await fetchPopularPosts(50)

  return {
    paths: popularPosts.map(p => ({ params: { id: p.id } })),
    fallback: 'blocking' // 搭配 ISR 使用
  }
}

export async function getStaticProps({ params }) {
  const post = await fetchPost(params.id)

  return {
    props: { post },
    revalidate: 60 // 新生成的頁面也會定期更新
  }
}
```

### 3. 使用 On-Demand Revalidation 處理緊急更新

```javascript
// 平時使用較長的 revalidate 時間
export async function getStaticProps() {
  const post = await fetchPost()

  return {
    props: { post },
    revalidate: 3600 // 1 小時
  }
}

// 內容更新時，通過 webhook 立即重新驗證
// 不需要等 1 小時
```

## 小結

ISR 是 Next.js 的殺手級功能，完美結合了 SSG 和 SSR 的優點。對於 Vue3 開發者來說，這是一個全新的概念，但一旦理解，就會發現它極大地簡化了內容網站的開發和維護。

**關鍵要點：**
- ISR 讓靜態頁面可以自動更新
- 使用 `revalidate` 設定更新頻率
- 使用 On-Demand Revalidation 處理即時更新需求
- 根據內容特性選擇合適的 revalidate 時間
- 是大多數內容型網站的最佳選擇
