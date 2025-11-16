# SSR (Server-Side Rendering) 完整指南

## 什麼是 SSR？

Server-Side Rendering（伺服器端渲染）是一種在伺服器上執行 JavaScript 並生成完整 HTML 的渲染技術。當用戶請求頁面時，伺服器會執行應用程式邏輯，將組件渲染成 HTML 字串，然後發送給客戶端。

### SSR 流程圖

```
客戶端請求
    ↓
伺服器接收請求
    ↓
執行 Vue 組件（Server）
    ↓
渲染成 HTML 字串
    ↓
發送 HTML + JS Bundle
    ↓
瀏覽器顯示 HTML（可見但不可互動）
    ↓
下載並執行 JavaScript
    ↓
Hydration（注入互動性）
    ↓
完全互動的應用程式
```

## SSR 的優勢與劣勢

### ✅ 優勢

1. **更好的 SEO**
   - 搜尋引擎爬蟲可以直接獲取完整的 HTML 內容
   - 不需要等待 JavaScript 執行
   - Meta 標籤和結構化資料立即可用

2. **更快的首次內容繪製 (FCP)**
   - 用戶更快看到頁面內容
   - 無需等待所有 JavaScript 下載完成
   - 改善感知效能

3. **更好的社群媒體分享**
   - Open Graph 和 Twitter Card 立即可用
   - 預覽圖片和描述正確顯示

4. **對低端設備友善**
   - 減少客戶端 JavaScript 執行負擔
   - 初始內容不依賴客戶端性能

### ❌ 劣勢

1. **伺服器負載增加**
   - 每次請求都需要渲染
   - 需要更強大的伺服器資源
   - 可能需要快取策略

2. **TTFB (Time To First Byte) 較慢**
   - 伺服器需要時間渲染
   - 比單純提供靜態檔案慢

3. **開發複雜度增加**
   - 需要考慮伺服器和客戶端環境差異
   - 某些瀏覽器 API 無法使用（如 `window`、`document`）

4. **成本較高**
   - 需要 Node.js 伺服器
   - 無法直接部署到靜態主機

## Nuxt 3 的 SSR 運作原理

### 1. 請求處理流程

```javascript
// Nuxt 3 SSR 內部流程（簡化版）

// 1. 伺服器接收請求
app.use((req, res) => {
  // 2. 建立 Vue 應用實例
  const { app, router } = createApp()

  // 3. 設定當前路由
  router.push(req.url)

  // 4. 等待路由和資料準備完成
  await router.isReady()

  // 5. 渲染應用成 HTML
  const html = await renderToString(app)

  // 6. 注入狀態和資源連結
  const fullHtml = `
    <!DOCTYPE html>
    <html>
      <head>
        <link rel="stylesheet" href="/assets/style.css">
      </head>
      <body>
        <div id="app">${html}</div>
        <script>
          window.__NUXT__ = ${JSON.stringify(state)}
        </script>
        <script src="/assets/app.js"></script>
      </body>
    </html>
  `

  // 7. 發送 HTML
  res.send(fullHtml)
})
```

### 2. Universal Rendering（通用渲染）

Nuxt 3 採用 Universal Rendering 模式，代碼可以在伺服器和客戶端執行：

```vue
<!-- pages/products/[id].vue -->
<template>
  <div>
    <h1>{{ product.name }}</h1>
    <p>{{ product.description }}</p>
    <p>價格: ${{ product.price }}</p>

    <!-- 這個按鈕在 SSR 時會渲染，在客戶端會有互動 -->
    <button @click="addToCart">加入購物車</button>

    <!-- 顯示當前環境 -->
    <p>渲染環境: {{ environment }}</p>
  </div>
</template>

<script setup>
const route = useRoute()

// ✅ useFetch 在伺服器和客戶端都會執行
const { data: product } = await useFetch(`/api/products/${route.params.id}`)

// 偵測執行環境
const environment = computed(() => {
  if (process.server) return '伺服器'
  if (process.client) return '客戶端'
  return '未知'
})

// ✅ 方法在客戶端執行（Hydration 後）
const addToCart = () => {
  console.log('加入購物車:', product.value.name)
  // 這個 console.log 只會在客戶端出現
}

// ⚠️ 生命週期鉤子在兩端的執行時機
onMounted(() => {
  // 只在客戶端執行
  console.log('Component mounted - 客戶端')
})

onServerPrefetch(async () => {
  // 只在伺服器執行
  console.log('Component prefetch - 伺服器端')
})
</script>
```

