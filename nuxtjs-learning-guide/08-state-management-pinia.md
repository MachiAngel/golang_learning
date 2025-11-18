# 08 - 狀態管理 - Pinia 整合

## 概述

Pinia 是 Vue 3 官方推薦的狀態管理庫，提供了類型安全、模組化和直觀的 API。本章將介紹如何在 Nuxt 3 中整合和使用 Pinia。

## 1. 安裝與設定 Pinia

### 安裝

```bash
npm install @pinia/nuxt pinia
```

### 配置 Nuxt

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  modules: [
    '@pinia/nuxt',
  ],
})
```

### 目錄結構

```
stores/
├── auth.ts
├── cart.ts
├── products.ts
└── index.ts
```

## 2. 建立 Store

### 基本 Store 結構（Setup Stores 風格）

```typescript
// stores/counter.ts
export const useCounterStore = defineStore('counter', () => {
  // State
  const count = ref(0)
  const name = ref('Counter')

  // Getters (Computed)
  const doubleCount = computed(() => count.value * 2)
  const isPositive = computed(() => count.value > 0)

  // Actions (Methods)
  const increment = () => {
    count.value++
  }

  const decrement = () => {
    count.value--
  }

  const incrementBy = (amount: number) => {
    count.value += amount
  }

  const reset = () => {
    count.value = 0
  }

  return {
    // State
    count,
    name,
    // Getters
    doubleCount,
    isPositive,
    // Actions
    increment,
    decrement,
    incrementBy,
    reset
  }
})
```

### Options API 風格 Store

```typescript
// stores/user.ts
interface User {
  id: number
  name: string
  email: string
  avatar?: string
}

interface UserState {
  user: User | null
  token: string | null
  isLoading: boolean
}

export const useUserStore = defineStore('user', {
  // State
  state: (): UserState => ({
    user: null,
    token: null,
    isLoading: false
  }),

  // Getters
  getters: {
    isAuthenticated: (state) => !!state.user && !!state.token,
    userName: (state) => state.user?.name || 'Guest',
    userInitials: (state) => {
      if (!state.user) return '?'
      return state.user.name
        .split(' ')
        .map(n => n[0])
        .join('')
        .toUpperCase()
    }
  },

  // Actions
  actions: {
    async login(email: string, password: string) {
      this.isLoading = true
      try {
        const response = await $fetch<{ user: User; token: string }>('/api/auth/login', {
          method: 'POST',
          body: { email, password }
        })

        this.user = response.user
        this.token = response.token

        // 儲存 token 到 cookie
        const tokenCookie = useCookie('auth-token')
        tokenCookie.value = response.token

        return { success: true }
      } catch (error: any) {
        return { success: false, error: error.message }
      } finally {
        this.isLoading = false
      }
    },

    async logout() {
      try {
        await $fetch('/api/auth/logout', { method: 'POST' })
      } finally {
        this.user = null
        this.token = null

        const tokenCookie = useCookie('auth-token')
        tokenCookie.value = null
      }
    },

    async fetchUser() {
      if (!this.token) return

      try {
        const user = await $fetch<User>('/api/auth/me', {
          headers: {
            Authorization: `Bearer ${this.token}`
          }
        })
        this.user = user
      } catch (error) {
        // Token 無效，清除登入狀態
        this.logout()
      }
    },

    updateProfile(updates: Partial<User>) {
      if (this.user) {
        this.user = { ...this.user, ...updates }
      }
    }
  }
})
```

## 3. State、Getters、Actions

### 完整購物車範例

```typescript
// stores/cart.ts
interface Product {
  id: number
  name: string
  price: number
  image: string
  description?: string
}

interface CartItem extends Product {
  quantity: number
}

interface CartState {
  items: CartItem[]
  checkoutStatus: 'idle' | 'pending' | 'success' | 'error'
  lastError: string | null
}

