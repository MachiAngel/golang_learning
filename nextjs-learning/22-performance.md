# 第 22 章：效能優化最佳實踐

## 22.1 效能優化概述

Next.js 提供了許多內建的效能優化功能，但要達到最佳效能，還需要開發者主動採取優化措施。本章將深入探討各種優化策略。

## 22.2 Core Web Vitals 優化

### 22.2.1 什麼是 Core Web Vitals？

Core Web Vitals 是 Google 定義的三個核心指標：

1. **LCP (Largest Contentful Paint)**：最大內容繪製時間（應 < 2.5 秒）
2. **FID (First Input Delay)**：首次輸入延遲（應 < 100 毫秒）
3. **CLS (Cumulative Layout Shift)**：累積版面配置位移（應 < 0.1）

### 22.2.2 測量 Web Vitals

```bash
npm install web-vitals
```

```tsx
// pages/_app.tsx
import type { AppProps } from 'next/app'
import { useReportWebVitals } from 'next/web-vitals'

function MyApp({ Component, pageProps }: AppProps) {
  return <Component {...pageProps} />
}

export function reportWebVitals(metric: any) {
  // 將指標發送到分析服務
  console.log(metric)

  switch (metric.name) {
    case 'FCP':
      // First Contentful Paint
      break
    case 'LCP':
      // Largest Contentful Paint
      break
    case 'CLS':
      // Cumulative Layout Shift
      break
    case 'FID':
      // First Input Delay
      break
    case 'TTFB':
      // Time to First Byte
      break
    default:
      break
  }
}

export default MyApp
```

### 22.2.3 優化 LCP

**策略一：優化圖片載入**

```tsx
// components/Hero.tsx
import Image from 'next/image'

export default function Hero() {
  return (
    <div className="hero">
      <Image
        src="/hero-image.jpg"
        alt="Hero"
        width={1200}
        height={600}
        priority // 關鍵：標記為優先載入
        placeholder="blur"
        blurDataURL="data:image/jpeg;base64,/9j/4AAQSkZJRg..." // 模糊佔位符
      />
    </div>
  )
}
```

**策略二：預載入關鍵資源**

```tsx
// pages/_document.tsx
import { Html, Head, Main, NextScript } from 'next/document'

export default function Document() {
  return (
    <Html>
      <Head>
        {/* 預載入關鍵字型 */}
        <link
          rel="preload"
          href="/fonts/inter-var.woff2"
          as="font"
          type="font/woff2"
          crossOrigin="anonymous"
        />

        {/* 預連接到外部域名 */}
        <link rel="preconnect" href="https://fonts.googleapis.com" />
        <link rel="preconnect" href="https://fonts.gstatic.com" crossOrigin="anonymous" />
      </Head>
      <body>
        <Main />
        <NextScript />
      </body>
    </Html>
  )
}
```

**策略三：使用 SSG 而非 SSR**

```tsx
// pages/blog/[slug].tsx - 使用 SSG
export const getStaticProps: GetStaticProps = async ({ params }) => {
  const post = await fetchPost(params?.slug as string)

  return {
    props: { post },
    revalidate: 3600 // ISR: 每小時重新生成
  }
}

export const getStaticPaths: GetStaticPaths = async () => {
  const posts = await fetchAllPosts()

  return {
    paths: posts.map(post => ({ params: { slug: post.slug } })),
    fallback: 'blocking' // 或 true
  }
}
```

### 22.2.4 優化 FID

**策略一：程式碼分割**

```tsx
// 使用動態導入延遲載入非關鍵組件
import dynamic from 'next/dynamic'

const HeavyComponent = dynamic(() => import('@/components/HeavyComponent'), {
  loading: () => <div>載入中...</div>,
  ssr: false // 禁用 SSR
})

export default function Page() {
  return (
    <div>
      <h1>頁面標題</h1>
      <HeavyComponent />
    </div>
  )
}
```

**策略二：減少 JavaScript 執行時間**

