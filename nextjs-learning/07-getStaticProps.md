# 第七章：資料獲取：getStaticProps

## 7.1 什麼是 getStaticProps？

`getStaticProps` 是 Next.js 提供的資料獲取方法，用於在**建置時（Build Time）**獲取資料並生成靜態頁面。

### 特點

- ✅ **靜態生成（SSG）**：頁面在建置時預先生成
- ✅ **高效能**：直接提供靜態 HTML，速度極快
- ✅ **SEO 友善**：搜尋引擎可以完整抓取內容
- ✅ **CDN 快取**：可以部署到 CDN，全球快速訪問
- ⚠️ **資料更新**：資料在建置時固定，需要重新建置才能更新

### Vue3/Nuxt3 對比

**Nuxt3（類似功能）**：
```vue
<script setup>
// 在 Nuxt3 中，這會在建置時執行
const { data } = await useFetch('/api/posts')
</script>
```

**Next.js**：
```javascript
export async function getStaticProps() {
  const res = await fetch('https://api.example.com/posts')
  const posts = await res.json()

  return {
    props: { posts }
  }
}
```

## 7.2 基本用法

### 7.2.1 簡單範例

**pages/blog.js**：
```javascript
export async function getStaticProps() {
  // 在建置時執行
  const res = await fetch('https://api.example.com/posts')
  const posts = await res.json()

  // 返回 props 給頁面元件
  return {
    props: {
      posts
    }
  }
}

export default function Blog({ posts }) {
  return (
    <div>
      <h1>部落格</h1>
      <ul>
        {posts.map(post => (
          <li key={post.id}>
            <h2>{post.title}</h2>
            <p>{post.excerpt}</p>
          </li>
        ))}
      </ul>
    </div>
  )
}
```

### 7.2.2 執行時機

```javascript
export async function getStaticProps() {
  console.log('這會在建置時執行，不會在瀏覽器執行')

  // 可以安全地使用：
  // - 檔案系統 (fs)
  // - 資料庫查詢
  // - 私密 API 金鑰

  return {
    props: {
      buildTime: new Date().toISOString()
    }
  }
}
```

**重要**：
- `getStaticProps` 只在**伺服器端**執行
- 永遠不會在客戶端（瀏覽器）執行
- 程式碼不會打包到客戶端 bundle

## 7.3 返回值選項

### 7.3.1 props

最基本的返回值，傳遞資料給頁面元件。

```javascript
export async function getStaticProps() {
  return {
    props: {
      message: 'Hello World',
      count: 42,
      items: ['a', 'b', 'c']
    }
  }
}

export default function Page({ message, count, items }) {
  return (
    <div>
      <p>{message}</p>
      <p>Count: {count}</p>
      <ul>
        {items.map(item => (
          <li key={item}>{item}</li>
        ))}
      </ul>
    </div>
  )
}
```

### 7.3.2 revalidate（增量靜態再生成 ISR）

設定頁面重新驗證的間隔時間（秒）。

```javascript
export async function getStaticProps() {
  const res = await fetch('https://api.example.com/posts')
  const posts = await res.json()

  return {
    props: { posts },
    revalidate: 60  // 每 60 秒重新驗證一次
  }
}
```

**工作原理**：
1. 首次請求返回建置時的靜態頁面
2. 60 秒後的下一次請求，觸發頁面重新生成
3. 重新生成完成後，新的靜態頁面取代舊的

**使用場景**：
- 部落格文章（需要定期更新）
- 產品列表（庫存變化）
- 新聞網站

### 7.3.3 notFound

返回 404 頁面。

