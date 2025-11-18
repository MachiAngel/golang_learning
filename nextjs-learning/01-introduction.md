# 第一章：Next.js 簡介與 Vue3 對比

## 1.1 什麼是 Next.js？

Next.js 是一個基於 React 的全端框架，由 Vercel 開發和維護。它提供了許多開箱即用的功能，讓開發者能夠快速建立生產級的 React 應用程式。

### 核心特性
- **伺服器端渲染（SSR）**：提升首屏載入速度和 SEO
- **靜態網站生成（SSG）**：預先生成靜態頁面
- **檔案系統路由**：基於檔案結構自動生成路由
- **API Routes**：內建 API 端點支援
- **圖片優化**：自動優化圖片載入
- **TypeScript 支援**：完整的 TypeScript 整合

## 1.2 Next.js vs Vue3 核心概念對比

### 1.2.1 框架定位

| 特性 | Next.js | Vue3 + Nuxt3 |
|------|---------|--------------|
| 基礎框架 | React | Vue |
| 全端框架 | Next.js | Nuxt3 |
| 官方支援 | Vercel | Vue.js 核心團隊 |

**對比說明**：
- Next.js 相當於 Vue 生態系統中的 **Nuxt3**
- 兩者都是基於各自基礎框架（React/Vue）的元框架（Meta-framework）
- 都提供 SSR、SSG、路由等全端解決方案

### 1.2.2 元件系統對比

**Vue3 元件範例：**
```vue
<template>
  <div class="card">
    <h2>{{ title }}</h2>
    <p>{{ description }}</p>
    <button @click="handleClick">點擊</button>
  </div>
</template>

<script setup>
import { ref } from 'vue'

const title = ref('Vue3 元件')
const description = ref('這是一個 Vue3 元件範例')

const handleClick = () => {
  console.log('按鈕被點擊')
}
</script>

<style scoped>
.card {
  padding: 20px;
  border: 1px solid #ddd;
}
</style>
```

**Next.js (React) 元件範例：**
```jsx
// components/Card.jsx
import { useState } from 'react'
import styles from './Card.module.css'

export default function Card() {
  const [title] = useState('Next.js 元件')
  const [description] = useState('這是一個 Next.js 元件範例')

  const handleClick = () => {
    console.log('按鈕被點擊')
  }

  return (
    <div className={styles.card}>
      <h2>{title}</h2>
      <p>{description}</p>
      <button onClick={handleClick}>點擊</button>
    </div>
  )
}
```

```css
/* Card.module.css */
.card {
  padding: 20px;
  border: 1px solid #ddd;
}
```

**關鍵差異：**
- Vue3 使用 **單檔案元件（SFC）**，HTML/JS/CSS 在同一檔案
- Next.js 使用 **JSX**，JavaScript 和 HTML 混合撰寫
- Vue3 使用 `@click`，React 使用 `onClick`（駝峰命名）
- Vue3 樣式預設 scoped，Next.js 使用 CSS Modules 實現隔離

### 1.2.3 響應式資料對比

**Vue3 響應式：**
```javascript
import { ref, reactive, computed } from 'vue'

// ref 用於基本型別
const count = ref(0)
const increment = () => count.value++

// reactive 用於物件
const state = reactive({
  user: 'John',
  age: 30
})

// computed 計算屬性
const doubleCount = computed(() => count.value * 2)
```

**Next.js (React) 響應式：**
```javascript
import { useState, useMemo } from 'react'

// useState 用於所有狀態
const [count, setCount] = useState(0)
const increment = () => setCount(count + 1)

// 物件狀態
const [state, setState] = useState({
  user: 'John',
  age: 30
})

// useMemo 類似 computed
const doubleCount = useMemo(() => count * 2, [count])
```

**關鍵差異：**
- Vue3 的 `ref` 需要用 `.value` 存取，React 直接使用變數
- Vue3 自動追蹤依賴，React 需要明確指定依賴陣列
- React 的 setState 不會合併物件，需要手動展開

### 1.2.4 生命週期對比

**Vue3 生命週期：**
```javascript
import { onMounted, onUpdated, onUnmounted } from 'vue'

onMounted(() => {
  console.log('元件已掛載')
})

onUpdated(() => {
  console.log('元件已更新')
})

onUnmounted(() => {
  console.log('元件已卸載')
})
```