## Hydration 過程

Hydration（水合）是指客戶端 JavaScript 接管伺服器渲染的 HTML，使其變成可互動的過程。

### Hydration 流程

```
1. 伺服器渲染 HTML
   <button>點擊我</button>

2. 瀏覽器顯示（可見但不可點擊）

3. 下載 JavaScript Bundle

4. Vue 應用啟動

5. 比對虛擬 DOM 和實際 DOM

6. 附加事件監聽器
   <button @click="handler">點擊我</button>

7. 完成 - 完全可互動
```

### Hydration 範例

```vue
<!-- components/InteractiveCounter.vue -->
<template>
  <div class="counter">
    <h2>計數器</h2>
    <!-- SSR: 渲染初始值 -->
    <!-- Hydration: 附加響應性 -->
    <p>當前計數: {{ count }}</p>

    <!-- SSR: 渲染按鈕 HTML -->
    <!-- Hydration: 附加點擊事件 -->
    <button @click="increment">增加</button>
    <button @click="decrement">減少</button>

    <!-- 顯示 Hydration 狀態 -->
    <p v-if="!isHydrated" class="status">等待 Hydration...</p>
    <p v-else class="status">✅ 已 Hydration，可互動</p>
  </div>
</template>

<script setup>
const count = ref(0)
const isHydrated = ref(false)

const increment = () => {
  count.value++
}

const decrement = () => {
  count.value--
}

// 偵測 Hydration 完成
onMounted(() => {
  isHydrated.value = true
  console.log('Hydration 完成！')
})
</script>
```

### Hydration Mismatch（不匹配）

當伺服器渲染的 HTML 和客戶端期望的結構不一致時，會發生 Hydration Mismatch：

```vue
<!-- ❌ 錯誤：會導致 Hydration Mismatch -->
<template>
  <div>
    <!-- 伺服器和客戶端渲染結果不同 -->
    <p>當前時間: {{ new Date().toLocaleString() }}</p>
  </div>
</template>

<!-- ✅ 正確：確保伺服器和客戶端一致 -->
<template>
  <div>
    <!-- 使用 ClientOnly 包裝客戶端專屬內容 -->
    <ClientOnly>
      <p>當前時間: {{ currentTime }}</p>
      <template #fallback>
        <p>載入中...</p>
      </template>
    </ClientOnly>
  </div>
</template>

<script setup>
const currentTime = ref('')

onMounted(() => {
  currentTime.value = new Date().toLocaleString()
  // 每秒更新
  setInterval(() => {
    currentTime.value = new Date().toLocaleString()
  }, 1000)
})
</script>
```

## SSR 常見問題

### 1. `window is not defined` 錯誤

這是 SSR 最常見的問題，因為 Node.js 環境沒有 `window` 物件。

```vue
<template>
  <div>
    <p>視窗寬度: {{ windowWidth }}</p>
  </div>
</template>

<script setup>
// ❌ 錯誤：伺服器端沒有 window
// const windowWidth = ref(window.innerWidth)

// ✅ 方法 1：使用 process.client 檢查
const windowWidth = ref(0)

if (process.client) {
  windowWidth.value = window.innerWidth

  window.addEventListener('resize', () => {
    windowWidth.value = window.innerWidth
  })
}

// ✅ 方法 2：在 onMounted 中使用
onMounted(() => {
  windowWidth.value = window.innerWidth

  window.addEventListener('resize', () => {
    windowWidth.value = window.innerWidth
  })
})

// ✅ 方法 3：使用 VueUse 的 useWindowSize
// import { useWindowSize } from '@vueuse/core'
// const { width: windowWidth } = useWindowSize()
</script>
```

### 2. `document is not defined` 錯誤

