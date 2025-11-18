# Nuxt.js 簡介與環境安裝

## Nuxt.js 是什麼？與 Vue 3 的差異

### Nuxt.js 簡介

Nuxt.js 是一個基於 Vue.js 的全端框架（Full-stack Framework），它為 Vue 應用提供了更高層次的抽象和最佳實踐。Nuxt 3 是基於 Vue 3 構建的最新版本，利用了 Vue 3 的 Composition API 和其他現代特性。

### 核心特點

- **混合渲染模式**：支援 SSR（Server-Side Rendering）、SSG（Static Site Generation）、CSR（Client-Side Rendering）
- **檔案式路由**：自動根據 `pages/` 目錄結構生成路由
- **自動導入**：組件、Composables、工具函數無需手動 import
- **伺服器引擎**：內建 Nitro 引擎，支援伺服器端 API 路由
- **TypeScript 支援**：原生 TypeScript 支援，無需額外配置
- **優化性能**：自動代碼分割、優化打包

### 與 Vue 3 的主要差異

| 特性 | Vue 3 | Nuxt 3 |
|------|-------|--------|
| 路由 | 需手動安裝 Vue Router | 檔案式路由，自動生成 |
| SSR | 需手動配置 | 內建支援 |
| 組件導入 | 需手動 import | 自動導入 |
| API 路由 | 需額外後端 | 內建 server/ 目錄 |
| 狀態管理 | Pinia/Vuex | 內建 useState |
| SEO | 需手動處理 | 內建 SEO 工具 |

## 為什麼選擇 Nuxt.js？

### 1. 開發效率提升

```typescript
// Vue 3 傳統寫法
import { ref } from 'vue'
import MyComponent from '@/components/MyComponent.vue'
import { useMyStore } from '@/stores/myStore'

export default {
  components: { MyComponent },
  setup() {
    const count = ref(0)
    const store = useMyStore()
    // ...
  }
}
```

```vue
<!-- Nuxt 3 - 自動導入 -->
<script setup lang="ts">
const count = ref(0) // ref 自動導入
const store = useMyStore() // composable 自動導入
// MyComponent 自動可用，無需導入
</script>

<template>
  <MyComponent />
</template>
```

### 2. SEO 優化

Nuxt 的 SSR 能力讓搜尋引擎可以直接爬取完整的 HTML 內容：

```vue
<script setup lang="ts">
useSeoMeta({
  title: '我的網站標題',
  description: '網站描述文字',
  ogTitle: 'Open Graph 標題',
  ogDescription: 'Open Graph 描述',
  ogImage: 'https://example.com/image.jpg',
})
</script>
```

### 3. 全端能力

可以在同一個專案中處理前端和後端邏輯：

```typescript
// server/api/users.ts
export default defineEventHandler(async (event) => {
  return {
    users: [
      { id: 1, name: '張三' },
      { id: 2, name: '李四' }
    ]
  }
})
```

```vue
<!-- pages/users.vue -->
<script setup lang="ts">
const { data } = await useFetch('/api/users')
</script>

<template>
  <div v-for="user in data.users" :key="user.id">
    {{ user.name }}
  </div>
</template>
```

## 系統需求

### 必要環境

- **Node.js**: v18.0.0 或更高版本（建議使用 LTS 版本）
- **套件管理器**: npm、yarn、pnpm 或 bun
- **作業系統**: Windows、macOS、Linux

### 檢查 Node.js 版本

```bash
node --version
# 應顯示 v18.0.0 或更高
```

### 推薦開發工具

- **VS Code** + Volar 擴充套件（取代 Vetur）
- **Vue DevTools** 瀏覽器擴充套件
- **TypeScript** 編輯器支援

## 安裝步驟（使用 nuxi）

### 1. 使用 nuxi 創建新專案

Nuxi 是 Nuxt 3 的官方腳手架工具。

```bash
# 使用 npx（不需要全局安裝）
npx nuxi@latest init my-nuxt-app

# 或使用 pnpm（推薦，速度更快）
pnpm dlx nuxi@latest init my-nuxt-app

# 或使用 yarn
yarn dlx nuxi@latest init my-nuxt-app

# 或使用 bun
bunx nuxi@latest init my-nuxt-app
```

### 2. 進入專案目錄

```bash
cd my-nuxt-app
```

### 3. 安裝依賴

```bash
# npm
npm install

# pnpm（推薦）
pnpm install

# yarn
yarn install

# bun
bun install
```

### 4. 啟動開發伺服器

```bash
# npm
npm run dev

# pnpm
pnpm dev

# yarn
yarn dev

# bun
bun run dev
```

預設會在 http://localhost:3000 啟動開發伺服器。

## 第一個 Nuxt 專案

### 專案初始結構

使用 `nuxi init` 創建的專案結構非常簡潔：

