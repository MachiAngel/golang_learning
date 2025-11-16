# 09 - 資料獲取 - useFetch 與 useAsyncData

## 概述

Nuxt 3 提供了多種資料獲取方式，其中 `useFetch` 和 `useAsyncData` 是最常用的兩個 composables。它們提供了 SSR 支援、自動快取、錯誤處理等強大功能。

## 1. useFetch 詳解

`useFetch` 是最簡單和最常用的資料獲取方式，它是 `useAsyncData` 和 `$fetch` 的組合。

### 基本使用

```vue
<template>
  <div>
    <div v-if="pending">載入中...</div>
    <div v-else-if="error">錯誤: {{ error.message }}</div>
    <div v-else>
      <h1>使用者列表</h1>
      <ul>
        <li v-for="user in data" :key="user.id">
          {{ user.name }} ({{ user.email }})
        </li>
      </ul>
    </div>
  </div>
</template>

<script setup lang="ts">
interface User {
  id: number
  name: string
  email: string
}

const { data, pending, error, refresh } = await useFetch<User[]>('/api/users')
</script>
```

### 返回值說明

```typescript
const {
  data,        // 響應數據
  pending,     // 載入狀態
  error,       // 錯誤對象
  refresh,     // 重新獲取函數
  execute,     // 手動執行（當 immediate: false 時）
  status       // 請求狀態: 'idle' | 'pending' | 'success' | 'error'
} = await useFetch('/api/endpoint')
```

### 帶參數的請求

```vue
<script setup lang="ts">
// GET 請求帶查詢參數
const { data: posts } = await useFetch('/api/posts', {
  query: {
    page: 1,
    limit: 10,
    category: 'tech'
  }
})
// 請求 URL: /api/posts?page=1&limit=10&category=tech

// POST 請求
const { data: newPost } = await useFetch('/api/posts', {
  method: 'POST',
  body: {
    title: '新文章',
    content: '文章內容...'
  }
})

// 帶 headers
const { data: protectedData } = await useFetch('/api/protected', {
  headers: {
    Authorization: `Bearer ${token.value}`
  }
})
</script>
```

### 動態 URL 和參數

```vue
<script setup lang="ts">
const userId = ref(1)
const includeDetails = ref(false)

// 使用函數返回動態 URL
const { data: user } = await useFetch(() => `/api/users/${userId.value}`, {
  // 當 userId 變化時重新獲取
  watch: [userId]
})

// 動態查詢參數
const { data: posts } = await useFetch('/api/posts', {
  query: {
    userId: userId,
    details: includeDetails
  },
  watch: [userId, includeDetails]
})
</script>

<template>
  <div>
    <input v-model.number="userId" type="number" placeholder="User ID" />
    <label>
      <input v-model="includeDetails" type="checkbox" />
      包含詳細資訊
    </label>

    <div v-if="user">
      <h2>{{ user.name }}</h2>
      <p>{{ user.email }}</p>
    </div>
  </div>
</template>
```

### 延遲執行（Lazy）

```vue
<script setup lang="ts">
// 不會阻塞導航，在背景獲取資料
const { data, pending } = await useFetch('/api/posts', {
  lazy: true
})

// 或使用 useLazyFetch（useFetch 的別名，lazy: true）
const { data: users, pending: usersPending } = await useLazyFetch('/api/users')
</script>

<template>
  <div>
    <!-- 頁面會立即渲染，不等待資料 -->
    <div v-if="pending">載入中...</div>
    <div v-else-if="data">
      <!-- 資料內容 -->
    </div>
  </div>
</template>
```

### 立即執行控制

```vue
<script setup lang="ts">
// 不自動執行，手動控制
const { data, execute, pending } = await useFetch('/api/data', {
  immediate: false
})

const loadData = async () => {
  await execute()
}
</script>

<template>
  <div>
    <button @click="loadData" :disabled="pending">
      {{ pending ? '載入中...' : '載入資料' }}
    </button>

    <div v-if="data">
      {{ data }}
    </div>
  </div>
</template>
```

## 2. useAsyncData 詳解

`useAsyncData` 提供更靈活的控制，適合複雜的資料獲取邏輯。

### 基本使用

