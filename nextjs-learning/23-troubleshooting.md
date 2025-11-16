# 第 23 章：常見問題與除錯技巧

## 23.1 除錯工具概覽

### 23.1.1 瀏覽器開發者工具

Next.js 應用程式可以使用標準的瀏覽器開發者工具進行除錯：

- **Chrome DevTools**
- **Firefox Developer Tools**
- **Safari Web Inspector**
- **Edge DevTools**

### 23.1.2 VS Code 除錯配置

```json
// .vscode/launch.json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Next.js: debug server-side",
      "type": "node-terminal",
      "request": "launch",
      "command": "npm run dev"
    },
    {
      "name": "Next.js: debug client-side",
      "type": "chrome",
      "request": "launch",
      "url": "http://localhost:3000"
    },
    {
      "name": "Next.js: debug full stack",
      "type": "node-terminal",
      "request": "launch",
      "command": "npm run dev",
      "serverReadyAction": {
        "pattern": "started server on .+, url: (https?://.+)",
        "uriFormat": "%s",
        "action": "debugWithChrome"
      }
    }
  ]
}
```

## 23.2 常見錯誤與解決方案

### 23.2.1 Hydration 錯誤

**錯誤訊息：**
```
Error: Hydration failed because the initial UI does not match what was rendered on the server.
```

**常見原因與解決方案：**

**原因 1：伺服器和客戶端渲染內容不一致**

```tsx
// ❌ 錯誤示範
export default function Component() {
  // Date.now() 在伺服器和客戶端會產生不同值
  return <div>{Date.now()}</div>
}

// ✅ 正確做法：使用 useEffect
import { useState, useEffect } from 'react'

export default function Component() {
  const [timestamp, setTimestamp] = useState<number | null>(null)

  useEffect(() => {
    setTimestamp(Date.now())
  }, [])

  return <div>{timestamp ?? 'Loading...'}</div>
}
```

**原因 2：使用瀏覽器專用 API**

```tsx
// ❌ 錯誤示範
export default function Component() {
  const width = window.innerWidth // SSR 時 window 不存在
  return <div>Width: {width}</div>
}

// ✅ 正確做法
import { useState, useEffect } from 'react'

export default function Component() {
  const [width, setWidth] = useState<number>(0)

  useEffect(() => {
    setWidth(window.innerWidth)

    const handleResize = () => setWidth(window.innerWidth)
    window.addEventListener('resize', handleResize)

    return () => window.removeEventListener('resize', handleResize)
  }, [])

  return <div>Width: {width}</div>
}
```

**原因 3：HTML 巢狀結構錯誤**

```tsx
// ❌ 錯誤：<p> 內不能包含 <div>
export default function Component() {
  return (
    <p>
      <div>This is invalid HTML</div>
    </p>
  )
}

// ✅ 正確做法
export default function Component() {
  return (
    <div>
      <p>This is valid HTML</p>
    </div>
  )
}
```

### 23.2.2 環境變數問題

**錯誤：環境變數在客戶端為 undefined**

```tsx
// ❌ 錯誤：未加 NEXT_PUBLIC_ 前綴
// .env.local
API_KEY=abc123

// 在客戶端組件中
console.log(process.env.API_KEY) // undefined

// ✅ 正確做法
// .env.local
NEXT_PUBLIC_API_KEY=abc123

// 在客戶端組件中
console.log(process.env.NEXT_PUBLIC_API_KEY) // 'abc123'
```

**安全處理敏感資料**

```tsx
// pages/api/data.ts
export default async function handler(req: NextApiRequest, res: NextApiResponse) {
  // 伺服器端可以安全使用私密環境變數
  const apiKey = process.env.SECRET_API_KEY

  const data = await fetch('https://api.example.com/data', {
    headers: {
      'Authorization': `Bearer ${apiKey}`
    }
  })

  res.status(200).json(await data.json())
}
```

### 23.2.3 動態路由問題

