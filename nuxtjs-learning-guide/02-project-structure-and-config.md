# 專案結構與配置檔案詳解

## Nuxt 3 專案結構說明

### 完整的專案目錄結構

```
my-nuxt-app/
├── .nuxt/                  # 自動生成的開發檔案（不要編輯）
├── .output/                # 建構後的輸出檔案
├── assets/                 # 需要編譯的資源（CSS, Sass, 圖片等）
├── components/             # Vue 元件（自動導入）
│   ├── common/
│   │   └── Button.vue
│   └── layout/
│       └── Header.vue
├── composables/            # Composable 函數（自動導入）
│   └── useCounter.ts
├── content/                # Nuxt Content 模組的內容檔案
├── layouts/                # 應用程式佈局
│   ├── default.vue
│   └── custom.vue
├── middleware/             # 路由中介軟體
│   └── auth.ts
├── node_modules/           # 依賴套件
├── pages/                  # 應用程式頁面（檔案式路由）
│   ├── index.vue
│   ├── about.vue
│   └── posts/
│       └── [id].vue
├── plugins/                # 插件（自動註冊）
│   └── api.ts
├── public/                 # 靜態檔案（直接提供服務）
│   ├── favicon.ico
│   └── robots.txt
├── server/                 # 伺服器端程式碼
│   ├── api/                # API 端點
│   │   └── users.ts
│   ├── middleware/         # 伺服器中介軟體
│   └── routes/             # 伺服器路由
├── utils/                  # 工具函數（自動導入）
│   └── format.ts
├── .env                    # 環境變數
├── .gitignore             # Git 忽略檔案
├── app.vue                 # 主要應用程式元件
├── nuxt.config.ts         # Nuxt 配置檔案
├── package.json           # 專案依賴和腳本
├── tsconfig.json          # TypeScript 配置
└── README.md              # 專案說明文件
```

### 各目錄詳細說明

#### 1. `.nuxt/` 目錄

- **用途**：開發時自動生成的檔案
- **內容**：TypeScript 類型定義、路由配置、模組註冊等
- **注意**：不要手動編輯，應加入 `.gitignore`

```bash
# .gitignore
.nuxt
```

#### 2. `assets/` 目錄

存放需要經過建構工具處理的資源：

```
assets/
├── styles/
│   ├── main.css
│   └── variables.scss
├── images/
│   └── logo.svg
└── fonts/
    └── custom-font.woff2
```

使用方式：

```vue
<template>
  <!-- 圖片會被 Vite 處理 -->
  <img src="~/assets/images/logo.svg" alt="Logo" />
</template>

<style scoped>
/* 引入樣式 */
@import '~/assets/styles/variables.scss';

.logo {
  /* 背景圖片會被處理 */
  background-image: url('~/assets/images/bg.png');
}
</style>
```

#### 3. `public/` 目錄

存放靜態檔案，不會被建構工具處理：

```
public/
├── favicon.ico
├── robots.txt
└── images/
    └── static-logo.png
```

使用方式：

```vue
<template>
  <!-- 直接引用，路徑從根目錄開始 -->
  <img src="/images/static-logo.png" alt="Logo" />
</template>
```

#### 4. `components/` 目錄

所有元件會自動導入，支援巢狀目錄：

```
components/
├── TheHeader.vue          → <TheHeader />
├── TheFooter.vue          → <TheFooter />
├── common/
│   ├── Button.vue         → <CommonButton />
│   └── Input.vue          → <CommonInput />
└── product/
    ├── Card.vue           → <ProductCard />
    └── List.vue           → <ProductList />
```

#### 5. `composables/` 目錄

可重用的 Composition API 函數：

```
composables/
├── useAuth.ts             → useAuth()
├── useFetch.ts            → useFetch()
└── states.ts              → 可匯出多個函數
```

#### 6. `pages/` 目錄

定義應用程式路由：

```
pages/
├── index.vue              → /
├── about.vue              → /about
├── posts/
│   ├── index.vue          → /posts
│   └── [id].vue           → /posts/:id
└── [...slug].vue          → 所有未匹配的路由
```

