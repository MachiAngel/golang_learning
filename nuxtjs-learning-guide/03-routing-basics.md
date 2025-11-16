# 路由系統基礎

## 檔案式路由（File-based Routing）

Nuxt 3 使用檔案系統來定義路由，不需要手動配置路由表。`pages/` 目錄中的每個 Vue 檔案都會自動生成對應的路由。

### 基本概念

```
pages/
├── index.vue           → /
├── about.vue          → /about
├── contact.vue        → /contact
└── products.vue       → /products
```

這比 Vue Router 的傳統配置方式簡單得多：

```typescript
// ❌ Vue Router 傳統方式（手動配置）
const routes = [
  { path: '/', component: () => import('./pages/index.vue') },
  { path: '/about', component: () => import('./pages/about.vue') },
  { path: '/contact', component: () => import('./pages/contact.vue') },
]

// ✅ Nuxt 3 方式（自動生成）
// 只需建立 pages/ 目錄下的檔案，路由自動生成
```

### 檔案式路由的優勢

1. **零配置**：不需要維護路由配置檔案
2. **自動代碼分割**：每個頁面自動分割成獨立的 chunk
3. **類型安全**：自動生成路由類型定義
4. **易於重構**：重新命名檔案即可更改路由

## pages/ 目錄結構

### 基本頁面

```vue
<!-- pages/index.vue -->
<template>
  <div class="home">
    <h1>首頁</h1>
    <p>歡迎來到我的網站</p>
  </div>
</template>

<script setup lang="ts">
// 頁面 meta 設定
definePageMeta({
  title: '首頁',
  description: '這是首頁'
})

// SEO 設定
useHead({
  title: '首頁',
  meta: [
    { name: 'description', content: '歡迎來到我的網站' }
  ]
})
</script>
```

### 巢狀目錄結構

```
pages/
├── index.vue                    → /
├── about.vue                   → /about
├── products/
│   ├── index.vue               → /products
│   ├── electronics.vue         → /products/electronics
│   └── clothing.vue            → /products/clothing
├── blog/
│   ├── index.vue               → /blog
│   └── [id].vue                → /blog/:id
└── user/
    ├── profile.vue             → /user/profile
    └── settings/
        ├── index.vue           → /user/settings
        ├── account.vue         → /user/settings/account
        └── privacy.vue         → /user/settings/privacy
```

### 完整範例：產品頁面

```vue
<!-- pages/products/index.vue -->
<template>
  <div class="products-page">
    <h1>產品列表</h1>

    <div class="products-grid">
      <div
        v-for="product in products"
        :key="product.id"
        class="product-card"
      >
        <h3>{{ product.name }}</h3>
        <p class="price">${{ product.price }}</p>
        <NuxtLink :to="`/products/${product.id}`" class="btn">
          查看詳情
        </NuxtLink>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
interface Product {
  id: number
  name: string
  price: number
  category: string
}

const products = ref<Product[]>([
  { id: 1, name: '筆記型電腦', price: 30000, category: 'electronics' },
  { id: 2, name: '智慧手機', price: 15000, category: 'electronics' },
  { id: 3, name: 'T恤', price: 500, category: 'clothing' },
])

useHead({
  title: '產品列表',
  meta: [
    { name: 'description', content: '瀏覽我們的產品' }
  ]
})
</script>

<style scoped>
.products-page {
  max-width: 1200px;
  margin: 0 auto;
  padding: 2rem;
}

h1 {
  color: #00DC82;
  margin-bottom: 2rem;
}

.products-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(280px, 1fr));
  gap: 2rem;
}

.product-card {
  padding: 1.5rem;
  border: 1px solid #e0e0e0;
  border-radius: 8px;
  transition: box-shadow 0.3s;
}

.product-card:hover {
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);
}

.price {
  font-size: 1.5rem;
  font-weight: bold;
  color: #00DC82;
  margin: 1rem 0;
}

.btn {
  display: inline-block;
  padding: 0.5rem 1rem;
  background: #00DC82;
  color: white;
  text-decoration: none;
  border-radius: 4px;
  transition: background 0.3s;
}

.btn:hover {
  background: #00b36b;
}
</style>
```

