# 第 9 章：資料獲取：getStaticPaths 與動態路由

## 概述

在 Vue3 中，我們使用 Vue Router 搭配動態路由參數（如 `/posts/:id`）來處理動態頁面。在 Next.js 中，我們使用檔案系統路由搭配 `getStaticPaths` 和 `getStaticProps` 來實現靜態生成的動態路由。

## Vue3 vs Next.js 動態路由對比

### Vue3 的做法

```javascript
// Vue Router 設定
const routes = [
  { path: '/posts/:id', component: PostDetail }
]

// PostDetail.vue
export default {
  async created() {
    const id = this.$route.params.id
    const response = await fetch(`/api/posts/${id}`)
    this.post = await response.json()
  }
}
```

### Next.js 的做法

```javascript
// pages/posts/[id].js
export async function getStaticPaths() {
  // 預先生成所有文章的路徑
  const response = await fetch('https://api.example.com/posts')
  const posts = await response.json()

  const paths = posts.map(post => ({
    params: { id: post.id.toString() }
  }))

  return { paths, fallback: false }
}

export async function getStaticProps({ params }) {
  const response = await fetch(`https://api.example.com/posts/${params.id}`)
  const post = await response.json()

  return { props: { post } }
}

export default function Post({ post }) {
  return (
    <div>
      <h1>{post.title}</h1>
      <p>{post.content}</p>
    </div>
  )
}
```

## getStaticPaths 核心概念

### 1. 基本結構

`getStaticPaths` 必須返回一個包含 `paths` 和 `fallback` 的物件：

```javascript
export async function getStaticPaths() {
  return {
    paths: [
      { params: { id: '1' } },
      { params: { id: '2' } },
      { params: { id: '3' } }
    ],
    fallback: false
  }
}
```

**重點說明（對 Vue 開發者）：**
- 類似 Vue 的預渲染，但更智能
- 在構建時執行，不是在用戶端
- 必須搭配 `getStaticProps` 使用

### 2. fallback 選項詳解

#### fallback: false

```javascript
export async function getStaticPaths() {
  const posts = await fetchAllPosts() // 假設有 100 篇文章

  const paths = posts.map(post => ({
    params: { id: post.id.toString() }
  }))

  return {
    paths,
    fallback: false // 未列出的路徑會返回 404
  }
}
```

**使用場景：**
- 頁面數量少且固定
- 所有頁面都要在構建時生成
- 不會有新內容加入

#### fallback: true

```javascript
export async function getStaticPaths() {
  // 只預先生成最熱門的 10 篇文章
  const popularPosts = await fetchPopularPosts(10)

  const paths = popularPosts.map(post => ({
    params: { id: post.id.toString() }
  }))

  return {
    paths,
    fallback: true // 其他路徑會在第一次請求時生成
  }
}

export default function Post({ post }) {
  const router = useRouter()

  // 如果頁面尚未生成，顯示載入狀態
  if (router.isFallback) {
    return <div>載入中...</div>
  }

  return (
    <div>
      <h1>{post.title}</h1>
      <p>{post.content}</p>
    </div>
  )
}
```

**使用場景：**
- 頁面數量很多（如電商網站有數千個商品）
- 只預先生成熱門頁面
- 其他頁面在首次請求時才生成

#### fallback: 'blocking'

```javascript
export async function getStaticPaths() {
  const paths = [] // 可以是空陣列

  return {
    paths,
    fallback: 'blocking' // 伺服器端生成，不顯示載入狀態
  }
}

export default function Post({ post }) {
  // 不需要檢查 isFallback
  // 用戶會等待直到頁面完全生成
  return (
    <div>
      <h1>{post.title}</h1>
      <p>{post.content}</p>
    </div>
  )
}
```

**使用場景：**
- 不想顯示載入狀態
- SEO 很重要（爬蟲會等待完整頁面）
- 可接受稍長的首次載入時間

### fallback 選項比較表

| 選項 | 未預生成頁面的行為 | 用戶體驗 | 適用場景 |
|------|-------------------|----------|----------|
| `false` | 返回 404 | 無等待 | 頁面少且固定 |
| `true` | 背景生成，顯示 fallback UI | 先看載入狀態 | 頁面多，需快速構建 |
| `'blocking'` | 伺服器端等待生成完成 | 等待完整頁面 | SEO 重要，頁面多 |

## 實際應用案例

### 案例 1：部落格文章系統

```javascript
// pages/posts/[slug].js
import { getAllPosts, getPostBySlug } from '../../lib/api'

