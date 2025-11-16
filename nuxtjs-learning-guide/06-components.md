# 06 - Components 組件開發與最佳實踐

## 概述

Nuxt 3 提供了強大的組件系統，支援自動導入、靈活的命名規則和巢狀組件目錄結構。本章將深入探討如何在 Nuxt 3 中開發和使用組件。

## 1. components/ 目錄自動導入

Nuxt 3 會自動掃描 `components/` 目錄並導入所有組件，無需手動 import。

### 基本結構

```
components/
├── AppHeader.vue
├── AppFooter.vue
└── TheButton.vue
```

### 自動導入範例

```vue
<!-- pages/index.vue -->
<template>
  <div>
    <AppHeader />
    <main>
      <h1>歡迎來到首頁</h1>
      <TheButton>點擊我</TheButton>
    </main>
    <AppFooter />
  </div>
</template>

<!-- 無需手動 import，組件會自動可用 -->
```

### 配置自動導入

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  components: [
    {
      path: '~/components',
      pathPrefix: false, // 不使用路徑作為前綴
    },
    {
      path: '~/components/ui',
      prefix: 'Ui', // 添加前綴
    }
  ]
})
```

## 2. 組件命名規則

### Pascal Case（推薦）

```vue
<!-- components/UserProfile.vue -->
<template>
  <div class="user-profile">
    <h2>{{ userName }}</h2>
  </div>
</template>

<script setup lang="ts">
interface Props {
  userName: string
}

defineProps<Props>()
</script>
```

使用方式：
```vue
<template>
  <UserProfile user-name="張三" />
  <!-- 或 -->
  <UserProfile userName="張三" />
</template>
```

### Kebab Case

```vue
<!-- components/user-card.vue -->
<template>
  <div class="user-card">
    <slot />
  </div>
</template>
```

使用方式：
```vue
<template>
  <user-card>內容</user-card>
</template>
```

### 命名最佳實踐

1. **使用 Pascal Case**：組件檔案名使用 PascalCase（如 `UserProfile.vue`）
2. **語意化命名**：名稱應清楚描述組件用途
3. **前綴約定**：
   - `App` - 應用層級組件（如 `AppHeader.vue`）
   - `The` - 單例組件（如 `TheWelcome.vue`）
   - `Base` - 基礎組件（如 `BaseButton.vue`）

## 3. 巢狀組件目錄

Nuxt 3 支援巢狀目錄結構，組件名稱會根據目錄路徑自動生成。

### 目錄結構

```
components/
├── base/
│   ├── Button.vue
│   └── Input.vue
├── user/
│   ├── Profile.vue
│   ├── Avatar.vue
│   └── settings/
│       ├── General.vue
│       └── Security.vue
└── product/
    ├── Card.vue
    └── List.vue
```

### 使用巢狀組件

```vue
<template>
  <div>
    <!-- 路徑會成為組件名稱的一部分 -->
    <BaseButton>按鈕</BaseButton>
    <BaseInput v-model="value" />

    <UserProfile />
    <UserAvatar :src="avatarUrl" />

    <UserSettingsGeneral />
    <UserSettingsSecurity />

    <ProductCard :product="product" />
    <ProductList :items="products" />
  </div>
</template>
```

### 關閉路徑前綴

如果不想使用目錄名稱作為前綴：

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  components: [
    {
      path: '~/components/base',
      pathPrefix: false,
    }
  ]
})
```

這樣 `components/base/Button.vue` 就可以直接用 `<Button />` 而非 `<BaseButton />`。

## 4. Props 與 Emits

### 定義 Props（TypeScript）