export const useCartStore = defineStore('cart', {
  // State
  state: (): CartState => ({
    items: [],
    checkoutStatus: 'idle',
    lastError: null
  }),

  // Getters
  getters: {
    // 購物車商品總數
    totalItems: (state) => {
      return state.items.reduce((total, item) => total + item.quantity, 0)
    },

    // 購物車總金額
    totalPrice: (state) => {
      return state.items.reduce((total, item) => {
        return total + item.price * item.quantity
      }, 0)
    },

    // 格式化的總金額
    formattedTotalPrice(): string {
      return `NT$ ${this.totalPrice.toLocaleString('zh-TW')}`
    },

    // 購物車是否為空
    isEmpty: (state) => state.items.length === 0,

    // 取得特定商品
    getItemById: (state) => {
      return (productId: number) => {
        return state.items.find(item => item.id === productId)
      }
    },

    // 檢查商品是否在購物車中
    hasItem: (state) => {
      return (productId: number) => {
        return state.items.some(item => item.id === productId)
      }
    },

    // 計算折扣（滿 1000 打 9 折）
    discount(): number {
      return this.totalPrice >= 1000 ? this.totalPrice * 0.1 : 0
    },

    // 最終金額（含折扣）
    finalPrice(): number {
      return this.totalPrice - this.discount
    }
  },

  // Actions
  actions: {
    // 加入商品到購物車
    addItem(product: Product, quantity = 1) {
      const existingItem = this.items.find(item => item.id === product.id)

      if (existingItem) {
        existingItem.quantity += quantity
      } else {
        this.items.push({
          ...product,
          quantity
        })
      }

      // 儲存到 localStorage
      this.saveToLocalStorage()
    },

    // 移除商品
    removeItem(productId: number) {
      const index = this.items.findIndex(item => item.id === productId)
      if (index > -1) {
        this.items.splice(index, 1)
        this.saveToLocalStorage()
      }
    },

    // 更新商品數量
    updateQuantity(productId: number, quantity: number) {
      const item = this.items.find(item => item.id === productId)

      if (item) {
        if (quantity <= 0) {
          this.removeItem(productId)
        } else {
          item.quantity = quantity
          this.saveToLocalStorage()
        }
      }
    },

    // 增加數量
    incrementQuantity(productId: number) {
      const item = this.items.find(item => item.id === productId)
      if (item) {
        item.quantity++
        this.saveToLocalStorage()
      }
    },

    // 減少數量
    decrementQuantity(productId: number) {
      const item = this.items.find(item => item.id === productId)
      if (item) {
        if (item.quantity > 1) {
          item.quantity--
        } else {
          this.removeItem(productId)
        }
        this.saveToLocalStorage()
      }
    },

    // 清空購物車
    clear() {
      this.items = []
      this.saveToLocalStorage()
    },

    // 結帳
    async checkout() {
      this.checkoutStatus = 'pending'
      this.lastError = null

      try {
        const orderData = {
          items: this.items.map(item => ({
            productId: item.id,
            quantity: item.quantity,
            price: item.price
          })),
          totalAmount: this.finalPrice
        }

        const response = await $fetch('/api/checkout', {
          method: 'POST',
          body: orderData
        })

        this.checkoutStatus = 'success'
        this.clear()

        return { success: true, orderId: response.orderId }
      } catch (error: any) {
        this.checkoutStatus = 'error'
        this.lastError = error.message || '結帳失敗'
        return { success: false, error: this.lastError }
      }
    },

    // 儲存到 localStorage
    saveToLocalStorage() {
      if (process.client) {
        localStorage.setItem('cart', JSON.stringify(this.items))
      }
    },

    // 從 localStorage 載入
    loadFromLocalStorage() {
      if (process.client) {
        const saved = localStorage.getItem('cart')
        if (saved) {
          try {
            this.items = JSON.parse(saved)
          } catch (e) {
            console.error('Failed to parse cart data:', e)
          }
        }
      }
    },

    // 重置結帳狀態
    resetCheckoutStatus() {
      this.checkoutStatus = 'idle'
      this.lastError = null
    }
  }
})
```

## 4. 在組件中使用 Store

### 基本使用

```vue
<template>
  <div class="cart-summary">
    <h2>購物車</h2>

    <div v-if="cart.isEmpty" class="empty-cart">
      <p>購物車是空的</p>
      <NuxtLink to="/products">去逛逛</NuxtLink>
    </div>

    <div v-else>
      <div class="cart-items">
        <div v-for="item in cart.items" :key="item.id" class="cart-item">
          <img :src="item.image" :alt="item.name" />
          <div class="item-details">
            <h3>{{ item.name }}</h3>
            <p class="price">NT$ {{ item.price.toLocaleString() }}</p>
          </div>

          <div class="quantity-controls">
            <button @click="cart.decrementQuantity(item.id)">-</button>
            <span>{{ item.quantity }}</span>
            <button @click="cart.incrementQuantity(item.id)">+</button>
          </div>

          <button @click="cart.removeItem(item.id)" class="remove-btn">
            移除
          </button>
        </div>
      </div>

      <div class="cart-totals">
        <div class="total-row">
          <span>商品總計</span>
          <span>NT$ {{ cart.totalPrice.toLocaleString() }}</span>
        </div>
        <div v-if="cart.discount > 0" class="total-row discount">
          <span>折扣</span>
          <span>- NT$ {{ cart.discount.toLocaleString() }}</span>
        </div>
        <div class="total-row final">
          <span>總計</span>
          <span>{{ cart.formattedTotalPrice }}</span>
        </div>
      </div>

      <div class="cart-actions">
        <button @click="cart.clear()" class="clear-btn">清空購物車</button>
        <button
          @click="handleCheckout"
          :disabled="cart.checkoutStatus === 'pending'"
          class="checkout-btn"
        >
          {{ cart.checkoutStatus === 'pending' ? '處理中...' : '結帳' }}
        </button>
      </div>

      <div v-if="cart.lastError" class="error-message">
        {{ cart.lastError }}
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
const cart = useCartStore()