```vue
<script setup lang="ts">
const { data, pending, error } = await useAsyncData('users', () => {
  return $fetch('/api/users')
})

// 第一個參數是唯一的 key，用於快取和去重
// 第二個參數是獲取資料的函數
</script>
```

### 複雜的資料處理

```vue
<script setup lang="ts">
interface User {
  id: number
  name: string
  posts: Post[]
}

interface Post {
  id: number
  title: string
}

const userId = ref(1)

const { data: userData } = await useAsyncData(
  `user-${userId.value}`,  // 動態 key
  async () => {
    // 可以執行多個請求
    const [user, posts, comments] = await Promise.all([
      $fetch<User>(`/api/users/${userId.value}`),
      $fetch<Post[]>(`/api/users/${userId.value}/posts`),
      $fetch(`/api/users/${userId.value}/comments`)
    ])

    // 處理和組合資料
    return {
      ...user,
      posts,
      comments,
      totalActivity: posts.length + comments.length
    }
  },
  {
    watch: [userId]
  }
)
</script>
```

### Transform 轉換資料

```vue
<script setup lang="ts">
interface ApiResponse {
  data: User[]
  meta: {
    total: number
    page: number
  }
}

const { data: users } = await useFetch<ApiResponse>('/api/users', {
  // 只返回需要的資料
  transform: (response) => response.data
})

// users.value 直接是 User[] 陣列，而非完整的 ApiResponse

// 更複雜的轉換
const { data: formattedUsers } = await useFetch('/api/users', {
  transform: (response: ApiResponse) => {
    return response.data.map(user => ({
      ...user,
      displayName: `${user.firstName} ${user.lastName}`,
      isActive: user.status === 'active'
    }))
  }
})
</script>
```

### Pick 選擇特定欄位

```vue
<script setup lang="ts">
const { data: userNames } = await useFetch('/api/users', {
  // 只選擇 name 欄位，減少傳輸資料量
  pick: ['name']
})

// 選擇多個欄位
const { data: userInfo } = await useFetch('/api/users', {
  pick: ['id', 'name', 'email']
})
</script>
```

## 3. $fetch 的使用

`$fetch` 是 Nuxt 對 `ofetch` 的封裝，可在任何地方使用（不僅限於 setup）。

### 基本用法

```typescript
// GET 請求
const users = await $fetch('/api/users')

// POST 請求
const newUser = await $fetch('/api/users', {
  method: 'POST',
  body: {
    name: 'John Doe',
    email: 'john@example.com'
  }
})

// PUT 請求
const updatedUser = await $fetch(`/api/users/${id}`, {
  method: 'PUT',
  body: userData
})

// DELETE 請求
await $fetch(`/api/users/${id}`, {
  method: 'DELETE'
})
```

### 在組件方法中使用

```vue
<script setup lang="ts">
const users = ref<User[]>([])
const isLoading = ref(false)
const error = ref<string | null>(null)

const loadUsers = async () => {
  isLoading.value = true
  error.value = null

  try {
    users.value = await $fetch<User[]>('/api/users')
  } catch (e: any) {
    error.value = e.message
  } finally {
    isLoading.value = false
  }
}

const createUser = async (userData: CreateUserDTO) => {
  try {
    const newUser = await $fetch<User>('/api/users', {
      method: 'POST',
      body: userData
    })

    users.value.push(newUser)
    return { success: true, user: newUser }
  } catch (e: any) {
    return { success: false, error: e.message }
  }
}

const deleteUser = async (userId: number) => {
  try {
    await $fetch(`/api/users/${userId}`, {
      method: 'DELETE'
    })

    users.value = users.value.filter(u => u.id !== userId)
    return { success: true }
  } catch (e: any) {
    return { success: false, error: e.message }
  }
}
</script>
```

### 攔截器（Interceptors）

```typescript
// plugins/api.ts
export default defineNuxtPlugin(() => {
  const config = useRuntimeConfig()

  // 創建自定義 $fetch 實例
  const api = $fetch.create({
    baseURL: config.public.apiBase,

    // 請求攔截器
    onRequest({ options }) {
      // 添加認證 token
      const token = useCookie('auth-token')
      if (token.value) {
        options.headers = {
          ...options.headers,
          Authorization: `Bearer ${token.value}`
        }
      }
    },

    // 響應攔截器
    onResponse({ response }) {
      // 處理響應
      console.log('Response:', response.status)
    },

    // 錯誤攔截器
    onResponseError({ response }) {
      if (response.status === 401) {
        // 未授權，導向登入頁
        navigateTo('/login')
      }

      if (response.status === 500) {
        console.error('伺服器錯誤')
      }
    }
  })

  return {
    provide: {
      api
    }
  }
})
```