```vue
<!-- components/ProductCard.vue -->
<template>
  <div class="product-card">
    <img :src="product.image" :alt="product.name" />
    <h3>{{ product.name }}</h3>
    <p class="price">NT$ {{ formattedPrice }}</p>
    <p v-if="showDescription">{{ product.description }}</p>
    <button @click="handleAddToCart">加入購物車</button>
  </div>
</template>

<script setup lang="ts">
interface Product {
  id: number
  name: string
  price: number
  image: string
  description?: string
}

interface Props {
  product: Product
  showDescription?: boolean
}

const props = withDefaults(defineProps<Props>(), {
  showDescription: false
})

const emit = defineEmits<{
  addToCart: [product: Product]
}>()

const formattedPrice = computed(() => {
  return props.product.price.toLocaleString('zh-TW')
})

const handleAddToCart = () => {
  emit('addToCart', props.product)
}
</script>

<style scoped>
.product-card {
  border: 1px solid #e0e0e0;
  border-radius: 8px;
  padding: 16px;
  transition: transform 0.2s;
}

.product-card:hover {
  transform: translateY(-4px);
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);
}

.price {
  font-size: 1.5rem;
  font-weight: bold;
  color: #e74c3c;
}
</style>
```

### 使用 Props 和 Emits

```vue
<template>
  <div class="product-list">
    <ProductCard
      v-for="product in products"
      :key="product.id"
      :product="product"
      :show-description="true"
      @add-to-cart="handleAddToCart"
    />
  </div>
</template>

<script setup lang="ts">
const products = ref([
  {
    id: 1,
    name: 'MacBook Pro',
    price: 59900,
    image: '/images/macbook.jpg',
    description: '強大的效能，輕薄的設計'
  },
  {
    id: 2,
    name: 'iPhone 15 Pro',
    price: 36900,
    image: '/images/iphone.jpg',
    description: '鈦金屬設計，A17 Pro 晶片'
  }
])

const handleAddToCart = (product: Product) => {
  console.log('加入購物車:', product.name)
  // 處理加入購物車邏輯
}
</script>
```

### Props 驗證（JavaScript）

```vue
<script setup>
const props = defineProps({
  title: {
    type: String,
    required: true
  },
  likes: {
    type: Number,
    default: 0
  },
  isPublished: {
    type: Boolean,
    default: false
  },
  tags: {
    type: Array,
    default: () => []
  },
  author: {
    type: Object,
    required: true,
    validator: (value) => {
      return value.name && value.email
    }
  }
})
</script>
```

## 5. Slots 使用

### 基本 Slot

```vue
<!-- components/BaseCard.vue -->
<template>
  <div class="card">
    <div class="card-header" v-if="$slots.header">
      <slot name="header"></slot>
    </div>
    <div class="card-body">
      <slot></slot>
    </div>
    <div class="card-footer" v-if="$slots.footer">
      <slot name="footer"></slot>
    </div>
  </div>
</template>

<style scoped>
.card {
  border: 1px solid #ddd;
  border-radius: 8px;
  overflow: hidden;
}

.card-header {
  background-color: #f5f5f5;
  padding: 16px;
  border-bottom: 1px solid #ddd;
}

.card-body {
  padding: 16px;
}

.card-footer {
  background-color: #f5f5f5;
  padding: 16px;
  border-top: 1px solid #ddd;
}
</style>
```

### 使用具名 Slot

```vue
<template>
  <BaseCard>
    <template #header>
      <h2>產品詳情</h2>
    </template>

    <p>這是產品的詳細描述...</p>
    <ul>
      <li>特色 1</li>
      <li>特色 2</li>
      <li>特色 3</li>
    </ul>

    <template #footer>
      <button>加入購物車</button>
      <button>立即購買</button>
    </template>
  </BaseCard>
</template>
```

### Scoped Slots

```vue
<!-- components/DataTable.vue -->
<template>
  <table class="data-table">
    <thead>
      <tr>
        <th v-for="column in columns" :key="column">
          {{ column }}
        </th>
      </tr>
    </thead>
    <tbody>
      <tr v-for="(item, index) in items" :key="index">
        <td v-for="column in columns" :key="column">
          <slot :item="item" :column="column" :index="index">
            {{ item[column] }}
          </slot>
        </td>
      </tr>
    </tbody>
  </table>
</template>

<script setup lang="ts">
interface Props {
  columns: string[]
  items: any[]
}

defineProps<Props>()
</script>

<style scoped>
.data-table {
  width: 100%;
  border-collapse: collapse;
}

.data-table th,
.data-table td {
  padding: 12px;
  text-align: left;
  border-bottom: 1px solid #ddd;
}

.data-table th {
  background-color: #f5f5f5;
  font-weight: bold;
}
</style>
```

