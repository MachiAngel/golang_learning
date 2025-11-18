# 動態路由與路由參數

## [id].vue 動態路由語法

動態路由使用方括號 `[]` 來定義路由參數，這讓我們可以使用單一元件處理多個相似的頁面。

### 基本概念

```
pages/
├── products/
│   ├── [id].vue          → /products/1, /products/2, /products/abc
│   └── index.vue         → /products
├── users/
│   └── [username].vue    → /users/john, /users/jane
└── blog/
    └── [slug].vue        → /blog/my-first-post
```

### 基本動態路由範例

```vue
<!-- pages/products/[id].vue -->
<template>
  <div class="product-page">
    <div v-if="pending" class="loading">
      <p>載入中...</p>
    </div>

    <div v-else-if="error" class="error">
      <h2>錯誤</h2>
      <p>{{ error.message }}</p>
      <NuxtLink to="/products">返回產品列表</NuxtLink>
    </div>

    <div v-else-if="product" class="product-detail">
      <div class="product-image">
        <img :src="product.image" :alt="product.name" />
      </div>

      <div class="product-info">
        <h1>{{ product.name }}</h1>

        <div class="meta">
          <span class="category">{{ product.category }}</span>
          <span class="sku">SKU: {{ product.sku }}</span>
        </div>

        <p class="description">{{ product.description }}</p>

        <div class="price-section">
          <p class="price">${{ product.price.toLocaleString() }}</p>
          <button @click="addToCart" class="add-to-cart">
            加入購物車
          </button>
        </div>

        <div class="stock-info">
          <p :class="stockClass">
            {{ stockText }}
          </p>
        </div>
      </div>
    </div>

    <div v-else class="not-found">
      <h2>找不到產品</h2>
      <NuxtLink to="/products">返回產品列表</NuxtLink>
    </div>
  </div>
</template>

<script setup lang="ts">
interface Product {
  id: number
  name: string
  description: string
  price: number
  category: string
  sku: string
  image: string
  stock: number
}

// 取得路由參數
const route = useRoute()
const id = computed(() => route.params.id)

// 取得產品資料
const { data: product, pending, error } = await useFetch<Product>(
  `/api/products/${id.value}`,
  {
    // 當路由參數變化時重新取得資料
    key: `product-${id.value}`,
    // 處理錯誤
    onResponseError({ response }) {
      if (response.status === 404) {
        throw createError({
          statusCode: 404,
          message: '找不到此產品'
        })
      }
    }
  }
)

// 計算屬性
const stockClass = computed(() => ({
  'in-stock': product.value && product.value.stock > 10,
  'low-stock': product.value && product.value.stock > 0 && product.value.stock <= 10,
  'out-of-stock': product.value && product.value.stock === 0
}))

const stockText = computed(() => {
  if (!product.value) return ''
  if (product.value.stock === 0) return '缺貨'
  if (product.value.stock <= 10) return `僅剩 ${product.value.stock} 件`
  return '有貨'
})

// 方法
const addToCart = () => {
  console.log('加入購物車:', product.value)
  // 實際應用中會更新購物車狀態
  alert(`已將 ${product.value?.name} 加入購物車`)
}

// SEO
useHead({
  title: () => product.value?.name || '產品詳情',
  meta: [
    {
      name: 'description',
      content: () => product.value?.description || ''
    },
    {
      property: 'og:title',
      content: () => product.value?.name || '產品詳情'
    },
    {
      property: 'og:image',
      content: () => product.value?.image || ''
    }
  ]
})
</script>

<style scoped>
.product-page {
  max-width: 1200px;
  margin: 0 auto;
  padding: 2rem;
}

.loading,
.error,
.not-found {
  text-align: center;
  padding: 4rem 2rem;
}

.error {
  color: #dc2626;
}

.product-detail {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 3rem;
}

.product-image img {
  width: 100%;
  border-radius: 8px;
}

.product-info h1 {
  color: #111;
  margin-bottom: 1rem;
}

.meta {
  display: flex;
  gap: 1rem;
  margin-bottom: 1rem;
}

.category,
.sku {
  padding: 0.25rem 0.75rem;
  background: #f5f5f5;
  border-radius: 4px;
  font-size: 0.875rem;
  color: #666;
}

.description {
  line-height: 1.8;
  color: #666;
  margin-bottom: 2rem;
}

.price-section {
  margin-bottom: 2rem;
}

.price {
  font-size: 2.5rem;
  font-weight: bold;
  color: #00DC82;
  margin-bottom: 1rem;
}

.add-to-cart {
  width: 100%;
  padding: 1rem 2rem;
  background: #00DC82;
  color: white;
  border: none;
  border-radius: 8px;
  font-size: 1.125rem;
  cursor: pointer;
  transition: background 0.3s;
}

.add-to-cart:hover {
  background: #00b36b;
}

.stock-info {
  margin-top: 1rem;
}

.in-stock {
  color: #059669;
  font-weight: 600;
}

.low-stock {
  color: #f59e0b;
  font-weight: 600;
}

.out-of-stock {
  color: #dc2626;
  font-weight: 600;
}

@media (max-width: 768px) {
  .product-detail {
    grid-template-columns: 1fr;
  }
}
</style>
```

