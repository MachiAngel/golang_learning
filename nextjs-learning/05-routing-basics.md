# 第五章：路由系統基礎

## 5.1 檔案系統路由

Next.js 使用**檔案系統路由**（File-system Routing），這意味著檔案結構直接對應 URL 路徑。

### 5.1.1 基本路由（Pages Router）

```
pages/
├── index.js              → /
├── about.js              → /about
├── contact.js            → /contact
└── products.js           → /products
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

訪問 `http://localhost:3000/about` 即可看到此頁面。

### 5.1.2 巢狀路由

```
pages/
├── index.js              → /
└── blog/
    ├── index.js          → /blog
    ├── first-post.js     → /blog/first-post
    └── second-post.js    → /blog/second-post
```

**範例：pages/blog/index.js**
```javascript
export default function Blog() {
  return (
    <div>
      <h1>部落格首頁</h1>
      <ul>
        <li>文章列表</li>
      </ul>
    </div>
  )
}
```

### 5.1.3 Vue Router 對比

**Vue Router（需手動配置）**：
```javascript
const routes = [
  { path: '/', component: Home },
  { path: '/about', component: About },
  { path: '/blog', component: Blog },
  { path: '/blog/first-post', component: FirstPost }
]
```

**Next.js（自動生成）**：
- 只需創建檔案
- 路由自動生成
- 不需要配置檔

## 5.2 頁面之間的導航

### 5.2.1 使用 Link 元件

Next.js 提供 `<Link>` 元件用於客戶端導航。

**基本用法**：
```javascript
import Link from 'next/link'

export default function Home() {
  return (
    <div>
      <h1>首頁</h1>
      <nav>
        <Link href="/">首頁</Link>
        <Link href="/about">關於</Link>
        <Link href="/contact">聯絡</Link>
      </nav>
    </div>
  )
}
```

**與 Vue 對比**：

```vue
<!-- Vue -->
<template>
  <nav>
    <router-link to="/">首頁</router-link>
    <router-link to="/about">關於</router-link>
  </nav>
</template>
```

```javascript
// Next.js
import Link from 'next/link'

<nav>
  <Link href="/">首頁</Link>
  <Link href="/about">關於</Link>
</nav>
```

### 5.2.2 Link 元件的特性

**預載入（Prefetching）**：
```javascript
// 預設會預載入可見的連結
<Link href="/about">關於</Link>

// 禁用預載入
<Link href="/about" prefetch={false}>關於</Link>
```

**樣式應用**：
```javascript
import Link from 'next/link'
import styles from './Navigation.module.css'

export default function Navigation() {
  return (
    <Link href="/about" className={styles.link}>
      關於我們
    </Link>
  )
}
```

**包裹自訂元件**：
```javascript
<Link href="/about">
  <button className="custom-button">前往關於頁面</button>
</Link>
```

### 5.2.3 動態連結

```javascript
import Link from 'next/link'

export default function BlogList({ posts }) {
  return (
    <ul>
      {posts.map(post => (
        <li key={post.id}>
          <Link href={`/blog/${post.slug}`}>
            {post.title}
          </Link>
        </li>
      ))}
    </ul>
  )
}
```

**使用物件語法**：
```javascript
<Link
  href={{
    pathname: '/blog/[slug]',
    query: { slug: 'my-post' }
  }}
>
  閱讀文章
</Link>
```

## 5.3 程式化導航

### 5.3.1 使用 useRouter Hook

```javascript
import { useRouter } from 'next/router'

export default function LoginForm() {
  const router = useRouter()

  const handleSubmit = async (e) => {
    e.preventDefault()
    // 登入邏輯...

    // 導航到儀表板
    router.push('/dashboard')
  }

  return (
    <form onSubmit={handleSubmit}>
      <button type="submit">登入</button>
    </form>
  )
}
```

### 5.3.2 router.push()

```javascript
import { useRouter } from 'next/router'

export default function Example() {
  const router = useRouter()

  // 基本導航
  const handleClick = () => {
    router.push('/about')
  }

  // 帶查詢參數
  const handleSearch = () => {
    router.push({
      pathname: '/search',
      query: { q: 'Next.js' }
    })
    // 結果：/search?q=Next.js
  }

  // 帶 hash
  const handleSection = () => {
    router.push('/docs#installation')
  }

  return (
    <div>
      <button onClick={handleClick}>前往關於頁面</button>
      <button onClick={handleSearch}>搜尋</button>
      <button onClick={handleSection}>查看文檔</button>
    </div>
  )
}
```

### 5.3.3 router.replace()

```javascript
// push：添加新的歷史記錄
router.push('/dashboard')  // 可以返回上一頁

// replace：替換當前歷史記錄
router.replace('/dashboard')  // 無法返回上一頁
```

**使用場景**：
```javascript
// 登入成功後重定向（不希望使用者返回登入頁）
const handleLogin = async () => {
  await login()
  router.replace('/dashboard')
}
```

### 5.3.4 router.back()