```vue
<script setup>
// ❌ 錯誤：伺服器端沒有 document
// const bodyElement = document.body

// ✅ 正確：檢查環境
const setupDocumentListeners = () => {
  if (process.client) {
    document.addEventListener('click', (e) => {
      console.log('Clicked at:', e.clientX, e.clientY)
    })
  }
}

onMounted(() => {
  setupDocumentListeners()
})
</script>
```

### 3. localStorage 和 sessionStorage

```vue
<script setup>
// ❌ 錯誤：伺服器端沒有 localStorage
// const savedData = localStorage.getItem('data')

// ✅ 正確：使用 Composable
const useLocalStorage = (key, defaultValue) => {
  const data = ref(defaultValue)

  onMounted(() => {
    const saved = localStorage.getItem(key)
    if (saved) {
      data.value = JSON.parse(saved)
    }
  })

  watch(data, (newValue) => {
    if (process.client) {
      localStorage.setItem(key, JSON.stringify(newValue))
    }
  })

  return data
}

// 使用
const userPreferences = useLocalStorage('preferences', {
  theme: 'light',
  language: 'zh-TW'
})
</script>
```

### 4. 第三方套件不支援 SSR

```vue
<script setup>
// ❌ 某些套件只能在客戶端運行
// import SomeLibrary from 'client-only-library'

// ✅ 方法 1：動態導入
const loadLibrary = async () => {
  if (process.client) {
    const module = await import('client-only-library')
    return module.default
  }
}

onMounted(async () => {
  const SomeLibrary = await loadLibrary()
  // 使用套件...
})

// ✅ 方法 2：使用 ClientOnly 組件
</script>

<template>
  <ClientOnly>
    <ClientOnlyComponent />
  </ClientOnly>
</template>
```

## process.server 與 process.client

Nuxt 提供兩個環境變數來判斷執行環境：

```vue
<script setup>
console.log('process.server:', process.server) // true 在伺服器
console.log('process.client:', process.client) // true 在客戶端

// 範例：根據環境執行不同邏輯
const fetchData = async () => {
  if (process.server) {
    // 伺服器端：直接查詢資料庫
    console.log('伺服器端：直接查詢資料庫')
    // return await db.query(...)
  } else {
    // 客戶端：透過 API
    console.log('客戶端：透過 API 請求')
    // return await $fetch('/api/data')
  }
}

// 環境專屬的初始化
if (process.server) {
  console.log('伺服器端初始化')
  // 設定伺服器端快取
}

if (process.client) {
  console.log('客戶端初始化')
  // 設定分析工具、註冊 Service Worker 等
}
</script>
```

### 實用模式

```javascript
// composables/useEnvironment.js
export const useEnvironment = () => {
  const isServer = process.server
  const isClient = process.client

  // 安全地取得 window 物件
  const getWindow = () => {
    return isClient ? window : undefined
  }

  // 安全地執行客戶端代碼
  const runOnClient = (fn) => {
    if (isClient) {
      return fn()
    }
  }

  // 安全地執行伺服器端代碼
  const runOnServer = (fn) => {
    if (isServer) {
      return fn()
    }
  }

  return {
    isServer,
    isClient,
    getWindow,
    runOnClient,
    runOnServer
  }
}
```

## 完整 SSR 應用範例

### 電商產品頁面

