# 07 - Composables 與組合式 API

## 概述

Composables 是 Vue 3 組合式 API 的核心概念，用於封裝和重用有狀態的邏輯。Nuxt 3 擴展了這個概念，提供了自動導入功能和一系列內建的 Composables。

## 1. composables/ 目錄自動導入

Nuxt 3 會自動掃描 `composables/` 目錄並導入所有函數。

### 目錄結構

```
composables/
├── useCounter.ts
├── useFetch.ts
├── useAuth.ts
└── utils/
    ├── useFormat.ts
    └── useValidation.ts
```

### 自動導入範例

```typescript
// composables/useCounter.ts
export const useCounter = (initialValue = 0) => {
  const count = ref(initialValue)

  const increment = () => count.value++
  const decrement = () => count.value--
  const reset = () => count.value = initialValue

  return {
    count: readonly(count),
    increment,
    decrement,
    reset
  }
}
```

使用時無需 import：

```vue
<template>
  <div>
    <p>計數：{{ count }}</p>
    <button @click="increment">+1</button>
    <button @click="decrement">-1</button>
    <button @click="reset">重置</button>
  </div>
</template>

<script setup lang="ts">
// 自動可用，無需 import
const { count, increment, decrement, reset } = useCounter(10)
</script>
```

## 2. 建立自定義 Composable

### 基本結構

Composables 應該：
- 以 `use` 開頭命名
- 返回響應式數據和方法
- 可以接受參數
- 可以使用其他 composables

### 範例：useToggle

```typescript
// composables/useToggle.ts
export const useToggle = (initialValue = false) => {
  const state = ref(initialValue)

  const toggle = () => {
    state.value = !state.value
  }

  const setTrue = () => {
    state.value = true
  }

  const setFalse = () => {
    state.value = false
  }

  return {
    state: readonly(state),
    toggle,
    setTrue,
    setFalse
  }
}
```

### 範例：useLocalStorage

```typescript
// composables/useLocalStorage.ts
export const useLocalStorage = <T>(key: string, defaultValue: T) => {
  const data = ref<T>(defaultValue)

  // 只在客戶端執行
  if (process.client) {
    const stored = localStorage.getItem(key)
    if (stored) {
      try {
        data.value = JSON.parse(stored)
      } catch (e) {
        console.error('Failed to parse localStorage item:', e)
      }
    }
  }

  // 監聽變化並同步到 localStorage
  watch(data, (newValue) => {
    if (process.client) {
      localStorage.setItem(key, JSON.stringify(newValue))
    }
  }, { deep: true })

  const remove = () => {
    if (process.client) {
      localStorage.removeItem(key)
      data.value = defaultValue
    }
  }

  return {
    data,
    remove
  }
}
```

使用範例：

```vue
<script setup lang="ts">
interface UserSettings {
  theme: 'light' | 'dark'
  language: string
  notifications: boolean
}

const defaultSettings: UserSettings = {
  theme: 'light',
  language: 'zh-TW',
  notifications: true
}

const { data: settings, remove } = useLocalStorage('user-settings', defaultSettings)

const toggleTheme = () => {
  settings.value.theme = settings.value.theme === 'light' ? 'dark' : 'light'
}
</script>

<template>
  <div>
    <p>主題：{{ settings.theme }}</p>
    <button @click="toggleTheme">切換主題</button>
    <button @click="remove">重置設定</button>
  </div>
</template>
```

### 範例：useAsync

```typescript
// composables/useAsync.ts
export const useAsync = <T>(
  asyncFunction: () => Promise<T>
) => {
  const data = ref<T | null>(null)
  const error = ref<Error | null>(null)
  const loading = ref(false)

  const execute = async () => {
    loading.value = true
    error.value = null

    try {
      data.value = await asyncFunction()
    } catch (e) {
      error.value = e as Error
    } finally {
      loading.value = false
    }
  }

  return {
    data: readonly(data),
    error: readonly(error),
    loading: readonly(loading),
    execute
  }
}
```

使用範例：

```vue
<script setup lang="ts">
const fetchUserData = async () => {
  const response = await fetch('/api/user')
  return response.json()
}

const { data, error, loading, execute } = useAsync(fetchUserData)

onMounted(() => {
  execute()
})
</script>

<template>
  <div>
    <div v-if="loading">載入中...</div>
    <div v-else-if="error">錯誤：{{ error.message }}</div>
    <div v-else-if="data">
      <pre>{{ data }}</pre>
    </div>
  </div>
</template>
```

## 3. Nuxt 內建 Composables

