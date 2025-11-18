# 第四章：Pages Router vs App Router

## 4.1 什麼是 Router（路由器）？

在 Next.js 中，Router 決定了如何組織和處理應用程式的路由。Next.js 提供兩種路由模式：

- **Pages Router**（傳統模式，Next.js 13 之前）
- **App Router**（新模式，Next.js 13+ 引入）

### Vue3 對比

類似於 Vue 生態的演進：
- **Vue Router**（Vue 2/3 的標準路由）
- **Nuxt File-based Routing**（檔案系統路由）

Next.js 的兩種路由器都是檔案系統路由，差異在於實作方式和功能。

## 4.2 Pages Router（傳統模式）

### 4.2.1 基本概念

Pages Router 是 Next.js 的原始路由系統，基於 `pages/` 目錄。

**目錄結構**：
```
pages/
├── index.js              → /
├── about.js              → /about
├── blog/
│   ├── index.js          → /blog
│   └── [slug].js         → /blog/:slug
└── api/
    └── hello.js          → /api/hello
```

### 4.2.2 建立頁面

**pages/index.js**：
```javascript
export default function Home() {
  return (
    <div>
      <h1>首頁</h1>
      <p>這是使用 Pages Router 的頁面</p>
    </div>
  )
}
```

**pages/about.js**：
```javascript
export default function About() {
  return (
    <div>
      <h1>關於我們</h1>
    </div>
  )
}
```

### 4.2.3 動態路由

**pages/blog/[slug].js**：
```javascript
import { useRouter } from 'next/router'

export default function BlogPost() {
  const router = useRouter()
  const { slug } = router.query

  return <h1>文章：{slug}</h1>
}
```

訪問 `/blog/hello-world` 會顯示「文章：hello-world」。

### 4.2.4 資料獲取方法

Pages Router 提供三種資料獲取方法：

```javascript
// 1. getStaticProps（靜態生成）
export async function getStaticProps() {
  const data = await fetchData()
  return { props: { data } }
}

// 2. getServerSideProps（伺服器端渲染）
export async function getServerSideProps(context) {
  const data = await fetchData()
  return { props: { data } }
}

// 3. getStaticPaths（動態路由的靜態生成）
export async function getStaticPaths() {
  const paths = await getPaths()
  return { paths, fallback: false }
}
```

### 4.2.5 Pages Router 優點

✅ **成熟穩定**：長期使用，生產環境驗證
✅ **簡單直觀**：容易理解和上手
✅ **豐富的社群資源**：大量教學和範例
✅ **完整的文檔**：詳盡的官方文檔

### 4.2.6 Pages Router 缺點

❌ **客戶端 Bundle 較大**：所有頁面的 JavaScript 都在客戶端
❌ **無法使用 React Server Components**
❌ **佈局重複渲染**：切換頁面時整個佈局重新渲染
❌ **資料獲取分離**：資料獲取邏輯與元件分離

## 4.3 App Router（新模式）

### 4.3.1 基本概念

App Router 是 Next.js 13 引入的新路由系統，基於 `app/` 目錄。

**目錄結構**：
```
app/
├── layout.js             # 根佈局
├── page.js               # 首頁 (/)
├── about/
│   └── page.js           # 關於頁面 (/about)
├── blog/
│   ├── layout.js         # blog 佈局
│   ├── page.js           # /blog
│   └── [slug]/
│       └── page.js       # /blog/:slug
└── api/
    └── hello/
        └── route.js      # API 路由
```

### 4.3.2 特殊檔案

App Router 使用特殊檔案名稱來定義路由：

| 檔案 | 用途 |
|------|------|
| `layout.js` | 佈局元件 |
| `page.js` | 頁面元件 |
| `loading.js` | 載入 UI |
| `error.js` | 錯誤處理 |
| `not-found.js` | 404 頁面 |
| `route.js` | API 路由 |

### 4.3.3 建立頁面

**app/page.js**（首頁）：
```javascript
export default function Home() {
  return (
    <div>
      <h1>首頁</h1>
      <p>這是使用 App Router 的頁面</p>
    </div>
  )
}
```

**app/about/page.js**：
```javascript
export default function About() {
  return (
    <div>
      <h1>關於我們</h1>
    </div>
  )
}
```

### 4.3.4 佈局系統

**app/layout.js**（根佈局）：
```javascript
export default function RootLayout({ children }) {
  return (
    <html lang="zh-TW">
      <body>
        <header>
          <nav>...</nav>
        </header>
        <main>{children}</main>
        <footer>...</footer>
      </body>
    </html>
  )
}
```

