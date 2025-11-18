# Hybrid Rendering 與 Route Rules

## Hybrid Rendering 概念

Hybrid Rendering（混合渲染）是 Nuxt 3 的一個強大特性，允許你在**同一個應用**中為不同的路由使用不同的渲染策略。這意味著你可以：

- 首頁使用 SSG（快速載入）
- 產品列表使用 ISR（定期更新）
- 產品詳情使用 SSR（即時資料）
- 管理後台使用 CSR（無 SEO 需求）

### 為什麼需要 Hybrid Rendering？

```
傳統方式（單一渲染模式）:
整個網站 → SSR  ❌ 首頁不需要即時渲染，浪費資源
整個網站 → SSG  ❌ 產品價格無法即時更新
整個網站 → CSR  ❌ SEO 表現差

Hybrid 方式（混合渲染）:
首頁           → SSG  ✅ 極快載入
產品列表       → ISR  ✅ 定期更新
產品詳情       → SSR  ✅ 即時資料
用戶儀表板     → CSR  ✅ 無需 SEO
API 路由       → SWR  ✅ 快取優化
```

### 視覺化流程

```
使用者請求 → Nuxt Router
                ↓
        Route Rules 判斷
                ↓
    ┌───────────┴───────────┐
    ↓           ↓           ↓
  SSG         SSR         CSR
 (預渲染)   (即時渲染)  (客戶端)
    ↓           ↓           ↓
  HTML        HTML        空殼HTML
  (已快取)   (動態生成)   + JS Bundle
```

## Route Rules 設定

Route Rules 是實現 Hybrid Rendering 的核心機制，在 `nuxt.config.ts` 中設定。

### 基本語法

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  routeRules: {
    // 路由模式: { 渲染規則 }
    '/': { prerender: true },
    '/admin/**': { ssr: false },
    '/api/**': { cors: true }
  }
})
```

### 可用的渲染選項

```typescript
export default defineNuxtConfig({
  routeRules: {
    // 1. 預渲染 (SSG)
    '/': { prerender: true },
    '/about': { prerender: true },

    // 2. SWR (Stale-While-Revalidate)
    '/blog': {
      swr: 3600  // 快取 1 小時
    },

    // 3. ISR (Incremental Static Regeneration)
    '/products': {
      swr: true,  // 啟用 ISR
      isr: 600    // 每 10 分鐘重新生成
    },

    // 4. 純客戶端渲染 (CSR)
    '/dashboard/**': { ssr: false },

    // 5. SSR（預設）
    '/user/**': {
      // 不需要特別設定，SSR 是預設值
    },

    // 6. 重定向
    '/old-page': { redirect: '/new-page' },

    // 7. 自訂標頭
    '/api/**': {
      headers: {
        'cache-control': 'public, max-age=3600',
        'access-control-allow-origin': '*'
      }
    },

    // 8. CORS 設定
    '/api/public/**': {
      cors: true,
      headers: {
        'access-control-allow-methods': 'GET, POST'
      }
    }
  }
})
```

## 不同路由使用不同渲染模式

### 完整電商網站範例

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  routeRules: {
    // ==================== 公開頁面 ====================

    // 首頁 - SSG（靜態生成，極快載入）
    '/': {
      prerender: true,
      headers: {
        'cache-control': 'public, max-age=3600, must-revalidate'
      }
    },

    // 關於我們 - SSG
    '/about': { prerender: true },
    '/contact': { prerender: true },
    '/terms': { prerender: true },
    '/privacy': { prerender: true },

    // ==================== 部落格 ====================

    // 部落格首頁 - ISR（每小時更新）
    '/blog': {
      swr: 3600,  // 1 小時快取
      isr: 3600   // 每小時重新生成
    },

    // 部落格文章 - ISR（每 30 分鐘更新）
    '/blog/**': {
      swr: 1800,
      isr: 1800
    },

    // ==================== 產品 ====================

    // 產品列表 - ISR（每 10 分鐘更新）
    '/products': {
      swr: 600,
      isr: 600
    },

    // 分類頁面 - ISR（每 10 分鐘更新）
    '/category/**': {
      swr: 600,
      isr: 600
    },

    // 產品詳情 - SSR（即時庫存和價格）
    '/products/**': {
      // SSR 是預設值，但可以明確設定
      ssr: true
    },

    // ==================== 用戶相關 ====================

    // 購物車 - CSR（個人化內容，無 SEO 需求）
    '/cart': { ssr: false },

    // 結帳 - CSR
    '/checkout/**': { ssr: false },

    // 用戶儀表板 - CSR
    '/dashboard/**': { ssr: false },
    '/account/**': { ssr: false },
    '/orders/**': { ssr: false },

    // ==================== API 路由 ====================

    // 公開 API - CORS + 快取
    '/api/products': {
      cors: true,
      swr: 300,  // 5 分鐘快取
      headers: {
        'cache-control': 'public, max-age=300'
      }
    },

    // 私有 API - 無快取
    '/api/user/**': {
      cors: true,
      headers: {
        'cache-control': 'private, no-cache'
      }
    },

    // ==================== 搜尋 ====================

    // 搜尋頁面 - SSR（動態查詢）
    '/search': { ssr: true },

    // ==================== 其他 ====================

    // 舊路由重定向
    '/old-shop': { redirect: '/products' },
    '/blog/old-post': { redirect: '/blog/new-post' }
  }
})
```