```tsx
// ❌ 不好：同步處理大量資料
function processData(items: any[]) {
  return items.map(item => heavyComputation(item))
}

// ✅ 好：使用 Web Worker
// workers/data-processor.ts
self.addEventListener('message', (e) => {
  const result = e.data.map((item: any) => heavyComputation(item))
  self.postMessage(result)
})

// 在組件中使用
useEffect(() => {
  const worker = new Worker(new URL('@/workers/data-processor', import.meta.url))

  worker.postMessage(largeDataset)
  worker.onmessage = (e) => {
    setProcessedData(e.data)
  }

  return () => worker.terminate()
}, [])
```

### 22.2.5 優化 CLS

**策略一：為媒體元素設定尺寸**

```tsx
// ❌ 不好：未設定尺寸
<img src="/image.jpg" alt="Image" />

// ✅ 好：使用 Next.js Image 組件自動處理
<Image
  src="/image.jpg"
  alt="Image"
  width={800}
  height={600}
  layout="responsive"
/>
```

**策略二：為動態內容預留空間**

```tsx
// components/AdPlaceholder.tsx
export default function AdPlaceholder() {
  return (
    <div
      className="ad-container"
      style={{
        minHeight: '250px', // 預留廣告空間
        backgroundColor: '#f0f0f0'
      }}
    >
      {/* 廣告內容 */}
    </div>
  )
}
```

**策略三：避免在現有內容上方插入內容**

```tsx
// ❌ 不好：橫幅會推開內容
function Page() {
  const [showBanner, setShowBanner] = useState(false)

  useEffect(() => {
    setTimeout(() => setShowBanner(true), 2000)
  }, [])

  return (
    <>
      {showBanner && <Banner />}
      <MainContent />
    </>
  )
}

// ✅ 好：使用固定定位
function Page() {
  const [showBanner, setShowBanner] = useState(false)

  useEffect(() => {
    setTimeout(() => setShowBanner(true), 2000)
  }, [])

  return (
    <>
      {showBanner && (
        <div className="fixed top-0 left-0 right-0 z-50">
          <Banner />
        </div>
      )}
      <MainContent />
    </>
  )
}
```

## 22.3 程式碼分割策略

### 22.3.1 路由層級的程式碼分割

Next.js 自動為每個頁面分割程式碼，無需額外配置。

### 22.3.2 組件層級的程式碼分割

```tsx
// pages/dashboard.tsx
import dynamic from 'next/dynamic'

// 動態載入大型圖表庫
const Chart = dynamic(() => import('react-chartjs-2').then(mod => mod.Line), {
  loading: () => <div>載入圖表中...</div>,
  ssr: false
})

// 條件性載入
const AdminPanel = dynamic(() => import('@/components/AdminPanel'), {
  ssr: false
})

export default function Dashboard({ user }: { user: User }) {
  return (
    <div>
      <h1>儀表板</h1>
      <Chart data={chartData} />

      {user.isAdmin && <AdminPanel />}
    </div>
  )
}
```

### 22.3.3 第三方套件優化

```tsx
// ❌ 不好：導入整個套件
import _ from 'lodash'

const result = _.debounce(fn, 300)

// ✅ 好：只導入需要的函數
import debounce from 'lodash/debounce'

const result = debounce(fn, 300)
```

**使用 Bundle Analyzer 分析**

```bash
npm install @next/bundle-analyzer
```

```js
// next.config.js
const withBundleAnalyzer = require('@next/bundle-analyzer')({
  enabled: process.env.ANALYZE === 'true',
})

module.exports = withBundleAnalyzer({
  // Next.js 配置
})
```

```bash
# 分析 bundle
ANALYZE=true npm run build
```

## 22.4 圖片優化

### 22.4.1 使用 Next.js Image 組件

```tsx
import Image from 'next/image'

// 基本用法
<Image
  src="/profile.jpg"
  alt="Profile"
  width={500}
  height={500}
/>

// 響應式圖片
<Image
  src="/banner.jpg"
  alt="Banner"
  layout="responsive"
  width={1200}
  height={400}
/>

// 填滿容器
<div style={{ position: 'relative', width: '100%', height: '400px' }}>
  <Image
    src="/background.jpg"
    alt="Background"
    layout="fill"
    objectFit="cover"
  />
</div>
```

### 22.4.2 圖片格式優化