## NuxtLink 組件使用

`<NuxtLink>` 是 Nuxt 提供的路由導航組件，它是 Vue Router 的 `<RouterLink>` 的增強版本。

### 基本用法

```vue
<template>
  <nav>
    <!-- 基本連結 -->
    <NuxtLink to="/">首頁</NuxtLink>
    <NuxtLink to="/about">關於我們</NuxtLink>
    <NuxtLink to="/products">產品</NuxtLink>

    <!-- 動態連結 -->
    <NuxtLink :to="`/products/${productId}`">
      產品詳情
    </NuxtLink>

    <!-- 使用物件語法 -->
    <NuxtLink :to="{ path: '/products', query: { category: 'electronics' } }">
      電子產品
    </NuxtLink>

    <!-- 外部連結（自動檢測） -->
    <NuxtLink to="https://nuxt.com" target="_blank" rel="noopener">
      Nuxt 官網
    </NuxtLink>
  </nav>
</template>

<script setup lang="ts">
const productId = ref(123)
</script>
```

### 樣式與 Active 狀態

```vue
<template>
  <nav class="main-nav">
    <NuxtLink
      to="/"
      class="nav-link"
      active-class="active"
      exact-active-class="exact-active"
    >
      首頁
    </NuxtLink>

    <NuxtLink to="/about" class="nav-link">
      關於
    </NuxtLink>

    <NuxtLink to="/products" class="nav-link">
      產品
    </NuxtLink>
  </nav>
</template>

<style scoped>
.main-nav {
  display: flex;
  gap: 1rem;
  padding: 1rem;
  background: #f5f5f5;
}

.nav-link {
  padding: 0.5rem 1rem;
  text-decoration: none;
  color: #333;
  border-radius: 4px;
  transition: all 0.3s;
}

.nav-link:hover {
  background: #e0e0e0;
}

/* 當路由匹配時的樣式（包含子路由） */
.nav-link.active {
  color: #00DC82;
}

/* 當路由完全匹配時的樣式 */
.nav-link.exact-active {
  background: #00DC82;
  color: white;
}
</style>
```

### 預載與預取

```vue
<template>
  <!-- 預取連結（滑鼠懸停時載入） -->
  <NuxtLink to="/products" prefetch>
    產品列表
  </NuxtLink>

  <!-- 不預取 -->
  <NuxtLink to="/admin" :prefetch="false">
    管理後台
  </NuxtLink>

  <!-- 自定義預取行為 -->
  <NuxtLink
    to="/blog"
    prefetch
    @mouseenter="handleMouseEnter"
  >
    部落格
  </NuxtLink>
</template>

<script setup lang="ts">
const handleMouseEnter = () => {
  console.log('預載頁面資源')
}
</script>
```

## 程式化導航

使用 `useRouter()` 和 `navigateTo()` 進行程式化導航。

### useRouter()

```vue
<template>
  <div>
    <button @click="goToAbout">關於我們</button>
    <button @click="goBack">返回</button>
    <button @click="goToProduct(123)">查看產品 #123</button>
  </div>
</template>

<script setup lang="ts">
const router = useRouter()

// 導航到指定路徑
const goToAbout = () => {
  router.push('/about')
}

// 返回上一頁
const goBack = () => {
  router.back()
}

// 前進一頁
const goForward = () => {
  router.forward()
}

// 導航到動態路由
const goToProduct = (id: number) => {
  router.push(`/products/${id}`)
}

// 使用物件語法
const goToProducts = () => {
  router.push({
    path: '/products',
    query: { category: 'electronics', sort: 'price' }
  })
}

// 替換當前記錄（不會在歷史記錄中留下新記錄）
const replaceRoute = () => {
  router.replace('/new-page')
}
</script>
```

### navigateTo()

`navigateTo()` 是 Nuxt 特有的導航助手，可在伺服器端和客戶端使用：