使用自定義實例：

```vue
<script setup lang="ts">
const { $api } = useNuxtApp()

const { data: users } = await useAsyncData('users', () => {
  return $api('/users')
})
</script>
```

## 4. 資料快取策略

### 快取控制

```vue
<script setup lang="ts">
// 使用快取（預設）
const { data } = await useFetch('/api/posts', {
  key: 'posts' // 快取 key
})

// 禁用快取
const { data: freshData } = await useFetch('/api/posts', {
  key: 'posts',
  getCachedData: () => null // 不使用快取
})

// 自定義快取策略
const { data: cachedData } = await useFetch('/api/posts', {
  key: 'posts',
  getCachedData: (key) => {
    const data = nuxtApp.payload.data[key] || nuxtApp.static.data[key]
    // 只在資料新鮮時返回快取
    if (data && Date.now() - data.timestamp < 60000) { // 1 分鐘
      return data.value
    }
    return null
  }
})
</script>
```

### 手動快取管理

```vue
<script setup lang="ts">
const nuxtApp = useNuxtApp()

// 清除特定快取
const clearCache = (key: string) => {
  nuxtApp.payload.data[key] = null
}

// 設置快取
const setCache = (key: string, data: any) => {
  nuxtApp.payload.data[key] = data
}

// 使用範例
const { data, refresh } = await useFetch('/api/posts', {
  key: 'posts'
})

const forceRefresh = async () => {
  clearCache('posts')
  await refresh()
}
</script>
```

## 5. 錯誤處理

### 基本錯誤處理

```vue
<script setup lang="ts">
const { data, error, status } = await useFetch('/api/posts')

// 檢查錯誤
watchEffect(() => {
  if (error.value) {
    console.error('載入失敗:', error.value.message)
  }
})
</script>

<template>
  <div>
    <div v-if="status === 'pending'">載入中...</div>
    <div v-else-if="status === 'error'" class="error">
      <h3>載入失敗</h3>
      <p>{{ error?.message }}</p>
      <button @click="refresh">重試</button>
    </div>
    <div v-else-if="data">
      <!-- 顯示資料 -->
    </div>
  </div>
</template>
```

### 詳細錯誤處理

```vue
<script setup lang="ts">
interface ApiError {
  statusCode: number
  message: string
  details?: any
}

const handleError = (error: any) => {
  if (error.statusCode === 404) {
    return '資源不存在'
  } else if (error.statusCode === 401) {
    return '需要登入'
  } else if (error.statusCode === 403) {
    return '沒有權限'
  } else if (error.statusCode >= 500) {
    return '伺服器錯誤，請稍後再試'
  }
  return error.message || '未知錯誤'
}

const { data, error } = await useFetch('/api/posts')

const errorMessage = computed(() => {
  return error.value ? handleError(error.value) : null
})
</script>

<template>
  <div v-if="errorMessage" class="error-banner">
    {{ errorMessage }}
  </div>
</template>
```

### Try-Catch 處理

```vue
<script setup lang="ts">
const posts = ref<Post[]>([])
const error = ref<string | null>(null)

const loadPosts = async () => {
  try {
    posts.value = await $fetch<Post[]>('/api/posts')
  } catch (e: any) {
    if (e.response?.status === 404) {
      error.value = '找不到文章'
    } else {
      error.value = '載入失敗: ' + e.message
    }
  }
}

onMounted(() => {
  loadPosts()
})
</script>
```

## 6. Loading 狀態

### 全域 Loading 指示器