```js
// next.config.js
module.exports = {
  images: {
    formats: ['image/avif', 'image/webp'], // 使用現代格式
    deviceSizes: [640, 750, 828, 1080, 1200, 1920, 2048, 3840],
    imageSizes: [16, 32, 48, 64, 96, 128, 256, 384],
    domains: ['example.com', 'cdn.example.com'], // 允許的外部圖片域名
  },
}
```

### 22.4.3 圖片懶載入

```tsx
// 預設情況下，Next.js Image 已啟用懶載入
<Image
  src="/image.jpg"
  alt="Lazy loaded"
  width={800}
  height={600}
  loading="lazy" // 預設值
/>

// 關鍵圖片使用 priority
<Image
  src="/hero.jpg"
  alt="Hero"
  width={1200}
  height={600}
  priority // 優先載入
/>
```

### 22.4.4 模糊佔位符

```tsx
import Image from 'next/image'

// 使用資料 URL
<Image
  src="/photo.jpg"
  alt="Photo"
  width={800}
  height={600}
  placeholder="blur"
  blurDataURL="data:image/jpeg;base64,/9j/4AAQSkZJRg..."
/>

// 動態生成模糊佔位符
import { getPlaiceholder } from 'plaiceholder'

export async function getStaticProps() {
  const { base64, img } = await getPlaiceholder('/photo.jpg')

  return {
    props: {
      imageProps: {
        ...img,
        blurDataURL: base64,
      },
    },
  }
}
```

## 22.5 字型優化

### 22.5.1 使用 next/font

```tsx
// app/layout.tsx (App Router) 或 pages/_app.tsx
import { Inter, Noto_Sans_TC } from 'next/font/google'

const inter = Inter({
  subsets: ['latin'],
  display: 'swap',
  variable: '--font-inter',
})

const notoSansTC = Noto_Sans_TC({
  subsets: ['chinese-traditional'],
  weight: ['400', '500', '700'],
  display: 'swap',
  variable: '--font-noto-sans-tc',
})

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="zh-TW" className={`${inter.variable} ${notoSansTC.variable}`}>
      <body>{children}</body>
    </html>
  )
}
```

### 22.5.2 本地字型優化

```tsx
// app/layout.tsx
import localFont from 'next/font/local'

const myFont = localFont({
  src: [
    {
      path: './fonts/MyFont-Regular.woff2',
      weight: '400',
      style: 'normal',
    },
    {
      path: './fonts/MyFont-Bold.woff2',
      weight: '700',
      style: 'normal',
    },
  ],
  variable: '--font-my-font',
  display: 'swap',
})

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="zh-TW" className={myFont.variable}>
      <body>{children}</body>
    </html>
  )
}
```

```css
/* globals.css */
body {
  font-family: var(--font-my-font), sans-serif;
}
```

### 22.5.3 字型載入策略

```tsx
// 預連接 Google Fonts
// pages/_document.tsx
import { Html, Head, Main, NextScript } from 'next/document'

export default function Document() {
  return (
    <Html>
      <Head>
        <link rel="preconnect" href="https://fonts.googleapis.com" />
        <link
          rel="preconnect"
          href="https://fonts.gstatic.com"
          crossOrigin="anonymous"
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

## 22.6 快取策略

### 22.6.1 HTTP 快取標頭

```js
// next.config.js
module.exports = {
  async headers() {
    return [
      {
        source: '/static/:path*',
        headers: [
          {
            key: 'Cache-Control',
            value: 'public, max-age=31536000, immutable',
          },
        ],
      },
      {
        source: '/:path*.jpg',
        headers: [
          {
            key: 'Cache-Control',
            value: 'public, max-age=86400, s-maxage=86400',
          },
        ],
      },
    ]
  },
}
```

### 22.6.2 API 路由快取

```tsx
// pages/api/posts.ts
import type { NextApiRequest, NextApiResponse } from 'next'

export default async function handler(
  req: NextApiRequest,
  res: NextApiResponse
) {
  // 設定快取標頭
  res.setHeader(
    'Cache-Control',
    'public, s-maxage=10, stale-while-revalidate=59'
  )

  const posts = await fetchPosts()

  res.status(200).json(posts)
}
```

### 22.6.3 使用 SWR 實現客戶端快取

```bash
npm install swr
```

```tsx
// hooks/usePosts.ts
import useSWR from 'swr'

const fetcher = (url: string) => fetch(url).then(r => r.json())