### 使用 Scoped Slots

```vue
<template>
  <DataTable :columns="['name', 'price', 'stock']" :items="products">
    <template #default="{ item, column }">
      <span v-if="column === 'price'" class="price">
        NT$ {{ item.price.toLocaleString() }}
      </span>
      <span v-else-if="column === 'stock'" :class="{ 'low-stock': item.stock < 10 }">
        {{ item.stock }}
      </span>
      <span v-else>
        {{ item[column] }}
      </span>
    </template>
  </DataTable>
</template>

<script setup lang="ts">
const products = ref([
  { name: '產品 A', price: 1200, stock: 5 },
  { name: '產品 B', price: 2400, stock: 15 },
  { name: '產品 C', price: 3600, stock: 8 }
])
</script>

<style scoped>
.price {
  color: #e74c3c;
  font-weight: bold;
}

.low-stock {
  color: #f39c12;
  font-weight: bold;
}
</style>
```

## 6. 客戶端專用組件（\<ClientOnly\>）

某些組件只能在客戶端運行（例如使用 `window` 或 `document`），需要使用 `<ClientOnly>` 包裹。

### 基本使用

```vue
<!-- components/MapWidget.vue -->
<template>
  <div class="map-widget">
    <div ref="mapContainer" class="map-container"></div>
  </div>
</template>

<script setup lang="ts">
const mapContainer = ref<HTMLElement | null>(null)

onMounted(() => {
  // 使用瀏覽器 API
  if (window.google) {
    initializeMap()
  }
})

const initializeMap = () => {
  // 初始化地圖邏輯
  console.log('Map initialized')
}
</script>
```

### 在頁面中使用

```vue
<template>
  <div>
    <h1>聯絡我們</h1>

    <!-- 只在客戶端渲染 -->
    <ClientOnly>
      <MapWidget />
      <template #fallback>
        <div class="loading">載入地圖中...</div>
      </template>
    </ClientOnly>
  </div>
</template>
```

### 條件性客戶端組件

```vue
<template>
  <div>
    <ClientOnly>
      <ThemeToggle v-if="showThemeToggle" />
    </ClientOnly>

    <ClientOnly>
      <template #fallback>
        <p>載入中...</p>
      </template>
      <LazyHeavyComponent />
    </ClientOnly>
  </div>
</template>

<script setup lang="ts">
const showThemeToggle = ref(true)
</script>
```

### 使用 process.client 檢查

```vue
<script setup lang="ts">
const userAgent = ref('')

onMounted(() => {
  if (process.client) {
    userAgent.value = navigator.userAgent
  }
})

// 或使用條件判斷
if (process.client) {
  console.log('這只在客戶端執行')
}

if (process.server) {
  console.log('這只在伺服器端執行')
}
</script>
```

## 7. 完整範例：評論系統組件

### 目錄結構

```
components/
└── comment/
    ├── List.vue
    ├── Item.vue
    ├── Form.vue
    └── Avatar.vue
```

### CommentAvatar.vue