```vue
<!-- components/LoadingIndicator.vue -->
<template>
  <Transition name="fade">
    <div v-if="isLoading" class="loading-overlay">
      <div class="spinner"></div>
      <p>載入中...</p>
    </div>
  </Transition>
</template>

<script setup lang="ts">
const nuxtApp = useNuxtApp()
const isLoading = ref(false)

nuxtApp.hook('page:start', () => {
  isLoading.value = true
})

nuxtApp.hook('page:finish', () => {
  isLoading.value = false
})
</script>

<style scoped>
.loading-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background-color: rgba(0, 0, 0, 0.5);
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  z-index: 9999;
  color: white;
}

.spinner {
  width: 50px;
  height: 50px;
  border: 4px solid rgba(255, 255, 255, 0.3);
  border-top-color: white;
  border-radius: 50%;
  animation: spin 1s linear infinite;
}

@keyframes spin {
  to { transform: rotate(360deg); }
}

.fade-enter-active,
.fade-leave-active {
  transition: opacity 0.3s;
}

.fade-enter-from,
.fade-leave-to {
  opacity: 0;
}
</style>
```

### 局部 Loading 狀態

```vue
<template>
  <div class="data-container">
    <div v-if="pending" class="loading-skeleton">
      <div class="skeleton-item" v-for="n in 5" :key="n"></div>
    </div>

    <div v-else-if="data" class="data-list">
      <div v-for="item in data" :key="item.id" class="data-item">
        {{ item.title }}
      </div>
    </div>

    <button @click="refresh" :disabled="pending" class="refresh-btn">
      <span v-if="pending">載入中...</span>
      <span v-else>🔄 重新整理</span>
    </button>
  </div>
</template>

<script setup lang="ts">
const { data, pending, refresh } = await useLazyFetch('/api/posts')
</script>

<style scoped>
.skeleton-item {
  height: 60px;
  background: linear-gradient(
    90deg,
    #f0f0f0 25%,
    #e0e0e0 50%,
    #f0f0f0 75%
  );
  background-size: 200% 100%;
  animation: loading 1.5s infinite;
  border-radius: 4px;
  margin-bottom: 12px;
}

@keyframes loading {
  0% { background-position: 200% 0; }
  100% { background-position: -200% 0; }
}
</style>
```

## 7. 重新獲取資料

### 自動重新整理

```vue
<script setup lang="ts">
// 每 30 秒自動重新整理
const { data, refresh } = await useFetch('/api/stats', {
  server: false, // 只在客戶端執行
})

const { pause, resume } = useIntervalFn(() => {
  refresh()
}, 30000) // 30 秒

// 頁面可見時才重新整理
const { isVisible } = usePageVisibility()

watch(isVisible, (visible) => {
  if (visible) {
    resume()
    refresh()
  } else {
    pause()
  }
})
</script>
```

### 手動重新整理

```vue
<script setup lang="ts">
const { data, pending, refresh } = await useFetch('/api/posts')

const handleRefresh = async () => {
  await refresh()
  console.log('資料已重新整理')
}

// 重新整理並重置錯誤狀態
const { data, error, refresh, clear } = await useFetch('/api/posts')

const retryLoad = async () => {
  clear() // 清除錯誤
  await refresh()
}
</script>

<template>
  <div>
    <button @click="handleRefresh" :disabled="pending">
      {{ pending ? '重新整理中...' : '重新整理' }}
    </button>

    <button v-if="error" @click="retryLoad">
      重試
    </button>
  </div>
</template>
```

## 8. 實際 API 整合範例

### 完整的部落格系統