**錯誤：getStaticPaths 返回空路徑**

```tsx
// ❌ 錯誤
export const getStaticPaths: GetStaticPaths = async () => {
  return {
    paths: [], // 空陣列會導致所有頁面 404
    fallback: false
  }
}

// ✅ 正確做法
export const getStaticPaths: GetStaticPaths = async () => {
  const posts = await getAllPosts()

  return {
    paths: posts.map(post => ({
      params: { slug: post.slug }
    })),
    fallback: 'blocking' // 或 true，允許動態生成新頁面
  }
}
```

### 23.2.4 API 路由 CORS 問題

```tsx
// pages/api/data.ts
import type { NextApiRequest, NextApiResponse } from 'next'

export default async function handler(
  req: NextApiRequest,
  res: NextApiResponse
) {
  // 設定 CORS 標頭
  res.setHeader('Access-Control-Allow-Credentials', 'true')
  res.setHeader('Access-Control-Allow-Origin', '*')
  res.setHeader('Access-Control-Allow-Methods', 'GET,OPTIONS,PATCH,DELETE,POST,PUT')
  res.setHeader(
    'Access-Control-Allow-Headers',
    'X-CSRF-Token, X-Requested-With, Accept, Accept-Version, Content-Length, Content-MD5, Content-Type, Date, X-Api-Version'
  )

  // 處理 OPTIONS 請求
  if (req.method === 'OPTIONS') {
    res.status(200).end()
    return
  }

  // 處理實際請求
  res.status(200).json({ message: 'Success' })
}
```

### 23.2.5 圖片載入失敗

**錯誤：Invalid src prop**

```tsx
// ❌ 錯誤：使用外部 URL 但未配置 domains
<Image src="https://example.com/image.jpg" width={500} height={500} alt="Image" />

// ✅ 正確做法：配置允許的域名
// next.config.js
module.exports = {
  images: {
    domains: ['example.com', 'cdn.example.com'],
    // 或使用 remotePatterns (更靈活)
    remotePatterns: [
      {
        protocol: 'https',
        hostname: '**.example.com',
      },
    ],
  },
}
```

### 23.2.6 字型閃爍問題 (FOUT/FOIT)

```tsx
// ❌ 問題：直接使用 Google Fonts 連結
// pages/_document.tsx
<Head>
  <link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;700&display=swap" rel="stylesheet" />
</Head>

// ✅ 解決方案：使用 next/font
import { Inter } from 'next/font/google'

const inter = Inter({
  subsets: ['latin'],
  display: 'swap', // 避免不可見文字閃爍
  preload: true,
})

export default function MyApp({ Component, pageProps }: AppProps) {
  return (
    <main className={inter.className}>
      <Component {...pageProps} />
    </main>
  )
}
```

## 23.3 從 Vue 3 遷移的常見陷阱

### 23.3.1 響應式系統差異

**Vue 3 vs Next.js/React**

```typescript
// Vue 3 - 使用 ref 和 reactive
import { ref, reactive } from 'vue'

const count = ref(0)
const state = reactive({ name: 'John' })

function increment() {
  count.value++ // 需要使用 .value
}

// React - 使用 useState
import { useState } from 'react'

const [count, setCount] = useState(0)
const [state, setState] = useState({ name: 'John' })

function increment() {
  setCount(count + 1) // 不可直接修改，必須使用 setter
}
```

### 23.3.2 模板語法差異

```vue
<!-- Vue 3 -->
<template>
  <div v-if="isVisible">
    <p v-for="item in items" :key="item.id">
      {{ item.name }}
    </p>
  </div>
</template>
```

```tsx
// React/Next.js - 使用 JSX
export default function Component({ items, isVisible }: Props) {
  return (
    <>
      {isVisible && (
        <div>
          {items.map(item => (
            <p key={item.id}>
              {item.name}
            </p>
          ))}
        </div>
      )}
    </>
  )
}
```

### 23.3.3 生命週期差異