```javascript
import { useRouter } from 'next/router'

export default function PostDetail() {
  const router = useRouter()

  return (
    <div>
      <button onClick={() => router.back()}>
        返回上一頁
      </button>
    </div>
  )
}
```

### 5.3.5 Vue Router 對比

**Vue**：
```javascript
import { useRouter } from 'vue-router'

const router = useRouter()

// 導航
router.push('/about')

// 替換
router.replace('/about')

// 返回
router.back()
```

**Next.js**：
```javascript
import { useRouter } from 'next/router'

const router = useRouter()

// 導航
router.push('/about')

// 替換
router.replace('/about')

// 返回
router.back()
```

語法幾乎相同！

## 5.4 獲取路由資訊

### 5.4.1 當前路徑

```javascript
import { useRouter } from 'next/router'

export default function Navigation() {
  const router = useRouter()

  return (
    <nav>
      <Link
        href="/"
        className={router.pathname === '/' ? 'active' : ''}
      >
        首頁
      </Link>
      <Link
        href="/about"
        className={router.pathname === '/about' ? 'active' : ''}
      >
        關於
      </Link>
    </nav>
  )
}
```

### 5.4.2 查詢參數

```javascript
import { useRouter } from 'next/router'

export default function SearchPage() {
  const router = useRouter()
  const { q, category } = router.query

  // URL: /search?q=Next.js&category=tutorial
  // q = "Next.js"
  // category = "tutorial"

  return (
    <div>
      <h1>搜尋結果：{q}</h1>
      <p>分類：{category}</p>
    </div>
  )
}
```

### 5.4.3 router 物件屬性

```javascript
import { useRouter } from 'next/router'

export default function DebugPage() {
  const router = useRouter()

  return (
    <div>
      <p>pathname: {router.pathname}</p>
      {/* /blog/[slug] */}

      <p>asPath: {router.asPath}</p>
      {/* /blog/hello-world?foo=bar */}

      <p>query: {JSON.stringify(router.query)}</p>
      {/* {"slug": "hello-world", "foo": "bar"} */}

      <p>locale: {router.locale}</p>
      {/* zh-TW */}
    </div>
  )
}
```

## 5.5 淺層路由（Shallow Routing）

淺層路由允許你改變 URL 而不重新執行資料獲取方法。

```javascript
import { useRouter } from 'next/router'
import { useEffect } from 'react'

export default function Page() {
  const router = useRouter()

  useEffect(() => {
    // 更改 URL 而不重新載入頁面
    router.push('/?counter=10', undefined, { shallow: true })
  }, [])

  useEffect(() => {
    // 監聽 URL 變化
    console.log('Counter changed to:', router.query.counter)
  }, [router.query.counter])

  return <div>查看控制台輸出</div>
}
```

**使用場景**：
- 篩選器、排序
- 分頁
- 標籤切換

**範例：產品篩選**
```javascript
import { useRouter } from 'next/router'

export default function Products() {
  const router = useRouter()
  const { sort = 'name' } = router.query

  const handleSort = (newSort) => {
    router.push(
      { query: { ...router.query, sort: newSort } },
      undefined,
      { shallow: true }
    )
  }

  return (
    <div>
      <button onClick={() => handleSort('name')}>依名稱排序</button>
      <button onClick={() => handleSort('price')}>依價格排序</button>
      <p>當前排序：{sort}</p>
    </div>
  )
}
```

## 5.6 路由事件

### 5.6.1 監聽路由變化

```javascript
import { useRouter } from 'next/router'
import { useEffect } from 'react'

export default function MyApp({ Component, pageProps }) {
  const router = useRouter()

  useEffect(() => {
    const handleStart = (url) => {
      console.log('路由開始變化:', url)
    }

    const handleComplete = (url) => {
      console.log('路由變化完成:', url)
    }

    const handleError = (err, url) => {
      console.log('路由錯誤:', err, url)
    }

    router.events.on('routeChangeStart', handleStart)
    router.events.on('routeChangeComplete', handleComplete)
    router.events.on('routeChangeError', handleError)

    return () => {
      router.events.off('routeChangeStart', handleStart)
      router.events.off('routeChangeComplete', handleComplete)
      router.events.off('routeChangeError', handleError)
    }
  }, [router])

  return <Component {...pageProps} />
}
```

### 5.6.2 載入進度條

```javascript
import { useRouter } from 'next/router'
import { useEffect, useState } from 'react'
import NProgress from 'nprogress'
import 'nprogress/nprogress.css'

export default function MyApp({ Component, pageProps }) {
  const router = useRouter()

  useEffect(() => {
    const handleStart = () => NProgress.start()
    const handleComplete = () => NProgress.done()

    router.events.on('routeChangeStart', handleStart)
    router.events.on('routeChangeComplete', handleComplete)
    router.events.on('routeChangeError', handleComplete)

    return () => {
      router.events.off('routeChangeStart', handleStart)
      router.events.off('routeChangeComplete', handleComplete)
      router.events.off('routeChangeError', handleComplete)
    }
  }, [router])

  return <Component {...pageProps} />
}
```

## 5.7 導航守衛