## nuxt.config.ts 配置詳解

### 基礎配置

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  // 開發工具
  devtools: { enabled: true },

  // TypeScript 配置
  typescript: {
    strict: true,
    typeCheck: true,
  },

  // 應用程式配置
  app: {
    // HTML head 設定
    head: {
      title: '我的 Nuxt 應用',
      charset: 'utf-8',
      viewport: 'width=device-width, initial-scale=1',
      meta: [
        { name: 'description', content: '這是我的 Nuxt 3 應用程式' },
        { name: 'format-detection', content: 'telephone=no' }
      ],
      link: [
        { rel: 'icon', type: 'image/x-icon', href: '/favicon.ico' }
      ],
      script: [
        // { src: 'https://example.com/script.js' }
      ]
    },

    // 頁面過渡效果
    pageTransition: { name: 'page', mode: 'out-in' },
    layoutTransition: { name: 'layout', mode: 'out-in' },

    // 基礎路徑（部署在子目錄時使用）
    baseURL: '/',

    // 建構輸出目錄
    buildAssetsDir: '/_nuxt/',
  },

  // CSS 配置
  css: [
    '~/assets/styles/main.css',
    '~/assets/styles/global.scss'
  ],

  // Vite 配置
  vite: {
    css: {
      preprocessorOptions: {
        scss: {
          additionalData: '@use "~/assets/styles/variables.scss" as *;'
        }
      }
    }
  }
})
```

### 模組配置

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  // Nuxt 模組
  modules: [
    '@nuxtjs/tailwindcss',  // Tailwind CSS
    '@pinia/nuxt',          // Pinia 狀態管理
    '@nuxt/content',        // 內容管理
    '@nuxt/image',          // 圖片優化
  ],

  // 模組特定配置
  tailwindcss: {
    cssPath: '~/assets/styles/tailwind.css',
    configPath: 'tailwind.config.ts',
  },

  content: {
    highlight: {
      theme: 'github-dark'
    }
  }
})
```

### 執行時配置

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  runtimeConfig: {
    // 私有配置（僅伺服器端）
    apiSecret: process.env.API_SECRET,
    databaseUrl: process.env.DATABASE_URL,

    // 公開配置（客戶端和伺服器端）
    public: {
      apiBase: process.env.API_BASE_URL || '/api',
      siteName: '我的網站',
      siteUrl: 'https://example.com'
    }
  }
})
```

使用執行時配置：

```vue
<script setup lang="ts">
const config = useRuntimeConfig()

// 在伺服器端可以訪問所有配置
// 在客戶端只能訪問 public 配置
console.log(config.public.apiBase)
</script>
```

### 路由配置

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  router: {
    options: {
      strict: true,  // 嚴格路由匹配
      sensitive: true // 區分大小寫
    }
  },

  // 路由規則
  routeRules: {
    // 靜態生成
    '/': { prerender: true },
    '/about': { prerender: true },

    // SWR (Stale-While-Revalidate)
    '/posts/**': { swr: 3600 }, // 1小時

    // 重新導向
    '/old-page': { redirect: '/new-page' },

    // SSR 配置
    '/admin/**': { ssr: false }, // 僅客戶端渲染
  }
})
```