```
my-nuxt-app/
├── .nuxt/              # 自動生成（開發時）
├── node_modules/       # 依賴套件
├── public/             # 靜態資源
├── server/             # 伺服器端代碼
│   └── tsconfig.json
├── .gitignore
├── app.vue             # 應用程式入口
├── nuxt.config.ts      # Nuxt 配置文件
├── package.json
├── README.md
└── tsconfig.json       # TypeScript 配置
```

### 預設的 app.vue

```vue
<template>
  <div>
    <NuxtWelcome />
  </div>
</template>
```

### 修改 app.vue 建立你的第一個頁面

將 `app.vue` 替換為以下內容：

```vue
<template>
  <div class="container">
    <h1>{{ title }}</h1>
    <p>歡迎來到 Nuxt 3！</p>
    <p>你已經成功建立了第一個 Nuxt 專案。</p>

    <div class="counter">
      <button @click="decrement">-</button>
      <span class="count">{{ count }}</span>
      <button @click="increment">+</button>
    </div>

    <div class="info">
      <p>當前時間: {{ currentTime }}</p>
    </div>
  </div>
</template>

<script setup lang="ts">
// 響應式數據
const title = ref('我的第一個 Nuxt 3 應用')
const count = ref(0)
const currentTime = ref(new Date().toLocaleString('zh-TW'))

// 方法
const increment = () => {
  count.value++
}

const decrement = () => {
  count.value--
}

// 生命週期
onMounted(() => {
  // 每秒更新時間
  setInterval(() => {
    currentTime.value = new Date().toLocaleString('zh-TW')
  }, 1000)
})
</script>

<style scoped>
.container {
  max-width: 800px;
  margin: 0 auto;
  padding: 2rem;
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
}

h1 {
  color: #00DC82;
  font-size: 2.5rem;
  margin-bottom: 1rem;
}

p {
  font-size: 1.1rem;
  color: #333;
  line-height: 1.6;
}

.counter {
  display: flex;
  align-items: center;
  gap: 1rem;
  margin: 2rem 0;
}

.counter button {
  width: 50px;
  height: 50px;
  font-size: 1.5rem;
  border: 2px solid #00DC82;
  background: white;
  color: #00DC82;
  border-radius: 8px;
  cursor: pointer;
  transition: all 0.3s;
}

.counter button:hover {
  background: #00DC82;
  color: white;
}

.count {
  font-size: 2rem;
  font-weight: bold;
  color: #00DC82;
  min-width: 60px;
  text-align: center;
}

.info {
  margin-top: 2rem;
  padding: 1rem;
  background: #f5f5f5;
  border-radius: 8px;
}
</style>
```

## 開發伺服器啟動

### 啟動開發伺服器

```bash
npm run dev -- -o
# -o 參數會自動在瀏覽器中開啟
```

開發伺服器特性：
- **熱模組替換 (HMR)**：修改代碼後自動更新頁面
- **快速重新整理**：保留應用狀態
- **錯誤覆蓋層**：顯示詳細錯誤資訊
- **TypeScript 檢查**：即時類型檢查

### 其他常用指令

```bash
# 建構生產版本
npm run build

# 預覽生產版本
npm run preview

# 生成靜態網站
npm run generate

# 分析打包結果
npm run build -- --analyze
```

## 完整範例程式碼

### 建立一個簡單的待辦事項應用

