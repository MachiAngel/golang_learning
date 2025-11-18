# 第三章：專案結構與檔案組織

## 3.1 標準專案結構

### 典型的 Next.js 專案（Pages Router）

```
my-nextjs-app/
├── pages/                    # 頁面與路由
│   ├── _app.js              # 自訂 App 元件
│   ├── _document.js         # 自訂 Document
│   ├── index.js             # 首頁 (/)
│   ├── about.js             # 關於頁面 (/about)
│   └── api/                 # API 路由
│       └── hello.js         # API 端點
├── components/              # 可重用元件
│   ├── Header.js
│   ├── Footer.js
│   └── Layout.js
├── public/                  # 靜態資源
│   ├── images/
│   ├── favicon.ico
│   └── robots.txt
├── styles/                  # 樣式檔案
│   ├── globals.css          # 全域樣式
│   └── Home.module.css      # CSS Modules
├── lib/                     # 工具函數
│   ├── api.js
│   └── utils.js
├── hooks/                   # 自訂 Hooks
│   └── useAuth.js
├── context/                 # React Context
│   └── AuthContext.js
├── .env.local              # 環境變數
├── .gitignore
├── next.config.js          # Next.js 配置
├── package.json
└── README.md
```

### Vue3/Nuxt3 對比

```
# Nuxt3 專案結構
my-nuxt-app/
├── pages/                   # 頁面與路由（同 Next.js）
├── components/              # 元件（同 Next.js）
├── composables/             # Composable 函數（≈ hooks/）
├── public/                  # 靜態資源（同 Next.js）
├── server/                  # 伺服器端點（≈ pages/api/）
├── assets/                  # 需編譯的資源（≈ styles/）
├── layouts/                 # 佈局元件（Next.js 用 components）
└── nuxt.config.ts          # Nuxt 配置（≈ next.config.js）
```

## 3.2 核心目錄詳解

### 3.2.1 pages/ 目錄

**用途**：定義應用程式的路由和頁面

**檔案即路由**：
```
pages/
├── index.js              → /
├── about.js              → /about
├── blog/
│   ├── index.js          → /blog
│   └── [slug].js         → /blog/:slug
└── products/
    └── [id].js           → /products/:id
```

**範例：pages/about.js**
```javascript
export default function About() {
  return (
    <div>
      <h1>關於我們</h1>
      <p>這是關於頁面</p>
    </div>
  )
}
```

**與 Vue Router 對比**：

Vue Router 需要手動配置：
```javascript
// Vue Router
const routes = [
  { path: '/', component: Home },
  { path: '/about', component: About }
]
```

Next.js 自動生成路由：
```
// 只需創建檔案
pages/about.js → /about
```

### 3.2.2 pages/_app.js

**用途**：自訂全域應用程式元件

**基本結構**：
```javascript
import '@/styles/globals.css'

export default function App({ Component, pageProps }) {
  return <Component {...pageProps} />
}
```

**常見用途：添加全域 Layout**
```javascript
import '@/styles/globals.css'
import Layout from '@/components/Layout'

export default function App({ Component, pageProps }) {
  return (
    <Layout>
      <Component {...pageProps} />
    </Layout>
  )
}
```

**添加狀態管理（Context）**：
```javascript
import { AuthProvider } from '@/context/AuthContext'
import '@/styles/globals.css'

export default function App({ Component, pageProps }) {
  return (
    <AuthProvider>
      <Component {...pageProps} />
    </AuthProvider>
  )
}
```

**Nuxt3 等價**：
```vue
<!-- app.vue -->
<template>
  <NuxtLayout>
    <NuxtPage />
  </NuxtLayout>
</template>
```

### 3.2.3 pages/_document.js

**用途**：自訂 HTML 文件結構（僅在伺服器端渲染）