### 實際頁面實作

#### 1. 首頁（SSG）

```vue
<!-- pages/index.vue -->
<template>
  <div class="homepage">
    <Head>
      <Title>首頁 - 我的電商網站</Title>
      <Meta name="description" content="最優質的線上購物體驗" />
    </Head>

    <!-- Hero 區塊 -->
    <section class="hero">
      <h1>歡迎來到我們的商店</h1>
      <p>發現最新的產品和優惠</p>
      <NuxtLink to="/products" class="cta-button">
        立即購物
      </NuxtLink>
    </section>

    <!-- 特色產品（建置時獲取） -->
    <section class="featured-products">
      <h2>熱門產品</h2>
      <div class="product-grid">
        <ProductCard
          v-for="product in featuredProducts"
          :key="product.id"
          :product="product"
        />
      </div>
    </section>

    <!-- 分類 -->
    <section class="categories">
      <h2>熱門分類</h2>
      <div class="category-grid">
        <CategoryCard
          v-for="category in categories"
          :key="category.id"
          :category="category"
        />
      </div>
    </section>
  </div>
</template>

<script setup>
// 在建置時獲取資料（SSG）
const { data: featuredProducts } = await useFetch('/api/products/featured')
const { data: categories } = await useFetch('/api/categories')
</script>
```

#### 2. 產品列表（ISR）

```vue
<!-- pages/products/index.vue -->
<template>
  <div class="products-page">
    <Head>
      <Title>所有產品 - 我的電商網站</Title>
    </Head>

    <h1>所有產品</h1>

    <!-- 篩選器 -->
    <div class="filters">
      <select v-model="selectedCategory" @change="filterProducts">
        <option value="">所有分類</option>
        <option v-for="cat in categories" :key="cat.id" :value="cat.id">
          {{ cat.name }}
        </option>
      </select>

      <select v-model="sortBy" @change="sortProducts">
        <option value="name">名稱</option>
        <option value="price-asc">價格: 低到高</option>
        <option value="price-desc">價格: 高到低</option>
      </select>
    </div>

    <!-- 產品網格 -->
    <div class="products-grid">
      <ProductCard
        v-for="product in displayProducts"
        :key="product.id"
        :product="product"
      />
    </div>

    <!-- 分頁 -->
    <Pagination
      :current-page="currentPage"
      :total-pages="totalPages"
      @page-change="changePage"
    />
  </div>
</template>

<script setup>
// ISR: 每 10 分鐘重新生成
// 首次請求使用快取，背景重新生成
const { data: products } = await useFetch('/api/products', {
  key: 'products-list'
})

const { data: categories } = await useFetch('/api/categories')

// 客戶端狀態
const selectedCategory = ref('')
const sortBy = ref('name')
const currentPage = ref(1)
const itemsPerPage = 12

// 計算屬性
const filteredProducts = computed(() => {
  if (!products.value) return []

  let result = products.value

  if (selectedCategory.value) {
    result = result.filter(p => p.categoryId === selectedCategory.value)
  }

  return result
})

const sortedProducts = computed(() => {
  const arr = [...filteredProducts.value]

  switch (sortBy.value) {
    case 'price-asc':
      return arr.sort((a, b) => a.price - b.price)
    case 'price-desc':
      return arr.sort((a, b) => b.price - a.price)
    default:
      return arr.sort((a, b) => a.name.localeCompare(b.name))
  }
})

const displayProducts = computed(() => {
  const start = (currentPage.value - 1) * itemsPerPage
  const end = start + itemsPerPage
  return sortedProducts.value.slice(start, end)
})

const totalPages = computed(() => {
  return Math.ceil(sortedProducts.value.length / itemsPerPage)
})

const filterProducts = () => {
  currentPage.value = 1
}

const sortProducts = () => {
  currentPage.value = 1
}

const changePage = (page) => {
  currentPage.value = page
  window.scrollTo({ top: 0, behavior: 'smooth' })
}
</script>
```