```vue
<!-- pages/blog/index.vue -->
<template>
  <div class="blog-page">
    <h1>部落格文章</h1>

    <!-- 搜尋和篩選 -->
    <div class="filters">
      <input
        v-model="searchQuery"
        type="text"
        placeholder="搜尋文章..."
        @input="debouncedSearch"
      />

      <select v-model="selectedCategory">
        <option value="">全部分類</option>
        <option v-for="cat in categories" :key="cat" :value="cat">
          {{ cat }}
        </option>
      </select>

      <select v-model="sortBy">
        <option value="latest">最新</option>
        <option value="popular">最熱門</option>
        <option value="oldest">最舊</option>
      </select>
    </div>

    <!-- Loading 狀態 -->
    <div v-if="pending" class="loading">
      <div class="skeleton" v-for="n in 3" :key="n"></div>
    </div>

    <!-- 錯誤狀態 -->
    <div v-else-if="error" class="error">
      <p>{{ errorMessage }}</p>
      <button @click="refresh">重試</button>
    </div>

    <!-- 文章列表 -->
    <div v-else-if="posts && posts.length > 0" class="posts-grid">
      <article v-for="post in posts" :key="post.id" class="post-card">
        <img :src="post.image" :alt="post.title" />
        <div class="post-content">
          <span class="category">{{ post.category }}</span>
          <h2>
            <NuxtLink :to="`/blog/${post.slug}`">
              {{ post.title }}
            </NuxtLink>
          </h2>
          <p class="excerpt">{{ post.excerpt }}</p>
          <div class="post-meta">
            <span class="author">{{ post.author.name }}</span>
            <span class="date">{{ formatDate(post.createdAt) }}</span>
            <span class="views">👁 {{ post.views }}</span>
          </div>
        </div>
      </article>
    </div>

    <!-- 空狀態 -->
    <div v-else class="empty-state">
      <p>沒有找到文章</p>
    </div>

    <!-- 分頁 -->
    <div v-if="totalPages > 1" class="pagination">
      <button
        @click="currentPage--"
        :disabled="currentPage === 1"
      >
        上一頁
      </button>

      <span>第 {{ currentPage }} 頁，共 {{ totalPages }} 頁</span>

      <button
        @click="currentPage++"
        :disabled="currentPage === totalPages"
      >
        下一頁
      </button>
    </div>
  </div>
</template>

<script setup lang="ts">
interface Post {
  id: number
  slug: string
  title: string
  excerpt: string
  content: string
  image: string
  category: string
  author: {
    name: string
    avatar: string
  }
  views: number
  createdAt: string
}

interface PostsResponse {
  posts: Post[]
  total: number
  page: number
  totalPages: number
}

// 狀態
const searchQuery = ref('')
const selectedCategory = ref('')
const sortBy = ref('latest')
const currentPage = ref(1)
const pageSize = 12

// 獲取分類列表
const { data: categories } = await useFetch<string[]>('/api/categories')

// 獲取文章列表
const {
  data: response,
  pending,
  error,
  refresh
} = await useFetch<PostsResponse>('/api/posts', {
  query: {
    search: searchQuery,
    category: selectedCategory,
    sort: sortBy,
    page: currentPage,
    limit: pageSize
  },
  watch: [selectedCategory, sortBy, currentPage],
  // 防抖處理在下面的 debouncedSearch 中
})

const posts = computed(() => response.value?.posts || [])
const totalPages = computed(() => response.value?.totalPages || 1)

const errorMessage = computed(() => {
  if (!error.value) return ''

  const status = error.value.statusCode
  if (status === 404) return '找不到文章'
  if (status === 500) return '伺服器錯誤，請稍後再試'
  return error.value.message || '載入失敗'
})

// 防抖搜尋
const debouncedSearch = useDebounceFn(() => {
  currentPage.value = 1 // 搜尋時重置頁碼
  refresh()
}, 500)

// 格式化日期
const formatDate = (dateString: string) => {
  const date = new Date(dateString)
  return date.toLocaleDateString('zh-TW', {
    year: 'numeric',
    month: 'long',
    day: 'numeric'
  })
}

// SEO
useHead({
  title: '部落格 - 最新文章',
  meta: [
    {
      name: 'description',
      content: '閱讀最新的技術文章和教學'
    }
  ]
})
</script>

<style scoped>
.blog-page {
  max-width: 1200px;
  margin: 0 auto;
  padding: 24px;
}

.filters {
  display: flex;
  gap: 12px;
  margin-bottom: 24px;
  flex-wrap: wrap;
}

.filters input,
.filters select {
  padding: 10px 16px;
  border: 1px solid #ddd;
  border-radius: 8px;
  font-size: 14px;
}

.filters input {
  flex: 1;
  min-width: 200px;
}

.posts-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(300px, 1fr));
  gap: 24px;
  margin-bottom: 32px;
}

.post-card {
  border: 1px solid #e0e0e0;
  border-radius: 12px;
  overflow: hidden;
  transition: transform 0.2s, box-shadow 0.2s;
}

.post-card:hover {
  transform: translateY(-4px);
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);
}

.post-card img {
  width: 100%;
  height: 200px;
  object-fit: cover;
}

.post-content {
  padding: 16px;
}

.category {
  display: inline-block;
  padding: 4px 12px;
  background-color: #3498db;
  color: white;
  border-radius: 12px;
  font-size: 12px;
  margin-bottom: 8px;
}

.post-card h2 {
  font-size: 1.25rem;
  margin: 8px 0;
}

.post-card h2 a {
  color: #2c3e50;
  text-decoration: none;
}

.post-card h2 a:hover {
  color: #3498db;
}

.excerpt {
  color: #7f8c8d;
  font-size: 0.9rem;
  line-height: 1.5;
  margin-bottom: 12px;
}

.post-meta {
  display: flex;
  gap: 16px;
  font-size: 0.85rem;
  color: #95a5a6;
}

.skeleton {
  height: 400px;
  background: linear-gradient(90deg, #f0f0f0 25%, #e0e0e0 50%, #f0f0f0 75%);
  background-size: 200% 100%;
  animation: loading 1.5s infinite;
  border-radius: 12px;
  margin-bottom: 24px;
}

@keyframes loading {
  0% { background-position: 200% 0; }
  100% { background-position: -200% 0; }
}

.error {
  text-align: center;
  padding: 48px;
  color: #e74c3c;
}

.empty-state {
  text-align: center;
  padding: 48px;
  color: #95a5a6;
}

.pagination {
  display: flex;
  justify-content: center;
  align-items: center;
  gap: 16px;
  margin-top: 32px;
}

.pagination button {
  padding: 8px 16px;
  background-color: #3498db;
  color: white;
  border: none;
  border-radius: 6px;
  cursor: pointer;
}

.pagination button:disabled {
  background-color: #bdc3c7;
  cursor: not-allowed;
}
</style>
```