**基本結構**：
```javascript
import { Html, Head, Main, NextScript } from 'next/document'

export default function Document() {
  return (
    <Html lang="zh-TW">
      <Head>
        {/* 添加全域 meta tags、fonts 等 */}
        <link rel="icon" href="/favicon.ico" />
      </Head>
      <body>
        <Main />
        <NextScript />
      </body>
    </Html>
  )
}
```

**添加 Google Fonts**：
```javascript
import { Html, Head, Main, NextScript } from 'next/document'

export default function Document() {
  return (
    <Html lang="zh-TW">
      <Head>
        <link
          href="https://fonts.googleapis.com/css2?family=Noto+Sans+TC:wght@400;700&display=swap"
          rel="stylesheet"
        />
      </Head>
      <body>
        <Main />
        <NextScript />
      </body>
    </Html>
  )
}
```

**重要**：
- `_document.js` 只在伺服器端執行
- 不能使用 React Hooks
- 不能添加事件處理器

### 3.2.4 pages/api/ 目錄

**用途**：創建 API 端點（無伺服器函數）

**基本 API 路由**：
```javascript
// pages/api/hello.js → /api/hello

export default function handler(req, res) {
  res.status(200).json({ message: 'Hello from Next.js!' })
}
```

**處理不同 HTTP 方法**：
```javascript
// pages/api/users.js

export default function handler(req, res) {
  const { method } = req

  switch (method) {
    case 'GET':
      // 取得使用者列表
      res.status(200).json({ users: [] })
      break
    case 'POST':
      // 創建新使用者
      const { name } = req.body
      res.status(201).json({ user: { name } })
      break
    default:
      res.setHeader('Allow', ['GET', 'POST'])
      res.status(405).end(`Method ${method} Not Allowed`)
  }
}
```

**動態 API 路由**：
```javascript
// pages/api/users/[id].js → /api/users/:id

export default function handler(req, res) {
  const { id } = req.query
  res.status(200).json({ userId: id })
}
```

**Nuxt3 等價**：
```typescript
// server/api/hello.ts
export default defineEventHandler(() => {
  return { message: 'Hello from Nuxt!' }
})
```

### 3.2.5 public/ 目錄

**用途**：存放靜態資源（直接提供服務，不經過 webpack）

**結構範例**：
```
public/
├── images/
│   ├── logo.png
│   └── hero.jpg
├── fonts/
│   └── custom-font.woff2
├── favicon.ico
└── robots.txt
```

**使用方式**：
```javascript
// 在元件中引用
export default function Home() {
  return (
    <div>
      <img src="/images/logo.png" alt="Logo" />
      {/* 注意：路徑從 / 開始，不包含 public */}
    </div>
  )
}
```

**與 Vue3 對比**：
- Next.js: `public/images/logo.png` → `/images/logo.png`
- Vue3: `public/images/logo.png` → `/images/logo.png`（相同）

### 3.2.6 components/ 目錄

**用途**：存放可重用的 React 元件

**範例：components/Header.js**
```javascript
import Link from 'next/link'
import styles from './Header.module.css'

export default function Header() {
  return (
    <header className={styles.header}>
      <nav>
        <Link href="/">首頁</Link>
        <Link href="/about">關於</Link>
        <Link href="/blog">部落格</Link>
      </nav>
    </header>
  )
}
```

**範例：components/Layout.js**
```javascript
import Header from './Header'
import Footer from './Footer'

export default function Layout({ children }) {
  return (
    <>
      <Header />
      <main>{children}</main>
      <Footer />
    </>
  )
}
```

**使用 Layout**：
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

### 3.2.7 styles/ 目錄

**用途**：存放樣式檔案

**結構**：
```
styles/
├── globals.css           # 全域樣式
├── Home.module.css       # 頁面專用 CSS Module
└── components/           # 元件樣式
    ├── Header.module.css
    └── Footer.module.css
```

**globals.css（全域樣式）**：
```css
/* styles/globals.css */
* {
  box-sizing: border-box;
  margin: 0;
  padding: 0;
}

body {
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen;
  line-height: 1.6;
}
```