```vue
<script setup lang="ts">
// 基本用法
const handleLogin = async () => {
  const success = await login()
  if (success) {
    await navigateTo('/dashboard')
  }
}

// 外部 URL
const goToExternal = () => {
  navigateTo('https://nuxt.com', {
    external: true,
    open: {
      target: '_blank'
    }
  })
}

// 替換歷史記錄
const redirect = () => {
  navigateTo('/new-page', { replace: true })
}

// 伺服器端重新導向
const serverRedirect = () => {
  // 在伺服器端會返回 302 重新導向
  navigateTo('/login', { redirectCode: 302 })
}
</script>
```

### 實際範例：表單提交後導航

```vue
<template>
  <div class="form-container">
    <h2>建立新文章</h2>

    <form @submit.prevent="handleSubmit">
      <div class="form-group">
        <label for="title">標題</label>
        <input
          id="title"
          v-model="form.title"
          type="text"
          required
        />
      </div>

      <div class="form-group">
        <label for="content">內容</label>
        <textarea
          id="content"
          v-model="form.content"
          required
        ></textarea>
      </div>

      <div class="actions">
        <button type="submit" :disabled="isSubmitting">
          {{ isSubmitting ? '提交中...' : '提交' }}
        </button>
        <button type="button" @click="handleCancel">
          取消
        </button>
      </div>
    </form>
  </div>
</template>

<script setup lang="ts">
const router = useRouter()

interface PostForm {
  title: string
  content: string
}

const form = reactive<PostForm>({
  title: '',
  content: ''
})

const isSubmitting = ref(false)

const handleSubmit = async () => {
  isSubmitting.value = true

  try {
    // 模擬 API 請求
    const response = await $fetch('/api/posts', {
      method: 'POST',
      body: form
    })

    // 成功後導航到文章詳情頁
    await navigateTo(`/posts/${response.id}`)
  } catch (error) {
    console.error('提交失敗:', error)
    alert('提交失敗，請稍後再試')
  } finally {
    isSubmitting.value = false
  }
}

const handleCancel = () => {
  if (confirm('確定要取消嗎？未儲存的變更將會遺失。')) {
    router.back()
  }
}
</script>

<style scoped>
.form-container {
  max-width: 600px;
  margin: 2rem auto;
  padding: 2rem;
}

.form-group {
  margin-bottom: 1.5rem;
}

label {
  display: block;
  margin-bottom: 0.5rem;
  font-weight: 600;
}

input, textarea {
  width: 100%;
  padding: 0.75rem;
  border: 1px solid #ddd;
  border-radius: 4px;
  font-size: 1rem;
}

textarea {
  min-height: 200px;
  resize: vertical;
}

.actions {
  display: flex;
  gap: 1rem;
}

button {
  padding: 0.75rem 1.5rem;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-size: 1rem;
}

button[type="submit"] {
  background: #00DC82;
  color: white;
}

button[type="submit"]:disabled {
  background: #ccc;
  cursor: not-allowed;
}

button[type="button"] {
  background: #f5f5f5;
  color: #333;
}
</style>
```

## 路由參數與查詢字串

### 取得路由資訊

```vue
<script setup lang="ts">
const route = useRoute()

// 取得路由參數
console.log(route.params.id) // 動態路由參數

// 取得查詢字串
console.log(route.query.search) // ?search=keyword
console.log(route.query.page) // ?page=2

// 取得完整路徑
console.log(route.path) // /products/123

// 取得路由名稱
console.log(route.name) // products-id

// 取得 hash
console.log(route.hash) // #section1
</script>
```

### 監聽路由變化

```vue
<script setup lang="ts">
const route = useRoute()

// 監聽整個路由變化
watch(() => route.path, (newPath, oldPath) => {
  console.log(`從 ${oldPath} 導航到 ${newPath}`)
})

// 監聽特定參數
watch(() => route.params.id, (newId) => {
  console.log('產品 ID 變更:', newId)
  // 重新載入資料
  fetchProduct(newId)
})

// 監聽查詢字串
watch(() => route.query, (newQuery) => {
  console.log('查詢參數變更:', newQuery)
}, { deep: true })
</script>
```