```typescript
// Vue 3
import { onMounted, onUnmounted } from 'vue'

onMounted(() => {
  console.log('組件已掛載')
})

onUnmounted(() => {
  console.log('組件已卸載')
})

// React/Next.js
import { useEffect } from 'react'

useEffect(() => {
  console.log('組件已掛載')

  return () => {
    console.log('組件已卸載')
  }
}, []) // 空依賴陣列 = 只執行一次
```

### 23.3.4 事件處理差異

```vue
<!-- Vue 3 -->
<template>
  <button @click="handleClick">
    Click me
  </button>
</template>

<script setup>
const handleClick = (event) => {
  console.log(event)
}
</script>
```

```tsx
// React/Next.js
export default function Component() {
  const handleClick = (event: React.MouseEvent<HTMLButtonElement>) => {
    console.log(event)
  }

  return (
    <button onClick={handleClick}>
      Click me
    </button>
  )
}
```

### 23.3.5 Props 和狀態管理

```typescript
// Vue 3 - defineProps
<script setup lang="ts">
const props = defineProps<{
  title: string
  count: number
}>()
</script>

// React/Next.js - 使用介面
interface ComponentProps {
  title: string
  count: number
}

export default function Component({ title, count }: ComponentProps) {
  return <div>{title}: {count}</div>
}
```

### 23.3.6 雙向綁定差異

```vue
<!-- Vue 3 - v-model -->
<template>
  <input v-model="text" />
</template>

<script setup>
import { ref } from 'vue'
const text = ref('')
</script>
```

```tsx
// React/Next.js - 受控組件
import { useState } from 'react'

export default function Component() {
  const [text, setText] = useState('')

  return (
    <input
      value={text}
      onChange={(e) => setText(e.target.value)}
    />
  )
}
```

## 23.4 效能問題除錯

### 23.4.1 識別重新渲染問題

```tsx
// 安裝 React DevTools Profiler
// 使用 console.log 追蹤渲染

import { useEffect } from 'react'

export default function Component({ prop }: { prop: string }) {
  console.log('Component rendered', prop)

  useEffect(() => {
    console.log('Component mounted or updated')
  })

  return <div>{prop}</div>
}
```

**使用 React DevTools Profiler**

1. 安裝 React DevTools 瀏覽器擴充功能
2. 開啟 Profiler 標籤
3. 點擊 Record 開始錄製
4. 執行操作
5. 停止錄製並分析火焰圖

### 23.4.2 記憶體洩漏偵測

```tsx
// ❌ 記憶體洩漏：未清理事件監聽器
import { useEffect } from 'react'

export default function Component() {
  useEffect(() => {
    const handleScroll = () => console.log('scrolling')
    window.addEventListener('scroll', handleScroll)

    // 缺少清理函數！
  }, [])

  return <div>Component</div>
}

// ✅ 正確：清理事件監聽器
import { useEffect } from 'react'

export default function Component() {
  useEffect(() => {
    const handleScroll = () => console.log('scrolling')
    window.addEventListener('scroll', handleScroll)

    return () => {
      window.removeEventListener('scroll', handleScroll)
    }
  }, [])

  return <div>Component</div>
}
```

### 23.4.3 使用 Performance API

```tsx
// lib/performance-monitor.ts
export function measureComponentRender(componentName: string) {
  if (typeof window === 'undefined') return

  return {
    start: () => {
      performance.mark(`${componentName}-start`)
    },
    end: () => {
      performance.mark(`${componentName}-end`)
      performance.measure(
        componentName,
        `${componentName}-start`,
        `${componentName}-end`
      )

      const measure = performance.getEntriesByName(componentName)[0]
      console.log(`${componentName} render time:`, measure.duration, 'ms')

      // 清理
      performance.clearMarks(`${componentName}-start`)
      performance.clearMarks(`${componentName}-end`)
      performance.clearMeasures(componentName)
    }
  }
}

// 使用
import { useEffect } from 'react'
import { measureComponentRender } from '@/lib/performance-monitor'

export default function ExpensiveComponent() {
  useEffect(() => {
    const perf = measureComponentRender('ExpensiveComponent')
    perf.start()

    // 組件渲染邏輯...

    perf.end()
  })

  return <div>Expensive Component</div>
}
```