// 載入購物車資料
onMounted(() => {
  cart.loadFromLocalStorage()
})

const handleCheckout = async () => {
  const result = await cart.checkout()

  if (result.success) {
    alert(`訂單 ${result.orderId} 已成立！`)
    navigateTo('/orders')
  } else {
    alert(`結帳失敗：${result.error}`)
  }
}
</script>

<style scoped>
.cart-summary {
  max-width: 800px;
  margin: 0 auto;
  padding: 24px;
}

.empty-cart {
  text-align: center;
  padding: 48px;
  color: #95a5a6;
}

.cart-item {
  display: flex;
  align-items: center;
  gap: 16px;
  padding: 16px;
  border: 1px solid #e0e0e0;
  border-radius: 8px;
  margin-bottom: 12px;
}

.cart-item img {
  width: 80px;
  height: 80px;
  object-fit: cover;
  border-radius: 4px;
}

.item-details {
  flex: 1;
}

.quantity-controls {
  display: flex;
  align-items: center;
  gap: 8px;
}

.quantity-controls button {
  width: 32px;
  height: 32px;
  border: 1px solid #ddd;
  background: white;
  cursor: pointer;
  border-radius: 4px;
}

.remove-btn {
  padding: 8px 16px;
  background-color: #e74c3c;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
}

.cart-totals {
  margin: 24px 0;
  padding: 16px;
  background-color: #f8f9fa;
  border-radius: 8px;
}

.total-row {
  display: flex;
  justify-content: space-between;
  margin-bottom: 8px;
}

.total-row.discount {
  color: #e74c3c;
}

.total-row.final {
  font-size: 1.25rem;
  font-weight: bold;
  padding-top: 12px;
  border-top: 2px solid #ddd;
}

.cart-actions {
  display: flex;
  gap: 12px;
}

.clear-btn,
.checkout-btn {
  flex: 1;
  padding: 12px;
  border: none;
  border-radius: 6px;
  font-size: 16px;
  cursor: pointer;
}

.clear-btn {
  background-color: #ecf0f1;
  color: #7f8c8d;
}

.checkout-btn {
  background-color: #27ae60;
  color: white;
}

.checkout-btn:disabled {
  background-color: #95a5a6;
  cursor: not-allowed;
}

.error-message {
  margin-top: 16px;
  padding: 12px;
  background-color: #ffe6e6;
  color: #e74c3c;
  border-radius: 4px;
}
</style>
```

### 在產品列表中使用

```vue
<template>
  <div class="product-list">
    <div v-for="product in products" :key="product.id" class="product-card">
      <img :src="product.image" :alt="product.name" />
      <h3>{{ product.name }}</h3>
      <p>{{ product.description }}</p>
      <p class="price">NT$ {{ product.price.toLocaleString() }}</p>

      <button
        @click="handleAddToCart(product)"
        :class="{ 'in-cart': cart.hasItem(product.id) }"
      >
        {{ cart.hasItem(product.id) ? '已在購物車' : '加入購物車' }}
      </button>
    </div>
  </div>
</template>

<script setup lang="ts">
const cart = useCartStore()

const products = ref([
  {
    id: 1,
    name: 'MacBook Pro 14"',
    price: 59900,
    image: '/images/macbook.jpg',
    description: 'M3 Pro 晶片，極致效能'
  },
  {
    id: 2,
    name: 'iPhone 15 Pro',
    price: 36900,
    image: '/images/iphone.jpg',
    description: '鈦金屬設計，A17 Pro'
  }
])