### 文章詳情頁

```vue
<!-- pages/blog/[slug].vue -->
<template>
  <div class="post-detail">
    <div v-if="pending" class="loading">載入中...</div>

    <div v-else-if="error" class="error">
      <h1>404 - 找不到文章</h1>
      <NuxtLink to="/blog">返回列表</NuxtLink>
    </div>

    <article v-else-if="post">
      <header class="post-header">
        <img :src="post.image" :alt="post.title" class="cover-image" />
        <div class="header-content">
          <span class="category">{{ post.category }}</span>
          <h1>{{ post.title }}</h1>
          <div class="post-meta">
            <div class="author-info">
              <img :src="post.author.avatar" :alt="post.author.name" />
              <div>
                <strong>{{ post.author.name }}</strong>
                <span>{{ formatDate(post.createdAt) }}</span>
              </div>
            </div>
            <div class="stats">
              <span>👁 {{ post.views }} 次觀看</span>
              <span>💬 {{ post.comments?.length || 0 }} 則留言</span>
            </div>
          </div>
        </div>
      </header>

      <div class="post-content" v-html="post.content"></div>

      <div class="post-tags">
        <span v-for="tag in post.tags" :key="tag" class="tag">
          #{{ tag }}
        </span>
      </div>

      <!-- 相關文章 -->
      <section v-if="relatedPosts && relatedPosts.length > 0" class="related-posts">
        <h2>相關文章</h2>
        <div class="related-grid">
          <NuxtLink
            v-for="related in relatedPosts"
            :key="related.id"
            :to="`/blog/${related.slug}`"
            class="related-card"
          >
            <img :src="related.image" :alt="related.title" />
            <h3>{{ related.title }}</h3>
          </NuxtLink>
        </div>
      </section>
    </article>
  </div>
</template>

<script setup lang="ts">
const route = useRoute()
const slug = route.params.slug as string

// 獲取文章詳情
const { data: post, pending, error } = await useFetch(`/api/posts/${slug}`)

// 獲取相關文章
const { data: relatedPosts } = await useFetch(`/api/posts/${slug}/related`, {
  lazy: true
})

// 增加觀看次數
const incrementViews = async () => {
  await $fetch(`/api/posts/${slug}/view`, {
    method: 'POST'
  })
}

onMounted(() => {
  if (post.value) {
    incrementViews()
  }
})

// SEO
useHead(() => ({
  title: post.value?.title || '文章',
  meta: [
    { name: 'description', content: post.value?.excerpt },
    { property: 'og:title', content: post.value?.title },
    { property: 'og:description', content: post.value?.excerpt },
    { property: 'og:image', content: post.value?.image },
    { property: 'og:type', content: 'article' }
  ]
}))

const formatDate = (dateString: string) => {
  return new Date(dateString).toLocaleDateString('zh-TW', {
    year: 'numeric',
    month: 'long',
    day: 'numeric'
  })
}
</script>

<style scoped>
.post-detail {
  max-width: 800px;
  margin: 0 auto;
  padding: 24px;
}

.cover-image {
  width: 100%;
  height: 400px;
  object-fit: cover;
  border-radius: 12px;
  margin-bottom: 24px;
}

.post-header {
  margin-bottom: 32px;
}

.category {
  display: inline-block;
  padding: 6px 16px;
  background-color: #3498db;
  color: white;
  border-radius: 16px;
  font-size: 14px;
  margin-bottom: 16px;
}

.post-meta {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-top: 16px;
  padding-top: 16px;
  border-top: 1px solid #eee;
}

.author-info {
  display: flex;
  gap: 12px;
  align-items: center;
}

.author-info img {
  width: 48px;
  height: 48px;
  border-radius: 50%;
}

.author-info div {
  display: flex;
  flex-direction: column;
}

.stats {
  display: flex;
  gap: 16px;
  color: #95a5a6;
  font-size: 0.9rem;
}

.post-content {
  line-height: 1.8;
  font-size: 1.1rem;
  color: #2c3e50;
  margin-bottom: 32px;
}

.post-tags {
  display: flex;
  gap: 8px;
  flex-wrap: wrap;
  margin-bottom: 48px;
}

.tag {
  padding: 6px 12px;
  background-color: #ecf0f1;
  color: #7f8c8d;
  border-radius: 12px;
  font-size: 0.9rem;
}

.related-posts {
  margin-top: 48px;
  padding-top: 32px;
  border-top: 2px solid #eee;
}

.related-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(200px, 1fr));
  gap: 16px;
  margin-top: 24px;
}

.related-card {
  text-decoration: none;
  color: inherit;
  border: 1px solid #eee;
  border-radius: 8px;
  overflow: hidden;
  transition: transform 0.2s;
}

.related-card:hover {
  transform: translateY(-4px);
}

.related-card img {
  width: 100%;
  height: 120px;
  object-fit: cover;
}

.related-card h3 {
  padding: 12px;
  font-size: 0.95rem;
}
</style>
```