#### 3. 產品詳情（SSR）

```vue
<!-- pages/products/[id].vue -->
<template>
  <div class="product-detail">
    <Head>
      <Title>{{ product?.name }} - 我的電商網站</Title>
      <Meta name="description" :content="product?.description" />
    </Head>

    <div v-if="product" class="product-container">
      <!-- 即時庫存狀態（SSR 每次請求都是最新的） -->
      <div class="stock-status" :class="stockClass">
        <span v-if="product.stock > 10">✅ 庫存充足</span>
        <span v-else-if="product.stock > 0">⚠️ 剩餘 {{ product.stock }} 件</span>
        <span v-else>❌ 已售完</span>
      </div>

      <!-- 即時價格（可能有折扣） -->
      <div class="price-info">
        <span v-if="product.discountPrice" class="original-price">
          原價: ${{ product.price }}
        </span>
        <span class="current-price">
          ${{ product.discountPrice || product.price }}
        </span>
        <span v-if="product.discountPercent" class="discount-badge">
          {{ product.discountPercent }}% OFF
        </span>
      </div>

      <!-- 產品資訊 -->
      <h1>{{ product.name }}</h1>
      <p>{{ product.description }}</p>

      <!-- 加入購物車 -->
      <button
        @click="addToCart"
        :disabled="product.stock === 0"
        class="add-to-cart-btn"
      >
        {{ product.stock > 0 ? '加入購物車' : '已售完' }}
      </button>
    </div>
  </div>
</template>

<script setup>
const route = useRoute()
const productId = route.params.id

// SSR: 每次請求都重新獲取最新資料
const { data: product } = await useFetch(`/api/products/${productId}`, {
  // 不使用快取，確保資料即時性
  key: `product-${productId}-${Date.now()}`
})

const stockClass = computed(() => {
  if (!product.value) return ''
  if (product.value.stock > 10) return 'in-stock'
  if (product.value.stock > 0) return 'low-stock'
  return 'out-of-stock'
})

const addToCart = async () => {
  await $fetch('/api/cart/add', {
    method: 'POST',
    body: {
      productId: product.value.id,
      quantity: 1
    }
  })
  alert('已加入購物車！')
}
</script>
```

#### 4. 用戶儀表板（CSR）