### 實際範例：產品列表篩選

```vue
<!-- pages/products/index.vue -->
<template>
  <div class="products-page">
    <h1>產品列表</h1>

    <!-- 篩選器 -->
    <div class="filters">
      <select v-model="selectedCategory" @change="updateFilters">
        <option value="">所有分類</option>
        <option value="electronics">電子產品</option>
        <option value="clothing">服飾</option>
        <option value="books">書籍</option>
      </select>

      <select v-model="sortBy" @change="updateFilters">
        <option value="name">名稱</option>
        <option value="price">價格</option>
        <option value="date">日期</option>
      </select>

      <input
        v-model="searchQuery"
        @input="debouncedUpdate"
        type="search"
        placeholder="搜尋產品..."
      />
    </div>

    <!-- 產品列表 -->
    <div v-if="pending" class="loading">載入中...</div>

    <div v-else-if="products" class="products-grid">
      <div
        v-for="product in products"
        :key="product.id"
        class="product-card"
      >
        <h3>{{ product.name }}</h3>
        <p class="category">{{ product.category }}</p>
        <p class="price">${{ product.price }}</p>
        <NuxtLink :to="`/products/${product.id}`">
          查看詳情
        </NuxtLink>
      </div>
    </div>

    <div v-else class="no-results">
      找不到符合條件的產品
    </div>
  </div>
</template>

<script setup lang="ts">
interface Product {
  id: number
  name: string
  price: number
  category: string
}

const route = useRoute()
const router = useRouter()

// 從 URL 查詢參數初始化篩選條件
const selectedCategory = ref(route.query.category as string || '')
const sortBy = ref(route.query.sort as string || 'name')
const searchQuery = ref(route.query.search as string || '')

// 取得產品資料
const { data: products, pending } = await useFetch<Product[]>('/api/products', {
  query: {
    category: selectedCategory,
    sort: sortBy,
    search: searchQuery
  },
  // 監聽查詢參數變化自動重新取得資料
  watch: [selectedCategory, sortBy, searchQuery]
})

// 更新 URL 查詢參數
const updateFilters = () => {
  router.push({
    query: {
      category: selectedCategory.value || undefined,
      sort: sortBy.value,
      search: searchQuery.value || undefined
    }
  })
}

// 防抖搜尋
let debounceTimer: NodeJS.Timeout
const debouncedUpdate = () => {
  clearTimeout(debounceTimer)
  debounceTimer = setTimeout(() => {
    updateFilters()
  }, 300)
}

// 監聽 URL 變化更新篩選條件
watch(() => route.query, (newQuery) => {
  selectedCategory.value = newQuery.category as string || ''
  sortBy.value = newQuery.sort as string || 'name'
  searchQuery.value = newQuery.search as string || ''
})
</script>

<style scoped>
.products-page {
  max-width: 1200px;
  margin: 0 auto;
  padding: 2rem;
}

.filters {
  display: flex;
  gap: 1rem;
  margin-bottom: 2rem;
}

.filters select,
.filters input {
  padding: 0.5rem;
  border: 1px solid #ddd;
  border-radius: 4px;
  font-size: 1rem;
}

.filters input {
  flex: 1;
}

.loading,
.no-results {
  text-align: center;
  padding: 3rem;
  color: #666;
}

.products-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(250px, 1fr));
  gap: 1.5rem;
}

.product-card {
  padding: 1.5rem;
  border: 1px solid #e0e0e0;
  border-radius: 8px;
}

.category {
  color: #666;
  font-size: 0.875rem;
  margin: 0.5rem 0;
}

.price {
  font-size: 1.25rem;
  font-weight: bold;
  color: #00DC82;
  margin: 0.5rem 0;
}
</style>
```

## 巢狀路由

使用 `<NuxtPage>` 組件來顯示子路由。