**CSS Modules（元件隔離樣式）**：
```css
/* styles/components/Header.module.css */
.header {
  padding: 1rem;
  background-color: #333;
  color: white;
}

.nav {
  display: flex;
  gap: 1rem;
}
```

```javascript
// 使用 CSS Module
import styles from './Header.module.css'

export default function Header() {
  return (
    <header className={styles.header}>
      <nav className={styles.nav}>...</nav>
    </header>
  )
}
```

### 3.2.8 lib/ 目錄

**用途**：存放工具函數和共用邏輯

**範例：lib/api.js**
```javascript
// API 呼叫工具
export async function fetchAPI(endpoint) {
  const res = await fetch(`${process.env.API_URL}${endpoint}`)

  if (!res.ok) {
    throw new Error('API 請求失敗')
  }

  return res.json()
}

// 取得文章列表
export async function getPosts() {
  return fetchAPI('/posts')
}

// 取得單篇文章
export async function getPost(id) {
  return fetchAPI(`/posts/${id}`)
}
```

**範例：lib/utils.js**
```javascript
// 日期格式化
export function formatDate(date) {
  return new Date(date).toLocaleDateString('zh-TW', {
    year: 'numeric',
    month: 'long',
    day: 'numeric'
  })
}

// 截斷文字
export function truncate(str, length) {
  return str.length > length ? str.substring(0, length) + '...' : str
}
```

### 3.2.9 hooks/ 目錄

**用途**：存放自訂 React Hooks

**範例：hooks/useAuth.js**
```javascript
import { useState, useEffect } from 'react'

export function useAuth() {
  const [user, setUser] = useState(null)
  const [loading, setLoading] = useState(true)

  useEffect(() => {
    // 檢查使用者登入狀態
    const checkAuth = async () => {
      try {
        const res = await fetch('/api/auth/me')
        const data = await res.json()
        setUser(data.user)
      } catch (error) {
        setUser(null)
      } finally {
        setLoading(false)
      }
    }

    checkAuth()
  }, [])

  return { user, loading }
}
```

**使用自訂 Hook**：
```javascript
import { useAuth } from '@/hooks/useAuth'

export default function Profile() {
  const { user, loading } = useAuth()

  if (loading) return <div>載入中...</div>
  if (!user) return <div>請先登入</div>

  return <div>歡迎，{user.name}</div>
}
```

**Vue3 對比（Composables）**：
```javascript
// composables/useAuth.js
import { ref, onMounted } from 'vue'

export function useAuth() {
  const user = ref(null)
  const loading = ref(true)

  onMounted(async () => {
    // ... 同樣的邏輯
  })

  return { user, loading }
}
```

## 3.3 配置檔案

### 3.3.1 next.config.js

```javascript
/** @type {import('next').NextConfig} */
const nextConfig = {
  // React 嚴格模式
  reactStrictMode: true,

  // 圖片優化設定
  images: {
    domains: ['example.com', 'cdn.example.com'],
  },

  // 環境變數（暴露給客戶端）
  env: {
    CUSTOM_KEY: 'my-value',
  },

  // 重定向
  async redirects() {
    return [
      {
        source: '/old-blog/:slug',
        destination: '/blog/:slug',
        permanent: true,
      },
    ]
  },

  // 自訂 Webpack 配置
  webpack: (config, { isServer }) => {
    // 自訂配置
    return config
  },
}

module.exports = nextConfig
```

### 3.3.2 .env.local（環境變數）

```bash
# .env.local

# 資料庫連線
DATABASE_URL=postgresql://user:password@localhost:5432/mydb

# API 金鑰（僅伺服器端）
API_SECRET_KEY=your-secret-key

# 公開的環境變數（加上 NEXT_PUBLIC_ 前綴）
NEXT_PUBLIC_API_URL=https://api.example.com
```