```vue
<!-- pages/dashboard/index.vue -->
<template>
  <div class="dashboard">
    <!-- CSR: 不會在伺服器端渲染 -->
    <ClientOnly>
      <div v-if="isAuthenticated">
        <h1>歡迎回來，{{ user.name }}</h1>

        <div class="dashboard-grid">
          <!-- 訂單摘要 -->
          <div class="card">
            <h2>最近訂單</h2>
            <OrderList :orders="recentOrders" />
          </div>

          <!-- 收藏清單 -->
          <div class="card">
            <h2>我的收藏</h2>
            <WishList :items="wishlist" />
          </div>

          <!-- 帳戶設定 -->
          <div class="card">
            <h2>帳戶設定</h2>
            <AccountSettings :user="user" />
          </div>
        </div>
      </div>

      <div v-else>
        <p>請先登入</p>
        <button @click="login">登入</button>
      </div>

      <template #fallback>
        <div class="loading">載入中...</div>
      </template>
    </ClientOnly>
  </div>
</template>

<script setup>
// CSR: 這些代碼只在客戶端執行
definePageMeta({
  middleware: 'auth' // 客戶端路由守衛
})

const isAuthenticated = ref(false)
const user = ref(null)
const recentOrders = ref([])
const wishlist = ref([])

onMounted(async () => {
  // 檢查登入狀態
  const authToken = localStorage.getItem('authToken')

  if (authToken) {
    try {
      // 獲取用戶資料
      const userData = await $fetch('/api/user/me', {
        headers: {
          Authorization: `Bearer ${authToken}`
        }
      })

      user.value = userData
      isAuthenticated.value = true

      // 並行獲取儀表板資料
      const [orders, wishes] = await Promise.all([
        $fetch('/api/orders/recent'),
        $fetch('/api/wishlist')
      ])

      recentOrders.value = orders
      wishlist.value = wishes
    } catch (error) {
      console.error('獲取用戶資料失敗:', error)
      localStorage.removeItem('authToken')
    }
  }
})

const login = () => {
  navigateTo('/login')
}
</script>
```

## ISR (Incremental Static Regeneration)

ISR 允許你在靜態網站中增量更新頁面，無需重新建置整個網站。

### ISR 工作原理

```
1. 首次請求
   用戶 → CDN (無快取) → 伺服器渲染 → 回傳 HTML → 儲存到 CDN

2. 後續請求（快取期間內）
   用戶 → CDN → 回傳快取的 HTML ⚡ (極快)

3. 快取過期後的請求
   用戶 → CDN → 回傳舊的 HTML ⚡ (仍然快)
   背景 → 重新渲染 → 更新 CDN 快取

4. 下次請求
   用戶 → CDN → 回傳新的 HTML ⚡
```

### ISR 設定範例

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  routeRules: {
    // 新聞頁面 - 每 5 分鐘重新生成
    '/news/**': {
      swr: 300,
      isr: 300
    },

    // 產品頁面 - 每 1 小時重新生成
    '/products/**': {
      swr: 3600,
      isr: 3600
    },

    // 部落格 - 每天重新生成
    '/blog/**': {
      swr: 86400,
      isr: 86400
    }
  }
})
```

### ISR 實際應用

```vue
<!-- pages/news/[id].vue -->
<template>
  <article class="news-article">
    <Head>
      <Title>{{ article?.title }}</Title>
    </Head>

    <!-- 顯示最後更新時間 -->
    <div class="update-info">
      最後更新: {{ lastUpdated }}
    </div>

    <h1>{{ article?.title }}</h1>
    <div class="meta">
      <span>{{ article?.author }}</span>
      <span>{{ article?.publishedAt }}</span>
    </div>

    <div v-html="article?.content"></div>

    <!-- 即時評論數（從 API 獲取） -->
    <div class="comments-count">
      <ClientOnly>
        <LiveCommentsCount :article-id="article?.id" />
      </ClientOnly>
    </div>
  </article>
</template>

<script setup>
const route = useRoute()
const articleId = route.params.id

// ISR: 使用 SWR 策略
const { data: article } = await useFetch(`/api/news/${articleId}`, {
  key: `news-${articleId}`
})

// 記錄頁面生成時間
const lastUpdated = new Date().toLocaleString('zh-TW')
</script>
```

## SWR (Stale-While-Revalidate)

SWR 是一種快取策略，允許立即回傳快取的內容（即使過期），同時在背景重新驗證。

### SWR vs 傳統快取

```
傳統快取:
請求 → 快取未過期 → 回傳快取 ✅
請求 → 快取過期 → 等待重新獲取 → 回傳新內容 🐢 (慢)