### 目錄結構

```
pages/
├── products/
│   ├── index.vue           → /products
│   ├── [id].vue           → /products/:id
│   └── [id]/
│       ├── index.vue      → /products/:id (預設視圖)
│       ├── reviews.vue    → /products/:id/reviews
│       └── specs.vue      → /products/:id/specs
```

### 父路由組件

```vue
<!-- pages/products/[id].vue -->
<template>
  <div class="product-detail">
    <div class="product-header">
      <h1>{{ product?.name }}</h1>
      <p class="price">${{ product?.price }}</p>
    </div>

    <!-- 子導航 -->
    <nav class="sub-nav">
      <NuxtLink :to="`/products/${id}`" exact>
        概覽
      </NuxtLink>
      <NuxtLink :to="`/products/${id}/reviews`">
        評價
      </NuxtLink>
      <NuxtLink :to="`/products/${id}/specs`">
        規格
      </NuxtLink>
    </nav>

    <!-- 子路由視圖 -->
    <div class="content">
      <NuxtPage :product="product" />
    </div>
  </div>
</template>

<script setup lang="ts">
const route = useRoute()
const id = computed(() => route.params.id)

interface Product {
  id: number
  name: string
  price: number
  description: string
}

const { data: product } = await useFetch<Product>(`/api/products/${id.value}`)
</script>

<style scoped>
.product-detail {
  max-width: 1000px;
  margin: 0 auto;
  padding: 2rem;
}

.product-header {
  margin-bottom: 2rem;
}

.price {
  font-size: 2rem;
  color: #00DC82;
  font-weight: bold;
}

.sub-nav {
  display: flex;
  gap: 1rem;
  border-bottom: 2px solid #e0e0e0;
  margin-bottom: 2rem;
}

.sub-nav a {
  padding: 1rem;
  text-decoration: none;
  color: #666;
  border-bottom: 2px solid transparent;
  margin-bottom: -2px;
  transition: all 0.3s;
}

.sub-nav a:hover {
  color: #00DC82;
}

.sub-nav a.router-link-active {
  color: #00DC82;
  border-bottom-color: #00DC82;
}
</style>
```

### 子路由組件

```vue
<!-- pages/products/[id]/index.vue -->
<template>
  <div class="overview">
    <h2>產品概覽</h2>
    <p>{{ product?.description }}</p>
  </div>
</template>

<script setup lang="ts">
defineProps<{
  product?: any
}>()
</script>
```

```vue
<!-- pages/products/[id]/reviews.vue -->
<template>
  <div class="reviews">
    <h2>顧客評價</h2>

    <div v-if="pending" class="loading">載入中...</div>

    <div v-else-if="reviews" class="reviews-list">
      <div
        v-for="review in reviews"
        :key="review.id"
        class="review-card"
      >
        <div class="rating">
          ⭐️ {{ review.rating }}/5
        </div>
        <p class="author">{{ review.author }}</p>
        <p class="comment">{{ review.comment }}</p>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
const route = useRoute()

interface Review {
  id: number
  rating: number
  author: string
  comment: string
}

const { data: reviews, pending } = await useFetch<Review[]>(
  `/api/products/${route.params.id}/reviews`
)
</script>

<style scoped>
.reviews-list {
  display: flex;
  flex-direction: column;
  gap: 1rem;
}

.review-card {
  padding: 1rem;
  border: 1px solid #e0e0e0;
  border-radius: 8px;
}

.rating {
  font-size: 1.25rem;
  margin-bottom: 0.5rem;
}

.author {
  font-weight: 600;
  margin-bottom: 0.5rem;
}

.comment {
  color: #666;
}
</style>
```

## 完整範例：電商網站路由

### 目錄結構