export function usePosts() {
  const { data, error, isLoading } = useSWR('/api/posts', fetcher, {
    revalidateOnFocus: false,
    revalidateOnReconnect: false,
    refreshInterval: 0,
    dedupingInterval: 60000, // 1 分鐘內不重複請求
  })

  return {
    posts: data,
    isLoading,
    isError: error
  }
}
```

### 22.6.4 ISR (Incremental Static Regeneration)

```tsx
// pages/products/[id].tsx
export const getStaticProps: GetStaticProps = async ({ params }) => {
  const product = await fetchProduct(params?.id as string)

  return {
    props: { product },
    revalidate: 60, // 每 60 秒重新生成一次
  }
}
```

## 22.7 Bundle 大小優化

### 22.7.1 移除未使用的依賴

```bash
# 檢查未使用的依賴
npm install -g depcheck
depcheck

# 移除未使用的依賴
npm uninstall [package-name]
```

### 22.7.2 Tree Shaking

```js
// next.config.js
module.exports = {
  webpack: (config, { isServer }) => {
    if (!isServer) {
      // Tree shaking 優化
      config.optimization.usedExports = true
    }
    return config
  },
}
```

### 22.7.3 使用 ES Modules

```tsx
// ❌ 不好：使用 CommonJS
const moment = require('moment')

// ✅ 好：使用 ES Modules（支援 tree shaking）
import { format } from 'date-fns'
```

### 22.7.4 動態導入第三方庫

```tsx
// 只在需要時載入
import { useState } from 'react'

export default function Editor() {
  const [Editor, setEditor] = useState<any>(null)

  const loadEditor = async () => {
    const { default: ReactQuill } = await import('react-quill')
    setEditor(() => ReactQuill)
  }

  return (
    <div>
      {!Editor && (
        <button onClick={loadEditor}>載入編輯器</button>
      )}
      {Editor && <Editor />}
    </div>
  )
}
```

### 22.7.5 模組化 CSS

```tsx
// ❌ 不好：全域導入
import 'bootstrap/dist/css/bootstrap.css'

// ✅ 好：只導入需要的樣式
import 'bootstrap/dist/css/bootstrap-grid.css'

// 或使用 CSS Modules
import styles from './Component.module.css'
```

## 22.8 運行時效能優化

### 22.8.1 使用 React.memo

```tsx
import { memo } from 'react'

interface ProductCardProps {
  product: Product
}

const ProductCard = memo(function ProductCard({ product }: ProductCardProps) {
  return (
    <div className="product-card">
      <h3>{product.name}</h3>
      <p>{product.price}</p>
    </div>
  )
})

export default ProductCard
```

### 22.8.2 使用 useMemo 和 useCallback

```tsx
import { useMemo, useCallback, useState } from 'react'

export default function ExpensiveList({ items }: { items: Item[] }) {
  const [filter, setFilter] = useState('')

  // 快取計算結果
  const filteredItems = useMemo(() => {
    return items.filter(item =>
      item.name.toLowerCase().includes(filter.toLowerCase())
    )
  }, [items, filter])

  // 快取回呼函數
  const handleFilterChange = useCallback((e: React.ChangeEvent<HTMLInputElement>) => {
    setFilter(e.target.value)
  }, [])

  return (
    <div>
      <input
        type="text"
        value={filter}
        onChange={handleFilterChange}
        placeholder="搜尋..."
      />
      <ul>
        {filteredItems.map(item => (
          <li key={item.id}>{item.name}</li>
        ))}
      </ul>
    </div>
  )
}
```

### 22.8.3 虛擬化長列表

```bash
npm install react-window
```

```tsx
import { FixedSizeList } from 'react-window'

interface RowProps {
  index: number
  style: React.CSSProperties
}

export default function VirtualList({ items }: { items: any[] }) {
  const Row = ({ index, style }: RowProps) => (
    <div style={style}>
      {items[index].name}
    </div>
  )

  return (
    <FixedSizeList
      height={600}
      itemCount={items.length}
      itemSize={50}
      width="100%"
    >
      {Row}
    </FixedSizeList>
  )
}
```

## 22.9 資料庫與 API 優化

### 22.9.1 資料庫查詢優化

```tsx
// ❌ 不好：N+1 查詢問題
async function getPostsWithAuthors() {
  const posts = await prisma.post.findMany()

  for (const post of posts) {
    post.author = await prisma.user.findUnique({
      where: { id: post.authorId }
    })
  }

  return posts
}