### 多個動態參數

```
pages/
└── blog/
    └── [category]/
        └── [slug].vue    → /blog/tech/nuxt-3-tutorial
```

```vue
<!-- pages/blog/[category]/[slug].vue -->
<template>
  <article class="post">
    <div class="breadcrumb">
      <NuxtLink to="/blog">部落格</NuxtLink>
      <span>/</span>
      <NuxtLink :to="`/blog/${category}`">{{ category }}</NuxtLink>
      <span>/</span>
      <span>{{ post?.title }}</span>
    </div>

    <header class="post-header">
      <h1>{{ post?.title }}</h1>
      <div class="meta">
        <time>{{ formatDate(post?.publishedAt) }}</time>
        <span class="category">{{ category }}</span>
      </div>
    </header>

    <div class="post-content" v-html="post?.content"></div>
  </article>
</template>

<script setup lang="ts">
interface Post {
  title: string
  slug: string
  category: string
  content: string
  publishedAt: string
}

const route = useRoute()
const category = computed(() => route.params.category as string)
const slug = computed(() => route.params.slug as string)

const { data: post } = await useFetch<Post>(
  `/api/blog/${category.value}/${slug.value}`
)

const formatDate = (date?: string) => {
  if (!date) return ''
  return new Date(date).toLocaleDateString('zh-TW', {
    year: 'numeric',
    month: 'long',
    day: 'numeric'
  })
}

useHead({
  title: () => post.value?.title || '文章',
  meta: [
    { name: 'description', content: () => post.value?.content?.substring(0, 160) }
  ]
})
</script>

<style scoped>
.post {
  max-width: 800px;
  margin: 0 auto;
  padding: 2rem;
}

.breadcrumb {
  display: flex;
  gap: 0.5rem;
  margin-bottom: 2rem;
  color: #666;
  font-size: 0.875rem;
}

.breadcrumb a {
  color: #00DC82;
  text-decoration: none;
}

.post-header {
  margin-bottom: 3rem;
}

.post-header h1 {
  font-size: 2.5rem;
  margin-bottom: 1rem;
}

.meta {
  display: flex;
  gap: 1rem;
  color: #666;
}

.post-content {
  line-height: 1.8;
  font-size: 1.125rem;
}
</style>
```

## [...slug].vue 萬用路由

萬用路由（Catch-all Route）使用 `...` 前綴，可以匹配任意深度的路徑。

### 基本概念

```
pages/
├── [...slug].vue         → 匹配所有未定義的路由
└── docs/
    └── [...slug].vue     → /docs/guide/intro, /docs/api/components/button
```

### 萬用路由範例：文件系統

```vue
<!-- pages/docs/[...slug].vue -->
<template>
  <div class="docs-page">
    <aside class="sidebar">
      <nav>
        <h3>文件導航</h3>
        <ul>
          <li>
            <NuxtLink to="/docs/getting-started">快速開始</NuxtLink>
          </li>
          <li>
            <NuxtLink to="/docs/guide/installation">安裝指南</NuxtLink>
          </li>
          <li>
            <NuxtLink to="/docs/guide/configuration">配置</NuxtLink>
          </li>
          <li>
            <NuxtLink to="/docs/api/components">組件 API</NuxtLink>
          </li>
        </ul>
      </nav>
    </aside>

    <main class="content">
      <div v-if="pending">載入中...</div>

      <article v-else-if="doc" class="doc-content">
        <h1>{{ doc.title }}</h1>
        <div v-html="doc.content"></div>

        <div class="doc-footer">
          <NuxtLink v-if="doc.prev" :to="`/docs/${doc.prev}`" class="nav-link prev">
            ← {{ doc.prevTitle }}
          </NuxtLink>
          <NuxtLink v-if="doc.next" :to="`/docs/${doc.next}`" class="nav-link next">
            {{ doc.nextTitle }} →
          </NuxtLink>
        </div>
      </article>

      <div v-else class="not-found">
        <h2>找不到文件</h2>
        <NuxtLink to="/docs">返回文件首頁</NuxtLink>
      </div>
    </main>
  </div>
</template>

<script setup lang="ts">
interface Doc {
  title: string
  content: string
  prev?: string
  prevTitle?: string
  next?: string
  nextTitle?: string
}

const route = useRoute()

// slug 是一個陣列，包含所有路徑段
const slug = computed(() => {
  const params = route.params.slug
  // 如果 slug 是陣列，用 / 連接
  return Array.isArray(params) ? params.join('/') : params
})

// 取得文件內容
const { data: doc, pending } = await useFetch<Doc>(
  `/api/docs/${slug.value}`
)

// 麵包屑導航
const breadcrumbs = computed(() => {
  const parts = slug.value.split('/')
  return parts.map((part, index) => ({
    name: part,
    path: '/docs/' + parts.slice(0, index + 1).join('/')
  }))
})

useHead({
  title: () => doc.value?.title || '文件',
  meta: [
    { name: 'description', content: '查看我們的文件' }
  ]
})
</script>

<style scoped>
.docs-page {
  display: grid;
  grid-template-columns: 250px 1fr;
  max-width: 1400px;
  margin: 0 auto;
  min-height: calc(100vh - 100px);
}

.sidebar {
  background: #f9fafb;
  padding: 2rem 1rem;
  border-right: 1px solid #e5e7eb;
}

.sidebar h3 {
  margin-bottom: 1rem;
  color: #111;
}

.sidebar ul {
  list-style: none;
  padding: 0;
}

.sidebar li {
  margin-bottom: 0.5rem;
}

.sidebar a {
  color: #666;
  text-decoration: none;
  display: block;
  padding: 0.5rem;
  border-radius: 4px;
  transition: all 0.3s;
}

.sidebar a:hover,
.sidebar a.router-link-active {
  background: #00DC82;
  color: white;
}

.content {
  padding: 3rem;
}

.doc-content h1 {
  margin-bottom: 2rem;
  color: #111;
}

.doc-footer {
  display: flex;
  justify-content: space-between;
  margin-top: 4rem;
  padding-top: 2rem;
  border-top: 1px solid #e5e7eb;
}

.nav-link {
  padding: 1rem;
  text-decoration: none;
  color: #00DC82;
  border: 1px solid #00DC82;
  border-radius: 4px;
  transition: all 0.3s;
}

.nav-link:hover {
  background: #00DC82;
  color: white;
}

.nav-link.next {
  margin-left: auto;
}
</style>
```