```
pages/
├── index.vue                      → /
├── products/
│   ├── index.vue                 → /products
│   ├── [id].vue                  → /products/:id
│   └── [id]/
│       ├── reviews.vue           → /products/:id/reviews
│       └── specs.vue             → /products/:id/specs
├── cart.vue                       → /cart
├── checkout/
│   ├── index.vue                 → /checkout
│   ├── shipping.vue              → /checkout/shipping
│   └── payment.vue               → /checkout/payment
└── user/
    ├── profile.vue               → /user/profile
    ├── orders.vue                → /user/orders
    └── orders/
        └── [orderId].vue         → /user/orders/:orderId
```

### 主要布局

```vue
<!-- app.vue -->
<template>
  <div id="app">
    <TheHeader />

    <main class="main-content">
      <NuxtPage />
    </main>

    <TheFooter />
  </div>
</template>

<style>
.main-content {
  min-height: calc(100vh - 200px);
  padding: 2rem 0;
}
</style>
```

### 導航組件

```vue
<!-- components/TheHeader.vue -->
<template>
  <header class="site-header">
    <div class="container">
      <NuxtLink to="/" class="logo">
        🛍️ 我的商店
      </NuxtLink>

      <nav class="main-nav">
        <NuxtLink to="/">首頁</NuxtLink>
        <NuxtLink to="/products">產品</NuxtLink>
        <NuxtLink to="/cart" class="cart-link">
          🛒 購物車 ({{ cartCount }})
        </NuxtLink>
        <NuxtLink to="/user/profile">會員中心</NuxtLink>
      </nav>
    </div>
  </header>
</template>

<script setup lang="ts">
// 假設有購物車狀態
const cartCount = ref(3)
</script>

<style scoped>
.site-header {
  background: white;
  border-bottom: 1px solid #e0e0e0;
  padding: 1rem 0;
}

.container {
  max-width: 1200px;
  margin: 0 auto;
  padding: 0 2rem;
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.logo {
  font-size: 1.5rem;
  font-weight: bold;
  text-decoration: none;
  color: #00DC82;
}

.main-nav {
  display: flex;
  gap: 2rem;
}

.main-nav a {
  text-decoration: none;
  color: #333;
  transition: color 0.3s;
}

.main-nav a:hover,
.main-nav a.router-link-active {
  color: #00DC82;
}

.cart-link {
  position: relative;
}
</style>
```

## 最佳實踐建議

### 1. 使用語意化的檔案命名

```
✅ 推薦
pages/products/[id].vue
pages/user/orders/[orderId].vue

❌ 不推薦
pages/p/[i].vue
pages/u/o/[oid].vue
```

### 2. 適當使用 definePageMeta

```vue
<script setup lang="ts">
definePageMeta({
  // 設定 Layout
  layout: 'admin',

  // 路由中介軟體
  middleware: 'auth',

  // 自定義屬性
  requiresAuth: true,
  roles: ['admin', 'editor']
})
</script>
```

### 3. SEO 優化

```vue
<script setup lang="ts">
// 為每個頁面設定適當的 meta 標籤
useHead({
  title: '產品列表',
  meta: [
    { name: 'description', content: '瀏覽我們的產品' },
    { property: 'og:title', content: '產品列表 - 我的商店' },
    { property: 'og:description', content: '瀏覽我們的產品' }
  ]
})
</script>
```

### 4. 路由驗證

```vue
<script setup lang="ts">
const route = useRoute()

// 驗證路由參數
const id = computed(() => {
  const paramId = route.params.id
  return typeof paramId === 'string' ? parseInt(paramId) : 0
})

if (isNaN(id.value) || id.value <= 0) {
  throw createError({
    statusCode: 400,
    message: '無效的產品 ID'
  })
}
</script>
```

## 總結

本章節涵蓋了 Nuxt 3 路由系統的基礎知識：

- ✅ 檔案式路由的概念與優勢
- ✅ pages/ 目錄結構與命名規則
- ✅ NuxtLink 組件的使用方法
- ✅ 程式化導航技巧
- ✅ 路由參數與查詢字串處理
- ✅ 巢狀路由實作
- ✅ 完整的電商網站範例

在下一章節中，我們將深入探討動態路由與路由參數的進階用法。