**使用環境變數**：
```javascript
// 伺服器端（API Routes、getServerSideProps 等）
const dbUrl = process.env.DATABASE_URL
const apiKey = process.env.API_SECRET_KEY

// 客戶端（React 元件）
const apiUrl = process.env.NEXT_PUBLIC_API_URL
```

**重要**：
- 不加 `NEXT_PUBLIC_` 的變數只能在伺服器端使用
- 加了 `NEXT_PUBLIC_` 的變數會打包到客戶端
- 永遠不要將 `.env.local` 提交到 Git

## 3.4 檔案命名慣例

### 3.4.1 元件檔案

```
components/
├── Header.js              # PascalCase
├── Footer.js
└── blog/
    ├── PostCard.js        # 子目錄也用 PascalCase
    └── PostList.js
```

### 3.4.2 頁面檔案

```
pages/
├── index.js               # 小寫
├── about.js
└── blog/
    ├── index.js
    └── [slug].js          # 動態路由用中括號
```

### 3.4.3 CSS 檔案

```
styles/
├── globals.css            # 小寫，連字號
├── Home.module.css        # Module: PascalCase
└── utils.css
```

### 3.4.4 工具檔案

```
lib/
├── api.js                 # 小寫
├── utils.js
└── constants.js
```

## 3.5 路徑別名（Path Alias）

### 設定 jsconfig.json 或 tsconfig.json

```json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/components/*": ["components/*"],
      "@/styles/*": ["styles/*"],
      "@/lib/*": ["lib/*"],
      "@/hooks/*": ["hooks/*"]
    }
  }
}
```

### 使用路徑別名

```javascript
// 不使用別名（相對路徑）
import Header from '../../../components/Header'
import { fetchAPI } from '../../../lib/api'

// 使用別名（絕對路徑）
import Header from '@/components/Header'
import { fetchAPI } from '@/lib/api'
```

## 3.6 最佳實踐

### 1. 元件組織

```
components/
├── layout/               # 佈局元件
│   ├── Header.js
│   ├── Footer.js
│   └── Sidebar.js
├── ui/                   # UI 元件
│   ├── Button.js
│   ├── Card.js
│   └── Modal.js
└── features/             # 功能模組
    ├── auth/
    │   ├── LoginForm.js
    │   └── SignupForm.js
    └── blog/
        ├── PostCard.js
        └── PostList.js
```

### 2. 每個元件一個目錄（進階）

```
components/
└── Header/
    ├── index.js          # 主元件
    ├── Header.module.css # 樣式
    ├── Header.test.js    # 測試
    └── Logo.js           # 子元件
```

### 3. 使用 index.js 簡化導入

```
components/
└── ui/
    ├── index.js          # 匯出所有 UI 元件
    ├── Button.js
    ├── Card.js
    └── Modal.js
```

```javascript
// components/ui/index.js
export { default as Button } from './Button'
export { default as Card } from './Card'
export { default as Modal } from './Modal'

// 使用時可以一次導入多個
import { Button, Card, Modal } from '@/components/ui'
```

## 3.7 本章小結

- Next.js 使用約定優於配置的原則
- `pages/` 目錄決定路由結構
- `_app.js` 和 `_document.js` 是特殊檔案
- 使用 CSS Modules 實現樣式隔離
- 善用路徑別名簡化導入
- 合理組織目錄結構提高可維護性

## 下一章預告

下一章將深入探討 Next.js 的兩種路由模式：Pages Router 和 App Router，幫助你選擇適合的路由策略。

## 練習題

1. 創建一個 `components/Layout.js` 元件，包含 Header 和 Footer
2. 在 `_app.js` 中套用 Layout
3. 創建一個 `lib/utils.js`，實作一個日期格式化函數
4. 設定路徑別名並在專案中使用
5. 創建一個簡單的 API 路由 `/api/time`，返回當前時間