### 萬用路由參數類型

```vue
<script setup lang="ts">
const route = useRoute()

// slug 可能是字串或字串陣列
const slug = route.params.slug

if (typeof slug === 'string') {
  // /docs/guide → 'guide'
  console.log('單一段:', slug)
} else if (Array.isArray(slug)) {
  // /docs/guide/intro → ['guide', 'intro']
  console.log('多個段:', slug)
  console.log('完整路徑:', slug.join('/'))
}
</script>
```

## 取得路由參數（useRoute）

`useRoute()` 是 Nuxt 提供的 Composable，用於訪問當前路由資訊。

### 基本用法

```vue
<script setup lang="ts">
const route = useRoute()

// 路由參數
console.log(route.params.id)       // 動態路由參數
console.log(route.params.slug)     // 萬用路由參數

// 查詢字串
console.log(route.query.search)    // ?search=keyword
console.log(route.query.page)      // ?page=2

// 其他屬性
console.log(route.path)            // 當前路徑 /products/123
console.log(route.name)            // 路由名稱 products-id
console.log(route.fullPath)        // 完整路徑包含查詢字串
console.log(route.hash)            // Hash 部分 #section1
console.log(route.matched)         // 匹配的路由記錄
</script>
```

### 響應式路由參數

```vue
<script setup lang="ts">
const route = useRoute()

// ✅ 使用 computed 讓參數保持響應式
const id = computed(() => route.params.id)

// ❌ 這樣會失去響應式
// const id = route.params.id

// 監聽參數變化
watch(id, (newId, oldId) => {
  console.log(`ID 從 ${oldId} 變更為 ${newId}`)
  // 重新取得資料
  fetchData(newId)
})

// 或者監聽整個 params 物件
watch(
  () => route.params,
  (newParams) => {
    console.log('路由參數變更:', newParams)
  },
  { deep: true }
)
</script>
```

### 實際範例：搜尋頁面