// ✅ 好：使用 include 一次查詢
async function getPostsWithAuthors() {
  return await prisma.post.findMany({
    include: {
      author: true
    }
  })
}
```

### 22.9.2 API 回應優化

```tsx
// pages/api/posts.ts
export default async function handler(req: NextApiRequest, res: NextApiResponse) {
  const posts = await prisma.post.findMany({
    select: {
      id: true,
      title: true,
      excerpt: true,
      // 只選擇需要的欄位，不傳送整個內容
      // content: false
    },
    take: 10, // 分頁
    orderBy: {
      createdAt: 'desc'
    }
  })

  // 壓縮回應
  res.setHeader('Content-Encoding', 'gzip')
  res.status(200).json(posts)
}
```

### 22.9.3 使用 API 路由快取

```tsx
// lib/cache.ts
const cache = new Map()

export function getCached<T>(key: string, fetcher: () => Promise<T>, ttl = 60000): Promise<T> {
  const cached = cache.get(key)

  if (cached && Date.now() - cached.timestamp < ttl) {
    return Promise.resolve(cached.value)
  }

  return fetcher().then(value => {
    cache.set(key, { value, timestamp: Date.now() })
    return value
  })
}

// pages/api/posts.ts
import { getCached } from '@/lib/cache'

export default async function handler(req: NextApiRequest, res: NextApiResponse) {
  const posts = await getCached(
    'all-posts',
    () => prisma.post.findMany(),
    60000 // 快取 1 分鐘
  )

  res.status(200).json(posts)
}
```

## 22.10 監控與分析

### 22.10.1 使用 Lighthouse

```bash
# 安裝 Lighthouse CLI
npm install -g lighthouse

# 執行分析
lighthouse https://your-site.com --view
```

### 22.10.2 Next.js Analytics

```tsx
// pages/_app.tsx
import { Analytics } from '@vercel/analytics/react'

function MyApp({ Component, pageProps }: AppProps) {
  return (
    <>
      <Component {...pageProps} />
      <Analytics />
    </>
  )
}
```

### 22.10.3 自訂效能監控

```tsx
// lib/performance.ts
export function measurePerformance(name: string, fn: () => void) {
  const start = performance.now()
  fn()
  const end = performance.now()

  console.log(`${name} took ${end - start}ms`)

  // 發送到分析服務
  if (typeof window !== 'undefined' && window.gtag) {
    window.gtag('event', 'timing_complete', {
      name: name,
      value: Math.round(end - start),
      event_category: 'Performance'
    })
  }
}
```

## 22.11 效能優化檢查清單

### 建置時優化
- ✅ 使用 SSG 而非 SSR（當可行時）
- ✅ 啟用 ISR 進行動態內容
- ✅ 分析並優化 bundle 大小
- ✅ 移除未使用的依賴和程式碼
- ✅ 使用正確的圖片格式（WebP、AVIF）

### 運行時優化
- ✅ 使用 React.memo、useMemo、useCallback
- ✅ 實作虛擬化處理長列表
- ✅ 避免不必要的重新渲染
- ✅ 使用 Web Workers 處理密集運算

### 資源優化
- ✅ 使用 next/image 優化圖片
- ✅ 使用 next/font 優化字型
- ✅ 懶載入非關鍵組件和資源
- ✅ 預載入關鍵資源

### 網路優化
- ✅ 設定適當的快取標頭
- ✅ 啟用 CDN
- ✅ 壓縮靜態資源
- ✅ 使用 HTTP/2 或 HTTP/3

## 22.12 總結

效能優化是一個持續的過程，需要定期監控和調整。關鍵要點：

1. **測量第一**：使用工具測量效能，找出瓶頸
2. **優先處理關鍵問題**：專注於影響 Core Web Vitals 的問題
3. **漸進式優化**：不要一次做太多改變
4. **持續監控**：使用 Analytics 和 Lighthouse 定期檢查

透過本章介紹的技術，您可以大幅提升 Next.js 應用程式的效能，提供更好的使用者體驗。