### useRoute 和 useRouter

```vue
<script setup lang="ts">
const route = useRoute()
const router = useRouter()

// 取得路由參數
const id = computed(() => route.params.id)

// 取得查詢參數
const search = computed(() => route.query.search)

// 監聽路由變化
watch(() => route.path, (newPath) => {
  console.log('路由變更:', newPath)
})

// 導航方法
const goToHome = () => {
  router.push('/')
}

const goToProduct = (productId: number) => {
  router.push({
    name: 'product-id',
    params: { id: productId }
  })
}

const goBack = () => {
  router.back()
}
</script>

<template>
  <div>
    <p>當前路徑：{{ route.path }}</p>
    <p>文章 ID：{{ id }}</p>
    <p>搜尋關鍵字：{{ search }}</p>

    <button @click="goToHome">返回首頁</button>
    <button @click="goToProduct(123)">查看產品</button>
    <button @click="goBack">返回上一頁</button>
  </div>
</template>
```

### useState - 跨組件共享狀態

```typescript
// composables/useCounter.ts
export const useSharedCounter = () => {
  // useState 會在伺服器和客戶端之間共享狀態
  const count = useState('counter', () => 0)

  const increment = () => count.value++
  const decrement = () => count.value--

  return {
    count,
    increment,
    decrement
  }
}
```

使用範例：

```vue
<!-- components/CounterDisplay.vue -->
<template>
  <div>
    <p>計數：{{ count }}</p>
  </div>
</template>

<script setup lang="ts">
const { count } = useSharedCounter()
</script>
```

```vue
<!-- components/CounterControls.vue -->
<template>
  <div>
    <button @click="increment">+1</button>
    <button @click="decrement">-1</button>
  </div>
</template>

<script setup lang="ts">
const { increment, decrement } = useSharedCounter()
</script>
```

```vue
<!-- pages/index.vue -->
<template>
  <div>
    <h1>計數器範例</h1>
    <!-- 兩個組件共享同一個狀態 -->
    <CounterDisplay />
    <CounterControls />
  </div>
</template>
```

### useFetch 和 useAsyncData

```vue
<script setup lang="ts">
// useFetch - 簡化的資料獲取
const { data: users, pending, error, refresh } = await useFetch('/api/users')

// useAsyncData - 更靈活的控制
const { data: posts } = await useAsyncData('posts', () => {
  return $fetch('/api/posts', {
    query: {
      limit: 10
    }
  })
})

// 帶參數的資料獲取
const userId = ref(1)
const { data: user } = await useFetch(() => `/api/users/${userId.value}`, {
  watch: [userId] // 當 userId 變化時重新獲取
})
</script>

<template>
  <div>
    <div v-if="pending">載入中...</div>
    <div v-else-if="error">發生錯誤</div>
    <div v-else>
      <ul>
        <li v-for="user in users" :key="user.id">
          {{ user.name }}
        </li>
      </ul>
      <button @click="refresh">重新整理</button>
    </div>
  </div>
</template>
```

### useHead - SEO 和 Meta 標籤

```vue
<script setup lang="ts">
const title = ref('我的頁面')

useHead({
  title: title,
  meta: [
    { name: 'description', content: '頁面描述' },
    { property: 'og:title', content: title }
  ],
  link: [
    { rel: 'icon', type: 'image/x-icon', href: '/favicon.ico' }
  ]
})

// 也可以使用 computed
useHead(() => ({
  title: `${title.value} - 我的網站`,
  meta: [
    { name: 'description', content: `關於 ${title.value} 的詳細資訊` }
  ]
}))
</script>
```

### useCookie

```vue
<script setup lang="ts">
// 讀取和設置 cookie
const token = useCookie('auth-token')

// 設置 cookie
token.value = 'new-token-value'

// 帶選項的 cookie
const userPreference = useCookie('user-pref', {
  maxAge: 60 * 60 * 24 * 7, // 7 天
  path: '/',
  secure: true,
  httpOnly: true,
  sameSite: 'strict'
})

// 刪除 cookie
const deleteCookie = () => {
  token.value = null
}
</script>
```

### useRuntimeConfig

```vue
<script setup lang="ts">
const config = useRuntimeConfig()

// 公開配置（客戶端和伺服器端都可用）
const apiBase = config.public.apiBase

// 私有配置（僅伺服器端可用）
// const secretKey = config.secretKey // 只在伺服器端可用
</script>
```