```vue
<!-- pages/search.vue -->
<template>
  <div class="search-page">
    <div class="search-bar">
      <input
        v-model="searchInput"
        @keyup.enter="performSearch"
        type="search"
        placeholder="搜尋..."
      />
      <button @click="performSearch">搜尋</button>
    </div>

    <div class="filters">
      <select v-model="selectedType" @change="updateFilters">
        <option value="">所有類型</option>
        <option value="product">產品</option>
        <option value="article">文章</option>
        <option value="user">用戶</option>
      </select>

      <select v-model="sortOrder" @change="updateFilters">
        <option value="relevance">相關性</option>
        <option value="date">日期</option>
        <option value="popularity">熱門度</option>
      </select>
    </div>

    <div v-if="pending" class="loading">
      搜尋中...
    </div>

    <div v-else-if="results && results.length > 0" class="results">
      <p class="result-count">
        找到 {{ totalCount }} 個結果（顯示第 {{ (currentPage - 1) * pageSize + 1 }} - {{ Math.min(currentPage * pageSize, totalCount) }} 個）
      </p>

      <div
        v-for="result in results"
        :key="result.id"
        class="result-item"
      >
        <h3>
          <NuxtLink :to="result.url">
            {{ highlightText(result.title, query) }}
          </NuxtLink>
        </h3>
        <p class="snippet">{{ result.snippet }}</p>
        <span class="type">{{ result.type }}</span>
      </div>

      <div class="pagination">
        <button
          @click="goToPage(currentPage - 1)"
          :disabled="currentPage === 1"
        >
          上一頁
        </button>

        <span>第 {{ currentPage }} / {{ totalPages }} 頁</span>

        <button
          @click="goToPage(currentPage + 1)"
          :disabled="currentPage === totalPages"
        >
          下一頁
        </button>
      </div>
    </div>

    <div v-else-if="query" class="no-results">
      <p>找不到與 "{{ query }}" 相關的結果</p>
      <p>請嘗試其他關鍵字</p>
    </div>

    <div v-else class="empty-state">
      <p>輸入關鍵字開始搜尋</p>
    </div>
  </div>
</template>

<script setup lang="ts">
interface SearchResult {
  id: number
  title: string
  snippet: string
  url: string
  type: string
}

interface SearchResponse {
  results: SearchResult[]
  total: number
  page: number
  pageSize: number
}

const route = useRoute()
const router = useRouter()

// 從 URL 初始化搜尋條件
const searchInput = ref((route.query.q as string) || '')
const query = computed(() => route.query.q as string || '')
const selectedType = ref((route.query.type as string) || '')
const sortOrder = ref((route.query.sort as string) || 'relevance')
const currentPage = computed(() => parseInt(route.query.page as string) || 1)
const pageSize = 10

// 執行搜尋
const { data: searchData, pending } = await useFetch<SearchResponse>(
  '/api/search',
  {
    query: {
      q: query,
      type: selectedType,
      sort: sortOrder,
      page: currentPage,
      pageSize
    },
    watch: [query, selectedType, sortOrder, currentPage]
  }
)

const results = computed(() => searchData.value?.results || [])
const totalCount = computed(() => searchData.value?.total || 0)
const totalPages = computed(() => Math.ceil(totalCount.value / pageSize))

// 執行搜尋
const performSearch = () => {
  if (searchInput.value.trim()) {
    router.push({
      path: '/search',
      query: {
        q: searchInput.value,
        type: selectedType.value || undefined,
        sort: sortOrder.value
      }
    })
  }
}

// 更新篩選條件
const updateFilters = () => {
  router.push({
    query: {
      ...route.query,
      type: selectedType.value || undefined,
      sort: sortOrder.value,
      page: undefined // 重置頁碼
    }
  })
}

// 切換頁面
const goToPage = (page: number) => {
  router.push({
    query: {
      ...route.query,
      page: page > 1 ? page : undefined
    }
  })
}

// 高亮搜尋關鍵字
const highlightText = (text: string, keyword: string) => {
  if (!keyword) return text
  const regex = new RegExp(`(${keyword})`, 'gi')
  return text.replace(regex, '<mark>$1</mark>')
}

// 監聽 URL 變化更新輸入框
watch(() => route.query.q, (newQuery) => {
  searchInput.value = newQuery as string || ''
})

useHead({
  title: () => query.value ? `搜尋: ${query.value}` : '搜尋',
  meta: [
    { name: 'robots', content: 'noindex' } // 搜尋頁面通常不需要索引
  ]
})
</script>

<style scoped>
.search-page {
  max-width: 900px;
  margin: 0 auto;
  padding: 2rem;
}

.search-bar {
  display: flex;
  gap: 0.5rem;
  margin-bottom: 2rem;
}

.search-bar input {
  flex: 1;
  padding: 0.75rem 1rem;
  font-size: 1rem;
  border: 2px solid #e0e0e0;
  border-radius: 8px;
}

.search-bar button {
  padding: 0.75rem 2rem;
  background: #00DC82;
  color: white;
  border: none;
  border-radius: 8px;
  cursor: pointer;
}

.filters {
  display: flex;
  gap: 1rem;
  margin-bottom: 2rem;
}

.filters select {
  padding: 0.5rem;
  border: 1px solid #e0e0e0;
  border-radius: 4px;
}

.result-count {
  color: #666;
  margin-bottom: 1.5rem;
}

.result-item {
  padding: 1.5rem;
  border: 1px solid #e0e0e0;
  border-radius: 8px;
  margin-bottom: 1rem;
}

.result-item h3 {
  margin-bottom: 0.5rem;
}

.result-item h3 a {
  color: #00DC82;
  text-decoration: none;
}

.snippet {
  color: #666;
  margin-bottom: 0.5rem;
}

.type {
  display: inline-block;
  padding: 0.25rem 0.5rem;
  background: #f0f0f0;
  border-radius: 4px;
  font-size: 0.75rem;
}

.pagination {
  display: flex;
  justify-content: center;
  align-items: center;
  gap: 1rem;
  margin-top: 2rem;
}

.pagination button {
  padding: 0.5rem 1rem;
  border: 1px solid #00DC82;
  background: white;
  color: #00DC82;
  border-radius: 4px;
  cursor: pointer;
}

.pagination button:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.no-results,
.empty-state {
  text-align: center;
  padding: 4rem 2rem;
  color: #666;
}
</style>
```