export async function getStaticPaths() {
  const posts = await getAllPosts()

  const paths = posts.map(post => ({
    params: { slug: post.slug }
  }))

  return {
    paths,
    fallback: false
  }
}

export async function getStaticProps({ params }) {
  const post = await getPostBySlug(params.slug)

  return {
    props: { post },
    revalidate: 3600 // 每小時重新驗證一次
  }
}

export default function PostPage({ post }) {
  return (
    <article>
      <h1>{post.title}</h1>
      <time>{new Date(post.date).toLocaleDateString('zh-TW')}</time>
      <div dangerouslySetInnerHTML={{ __html: post.content }} />
    </article>
  )
}
```

### 案例 2：電商產品頁面

```javascript
// pages/products/[id].js
export async function getStaticPaths() {
  // 只預先生成前 100 個熱門產品
  const response = await fetch('https://api.shop.com/products/popular?limit=100')
  const products = await response.json()

  const paths = products.map(product => ({
    params: { id: product.id.toString() }
  }))

  return {
    paths,
    fallback: 'blocking' // 其他產品頁面會在請求時生成
  }
}

export async function getStaticProps({ params }) {
  try {
    const response = await fetch(`https://api.shop.com/products/${params.id}`)

    if (!response.ok) {
      return { notFound: true } // 產品不存在時返回 404
    }

    const product = await response.json()

    return {
      props: { product },
      revalidate: 60 // 每分鐘重新驗證（價格可能變動）
    }
  } catch (error) {
    return { notFound: true }
  }
}

export default function ProductPage({ product }) {
  return (
    <div>
      <h1>{product.name}</h1>
      <p>價格：NT$ {product.price}</p>
      <p>庫存：{product.stock}</p>
      <img src={product.image} alt={product.name} />
      <div dangerouslySetInnerHTML={{ __html: product.description }} />
    </div>
  )
}
```

### 案例 3：多層動態路由

```javascript
// pages/categories/[category]/[product].js
export async function getStaticPaths() {
  const categories = await fetchCategories()
  const paths = []

  // 遍歷每個分類，取得該分類下的產品
  for (const category of categories) {
    const products = await fetchProductsByCategory(category.slug)

    products.forEach(product => {
      paths.push({
        params: {
          category: category.slug,
          product: product.slug
        }
      })
    })
  }

  return {
    paths,
    fallback: 'blocking'
  }
}

export async function getStaticProps({ params }) {
  const { category, product } = params

  const productData = await fetchProduct(category, product)

  if (!productData) {
    return { notFound: true }
  }

  return {
    props: {
      category,
      product: productData
    },
    revalidate: 3600
  }
}

export default function ProductPage({ category, product }) {
  return (
    <div>
      <nav>
        <a href="/">首頁</a> /
        <a href={`/categories/${category}`}>{category}</a> /
        <span>{product.name}</span>
      </nav>
      <h1>{product.name}</h1>
      <p>{product.description}</p>
    </div>
  )
}
```

## 動態路由檔案命名規則

### 單一動態參數

```
pages/
  posts/
    [id].js        → /posts/1, /posts/2, /posts/abc
    [slug].js      → /posts/hello-world, /posts/next-js
```

### 多個動態參數

```
pages/
  [category]/
    [product].js   → /electronics/phone, /books/novel
```

### Catch-all 路由

```javascript
// pages/docs/[...slug].js
// 匹配：/docs/a, /docs/a/b, /docs/a/b/c

export async function getStaticPaths() {
  return {
    paths: [
      { params: { slug: ['getting-started'] } },
      { params: { slug: ['api', 'reference'] } },
      { params: { slug: ['guides', 'deployment', 'vercel'] } }
    ],
    fallback: false
  }
}

export async function getStaticProps({ params }) {
  // params.slug 是一個陣列
  const slug = params.slug.join('/')
  const doc = await fetchDoc(slug)

  return {
    props: { doc }
  }
}

export default function DocPage({ doc }) {
  return (
    <div>
      <h1>{doc.title}</h1>
      <div dangerouslySetInnerHTML={{ __html: doc.content }} />
    </div>
  )
}
```

### Optional Catch-all 路由

```javascript
// pages/docs/[[...slug]].js
// 匹配：/docs, /docs/a, /docs/a/b