### 完整配置範例

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  // 開發工具
  devtools: { enabled: true },

  // 應用程式配置
  app: {
    head: {
      title: '電商平台',
      htmlAttrs: { lang: 'zh-TW' },
      meta: [
        { charset: 'utf-8' },
        { name: 'viewport', content: 'width=device-width, initial-scale=1' },
        { name: 'description', content: '最好的線上購物平台' }
      ],
      link: [
        { rel: 'icon', type: 'image/x-icon', href: '/favicon.ico' }
      ]
    }
  },

  // CSS
  css: ['~/assets/styles/main.css'],

  // 模組
  modules: [
    '@nuxtjs/tailwindcss',
    '@pinia/nuxt'
  ],

  // TypeScript
  typescript: {
    strict: true,
    typeCheck: true
  },

  // 執行時配置
  runtimeConfig: {
    apiSecret: '',
    public: {
      apiBase: '/api'
    }
  },

  // 實驗性功能
  experimental: {
    payloadExtraction: true,
    renderJsonPayloads: true
  },

  // Nitro 配置（伺服器）
  nitro: {
    preset: 'node-server',
    compressPublicAssets: true
  }
})
```

## app.vue vs pages/

### app.vue 模式（單頁應用）

當專案中**沒有** `pages/` 目錄時，使用 `app.vue` 作為應用入口：

```vue
<!-- app.vue -->
<template>
  <div>
    <h1>單頁應用</h1>
    <p>這是一個沒有路由的應用</p>
  </div>
</template>

<script setup lang="ts">
// 這裡是應用的主要邏輯
const message = ref('Hello Nuxt 3')
</script>
```

**使用場景**：
- 簡單的單頁應用
- 不需要多個路由的應用
- 嵌入式應用

### pages/ 模式（多頁應用）

當專案中**有** `pages/` 目錄時，需要在 `app.vue` 中使用 `<NuxtPage>`：

```vue
<!-- app.vue -->
<template>
  <div>
    <header>
      <nav>
        <NuxtLink to="/">首頁</NuxtLink>
        <NuxtLink to="/about">關於</NuxtLink>
      </nav>
    </header>

    <main>
      <!-- 這裡會渲染當前路由對應的頁面 -->
      <NuxtPage />
    </main>

    <footer>
      <p>&copy; 2024 我的網站</p>
    </footer>
  </div>
</template>

<script setup lang="ts">
// 全域狀態或邏輯
const isLoading = ref(false)

// 路由監聽
const route = useRoute()
watch(() => route.path, (newPath) => {
  console.log('路由變更:', newPath)
})
</script>
```

### 兩者比較

| 特性 | app.vue 模式 | pages/ 模式 |
|------|-------------|-------------|
| 路由 | 無 | 檔案式路由 |
| `<NuxtPage>` | 不需要 | 必須 |
| 適用場景 | 單頁應用 | 多頁應用 |
| Layout | 不支援 | 支援 |

## .nuxt/ 與 .output/ 目錄

### .nuxt/ 目錄

開發階段自動生成的檔案：

```
.nuxt/
├── components.d.ts        # 元件類型定義
├── types/                 # 自動生成的類型
├── routes.mjs            # 路由配置
├── nuxt.d.ts             # Nuxt 類型增強
└── ...                   # 其他開發檔案
```

**不要**：
- 手動編輯這些檔案
- 提交到版本控制

**可以**：
- 查看類型定義
- 除錯路由問題

### .output/ 目錄

建構後的輸出檔案：

```
.output/
├── public/               # 靜態資源
│   ├── _nuxt/           # 建構後的 JS/CSS
│   └── ...
└── server/              # 伺服器端程式碼
    ├── chunks/
    └── index.mjs
```

**用途**：
- 生產環境部署
- 預覽建構結果

```bash
# 建構
npm run build

# 預覽建構結果
npm run preview
```

## 環境變數設定

### .env 檔案

```bash
# .env
API_SECRET=my-secret-key
API_BASE_URL=https://api.example.com
DATABASE_URL=postgresql://localhost/mydb

# 公開變數（NUXT_PUBLIC_ 前綴）
NUXT_PUBLIC_API_BASE=/api
NUXT_PUBLIC_SITE_URL=https://example.com
```

### 在配置中使用

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  runtimeConfig: {
    // 私有（僅伺服器端）
    apiSecret: process.env.API_SECRET,
    databaseUrl: process.env.DATABASE_URL,

    // 公開（客戶端和伺服器端）
    public: {
      apiBase: process.env.NUXT_PUBLIC_API_BASE,
      siteUrl: process.env.NUXT_PUBLIC_SITE_URL
    }
  }
})
```

### 在程式碼中使用