## 路由驗證

使用 `definePageMeta` 中的 `validate` 函數驗證路由參數。

### 基本驗證

```vue
<!-- pages/products/[id].vue -->
<script setup lang="ts">
definePageMeta({
  validate: async (route) => {
    // 驗證 ID 是否為數字
    const id = route.params.id
    return typeof id === 'string' && /^\d+$/.test(id)
  }
})

// 如果驗證失敗，會自動顯示 404 頁面
</script>
```

### 進階驗證範例

```vue
<!-- pages/blog/[category]/[slug].vue -->
<script setup lang="ts">
const validCategories = ['tech', 'lifestyle', 'business', 'travel']

definePageMeta({
  validate: async (route) => {
    const { category, slug } = route.params

    // 驗證分類
    if (typeof category !== 'string' || !validCategories.includes(category)) {
      return false
    }

    // 驗證 slug 格式（只允許字母、數字和連字號）
    if (typeof slug !== 'string' || !/^[a-z0-9-]+$/.test(slug)) {
      return false
    }

    // 可以進行異步驗證，例如檢查文章是否存在
    try {
      const response = await $fetch(`/api/blog/${category}/${slug}/check`)
      return response.exists
    } catch {
      return false
    }
  }
})
</script>
```

### 自定義錯誤訊息

```vue
<script setup lang="ts">
definePageMeta({
  validate: async (route) => {
    const id = route.params.id

    if (typeof id !== 'string' || !/^\d+$/.test(id)) {
      throw createError({
        statusCode: 400,
        statusMessage: '無效的 ID 格式',
        message: 'ID 必須是數字'
      })
    }

    const numId = parseInt(id)
    if (numId <= 0 || numId > 10000) {
      throw createError({
        statusCode: 400,
        statusMessage: 'ID 超出範圍',
        message: 'ID 必須在 1 到 10000 之間'
      })
    }

    return true
  }
})
</script>
```

## 404 頁面處理

### 建立自定義 404 頁面

```vue
<!-- error.vue (專案根目錄) -->
<template>
  <div class="error-page">
    <div class="error-content">
      <h1>{{ error.statusCode }}</h1>

      <div v-if="error.statusCode === 404">
        <h2>找不到頁面</h2>
        <p>抱歉，您訪問的頁面不存在。</p>
      </div>

      <div v-else-if="error.statusCode === 500">
        <h2>伺服器錯誤</h2>
        <p>抱歉，伺服器發生錯誤。</p>
      </div>

      <div v-else>
        <h2>發生錯誤</h2>
        <p>{{ error.message }}</p>
      </div>

      <div class="actions">
        <NuxtLink to="/" class="btn-primary">
          返回首頁
        </NuxtLink>

        <button @click="handleError" class="btn-secondary">
          重試
        </button>
      </div>

      <details v-if="error.stack" class="error-details">
        <summary>錯誤詳情（開發模式）</summary>
        <pre>{{ error.stack }}</pre>
      </details>
    </div>
  </div>
</template>

<script setup lang="ts">
interface ErrorProps {
  error: {
    statusCode: number
    statusMessage: string
    message: string
    stack?: string
  }
}

const props = defineProps<ErrorProps>()

const handleError = () => clearError({ redirect: '/' })

useHead({
  title: `錯誤 ${props.error.statusCode}`
})
</script>

<style scoped>
.error-page {
  min-height: 100vh;
  display: flex;
  align-items: center;
  justify-content: center;
  background: #f9fafb;
}

.error-content {
  text-align: center;
  padding: 3rem;
  max-width: 600px;
}

.error-content h1 {
  font-size: 6rem;
  color: #00DC82;
  margin-bottom: 1rem;
}

.error-content h2 {
  font-size: 2rem;
  margin-bottom: 1rem;
}

.error-content p {
  color: #666;
  font-size: 1.125rem;
  margin-bottom: 2rem;
}

.actions {
  display: flex;
  gap: 1rem;
  justify-content: center;
  margin-top: 2rem;
}

.btn-primary,
.btn-secondary {
  padding: 0.75rem 2rem;
  border-radius: 8px;
  text-decoration: none;
  font-size: 1rem;
  cursor: pointer;
  transition: all 0.3s;
}

.btn-primary {
  background: #00DC82;
  color: white;
  border: none;
}

.btn-primary:hover {
  background: #00b36b;
}

.btn-secondary {
  background: white;
  color: #00DC82;
  border: 2px solid #00DC82;
}

.btn-secondary:hover {
  background: #f0fdf4;
}

.error-details {
  margin-top: 2rem;
  text-align: left;
  background: white;
  padding: 1rem;
  border-radius: 8px;
}

.error-details summary {
  cursor: pointer;
  font-weight: 600;
  margin-bottom: 0.5rem;
}

.error-details pre {
  overflow-x: auto;
  font-size: 0.875rem;
  color: #dc2626;
}
</style>
```

