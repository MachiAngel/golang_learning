# 15. Plugins 插件開發與使用

## 目錄
- [Plugin 系統介紹](#plugin-系統介紹)
- [plugins/ 目錄](#plugins-目錄)
- [建立 Plugin](#建立-plugin)
- [Plugin 註冊與使用](#plugin-註冊與使用)
- [nuxtApp 物件](#nuxtapp-物件)
- [provide/inject 機制](#provideinject-機制)
- [客戶端專用 Plugin](#客戶端專用-plugin)
- [伺服器端專用 Plugin](#伺服器端專用-plugin)
- [第三方套件整合](#第三方套件整合)
- [Plugin 執行順序](#plugin-執行順序)
- [完整範例](#完整範例)

---

## Plugin 系統介紹

Plugins 是 Nuxt.js 應用啟動時自動執行的函式，用於擴展應用功能。

### 主要用途

1. **註冊全域組件** - 讓組件在整個應用中可用
2. **提供工具函式** - 如 API 客戶端、日期格式化等
3. **整合第三方庫** - 如 Axios、Chart.js、Day.js
4. **設定全域指令** - Vue directives
5. **初始化服務** - 如分析工具、錯誤追蹤
6. **擴展 Vue 實例** - 添加全域屬性和方法

### 執行時機

- **應用啟動時執行一次**
- 在路由渲染之前執行
- 支援異步初始化
- 可以在伺服器端和/或客戶端執行

---

## plugins/ 目錄

### 目錄結構

```
plugins/
├── 01.init.ts              # 優先執行（數字前綴控制順序）
├── 02.api.ts               # API 客戶端
├── directives.ts           # 自訂指令
├── components.global.ts    # 全域組件註冊
├── analytics.client.ts     # 僅客戶端（.client 後綴）
├── logger.server.ts        # 僅伺服器端（.server 後綴）
├── vuetify.ts             # UI 框架
└── dayjs.ts               # 日期處理
```

### 命名規則

- **普通 Plugin**: `filename.ts` - 伺服器端和客戶端都執行
- **客戶端 Plugin**: `filename.client.ts` - 僅在客戶端執行
- **伺服器端 Plugin**: `filename.server.ts` - 僅在伺服器端執行
- **執行順序**: 使用數字前綴如 `01.`, `02.` 控制順序

---

## 建立 Plugin

### 基本語法

```typescript
// plugins/hello.ts
export default defineNuxtPlugin((nuxtApp) => {
  // Plugin 邏輯
  console.log('Hello from plugin!')

  // 返回提供的內容
  return {
    provide: {
      hello: (name: string) => `Hello ${name}!`
    }
  }
})
```

### 使用 Plugin

```vue
<script setup>
// 使用 $ 前綴訪問
const { $hello } = useNuxtApp()

console.log($hello('World'))  // "Hello World!"
</script>
```

### 異步 Plugin

```typescript
// plugins/async-init.ts
export default defineNuxtPlugin(async (nuxtApp) => {
  // 異步初始化
  const config = await $fetch('/api/config')

  return {
    provide: {
      config
    }
  }
})
```

---

## Plugin 註冊與使用

### 自動註冊

Nuxt 會自動掃描 `plugins/` 目錄並註冊所有 Plugin，無需手動配置。

### 手動控制（可選）

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  plugins: [
    // 明確指定 Plugin 和執行環境
    { src: '~/plugins/analytics.ts', mode: 'client' },
    { src: '~/plugins/logger.ts', mode: 'server' }
  ]
})
```

### 條件載入

```typescript
// plugins/admin-tools.ts
export default defineNuxtPlugin((nuxtApp) => {
  const user = useState('user')

  // 只為管理員載入
  if (user.value?.role !== 'admin') {
    return
  }

  return {
    provide: {
      adminTools: {
        // 管理員工具
      }
    }
  }
})
```

---

## nuxtApp 物件

`nuxtApp` 是 Nuxt 應用的核心實例，提供豐富的 API。

### nuxtApp 屬性和方法

```typescript
// plugins/nuxt-app-demo.ts
export default defineNuxtPlugin((nuxtApp) => {
  // 1. Vue 應用實例
  nuxtApp.vueApp  // Vue app instance

  // 2. SSR 上下文
  nuxtApp.ssrContext  // 伺服器端渲染上下文

  // 3. Payload
  nuxtApp.payload  // SSR 數據負載

  // 4. 提供值
  nuxtApp.provide('myValue', 'Hello')

  // 5. Hooks
  nuxtApp.hook('page:start', () => {
    console.log('頁面開始載入')
  })

  nuxtApp.hook('page:finish', () => {
    console.log('頁面載入完成')
  })

  // 6. $fetch
  nuxtApp.$fetch  // 內建的 fetch 實例
})
```

### 訪問 nuxtApp

```vue
<script setup>
// 在組件中使用
const nuxtApp = useNuxtApp()

// 訪問提供的值
const myValue = nuxtApp.$myValue

// 調用 hooks
nuxtApp.hook('page:start', () => {
  console.log('頁面開始')
})
</script>
```

### 實用 Hooks

```typescript
// plugins/page-tracking.client.ts
export default defineNuxtPlugin((nuxtApp) => {
  nuxtApp.hook('page:start', () => {
    console.log('頁面開始載入')
  })

  nuxtApp.hook('page:finish', () => {
    console.log('頁面載入完成')
  })

  nuxtApp.hook('page:transition:finish', () => {
    console.log('頁面轉場完成')
  })

  nuxtApp.hook('app:error', (error) => {
    console.error('應用錯誤:', error)
  })

  nuxtApp.hook('vue:error', (error, instance, info) => {
    console.error('Vue 錯誤:', error, info)
  })
})
```

---

## provide/inject 機制

### 提供值 (Provide)

```typescript
// plugins/utils.ts
export default defineNuxtPlugin((nuxtApp) => {
  // 方式一：使用 return
  return {
    provide: {
      formatDate: (date: Date) => {
        return new Intl.DateTimeFormat('zh-TW').format(date)
      },
      formatCurrency: (amount: number) => {
        return new Intl.NumberFormat('zh-TW', {
          style: 'currency',
          currency: 'TWD'
        }).format(amount)
      }
    }
  }

  // 方式二：使用 nuxtApp.provide()
  // nuxtApp.provide('formatDate', (date: Date) => { ... })
})
```

### 注入使用 (Inject)

```vue
<script setup>
// 方式一：使用 useNuxtApp
const { $formatDate, $formatCurrency } = useNuxtApp()

const now = new Date()
const price = 1999

console.log($formatDate(now))         // "2025/11/16"
console.log($formatCurrency(price))   // "NT$1,999"
</script>

<template>
  <div>
    <p>日期: {{ $formatDate(new Date()) }}</p>
    <p>價格: {{ $formatCurrency(1999) }}</p>
  </div>
</template>
```

### TypeScript 類型支援

```typescript
// types/nuxt.d.ts
declare module '#app' {
  interface NuxtApp {
    $formatDate(date: Date): string
    $formatCurrency(amount: number): string
  }
}

declare module 'vue' {
  interface ComponentCustomProperties {
    $formatDate(date: Date): string
    $formatCurrency(amount: number): string
  }
}

export {}
```

---

## 客戶端專用 Plugin

客戶端 Plugin 僅在瀏覽器中執行，適合瀏覽器專用的功能。

### 使用 .client.ts 後綴

```typescript
// plugins/analytics.client.ts
export default defineNuxtPlugin((nuxtApp) => {
  // 初始化 Google Analytics
  const gaId = 'GA-XXXXX'

  // 載入 GA 腳本
  const script = document.createElement('script')
  script.async = true
  script.src = `https://www.googletagmanager.com/gtag/js?id=${gaId}`
  document.head.appendChild(script)

  // 初始化 GA
  window.dataLayer = window.dataLayer || []
  function gtag(...args: any[]) {
    window.dataLayer.push(args)
  }
  gtag('js', new Date())
  gtag('config', gaId)

  // 追蹤頁面瀏覽
  nuxtApp.hook('page:finish', () => {
    gtag('event', 'page_view', {
      page_path: window.location.pathname
    })
  })

  return {
    provide: {
      gtag
    }
  }
})
```

### LocalStorage Plugin

```typescript
// plugins/storage.client.ts
export default defineNuxtPlugin(() => {
  const storage = {
    get(key: string) {
      try {
        const item = localStorage.getItem(key)
        return item ? JSON.parse(item) : null
      } catch {
        return null
      }
    },

    set(key: string, value: any) {
      try {
        localStorage.setItem(key, JSON.stringify(value))
        return true
      } catch {
        return false
      }
    },

    remove(key: string) {
      try {
        localStorage.removeItem(key)
        return true
      } catch {
        return false
      }
    },

    clear() {
      try {
        localStorage.clear()
        return true
      } catch {
        return false
      }
    }
  }

  return {
    provide: {
      storage
    }
  }
})
```

### 使用範例

```vue
<script setup>
const { $gtag, $storage } = useNuxtApp()

// 追蹤事件
const trackClick = () => {
  $gtag('event', 'button_click', {
    event_category: 'engagement',
    event_label: 'CTA Button'
  })
}

// 使用 storage
const savePreference = () => {
  $storage.set('theme', 'dark')
}

const preference = $storage.get('theme')
</script>
```

---

## 伺服器端專用 Plugin

伺服器端 Plugin 僅在 SSR 時執行，適合伺服器端邏輯。

### 使用 .server.ts 後綴

```typescript
// plugins/logger.server.ts
export default defineNuxtPlugin((nuxtApp) => {
  const logger = {
    info(message: string, meta?: any) {
      console.log(`[INFO] ${new Date().toISOString()} - ${message}`, meta || '')
    },

    error(message: string, error?: Error) {
      console.error(`[ERROR] ${new Date().toISOString()} - ${message}`, error || '')
    },

    warn(message: string, meta?: any) {
      console.warn(`[WARN] ${new Date().toISOString()} - ${message}`, meta || '')
    }
  }

  // 記錄每個請求
  if (nuxtApp.ssrContext) {
    const req = nuxtApp.ssrContext.event.node.req
    logger.info(`SSR Request: ${req.method} ${req.url}`)
  }

  return {
    provide: {
      logger
    }
  }
})
```

### 資料庫連接 Plugin

```typescript
// plugins/database.server.ts
import { PrismaClient } from '@prisma/client'

let prisma: PrismaClient

export default defineNuxtPlugin(() => {
  if (!prisma) {
    prisma = new PrismaClient({
      log: ['query', 'info', 'warn', 'error']
    })
  }

  return {
    provide: {
      prisma
    }
  }
})
```

### API 預設配置

```typescript
// plugins/api-defaults.server.ts
export default defineNuxtPlugin((nuxtApp) => {
  // 設定伺服器端 API 請求的預設選項
  nuxtApp.hook('app:created', () => {
    const config = useRuntimeConfig()

    // 設定預設 headers
    if (nuxtApp.ssrContext) {
      nuxtApp.ssrContext.event.headers = {
        ...nuxtApp.ssrContext.event.headers,
        'X-Requested-With': 'Nuxt-SSR',
        'User-Agent': 'Nuxt/3.0'
      }
    }
  })
})
```

---

## 第三方套件整合

### Axios 整合

```typescript
// plugins/axios.ts
import axios from 'axios'

export default defineNuxtPlugin((nuxtApp) => {
  const config = useRuntimeConfig()

  // 創建 axios 實例
  const api = axios.create({
    baseURL: config.public.apiBase,
    timeout: 10000,
    headers: {
      'Content-Type': 'application/json'
    }
  })

  // 請求攔截器
  api.interceptors.request.use(
    (config) => {
      // 添加認證 token
      const token = useCookie('auth_token')
      if (token.value) {
        config.headers.Authorization = `Bearer ${token.value}`
      }

      return config
    },
    (error) => {
      return Promise.reject(error)
    }
  )

  // 響應攔截器
  api.interceptors.response.use(
    (response) => {
      return response.data
    },
    (error) => {
      // 處理錯誤
      if (error.response?.status === 401) {
        // 未授權，重定向到登入頁
        navigateTo('/login')
      }

      return Promise.reject(error)
    }
  )

  return {
    provide: {
      api
    }
  }
})
```

### Day.js 整合

```typescript
// plugins/dayjs.ts
import dayjs from 'dayjs'
import relativeTime from 'dayjs/plugin/relativeTime'
import utc from 'dayjs/plugin/utc'
import timezone from 'dayjs/plugin/timezone'
import 'dayjs/locale/zh-tw'

export default defineNuxtPlugin(() => {
  // 擴展功能
  dayjs.extend(relativeTime)
  dayjs.extend(utc)
  dayjs.extend(timezone)

  // 設定語言
  dayjs.locale('zh-tw')

  // 設定預設時區
  dayjs.tz.setDefault('Asia/Taipei')

  return {
    provide: {
      dayjs
    }
  }
})
```

### Chart.js 整合

```typescript
// plugins/chart.client.ts
import {
  Chart,
  Title,
  Tooltip,
  Legend,
  BarElement,
  CategoryScale,
  LinearScale,
  LineElement,
  PointElement
} from 'chart.js'

export default defineNuxtPlugin(() => {
  // 註冊 Chart.js 組件
  Chart.register(
    Title,
    Tooltip,
    Legend,
    BarElement,
    CategoryScale,
    LinearScale,
    LineElement,
    PointElement
  )

  // 預設配置
  Chart.defaults.font.family = 'Arial, sans-serif'
  Chart.defaults.color = '#333'

  return {
    provide: {
      Chart
    }
  }
})
```

### 使用第三方套件

```vue
<script setup>
const { $api, $dayjs, $Chart } = useNuxtApp()

// 使用 Axios
const fetchUsers = async () => {
  const users = await $api.get('/users')
  return users
}

// 使用 Day.js
const formatDate = (date: string) => {
  return $dayjs(date).format('YYYY-MM-DD HH:mm:ss')
}

const timeAgo = (date: string) => {
  return $dayjs(date).fromNow()
}

// 使用 Chart.js
onMounted(() => {
  const ctx = document.getElementById('myChart')
  new $Chart(ctx, {
    type: 'bar',
    data: {
      labels: ['一月', '二月', '三月'],
      datasets: [{
        label: '銷售額',
        data: [12, 19, 3]
      }]
    }
  })
})
</script>
```

---

## Plugin 執行順序

### 預設順序

Plugin 按照以下順序執行：
1. 數字前綴順序（如 `01.`, `02.`）
2. 字母順序

### 控制執行順序

```
plugins/
├── 01.config.ts      # 1. 首先執行
├── 02.auth.ts        # 2. 第二執行
├── 03.api.ts         # 3. 第三執行
└── utils.ts          # 4. 最後執行（無數字前綴）
```

### 範例：有序初始化

```typescript
// plugins/01.init.ts
export default defineNuxtPlugin(async (nuxtApp) => {
  console.log('[1] 初始化配置')

  const config = await $fetch('/api/config')

  return {
    provide: {
      config
    }
  }
})
```

```typescript
// plugins/02.auth.ts
export default defineNuxtPlugin((nuxtApp) => {
  console.log('[2] 初始化認證')

  // 可以訪問先前 plugin 提供的值
  const config = nuxtApp.$config

  const auth = {
    // 使用 config
  }

  return {
    provide: {
      auth
    }
  }
})
```

```typescript
// plugins/03.api.ts
export default defineNuxtPlugin((nuxtApp) => {
  console.log('[3] 初始化 API')

  // 可以訪問 config 和 auth
  const { $config, $auth } = nuxtApp

  // 創建 API 客戶端
})
```

### 依賴管理

```typescript
// plugins/dependent.ts
export default defineNuxtPlugin((nuxtApp) => {
  // 確保依賴的 plugin 已執行
  if (!nuxtApp.$config) {
    console.error('Config plugin 尚未執行！')
    return
  }

  // 使用依賴
  const config = nuxtApp.$config
})
```

---

## 完整範例

### 範例 1：完整的 API 客戶端

```typescript
// plugins/api-client.ts
interface ApiClient {
  get<T>(url: string, params?: any): Promise<T>
  post<T>(url: string, data?: any): Promise<T>
  put<T>(url: string, data?: any): Promise<T>
  delete<T>(url: string): Promise<T>
}

export default defineNuxtPlugin((nuxtApp) => {
  const config = useRuntimeConfig()
  const token = useCookie('auth_token')

  const request = async (
    method: string,
    url: string,
    data?: any,
    params?: any
  ) => {
    const headers: Record<string, string> = {
      'Content-Type': 'application/json'
    }

    if (token.value) {
      headers.Authorization = `Bearer ${token.value}`
    }

    try {
      const response = await $fetch(url, {
        baseURL: config.public.apiBase,
        method: method as any,
        headers,
        body: data,
        params
      })

      return response
    } catch (error: any) {
      // 錯誤處理
      if (error.response?.status === 401) {
        // 清除 token 並重定向
        token.value = null
        await navigateTo('/login')
      }

      throw error
    }
  }

  const apiClient: ApiClient = {
    get: (url, params) => request('GET', url, undefined, params),
    post: (url, data) => request('POST', url, data),
    put: (url, data) => request('PUT', url, data),
    delete: (url) => request('DELETE', url)
  }

  return {
    provide: {
      apiClient
    }
  }
})
```

### 範例 2：全域錯誤處理

```typescript
// plugins/error-handler.ts
export default defineNuxtPlugin((nuxtApp) => {
  // Vue 錯誤處理
  nuxtApp.vueApp.config.errorHandler = (error, instance, info) => {
    console.error('Vue Error:', error)
    console.error('Component:', instance)
    console.error('Info:', info)

    // 發送錯誤報告到服務器
    $fetch('/api/error-log', {
      method: 'POST',
      body: {
        type: 'vue',
        error: error.toString(),
        stack: error.stack,
        info
      }
    }).catch(() => {
      // 忽略錯誤報告失敗
    })
  }

  // Nuxt 錯誤處理
  nuxtApp.hook('vue:error', (error, instance, info) => {
    console.error('Nuxt Error:', error, info)
  })

  // 未捕獲的 Promise 錯誤
  if (process.client) {
    window.addEventListener('unhandledrejection', (event) => {
      console.error('Unhandled Promise Rejection:', event.reason)

      $fetch('/api/error-log', {
        method: 'POST',
        body: {
          type: 'unhandled-rejection',
          error: event.reason?.toString(),
          stack: event.reason?.stack
        }
      }).catch(() => {})
    })
  }
})
```

### 範例 3：全域組件註冊

```typescript
// plugins/components.ts
import { defineAsyncComponent } from 'vue'

export default defineNuxtPlugin((nuxtApp) => {
  // 註冊全域組件
  nuxtApp.vueApp.component(
    'VButton',
    defineAsyncComponent(() => import('~/components/VButton.vue'))
  )

  nuxtApp.vueApp.component(
    'VInput',
    defineAsyncComponent(() => import('~/components/VInput.vue'))
  )

  nuxtApp.vueApp.component(
    'VModal',
    defineAsyncComponent(() => import('~/components/VModal.vue'))
  )
})
```

### 範例 4：全域指令

```typescript
// plugins/directives.ts
export default defineNuxtPlugin((nuxtApp) => {
  // v-focus 指令
  nuxtApp.vueApp.directive('focus', {
    mounted(el) {
      el.focus()
    }
  })

  // v-click-outside 指令
  nuxtApp.vueApp.directive('click-outside', {
    mounted(el, binding) {
      el.clickOutsideEvent = (event: Event) => {
        if (!(el === event.target || el.contains(event.target as Node))) {
          binding.value(event)
        }
      }
      document.addEventListener('click', el.clickOutsideEvent)
    },
    unmounted(el) {
      document.removeEventListener('click', el.clickOutsideEvent)
    }
  })

  // v-tooltip 指令
  nuxtApp.vueApp.directive('tooltip', {
    mounted(el, binding) {
      el.setAttribute('title', binding.value)
      el.style.position = 'relative'
    }
  })
})
```

### 範例 5：主題切換系統

```typescript
// plugins/theme.ts
export default defineNuxtPlugin((nuxtApp) => {
  const themeCookie = useCookie('theme', {
    default: () => 'light',
    maxAge: 60 * 60 * 24 * 365 // 1 年
  })

  const theme = useState('theme', () => themeCookie.value)

  const setTheme = (newTheme: 'light' | 'dark') => {
    theme.value = newTheme
    themeCookie.value = newTheme

    if (process.client) {
      document.documentElement.setAttribute('data-theme', newTheme)
      localStorage.setItem('theme', newTheme)
    }
  }

  const toggleTheme = () => {
    setTheme(theme.value === 'light' ? 'dark' : 'light')
  }

  // 初始化主題
  if (process.client) {
    document.documentElement.setAttribute('data-theme', theme.value)
  }

  return {
    provide: {
      theme: readonly(theme),
      setTheme,
      toggleTheme
    }
  }
})
```

### 範例 6：國際化 (i18n)

```typescript
// plugins/i18n.ts
interface Messages {
  [key: string]: string | Messages
}

interface I18n {
  locale: Ref<string>
  messages: Record<string, Messages>
  t(key: string, params?: Record<string, any>): string
  setLocale(locale: string): void
}

export default defineNuxtPlugin(() => {
  const localeCookie = useCookie('locale', {
    default: () => 'zh-TW'
  })

  const locale = useState('locale', () => localeCookie.value)

  const messages: Record<string, Messages> = {
    'zh-TW': {
      welcome: '歡迎',
      hello: '你好 {name}',
      nav: {
        home: '首頁',
        about: '關於我們'
      }
    },
    'en': {
      welcome: 'Welcome',
      hello: 'Hello {name}',
      nav: {
        home: 'Home',
        about: 'About'
      }
    }
  }

  const t = (key: string, params?: Record<string, any>): string => {
    const keys = key.split('.')
    let value: any = messages[locale.value]

    for (const k of keys) {
      value = value?.[k]
    }

    if (typeof value !== 'string') {
      return key
    }

    // 替換參數
    if (params) {
      Object.keys(params).forEach(param => {
        value = value.replace(`{${param}}`, params[param])
      })
    }

    return value
  }

  const setLocale = (newLocale: string) => {
    locale.value = newLocale
    localeCookie.value = newLocale
  }

  const i18n: I18n = {
    locale: readonly(locale) as Ref<string>,
    messages,
    t,
    setLocale
  }

  return {
    provide: {
      i18n,
      t
    }
  }
})
```

### 使用完整範例

```vue
<script setup>
const {
  $apiClient,
  $theme,
  $toggleTheme,
  $t,
  $i18n
} = useNuxtApp()

// API 請求
const users = ref([])
const fetchUsers = async () => {
  users.value = await $apiClient.get('/users')
}

// 主題切換
const handleThemeToggle = () => {
  $toggleTheme()
}

// 國際化
const greeting = computed(() => {
  return $t('hello', { name: 'World' })
})

// 切換語言
const changeLanguage = (lang: string) => {
  $i18n.setLocale(lang)
}

onMounted(() => {
  fetchUsers()
})
</script>

<template>
  <div>
    <h1>{{ greeting }}</h1>
    <p>{{ $t('welcome') }}</p>

    <nav>
      <a href="/">{{ $t('nav.home') }}</a>
      <a href="/about">{{ $t('nav.about') }}</a>
    </nav>

    <button @click="handleThemeToggle">
      當前主題: {{ $theme }}
    </button>

    <button @click="changeLanguage('en')">English</button>
    <button @click="changeLanguage('zh-TW')">繁體中文</button>

    <ul>
      <li v-for="user in users" :key="user.id">
        {{ user.name }}
      </li>
    </ul>
  </div>
</template>
```

### 範例 7：性能監控

```typescript
// plugins/performance.client.ts
export default defineNuxtPlugin((nuxtApp) => {
  const metrics = ref({
    ttfb: 0,
    fcp: 0,
    lcp: 0,
    fid: 0,
    cls: 0
  })

  // Time to First Byte
  const measureTTFB = () => {
    const navigation = performance.getEntriesByType('navigation')[0] as PerformanceNavigationTiming
    metrics.value.ttfb = navigation.responseStart - navigation.requestStart
  }

  // First Contentful Paint
  const measureFCP = () => {
    const paint = performance.getEntriesByType('paint')
      .find(entry => entry.name === 'first-contentful-paint')

    if (paint) {
      metrics.value.fcp = paint.startTime
    }
  }

  // Largest Contentful Paint
  const measureLCP = () => {
    const observer = new PerformanceObserver((list) => {
      const entries = list.getEntries()
      const lastEntry = entries[entries.length - 1]
      metrics.value.lcp = lastEntry.startTime
    })

    observer.observe({ entryTypes: ['largest-contentful-paint'] })
  }

  // 頁面載入完成後測量
  nuxtApp.hook('page:finish', () => {
    setTimeout(() => {
      measureTTFB()
      measureFCP()
      measureLCP()

      console.log('Performance Metrics:', metrics.value)

      // 發送到分析服務
      $fetch('/api/analytics/performance', {
        method: 'POST',
        body: metrics.value
      }).catch(() => {})
    }, 0)
  })

  return {
    provide: {
      metrics: readonly(metrics)
    }
  }
})
```

## 最佳實踐

### 1. Plugin 命名規範
- 使用描述性名稱
- 使用數字前綴控制順序
- 使用 `.client` 或 `.server` 後綴明確執行環境

### 2. 性能考量
- 避免在 Plugin 中執行耗時操作
- 使用異步載入大型庫
- 僅在需要時註冊組件和指令

### 3. 錯誤處理
- 總是處理異步操作的錯誤
- 提供有意義的錯誤訊息
- 避免 Plugin 失敗導致應用崩潰

### 4. TypeScript 支援
- 定義清晰的類型
- 擴展 NuxtApp 介面
- 使用類型安全的 provide/inject

### 5. SSR 相容性
- 明確區分客戶端和伺服器端邏輯
- 避免在伺服器端使用瀏覽器 API
- 正確處理狀態同步

這個完整的 Plugin 系統為您的 Nuxt.js 應用提供了強大的擴展能力！