const handleAddToCart = (product: any) => {
  cart.addItem(product)
  // 可以顯示通知
  alert(`${product.name} 已加入購物車`)
}
</script>
```

### Store 組合使用

```vue
<script setup lang="ts">
// 使用多個 store
const userStore = useUserStore()
const cartStore = useCartStore()
const productsStore = useProductsStore()

// 組合使用
const canCheckout = computed(() => {
  return userStore.isAuthenticated && !cartStore.isEmpty
})

const handleCheckout = async () => {
  if (!userStore.isAuthenticated) {
    // 導向登入頁面
    await navigateTo('/login')
    return
  }

  const result = await cartStore.checkout()
  if (result.success) {
    // 結帳成功，導向訂單頁面
    await navigateTo(`/orders/${result.orderId}`)
  }
}
</script>
```

## 5. SSR 注意事項

### 狀態初始化

```typescript
// stores/products.ts
export const useProductsStore = defineStore('products', () => {
  const products = ref<Product[]>([])
  const isLoading = ref(false)

  const fetchProducts = async () => {
    // 避免重複獲取
    if (products.value.length > 0) {
      return
    }

    isLoading.value = true
    try {
      const data = await $fetch<Product[]>('/api/products')
      products.value = data
    } finally {
      isLoading.value = false
    }
  }

  return {
    products: readonly(products),
    isLoading: readonly(isLoading),
    fetchProducts
  }
})
```

### 在 Plugin 中初始化

```typescript
// plugins/init-stores.ts
export default defineNuxtPlugin(async (nuxtApp) => {
  const userStore = useUserStore()

  // 從 cookie 恢復登入狀態
  const token = useCookie('auth-token')
  if (token.value) {
    userStore.token = token.value
    await userStore.fetchUser()
  }
})
```

### 狀態水合（Hydration）

```typescript
// stores/auth.ts
export const useAuthStore = defineStore('auth', () => {
  const user = ref<User | null>(null)

  // SSR 友善的初始化
  const initialize = async () => {
    // 只在客戶端執行
    if (process.client) {
      const token = localStorage.getItem('auth-token')
      if (token) {
        await fetchUser(token)
      }
    }
  }

  return {
    user,
    initialize
  }
})
```

### 使用 $nuxt.payload

```typescript
// server/api/initial-data.ts
export default defineEventHandler(() => {
  return {
    user: { id: 1, name: 'John' },
    settings: { theme: 'dark' }
  }
})

// app.vue
<script setup lang="ts">
const { data } = await useFetch('/api/initial-data')

if (data.value) {
  const userStore = useUserStore()
  userStore.user = data.value.user
}
</script>
```

## 6. 完整購物車範例（整合所有功能）

### 產品 Store

```typescript
// stores/products.ts
interface Product {
  id: number
  name: string
  price: number
  image: string
  description: string
  stock: number
  category: string
}