配置檔案：

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  runtimeConfig: {
    // 私有配置（僅伺服器端）
    apiSecret: process.env.API_SECRET,

    // 公開配置（客戶端和伺服器端）
    public: {
      apiBase: process.env.API_BASE_URL || 'https://api.example.com'
    }
  }
})
```

## 4. 狀態共享

### 全域狀態管理

```typescript
// composables/useAppState.ts
export const useAppState = () => {
  const user = useState<User | null>('user', () => null)
  const isAuthenticated = computed(() => !!user.value)

  const login = async (email: string, password: string) => {
    try {
      const response = await $fetch('/api/auth/login', {
        method: 'POST',
        body: { email, password }
      })
      user.value = response.user
    } catch (error) {
      console.error('Login failed:', error)
      throw error
    }
  }

  const logout = async () => {
    await $fetch('/api/auth/logout', { method: 'POST' })
    user.value = null
  }

  return {
    user: readonly(user),
    isAuthenticated,
    login,
    logout
  }
}

interface User {
  id: number
  name: string
  email: string
}
```

### 購物車狀態管理

```typescript
// composables/useCart.ts
export const useCart = () => {
  interface CartItem {
    id: number
    name: string
    price: number
    quantity: number
    image: string
  }

  const items = useState<CartItem[]>('cart-items', () => [])

  const totalItems = computed(() => {
    return items.value.reduce((sum, item) => sum + item.quantity, 0)
  })

  const totalPrice = computed(() => {
    return items.value.reduce((sum, item) => sum + item.price * item.quantity, 0)
  })

  const addItem = (product: Omit<CartItem, 'quantity'>) => {
    const existingItem = items.value.find(item => item.id === product.id)

    if (existingItem) {
      existingItem.quantity++
    } else {
      items.value.push({ ...product, quantity: 1 })
    }
  }

  const removeItem = (productId: number) => {
    const index = items.value.findIndex(item => item.id === productId)
    if (index > -1) {
      items.value.splice(index, 1)
    }
  }

  const updateQuantity = (productId: number, quantity: number) => {
    const item = items.value.find(item => item.id === productId)
    if (item) {
      if (quantity <= 0) {
        removeItem(productId)
      } else {
        item.quantity = quantity
      }
    }
  }

  const clear = () => {
    items.value = []
  }

  return {
    items: readonly(items),
    totalItems,
    totalPrice,
    addItem,
    removeItem,
    updateQuantity,
    clear
  }
}
```

使用購物車：

```vue
<template>
  <div class="cart">
    <h2>購物車 ({{ totalItems }})</h2>

    <div v-if="items.length === 0" class="empty-cart">
      購物車是空的
    </div>

    <div v-else>
      <div v-for="item in items" :key="item.id" class="cart-item">
        <img :src="item.image" :alt="item.name" />
        <div class="item-details">
          <h3>{{ item.name }}</h3>
          <p>NT$ {{ item.price }}</p>
        </div>
        <div class="item-controls">
          <button @click="updateQuantity(item.id, item.quantity - 1)">-</button>
          <span>{{ item.quantity }}</span>
          <button @click="updateQuantity(item.id, item.quantity + 1)">+</button>
        </div>
        <button @click="removeItem(item.id)" class="remove-btn">移除</button>
      </div>

      <div class="cart-summary">
        <p>總計：NT$ {{ totalPrice.toLocaleString() }}</p>
        <button @click="clear" class="clear-btn">清空購物車</button>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
const { items, totalItems, totalPrice, updateQuantity, removeItem, clear } = useCart()
</script>
```

## 5. 生命週期處理

### 在 Composables 中使用生命週期

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

  // 組件掛載時執行
  onMounted(() => {
    update()
    window.addEventListener('resize', update)
  })

  // 組件卸載時清理
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

### 文件可見性追蹤

```typescript
// composables/usePageVisibility.ts
export const usePageVisibility = () => {
  const isVisible = ref(true)

  const handleVisibilityChange = () => {
    if (process.client) {
      isVisible.value = !document.hidden
    }
  }

  onMounted(() => {
    if (process.client) {
      isVisible.value = !document.hidden
      document.addEventListener('visibilitychange', handleVisibilityChange)
    }
  })

  onUnmounted(() => {
    if (process.client) {
      document.removeEventListener('visibilitychange', handleVisibilityChange)
    }
  })

  return {
    isVisible: readonly(isVisible)
  }
}
```

### 定時器管理

```typescript
// composables/useInterval.ts
export const useInterval = (callback: () => void, interval: number) => {
  const isActive = ref(false)
  let timerId: NodeJS.Timeout | null = null

  const start = () => {
    if (!isActive.value) {
      isActive.value = true
      timerId = setInterval(callback, interval)
    }
  }

  const stop = () => {
    if (timerId) {
      clearInterval(timerId)
      timerId = null
      isActive.value = false
    }
  }

  const restart = () => {
    stop()
    start()
  }

  // 組件卸載時自動清理
  onUnmounted(() => {
    stop()
  })

  return {
    isActive: readonly(isActive),
    start,
    stop,
    restart
  }
}
```

使用範例：

```vue
<script setup lang="ts">
const count = ref(0)