export async function getStaticPaths() {
  return {
    paths: [
      { params: { slug: [] } },           // /docs
      { params: { slug: ['intro'] } },     // /docs/intro
      { params: { slug: ['api', 'v1'] } }  // /docs/api/v1
    ],
    fallback: false
  }
}
```

## 錯誤處理與 notFound

```javascript
export async function getStaticProps({ params }) {
  try {
    const post = await fetchPost(params.id)

    // 如果文章不存在
    if (!post) {
      return {
        notFound: true // 會顯示 404 頁面
      }
    }

    return {
      props: { post }
    }
  } catch (error) {
    console.error('獲取文章失敗：', error)

    // 發生錯誤時也返回 404
    return {
      notFound: true
    }
  }
}
```

## 與 Vue3 的比較總結

| 功能 | Vue3 + Vue Router | Next.js |
|------|-------------------|---------|
| 動態路由定義 | 在 router 設定中定義 | 檔案名稱使用 `[param]` |
| 資料獲取時機 | 組件掛載時（用戶端） | 構建時（伺服器端） |
| SEO | 需要額外配置 SSR | 天生支援 |
| 效能 | 首次載入需要獲取資料 | 預先生成，載入即可用 |
| 彈性 | 完全動態 | 靜態 + 動態混合 |

## 最佳實踐

### 1. 合理使用 fallback

```javascript
// ❌ 不好的做法：頁面很多但設定 fallback: false
export async function getStaticPaths() {
  const products = await fetchAllProducts() // 10000+ 個產品
  return {
    paths: products.map(p => ({ params: { id: p.id } })),
    fallback: false // 構建時間會很長！
  }
}

// ✅ 好的做法：使用 fallback: 'blocking'
export async function getStaticPaths() {
  const popularProducts = await fetchPopularProducts(50)
  return {
    paths: popularProducts.map(p => ({ params: { id: p.id } })),
    fallback: 'blocking' // 其他產品在請求時生成
  }
}
```

### 2. 參數驗證

```javascript
export async function getStaticProps({ params }) {
  // 驗證參數格式
  if (!/^\d+$/.test(params.id)) {
    return { notFound: true }
  }

  const id = parseInt(params.id)

  if (id < 1 || id > 10000) {
    return { notFound: true }
  }

  const post = await fetchPost(id)

  return {
    props: { post }
  }
}
```

### 3. 使用 TypeScript

```typescript
// pages/posts/[id].tsx
import { GetStaticPaths, GetStaticProps } from 'next'

interface Post {
  id: number
  title: string
  content: string
}

interface PostPageProps {
  post: Post
}

export const getStaticPaths: GetStaticPaths = async () => {
  const posts = await fetchAllPosts()

  return {
    paths: posts.map((post) => ({
      params: { id: post.id.toString() }
    })),
    fallback: false
  }
}

export const getStaticProps: GetStaticProps<PostPageProps> = async ({ params }) => {
  const post = await fetchPost(params!.id as string)

  return {
    props: { post }
  }
}

export default function PostPage({ post }: PostPageProps) {
  return <h1>{post.title}</h1>
}
```

## 常見問題

### Q1: getStaticPaths 可以在用戶端執行嗎？
**A:** 不行，`getStaticPaths` 只在構建時執行，永遠不會在用戶端執行。

### Q2: 我可以在 getStaticPaths 中使用環境變數嗎？
**A:** 可以，但只能使用構建時的環境變數（不是運行時環境變數）。

### Q3: fallback: true 和 fallback: 'blocking' 哪個比較好？
**A:**
- 如果 SEO 很重要，使用 `'blocking'`
- 如果用戶體驗優先（想顯示載入狀態），使用 `true`

### Q4: 動態路由可以搭配 getServerSideProps 使用嗎？
**A:** 可以！動態路由不一定要用 `getStaticPaths`，也可以用 `getServerSideProps`（適合每次請求都需要最新資料的場景）。

## 小結

`getStaticPaths` 是 Next.js 靜態生成動態路由的核心功能，搭配不同的 `fallback` 選項，可以在構建時間、用戶體驗和 SEO 之間取得平衡。對於 Vue3 開發者來說，這是一個全新的思維模式，但一旦掌握，就能充分發揮靜態生成的優勢。