export const useProductsStore = defineStore('products', () => {
  const products = ref<Product[]>([])
  const isLoading = ref(false)
  const error = ref<string | null>(null)

  // Getters
  const categories = computed(() => {
    return [...new Set(products.value.map(p => p.category))]
  })

  const getProductById = computed(() => {
    return (id: number) => products.value.find(p => p.id === id)
  })

  const getProductsByCategory = computed(() => {
    return (category: string) => {
      return products.value.filter(p => p.category === category)
    }
  })

  // Actions
  const fetchProducts = async () => {
    if (products.value.length > 0) return

    isLoading.value = true
    error.value = null

    try {
      const data = await $fetch<Product[]>('/api/products')
      products.value = data
    } catch (e: any) {
      error.value = e.message
    } finally {
      isLoading.value = false
    }
  }

  const updateStock = (productId: number, quantity: number) => {
    const product = products.value.find(p => p.id === productId)
    if (product) {
      product.stock -= quantity
    }
  }

  return {
    products: readonly(products),
    isLoading: readonly(isLoading),
    error: readonly(error),
    categories,
    getProductById,
    getProductsByCategory,
    fetchProducts,
    updateStock
  }
})
```

### 完整頁面範例

```vue
<!-- pages/shop.vue -->
<template>
  <div class="shop-page">
    <header class="shop-header">
      <h1>線上商店</h1>
      <div class="cart-badge" @click="showCart = true">
        🛒 購物車 ({{ cartStore.totalItems }})
      </div>
    </header>

    <!-- 分類篩選 -->
    <div class="categories">
      <button
        @click="selectedCategory = null"
        :class="{ active: selectedCategory === null }"
      >
        全部
      </button>
      <button
        v-for="category in productsStore.categories"
        :key="category"
        @click="selectedCategory = category"
        :class="{ active: selectedCategory === category }"
      >
        {{ category }}
      </button>
    </div>

    <!-- 產品列表 -->
    <div v-if="productsStore.isLoading" class="loading">
      載入中...
    </div>

    <div v-else class="products-grid">
      <div
        v-for="product in filteredProducts"
        :key="product.id"
        class="product-card"
      >
        <img :src="product.image" :alt="product.name" />
        <h3>{{ product.name }}</h3>
        <p class="description">{{ product.description }}</p>
        <div class="product-footer">
          <span class="price">NT$ {{ product.price.toLocaleString() }}</span>
          <span class="stock">庫存: {{ product.stock }}</span>
        </div>
        <button
          @click="addToCart(product)"
          :disabled="product.stock === 0"
          :class="{ 'in-cart': cartStore.hasItem(product.id) }"
        >
          {{ product.stock === 0 ? '售完' : cartStore.hasItem(product.id) ? '已在購物車' : '加入購物車' }}
        </button>
      </div>
    </div>

    <!-- 購物車側邊欄 -->
    <div v-if="showCart" class="cart-sidebar" @click.self="showCart = false">
      <div class="cart-content">
        <div class="cart-header">
          <h2>購物車</h2>
          <button @click="showCart = false">✕</button>
        </div>

        <div v-if="cartStore.isEmpty" class="empty-cart">
          購物車是空的
        </div>

        <div v-else>
          <div class="cart-items">
            <div v-for="item in cartStore.items" :key="item.id" class="cart-item">
              <img :src="item.image" :alt="item.name" />
              <div class="item-info">
                <h4>{{ item.name }}</h4>
                <p>NT$ {{ item.price }}</p>
              </div>
              <div class="quantity-controls">
                <button @click="cartStore.decrementQuantity(item.id)">-</button>
                <span>{{ item.quantity }}</span>
                <button @click="cartStore.incrementQuantity(item.id)">+</button>
              </div>
            </div>
          </div>

          <div class="cart-summary">
            <div class="summary-row">
              <span>小計</span>
              <span>NT$ {{ cartStore.totalPrice.toLocaleString() }}</span>
            </div>
            <div v-if="cartStore.discount > 0" class="summary-row discount">
              <span>折扣</span>
              <span>- NT$ {{ cartStore.discount.toLocaleString() }}</span>
            </div>
            <div class="summary-row total">
              <span>總計</span>
              <span>NT$ {{ cartStore.finalPrice.toLocaleString() }}</span>
            </div>

            <button @click="handleCheckout" class="checkout-btn">
              前往結帳
            </button>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
const productsStore = useProductsStore()
const cartStore = useCartStore()
const showCart = ref(false)
const selectedCategory = ref<string | null>(null)

// 篩選產品
const filteredProducts = computed(() => {
  if (!selectedCategory.value) {
    return productsStore.products
  }
  return productsStore.getProductsByCategory(selectedCategory.value)
})

// 載入產品
onMounted(async () => {
  await productsStore.fetchProducts()
  cartStore.loadFromLocalStorage()
})

// 加入購物車
const addToCart = (product: any) => {
  if (product.stock > 0) {
    cartStore.addItem(product)
  }
}

// 結帳
const handleCheckout = () => {
  showCart.value = false
  navigateTo('/checkout')
}
</script>

<style scoped>
.shop-page {
  max-width: 1200px;
  margin: 0 auto;
  padding: 24px;
}

.shop-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 32px;
}

.cart-badge {
  padding: 12px 24px;
  background-color: #3498db;
  color: white;
  border-radius: 24px;
  cursor: pointer;
  font-weight: bold;
}

.categories {
  display: flex;
  gap: 12px;
  margin-bottom: 24px;
  flex-wrap: wrap;
}

.categories button {
  padding: 8px 16px;
  border: 1px solid #ddd;
  background: white;
  border-radius: 20px;
  cursor: pointer;
  transition: all 0.2s;
}