```vue
<!-- pages/products/[id].vue -->
<template>
  <div class="product-page">
    <!-- SEO Meta Tags -->
    <Head>
      <Title>{{ product?.name }} - 我的商店</Title>
      <Meta name="description" :content="product?.description" />
      <Meta property="og:title" :content="product?.name" />
      <Meta property="og:description" :content="product?.description" />
      <Meta property="og:image" :content="product?.image" />
      <Meta property="og:type" content="product" />
    </Head>

    <div v-if="pending" class="loading">
      載入中...
    </div>

    <div v-else-if="error" class="error">
      {{ error.message }}
    </div>

    <div v-else-if="product" class="product-container">
      <!-- 產品圖片 -->
      <div class="product-images">
        <img :src="product.image" :alt="product.name" />
      </div>

      <!-- 產品資訊 -->
      <div class="product-info">
        <h1>{{ product.name }}</h1>
        <p class="price">${{ product.price }}</p>
        <p class="description">{{ product.description }}</p>

        <!-- 庫存狀態 -->
        <div class="stock">
          <span v-if="product.stock > 0" class="in-stock">
            ✅ 庫存充足 ({{ product.stock }} 件)
          </span>
          <span v-else class="out-of-stock">
            ❌ 已售完
          </span>
        </div>

        <!-- 數量選擇（客戶端互動） -->
        <div class="quantity">
          <label>數量:</label>
          <button @click="decreaseQuantity" :disabled="quantity <= 1">-</button>
          <input v-model.number="quantity" type="number" min="1" :max="product.stock" />
          <button @click="increaseQuantity" :disabled="quantity >= product.stock">+</button>
        </div>

        <!-- 加入購物車按鈕 -->
        <button
          @click="addToCart"
          :disabled="product.stock === 0 || isAdding"
          class="add-to-cart"
        >
          {{ isAdding ? '加入中...' : '加入購物車' }}
        </button>

        <!-- 客戶端顯示購物車狀態 -->
        <ClientOnly>
          <div v-if="cartMessage" class="cart-message">
            {{ cartMessage }}
          </div>
        </ClientOnly>
      </div>

      <!-- 產品評論（伺服器渲染以利 SEO） -->
      <div class="reviews">
        <h2>顧客評論</h2>
        <div v-for="review in product.reviews" :key="review.id" class="review">
          <div class="review-header">
            <span class="author">{{ review.author }}</span>
            <span class="rating">⭐ {{ review.rating }}/5</span>
          </div>
          <p class="review-content">{{ review.content }}</p>
        </div>
      </div>

      <!-- 相關產品 -->
      <div class="related-products">
        <h2>相關產品</h2>
        <div class="product-grid">
          <NuxtLink
            v-for="item in relatedProducts"
            :key="item.id"
            :to="`/products/${item.id}`"
            class="product-card"
          >
            <img :src="item.image" :alt="item.name" />
            <h3>{{ item.name }}</h3>
            <p class="price">${{ item.price }}</p>
          </NuxtLink>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
const route = useRoute()
const productId = route.params.id

// 客戶端狀態
const quantity = ref(1)
const isAdding = ref(false)
const cartMessage = ref('')

// SSR: 在伺服器端獲取產品資料
const { data: product, pending, error } = await useFetch(`/api/products/${productId}`, {
  // 這個請求會在伺服器端執行，資料會被序列化到 HTML
  key: `product-${productId}`,
})

// SSR: 獲取相關產品
const { data: relatedProducts } = await useFetch('/api/products/related', {
  query: { productId },
  key: `related-${productId}`,
})

// 客戶端方法
const increaseQuantity = () => {
  if (quantity.value < product.value.stock) {
    quantity.value++
  }
}

const decreaseQuantity = () => {
  if (quantity.value > 1) {
    quantity.value--
  }
}

const addToCart = async () => {
  isAdding.value = true

  try {
    // 呼叫 API 加入購物車
    await $fetch('/api/cart/add', {
      method: 'POST',
      body: {
        productId: product.value.id,
        quantity: quantity.value
      }
    })

    cartMessage.value = `✅ 已加入 ${quantity.value} 件到購物車`

    // 3 秒後清除訊息
    setTimeout(() => {
      cartMessage.value = ''
    }, 3000)
  } catch (error) {
    cartMessage.value = `❌ 加入失敗: ${error.message}`
  } finally {
    isAdding.value = false
  }
}

// 客戶端追蹤（只在客戶端執行）
onMounted(() => {
  // 追蹤頁面瀏覽
  if (window.gtag) {
    window.gtag('event', 'page_view', {
      page_path: route.path,
      page_title: product.value?.name
    })
  }

  // 追蹤產品瀏覽
  console.log('User viewed product:', product.value?.name)
})
</script>

<style scoped>
.product-page {
  max-width: 1200px;
  margin: 0 auto;
  padding: 20px;
}

.product-container {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 40px;
  margin-bottom: 60px;
}

.product-images img {
  width: 100%;
  border-radius: 8px;
}

.product-info h1 {
  font-size: 32px;
  margin-bottom: 16px;
}

.price {
  font-size: 24px;
  color: #e74c3c;
  font-weight: bold;
  margin-bottom: 16px;
}

.quantity {
  display: flex;
  align-items: center;
  gap: 10px;
  margin: 20px 0;
}

.quantity input {
  width: 60px;
  text-align: center;
  padding: 8px;
  border: 1px solid #ddd;
  border-radius: 4px;
}

.quantity button {
  padding: 8px 16px;
  border: 1px solid #ddd;
  background: white;
  border-radius: 4px;
  cursor: pointer;
}

.add-to-cart {
  width: 100%;
  padding: 16px;
  background: #3498db;
  color: white;
  border: none;
  border-radius: 8px;
  font-size: 18px;
  cursor: pointer;
  transition: background 0.3s;
}

.add-to-cart:hover:not(:disabled) {
  background: #2980b9;
}

.add-to-cart:disabled {
  background: #bdc3c7;
  cursor: not-allowed;
}

.cart-message {
  margin-top: 16px;
  padding: 12px;
  border-radius: 4px;
  background: #d4edda;
  color: #155724;
}

.reviews {
  margin: 40px 0;
}

.review {
  border: 1px solid #ddd;
  border-radius: 8px;
  padding: 16px;
  margin-bottom: 16px;
}

.review-header {
  display: flex;
  justify-content: space-between;
  margin-bottom: 8px;
  font-weight: bold;
}

.related-products {
  margin-top: 60px;
}

.product-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(200px, 1fr));
  gap: 20px;
}

.product-card {
  border: 1px solid #ddd;
  border-radius: 8px;
  padding: 16px;
  text-decoration: none;
  color: inherit;
  transition: box-shadow 0.3s;
}

.product-card:hover {
  box-shadow: 0 4px 8px rgba(0,0,0,0.1);
}

.product-card img {
  width: 100%;
  border-radius: 4px;
  margin-bottom: 12px;
}

@media (max-width: 768px) {
  .product-container {
    grid-template-columns: 1fr;
  }
}
</style>
```