### 在頁面中觸發 404

```vue
<script setup lang="ts">
const route = useRoute()
const id = route.params.id

const { data: product } = await useFetch(`/api/products/${id}`)

if (!product.value) {
  throw createError({
    statusCode: 404,
    statusMessage: '找不到產品',
    message: `ID 為 ${id} 的產品不存在`
  })
}
</script>
```

## 實際應用範例（部落格文章頁）

完整的部落格文章系統範例：

### 目錄結構

```
pages/
└── blog/
    ├── index.vue              → /blog（文章列表）
    ├── [slug].vue             → /blog/my-post（文章詳情）
    └── category/
        └── [name].vue         → /blog/category/tech（分類頁）
```

### 文章列表頁

```vue
<!-- pages/blog/index.vue -->
<template>
  <div class="blog-page">
    <header class="page-header">
      <h1>部落格文章</h1>
      <p>分享技術與生活</p>
    </header>

    <div class="categories">
      <NuxtLink
        to="/blog"
        :class="{ active: !currentCategory }"
        class="category-tag"
      >
        全部
      </NuxtLink>
      <NuxtLink
        v-for="cat in categories"
        :key="cat.slug"
        :to="`/blog/category/${cat.slug}`"
        class="category-tag"
      >
        {{ cat.name }} ({{ cat.count }})
      </NuxtLink>
    </div>

    <div v-if="pending" class="loading">載入中...</div>

    <div v-else-if="posts && posts.length > 0" class="posts-grid">
      <article
        v-for="post in posts"
        :key="post.slug"
        class="post-card"
      >
        <NuxtLink :to="`/blog/${post.slug}`">
          <img :src="post.coverImage" :alt="post.title" />
        </NuxtLink>

        <div class="post-content">
          <div class="meta">
            <NuxtLink :to="`/blog/category/${post.category}`" class="category">
              {{ post.categoryName }}
            </NuxtLink>
            <time>{{ formatDate(post.publishedAt) }}</time>
          </div>

          <h2>
            <NuxtLink :to="`/blog/${post.slug}`">
              {{ post.title }}
            </NuxtLink>
          </h2>

          <p class="excerpt">{{ post.excerpt }}</p>

          <div class="post-footer">
            <span class="read-time">{{ post.readTime }} 分鐘閱讀</span>
            <NuxtLink :to="`/blog/${post.slug}`" class="read-more">
              閱讀更多 →
            </NuxtLink>
          </div>
        </div>
      </article>
    </div>
  </div>
</template>

<script setup lang="ts">
interface Post {
  slug: string
  title: string
  excerpt: string
  coverImage: string
  category: string
  categoryName: string
  publishedAt: string
  readTime: number
}

interface Category {
  slug: string
  name: string
  count: number
}

const route = useRoute()
const currentCategory = computed(() => route.query.category as string)

// 取得分類列表
const { data: categories } = await useFetch<Category[]>('/api/blog/categories')

// 取得文章列表
const { data: posts, pending } = await useFetch<Post[]>('/api/blog/posts', {
  query: { category: currentCategory },
  watch: [currentCategory]
})

const formatDate = (date: string) => {
  return new Date(date).toLocaleDateString('zh-TW', {
    year: 'numeric',
    month: 'long',
    day: 'numeric'
  })
}

useHead({
  title: '部落格',
  meta: [
    { name: 'description', content: '閱讀我們的部落格文章' }
  ]
})
</script>

<style scoped>
.blog-page {
  max-width: 1200px;
  margin: 0 auto;
  padding: 2rem;
}

.page-header {
  text-align: center;
  margin-bottom: 3rem;
}

.page-header h1 {
  font-size: 3rem;
  color: #111;
  margin-bottom: 0.5rem;
}

.page-header p {
  font-size: 1.25rem;
  color: #666;
}

.categories {
  display: flex;
  flex-wrap: wrap;
  gap: 0.75rem;
  margin-bottom: 3rem;
  justify-content: center;
}

.category-tag {
  padding: 0.5rem 1rem;
  background: #f5f5f5;
  color: #666;
  text-decoration: none;
  border-radius: 20px;
  transition: all 0.3s;
}

.category-tag:hover,
.category-tag.active {
  background: #00DC82;
  color: white;
}

.posts-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(350px, 1fr));
  gap: 2rem;
}

.post-card {
  background: white;
  border-radius: 12px;
  overflow: hidden;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
  transition: transform 0.3s, box-shadow 0.3s;
}

.post-card:hover {
  transform: translateY(-4px);
  box-shadow: 0 4px 16px rgba(0, 0, 0, 0.15);
}

.post-card img {
  width: 100%;
  height: 200px;
  object-fit: cover;
}

.post-content {
  padding: 1.5rem;
}

.meta {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 1rem;
  font-size: 0.875rem;
}

.category {
  background: #00DC82;
  color: white;
  padding: 0.25rem 0.75rem;
  border-radius: 12px;
  text-decoration: none;
}

time {
  color: #999;
}

.post-content h2 {
  margin-bottom: 0.75rem;
  font-size: 1.5rem;
}

.post-content h2 a {
  color: #111;
  text-decoration: none;
}

.post-content h2 a:hover {
  color: #00DC82;
}

.excerpt {
  color: #666;
  line-height: 1.6;
  margin-bottom: 1rem;
}

.post-footer {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.read-time {
  color: #999;
  font-size: 0.875rem;
}

.read-more {
  color: #00DC82;
  text-decoration: none;
  font-weight: 600;
}
</style>
```