```javascript
export async function getStaticProps({ params }) {
  const post = await getPost(params.id)

  // 如果文章不存在，返回 404
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

### 7.3.4 redirect

重定向到其他頁面。

```javascript
export async function getStaticProps() {
  const isMaintenanceMode = await checkMaintenanceMode()

  if (isMaintenanceMode) {
    return {
      redirect: {
        destination: '/maintenance',
        permanent: false
      }
    }
  }

  return {
    props: {}
  }
}
```

## 7.4 Context 參數

`getStaticProps` 接收一個 context 物件作為參數。

```javascript
export async function getStaticProps(context) {
  const {
    params,      // 動態路由參數
    preview,     // 預覽模式
    previewData, // 預覽資料
    locale,      // 當前語系
    locales,     // 所有語系
    defaultLocale // 預設語系
  } = context

  return {
    props: {}
  }
}
```

### 7.4.1 params（動態路由）

```javascript
// pages/posts/[id].js
export async function getStaticProps({ params }) {
  const { id } = params
  const post = await getPostById(id)

  return {
    props: { post }
  }
}
```

### 7.4.2 preview 和 previewData

用於內容預覽功能（CMS 整合）。

```javascript
export async function getStaticProps({ preview, previewData }) {
  // 預覽模式下獲取草稿內容
  if (preview) {
    const draftPost = await getDraftPost(previewData.id)
    return {
      props: { post: draftPost }
    }
  }

  // 正常模式下獲取已發佈內容
  const post = await getPublishedPost()
  return {
    props: { post }
  }
}
```

## 7.5 資料來源範例

### 7.5.1 從檔案系統讀取

```javascript
import fs from 'fs'
import path from 'path'
import matter from 'gray-matter'

export async function getStaticProps() {
  const postsDirectory = path.join(process.cwd(), 'posts')
  const filenames = fs.readdirSync(postsDirectory)

  const posts = filenames.map(filename => {
    const filePath = path.join(postsDirectory, filename)
    const fileContents = fs.readFileSync(filePath, 'utf8')
    const { data, content } = matter(fileContents)

    return {
      slug: filename.replace('.md', ''),
      title: data.title,
      date: data.date,
      content
    }
  })

  return {
    props: { posts }
  }
}
```

### 7.5.2 從 API 獲取

```javascript
export async function getStaticProps() {
  const res = await fetch('https://api.example.com/posts')
  const posts = await res.json()

  return {
    props: { posts },
    revalidate: 3600  // 每小時重新驗證
  }
}
```

### 7.5.3 從資料庫查詢

```javascript
import { PrismaClient } from '@prisma/client'

const prisma = new PrismaClient()

export async function getStaticProps() {
  const posts = await prisma.post.findMany({
    where: { published: true },
    orderBy: { createdAt: 'desc' }
  })

  return {
    props: {
      posts: JSON.parse(JSON.stringify(posts))  // 序列化 Date 物件
    },
    revalidate: 60
  }
}
```

### 7.5.4 從 CMS（Headless CMS）

```javascript
export async function getStaticProps() {
  // 從 Contentful 獲取
  const client = require('contentful').createClient({
    space: process.env.CONTENTFUL_SPACE_ID,
    accessToken: process.env.CONTENTFUL_ACCESS_TOKEN
  })

  const entries = await client.getEntries({
    content_type: 'blogPost'
  })

  return {
    props: {
      posts: entries.items
    },
    revalidate: 3600
  }
}
```

## 7.6 完整範例：部落格

### 7.6.1 文章列表頁

**pages/blog/index.js**：
```javascript
import Link from 'next/link'
import { getAllPosts } from '@/lib/posts'

export async function getStaticProps() {
  const posts = await getAllPosts()

  return {
    props: { posts },
    revalidate: 60  // 每分鐘重新驗證
  }
}

export default function BlogIndex({ posts }) {
  return (
    <div>
      <h1>部落格</h1>
      <div>
        {posts.map(post => (
          <article key={post.slug}>
            <Link href={`/blog/${post.slug}`}>
              <h2>{post.title}</h2>
            </Link>
            <time>{post.date}</time>
            <p>{post.excerpt}</p>
          </article>
        ))}
      </div>
    </div>
  )
}
```

### 7.6.2 文章詳情頁

**pages/blog/[slug].js**：
```javascript
import { getAllPosts, getPostBySlug } from '@/lib/posts'