### API 路由範例

```javascript
// server/api/products/[id].get.js
export default defineEventHandler(async (event) => {
  const id = getRouterParam(event, 'id')

  // 模擬資料庫查詢
  // 實際應用中會從資料庫獲取
  const product = {
    id: id,
    name: 'Nuxt 3 教學書籍',
    price: 599,
    description: '深入淺出學習 Nuxt 3，從基礎到進階的完整指南。',
    image: 'https://picsum.photos/600/600',
    stock: 50,
    reviews: [
      {
        id: 1,
        author: '張小明',
        rating: 5,
        content: '非常實用的教學書，內容詳細易懂！'
      },
      {
        id: 2,
        author: '李小華',
        rating: 4,
        content: '對初學者很友善，範例豐富。'
      }
    ]
  }

  // 模擬網路延遲
  await new Promise(resolve => setTimeout(resolve, 100))

  return product
})
```

```javascript
// server/api/products/related.get.js
export default defineEventHandler(async (event) => {
  const query = getQuery(event)
  const productId = query.productId

  // 模擬相關產品
  const relatedProducts = [
    {
      id: 'nuxt-advanced',
      name: 'Nuxt 3 進階開發',
      price: 799,
      image: 'https://picsum.photos/200/200?random=1'
    },
    {
      id: 'vue-basics',
      name: 'Vue 3 基礎課程',
      price: 499,
      image: 'https://picsum.photos/200/200?random=2'
    },
    {
      id: 'typescript-guide',
      name: 'TypeScript 完全指南',
      price: 699,
      image: 'https://picsum.photos/200/200?random=3'
    }
  ]

  return relatedProducts
})
```

## 效能最佳化

### 1. 使用快取

```javascript
// nuxt.config.ts
export default defineNuxtConfig({
  routeRules: {
    // 快取首頁 1 小時
    '/': { swr: 3600 },

    // 快取產品列表 10 分鐘
    '/products': { swr: 600 },

    // 快取個別產品頁面 5 分鐘
    '/products/**': { swr: 300 },

    // API 路由快取
    '/api/products': { swr: 600 }
  }
})
```

### 2. 資料預取策略