### 文章詳情頁

```vue
<!-- pages/blog/[slug].vue -->
<template>
  <article class="post-page">
    <div v-if="pending" class="loading">載入中...</div>

    <div v-else-if="post" class="post-content">
      <header class="post-header">
        <div class="breadcrumb">
          <NuxtLink to="/blog">部落格</NuxtLink>
          <span>/</span>
          <NuxtLink :to="`/blog/category/${post.category}`">
            {{ post.categoryName }}
          </NuxtLink>
        </div>

        <h1>{{ post.title }}</h1>

        <div class="meta">
          <img :src="post.author.avatar" :alt="post.author.name" class="avatar" />
          <div class="author-info">
            <p class="author-name">{{ post.author.name }}</p>
            <div class="post-meta">
              <time>{{ formatDate(post.publishedAt) }}</time>
              <span>·</span>
              <span>{{ post.readTime }} 分鐘閱讀</span>
              <span>·</span>
              <span>{{ post.views }} 次觀看</span>
            </div>
          </div>
        </div>

        <img :src="post.coverImage" :alt="post.title" class="cover-image" />
      </header>

      <div class="article-content" v-html="post.content"></div>

      <footer class="post-footer">
        <div class="tags">
          <span
            v-for="tag in post.tags"
            :key="tag"
            class="tag"
          >
            #{{ tag }}
          </span>
        </div>

        <div class="share">
          <p>分享文章：</p>
          <button @click="shareToFacebook">Facebook</button>
          <button @click="shareToTwitter">Twitter</button>
          <button @click="copyLink">複製連結</button>
        </div>
      </footer>

      <div class="related-posts">
        <h3>相關文章</h3>
        <div class="posts-grid">
          <NuxtLink
            v-for="related in post.relatedPosts"
            :key="related.slug"
            :to="`/blog/${related.slug}`"
            class="related-card"
          >
            <img :src="related.coverImage" :alt="related.title" />
            <h4>{{ related.title }}</h4>
          </NuxtLink>
        </div>
      </div>
    </div>
  </article>
</template>

<script setup lang="ts">
interface Post {
  slug: string
  title: string
  content: string
  excerpt: string
  coverImage: string
  category: string
  categoryName: string
  publishedAt: string
  readTime: number
  views: number
  tags: string[]
  author: {
    name: string
    avatar: string
  }
  relatedPosts: Array<{
    slug: string
    title: string
    coverImage: string
  }>
}

const route = useRoute()
const slug = computed(() => route.params.slug as string)

// 驗證 slug 格式
definePageMeta({
  validate: async (route) => {
    const slug = route.params.slug
    return typeof slug === 'string' && /^[a-z0-9-]+$/.test(slug)
  }
})

// 取得文章資料
const { data: post, pending } = await useFetch<Post>(
  `/api/blog/posts/${slug.value}`,
  {
    key: `post-${slug.value}`,
    onResponseError({ response }) {
      if (response.status === 404) {
        throw createError({
          statusCode: 404,
          message: '找不到此文章'
        })
      }
    }
  }
)

const formatDate = (date: string) => {
  return new Date(date).toLocaleDateString('zh-TW', {
    year: 'numeric',
    month: 'long',
    day: 'numeric'
  })
}

// 分享功能
const shareToFacebook = () => {
  const url = window.location.href
  window.open(`https://www.facebook.com/sharer/sharer.php?u=${url}`, '_blank')
}

const shareToTwitter = () => {
  const url = window.location.href
  const text = post.value?.title
  window.open(`https://twitter.com/intent/tweet?url=${url}&text=${text}`, '_blank')
}

const copyLink = async () => {
  try {
    await navigator.clipboard.writeText(window.location.href)
    alert('連結已複製')
  } catch (error) {
    console.error('複製失敗:', error)
  }
}

