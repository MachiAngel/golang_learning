# 第六章：動態路由與參數處理

## 6.1 什麼是動態路由？

動態路由允許你使用變數作為路由的一部分，非常適合建立像部落格文章、產品詳情等頁面。

### Vue Router 對比

**Vue Router**：
```javascript
const routes = [
  { path: '/user/:id', component: User },
  { path: '/post/:slug', component: Post }
]
```

**Next.js**：
```
pages/
├── user/
│   └── [id].js          → /user/:id
└── post/
    └── [slug].js        → /post/:slug
```

使用**方括號 `[]`** 表示動態路由段。

## 6.2 基本動態路由

### 6.2.1 創建動態路由

**檔案結構**：
```
pages/
└── posts/
    └── [id].js          → /posts/:id
```

**pages/posts/[id].js**：
```javascript
import { useRouter } from 'next/router'

export default function Post() {
  const router = useRouter()
  const { id } = router.query

  return (
    <div>
      <h1>文章 ID: {id}</h1>
      <p>這是文章內容</p>
    </div>
  )
}
```

**訪問範例**：
- `/posts/1` → `id = "1"`
- `/posts/hello-world` → `id = "hello-world"`
- `/posts/123` → `id = "123"`

### 6.2.2 獲取動態參數

```javascript
import { useRouter } from 'next/router'

export default function BlogPost() {
  const router = useRouter()
  const { slug } = router.query

  console.log('slug:', slug)
  // URL: /posts/my-first-post
  // slug: "my-first-post"

  return <h1>文章：{slug}</h1>
}
```

### 6.2.3 多個動態參數

**檔案結構**：
```
pages/
└── shop/
    └── [category]/
        └── [product].js    → /shop/:category/:product
```

**pages/shop/[category]/[product].js**：
```javascript
import { useRouter } from 'next/router'

export default function Product() {
  const router = useRouter()
  const { category, product } = router.query

  return (
    <div>
      <h1>分類：{category}</h1>
      <h2>產品：{product}</h2>
    </div>
  )
}
```

**訪問範例**：
- `/shop/electronics/laptop` → `category: "electronics"`, `product: "laptop"`
- `/shop/books/javascript-guide` → `category: "books"`, `product: "javascript-guide"`

## 6.3 捕獲所有路由（Catch-all Routes）

### 6.3.1 基本捕獲所有路由

使用 `[...slug].js` 捕獲多個路由段。

**檔案結構**：
```
pages/
└── docs/
    └── [...slug].js    → /docs/*
```

**pages/docs/[...slug].js**：
```javascript
import { useRouter } from 'next/router'

export default function Docs() {
  const router = useRouter()
  const { slug } = router.query

  // slug 是一個陣列
  console.log('slug:', slug)

  return (
    <div>
      <h1>文檔路徑</h1>
      <p>{slug ? slug.join(' / ') : '首頁'}</p>
    </div>
  )
}
```

**訪問範例**：
- `/docs/getting-started` → `slug: ["getting-started"]`
- `/docs/api/reference` → `slug: ["api", "reference"]`
- `/docs/guides/auth/setup` → `slug: ["guides", "auth", "setup"]`

### 6.3.2 可選捕獲所有路由

使用 `[[...slug]].js`（雙方括號）使路由參數可選。

**檔案結構**：
```
pages/
└── blog/
    └── [[...slug]].js    → /blog 和 /blog/*
```

**pages/blog/[[...slug]].js**：
```javascript
import { useRouter } from 'next/router'

export default function Blog() {
  const router = useRouter()
  const { slug } = router.query

  if (!slug) {
    return <h1>部落格首頁</h1>
  }

  return (
    <div>
      <h1>文章分類</h1>
      <p>{slug.join(' / ')}</p>
    </div>
  )
}
```

**訪問範例**：
- `/blog` → `slug: undefined`（顯示部落格首頁）
- `/blog/tutorial` → `slug: ["tutorial"]`
- `/blog/tutorial/nextjs` → `slug: ["tutorial", "nextjs"]`

## 6.4 動態路由的資料獲取

### 6.4.1 使用 getStaticProps

```javascript
// pages/posts/[id].js
export async function getStaticPaths() {
  // 獲取所有文章 ID
  const posts = await getAllPosts()

  const paths = posts.map(post => ({
    params: { id: post.id.toString() }
  }))

  return {
    paths,
    fallback: false
  }
}

export async function getStaticProps({ params }) {
  // 根據 ID 獲取文章資料
  const post = await getPostById(params.id)

  return {
    props: {
      post
    }
  }
}

export default function Post({ post }) {
  return (
    <article>
      <h1>{post.title}</h1>
      <p>{post.content}</p>
    </article>
  )
}
```

### 6.4.2 使用 getServerSideProps