SWR:
請求 → 快取未過期 → 回傳快取 ✅
請求 → 快取過期 → 立即回傳舊快取 ⚡ + 背景更新 → 下次請求使用新內容 ✅
```

### SWR 設定

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  routeRules: {
    // 部落格列表 - SWR 1 小時
    '/blog': {
      swr: 3600 // 秒數
    },

    // API 路由 - SWR 10 分鐘
    '/api/posts': {
      swr: 600,
      headers: {
        'cache-control': 'public, max-age=600, stale-while-revalidate=3600'
      }
    },

    // 產品 API - SWR 5 分鐘，過期後最多使用 1 小時的舊快取
    '/api/products': {
      swr: 300,
      headers: {
        'cache-control': 'public, max-age=300, stale-while-revalidate=3600'
      }
    }
  }
})
```

### 客戶端 SWR（使用 VueUse）

```vue
<script setup>
import { useFetch as useVueFetch } from '@vueuse/core'

// 使用 VueUse 的 SWR 模式
const { data: products, isFetching, refetch } = useVueFetch('/api/products', {
  refetch: true,        // 啟用重新獲取
  refetchInterval: 60000 // 每 60 秒自動重新獲取
}).json()

// 手動重新整理
const refresh = () => {
  refetch()
}
</script>

<template>
  <div>
    <button @click="refresh" :disabled="isFetching">
      {{ isFetching ? '更新中...' : '重新整理' }}
    </button>

    <ProductList :products="products" />
  </div>
</template>
```

## prerender 選項

`prerender` 選項明確指定路由在建置時預渲染。

### 基本用法

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  routeRules: {
    // 明確預渲染
    '/': { prerender: true },
    '/about': { prerender: true },
    '/contact': { prerender: true },

    // 預渲染整個部落格目錄
    '/blog/**': { prerender: true },

    // 不要預渲染
    '/admin/**': { prerender: false },

    // 預渲染 + 快取設定
    '/docs/**': {
      prerender: true,
      headers: {
        'cache-control': 'public, max-age=86400' // 1 天
      }
    }
  }
})
```

### 條件預渲染

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  routeRules: {
    // 根據環境決定是否預渲染
    '/analytics': {
      prerender: process.env.NODE_ENV === 'production'
    }
  },

  hooks: {
    'nitro:config'(nitroConfig) {
      // 動態決定預渲染路由
      if (process.env.PRERENDER_BLOG === 'true') {
        nitroConfig.prerender = nitroConfig.prerender || {}
        nitroConfig.prerender.routes = [
          ...(nitroConfig.prerender.routes || []),
          '/blog'
        ]
      }
    }
  }
})
```

## 實際混合渲染範例

### 完整電商網站架構

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  routeRules: {
    // ========== 靜態頁面（SSG）==========
    '/': { prerender: true },
    '/about': { prerender: true },
    '/contact': { prerender: true },
    '/faq': { prerender: true },

    // ========== 內容頁面（ISR）==========
    // 部落格 - 每天更新一次
    '/blog': {
      swr: 86400,
      isr: 86400
    },
    '/blog/**': {
      swr: 86400,
      isr: 86400
    },

    // 分類頁面 - 每小時更新
    '/category/**': {
      swr: 3600,
      isr: 3600
    },

    // 產品列表 - 每 30 分鐘更新
    '/products': {
      swr: 1800,
      isr: 1800
    },

    // ========== 動態頁面（SSR）==========
    // 產品詳情 - 即時庫存和價格
    '/products/**': {
      ssr: true
    },

    // 搜尋結果 - 即時查詢
    '/search': {
      ssr: true
    },

    // ========== 客戶端頁面（CSR）==========
    // 用戶相關頁面
    '/cart': { ssr: false },
    '/checkout/**': { ssr: false },
    '/account/**': { ssr: false },
    '/dashboard/**': { ssr: false },

    // ========== API 路由 ==========
    // 公開 API - 快取
    '/api/products': {
      swr: 600,
      cors: true
    },
    '/api/categories': {
      swr: 3600,
      cors: true
    },

    // 私有 API - 不快取
    '/api/user/**': {
      headers: {
        'cache-control': 'private, no-cache'
      }
    },
    '/api/cart/**': {
      headers: {
        'cache-control': 'private, no-cache'
      }
    }
  }
})
```

### 監控和偵錯

```vue
<!-- components/RenderingInfo.vue -->
<template>
  <div v-if="showInfo" class="rendering-info">
    <h4>渲染資訊</h4>
    <p>渲染模式: {{ renderMode }}</p>
    <p>環境: {{ environment }}</p>
    <p>時間戳: {{ timestamp }}</p>
  </div>