.categories button.active {
  background-color: #3498db;
  color: white;
  border-color: #3498db;
}

.products-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(280px, 1fr));
  gap: 24px;
}

.product-card {
  border: 1px solid #e0e0e0;
  border-radius: 12px;
  padding: 16px;
  transition: transform 0.2s, box-shadow 0.2s;
}

.product-card:hover {
  transform: translateY(-4px);
  box-shadow: 0 4px 12px rgba(0,0,0,0.1);
}

.product-card img {
  width: 100%;
  height: 200px;
  object-fit: cover;
  border-radius: 8px;
  margin-bottom: 12px;
}

.product-footer {
  display: flex;
  justify-content: space-between;
  margin: 12px 0;
}

.price {
  font-size: 1.25rem;
  font-weight: bold;
  color: #e74c3c;
}

.stock {
  color: #95a5a6;
}

.product-card button {
  width: 100%;
  padding: 12px;
  border: none;
  border-radius: 8px;
  background-color: #27ae60;
  color: white;
  cursor: pointer;
  font-weight: bold;
  transition: background-color 0.2s;
}

.product-card button:hover:not(:disabled) {
  background-color: #229954;
}

.product-card button:disabled {
  background-color: #bdc3c7;
  cursor: not-allowed;
}

.product-card button.in-cart {
  background-color: #3498db;
}

/* 購物車側邊欄 */
.cart-sidebar {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background-color: rgba(0,0,0,0.5);
  z-index: 1000;
  display: flex;
  justify-content: flex-end;
}

.cart-content {
  width: 400px;
  background: white;
  height: 100vh;
  overflow-y: auto;
  padding: 24px;
}

.cart-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 24px;
}

.cart-header button {
  background: none;
  border: none;
  font-size: 24px;
  cursor: pointer;
}

.cart-item {
  display: flex;
  gap: 12px;
  padding: 12px;
  border-bottom: 1px solid #eee;
}

.cart-item img {
  width: 60px;
  height: 60px;
  object-fit: cover;
  border-radius: 4px;
}

.quantity-controls {
  display: flex;
  align-items: center;
  gap: 8px;
}

.quantity-controls button {
  width: 24px;
  height: 24px;
  border: 1px solid #ddd;
  background: white;
  cursor: pointer;
  border-radius: 4px;
}

.cart-summary {
  margin-top: 24px;
  padding-top: 16px;
  border-top: 2px solid #ddd;
}

.summary-row {
  display: flex;
  justify-content: space-between;
  margin-bottom: 8px;
}

.summary-row.total {
  font-size: 1.25rem;
  font-weight: bold;
  margin-top: 12px;
  padding-top: 12px;
  border-top: 1px solid #ddd;
}

.checkout-btn {
  width: 100%;
  padding: 16px;
  background-color: #27ae60;
  color: white;
  border: none;
  border-radius: 8px;
  font-size: 16px;
  font-weight: bold;
  cursor: pointer;
  margin-top: 16px;
}
</style>
```

## 最佳實踐建議

### 1. Store 組織

```
stores/
├── index.ts           # 匯出所有 stores
├── auth.ts            # 認證相關
├── cart.ts            # 購物車
├── products.ts        # 產品
└── ui.ts              # UI 狀態（modal, toast 等）
```

### 2. 類型定義

```typescript
// types/store.ts
export interface Product {
  id: number
  name: string
  price: number
}

// stores/products.ts
import type { Product } from '~/types/store'
```

### 3. 避免直接修改 State

```typescript
// ❌ 不好
const cart = useCartStore()
cart.items.push(newItem)

// ✅ 好
const cart = useCartStore()
cart.addItem(newItem)
```

### 4. 使用 Getters 而非重複計算

```typescript
// ❌ 不好
const totalPrice = computed(() => {
  return cart.items.reduce((sum, item) => sum + item.price, 0)
})

// ✅ 好 - 在 Store 中定義 getter
const cart = useCartStore()
const totalPrice = cart.totalPrice
```

## 總結

Pinia 為 Nuxt 3 提供了強大的狀態管理能力：

✅ **類型安全** - 完整的 TypeScript 支援
✅ **模組化** - 每個 store 獨立管理
✅ **DevTools** - 優秀的開發者工具支援
✅ **SSR 友善** - 完整的伺服器端渲染支援
✅ **自動導入** - 無需手動註冊

掌握 Pinia 將使你的 Nuxt 3 應用程式狀態管理更加清晰和高效！