const { isActive, start, stop, restart } = useInterval(() => {
  count.value++
}, 1000)

// 自動開始
onMounted(() => {
  start()
})
</script>

<template>
  <div>
    <p>計數：{{ count }}</p>
    <p>狀態：{{ isActive ? '運行中' : '已停止' }}</p>
    <button @click="start">開始</button>
    <button @click="stop">停止</button>
    <button @click="restart">重新開始</button>
  </div>
</template>
```

## 6. 實際應用範例

### 完整的表單驗證 Composable

```typescript
// composables/useForm.ts
export const useForm = <T extends Record<string, any>>(
  initialValues: T,
  validationRules?: Partial<Record<keyof T, (value: any) => string | null>>
) => {
  const values = ref<T>({ ...initialValues })
  const errors = ref<Partial<Record<keyof T, string>>>({})
  const touched = ref<Partial<Record<keyof T, boolean>>>({})
  const isSubmitting = ref(false)

  const validate = (field?: keyof T): boolean => {
    if (!validationRules) return true

    const fieldsToValidate = field ? [field] : Object.keys(validationRules) as (keyof T)[]
    let isValid = true

    for (const fieldName of fieldsToValidate) {
      const rule = validationRules[fieldName]
      if (rule) {
        const error = rule(values.value[fieldName])
        if (error) {
          errors.value[fieldName] = error
          isValid = false
        } else {
          delete errors.value[fieldName]
        }
      }
    }

    return isValid
  }

  const setFieldValue = (field: keyof T, value: any) => {
    values.value[field] = value
    touched.value[field] = true
    validate(field)
  }

  const setFieldTouched = (field: keyof T, isTouched = true) => {
    touched.value[field] = isTouched
    if (isTouched) {
      validate(field)
    }
  }

  const handleSubmit = async (onSubmit: (values: T) => Promise<void> | void) => {
    // 標記所有欄位為已觸碰
    Object.keys(values.value).forEach(key => {
      touched.value[key as keyof T] = true
    })

    // 驗證所有欄位
    const isValid = validate()

    if (!isValid) {
      return
    }

    isSubmitting.value = true
    try {
      await onSubmit(values.value)
    } finally {
      isSubmitting.value = false
    }
  }

  const reset = () => {
    values.value = { ...initialValues }
    errors.value = {}
    touched.value = {}
    isSubmitting.value = false
  }

  const isValid = computed(() => {
    return Object.keys(errors.value).length === 0
  })

  return {
    values,
    errors: readonly(errors),
    touched: readonly(touched),
    isSubmitting: readonly(isSubmitting),
    isValid,
    setFieldValue,
    setFieldTouched,
    handleSubmit,
    validate,
    reset
  }
}
```

### 使用表單驗證

```vue
<template>
  <form @submit.prevent="handleSubmit(onSubmit)" class="register-form">
    <div class="form-group">
      <label for="email">電子郵件</label>
      <input
        id="email"
        v-model="values.email"
        @blur="setFieldTouched('email')"
        type="email"
        :class="{ 'error': touched.email && errors.email }"
      />
      <span v-if="touched.email && errors.email" class="error-message">
        {{ errors.email }}
      </span>
    </div>

    <div class="form-group">
      <label for="password">密碼</label>
      <input
        id="password"
        v-model="values.password"
        @blur="setFieldTouched('password')"
        type="password"
        :class="{ 'error': touched.password && errors.password }"
      />
      <span v-if="touched.password && errors.password" class="error-message">
        {{ errors.password }}
      </span>
    </div>

    <div class="form-group">
      <label for="confirmPassword">確認密碼</label>
      <input
        id="confirmPassword"
        v-model="values.confirmPassword"
        @blur="setFieldTouched('confirmPassword')"
        type="password"
        :class="{ 'error': touched.confirmPassword && errors.confirmPassword }"
      />
      <span v-if="touched.confirmPassword && errors.confirmPassword" class="error-message">
        {{ errors.confirmPassword }}
      </span>
    </div>

    <button type="submit" :disabled="!isValid || isSubmitting">
      {{ isSubmitting ? '註冊中...' : '註冊' }}
    </button>
  </form>