**app/blog/layout.js**（巢狀佈局）：
```javascript
export default function BlogLayout({ children }) {
  return (
    <div className="blog-container">
      <aside>側邊欄</aside>
      <article>{children}</article>
    </div>
  )
}
```

佈局會自動巢狀：
```
RootLayout
  └── BlogLayout
      └── BlogPost Page
```

### 4.3.5 載入與錯誤處理

**app/blog/loading.js**：
```javascript
export default function Loading() {
  return <div>載入中...</div>
}
```

**app/blog/error.js**：
```javascript
'use client'

export default function Error({ error, reset }) {
  return (
    <div>
      <h2>發生錯誤！</h2>
      <p>{error.message}</p>
      <button onClick={reset}>重試</button>
    </div>
  )
}
```

### 4.3.6 伺服器元件 vs 客戶端元件

**預設：伺服器元件**
```javascript
// app/page.js
// 這是伺服器元件（預設）
export default async function Home() {
  const data = await fetch('https://api.example.com/data')
  const json = await data.json()

  return <div>{JSON.stringify(json)}</div>
}
```

**客戶端元件**（需要使用 Hooks 或事件處理）：
```javascript
'use client'

import { useState } from 'react'

export default function Counter() {
  const [count, setCount] = useState(0)

  return (
    <button onClick={() => setCount(count + 1)}>
      點擊次數：{count}
    </button>
  )
}
```

### 4.3.7 資料獲取

App Router 簡化了資料獲取：

```javascript
// 不需要 getServerSideProps，直接在元件中 fetch
export default async function Page() {
  const res = await fetch('https://api.example.com/data', {
    cache: 'no-store' // 相當於 SSR
  })
  const data = await res.json()

  return <div>{JSON.stringify(data)}</div>
}
```

**快取選項**：
```javascript
// 靜態生成（SSG）
fetch(url, { cache: 'force-cache' })

// 伺服器端渲染（SSR）
fetch(url, { cache: 'no-store' })

// 增量靜態再生成（ISR）
fetch(url, { next: { revalidate: 3600 } })
```

### 4.3.8 App Router 優點

✅ **React Server Components**：減少客戶端 JavaScript
✅ **自動程式碼分割**：更精細的分割
✅ **巢狀佈局**：不重新渲染共用佈局
✅ **內建載入與錯誤處理**：更好的使用者體驗
✅ **簡化的資料獲取**：直接在元件中 async/await

### 4.3.9 App Router 缺點

❌ **學習曲線較陡**：新概念（伺服器元件、客戶端元件）
❌ **生態系統仍在成長**：部分套件可能不相容
❌ **文檔較少**：相對新的功能

## 4.4 詳細對比

### 4.4.1 路由定義

| 功能 | Pages Router | App Router |
|------|--------------|------------|
| 頁面檔案 | `pages/about.js` | `app/about/page.js` |
| 動態路由 | `pages/blog/[slug].js` | `app/blog/[slug]/page.js` |
| API 路由 | `pages/api/hello.js` | `app/api/hello/route.js` |
| 巢狀路由 | 手動實作 | 自動支援 |

### 4.4.2 佈局

**Pages Router**：
```javascript
// pages/_app.js
import Layout from '@/components/Layout'

export default function App({ Component, pageProps }) {
  return (
    <Layout>
      <Component {...pageProps} />
    </Layout>
  )
}
```

問題：每次切換頁面，Layout 都會重新渲染。

**App Router**：
```javascript
// app/layout.js
export default function RootLayout({ children }) {
  return (
    <html>
      <body>
        <Header />
        {children}
        <Footer />
      </body>
    </html>
  )
}
```

優勢：切換頁面時，Layout 不會重新渲染。

### 4.4.3 資料獲取

**Pages Router**：
```javascript
// pages/blog/[slug].js
export async function getServerSideProps({ params }) {
  const post = await getPost(params.slug)
  return { props: { post } }
}

export default function BlogPost({ post }) {
  return <h1>{post.title}</h1>
}
```

**App Router**：
```javascript
// app/blog/[slug]/page.js
async function getPost(slug) {
  const res = await fetch(`https://api.example.com/posts/${slug}`)
  return res.json()
}

export default async function BlogPost({ params }) {
  const post = await getPost(params.slug)
  return <h1>{post.title}</h1>
}
```

### 4.4.4 Loading 狀態

**Pages Router**（需手動實作）：
```javascript
import { useState, useEffect } from 'react'