```vue
<!-- app.vue -->
<template>
  <div class="app">
    <header>
      <h1>📝 Nuxt 3 待辦事項</h1>
    </header>

    <main>
      <div class="input-section">
        <input
          v-model="newTodo"
          @keyup.enter="addTodo"
          placeholder="輸入新的待辦事項..."
          class="todo-input"
        />
        <button @click="addTodo" class="add-btn">新增</button>
      </div>

      <div class="stats">
        <p>總計: {{ todos.length }} | 已完成: {{ completedCount }} | 待完成: {{ activeCount }}</p>
      </div>

      <ul class="todo-list">
        <li
          v-for="todo in todos"
          :key="todo.id"
          :class="{ completed: todo.completed }"
        >
          <input
            type="checkbox"
            v-model="todo.completed"
            class="checkbox"
          />
          <span class="todo-text">{{ todo.text }}</span>
          <button @click="removeTodo(todo.id)" class="delete-btn">刪除</button>
        </li>
      </ul>

      <div v-if="todos.length === 0" class="empty-state">
        <p>🎉 目前沒有待辦事項，開始新增一個吧！</p>
      </div>
    </main>
  </div>
</template>

<script setup lang="ts">
interface Todo {
  id: number
  text: string
  completed: boolean
}

// 響應式狀態
const newTodo = ref('')
const todos = ref<Todo[]>([
  { id: 1, text: '學習 Nuxt 3 基礎', completed: true },
  { id: 2, text: '建立第一個專案', completed: false },
  { id: 3, text: '部署到生產環境', completed: false },
])

// 計算屬性
const completedCount = computed(() =>
  todos.value.filter(todo => todo.completed).length
)

const activeCount = computed(() =>
  todos.value.filter(todo => !todo.completed).length
)

// 方法
const addTodo = () => {
  if (newTodo.value.trim()) {
    todos.value.push({
      id: Date.now(),
      text: newTodo.value,
      completed: false
    })
    newTodo.value = ''
  }
}

const removeTodo = (id: number) => {
  todos.value = todos.value.filter(todo => todo.id !== id)
}

// SEO Meta
useHead({
  title: 'Nuxt 3 待辦事項應用',
  meta: [
    { name: 'description', content: '使用 Nuxt 3 建立的待辦事項應用範例' }
  ]
})
</script>

<style scoped>
.app {
  max-width: 600px;
  margin: 0 auto;
  padding: 2rem;
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
}

header {
  text-align: center;
  margin-bottom: 2rem;
}

h1 {
  color: #00DC82;
  font-size: 2rem;
}

.input-section {
  display: flex;
  gap: 0.5rem;
  margin-bottom: 1.5rem;
}

.todo-input {
  flex: 1;
  padding: 0.75rem 1rem;
  font-size: 1rem;
  border: 2px solid #e0e0e0;
  border-radius: 8px;
  outline: none;
  transition: border-color 0.3s;
}

.todo-input:focus {
  border-color: #00DC82;
}

.add-btn {
  padding: 0.75rem 1.5rem;
  font-size: 1rem;
  background: #00DC82;
  color: white;
  border: none;
  border-radius: 8px;
  cursor: pointer;
  transition: background 0.3s;
}

.add-btn:hover {
  background: #00b36b;
}

.stats {
  text-align: center;
  margin-bottom: 1rem;
  color: #666;
  font-size: 0.9rem;
}

.todo-list {
  list-style: none;
  padding: 0;
}

.todo-list li {
  display: flex;
  align-items: center;
  padding: 1rem;
  margin-bottom: 0.5rem;
  background: white;
  border: 1px solid #e0e0e0;
  border-radius: 8px;
  transition: all 0.3s;
}

.todo-list li:hover {
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
}

.todo-list li.completed {
  opacity: 0.6;
}

.checkbox {
  width: 20px;
  height: 20px;
  margin-right: 1rem;
  cursor: pointer;
}

.todo-text {
  flex: 1;
  font-size: 1rem;
}

.completed .todo-text {
  text-decoration: line-through;
  color: #999;
}

.delete-btn {
  padding: 0.5rem 1rem;
  background: #ff4444;
  color: white;
  border: none;
  border-radius: 6px;
  cursor: pointer;
  font-size: 0.875rem;
  transition: background 0.3s;
}

.delete-btn:hover {
  background: #cc0000;
}

.empty-state {
  text-align: center;
  padding: 3rem;
  color: #999;
  font-size: 1.1rem;
}
</style>
```

## 最佳實踐建議

### 1. 使用 TypeScript

```bash
# Nuxt 3 預設支援 TypeScript，無需額外配置
# 只需使用 .ts 或在 <script> 中添加 lang="ts"
```

### 2. 利用自動導入

```vue
<script setup lang="ts">
// ✅ 推薦：自動導入
const count = ref(0)
const router = useRouter()

// ❌ 不推薦：手動導入
import { ref } from 'vue'
import { useRouter } from 'vue-router'
</script>
```

### 3. 使用 Composition API

```vue
<script setup lang="ts">
// ✅ 推薦：使用 <script setup>
const message = ref('Hello')
const doubled = computed(() => message.value + message.value)
</script>
```

### 4. SEO 優化

```vue
<script setup lang="ts">
// 為每個頁面設定適當的 meta 標籤
useHead({
  title: '頁面標題',
  meta: [
    { name: 'description', content: '頁面描述' },
    { property: 'og:title', content: '社群分享標題' },
  ]
})
</script>
```

### 5. 環境變量

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  runtimeConfig: {
    // 僅在伺服器端可用
    apiSecret: '',

    public: {
      // 客戶端和伺服器端都可用
      apiBase: '/api'
    }
  }
})
```

```vue
<script setup lang="ts">
const config = useRuntimeConfig()
console.log(config.public.apiBase) // 可以在客戶端使用
</script>
```

## 下一步

恭喜！你已經成功安裝並建立了第一個 Nuxt 3 專案。在下一章節中，我們將深入探討 Nuxt 3 的專案結構和配置文件。

### 推薦閱讀順序

1. ✅ Nuxt.js 簡介與環境安裝（當前章節）
2. ⏭️ 專案結構與配置檔案詳解
3. 路由系統基礎
4. 動態路由與路由參數
5. Layouts 布局系統

### 參考資源

- [Nuxt 3 官方文檔](https://nuxt.com)
- [Vue 3 官方文檔](https://vuejs.org)
- [Nuxt 3 範例](https://nuxt.com/docs/examples/hello-world)