```vue
<!-- components/comment/Avatar.vue -->
<template>
  <div class="avatar" :style="{ width: size + 'px', height: size + 'px' }">
    <img v-if="src" :src="src" :alt="alt" />
    <div v-else class="avatar-placeholder">
      {{ initials }}
    </div>
  </div>
</template>

<script setup lang="ts">
interface Props {
  src?: string
  alt?: string
  name?: string
  size?: number
}

const props = withDefaults(defineProps<Props>(), {
  size: 40,
  alt: 'User avatar'
})

const initials = computed(() => {
  if (!props.name) return '?'
  return props.name
    .split(' ')
    .map(n => n[0])
    .join('')
    .toUpperCase()
    .slice(0, 2)
})
</script>

<style scoped>
.avatar {
  border-radius: 50%;
  overflow: hidden;
  display: flex;
  align-items: center;
  justify-content: center;
  background-color: #3498db;
  color: white;
  font-weight: bold;
}

.avatar img {
  width: 100%;
  height: 100%;
  object-fit: cover;
}

.avatar-placeholder {
  font-size: 14px;
}
</style>
```

### CommentItem.vue

```vue
<!-- components/comment/Item.vue -->
<template>
  <div class="comment-item">
    <CommentAvatar
      :src="comment.author.avatar"
      :name="comment.author.name"
      :size="48"
    />

    <div class="comment-content">
      <div class="comment-header">
        <span class="author-name">{{ comment.author.name }}</span>
        <span class="comment-date">{{ formattedDate }}</span>
      </div>

      <p class="comment-text">{{ comment.content }}</p>

      <div class="comment-actions">
        <button @click="handleLike" class="action-btn">
          <span>👍</span>
          <span v-if="comment.likes > 0">{{ comment.likes }}</span>
        </button>
        <button @click="handleReply" class="action-btn">
          回覆
        </button>
      </div>

      <!-- 嵌套回覆 -->
      <div v-if="comment.replies && comment.replies.length > 0" class="replies">
        <CommentItem
          v-for="reply in comment.replies"
          :key="reply.id"
          :comment="reply"
          @like="emit('like', $event)"
          @reply="emit('reply', $event)"
        />
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
interface Author {
  name: string
  avatar?: string
}

interface Comment {
  id: number
  author: Author
  content: string
  createdAt: string
  likes: number
  replies?: Comment[]
}

interface Props {
  comment: Comment
}

const props = defineProps<Props>()

const emit = defineEmits<{
  like: [commentId: number]
  reply: [commentId: number]
}>()

const formattedDate = computed(() => {
  const date = new Date(props.comment.createdAt)
  const now = new Date()
  const diff = now.getTime() - date.getTime()
  const days = Math.floor(diff / (1000 * 60 * 60 * 24))

  if (days === 0) return '今天'
  if (days === 1) return '昨天'
  if (days < 7) return `${days} 天前`
  return date.toLocaleDateString('zh-TW')
})

const handleLike = () => {
  emit('like', props.comment.id)
}

const handleReply = () => {
  emit('reply', props.comment.id)
}
</script>

<style scoped>
.comment-item {
  display: flex;
  gap: 12px;
  padding: 16px 0;
  border-bottom: 1px solid #f0f0f0;
}

.comment-content {
  flex: 1;
}

.comment-header {
  display: flex;
  align-items: center;
  gap: 12px;
  margin-bottom: 8px;
}

.author-name {
  font-weight: bold;
  color: #2c3e50;
}

.comment-date {
  font-size: 0.875rem;
  color: #95a5a6;
}

.comment-text {
  color: #34495e;
  line-height: 1.6;
  margin-bottom: 12px;
}

.comment-actions {
  display: flex;
  gap: 16px;
}

.action-btn {
  background: none;
  border: none;
  color: #7f8c8d;
  cursor: pointer;
  display: flex;
  align-items: center;
  gap: 4px;
  padding: 4px 8px;
  border-radius: 4px;
  transition: all 0.2s;
}

.action-btn:hover {
  background-color: #f8f9fa;
  color: #3498db;
}

.replies {
  margin-left: 24px;
  margin-top: 16px;
  border-left: 2px solid #ecf0f1;
  padding-left: 16px;
}
</style>
```

### CommentForm.vue