## 23.5 網路請求除錯

### 23.5.1 API 請求失敗處理

```tsx
// lib/api-client.ts
export async function fetchAPI<T>(
  url: string,
  options?: RequestInit
): Promise<T> {
  try {
    const response = await fetch(url, {
      ...options,
      headers: {
        'Content-Type': 'application/json',
        ...options?.headers,
      },
    })

    if (!response.ok) {
      // 詳細錯誤處理
      const error = await response.json().catch(() => ({
        message: response.statusText
      }))

      throw new Error(
        `API Error (${response.status}): ${error.message || '未知錯誤'}`
      )
    }

    return await response.json()
  } catch (error) {
    // 網路錯誤
    if (error instanceof TypeError) {
      console.error('Network error:', error)
      throw new Error('網路連線失敗，請檢查網路連線')
    }

    // 其他錯誤
    console.error('API request failed:', error)
    throw error
  }
}
```

### 23.5.2 請求超時處理

```tsx
// lib/fetch-with-timeout.ts
export async function fetchWithTimeout(
  url: string,
  options: RequestInit = {},
  timeout = 10000
) {
  const controller = new AbortController()
  const timeoutId = setTimeout(() => controller.abort(), timeout)

  try {
    const response = await fetch(url, {
      ...options,
      signal: controller.signal
    })

    clearTimeout(timeoutId)
    return response
  } catch (error) {
    clearTimeout(timeoutId)

    if (error instanceof Error && error.name === 'AbortError') {
      throw new Error(`請求超時（${timeout}ms）`)
    }

    throw error
  }
}
```

### 23.5.3 請求重試邏輯

```tsx
// lib/retry-fetch.ts
export async function retryFetch<T>(
  fetcher: () => Promise<T>,
  maxRetries = 3,
  delay = 1000
): Promise<T> {
  let lastError: Error | null = null

  for (let i = 0; i < maxRetries; i++) {
    try {
      return await fetcher()
    } catch (error) {
      lastError = error as Error
      console.warn(`請求失敗，重試 ${i + 1}/${maxRetries}`)

      if (i < maxRetries - 1) {
        // 指數退避
        await new Promise(resolve =>
          setTimeout(resolve, delay * Math.pow(2, i))
        )
      }
    }
  }

  throw lastError || new Error('請求失敗')
}

// 使用
const data = await retryFetch(
  () => fetch('/api/data').then(r => r.json()),
  3, // 最多重試 3 次
  1000 // 初始延遲 1 秒
)
```

## 23.6 除錯工具和技巧

### 23.6.1 使用 next.config.js 啟用詳細日誌

```js
// next.config.js
module.exports = {
  // 顯示詳細的建置資訊
  productionBrowserSourceMaps: true,

  // 自訂 webpack 配置以顯示更多資訊
  webpack: (config, { dev, isServer }) => {
    if (dev) {
      config.devtool = 'eval-source-map'
    }

    return config
  },
}
```

### 23.6.2 環境特定的除錯

```tsx
// lib/debug.ts
export const isDev = process.env.NODE_ENV === 'development'

export function debugLog(...args: any[]) {
  if (isDev) {
    console.log('[DEBUG]', ...args)
  }
}

export function debugError(...args: any[]) {
  if (isDev) {
    console.error('[ERROR]', ...args)
  }
}

// 使用
import { debugLog } from '@/lib/debug'

export default function Component() {
  debugLog('Component rendered')
  return <div>Component</div>
}
```

### 23.6.3 React Query DevTools

```bash
npm install @tanstack/react-query-devtools
```