## 最佳實踐建議

### 1. 使用 Key 避免重複請求

```typescript
// ✅ 好 - 使用唯一 key
await useFetch('/api/users', { key: 'users' })

// ❌ 不好 - 沒有 key，可能導致重複請求
await useFetch('/api/users')
```

### 2. 選擇合適的 composable

```typescript
// useFetch - 簡單的 API 請求
await useFetch('/api/users')

// useAsyncData - 複雜的資料處理
await useAsyncData('users', async () => {
  const [users, roles] = await Promise.all([
    $fetch('/api/users'),
    $fetch('/api/roles')
  ])
  return { users, roles }
})

// $fetch - 事件處理器中
const handleSubmit = async () => {
  await $fetch('/api/submit', { method: 'POST', body: data })
}
```

### 3. 善用 Transform 和 Pick

```typescript
// 減少不必要的資料傳輸
await useFetch('/api/users', {
  pick: ['id', 'name'], // 只選擇需要的欄位
  transform: (users) => users.slice(0, 10) // 只要前 10 筆
})
```

### 4. 錯誤處理

```typescript
// ✅ 好 - 完整的錯誤處理
const { data, error } = await useFetch('/api/data')

if (error.value) {
  console.error('錯誤:', error.value)
  // 處理錯誤
}
```

## 總結

Nuxt 3 的資料獲取功能提供了：

✅ **SSR 支援** - 完整的伺服器端渲染
✅ **自動快取** - 智慧的資料快取機制
✅ **類型安全** - TypeScript 完整支援
✅ **錯誤處理** - 內建錯誤處理機制
✅ **Loading 狀態** - 自動追蹤載入狀態

掌握這些資料獲取技巧，將大幅提升你的 Nuxt 3 開發效率！
