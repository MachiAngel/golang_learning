# 14. Middleware 中介軟體系統

## 目錄
- [Middleware 概念與用途](#middleware-概念與用途)
- [路由中介軟體（Route Middleware）](#路由中介軟體route-middleware)
- [全域中介軟體（Global Middleware）](#全域中介軟體global-middleware)
- [middleware/ 目錄結構](#middleware-目錄結構)
- [命名中介軟體](#命名中介軟體)
- [匿名中介軟體](#匿名中介軟體)
- [中介軟體執行順序](#中介軟體執行順序)
- [導航守衛](#導航守衛)
- [身份驗證範例](#身份驗證範例)
- [權限控制範例](#權限控制範例)
- [SSR 中介軟體注意事項](#ssr-中介軟體注意事項)

---

## Middleware 概念與用途

Middleware（中介軟體）是在路由導航前執行的函式，用於：

### 主要用途
1. **身份驗證** - 檢查用戶是否登入
2. **權限控制** - 驗證用戶權限
3. **路由重定向** - 根據條件導航到不同頁面
4. **數據預取** - 在進入頁面前載入必要數據
5. **日誌記錄** - 追蹤用戶行為
6. **狀態檢查** - 驗證應用狀態

### 執行時機
- 在頁面組件渲染之前執行
- 在伺服器端渲染（SSR）和客戶端導航時都會執行
- 可以阻止或重定向路由導航

---

## 路由中介軟體（Route Middleware）

路由中介軟體只在特定頁面上執行，需要在頁面組件中明確指定。

### 基本使用

**頁面組件中定義：**

```vue
<!-- pages/dashboard.vue -->
<script setup>
// 方式一：使用 definePageMeta（推薦）
definePageMeta({
  middleware: 'auth'
})
</script>

<template>
  <div>
    <h1>儀表板</h1>
    <p>只有登入用戶才能看到這個頁面</p>
  </div>
</template>
```

### 多個中介軟體

```vue
<!-- pages/admin/users.vue -->
<script setup>
definePageMeta({
  middleware: ['auth', 'admin', 'verify-permissions']
})
</script>

<template>
  <div>
    <h1>用戶管理</h1>
    <p>需要管理員權限</p>
  </div>
</template>
```

---

## 全域中介軟體（Global Middleware）

全域中介軟體會在每次路由變更時自動執行，無需在頁面中指定。

### 創建方式

使用 `.global.ts` 或 `.global.js` 後綴命名：

```typescript
// middleware/analytics.global.ts
export default defineNuxtRouteMiddleware((to, from) => {
  // 每次路由變更都會執行
  console.log('導航至:', to.path)
  console.log('來自:', from.path)

  // 發送分析數據
  if (process.client) {
    // 追蹤頁面瀏覽
    window.gtag?.('event', 'page_view', {
      page_path: to.path
    })
  }
})
```

---

## middleware/ 目錄結構

### 推薦的目錄結構

```
middleware/
├── auth.ts              # 身份驗證
├── guest.ts             # 訪客檢查
├── admin.ts             # 管理員權限
├── role.ts              # 角色權限
├── analytics.global.ts  # 全域分析追蹤
├── logger.global.ts     # 全域日誌記錄
└── redirect.ts          # 重定向邏輯
```

### 實際範例

**middleware/auth.ts**

```typescript
export default defineNuxtRouteMiddleware((to, from) => {
  const user = useState('user')

  // 檢查用戶是否登入
  if (!user.value) {
    // 儲存原始目標路徑
    const redirect = to.fullPath

    // 重定向到登入頁
    return navigateTo({
      path: '/login',
      query: { redirect }
    })
  }
})
```

**middleware/guest.ts**

```typescript
export default defineNuxtRouteMiddleware((to, from) => {
  const user = useState('user')

  // 如果已登入，重定向到首頁
  if (user.value) {
    return navigateTo('/')
  }
})
```

**middleware/admin.ts**

```typescript
export default defineNuxtRouteMiddleware((to, from) => {
  const user = useState('user')

  // 檢查是否為管理員
  if (!user.value || user.value.role !== 'admin') {
    return abortNavigation({
      statusCode: 403,
      statusMessage: '需要管理員權限'
    })
  }
})
```

---

## 命名中介軟體

命名中介軟體定義在 `middleware/` 目錄中，可在頁面中通過名稱引用。

### 創建命名中介軟體

```typescript
// middleware/verify-email.ts
export default defineNuxtRouteMiddleware((to, from) => {
  const user = useState('user')

  if (user.value && !user.value.emailVerified) {
    return navigateTo('/verify-email')
  }
})
```

### 在頁面中使用

```vue
<!-- pages/profile.vue -->
<script setup>
definePageMeta({
  middleware: 'verify-email'
})
</script>

<template>
  <div>
    <h1>個人資料</h1>
    <p>需要驗證郵箱</p>
  </div>
</template>
```

---

## 匿名中介軟體

匿名中介軟體直接在頁面組件中定義，不需要單獨的文件。

### 基本語法

```vue
<!-- pages/special.vue -->
<script setup>
definePageMeta({
  middleware: [
    // 匿名中介軟體
    function (to, from) {
      const config = useState('config')

      if (!config.value?.specialFeatureEnabled) {
        return navigateTo('/404')
      }
    },
    // 也可以使用箭頭函式
    (to, from) => {
      console.log('訪問特殊頁面')
    }
  ]
})
</script>

<template>
  <div>
    <h1>特殊功能頁面</h1>
  </div>
</template>
```

### 混合使用

```vue
<!-- pages/secure.vue -->
<script setup>
definePageMeta({
  middleware: [
    'auth',  // 命名中介軟體
    (to, from) => {  // 匿名中介軟體
      // 額外的頁面特定邏輯
      const lastVisit = localStorage.getItem('lastVisit')
      if (lastVisit) {
        console.log('上次訪問時間:', lastVisit)
      }
      localStorage.setItem('lastVisit', new Date().toISOString())
    }
  ]
})
</script>

<template>
  <div>
    <h1>安全頁面</h1>
  </div>
</template>
```

---

## 中介軟體執行順序

### 執行順序規則

1. **全域中介軟體** - 按檔案名稱字母順序執行
2. **頁面定義的中介軟體** - 按陣列定義順序執行

### 範例說明

```
middleware/
├── 01.analytics.global.ts  # 1. 首先執行
├── 02.logger.global.ts     # 2. 第二執行
└── auth.ts                 # 3. 如果頁面使用，在全域後執行
```

```vue
<!-- pages/admin.vue -->
<script setup>
definePageMeta({
  middleware: [
    'auth',      // 3. 全域中介軟體後執行
    'admin',     // 4. auth 之後執行
    (to, from) => {  // 5. 最後執行
      console.log('頁面特定檢查')
    }
  ]
})
</script>
```

### 控制執行流程

```typescript
// middleware/logger.global.ts
export default defineNuxtRouteMiddleware((to, from) => {
  console.log('[1] Logger Global Middleware')

  // 返回值會影響後續執行
  // return navigateTo('/') - 停止並重定向
  // return abortNavigation() - 停止導航
  // return - 繼續執行下一個中介軟體
})
```

---

## 導航守衛

### 可用的導航控制方法

#### 1. navigateTo - 重定向

```typescript
// middleware/subscription.ts
export default defineNuxtRouteMiddleware((to, from) => {
  const user = useState('user')

  if (!user.value?.hasActiveSubscription) {
    // 重定向到訂閱頁面
    return navigateTo('/subscribe')
  }
})
```

#### 2. abortNavigation - 中止導航

```typescript
// middleware/maintenance.ts
export default defineNuxtRouteMiddleware((to, from) => {
  const config = useState('config')

  if (config.value?.maintenanceMode && to.path !== '/maintenance') {
    return abortNavigation({
      statusCode: 503,
      statusMessage: '系統維護中'
    })
  }
})
```

#### 3. 返回錯誤

```typescript
// middleware/permission.ts
export default defineNuxtRouteMiddleware((to, from) => {
  const user = useState('user')
  const requiredPermission = to.meta.permission

  if (requiredPermission && !user.value?.permissions?.includes(requiredPermission)) {
    // 拋出錯誤
    throw createError({
      statusCode: 403,
      statusMessage: '權限不足',
      fatal: true
    })
  }
})
```

### 條件導航範例

```typescript
// middleware/redirect.ts
export default defineNuxtRouteMiddleware((to, from) => {
  const user = useState('user')

  // 多條件判斷
  if (!user.value) {
    return navigateTo('/login')
  }

  if (!user.value.emailVerified) {
    return navigateTo('/verify-email')
  }

  if (user.value.needsPasswordChange) {
    return navigateTo('/change-password')
  }

  // 所有條件通過，繼續導航
})
```

---

## 身份驗證範例

### 完整的身份驗證系統

**1. 身份驗證狀態管理**

```typescript
// composables/useAuth.ts
export const useAuth = () => {
  const user = useState('user', () => null)
  const token = useCookie('auth_token')

  const login = async (credentials: { email: string; password: string }) => {
    try {
      const { data } = await $fetch('/api/auth/login', {
        method: 'POST',
        body: credentials
      })

      user.value = data.user
      token.value = data.token

      return { success: true }
    } catch (error) {
      return { success: false, error }
    }
  }

  const logout = async () => {
    user.value = null
    token.value = null
    await navigateTo('/login')
  }

  const checkAuth = async () => {
    if (!token.value) return false

    try {
      const { data } = await $fetch('/api/auth/me', {
        headers: {
          Authorization: `Bearer ${token.value}`
        }
      })

      user.value = data.user
      return true
    } catch (error) {
      token.value = null
      return false
    }
  }

  return {
    user: readonly(user),
    login,
    logout,
    checkAuth
  }
}
```

**2. 身份驗證中介軟體**

```typescript
// middleware/auth.ts
export default defineNuxtRouteMiddleware(async (to, from) => {
  const { user, checkAuth } = useAuth()

  // 如果沒有用戶資訊，嘗試從 token 恢復
  if (!user.value) {
    const isAuthenticated = await checkAuth()

    if (!isAuthenticated) {
      // 儲存原始路徑用於登入後重定向
      return navigateTo({
        path: '/login',
        query: { redirect: to.fullPath }
      })
    }
  }

  // 檢查 token 是否過期
  const token = useCookie('auth_token')
  if (token.value) {
    try {
      const payload = JSON.parse(atob(token.value.split('.')[1]))
      const expiry = payload.exp * 1000

      if (Date.now() >= expiry) {
        return navigateTo('/login?expired=true')
      }
    } catch (error) {
      console.error('Token 解析錯誤:', error)
      return navigateTo('/login')
    }
  }
})
```

**3. 訪客中介軟體**

```typescript
// middleware/guest.ts
export default defineNuxtRouteMiddleware((to, from) => {
  const { user } = useAuth()

  // 已登入用戶重定向到首頁
  if (user.value) {
    const redirect = to.query.redirect as string
    return navigateTo(redirect || '/dashboard')
  }
})
```

**4. 登入頁面**

```vue
<!-- pages/login.vue -->
<script setup>
definePageMeta({
  middleware: 'guest',
  layout: 'auth'
})

const route = useRoute()
const { login } = useAuth()

const form = ref({
  email: '',
  password: ''
})

const loading = ref(false)
const error = ref('')

const handleLogin = async () => {
  loading.value = true
  error.value = ''

  const result = await login(form.value)

  if (result.success) {
    const redirect = route.query.redirect as string
    await navigateTo(redirect || '/dashboard')
  } else {
    error.value = '登入失敗，請檢查您的帳號密碼'
  }

  loading.value = false
}
</script>

<template>
  <div class="login-page">
    <h1>登入</h1>

    <div v-if="route.query.expired" class="alert alert-warning">
      您的登入已過期，請重新登入
    </div>

    <form @submit.prevent="handleLogin">
      <div class="form-group">
        <label>電子郵件</label>
        <input
          v-model="form.email"
          type="email"
          required
        />
      </div>

      <div class="form-group">
        <label>密碼</label>
        <input
          v-model="form.password"
          type="password"
          required
        />
      </div>

      <div v-if="error" class="error">{{ error }}</div>

      <button type="submit" :disabled="loading">
        {{ loading ? '登入中...' : '登入' }}
      </button>
    </form>
  </div>
</template>
```

**5. 受保護的頁面**

```vue
<!-- pages/dashboard.vue -->
<script setup>
definePageMeta({
  middleware: 'auth'
})

const { user, logout } = useAuth()
</script>

<template>
  <div>
    <h1>歡迎，{{ user?.name }}</h1>
    <p>Email: {{ user?.email }}</p>
    <button @click="logout">登出</button>
  </div>
</template>
```

---

## 權限控制範例

### 基於角色的存取控制（RBAC）

**1. 角色定義**

```typescript
// types/auth.ts
export enum Role {
  GUEST = 'guest',
  USER = 'user',
  MODERATOR = 'moderator',
  ADMIN = 'admin',
  SUPER_ADMIN = 'super_admin'
}

export interface User {
  id: string
  name: string
  email: string
  role: Role
  permissions: string[]
}
```

**2. 角色階層檢查**

```typescript
// utils/permissions.ts
const roleHierarchy = {
  [Role.SUPER_ADMIN]: 5,
  [Role.ADMIN]: 4,
  [Role.MODERATOR]: 3,
  [Role.USER]: 2,
  [Role.GUEST]: 1
}

export const hasRoleLevel = (userRole: Role, requiredRole: Role): boolean => {
  return roleHierarchy[userRole] >= roleHierarchy[requiredRole]
}

export const hasPermission = (user: User, permission: string): boolean => {
  return user.permissions.includes(permission)
}

export const hasAnyPermission = (user: User, permissions: string[]): boolean => {
  return permissions.some(p => user.permissions.includes(p))
}

export const hasAllPermissions = (user: User, permissions: string[]): boolean => {
  return permissions.every(p => user.permissions.includes(p))
}
```

**3. 角色中介軟體**

```typescript
// middleware/role.ts
export default defineNuxtRouteMiddleware((to, from) => {
  const user = useState<User>('user')
  const requiredRole = to.meta.role as Role

  if (!requiredRole) return

  if (!user.value || !hasRoleLevel(user.value.role, requiredRole)) {
    throw createError({
      statusCode: 403,
      statusMessage: `需要 ${requiredRole} 或更高權限`
    })
  }
})
```

**4. 權限中介軟體**

```typescript
// middleware/permission.ts
export default defineNuxtRouteMiddleware((to, from) => {
  const user = useState<User>('user')
  const requiredPermissions = to.meta.permissions as string[]
  const requireAll = to.meta.requireAllPermissions as boolean

  if (!requiredPermissions || requiredPermissions.length === 0) return

  if (!user.value) {
    return navigateTo('/login')
  }

  const hasAccess = requireAll
    ? hasAllPermissions(user.value, requiredPermissions)
    : hasAnyPermission(user.value, requiredPermissions)

  if (!hasAccess) {
    throw createError({
      statusCode: 403,
      statusMessage: '您沒有足夠的權限訪問此頁面'
    })
  }
})
```

**5. 使用權限控制**

```vue
<!-- pages/admin/users.vue -->
<script setup>
import { Role } from '~/types/auth'

definePageMeta({
  middleware: ['auth', 'role'],
  role: Role.ADMIN
})
</script>

<template>
  <div>
    <h1>用戶管理</h1>
    <p>只有管理員可以訪問</p>
  </div>
</template>
```

```vue
<!-- pages/admin/settings.vue -->
<script setup>
definePageMeta({
  middleware: ['auth', 'permission'],
  permissions: ['settings.view', 'settings.edit'],
  requireAllPermissions: true
})
</script>

<template>
  <div>
    <h1>系統設定</h1>
    <p>需要特定權限</p>
  </div>
</template>
```

**6. 動態權限檢查**

```typescript
// middleware/dynamic-permission.ts
export default defineNuxtRouteMiddleware(async (to, from) => {
  const user = useState<User>('user')

  // 從路由參數獲取資源 ID
  const resourceId = to.params.id

  if (resourceId) {
    try {
      // 檢查用戶是否有權訪問此資源
      const { data } = await $fetch(`/api/resources/${resourceId}/check-access`, {
        headers: {
          Authorization: `Bearer ${useCookie('auth_token').value}`
        }
      })

      if (!data.hasAccess) {
        throw createError({
          statusCode: 403,
          statusMessage: '您無權訪問此資源'
        })
      }
    } catch (error) {
      throw createError({
        statusCode: 403,
        statusMessage: '訪問檢查失敗'
      })
    }
  }
})
```

---

## SSR 中介軟體注意事項

### 伺服器端與客戶端差異

#### 1. 環境檢測

```typescript
// middleware/analytics.global.ts
export default defineNuxtRouteMiddleware((to, from) => {
  // 僅在客戶端執行
  if (process.client) {
    console.log('客戶端導航:', to.path)
    // 使用 window 對象
    window.gtag?.('event', 'page_view')
  }

  // 僅在伺服器端執行
  if (process.server) {
    console.log('伺服器端渲染:', to.path)
    // 可以訪問 Node.js API
  }
})
```

#### 2. Cookie 處理

```typescript
// middleware/locale.global.ts
export default defineNuxtRouteMiddleware((to, from) => {
  // useCookie 在 SSR 和客戶端都可用
  const locale = useCookie('locale')

  if (!locale.value) {
    // 伺服器端可以從請求標頭獲取
    if (process.server) {
      const headers = useRequestHeaders(['accept-language'])
      const preferredLocale = headers['accept-language']?.split(',')[0]
      locale.value = preferredLocale || 'zh-TW'
    } else {
      // 客戶端使用瀏覽器語言
      locale.value = navigator.language || 'zh-TW'
    }
  }
})
```

#### 3. 狀態同步

```typescript
// middleware/theme.global.ts
export default defineNuxtRouteMiddleware((to, from) => {
  const theme = useState('theme', () => 'light')
  const themeCookie = useCookie('theme')

  // 伺服器端從 Cookie 讀取
  if (process.server && themeCookie.value) {
    theme.value = themeCookie.value
  }

  // 客戶端從 localStorage 讀取
  if (process.client && !theme.value) {
    const savedTheme = localStorage.getItem('theme')
    if (savedTheme) {
      theme.value = savedTheme
      themeCookie.value = savedTheme
    }
  }
})
```

### API 請求注意事項

#### 1. 正確的請求方式

```typescript
// middleware/config.global.ts
export default defineNuxtRouteMiddleware(async (to, from) => {
  const config = useState('appConfig')

  if (!config.value) {
    try {
      // 使用 $fetch，自動處理 SSR
      const data = await $fetch('/api/config')
      config.value = data
    } catch (error) {
      console.error('載入配置失敗:', error)
    }
  }
})
```

#### 2. 避免重複請求

```typescript
// middleware/user-data.ts
let fetchPromise: Promise<any> | null = null

export default defineNuxtRouteMiddleware(async (to, from) => {
  const user = useState('user')
  const token = useCookie('auth_token')

  if (token.value && !user.value) {
    // 避免多次請求
    if (!fetchPromise) {
      fetchPromise = $fetch('/api/auth/me', {
        headers: {
          Authorization: `Bearer ${token.value}`
        }
      }).finally(() => {
        fetchPromise = null
      })
    }

    try {
      const data = await fetchPromise
      user.value = data.user
    } catch (error) {
      console.error('獲取用戶資料失敗:', error)
    }
  }
})
```

### 重定向處理

#### 1. 絕對 URL vs 相對路徑

```typescript
// middleware/redirect-handling.ts
export default defineNuxtRouteMiddleware((to, from) => {
  const isAuthenticated = useState('isAuthenticated')

  if (!isAuthenticated.value) {
    // ✅ 推薦：使用相對路徑
    return navigateTo('/login')

    // ❌ 避免：在 SSR 時使用絕對 URL 可能導致問題
    // return navigateTo('https://example.com/login')
  }
})
```

#### 2. 外部重定向

```typescript
// middleware/external-redirect.ts
export default defineNuxtRouteMiddleware((to, from) => {
  const needsExternalAuth = to.query.external === 'true'

  if (needsExternalAuth) {
    // 使用 navigateTo 的 external 選項
    return navigateTo('https://auth.example.com/login', {
      external: true
    })
  }
})
```

### 完整的 SSR 安全範例

```typescript
// middleware/ssr-safe.global.ts
export default defineNuxtRouteMiddleware(async (to, from) => {
  // 1. 使用 useState 確保狀態在 SSR 和客戶端共享
  const appReady = useState('appReady', () => false)

  // 2. 僅在首次載入時初始化
  if (!appReady.value) {
    try {
      // 3. 使用 $fetch 進行 API 請求
      const config = await $fetch('/api/init')

      // 4. 更新共享狀態
      useState('config').value = config

      // 5. 標記為已就緒
      appReady.value = true
    } catch (error) {
      console.error('初始化失敗:', error)
    }
  }

  // 6. 客戶端特定邏輯
  if (process.client) {
    // 檢查瀏覽器功能
    const hasLocalStorage = typeof localStorage !== 'undefined'
    if (hasLocalStorage) {
      localStorage.setItem('lastVisit', Date.now().toString())
    }
  }

  // 7. 伺服器端特定邏輯
  if (process.server) {
    // 記錄伺服器日誌
    console.log(`SSR 渲染: ${to.path}`)
  }
})
```

### 最佳實踐總結

1. **使用 `process.client` 和 `process.server` 區分環境**
2. **使用 `useState` 在伺服器和客戶端共享狀態**
3. **使用 `useCookie` 處理 Cookie，自動同步**
4. **使用 `$fetch` 進行 API 請求，支援 SSR**
5. **避免在 SSR 中使用瀏覽器專用 API（如 `window`、`localStorage`）**
6. **重定向使用相對路徑而非絕對 URL**
7. **注意異步操作的時機和順序**
8. **適當處理錯誤，避免 SSR 失敗**

---

## 完整實戰範例：電商網站中介軟體

```typescript
// middleware/01.init.global.ts
export default defineNuxtRouteMiddleware(async (to, from) => {
  const initialized = useState('initialized', () => false)

  if (!initialized.value) {
    // 初始化應用配置
    const config = await $fetch('/api/config')
    useState('config').value = config
    initialized.value = true
  }
})
```

```typescript
// middleware/02.auth.global.ts
export default defineNuxtRouteMiddleware(async (to, from) => {
  const user = useState('user')
  const token = useCookie('auth_token')

  if (token.value && !user.value) {
    try {
      const data = await $fetch('/api/auth/me', {
        headers: { Authorization: `Bearer ${token.value}` }
      })
      user.value = data.user
    } catch {
      token.value = null
    }
  }
})
```

```typescript
// middleware/cart.ts
export default defineNuxtRouteMiddleware((to, from) => {
  const cart = useState('cart', () => ({ items: [] }))

  // 從 localStorage 恢復購物車
  if (process.client && cart.value.items.length === 0) {
    const savedCart = localStorage.getItem('cart')
    if (savedCart) {
      cart.value = JSON.parse(savedCart)
    }
  }
})
```

```typescript
// middleware/checkout-guard.ts
export default defineNuxtRouteMiddleware((to, from) => {
  const cart = useState('cart')
  const user = useState('user')

  // 結帳頁面保護
  if (to.path === '/checkout') {
    if (!user.value) {
      return navigateTo('/login?redirect=/checkout')
    }

    if (!cart.value?.items?.length) {
      return navigateTo('/cart')
    }
  }
})
```

這個完整的中介軟體系統為您的 Nuxt.js 應用提供了強大的路由保護和邏輯處理能力！