```javascript
// pages/users/[id].js
export async function getServerSideProps({ params }) {
  const { id } = params

  // 每次請求時獲取使用者資料
  const res = await fetch(`https://api.example.com/users/${id}`)
  const user = await res.json()

  // 如果使用者不存在，返回 404
  if (!user) {
    return {
      notFound: true
    }
  }

  return {
    props: {
      user
    }
  }
}

export default function UserProfile({ user }) {
  return (
    <div>
      <h1>{user.name}</h1>
      <p>Email: {user.email}</p>
    </div>
  )
}
```

## 6.5 連結到動態路由

### 6.5.1 基本連結

```javascript
import Link from 'next/link'

export default function PostList({ posts }) {
  return (
    <ul>
      {posts.map(post => (
        <li key={post.id}>
          <Link href={`/posts/${post.id}`}>
            {post.title}
          </Link>
        </li>
      ))}
    </ul>
  )
}
```

### 6.5.2 使用物件語法

```javascript
import Link from 'next/link'

export default function PostList({ posts }) {
  return (
    <ul>
      {posts.map(post => (
        <li key={post.id}>
          <Link
            href={{
              pathname: '/posts/[id]',
              query: { id: post.id }
            }}
          >
            {post.title}
          </Link>
        </li>
      ))}
    </ul>
  )
}
```

### 6.5.3 帶查詢參數的動態連結

```javascript
import Link from 'next/link'

export default function ProductCard({ product }) {
  return (
    <Link
      href={{
        pathname: '/products/[id]',
        query: {
          id: product.id,
          ref: 'homepage',
          utm_source: 'email'
        }
      }}
    >
      {product.name}
    </Link>
  )
  // 結果：/products/123?ref=homepage&utm_source=email
}
```

## 6.6 程式化導航到動態路由

### 6.6.1 基本導航

```javascript
import { useRouter } from 'next/router'

export default function ProductCard({ product }) {
  const router = useRouter()

  const handleClick = () => {
    router.push(`/products/${product.id}`)
  }

  return (
    <div onClick={handleClick}>
      <h3>{product.name}</h3>
    </div>
  )
}
```

### 6.6.2 使用物件語法

```javascript
import { useRouter } from 'next/router'

export default function SearchResults({ results }) {
  const router = useRouter()

  const viewProduct = (productId) => {
    router.push({
      pathname: '/products/[id]',
      query: { id: productId }
    })
  }

  return (
    <div>
      {results.map(product => (
        <button key={product.id} onClick={() => viewProduct(product.id)}>
          查看 {product.name}
        </button>
      ))}
    </div>
  )
}
```

## 6.7 查詢參數 vs 路由參數

### 6.7.1 路由參數（動態路由段）

```javascript
// pages/posts/[id].js
// URL: /posts/123

const router = useRouter()
const { id } = router.query  // id = "123"
```

### 6.7.2 查詢參數

```javascript
// pages/search.js
// URL: /search?q=nextjs&category=tutorial

const router = useRouter()
const { q, category } = router.query
// q = "nextjs"
// category = "tutorial"
```

### 6.7.3 混合使用

```javascript
// pages/products/[id].js
// URL: /products/123?tab=reviews&sort=date

const router = useRouter()
const { id, tab, sort } = router.query
// id = "123" （路由參數）
// tab = "reviews" （查詢參數）
// sort = "date" （查詢參數）
```

## 6.8 實戰範例

### 6.8.1 部落格系統

**目錄結構**：
```
pages/
├── blog/
│   ├── index.js              → /blog（文章列表）
│   ├── [slug].js             → /blog/:slug（單篇文章）
│   └── category/
│       └── [category].js     → /blog/category/:category
```

**pages/blog/index.js**：
```javascript
import Link from 'next/link'

export async function getStaticProps() {
  const posts = await getAllPosts()

  return {
    props: { posts }
  }
}

export default function BlogIndex({ posts }) {
  return (
    <div>
      <h1>部落格</h1>
      <ul>
        {posts.map(post => (
          <li key={post.slug}>
            <Link href={`/blog/${post.slug}`}>
              <h2>{post.title}</h2>
              <p>{post.excerpt}</p>
            </Link>
          </li>
        ))}
      </ul>
    </div>
  )
}
```

**pages/blog/[slug].js**：
```javascript
import { useRouter } from 'next/router'
import Link from 'next/link'

export async function getStaticPaths() {
  const posts = await getAllPosts()

  return {
    paths: posts.map(post => ({
      params: { slug: post.slug }
    })),
    fallback: false
  }
}

export async function getStaticProps({ params }) {
  const post = await getPostBySlug(params.slug)

  return {
    props: { post }
  }
}

export default function BlogPost({ post }) {
  const router = useRouter()

  if (router.isFallback) {
    return <div>載入中...</div>
  }

  return (
    <article>
      <h1>{post.title}</h1>
      <p>
        <time>{post.date}</time>
        {' | '}
        <Link href={`/blog/category/${post.category}`}>
          {post.category}
        </Link>
      </p>
      <div dangerouslySetInnerHTML={{ __html: post.content }} />
    </article>
  )
}
```

### 6.8.2 電商產品頁

**pages/products/[id].js**：
```javascript
import { useState } from 'react'
import { useRouter } from 'next/router'