```vue
<script setup lang="ts">
// 客戶端和伺服器端都可使用
const config = useRuntimeConfig()
console.log(config.public.apiBase)

// 僅在伺服器端可用（API 路由、伺服器中介軟體等）
// console.log(config.apiSecret)
</script>
```

### 環境特定配置

```bash
# .env.development
NUXT_PUBLIC_API_BASE=http://localhost:3001/api

# .env.production
NUXT_PUBLIC_API_BASE=https://api.example.com
```

## TypeScript 設定

### tsconfig.json

Nuxt 自動生成並擴充 TypeScript 配置：

```json
{
  "extends": "./.nuxt/tsconfig.json",
  "compilerOptions": {
    "strict": true,
    "types": [
      "node"
    ]
  }
}
```

### 類型增強

```typescript
// types/index.d.ts
declare module '#app' {
  interface PageMeta {
    layout?: string
    requiresAuth?: boolean
    roles?: string[]
  }
}

export {}
```

### 自動類型推導

```vue
<script setup lang="ts">
// 自動推導類型
const count = ref(0) // Ref<number>
const message = ref('Hello') // Ref<string>

// 明確指定類型
interface User {
  id: number
  name: string
  email: string
}

const user = ref<User | null>(null)
const users = ref<User[]>([])
</script>
```

## 實際範例

### 完整的專案配置範例

建立一個部落格應用的完整配置：

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  devtools: { enabled: true },

  app: {
    head: {
      title: '我的部落格',
      htmlAttrs: { lang: 'zh-TW' },
      meta: [
        { charset: 'utf-8' },
        { name: 'viewport', content: 'width=device-width, initial-scale=1' },
        { name: 'description', content: '分享技術文章的部落格' }
      ]
    },
    pageTransition: { name: 'page', mode: 'out-in' }
  },

  css: ['~/assets/styles/main.css'],

  modules: [
    '@nuxtjs/tailwindcss',
    '@nuxt/content'
  ],

  content: {
    highlight: {
      theme: 'github-dark',
      preload: ['typescript', 'javascript', 'vue']
    },
    markdown: {
      toc: { depth: 3, searchDepth: 3 }
    }
  },

  runtimeConfig: {
    public: {
      siteUrl: 'https://myblog.com',
      siteName: '我的部落格',
      authorName: '張三'
    }
  },

  routeRules: {
    '/': { prerender: true },
    '/about': { prerender: true },
    '/posts/**': { swr: 3600 }
  },

  typescript: {
    strict: true,
    typeCheck: true
  }
})
```

### 目錄結構範例

```
blog-app/
├── assets/
│   └── styles/
│       └── main.css
├── components/
│   ├── blog/
│   │   ├── PostCard.vue
│   │   └── PostList.vue
│   └── common/
│       ├── Header.vue
│       └── Footer.vue
├── composables/
│   ├── useBlog.ts
│   └── useFormatDate.ts
├── content/
│   └── posts/
│       ├── 2024-01-01-first-post.md
│       └── 2024-01-02-second-post.md
├── layouts/
│   ├── default.vue
│   └── post.vue
├── pages/
│   ├── index.vue
│   ├── about.vue
│   └── posts/
│       ├── index.vue
│       └── [slug].vue
├── public/
│   ├── favicon.ico
│   └── images/
├── server/
│   └── api/
│       └── contact.post.ts
├── app.vue
├── nuxt.config.ts
└── package.json
```

### app.vue（使用 pages/）

```vue
<!-- app.vue -->
<template>
  <div id="app">
    <NuxtLoadingIndicator />
    <NuxtPage />
  </div>
</template>

<script setup lang="ts">
// 全域設定
useHead({
  titleTemplate: '%s - 我的部落格',
  meta: [
    { name: 'author', content: '張三' }
  ]
})
</script>

<style>
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
  line-height: 1.6;
  color: #333;
}