export async function getStaticPaths() {
  const posts = await getAllPosts()

  return {
    paths: posts.map(post => ({
      params: { slug: post.slug }
    })),
    fallback: 'blocking'  // 新文章會自動生成
  }
}

export async function getStaticProps({ params }) {
  const post = await getPostBySlug(params.slug)

  if (!post) {
    return {
      notFound: true
    }
  }

  return {
    props: { post },
    revalidate: 60
  }
}

export default function BlogPost({ post }) {
  return (
    <article>
      <h1>{post.title}</h1>
      <time>{post.date}</time>
      <div dangerouslySetInnerHTML={{ __html: post.content }} />
    </article>
  )
}
```

### 7.6.3 工具函數

**lib/posts.js**：
```javascript
import fs from 'fs'
import path from 'path'
import matter from 'gray-matter'
import { remark } from 'remark'
import html from 'remark-html'

const postsDirectory = path.join(process.cwd(), 'content/posts')

export async function getAllPosts() {
  const filenames = fs.readdirSync(postsDirectory)

  const posts = await Promise.all(
    filenames.map(async filename => {
      const slug = filename.replace('.md', '')
      const post = await getPostBySlug(slug)
      return post
    })
  )

  // 依日期排序
  return posts.sort((a, b) => (a.date > b.date ? -1 : 1))
}

export async function getPostBySlug(slug) {
  const filePath = path.join(postsDirectory, `${slug}.md`)

  if (!fs.existsSync(filePath)) {
    return null
  }

  const fileContents = fs.readFileSync(filePath, 'utf8')
  const { data, content } = matter(fileContents)

  // Markdown 轉 HTML
  const processedContent = await remark().use(html).process(content)
  const contentHtml = processedContent.toString()

  return {
    slug,
    title: data.title,
    date: data.date,
    excerpt: data.excerpt,
    content: contentHtml
  }
}
```

## 7.7 增量靜態再生成（ISR）深入

### 7.7.1 工作流程

```javascript
export async function getStaticProps() {
  const data = await fetchData()

  return {
    props: { data },
    revalidate: 10  // 10 秒後重新驗證
  }
}
```

**時間軸**：
1. **T=0s**：建置時生成靜態頁面
2. **T=5s**：使用者訪問，返回靜態頁面（快取）
3. **T=15s**：使用者訪問，返回靜態頁面，**觸發後台重新生成**
4. **T=16s**：後台重新生成完成
5. **T=20s**：使用者訪問，返回**新的**靜態頁面

### 7.7.2 On-Demand ISR（按需重新驗證）

**API Route（pages/api/revalidate.js）**：
```javascript
export default async function handler(req, res) {
  // 驗證密鑰
  if (req.query.secret !== process.env.REVALIDATE_SECRET) {
    return res.status(401).json({ message: 'Invalid token' })
  }

  try {
    // 重新驗證特定頁面
    await res.revalidate('/blog/my-post')
    return res.json({ revalidated: true })
  } catch (err) {
    return res.status(500).send('Error revalidating')
  }
}
```

**使用（Webhook）**：
```bash
# 當 CMS 內容更新時，呼叫此 API
curl https://your-site.com/api/revalidate?secret=YOUR_SECRET
```

### 7.7.3 Fallback 模式

**fallback: false**：
```javascript
export async function getStaticPaths() {
  return {
    paths: [{ params: { id: '1' } }],
    fallback: false  // 其他路徑返回 404
  }
}
```

**fallback: true**：
```javascript
export async function getStaticPaths() {
  return {
    paths: [],  // 不預先生成
    fallback: true
  }
}