export default function Page() {
  const [loading, setLoading] = useState(true)
  const [data, setData] = useState(null)

  useEffect(() => {
    fetchData().then(data => {
      setData(data)
      setLoading(false)
    })
  }, [])

  if (loading) return <div>載入中...</div>
  return <div>{JSON.stringify(data)}</div>
}
```

**App Router**（自動支援）：
```javascript
// app/page.js
export default async function Page() {
  const data = await fetchData()
  return <div>{JSON.stringify(data)}</div>
}

// app/loading.js
export default function Loading() {
  return <div>載入中...</div>
}
```

## 4.5 如何選擇？

### 選擇 Pages Router 的情況

✅ 你剛開始學習 Next.js
✅ 專案較小，不需要複雜的佈局
✅ 團隊對 Pages Router 更熟悉
✅ 需要使用某些尚未支援 App Router 的套件
✅ 偏好傳統的資料獲取方式

### 選擇 App Router 的情況

✅ 新專案，想使用最新功能
✅ 需要複雜的巢狀佈局
✅ 想減少客戶端 JavaScript
✅ 需要更細粒度的載入狀態
✅ 想利用 React Server Components

## 4.6 混合使用（過渡期）

Next.js 允許在同一專案中同時使用兩種路由器：

```
my-app/
├── app/                  # App Router 頁面
│   └── dashboard/
│       └── page.js
└── pages/                # Pages Router 頁面
    └── legacy-page.js
```

**注意**：
- `app/` 優先於 `pages/`
- 相同路徑會使用 `app/` 的路由
- 建議逐步遷移，不要長期混用

## 4.7 遷移指南（Pages → App）

### 步驟 1：創建 app 目錄

```bash
mkdir app
```

### 步驟 2：創建根佈局

```javascript
// app/layout.js
export default function RootLayout({ children }) {
  return (
    <html lang="zh-TW">
      <body>{children}</body>
    </html>
  )
}
```

### 步驟 3：逐頁遷移

**Pages Router**：
```javascript
// pages/about.js
export default function About() {
  return <h1>關於</h1>
}
```

**App Router**：
```javascript
// app/about/page.js
export default function About() {
  return <h1>關於</h1>
}
```

### 步驟 4：遷移資料獲取

**Before（Pages Router）**：
```javascript
export async function getServerSideProps() {
  const data = await fetchData()
  return { props: { data } }
}

export default function Page({ data }) {
  return <div>{JSON.stringify(data)}</div>
}
```

**After（App Router）**：
```javascript
async function fetchData() {
  const res = await fetch('...', { cache: 'no-store' })
  return res.json()
}

export default async function Page() {
  const data = await fetchData()
  return <div>{JSON.stringify(data)}</div>
}
```

## 4.8 Vue3/Nuxt3 開發者的理解

### Nuxt3 類似 App Router

Nuxt3 的設計與 App Router 非常相似：

| Nuxt3 | Next.js App Router |
|-------|-------------------|
| `app.vue` | `app/layout.js` |
| `pages/index.vue` | `app/page.js` |
| `pages/about.vue` | `app/about/page.js` |
| `layouts/default.vue` | `app/layout.js` |
| `server/api/hello.ts` | `app/api/hello/route.js` |

**Nuxt3 範例**：
```vue
<!-- pages/index.vue -->
<script setup>
const data = await $fetch('/api/data')
</script>

<template>
  <div>{{ data }}</div>
</template>
```

**App Router 範例**：
```javascript
// app/page.js
export default async function Page() {
  const data = await fetch('/api/data').then(r => r.json())
  return <div>{JSON.stringify(data)}</div>
}
```

## 4.9 本章小結

- Next.js 提供兩種路由模式：Pages Router 和 App Router
- Pages Router 適合初學者和傳統專案
- App Router 提供更現代的功能和更好的效能
- 可以根據專案需求選擇合適的路由模式
- App Router 與 Nuxt3 的設計理念相似

## 下一章預告

下一章將深入探討路由系統的基礎，包括如何建立頁面、連結和導航。

## 練習題

1. 使用 Pages Router 創建三個頁面：首頁、關於、聯絡
2. 使用 App Router 創建相同的頁面結構
3. 在 App Router 中創建一個巢狀佈局
4. 比較兩種模式下的資料獲取方式
5. 創建一個簡單的 loading.js 檔案