**Next.js (React) 生命週期：**
```javascript
import { useEffect } from 'react'

useEffect(() => {
  // 等同於 onMounted
  console.log('元件已掛載')

  // 返回的函數等同於 onUnmounted
  return () => {
    console.log('元件已卸載')
  }
}, []) // 空依賴陣列表示只在掛載時執行

useEffect(() => {
  // 等同於 onUpdated（每次渲染都執行）
  console.log('元件已更新')
})
```

### 1.2.5 路由系統對比

**Vue Router (Vue3)：**
```javascript
// router/index.js
import { createRouter, createWebHistory } from 'vue-router'
import Home from '@/views/Home.vue'
import About from '@/views/About.vue'

const routes = [
  { path: '/', component: Home },
  { path: '/about', component: About },
  { path: '/user/:id', component: User }
]

const router = createRouter({
  history: createWebHistory(),
  routes
})
```

**Next.js 路由（檔案系統）：**
```
pages/
  index.js       → /
  about.js       → /about
  user/
    [id].js      → /user/:id
```

**關鍵差異：**
- Vue Router 需要手動配置路由
- Next.js 使用**檔案系統路由**，自動生成路由
- Next.js 更直觀，減少配置

### 1.2.6 資料獲取對比

**Vue3 (Composition API)：**
```javascript
import { ref, onMounted } from 'vue'

const data = ref(null)

onMounted(async () => {
  const response = await fetch('/api/data')
  data.value = await response.json()
})
```

**Next.js SSR：**
```javascript
export async function getServerSideProps() {
  const response = await fetch('https://api.example.com/data')
  const data = await response.json()

  return {
    props: { data }
  }
}

export default function Page({ data }) {
  return <div>{JSON.stringify(data)}</div>
}
```

**關鍵差異：**
- Vue3 主要在客戶端獲取資料
- Next.js 提供 `getServerSideProps`、`getStaticProps` 等伺服器端資料獲取方法
- Next.js 的 SSR 資料獲取發生在伺服器端，SEO 更友善

## 1.3 為什麼學習 Next.js？

### 對 Vue3 開發者的價值

1. **擴展技能樹**：掌握 React 生態系統，增加就業機會
2. **理解不同思維**：React 的單向資料流 vs Vue 的響應式系統
3. **SSR/SSG 最佳實踐**：Next.js 在這方面非常成熟
4. **豐富的生態系統**：React 社群龐大，第三方套件豐富

### 適用場景

- 需要 SEO 的內容網站（部落格、電商）
- 企業級應用
- 需要 SSR 的高效能應用
- 靜態網站生成

## 1.4 學習路線圖

對於有 Vue3 經驗的開發者，建議的學習順序：

1. **React 基礎** (1-2 週)
   - JSX 語法
   - useState, useEffect
   - 元件通訊

2. **Next.js 核心** (2-3 週)
   - 路由系統
   - 資料獲取方法
   - API Routes

3. **進階主題** (2-4 週)
   - 效能優化
   - 部署策略
   - 狀態管理（Redux/Zustand）

## 1.5 概念映射表

| Vue3/Nuxt3 | Next.js/React | 說明 |
|------------|---------------|------|
| `<template>` | JSX | 模板語法 |
| `ref()` | `useState()` | 響應式狀態 |
| `reactive()` | `useState({})` | 物件狀態 |
| `computed()` | `useMemo()` | 計算屬性 |
| `watch()` | `useEffect()` | 監聽變化 |
| `onMounted()` | `useEffect(() => {}, [])` | 掛載鉤子 |
| `v-if` | `{condition && <Component />}` | 條件渲染 |
| `v-for` | `{array.map()}` | 列表渲染 |
| `v-model` | `value` + `onChange` | 雙向綁定 |
| `<nuxt-link>` | `<Link>` | 路由連結 |
| `useRouter()` | `useRouter()` | 路由操作 |
| `pages/` (Nuxt) | `pages/` (Next) | 頁面目錄 |

## 1.6 本章小結

- Next.js 是 React 的全端框架，相當於 Vue 生態的 Nuxt3
- 主要差異在於語法（JSX vs Template）和響應式系統
- Next.js 在 SSR/SSG 方面有成熟的解決方案
- 有 Vue3 經驗能幫助你更快理解 Next.js 的概念

## 下一章預告

在下一章中，我們將實際建立開發環境，並創建第一個 Next.js 專案。