export default function Post({ post }) {
  const router = useRouter()

  if (router.isFallback) {
    return <div>載入中...</div>
  }

  return <h1>{post.title}</h1>
}
```

**fallback: 'blocking'**：
```javascript
export async function getStaticPaths() {
  return {
    paths: [],
    fallback: 'blocking'  // SSR 方式處理，不顯示載入狀態
  }
}
```

## 7.8 效能優化

### 7.8.1 平行獲取資料

```javascript
export async function getStaticProps() {
  // ❌ 序列執行（慢）
  const posts = await fetchPosts()
  const authors = await fetchAuthors()
  const categories = await fetchCategories()

  // ✅ 平行執行（快）
  const [posts, authors, categories] = await Promise.all([
    fetchPosts(),
    fetchAuthors(),
    fetchCategories()
  ])

  return {
    props: { posts, authors, categories }
  }
}
```

### 7.8.2 限制資料大小

```javascript
export async function getStaticProps() {
  const posts = await getAllPosts()

  // 只返回必要的欄位
  const postsData = posts.map(post => ({
    id: post.id,
    title: post.title,
    excerpt: post.excerpt,
    date: post.date
    // 不包含 content（內容過大）
  }))

  return {
    props: { posts: postsData }
  }
}
```

### 7.8.3 快取策略

```javascript
import NodeCache from 'node-cache'

const cache = new NodeCache({ stdTTL: 600 })  // 10 分鐘快取

export async function getStaticProps() {
  const cacheKey = 'all-posts'
  let posts = cache.get(cacheKey)

  if (!posts) {
    posts = await fetchPosts()
    cache.set(cacheKey, posts)
  }

  return {
    props: { posts },
    revalidate: 60
  }
}
```

## 7.9 錯誤處理

### 7.9.1 Try-Catch

```javascript
export async function getStaticProps() {
  try {
    const res = await fetch('https://api.example.com/posts')

    if (!res.ok) {
      throw new Error('Failed to fetch posts')
    }

    const posts = await res.json()

    return {
      props: { posts }
    }
  } catch (error) {
    console.error('Error in getStaticProps:', error)

    // 返回空資料或預設值
    return {
      props: {
        posts: [],
        error: 'Failed to load posts'
      }
    }
  }
}

export default function Blog({ posts, error }) {
  if (error) {
    return <div>錯誤：{error}</div>
  }

  return (
    <ul>
      {posts.map(post => (
        <li key={post.id}>{post.title}</li>
      ))}
    </ul>
  )
}
```

### 7.9.2 Fallback 資料

```javascript
export async function getStaticProps() {
  try {
    const posts = await fetchPosts()
    return { props: { posts } }
  } catch (error) {
    // 使用本地備份資料
    const fallbackPosts = require('../data/posts-fallback.json')
    return { props: { posts: fallbackPosts } }
  }
}
```

## 7.10 最佳實踐

### ✅ 適合使用 getStaticProps 的場景

- 內容不常更新的頁面（部落格、文檔）
- 資料可以在建置時獲取
- 需要最佳 SEO
- 需要最快的首屏載入
- 可以預先知道所有頁面路徑

### ❌ 不適合使用 getStaticProps 的場景

- 資料頻繁更新
- 需要即時資料
- 使用者特定的內容（如個人儀表板）
- 依賴請求時資訊（如查詢參數、Headers）

### 📌 建議

1. **結合 ISR**：使用 `revalidate` 定期更新
2. **使用 fallback**：處理大量動態路由
3. **限制資料大小**：只返回必要的資料
4. **錯誤處理**：總是處理 API 失敗的情況
5. **利用快取**：減少重複的資料獲取

## 7.11 本章小結

- `getStaticProps` 用於建置時獲取資料
- 返回的資料會傳遞給頁面元件
- 支援 ISR（增量靜態再生成）
- 可以設定 `revalidate` 定期更新頁面
- 適合內容驅動的網站（部落格、文檔等）

## 下一章預告

下一章將探討 `getServerSideProps`，學習如何在每次請求時獲取資料，實現伺服器端渲染。

## 練習題

1. 創建一個部落格首頁，使用 `getStaticProps` 獲取文章列表
2. 實作 ISR，設定 60 秒的重新驗證時間
3. 處理資料獲取失敗的情況
4. 創建一個產品列表頁，從 API 獲取資料
5. 實作 On-Demand ISR API 端點
6. 比較不同 `fallback` 模式的差異