```vue
<!-- components/comment/Form.vue -->
<template>
  <form @submit.prevent="handleSubmit" class="comment-form">
    <textarea
      v-model="content"
      :placeholder="placeholder"
      rows="4"
      required
      class="comment-input"
    ></textarea>

    <div class="form-footer">
      <span class="char-count" :class="{ 'limit-exceeded': isLimitExceeded }">
        {{ content.length }} / {{ maxLength }}
      </span>

      <div class="form-actions">
        <button
          v-if="showCancel"
          type="button"
          @click="handleCancel"
          class="btn btn-cancel"
        >
          取消
        </button>
        <button
          type="submit"
          :disabled="!isValid"
          class="btn btn-submit"
        >
          {{ submitText }}
        </button>
      </div>
    </div>
  </form>
</template>

<script setup lang="ts">
interface Props {
  placeholder?: string
  submitText?: string
  showCancel?: boolean
  maxLength?: number
}

const props = withDefaults(defineProps<Props>(), {
  placeholder: '撰寫你的評論...',
  submitText: '發表評論',
  showCancel: false,
  maxLength: 500
})

const emit = defineEmits<{
  submit: [content: string]
  cancel: []
}>()

const content = ref('')

const isLimitExceeded = computed(() => {
  return content.value.length > props.maxLength
})

const isValid = computed(() => {
  return content.value.trim().length > 0 && !isLimitExceeded.value
})

const handleSubmit = () => {
  if (isValid.value) {
    emit('submit', content.value)
    content.value = ''
  }
}

const handleCancel = () => {
  content.value = ''
  emit('cancel')
}
</script>

<style scoped>
.comment-form {
  margin-bottom: 24px;
}

.comment-input {
  width: 100%;
  padding: 12px;
  border: 1px solid #ddd;
  border-radius: 8px;
  font-family: inherit;
  font-size: 14px;
  resize: vertical;
  transition: border-color 0.2s;
}

.comment-input:focus {
  outline: none;
  border-color: #3498db;
}

.form-footer {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-top: 12px;
}

.char-count {
  font-size: 0.875rem;
  color: #95a5a6;
}

.char-count.limit-exceeded {
  color: #e74c3c;
  font-weight: bold;
}

.form-actions {
  display: flex;
  gap: 8px;
}

.btn {
  padding: 8px 16px;
  border: none;
  border-radius: 6px;
  cursor: pointer;
  font-weight: 500;
  transition: all 0.2s;
}

.btn-cancel {
  background-color: #ecf0f1;
  color: #7f8c8d;
}

.btn-cancel:hover {
  background-color: #d5dbdb;
}

.btn-submit {
  background-color: #3498db;
  color: white;
}

.btn-submit:hover:not(:disabled) {
  background-color: #2980b9;
}

.btn-submit:disabled {
  background-color: #bdc3c7;
  cursor: not-allowed;
}
</style>
```

### CommentList.vue