// SEO
useHead({
  title: () => post.value?.title || '文章',
  meta: [
    { name: 'description', content: () => post.value?.excerpt || '' },
    { property: 'og:title', content: () => post.value?.title || '' },
    { property: 'og:description', content: () => post.value?.excerpt || '' },
    { property: 'og:image', content: () => post.value?.coverImage || '' },
    { property: 'og:type', content: 'article' },
    { name: 'twitter:card', content: 'summary_large_image' }
  ],
  script: [
    {
      type: 'application/ld+json',
      children: () => JSON.stringify({
        '@context': 'https://schema.org',
        '@type': 'BlogPosting',
        headline: post.value?.title,
        image: post.value?.coverImage,
        datePublished: post.value?.publishedAt,
        author: {
          '@type': 'Person',
          name: post.value?.author.name
        }
      })
    }
  ]
})
</script>

<style scoped>
.post-page {
  max-width: 800px;
  margin: 0 auto;
  padding: 2rem;
}

.breadcrumb {
  display: flex;
  gap: 0.5rem;
  margin-bottom: 2rem;
  font-size: 0.875rem;
  color: #666;
}

.breadcrumb a {
  color: #00DC82;
  text-decoration: none;
}

.post-header h1 {
  font-size: 3rem;
  line-height: 1.2;
  margin-bottom: 1.5rem;
  color: #111;
}

.meta {
  display: flex;
  align-items: center;
  gap: 1rem;
  margin-bottom: 2rem;
}

.avatar {
  width: 50px;
  height: 50px;
  border-radius: 50%;
}

.author-name {
  font-weight: 600;
  margin-bottom: 0.25rem;
}

.post-meta {
  display: flex;
  gap: 0.5rem;
  font-size: 0.875rem;
  color: #666;
}

.cover-image {
  width: 100%;
  height: 400px;
  object-fit: cover;
  border-radius: 12px;
  margin-bottom: 3rem;
}

.article-content {
  font-size: 1.125rem;
  line-height: 1.8;
  color: #333;
  margin-bottom: 3rem;
}

.post-footer {
  border-top: 1px solid #e5e7eb;
  border-bottom: 1px solid #e5e7eb;
  padding: 2rem 0;
  margin-bottom: 3rem;
}

.tags {
  display: flex;
  flex-wrap: wrap;
  gap: 0.5rem;
  margin-bottom: 1.5rem;
}

.tag {
  padding: 0.5rem 1rem;
  background: #f5f5f5;
  color: #666;
  border-radius: 20px;
  font-size: 0.875rem;
}

.share p {
  margin-bottom: 0.5rem;
  font-weight: 600;
}

.share button {
  margin-right: 0.5rem;
  padding: 0.5rem 1rem;
  background: #00DC82;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
}

.related-posts h3 {
  margin-bottom: 1.5rem;
}

.posts-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(200px, 1fr));
  gap: 1.5rem;
}

.related-card {
  text-decoration: none;
  color: inherit;
}

.related-card img {
  width: 100%;
  height: 150px;
  object-fit: cover;
  border-radius: 8px;
  margin-bottom: 0.5rem;
}

.related-card h4 {
  font-size: 1rem;
  color: #111;
}

.related-card:hover h4 {
  color: #00DC82;
}
</style>
```

## 最佳實踐建議

### 1. 路由參數驗證

```vue
<script setup lang="ts">
// ✅ 始終驗證路由參數
definePageMeta({
  validate: (route) => {
    return /^\d+$/.test(route.params.id as string)
  }
})
</script>
```

### 2. 使用 computed 保持響應式

```vue
<script setup lang="ts">
const route = useRoute()

// ✅ 使用 computed
const id = computed(() => route.params.id)

// ❌ 直接賦值會失去響應式
// const id = route.params.id
</script>
```

### 3. SEO 優化

```vue
<script setup lang="ts">
// ✅ 為每個動態頁面設定適當的 meta 標籤
useHead({
  title: () => product.value?.name,
  meta: [
    { name: 'description', content: () => product.value?.description },
    { property: 'og:image', content: () => product.value?.image }
  ]
})
</script>
```

### 4. 錯誤處理

```vue
<script setup lang="ts">
const { data, error } = await useFetch('/api/data')

if (error.value) {
  throw createError({
    statusCode: error.value.statusCode || 500,
    message: error.value.message || '發生錯誤'
  })
}
</script>
```

## 總結

本章節深入探討了 Nuxt 3 的動態路由系統：

- ✅ `[id].vue` 動態路由語法與用法
- ✅ `[...slug].vue` 萬用路由實作
- ✅ useRoute() 取得路由參數
- ✅ 路由驗證與錯誤處理
- ✅ 自定義 404 頁面
- ✅ 完整的部落格應用範例

在下一章節中，我們將探討 Nuxt 3 的 Layouts 布局系統。