```tsx
// pages/_app.tsx
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'
import { ReactQueryDevtools } from '@tanstack/react-query-devtools'

const queryClient = new QueryClient()

export default function MyApp({ Component, pageProps }: AppProps) {
  return (
    <QueryClientProvider client={queryClient}>
      <Component {...pageProps} />
      <ReactQueryDevtools initialIsOpen={false} />
    </QueryClientProvider>
  )
}
```

### 23.6.4 錯誤邊界（Error Boundaries）

```tsx
// components/ErrorBoundary.tsx
import React, { Component, ErrorInfo, ReactNode } from 'react'

interface Props {
  children: ReactNode
  fallback?: ReactNode
}

interface State {
  hasError: boolean
  error: Error | null
}

export class ErrorBoundary extends Component<Props, State> {
  public state: State = {
    hasError: false,
    error: null
  }

  public static getDerivedStateFromError(error: Error): State {
    return { hasError: true, error }
  }

  public componentDidCatch(error: Error, errorInfo: ErrorInfo) {
    console.error('Uncaught error:', error, errorInfo)

    // 發送錯誤到監控服務（如 Sentry）
    if (typeof window !== 'undefined') {
      // window.Sentry?.captureException(error)
    }
  }

  public render() {
    if (this.state.hasError) {
      return this.props.fallback || (
        <div className="error-boundary">
          <h2>發生錯誤</h2>
          <details style={{ whiteSpace: 'pre-wrap' }}>
            {this.state.error?.toString()}
          </details>
          <button onClick={() => this.setState({ hasError: false, error: null })}>
            重試
          </button>
        </div>
      )
    }

    return this.props.children
  }
}

// 使用
export default function Page() {
  return (
    <ErrorBoundary fallback={<div>出錯了！</div>}>
      <MyComponent />
    </ErrorBoundary>
  )
}
```

## 23.7 效能監控工具

### 23.7.1 整合 Sentry

```bash
npm install @sentry/nextjs
```

```js
// sentry.client.config.js
import * as Sentry from '@sentry/nextjs'

Sentry.init({
  dsn: process.env.NEXT_PUBLIC_SENTRY_DSN,
  tracesSampleRate: 1.0,
  environment: process.env.NODE_ENV,
})

// sentry.server.config.js
import * as Sentry from '@sentry/nextjs'

Sentry.init({
  dsn: process.env.NEXT_PUBLIC_SENTRY_DSN,
  tracesSampleRate: 1.0,
})
```

### 23.7.2 自訂錯誤報告

```tsx
// lib/error-reporter.ts
export function reportError(error: Error, context?: Record<string, any>) {
  // 開發環境只在 console 顯示
  if (process.env.NODE_ENV === 'development') {
    console.error('Error:', error)
    console.error('Context:', context)
    return
  }

  // 生產環境發送到監控服務
  if (typeof window !== 'undefined' && window.Sentry) {
    window.Sentry.captureException(error, { extra: context })
  }

  // 或發送到自訂 API
  fetch('/api/error-log', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      message: error.message,
      stack: error.stack,
      context,
      timestamp: new Date().toISOString()
    })
  }).catch(console.error)
}
```

### 23.7.3 效能追蹤

```tsx
// lib/performance-tracker.ts
export class PerformanceTracker {
  private marks: Map<string, number> = new Map()

  start(label: string) {
    this.marks.set(label, performance.now())
  }

  end(label: string) {
    const startTime = this.marks.get(label)
    if (!startTime) {
      console.warn(`No start time found for: ${label}`)
      return
    }

    const duration = performance.now() - startTime
    this.marks.delete(label)

    // 記錄效能資料
    console.log(`[Performance] ${label}: ${duration.toFixed(2)}ms`)

    // 發送到分析服務
    if (typeof window !== 'undefined' && window.gtag) {
      window.gtag('event', 'timing_complete', {
        name: label,
        value: Math.round(duration),
        event_category: 'Performance'
      })
    }

    return duration
  }
}

export const perfTracker = new PerformanceTracker()
```