</template>

<script setup>
const showInfo = process.env.NODE_ENV === 'development'

const renderMode = computed(() => {
  // 偵測渲染模式
  if (process.server && process.static) return 'SSG (建置時)'
  if (process.server) return 'SSR (請求時)'
  if (process.client) return 'CSR (客戶端)'
  return '未知'
})

const environment = process.server ? '伺服器' : '客戶端'
const timestamp = new Date().toISOString()
</script>

<style scoped>
.rendering-info {
  position: fixed;
  bottom: 20px;
  right: 20px;
  background: rgba(0, 0, 0, 0.8);
  color: white;
  padding: 15px;
  border-radius: 8px;
  font-size: 12px;
  z-index: 9999;
}
</style>
```

## 選擇適合的渲染策略

### 決策流程圖

```
開始
  ↓
需要 SEO？
  ↓ 是
內容多久更新一次？
  ↓
每天或更久 → SSG ✅
  ↓
每小時 → ISR ✅
  ↓
即時 → SSR ✅

需要 SEO？
  ↓ 否
個人化內容？
  ↓ 是
CSR ✅
  ↓ 否
考慮效能 → SSG 或 ISR ✅
```

### 渲染模式選擇表

| 頁面類型 | 推薦模式 | 原因 |
|---------|---------|------|
| 首頁 | SSG | 靜態內容，極快載入 |
| 關於我們 | SSG | 很少變動 |
| 部落格列表 | ISR | 定期新增文章 |
| 部落格文章 | ISR | 偶爾更新 |
| 產品列表 | ISR | 定期更新 |
| 產品詳情 | SSR | 即時庫存/價格 |
| 搜尋頁面 | SSR | 動態查詢 |
| 購物車 | CSR | 個人化，無 SEO |
| 結帳流程 | CSR | 個人化，安全性 |
| 用戶儀表板 | CSR | 個人化，無 SEO |
| API (公開) | SWR | 快取優化 |
| API (私有) | 無快取 | 安全性 |

### 效能比較

```
場景: 電商網站 (1000 個產品)

全 SSG:
✅ TTFB: 極快 (10-50ms)
❌ 建置時間: 長 (10 分鐘+)
❌ 內容更新: 需重新建置
成本: 極低

全 SSR:
❌ TTFB: 較慢 (200-500ms)
✅ 建置時間: 短 (1 分鐘)
✅ 內容更新: 即時
成本: 中等

Hybrid (推薦):
✅ TTFB: 快 (50-200ms)
✅ 建置時間: 短 (1-2 分鐘)
✅ 內容更新: 靈活
成本: 低