### 5.7.1 阻止路由變化

```javascript
import { useRouter } from 'next/router'
import { useEffect, useState } from 'react'

export default function EditForm() {
  const router = useRouter()
  const [hasUnsavedChanges, setHasUnsavedChanges] = useState(false)

  useEffect(() => {
    const handleWindowClose = (e) => {
      if (!hasUnsavedChanges) return
      e.preventDefault()
      return (e.returnValue = '你有未儲存的變更，確定要離開嗎？')
    }

    const handleBrowseAway = () => {
      if (!hasUnsavedChanges) return
      if (window.confirm('你有未儲存的變更，確定要離開嗎？')) {
        return true
      }
      router.events.emit('routeChangeError')
      throw 'routeChange aborted.'
    }

    window.addEventListener('beforeunload', handleWindowClose)
    router.events.on('routeChangeStart', handleBrowseAway)

    return () => {
      window.removeEventListener('beforeunload', handleWindowClose)
      router.events.off('routeChangeStart', handleBrowseAway)
    }
  }, [hasUnsavedChanges, router])

  return (
    <form onChange={() => setHasUnsavedChanges(true)}>
      <input type="text" placeholder="編輯內容..." />
    </form>
  )
}
```

### 5.7.2 認證守衛

```javascript
import { useRouter } from 'next/router'
import { useEffect } from 'react'

export default function ProtectedPage() {
  const router = useRouter()

  useEffect(() => {
    const isAuthenticated = checkAuth() // 檢查認證狀態

    if (!isAuthenticated) {
      router.replace('/login')
    }
  }, [router])

  return <div>受保護的內容</div>
}
```

**更好的做法：使用高階元件（HOC）**
```javascript
// lib/withAuth.js
import { useRouter } from 'next/router'
import { useEffect } from 'react'

export function withAuth(Component) {
  return function ProtectedRoute(props) {
    const router = useRouter()
    const isAuthenticated = checkAuth()

    useEffect(() => {
      if (!isAuthenticated) {
        router.replace('/login')
      }
    }, [isAuthenticated, router])

    if (!isAuthenticated) {
      return <div>載入中...</div>
    }

    return <Component {...props} />
  }
}

// 使用
import { withAuth } from '@/lib/withAuth'

function Dashboard() {
  return <div>儀表板</div>
}

export default withAuth(Dashboard)
```

## 5.8 404 頁面

### 5.8.1 自訂 404 頁面

創建 `pages/404.js`：

```javascript
import Link from 'next/link'

export default function Custom404() {
  return (
    <div style={{ textAlign: 'center', marginTop: '50px' }}>
      <h1>404 - 頁面不存在</h1>
      <p>抱歉，你訪問的頁面不存在。</p>
      <Link href="/">返回首頁</Link>
    </div>
  )
}
```

### 5.8.2 自訂 500 頁面

創建 `pages/500.js`：

```javascript
export default function Custom500() {
  return (
    <div style={{ textAlign: 'center', marginTop: '50px' }}>
      <h1>500 - 伺服器錯誤</h1>
      <p>抱歉，伺服器發生錯誤。</p>
    </div>
  )
}
```

## 5.9 實戰範例：導航選單

### 5.9.1 Active Link 元件

```javascript
// components/ActiveLink.js
import { useRouter } from 'next/router'
import Link from 'next/link'

export default function ActiveLink({ href, children, className = '', activeClassName = 'active' }) {
  const router = useRouter()
  const isActive = router.pathname === href

  return (
    <Link
      href={href}
      className={`${className} ${isActive ? activeClassName : ''}`}
    >
      {children}
    </Link>
  )
}
```

**使用**：
```javascript
import ActiveLink from '@/components/ActiveLink'

export default function Navigation() {
  return (
    <nav>
      <ActiveLink href="/">首頁</ActiveLink>
      <ActiveLink href="/about">關於</ActiveLink>
      <ActiveLink href="/blog">部落格</ActiveLink>
    </nav>
  )
}
```

**CSS**：
```css
/* Navigation.module.css */
.nav a {
  color: #333;
  text-decoration: none;
  padding: 0.5rem 1rem;
}

.nav a.active {
  color: #0070f3;
  font-weight: bold;
  border-bottom: 2px solid #0070f3;
}
```

## 5.10 本章小結

- Next.js 使用檔案系統路由，檔案結構即路由結構
- 使用 `<Link>` 元件進行客戶端導航
- 使用 `useRouter` Hook 進行程式化導航
- 可以監聽路由事件並實作守衛
- 支援自訂 404 和 500 錯誤頁面

## 下一章預告

下一章將深入探討動態路由與參數處理，學習如何建立動態頁面和處理 URL 參數。

## 練習題

1. 創建一個導航選單，包含首頁、關於、部落格三個頁面
2. 實作 ActiveLink 元件，高亮當前頁面
3. 創建一個搜尋頁面，使用查詢參數顯示搜尋關鍵字
4. 實作一個簡單的認證守衛
5. 自訂 404 頁面，添加一些創意設計
6. 使用路由事件添加頁面載入進度條