## 23.8 常見建置錯誤

### 23.8.1 TypeScript 型別錯誤

```bash
# 嚴格的 TypeScript 檢查
# tsconfig.json
{
  "compilerOptions": {
    "strict": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noImplicitReturns": true
  }
}
```

### 23.8.2 ESLint 錯誤

```bash
# 執行 ESLint 檢查
npm run lint

# 自動修復
npm run lint -- --fix
```

### 23.8.3 建置超時

```js
// next.config.js
module.exports = {
  // 增加建置超時時間
  staticPageGenerationTimeout: 180, // 預設 60 秒

  // 限制並發建置數量
  experimental: {
    workerThreads: false,
    cpus: 1
  }
}
```

## 23.9 除錯最佳實踐

### 23.9.1 日誌最佳實踐

```tsx
// lib/logger.ts
enum LogLevel {
  DEBUG = 'DEBUG',
  INFO = 'INFO',
  WARN = 'WARN',
  ERROR = 'ERROR'
}

class Logger {
  private shouldLog(level: LogLevel): boolean {
    if (process.env.NODE_ENV === 'production') {
      return level === LogLevel.ERROR || level === LogLevel.WARN
    }
    return true
  }

  debug(...args: any[]) {
    if (this.shouldLog(LogLevel.DEBUG)) {
      console.log(`[${LogLevel.DEBUG}]`, ...args)
    }
  }

  info(...args: any[]) {
    if (this.shouldLog(LogLevel.INFO)) {
      console.info(`[${LogLevel.INFO}]`, ...args)
    }
  }

  warn(...args: any[]) {
    if (this.shouldLog(LogLevel.WARN)) {
      console.warn(`[${LogLevel.WARN}]`, ...args)
    }
  }

  error(...args: any[]) {
    if (this.shouldLog(LogLevel.ERROR)) {
      console.error(`[${LogLevel.ERROR}]`, ...args)
    }
  }
}

export const logger = new Logger()
```

### 23.9.2 條件式除錯

```tsx
// 使用環境變數控制除錯輸出
const DEBUG = process.env.NEXT_PUBLIC_DEBUG === 'true'

export default function Component() {
  if (DEBUG) {
    console.log('Component props:', props)
  }

  return <div>Component</div>
}
```

### 23.9.3 Source Maps

```js
// next.config.js
module.exports = {
  // 生產環境啟用 source maps（謹慎使用，會增加 bundle 大小）
  productionBrowserSourceMaps: true,
}
```

## 23.10 除錯檢查清單

### 開發階段
- ✅ 使用 TypeScript 進行型別檢查
- ✅ 配置 ESLint 和 Prettier
- ✅ 使用 React DevTools 和 Next.js DevTools
- ✅ 實作錯誤邊界
- ✅ 添加適當的日誌

### 測試階段
- ✅ 單元測試覆蓋關鍵邏輯
- ✅ 整合測試 API 路由
- ✅ E2E 測試關鍵流程
- ✅ 效能測試（Lighthouse）
- ✅ 跨瀏覽器測試

### 生產階段
- ✅ 配置錯誤監控（Sentry）
- ✅ 效能監控（Vercel Analytics）
- ✅ 日誌聚合和分析
- ✅ 設定告警機制
- ✅ 定期審查錯誤報告

## 23.11 總結

除錯是開發過程中不可或缺的一部分。掌握以下技能可以提高除錯效率：

1. **熟悉工具**：善用瀏覽器開發者工具、React DevTools、VS Code 除錯器
2. **理解錯誤訊息**：仔細閱讀錯誤堆疊，理解問題根源
3. **系統化方法**：使用二分法、日誌、斷點等系統化除錯
4. **預防勝於治療**：使用 TypeScript、ESLint、測試來預防錯誤
5. **監控和追蹤**：在生產環境使用監控工具及時發現問題

透過本章介紹的技巧和工具，您可以更有效地識別和解決 Next.js 應用程式中的問題。