</template>

<script setup lang="ts">
interface RegisterForm {
  email: string
  password: string
  confirmPassword: string
}

const {
  values,
  errors,
  touched,
  isSubmitting,
  isValid,
  setFieldTouched,
  handleSubmit,
  reset
} = useForm<RegisterForm>(
  {
    email: '',
    password: '',
    confirmPassword: ''
  },
  {
    email: (value) => {
      if (!value) return '請輸入電子郵件'
      if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(value)) return '電子郵件格式不正確'
      return null
    },
    password: (value) => {
      if (!value) return '請輸入密碼'
      if (value.length < 8) return '密碼長度至少需要 8 個字元'
      if (!/(?=.*[a-z])(?=.*[A-Z])(?=.*\d)/.test(value)) {
        return '密碼需包含大小寫字母和數字'
      }
      return null
    },
    confirmPassword: (value) => {
      if (!value) return '請確認密碼'
      if (value !== values.value.password) return '密碼不一致'
      return null
    }
  }
)

const onSubmit = async (formValues: RegisterForm) => {
  try {
    await $fetch('/api/auth/register', {
      method: 'POST',
      body: formValues
    })
    alert('註冊成功！')
    reset()
  } catch (error) {
    alert('註冊失敗，請稍後再試')
  }
}
</script>

<style scoped>
.register-form {
  max-width: 400px;
  margin: 0 auto;
}

.form-group {
  margin-bottom: 20px;
}

label {
  display: block;
  margin-bottom: 5px;
  font-weight: bold;
}

input {
  width: 100%;
  padding: 10px;
  border: 1px solid #ddd;
  border-radius: 4px;
  font-size: 14px;
}

input.error {
  border-color: #e74c3c;
}

.error-message {
  color: #e74c3c;
  font-size: 12px;
  margin-top: 5px;
  display: block;
}

button {
  width: 100%;
  padding: 12px;
  background-color: #3498db;
  color: white;
  border: none;
  border-radius: 4px;
  font-size: 16px;
  cursor: pointer;
}

button:disabled {
  background-color: #bdc3c7;
  cursor: not-allowed;
}
</style>
```

## 最佳實踐建議

### 1. 命名規範

✅ **使用 `use` 前綴**
```typescript
// ✅ 好
export const useCounter = () => {}
export const useAuth = () => {}

// ❌ 不好
export const counter = () => {}
export const getAuth = () => {}
```

### 2. 返回只讀狀態

```typescript
// ✅ 好 - 保護內部狀態
export const useCounter = () => {
  const count = ref(0)
  const increment = () => count.value++

  return {
    count: readonly(count), // 只讀
    increment
  }
}

// ❌ 不好 - 暴露可變狀態
export const useCounter = () => {
  const count = ref(0)
  return { count } // 外部可以直接修改
}
```

### 3. TypeScript 類型定義

```typescript
export const useApi = <T>() => {
  const data = ref<T | null>(null)
  const error = ref<Error | null>(null)

  return {
    data: readonly(data) as Readonly<Ref<T | null>>,
    error: readonly(error) as Readonly<Ref<Error | null>>
  }
}
```

### 4. 清理副作用

```typescript
export const useEventListener = (
  target: EventTarget,
  event: string,
  handler: EventListener
) => {
  onMounted(() => {
    target.addEventListener(event, handler)
  })

  onUnmounted(() => {
    target.removeEventListener(event, handler)
  })
}
```

### 5. 組合多個 Composables

```typescript
export const useAuthenticatedUser = () => {
  const { user, isAuthenticated } = useAppState()
  const { data: profile, refresh } = useFetch(() => `/api/users/${user.value?.id}`, {
    immediate: false
  })

  watch(isAuthenticated, (authenticated) => {
    if (authenticated) {
      refresh()
    }
  })

  return {
    user,
    profile,
    isAuthenticated
  }
}
```

## 總結

Composables 是 Nuxt 3 中重用邏輯的核心機制：

✅ **自動導入** - 提高開發效率
✅ **類型安全** - 完整的 TypeScript 支援
✅ **狀態共享** - 使用 `useState` 跨組件共享
✅ **生命週期** - 完整的生命週期管理
✅ **可組合** - 可以組合多個 composables

掌握 Composables 將大大提升你的 Nuxt 3 開發效率和代碼品質！
