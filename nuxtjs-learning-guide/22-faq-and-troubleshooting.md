# 第 22 章：常見問題與疑難排解

## 目錄
- [常見錯誤訊息解析](#常見錯誤訊息解析)
- [window/document is not defined](#windowdocument-is-not-defined)
- [Hydration Mismatch](#hydration-mismatch)
- [模組解析問題](#模組解析問題)
- [環境變數無法讀取](#環境變數無法讀取)
- [路由問題](#路由問題)
- [API 請求問題](#api-請求問題)
- [建置問題](#建置問題)
- [部署問題](#部署問題)
- [除錯技巧](#除錯技巧)
- [社群資源](#社群資源)
- [官方文件連結](#官方文件連結)
- [實用工具推薦](#實用工具推薦)

---

## 常見錯誤訊息解析

### 1. "Cannot find module"

**錯誤範例：**
```
Error: Cannot find module '@/components/MyComponent.vue'
```

**原因：**
- 路徑別名配置錯誤
- 模組未安裝
- 檔案路徑錯誤

**解決方法：**

```typescript
// nuxt.config.ts - 檢查路徑別名配置
export default defineNuxtConfig({
  alias: {
    '@': fileURLToPath(new URL('./', import.meta.url)),
    '~': fileURLToPath(new URL('./', import.meta.url))
  }
})
```

```bash
# 確保模組已安裝
npm install

# 清除快取重新建置
rm -rf .nuxt node_modules/.vite
npm install
npm run dev
```

### 2. "500 Internal Server Error"

**錯誤範例：**
```
[nuxt] [request error] [unhandled] [500] Internal Server Error
```

**常見原因：**

```typescript
// ❌ 錯誤：在 setup 中使用 async 但沒有正確處理
<script setup>
const data = await $fetch('/api/data') // 可能導致 500 錯誤
</script>

// ✅ 正確：使用 useAsyncData 或 useFetch
<script setup>
const { data } = await useAsyncData('key', () => $fetch('/api/data'))
// 或
const { data } = await useFetch('/api/data')
</script>
```

### 3. "Failed to fetch dynamically imported module"

**錯誤範例：**
```
Failed to fetch dynamically imported module:
http://localhost:3000/_nuxt/pages/about.vue.js
```

**原因：**
- 建置後檔案變更但瀏覽器快取舊版本
- 部署時檔案路徑不正確

**解決方法：**

```bash
# 開發環境：清除快取
rm -rf .nuxt .output
npm run dev

# 生產環境：檢查 CDN 設定
# nuxt.config.ts
export default defineNuxtConfig({
  app: {
    buildAssetsDir: '/_nuxt/',
    cdnURL: process.env.CDN_URL
  }
})
```

### 4. "Hydration completed but contains mismatches"

**錯誤範例：**
```
[Vue warn]: Hydration completed but contains mismatches.
```

詳見 [Hydration Mismatch](#hydration-mismatch) 章節。

---

## window/document is not defined

### 問題說明

在 SSR (Server-Side Rendering) 環境中，`window`、`document`、`localStorage` 等瀏覽器 API 在伺服器端不存在。

### 錯誤範例

```typescript
// ❌ 錯誤：直接使用瀏覽器 API
<script setup>
const width = window.innerWidth // ReferenceError: window is not defined
</script>
```

### 解決方案

#### 方案 1：使用 process.client 檢查

```typescript
<script setup>
const width = ref(0)

if (process.client) {
  width.value = window.innerWidth
}

// 或在 onMounted 中使用
onMounted(() => {
  width.value = window.innerWidth
})
</script>
```

#### 方案 2：使用 ClientOnly 組件

```vue
<template>
  <div>
    <!-- 僅在客戶端渲染 -->
    <ClientOnly>
      <BrowserOnlyComponent />
      <template #fallback>
        <div>載入中...</div>
      </template>
    </ClientOnly>
  </div>
</template>
```

#### 方案 3：使用 .client 後綴

```
components/
├── Chart.client.vue     # 僅在客戶端載入
└── ServerSafe.vue       # 在伺服器和客戶端都載入
```

```vue
<template>
  <div>
    <!-- Nuxt 會自動僅在客戶端渲染此組件 -->
    <Chart />
  </div>
</template>
```

#### 方案 4：建立跨平台的 Composable

```typescript
// composables/useWindowSize.ts
export const useWindowSize = () => {
  const width = ref(0)
  const height = ref(0)

  const update = () => {
    if (process.client) {
      width.value = window.innerWidth
      height.value = window.innerHeight
    }
  }

  onMounted(() => {
    update()
    window.addEventListener('resize', update)
  })

  onUnmounted(() => {
    if (process.client) {
      window.removeEventListener('resize', update)
    }
  })

  return {
    width: readonly(width),
    height: readonly(height)
  }
}
```

#### 方案 5：處理 localStorage

```typescript
// composables/useLocalStorage.ts
export const useLocalStorage = <T>(key: string, defaultValue: T) => {
  const data = ref<T>(defaultValue)

  // SSR 時返回默認值
  if (process.server) {
    return {
      data: readonly(data),
      set: (value: T) => {},
      remove: () => {}
    }
  }

  // 客戶端實現
  const set = (value: T) => {
    try {
      data.value = value
      localStorage.setItem(key, JSON.stringify(value))
    } catch (error) {
      console.error('localStorage set error:', error)
    }
  }

  const remove = () => {
    try {
      localStorage.removeItem(key)
      data.value = defaultValue
    } catch (error) {
      console.error('localStorage remove error:', error)
    }
  }

  // 初始化
  onMounted(() => {
    try {
      const stored = localStorage.getItem(key)
      if (stored) {
        data.value = JSON.parse(stored)
      }
    } catch (error) {
      console.error('localStorage get error:', error)
    }
  })

  return {
    data,
    set,
    remove
  }
}
```

**使用範例：**

```vue
<script setup>
const { data: theme, set: setTheme } = useLocalStorage('theme', 'light')

const toggleTheme = () => {
  setTheme(theme.value === 'light' ? 'dark' : 'light')
}
</script>
```

---

## Hydration Mismatch

### 問題說明

Hydration mismatch 發生在伺服器渲染的 HTML 與客戶端 Vue 應用期望的 DOM 結構不一致時。

### 常見原因與解決方案

#### 1. 使用隨機數或時間戳

```vue
<!-- ❌ 錯誤 -->
<template>
  <div>{{ Math.random() }}</div>
  <!-- 伺服器端和客戶端生成的隨機數不同 -->
</template>

<!-- ✅ 正確 -->
<template>
  <ClientOnly>
    <div>{{ Math.random() }}</div>
  </ClientOnly>
</template>
```

#### 2. 日期格式化問題

```typescript
// ❌ 錯誤：時區不一致
<template>
  <div>{{ new Date().toLocaleString() }}</div>
</template>

// ✅ 正確：使用 ISO 格式或僅在客戶端顯示
<script setup>
const date = ref('')

onMounted(() => {
  date.value = new Date().toLocaleString('zh-TW')
})
</script>

<template>
  <div>{{ date || '載入中...' }}</div>
</template>
```

#### 3. 條件渲染基於瀏覽器 API

```vue
<!-- ❌ 錯誤 -->
<script setup>
const isMobile = window.innerWidth < 768 // SSR 時 window 不存在
</script>

<template>
  <div v-if="isMobile">Mobile View</div>
</template>

<!-- ✅ 正確 -->
<script setup>
const isMobile = ref(false)

onMounted(() => {
  isMobile.value = window.innerWidth < 768
})
</script>

<template>
  <ClientOnly>
    <div v-if="isMobile">Mobile View</div>
    <div v-else>Desktop View</div>
  </ClientOnly>
</template>
```

#### 4. 第三方庫 DOM 操作

```vue
<!-- ❌ 錯誤：第三方庫直接操作 DOM -->
<script setup>
import SomeLibrary from 'some-library'

const instance = new SomeLibrary() // 可能在 SSR 時失敗
</script>

<!-- ✅ 正確：僅在客戶端初始化 -->
<script setup>
import SomeLibrary from 'some-library'

let instance = null

onMounted(() => {
  instance = new SomeLibrary()
})

onUnmounted(() => {
  instance?.destroy()
})
</script>
```

#### 5. HTML 結構不一致

```vue
<!-- ❌ 錯誤：條件渲染導致結構不同 -->
<template>
  <ul>
    <li v-for="item in items" :key="item.id">
      {{ item.name }}
      <span v-if="process.client">(客戶端)</span>
    </li>
  </ul>
</template>

<!-- ✅ 正確：保持結構一致 -->
<template>
  <ul>
    <li v-for="item in items" :key="item.id">
      {{ item.name }}
      <ClientOnly>
        <span>(客戶端)</span>
      </ClientOnly>
    </li>
  </ul>
</template>
```

### 除錯 Hydration 問題

```typescript
// nuxt.config.ts - 啟用詳細的 hydration 警告
export default defineNuxtConfig({
  vue: {
    compilerOptions: {
      // 在開發環境顯示詳細警告
      isCustomElement: (tag) => false
    }
  },

  // 開發環境配置
  devtools: {
    enabled: true
  }
})
```

---

## 模組解析問題

### 1. TypeScript 路徑別名不生效

**問題：**
```typescript
import { MyComponent } from '@/components/MyComponent'
// Error: Cannot find module '@/components/MyComponent'
```

**解決方法：**

```json
// tsconfig.json
{
  "extends": "./.nuxt/tsconfig.json",
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["./*"],
      "~/*": ["./*"]
    }
  }
}
```

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  alias: {
    '@': fileURLToPath(new URL('./', import.meta.url)),
    '~': fileURLToPath(new URL('./', import.meta.url)),
    '@components': fileURLToPath(new URL('./components', import.meta.url))
  }
})
```

### 2. 自動導入失效

**問題：**
```vue
<script setup>
// useFetch 未定義
const { data } = await useFetch('/api/data')
</script>
```

**解決方法：**

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  imports: {
    autoImport: true,
    dirs: [
      'composables/**',
      'utils/**'
    ]
  }
})
```

```bash
# 重新生成類型定義
npx nuxi prepare
npx nuxi typecheck
```

### 3. 組件未自動註冊

**問題：**
```vue
<template>
  <!-- MyComponent 未定義 -->
  <MyComponent />
</template>
```

**解決方法：**

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  components: [
    {
      path: '~/components',
      pathPrefix: false
    },
    {
      path: '~/components/ui',
      prefix: 'Ui'
    }
  ]
})
```

```
components/
├── MyComponent.vue        → <MyComponent />
├── ui/
│   └── Button.vue         → <UiButton />
└── icons/
    └── Logo.vue           → <IconsLogo />
```

---

## 環境變數無法讀取

### 1. 環境變數命名規則

**問題：**
```bash
# .env
API_URL=https://api.example.com  # ❌ 無法在客戶端讀取
```

**解決方法：**

```bash
# .env
# 伺服器端變數
API_SECRET_KEY=secret123

# 客戶端變數（必須以 NUXT_PUBLIC_ 開頭）
NUXT_PUBLIC_API_URL=https://api.example.com
NUXT_PUBLIC_SITE_NAME=My Site
```

### 2. 正確使用 runtimeConfig

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  runtimeConfig: {
    // 僅伺服器端可訪問
    apiSecret: process.env.API_SECRET_KEY,
    databaseUrl: process.env.DATABASE_URL,

    // 客戶端和伺服器端都可訪問
    public: {
      apiUrl: process.env.NUXT_PUBLIC_API_URL,
      siteName: process.env.NUXT_PUBLIC_SITE_NAME
    }
  }
})
```

**使用方式：**

```typescript
// 在 API 路由中（伺服器端）
export default defineEventHandler((event) => {
  const config = useRuntimeConfig()
  const secret = config.apiSecret // ✅ 可訪問
  const apiUrl = config.public.apiUrl // ✅ 可訪問
})

// 在組件中（客戶端）
<script setup>
const config = useRuntimeConfig()
// const secret = config.apiSecret // ❌ undefined
const apiUrl = config.public.apiUrl // ✅ 可訪問
</script>
```

### 3. 環境變數未載入

```bash
# 確保 .env 文件在專案根目錄
project/
├── .env              # ✅ 正確位置
├── nuxt.config.ts
└── package.json

# 重啟開發伺服器
npm run dev
```

---

## 路由問題

### 1. 動態路由參數獲取失敗

**問題：**
```typescript
// pages/blog/[slug].vue
const route = useRoute()
console.log(route.params.slug) // undefined
```

**解決方法：**

```vue
<!-- pages/blog/[slug].vue -->
<script setup>
const route = useRoute()

// ✅ 方法 1：直接訪問
console.log(route.params.slug)

// ✅ 方法 2：使用 computed 確保響應式
const slug = computed(() => route.params.slug)

// ✅ 方法 3：使用 watch 監聽變化
watch(() => route.params.slug, (newSlug) => {
  console.log('Slug changed:', newSlug)
})
</script>
```

### 2. 路由跳轉不生效

**問題：**
```typescript
// ❌ 錯誤：使用 window.location
window.location.href = '/about' // 會重新載入整個頁面
```

**解決方法：**

```typescript
// ✅ 正確：使用 navigateTo
await navigateTo('/about')

// 帶查詢參數
await navigateTo({
  path: '/search',
  query: { q: 'nuxt' }
})

// 外部連結
await navigateTo('https://nuxt.com', {
  external: true,
  open: {
    target: '_blank'
  }
})

// 使用 router
const router = useRouter()
await router.push('/about')
```

### 3. 中介軟體 (Middleware) 問題

**問題：**
```typescript
// middleware/auth.ts
export default defineNuxtRouteMiddleware((to, from) => {
  // ❌ 錯誤：沒有返回值
  if (!isAuthenticated()) {
    navigateTo('/login')
  }
})
```

**解決方法：**

```typescript
// middleware/auth.ts
export default defineNuxtRouteMiddleware((to, from) => {
  const { isAuthenticated } = useAuth()

  // ✅ 正確：返回 navigateTo 的結果
  if (!isAuthenticated.value) {
    return navigateTo('/login')
  }
})
```

```typescript
// middleware/auth.global.ts - 全域中介軟體
export default defineNuxtRouteMiddleware((to) => {
  // 排除公開路由
  const publicPages = ['/login', '/register', '/']
  if (publicPages.includes(to.path)) {
    return
  }

  const { isAuthenticated } = useAuth()
  if (!isAuthenticated.value) {
    return navigateTo('/login')
  }
})
```

### 4. 404 頁面配置

```vue
<!-- pages/404.vue 或 error.vue -->
<template>
  <div class="error-page">
    <h1>{{ error.statusCode }}</h1>
    <p>{{ error.message }}</p>
    <button @click="handleError">返回首頁</button>
  </div>
</template>

<script setup lang="ts">
interface Props {
  error: {
    statusCode: number
    message: string
  }
}

defineProps<Props>()

const handleError = () => {
  clearError({ redirect: '/' })
}
</script>
```

---

## API 請求問題

### 1. CORS 錯誤

**錯誤訊息：**
```
Access to fetch at 'https://api.example.com' from origin 'http://localhost:3000'
has been blocked by CORS policy
```

**解決方法：**

```typescript
// server/api/proxy.ts - 建立代理 API
export default defineEventHandler(async (event) => {
  const query = getQuery(event)
  const body = await readBody(event).catch(() => ({}))

  // 轉發請求到實際 API
  return await $fetch('https://api.example.com/data', {
    method: event.method,
    query,
    body
  })
})
```

```vue
<!-- 在組件中使用 -->
<script setup>
// ✅ 使用自己的 API 路由，避免 CORS
const { data } = await useFetch('/api/proxy', {
  query: { id: 1 }
})
</script>
```

### 2. SSR 請求超時

**問題：**
```typescript
const { data } = await useFetch('/api/slow-endpoint')
// 請求超時導致頁面無法載入
```

**解決方法：**

```typescript
// 設定超時時間
const { data, error } = await useFetch('/api/slow-endpoint', {
  timeout: 5000, // 5 秒超時

  // 錯誤處理
  onResponseError({ response }) {
    console.error('API Error:', response.status)
  }
})

// 或使用 lazy: true 避免阻塞渲染
const { data, pending } = await useFetch('/api/slow-endpoint', {
  lazy: true
})
```

### 3. 請求重複發送

**問題：**
```typescript
// ❌ 每次組件重新渲染都會發送請求
const fetchData = async () => {
  const data = await $fetch('/api/data')
  return data
}
```

**解決方法：**

```typescript
// ✅ 使用 useAsyncData 自動去重
const { data, refresh } = await useAsyncData('unique-key', () =>
  $fetch('/api/data')
)

// ✅ 或使用 useFetch (內部使用 useAsyncData)
const { data, refresh } = await useFetch('/api/data', {
  key: 'unique-key'
})

// 手動刷新
const handleRefresh = async () => {
  await refresh()
}
```

### 4. 錯誤處理

```typescript
// composables/useApi.ts
export const useApi = () => {
  const handleError = (error: any) => {
    if (error.response) {
      // HTTP 錯誤
      switch (error.response.status) {
        case 401:
          navigateTo('/login')
          break
        case 403:
          console.error('權限不足')
          break
        case 404:
          console.error('資源不存在')
          break
        case 500:
          console.error('伺服器錯誤')
          break
        default:
          console.error('未知錯誤:', error.message)
      }
    } else {
      // 網路錯誤
      console.error('網路錯誤:', error.message)
    }
  }

  const get = async <T>(url: string, options = {}) => {
    try {
      return await $fetch<T>(url, {
        method: 'GET',
        ...options
      })
    } catch (error) {
      handleError(error)
      throw error
    }
  }

  const post = async <T>(url: string, body: any, options = {}) => {
    try {
      return await $fetch<T>(url, {
        method: 'POST',
        body,
        ...options
      })
    } catch (error) {
      handleError(error)
      throw error
    }
  }

  return {
    get,
    post
  }
}
```

---

## 建置問題

### 1. 記憶體不足

**錯誤訊息：**
```
FATAL ERROR: Reached heap limit Allocation failed - JavaScript heap out of memory
```

**解決方法：**

```json
// package.json
{
  "scripts": {
    "build": "NODE_OPTIONS='--max-old-space-size=4096' nuxt build",
    "generate": "NODE_OPTIONS='--max-old-space-size=4096' nuxt generate"
  }
}
```

### 2. 建置超時

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  vite: {
    build: {
      // 調整 chunk 大小警告閾值
      chunkSizeWarningLimit: 1000,

      // 優化建置速度
      rollupOptions: {
        output: {
          manualChunks(id) {
            if (id.includes('node_modules')) {
              return 'vendor'
            }
          }
        }
      }
    }
  }
})
```

### 3. 清除快取重建

```bash
# 完整清理
rm -rf .nuxt .output node_modules/.vite node_modules/.cache

# 重新安裝依賴
npm install

# 重新建置
npm run build
```

---

## 部署問題

### 1. 環境變數未生效

**檢查清單：**

```bash
# ✅ 在部署平台設定環境變數
NUXT_PUBLIC_API_URL=https://api.production.com
DATABASE_URL=postgresql://...

# ✅ 確認 nuxt.config.ts 配置正確
runtimeConfig: {
  databaseUrl: process.env.DATABASE_URL,
  public: {
    apiUrl: process.env.NUXT_PUBLIC_API_URL
  }
}

# ✅ 驗證環境變數
echo $NUXT_PUBLIC_API_URL
```

### 2. 靜態資源 404

**問題：**
```
GET /_nuxt/app.js 404 Not Found
```

**解決方法：**

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  app: {
    baseURL: process.env.BASE_URL || '/',
    buildAssetsDir: '/_nuxt/',
    cdnURL: process.env.CDN_URL
  }
})
```

### 3. API 路由失效

**Vercel/Netlify 配置：**

```json
// vercel.json
{
  "builds": [
    {
      "src": "nuxt.config.ts",
      "use": "@nuxtjs/vercel-builder"
    }
  ]
}
```

```toml
# netlify.toml
[build]
  command = "npm run build"
  publish = ".output/public"

[[redirects]]
  from = "/*"
  to = "/.netlify/functions/server/:splat"
  status = 200
```

### 4. SSR vs SSG 選擇

```typescript
// nuxt.config.ts

// SSR（伺服器端渲染）- 適合動態內容
export default defineNuxtConfig({
  ssr: true
})

// SSG（靜態生成）- 適合靜態網站
export default defineNuxtConfig({
  ssr: true,
  nitro: {
    preset: 'static'
  }
})

// SPA（單頁應用）- 適合客戶端渲染
export default defineNuxtConfig({
  ssr: false
})
```

---

## 除錯技巧

### 1. 使用 Vue DevTools

```bash
# 安裝 Vue DevTools 瀏覽器擴充功能
# Chrome: https://chrome.google.com/webstore/detail/vuejs-devtools/
# Firefox: https://addons.mozilla.org/firefox/addon/vue-js-devtools/
```

```typescript
// nuxt.config.ts - 啟用 DevTools
export default defineNuxtConfig({
  devtools: {
    enabled: true
  }
})
```

### 2. 啟用詳細日誌

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  // 開發環境詳細日誌
  debug: process.env.NODE_ENV === 'development',

  // Nitro 日誌
  nitro: {
    logLevel: 'verbose'
  }
})
```

```bash
# 執行時啟用除錯
DEBUG=nuxt:* npm run dev
```

### 3. 伺服器端除錯

```typescript
// server/api/debug.ts
export default defineEventHandler((event) => {
  console.log('Request URL:', event.node.req.url)
  console.log('Request Method:', event.node.req.method)
  console.log('Headers:', event.node.req.headers)

  return {
    message: 'Debug info logged'
  }
})
```

### 4. 效能分析

```vue
<script setup>
// 使用 Performance API
const measureRenderTime = () => {
  performance.mark('render-start')

  // ... 渲染邏輯

  performance.mark('render-end')
  performance.measure('render', 'render-start', 'render-end')

  const measure = performance.getEntriesByName('render')[0]
  console.log(`渲染時間: ${measure.duration}ms`)
}

onMounted(() => {
  measureRenderTime()
})
</script>
```

### 5. 網路請求除錯

```typescript
// plugins/fetch-debug.client.ts
export default defineNuxtPlugin(() => {
  const originalFetch = window.fetch

  window.fetch = function(...args) {
    console.log('Fetch 請求:', args[0])

    return originalFetch.apply(this, args)
      .then(response => {
        console.log('Fetch 回應:', response.status, args[0])
        return response
      })
      .catch(error => {
        console.error('Fetch 錯誤:', error, args[0])
        throw error
      })
  }
})
```

### 6. 使用 Nuxi 檢查工具

```bash
# 檢查專案配置
npx nuxi info

# 分析專案結構
npx nuxi analyze

# 檢查類型
npx nuxi typecheck

# 清理快取
npx nuxi cleanup
```

---

## 社群資源

### 官方社群

- **Discord**：[https://discord.com/invite/nuxt](https://discord.com/invite/nuxt)
  - 活躍的社群
  - 即時技術支援
  - 公告和更新

- **GitHub Discussions**：[https://github.com/nuxt/nuxt/discussions](https://github.com/nuxt/nuxt/discussions)
  - 技術討論
  - 功能建議
  - 問題回報

- **Twitter/X**：[@nuxt_js](https://twitter.com/nuxt_js)
  - 最新動態
  - 社群分享

### 中文社群

- **Nuxt 中文社群**
  - 微信群組
  - QQ 群組
  - 知乎專欄

- **掘金 Nuxt 專區**：[https://juejin.cn/tag/Nuxt.js](https://juejin.cn/tag/Nuxt.js)
  - 中文教學文章
  - 經驗分享

### 學習資源

- **Nuxt YouTube 頻道**：[https://www.youtube.com/@NuxtLabs](https://www.youtube.com/@NuxtLabs)
  - 官方教學影片
  - 實戰案例

- **Vue School Nuxt 課程**：[https://vueschool.io/courses/nuxt-js-3-fundamentals](https://vueschool.io/courses/nuxt-js-3-fundamentals)
  - 系統化課程
  - 免費和付費內容

- **Nuxt Modules**：[https://nuxt.com/modules](https://nuxt.com/modules)
  - 官方和社群模組
  - 擴展功能

---

## 官方文件連結

### 核心文件

- **Nuxt 3 官方文件**：[https://nuxt.com/docs](https://nuxt.com/docs)
- **API 參考**：[https://nuxt.com/docs/api](https://nuxt.com/docs/api)
- **範例專案**：[https://nuxt.com/docs/examples](https://nuxt.com/docs/examples)
- **遷移指南**：[https://nuxt.com/docs/migration/overview](https://nuxt.com/docs/migration/overview)

### Vue 相關

- **Vue 3 文件**：[https://vuejs.org/](https://vuejs.org/)
- **Composition API**：[https://vuejs.org/guide/extras/composition-api-faq.html](https://vuejs.org/guide/extras/composition-api-faq.html)
- **Vue Router**：[https://router.vuejs.org/](https://router.vuejs.org/)
- **Pinia**：[https://pinia.vuejs.org/](https://pinia.vuejs.org/)

### 工具鏈

- **Vite**：[https://vitejs.dev/](https://vitejs.dev/)
- **TypeScript**：[https://www.typescriptlang.org/](https://www.typescriptlang.org/)
- **Nitro**：[https://nitro.unjs.io/](https://nitro.unjs.io/)
- **UnJS**：[https://unjs.io/](https://unjs.io/)

### 部署平台

- **Vercel**：[https://vercel.com/docs/frameworks/nuxt](https://vercel.com/docs/frameworks/nuxt)
- **Netlify**：[https://docs.netlify.com/frameworks/nuxt/](https://docs.netlify.com/frameworks/nuxt/)
- **Cloudflare Pages**：[https://developers.cloudflare.com/pages/framework-guides/deploy-a-nuxt-site/](https://developers.cloudflare.com/pages/framework-guides/deploy-a-nuxt-site/)
- **AWS Amplify**：[https://docs.amplify.aws/guides/hosting/nuxt/](https://docs.amplify.aws/guides/hosting/nuxt/)

---

## 實用工具推薦

### 開發工具

#### 1. VS Code 擴充功能

```json
{
  "recommendations": [
    "vue.volar",                    // Vue 語言支援
    "vue.vscode-typescript-vue-plugin", // TypeScript Vue 插件
    "antfu.iconify",                // 圖示支援
    "dbaeumer.vscode-eslint",       // ESLint
    "esbenp.prettier-vscode",       // Prettier
    "bradlc.vscode-tailwindcss",    // Tailwind CSS IntelliSense
    "nuxt.mdc"                      // MDC 語法支援
  ]
}
```

#### 2. Chrome 擴充功能

- **Vue DevTools**：Vue 元件除錯
- **Nuxt DevTools**：Nuxt 專用除錯工具
- **Lighthouse**：效能分析
- **JSON Viewer**：API 回應格式化

### CLI 工具

```bash
# Nuxt CLI
npm install -g nuxi

# 常用命令
nuxi init my-app           # 建立新專案
nuxi add component Foo     # 建立組件
nuxi add page about        # 建立頁面
nuxi add api hello         # 建立 API 路由
nuxi add plugin analytics  # 建立插件
nuxi typecheck             # 類型檢查
nuxi analyze               # 分析 bundle
nuxi cleanup               # 清理快取
```

### 測試工具

```bash
# Vitest - 單元測試
npm install -D @nuxt/test-utils vitest

# Playwright - E2E 測試
npm install -D @playwright/test

# Testing Library
npm install -D @testing-library/vue @testing-library/user-event
```

```typescript
// vitest.config.ts
import { defineConfig } from 'vitest/config'

export default defineConfig({
  test: {
    environment: 'nuxt',
    globals: true
  }
})
```

### 程式碼品質工具

```bash
# ESLint
npm install -D @nuxtjs/eslint-config-typescript eslint

# Prettier
npm install -D prettier

# Husky + lint-staged
npm install -D husky lint-staged
npx husky init
```

```json
// package.json
{
  "lint-staged": {
    "*.{js,ts,vue}": [
      "eslint --fix",
      "prettier --write"
    ]
  }
}
```

### 效能監控

```bash
# Lighthouse CI
npm install -g @lhci/cli

# Bundle Analyzer
npm install -D rollup-plugin-visualizer

# Nuxt Speed Kit
npm install nuxt-speed-kit
```

### 實用模組

```bash
# UI 框架
npm install @nuxt/ui                 # Nuxt UI
npm install @nuxtjs/tailwindcss     # Tailwind CSS

# 圖片優化
npm install @nuxt/image             # Nuxt Image

# 內容管理
npm install @nuxt/content           # Nuxt Content

# 國際化
npm install @nuxtjs/i18n            # i18n

# SEO
npm install nuxt-seo-kit            # SEO Kit

# 分析
npm install @nuxtjs/google-analytics # Google Analytics
npm install nuxt-gtag               # Google Tag Manager

# PWA
npm install @vite-pwa/nuxt          # PWA

# 安全
npm install nuxt-security           # Security Headers
```

---

## 常見開發模式

### 1. 錯誤邊界

```vue
<!-- components/ErrorBoundary.vue -->
<template>
  <div>
    <slot v-if="!error" />
    <div v-else class="error-boundary">
      <h2>發生錯誤</h2>
      <p>{{ error.message }}</p>
      <button @click="reset">重試</button>
    </div>
  </div>
</template>

<script setup lang="ts">
const error = ref<Error | null>(null)

const reset = () => {
  error.value = null
}

onErrorCaptured((err) => {
  error.value = err
  return false // 阻止錯誤繼續傳播
})
</script>
```

### 2. 載入狀態管理

```typescript
// composables/useLoading.ts
export const useLoading = () => {
  const loadingMap = reactive<Record<string, boolean>>({})

  const setLoading = (key: string, value: boolean) => {
    loadingMap[key] = value
  }

  const isLoading = (key: string) => {
    return loadingMap[key] || false
  }

  const withLoading = async <T>(
    key: string,
    fn: () => Promise<T>
  ): Promise<T> => {
    setLoading(key, true)
    try {
      return await fn()
    } finally {
      setLoading(key, false)
    }
  }

  return {
    loadingMap: readonly(loadingMap),
    setLoading,
    isLoading,
    withLoading
  }
}
```

### 3. 表單驗證

```typescript
// composables/useFormValidation.ts
export const useFormValidation = <T extends Record<string, any>>(
  initialValues: T,
  rules: Record<keyof T, (value: any) => string | null>
) => {
  const values = reactive({ ...initialValues })
  const errors = reactive<Record<keyof T, string | null>>({} as any)
  const touched = reactive<Record<keyof T, boolean>>({} as any)

  const validate = (field?: keyof T) => {
    const fieldsToValidate = field ? [field] : Object.keys(rules) as Array<keyof T>

    fieldsToValidate.forEach((f) => {
      const rule = rules[f]
      if (rule) {
        errors[f] = rule(values[f])
      }
    })

    return !Object.values(errors).some(error => error !== null)
  }

  const handleBlur = (field: keyof T) => {
    touched[field] = true
    validate(field)
  }

  const handleSubmit = async (onSubmit: (values: T) => Promise<void>) => {
    // 標記所有欄位為已觸碰
    Object.keys(rules).forEach(field => {
      touched[field as keyof T] = true
    })

    if (validate()) {
      await onSubmit(values)
    }
  }

  return {
    values,
    errors: readonly(errors),
    touched: readonly(touched),
    validate,
    handleBlur,
    handleSubmit
  }
}
```

---

## 疑難排解流程圖

```
遇到問題
    │
    ├─→ 檢查錯誤訊息
    │   ├─→ 搜尋官方文件
    │   ├─→ 搜尋 GitHub Issues
    │   └─→ 搜尋 Stack Overflow
    │
    ├─→ 清除快取重試
    │   ├─→ rm -rf .nuxt .output
    │   ├─→ npm install
    │   └─→ npm run dev
    │
    ├─→ 檢查環境配置
    │   ├─→ Node.js 版本
    │   ├─→ 套件版本
    │   └─→ 環境變數
    │
    ├─→ 簡化問題
    │   ├─→ 建立最小可復現範例
    │   ├─→ 逐步排除變數
    │   └─→ 對比正常案例
    │
    └─→ 尋求幫助
        ├─→ Discord 社群
        ├─→ GitHub Discussions
        └─→ Stack Overflow
```

---

## 效能問題排查

### 檢查清單

```bash
# 1. 分析 bundle 大小
npm run build
npx vite-bundle-visualizer

# 2. 檢查依賴版本
npm outdated

# 3. 檢查未使用的依賴
npx depcheck

# 4. 分析啟動時間
DEBUG=nuxt:* npm run dev

# 5. 使用 Lighthouse 分析
lighthouse http://localhost:3000 --view
```

---

## 總結

### 記住這些關鍵點

1. **SSR 環境**：永遠注意 `window`/`document` 的使用
2. **環境變數**：客戶端變數必須以 `NUXT_PUBLIC_` 開頭
3. **Hydration**：保持伺服器端和客戶端渲染一致
4. **錯誤處理**：總是處理 API 錯誤和邊界情況
5. **效能優化**：使用延遲載入和程式碼分割

### 開發最佳實踐

- ✅ 使用 TypeScript
- ✅ 啟用 ESLint 和 Prettier
- ✅ 編寫單元測試
- ✅ 使用 Git Hook 檢查程式碼
- ✅ 定期更新依賴
- ✅ 監控效能指標
- ✅ 閱讀官方文件
- ✅ 參與社群討論

### 遇到問題時

1. 不要慌張，仔細閱讀錯誤訊息
2. 搜尋官方文件和 GitHub Issues
3. 建立最小可復現範例
4. 在社群尋求幫助
5. 記錄解決方案供未來參考

---

**祝您開發順利！** 🎉

如果本指南對您有幫助，歡迎分享給其他 Nuxt.js 開發者。有任何問題或建議，歡迎在社群中討論！