```vue
<script setup>
// ❌ 序列請求 - 慢
const { data: user } = await useFetch('/api/user')
const { data: posts } = await useFetch(`/api/posts?userId=${user.value.id}`)

// ✅ 並行請求 - 快
const [
  { data: user },
  { data: posts },
  { data: categories }
] = await Promise.all([
  useFetch('/api/user'),
  useFetch('/api/posts'),
  useFetch('/api/categories')
])
</script>
```

### 3. 組件延遲載入

```vue
<template>
  <div>
    <!-- 立即載入關鍵內容 -->
    <ProductHeader :product="product" />

    <!-- 延遲載入次要內容 -->
    <LazyProductReviews :product-id="product.id" />
    <LazyRelatedProducts :category="product.category" />
  </div>
</template>
```

### 4. Payload 最佳化

```vue
<script setup>
// 只取得需要的欄位
const { data: products } = await useFetch('/api/products', {
  // 使用 query 參數限制回傳欄位
  query: {
    fields: 'id,name,price,image' // 不需要完整描述等資訊
  }
})

// 使用 transform 清理資料
const { data: user } = await useFetch('/api/user', {
  transform: (data) => {
    // 移除敏感資料
    const { password, email, ...publicData } = data
    return publicData
  }
})
</script>
```

### 5. 錯誤處理與降級

```vue
<script setup>
const { data: criticalData, error: criticalError } = await useFetch('/api/critical', {
  // 發生錯誤時不中斷 SSR
  onResponseError({ response }) {
    console.error('Critical data fetch failed:', response.status)
  }
})

// 提供預設值
const displayData = computed(() => {
  if (criticalError.value) {
    return { message: '資料載入失敗，請稍後再試' }
  }
  return criticalData.value
})
</script>
```

### 6. 條件渲染大型內容

```vue
<template>
  <div>
    <!-- 小螢幕不渲染側邊欄 -->
    <aside v-if="!isMobile" class="sidebar">
      <SidebarContent />
    </aside>

    <!-- 主要內容 -->
    <main>
      <MainContent />
    </main>
  </div>
</template>

<script setup>
const { width } = useWindowSize()
const isMobile = computed(() => width.value < 768)
</script>
```

## SSR 最佳實踐

### ✅ 應該做的事

1. **使用 `useFetch` 和 `useAsyncData`**
   - 自動處理 SSR 和客戶端
   - 避免重複請求
   - 自動序列化狀態

2. **善用 `process.client` 和 `process.server`**
   - 區分環境特定代碼
   - 避免不必要的伺服器端執行

3. **使用 `ClientOnly` 組件**
   - 包裝純客戶端組件
   - 避免 Hydration 問題

4. **設定適當的快取策略**
   - 減少伺服器負載
   - 提升響應速度

5. **處理錯誤和邊界情況**
   - 提供降級方案
   - 友善的錯誤訊息

### ❌ 不應該做的事

1. **不要直接使用瀏覽器 API**
   - 不檢查環境就使用 `window`、`document`
   - 應該使用條件判斷或 `onMounted`

2. **不要在 SSR 中執行副作用**
   - 避免修改全域狀態
   - 不要在 `<script setup>` 頂層操作 DOM

3. **不要忽略 Hydration Mismatch**
   - 確保伺服器和客戶端渲染一致
   - 避免時間戳等動態內容

4. **不要過度依賴 SSR**
   - 某些頁面可能更適合 CSR（如管理後台）
   - 根據需求選擇渲染策略

## 總結

SSR 是 Nuxt 3 的核心功能，提供了：

- ✅ 更好的 SEO 和社群媒體分享
- ✅ 更快的首次內容顯示
- ✅ 更好的使用者體驗
- ⚠️ 需要處理環境差異
- ⚠️ 需要適當的快取策略

選擇 SSR 的時機：
- 需要 SEO 的公開頁面（產品頁、部落格、官網）
- 需要社群媒體預覽的頁面
- 內容為主的網站
- 需要快速首次顯示的應用

不適合 SSR 的情境：
- 純後台管理系統（無 SEO 需求）
- 高度互動的應用（如遊戲、繪圖工具）
- 依賴大量客戶端 API 的應用

下一章我們將探討 **SSG (Static Site Generation)**，了解如何將頁面預先生成為靜態檔案！