export async function getServerSideProps({ params, query }) {
  const { id } = params
  const { variant } = query

  const product = await getProduct(id)

  return {
    props: {
      product,
      selectedVariant: variant || product.variants[0].id
    }
  }
}

export default function Product({ product, selectedVariant }) {
  const router = useRouter()
  const [quantity, setQuantity] = useState(1)

  const handleVariantChange = (variantId) => {
    router.push(
      {
        pathname: `/products/${product.id}`,
        query: { variant: variantId }
      },
      undefined,
      { shallow: true }
    )
  }

  return (
    <div>
      <h1>{product.name}</h1>
      <p>${product.price}</p>

      <div>
        <label>規格：</label>
        <select
          value={selectedVariant}
          onChange={(e) => handleVariantChange(e.target.value)}
        >
          {product.variants.map(variant => (
            <option key={variant.id} value={variant.id}>
              {variant.name}
            </option>
          ))}
        </select>
      </div>

      <div>
        <label>數量：</label>
        <input
          type="number"
          value={quantity}
          onChange={(e) => setQuantity(e.target.value)}
          min="1"
        />
      </div>

      <button>加入購物車</button>
    </div>
  )
}
```

### 6.8.3 多語系路由

**pages/[lang]/about.js**：
```javascript
import { useRouter } from 'next/router'

export async function getStaticPaths() {
  const languages = ['zh-TW', 'en', 'ja']

  return {
    paths: languages.map(lang => ({
      params: { lang }
    })),
    fallback: false
  }
}

export async function getStaticProps({ params }) {
  const { lang } = params
  const translations = await getTranslations(lang)

  return {
    props: { lang, translations }
  }
}

export default function About({ lang, translations }) {
  const router = useRouter()

  const changeLanguage = (newLang) => {
    router.push(`/${newLang}/about`)
  }

  return (
    <div>
      <nav>
        <button onClick={() => changeLanguage('zh-TW')}>繁體中文</button>
        <button onClick={() => changeLanguage('en')}>English</button>
        <button onClick={() => changeLanguage('ja')}>日本語</button>
      </nav>

      <h1>{translations.title}</h1>
      <p>{translations.content}</p>
    </div>
  )
}
```

## 6.9 Fallback 模式

### 6.9.1 fallback: false

```javascript
export async function getStaticPaths() {
  return {
    paths: [
      { params: { id: '1' } },
      { params: { id: '2' } }
    ],
    fallback: false  // 其他路徑返回 404
  }
}
```

訪問 `/posts/3` 會顯示 404。

### 6.9.2 fallback: true

```javascript
export async function getStaticPaths() {
  // 只預先生成熱門文章
  const popularPosts = await getPopularPosts()

  return {
    paths: popularPosts.map(post => ({
      params: { id: post.id }
    })),
    fallback: true  // 其他頁面在首次訪問時生成
  }
}

export default function Post({ post }) {
  const router = useRouter()

  // 如果頁面尚未生成，顯示載入狀態
  if (router.isFallback) {
    return <div>載入中...</div>
  }

  return <h1>{post.title}</h1>
}
```

### 6.9.3 fallback: 'blocking'

```javascript
export async function getStaticPaths() {
  return {
    paths: [],
    fallback: 'blocking'  // 伺服器端等待頁面生成完成
  }
}
```

不會顯示載入狀態，使用者會等待頁面完全生成。

## 6.10 Vue3/Nuxt3 對比

### 動態路由

**Nuxt3**：
```
pages/
└── posts/
    └── [id].vue
```

```vue
<script setup>
const route = useRoute()
const { id } = route.params
</script>

<template>
  <div>文章 ID: {{ id }}</div>
</template>
```

**Next.js**：
```
pages/
└── posts/
    └── [id].js
```

```javascript
import { useRouter } from 'next/router'

export default function Post() {
  const router = useRouter()
  const { id } = router.query

  return <div>文章 ID: {id}</div>
}
```

語法非常相似！

## 6.11 本章小結

- 使用方括號 `[param]` 創建動態路由
- 使用 `[...slug]` 捕獲所有路由
- 使用 `[[...slug]]` 創建可選捕獲所有路由
- 透過 `router.query` 獲取動態參數
- 可以混合使用路由參數和查詢參數
- 使用 `fallback` 控制未預先生成頁面的行為

## 下一章預告

下一章將深入探討 `getStaticProps`，學習如何在建置時獲取資料並生成靜態頁面。

## 練習題

1. 創建一個部落格，包含文章列表和文章詳情頁
2. 實作產品分類頁面（`/products/[category]`）
3. 創建一個捕獲所有路由的文檔頁面
4. 實作使用者個人頁面（`/users/[username]`）
5. 在動態路由中添加查詢參數（如篩選、排序）
6. 試驗不同的 `fallback` 模式