配置建議:
- 首頁: SSG
- 產品列表: ISR (10 分鐘)
- 產品詳情: SSR
- 用戶頁面: CSR
```

## 最佳實踐

### 1. 合理設定快取時間

```typescript
export default defineNuxtConfig({
  routeRules: {
    // ❌ 過短 - 伺服器壓力大
    '/products': { swr: 10 },

    // ❌ 過長 - 內容不夠即時
    '/products': { swr: 604800 }, // 7 天

    // ✅ 適中 - 根據更新頻率設定
    '/products': { swr: 1800 }, // 30 分鐘

    // ✅ 根據內容特性調整
    '/news': { swr: 300 },      // 5 分鐘（即時性高）
    '/docs': { swr: 86400 },    // 1 天（很少變動）
  }
})
```

### 2. 使用分層快取策略

```typescript
export default defineNuxtConfig({
  routeRules: {
    // 首頁 - 靜態 + CDN 長期快取
    '/': {
      prerender: true,
      headers: {
        'cache-control': 'public, max-age=3600, s-maxage=86400'
      }
    },

    // API - 短期快取 + SWR
    '/api/products': {
      swr: 600,
      headers: {
        'cache-control': 'public, max-age=600, stale-while-revalidate=3600'
      }
    },

    // 用戶 API - 私有，無快取
    '/api/user/**': {
      headers: {
        'cache-control': 'private, no-cache, no-store, must-revalidate'
      }
    }
  }
})
```

### 3. 監控和記錄

```typescript
// server/middleware/logging.ts
export default defineEventHandler((event) => {
  const url = getRequestURL(event)
  const start = Date.now()

  // 記錄請求
  console.log(`[${new Date().toISOString()}] ${event.method} ${url.pathname}`)

  // 在回應時記錄時間
  event.node.res.on('finish', () => {
    const duration = Date.now() - start
    console.log(`[${url.pathname}] 完成，耗時: ${duration}ms`)
  })
})
```

### 4. 優雅降級

```vue
<template>
  <div>
    <!-- 優先顯示 SSR/SSG 內容 -->
    <div v-if="initialData">
      <ProductList :products="initialData" />
    </div>

    <!-- 客戶端增強 -->
    <ClientOnly>
      <div v-if="realtimeData">
        <LiveStockIndicator :products="realtimeData" />
      </div>
    </ClientOnly>
  </div>
</template>

<script setup>
// SSR/SSG 資料
const { data: initialData } = await useFetch('/api/products')

// 客戶端即時資料
const realtimeData = ref(null)

onMounted(async () => {
  // 客戶端獲取即時庫存
  realtimeData.value = await $fetch('/api/products/stock')

  // 定期更新
  setInterval(async () => {
    realtimeData.value = await $fetch('/api/products/stock')
  }, 30000) // 每 30 秒
})
</script>
```

### 5. A/B 測試不同策略

```typescript
// nuxt.config.ts
const useISR = process.env.EXPERIMENT_ISR === 'true'

export default defineNuxtConfig({
  routeRules: {
    '/products': useISR
      ? { swr: 1800, isr: 1800 }  // 實驗組: ISR
      : { ssr: true }              // 對照組: SSR
  }
})
```

## 總結

Hybrid Rendering 是 Nuxt 3 最強大的特性之一：

### 核心優勢
- 🎯 **靈活性**: 不同路由使用最適合的渲染策略
- ⚡ **效能**: 結合各種模式的優點
- 💰 **成本**: 最佳化伺服器和 CDN 使用
- 🔧 **可維護**: 集中設定，易於調整

### 渲染策略總結

| 模式 | 適用場景 | 優點 | 缺點 |
|------|---------|------|------|
| SSG | 靜態內容 | 極快、低成本 | 更新需重建 |
| ISR | 半靜態內容 | 快、自動更新 | 略複雜 |
| SSR | 動態內容 | 即時、SEO | 較慢、成本高 |
| CSR | 個人化內容 | 互動性強 | SEO 差、慢 |
| SWR | API/資料 | 快取優化 | 可能過期 |

### 實務建議

1. **從 SSG 開始**: 預設使用 SSG，有需要再改
2. **監控效能**: 使用 Lighthouse、WebPageTest
3. **漸進增強**: 先 SSR/SSG，再加 CSR 互動
4. **測試快取**: 驗證快取策略是否符合預期
5. **文件化**: 記錄為什麼選擇特定策略

### 下一步

你現在已經掌握了 Nuxt 3 的三大渲染模式（SSR、SSG、Hybrid），可以根據需求選擇最適合的策略來建構高效能的 Web 應用！

**推薦學習路徑**:
1. 先熟悉 SSR 和 SSG
2. 理解 ISR 和 SWR 的差異
3. 實作混合渲染專案
4. 監控和優化效能
5. 根據實際需求調整策略

記住：**沒有一種渲染模式適合所有場景，Hybrid Rendering 讓你可以為每個路由選擇最佳策略！** 🚀