```vue
<!-- components/comment/List.vue -->
<template>
  <div class="comment-list">
    <div class="comment-header">
      <h3>評論 ({{ comments.length }})</h3>
    </div>

    <CommentForm
      @submit="handleAddComment"
      placeholder="分享你的想法..."
    />

    <div v-if="comments.length === 0" class="empty-state">
      <p>還沒有評論，成為第一個留言的人吧！</p>
    </div>

    <div v-else class="comments">
      <CommentItem
        v-for="comment in comments"
        :key="comment.id"
        :comment="comment"
        @like="handleLike"
        @reply="handleReply"
      />
    </div>
  </div>
</template>

<script setup lang="ts">
interface Author {
  name: string
  avatar?: string
}

interface Comment {
  id: number
  author: Author
  content: string
  createdAt: string
  likes: number
  replies?: Comment[]
}

const comments = ref<Comment[]>([
  {
    id: 1,
    author: {
      name: '王小明',
      avatar: ''
    },
    content: '這篇文章寫得真好！對我幫助很大。',
    createdAt: '2024-01-15T10:30:00',
    likes: 5,
    replies: [
      {
        id: 2,
        author: {
          name: '李大華'
        },
        content: '同意！我也學到很多。',
        createdAt: '2024-01-15T11:00:00',
        likes: 2
      }
    ]
  },
  {
    id: 3,
    author: {
      name: '陳美麗'
    },
    content: '請問有更多相關的資源可以參考嗎？',
    createdAt: '2024-01-15T14:20:00',
    likes: 3
  }
])

const handleAddComment = (content: string) => {
  const newComment: Comment = {
    id: Date.now(),
    author: {
      name: '訪客'
    },
    content,
    createdAt: new Date().toISOString(),
    likes: 0
  }

  comments.value.unshift(newComment)
}

const handleLike = (commentId: number) => {
  const findAndLike = (items: Comment[]): boolean => {
    for (const comment of items) {
      if (comment.id === commentId) {
        comment.likes++
        return true
      }
      if (comment.replies && findAndLike(comment.replies)) {
        return true
      }
    }
    return false
  }

  findAndLike(comments.value)
}

const handleReply = (commentId: number) => {
  console.log('Reply to comment:', commentId)
  // 實作回覆邏輯
}
</script>

<style scoped>
.comment-list {
  max-width: 800px;
  margin: 0 auto;
  padding: 24px;
}

.comment-header h3 {
  font-size: 1.5rem;
  color: #2c3e50;
  margin-bottom: 24px;
}

.empty-state {
  text-align: center;
  padding: 48px 24px;
  color: #95a5a6;
}

.comments {
  margin-top: 24px;
}
</style>
```

### 使用完整評論系統

```vue
<!-- pages/article/[id].vue -->
<template>
  <div class="article-page">
    <article class="article-content">
      <h1>文章標題</h1>
      <p>文章內容...</p>
    </article>

    <CommentList />
  </div>
</template>

<style scoped>
.article-page {
  max-width: 1200px;
  margin: 0 auto;
  padding: 24px;
}

.article-content {
  margin-bottom: 48px;
}
</style>
```

## 最佳實踐建議

### 1. 組件拆分原則

- **單一職責**：每個組件只負責一個功能
- **可重用性**：設計通用的基礎組件
- **合理大小**：組件代碼不超過 200-300 行

### 2. Props 設計

- 使用 TypeScript 定義清晰的介面
- 提供合理的預設值
- 避免過多的 props（超過 5-7 個考慮重構）

### 3. 效能優化

```vue
<script setup lang="ts">
// 使用 computed 快取計算結果
const expensiveValue = computed(() => {
  return heavyCalculation(props.data)
})

// 使用 v-once 對靜態內容
</script>

<template>
  <div v-once class="static-content">
    <!-- 靜態內容 -->
  </div>
</template>
```

### 4. 組件文件組織

```typescript
// 推薦順序
<script setup lang="ts">
// 1. Imports
import { xxx } from 'xxx'

// 2. Props & Emits
interface Props { }
const props = defineProps<Props>()
const emit = defineEmits<{ }>()

// 3. Composables
const route = useRoute()

// 4. Reactive State
const state = ref()

// 5. Computed
const computed = computed(() => {})

// 6. Methods
const method = () => {}

// 7. Lifecycle
onMounted(() => {})
</script>

<template>
  <!-- 模板 -->
</template>

<style scoped>
/* 樣式 */
</style>
```

### 5. 命名一致性

- 組件檔案：`PascalCase.vue`
- 組件使用：`<PascalCase />` 或 `<pascal-case />`
- Props：`camelCase`
- Events：`kebab-case`

## 總結

Nuxt 3 的組件系統提供了：

✅ **自動導入** - 無需手動 import，提高開發效率
✅ **靈活命名** - 支援多種命名方式和目錄結構
✅ **TypeScript 支援** - 完整的類型檢查
✅ **Slots 系統** - 靈活的內容分發
✅ **SSR 友善** - `<ClientOnly>` 處理客戶端專用邏輯

掌握這些組件開發技巧，將幫助你建立可維護、可擴展的 Nuxt 3 應用程式。