#app {
  min-height: 100vh;
}
</style>
```

### composable 範例

```typescript
// composables/useBlog.ts
export const useBlog = () => {
  const posts = ref<any[]>([])
  const loading = ref(false)
  const error = ref<Error | null>(null)

  const fetchPosts = async () => {
    loading.value = true
    error.value = null

    try {
      const { data } = await useAsyncData('posts', () =>
        queryContent('/posts').find()
      )
      posts.value = data.value || []
    } catch (e) {
      error.value = e as Error
    } finally {
      loading.value = false
    }
  }

  return {
    posts: readonly(posts),
    loading: readonly(loading),
    error: readonly(error),
    fetchPosts
  }
}
```

```typescript
// composables/useFormatDate.ts
export const useFormatDate = () => {
  const formatDate = (date: string | Date): string => {
    const d = new Date(date)
    return d.toLocaleDateString('zh-TW', {
      year: 'numeric',
      month: 'long',
      day: 'numeric'
    })
  }

  const formatDateTime = (date: string | Date): string => {
    const d = new Date(date)
    return d.toLocaleString('zh-TW')
  }

  return {
    formatDate,
    formatDateTime
  }
}
```

### 環境變數範例

```bash
# .env
NUXT_PUBLIC_SITE_URL=http://localhost:3000
NUXT_PUBLIC_SITE_NAME=我的部落格
NUXT_PUBLIC_AUTHOR_NAME=張三

# API 設定
API_SECRET=your-secret-key
```

使用：

```vue
<script setup lang="ts">
const config = useRuntimeConfig()

console.log(config.public.siteName) // "我的部落格"
console.log(config.public.authorName) // "張三"
</script>
```

## 最佳實踐建議

### 1. 目錄組織

```
✅ 推薦：按功能分組
components/
├── blog/
│   ├── PostCard.vue
│   └── PostList.vue
└── user/
    ├── Profile.vue
    └── Avatar.vue

❌ 不推薦：扁平結構
components/
├── BlogPostCard.vue
├── BlogPostList.vue
├── UserProfile.vue
└── UserAvatar.vue
```

### 2. 命名規範

```typescript
// ✅ 元件使用 PascalCase
components/TheHeader.vue → <TheHeader />
components/blog/PostCard.vue → <BlogPostCard />

// ✅ Composables 使用 use 前綴
composables/useAuth.ts → useAuth()

// ✅ Utils 使用描述性名稱
utils/formatDate.ts → formatDate()
```

### 3. TypeScript 優先

```vue
<script setup lang="ts">
// ✅ 定義清晰的介面
interface Post {
  id: number
  title: string
  content: string
  createdAt: Date
}

const posts = ref<Post[]>([])
</script>
```

### 4. 環境變數管理

```typescript
// ✅ 在 nuxt.config.ts 中集中管理
export default defineNuxtConfig({
  runtimeConfig: {
    apiSecret: process.env.API_SECRET,
    public: {
      apiBase: process.env.NUXT_PUBLIC_API_BASE
    }
  }
})

// ❌ 不要直接使用 process.env
// const apiKey = process.env.API_KEY
```

### 5. 自動導入配置

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  components: {
    dirs: [
      {
        path: '~/components',
        pathPrefix: false, // 移除路徑前綴
      }
    ]
  }
})
```

## 除錯技巧

### 1. 查看自動生成的類型

```bash
# 檢查 .nuxt/components.d.ts
cat .nuxt/components.d.ts
```

### 2. 驗證路由配置

```bash
# 查看生成的路由
cat .nuxt/routes.mjs
```

### 3. 類型檢查

```bash
# 執行 TypeScript 類型檢查
npm run typecheck

# 或在 nuxt.config.ts 中啟用
export default defineNuxtConfig({
  typescript: {
    typeCheck: true
  }
})
```

## 總結

本章節涵蓋了 Nuxt 3 專案結構的所有重要方面：

- ✅ 完整的目錄結構說明
- ✅ nuxt.config.ts 配置詳解
- ✅ app.vue vs pages/ 的區別
- ✅ 環境變數設定方法
- ✅ TypeScript 配置
- ✅ 實際專案範例

在下一章節中，我們將深入探討 Nuxt 3 的路由系統基礎。
