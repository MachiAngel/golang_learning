# SSG (Static Site Generation) 完整指南

## 什麼是 SSG？

Static Site Generation（靜態網站生成）是一種在**建置時期（Build Time）**預先將所有頁面渲染成 HTML 檔案的技術。生成的靜態檔案可以直接部署到 CDN，無需 Node.js 伺服器。

### SSG 流程圖

```
開發階段
    ↓
執行 nuxi generate
    ↓
讀取所有路由
    ↓
對每個路由執行 SSR
    ↓
生成靜態 HTML/CSS/JS
    ↓
輸出到 .output/public
    ↓
部署到 CDN
    ↓
用戶訪問 → 直接取得靜態檔案
    ↓
瀏覽器執行 JavaScript
    ↓
Hydration（成為互動式應用）
```

## SSG vs SSR 差異

### 對照表

| 特性 | SSG | SSR |
|------|-----|-----|
| 渲染時機 | 建置時期 | 請求時期 |
| 伺服器需求 | 不需要 Node.js | 需要 Node.js |
| 回應速度 | 極快（CDN） | 較慢（需渲染） |
| 部署成本 | 極低 | 較高 |
| 內容更新 | 需重新建置 | 即時 |
| 動態內容 | 受限 | 完全支援 |
| SEO | 優秀 | 優秀 |
| 擴展性 | 優秀（CDN） | 需負載平衡 |
| 適用場景 | 靜態內容 | 動態內容 |

### 視覺化比較

```
SSG:
建置時 → [生成所有 HTML] → 部署到 CDN
用戶請求 → CDN → 立即回傳 HTML ⚡ (極快)

SSR:
用戶請求 → Node.js Server → 執行渲染 → 回傳 HTML 🐢 (較慢)

CSR (Client-Side Rendering):
用戶請求 → 空白 HTML + JS → 瀏覽器執行 → 渲染內容 🐌 (最慢)
```

## 使用 nuxi generate 建立靜態網站

### 基本指令

```bash
# 生成靜態網站
npx nuxi generate

# 或使用 npm scripts
npm run generate

# 生成後預覽
npx nuxi preview
```

### 輸出結構

```
.output/
├── public/              # 靜態檔案（可直接部署）
│   ├── index.html      # 首頁
│   ├── about.html      # 關於頁面
│   ├── _nuxt/          # JS/CSS 資源
│   │   ├── entry.js
│   │   └── app.css
│   └── _payload.json   # 資料 payload
└── server/             # 伺服器端代碼（SSG 不使用）
```

### nuxt.config.ts 設定

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  // 確保使用 SSG 模式
  ssr: true, // SSG 仍需要 SSR（在建置時）

  // Nitro 預設設定
  nitro: {
    preset: 'static', // 可選，預設會自動偵測

    // 預渲染設定
    prerender: {
      // 自動爬取並生成所有可達路由
      crawlLinks: true,

      // 手動指定要預渲染的路由
      routes: [
        '/',
        '/about',
        '/contact'
      ],

      // 忽略的路由
      ignore: [
        '/admin',
        '/api'
      ]
    }
  },

  // 實驗性功能
  experimental: {
    payloadExtraction: true // 提取資料到 _payload.json
  }
})
```

## 預渲染路由

### 自動偵測路由

Nuxt 會自動偵測 `pages/` 目錄中的檔案：

```
pages/
├── index.vue           → /
├── about.vue           → /about
├── contact.vue         → /contact
└── blog/
    ├── index.vue       → /blog
    └── [slug].vue      → /blog/* (需手動指定)
```

### 手動指定路由

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  nitro: {
    prerender: {
      routes: [
        '/',
        '/about',
        '/blog',
        // 動態路由需要明確指定
        '/blog/nuxt-3-intro',
        '/blog/vue-composition-api',
        '/blog/typescript-guide'
      ]
    }
  }
})
```

### 使用 Nitro Hook 動態生成路由

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  hooks: {
    async 'nitro:config'(nitroConfig) {
      // 從 API 或資料庫獲取所有路由
      const posts = await fetch('https://api.example.com/posts')
        .then(r => r.json())

      const routes = posts.map(post => `/blog/${post.slug}`)

      nitroConfig.prerender = nitroConfig.prerender || {}
      nitroConfig.prerender.routes = [
        ...(nitroConfig.prerender.routes || []),
        ...routes
      ]
    }
  }
})
```

### 從檔案系統讀取路由

```typescript
// nuxt.config.ts
import fs from 'fs'
import path from 'path'

export default defineNuxtConfig({
  hooks: {
    'nitro:config'(nitroConfig) {
      // 讀取 Markdown 文章
      const contentDir = path.resolve(__dirname, 'content/blog')
      const files = fs.readdirSync(contentDir)

      const routes = files
        .filter(file => file.endsWith('.md'))
        .map(file => `/blog/${file.replace('.md', '')}`)

      nitroConfig.prerender = nitroConfig.prerender || {}
      nitroConfig.prerender.routes = [
        ...(nitroConfig.prerender.routes || []),
        ...routes
      ]
    }
  }
})
```

## 動態路由的靜態生成

### 完整部落格範例

```vue
<!-- pages/blog/[slug].vue -->
<template>
  <article class="blog-post">
    <Head>
      <Title>{{ post?.title }} - 我的部落格</Title>
      <Meta name="description" :content="post?.excerpt" />
      <Meta property="og:title" :content="post?.title" />
      <Meta property="og:description" :content="post?.excerpt" />
      <Meta property="og:image" :content="post?.coverImage" />
    </Head>

    <div v-if="!post" class="loading">
      載入中...
    </div>

    <div v-else class="post-content">
      <!-- 封面圖片 -->
      <img :src="post.coverImage" :alt="post.title" class="cover-image" />

      <!-- 文章標題 -->
      <h1>{{ post.title }}</h1>

      <!-- 文章資訊 -->
      <div class="post-meta">
        <span class="author">作者: {{ post.author }}</span>
        <span class="date">發布日期: {{ formatDate(post.publishedAt) }}</span>
        <span class="reading-time">閱讀時間: {{ post.readingTime }} 分鐘</span>
      </div>

      <!-- 標籤 -->
      <div class="tags">
        <NuxtLink
          v-for="tag in post.tags"
          :key="tag"
          :to="`/blog/tag/${tag}`"
          class="tag"
        >
          #{{ tag }}
        </NuxtLink>
      </div>

      <!-- 文章內容 -->
      <div class="content" v-html="post.content"></div>

      <!-- 分享按鈕 -->
      <div class="share-buttons">
        <h3>分享這篇文章</h3>
        <button @click="shareOnTwitter">Twitter</button>
        <button @click="shareOnFacebook">Facebook</button>
        <button @click="copyLink">複製連結</button>
      </div>

      <!-- 相關文章 -->
      <div class="related-posts">
        <h3>相關文章</h3>
        <div class="post-grid">
          <NuxtLink
            v-for="related in relatedPosts"
            :key="related.slug"
            :to="`/blog/${related.slug}`"
            class="post-card"
          >
            <img :src="related.coverImage" :alt="related.title" />
            <h4>{{ related.title }}</h4>
            <p>{{ related.excerpt }}</p>
          </NuxtLink>
        </div>
      </div>
    </div>
  </article>
</template>

<script setup>
const route = useRoute()
const slug = route.params.slug

// 在建置時獲取文章資料
const { data: post } = await useAsyncData(`post-${slug}`, () =>
  $fetch(`/api/blog/${slug}`)
)

// 獲取相關文章
const { data: relatedPosts } = await useAsyncData(`related-${slug}`, () =>
  $fetch('/api/blog/related', {
    query: { slug, limit: 3 }
  })
)

// 格式化日期
const formatDate = (dateString) => {
  return new Date(dateString).toLocaleDateString('zh-TW', {
    year: 'numeric',
    month: 'long',
    day: 'numeric'
  })
}

// 客戶端互動功能
const shareOnTwitter = () => {
  if (process.client) {
    const url = window.location.href
    const text = post.value.title
    window.open(`https://twitter.com/intent/tweet?url=${url}&text=${text}`, '_blank')
  }
}

const shareOnFacebook = () => {
  if (process.client) {
    const url = window.location.href
    window.open(`https://www.facebook.com/sharer/sharer.php?u=${url}`, '_blank')
  }
}

const copyLink = () => {
  if (process.client) {
    navigator.clipboard.writeText(window.location.href)
    alert('連結已複製！')
  }
}
</script>

<style scoped>
.blog-post {
  max-width: 800px;
  margin: 0 auto;
  padding: 40px 20px;
}

.cover-image {
  width: 100%;
  height: 400px;
  object-fit: cover;
  border-radius: 12px;
  margin-bottom: 30px;
}

h1 {
  font-size: 42px;
  line-height: 1.2;
  margin-bottom: 20px;
}

.post-meta {
  display: flex;
  gap: 20px;
  color: #666;
  margin-bottom: 20px;
  flex-wrap: wrap;
}

.tags {
  display: flex;
  gap: 10px;
  margin-bottom: 30px;
}

.tag {
  padding: 6px 12px;
  background: #e3f2fd;
  color: #1976d2;
  border-radius: 16px;
  text-decoration: none;
  font-size: 14px;
}

.content {
  font-size: 18px;
  line-height: 1.8;
  margin-bottom: 40px;
}

.content :deep(h2) {
  margin-top: 40px;
  margin-bottom: 20px;
}

.content :deep(p) {
  margin-bottom: 20px;
}

.content :deep(pre) {
  background: #f5f5f5;
  padding: 20px;
  border-radius: 8px;
  overflow-x: auto;
}

.share-buttons {
  border-top: 2px solid #eee;
  padding-top: 30px;
  margin-bottom: 60px;
}

.share-buttons button {
  margin-right: 10px;
  padding: 10px 20px;
  border: none;
  background: #3498db;
  color: white;
  border-radius: 4px;
  cursor: pointer;
}

.related-posts {
  border-top: 2px solid #eee;
  padding-top: 40px;
}

.post-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(200px, 1fr));
  gap: 20px;
  margin-top: 20px;
}

.post-card {
  border: 1px solid #eee;
  border-radius: 8px;
  overflow: hidden;
  text-decoration: none;
  color: inherit;
  transition: box-shadow 0.3s;
}

.post-card:hover {
  box-shadow: 0 4px 12px rgba(0,0,0,0.1);
}

.post-card img {
  width: 100%;
  height: 150px;
  object-fit: cover;
}

.post-card h4 {
  padding: 12px;
  margin: 0;
}

.post-card p {
  padding: 0 12px 12px;
  margin: 0;
  font-size: 14px;
  color: #666;
}
</style>
```

### API 路由實作

```javascript
// server/api/blog/[slug].get.js
// 模擬從資料庫或 CMS 獲取文章
export default defineEventHandler(async (event) => {
  const slug = getRouterParam(event, 'slug')

  // 模擬資料（實際應從資料庫或檔案系統讀取）
  const posts = {
    'nuxt-3-intro': {
      slug: 'nuxt-3-intro',
      title: 'Nuxt 3 完整介紹',
      excerpt: '深入了解 Nuxt 3 的新功能和改進...',
      content: '<p>Nuxt 3 是一個強大的 Vue.js 框架...</p>',
      coverImage: 'https://picsum.photos/800/400?random=1',
      author: '張小明',
      publishedAt: '2024-01-15',
      readingTime: 8,
      tags: ['Nuxt', 'Vue', 'SSR']
    },
    'vue-composition-api': {
      slug: 'vue-composition-api',
      title: 'Vue 3 Composition API 深入解析',
      excerpt: 'Composition API 讓你的代碼更有組織性...',
      content: '<p>Composition API 是 Vue 3 的核心特性...</p>',
      coverImage: 'https://picsum.photos/800/400?random=2',
      author: '李小華',
      publishedAt: '2024-01-20',
      readingTime: 10,
      tags: ['Vue', 'JavaScript', 'Composition API']
    },
    'typescript-guide': {
      slug: 'typescript-guide',
      title: 'TypeScript 完全指南',
      excerpt: 'TypeScript 讓你的 JavaScript 更安全...',
      content: '<p>TypeScript 提供強大的型別系統...</p>',
      coverImage: 'https://picsum.photos/800/400?random=3',
      author: '王大明',
      publishedAt: '2024-02-01',
      readingTime: 12,
      tags: ['TypeScript', 'JavaScript', 'Type Safety']
    }
  }

  const post = posts[slug]

  if (!post) {
    throw createError({
      statusCode: 404,
      statusMessage: '找不到文章'
    })
  }

  return post
})
```

```javascript
// server/api/blog/related.get.js
export default defineEventHandler(async (event) => {
  const query = getQuery(event)
  const currentSlug = query.slug
  const limit = parseInt(query.limit) || 3

  // 模擬相關文章邏輯
  const allPosts = [
    {
      slug: 'nuxt-3-intro',
      title: 'Nuxt 3 完整介紹',
      excerpt: '深入了解 Nuxt 3 的新功能和改進...',
      coverImage: 'https://picsum.photos/300/200?random=1'
    },
    {
      slug: 'vue-composition-api',
      title: 'Vue 3 Composition API 深入解析',
      excerpt: 'Composition API 讓你的代碼更有組織性...',
      coverImage: 'https://picsum.photos/300/200?random=2'
    },
    {
      slug: 'typescript-guide',
      title: 'TypeScript 完全指南',
      excerpt: 'TypeScript 讓你的 JavaScript 更安全...',
      coverImage: 'https://picsum.photos/300/200?random=3'
    }
  ]

  // 排除當前文章
  const related = allPosts
    .filter(post => post.slug !== currentSlug)
    .slice(0, limit)

  return related
})
```

### 文章列表頁面

```vue
<!-- pages/blog/index.vue -->
<template>
  <div class="blog-page">
    <Head>
      <Title>部落格 - 我的網站</Title>
      <Meta name="description" content="閱讀我們的最新文章和教學" />
    </Head>

    <header class="blog-header">
      <h1>部落格</h1>
      <p>探索最新的技術文章和教學</p>
    </header>

    <!-- 文章列表 -->
    <div class="posts-grid">
      <article
        v-for="post in posts"
        :key="post.slug"
        class="post-card"
      >
        <NuxtLink :to="`/blog/${post.slug}`">
          <img :src="post.coverImage" :alt="post.title" />
          <div class="card-content">
            <h2>{{ post.title }}</h2>
            <p class="excerpt">{{ post.excerpt }}</p>
            <div class="card-meta">
              <span>{{ post.author }}</span>
              <span>{{ formatDate(post.publishedAt) }}</span>
            </div>
            <div class="tags">
              <span v-for="tag in post.tags" :key="tag" class="tag">
                #{{ tag }}
              </span>
            </div>
          </div>
        </NuxtLink>
      </article>
    </div>
  </div>
</template>

<script setup>
// 在建置時獲取所有文章
const { data: posts } = await useAsyncData('blog-posts', () =>
  $fetch('/api/blog')
)

const formatDate = (dateString) => {
  return new Date(dateString).toLocaleDateString('zh-TW', {
    year: 'numeric',
    month: 'long',
    day: 'numeric'
  })
}
</script>

<style scoped>
.blog-page {
  max-width: 1200px;
  margin: 0 auto;
  padding: 40px 20px;
}

.blog-header {
  text-align: center;
  margin-bottom: 60px;
}

.blog-header h1 {
  font-size: 48px;
  margin-bottom: 16px;
}

.posts-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(350px, 1fr));
  gap: 30px;
}

.post-card {
  border: 1px solid #eee;
  border-radius: 12px;
  overflow: hidden;
  transition: transform 0.3s, box-shadow 0.3s;
}

.post-card:hover {
  transform: translateY(-4px);
  box-shadow: 0 8px 24px rgba(0,0,0,0.1);
}

.post-card a {
  text-decoration: none;
  color: inherit;
}

.post-card img {
  width: 100%;
  height: 250px;
  object-fit: cover;
}

.card-content {
  padding: 20px;
}

.card-content h2 {
  font-size: 24px;
  margin-bottom: 12px;
}

.excerpt {
  color: #666;
  margin-bottom: 16px;
}

.card-meta {
  display: flex;
  justify-content: space-between;
  color: #999;
  font-size: 14px;
  margin-bottom: 12px;
}

.tags {
  display: flex;
  gap: 8px;
  flex-wrap: wrap;
}

.tag {
  padding: 4px 10px;
  background: #f0f0f0;
  border-radius: 12px;
  font-size: 12px;
}
</style>
```

```javascript
// server/api/blog/index.get.js
export default defineEventHandler(async () => {
  // 回傳所有文章列表
  const posts = [
    {
      slug: 'nuxt-3-intro',
      title: 'Nuxt 3 完整介紹',
      excerpt: '深入了解 Nuxt 3 的新功能和改進...',
      coverImage: 'https://picsum.photos/800/400?random=1',
      author: '張小明',
      publishedAt: '2024-01-15',
      tags: ['Nuxt', 'Vue', 'SSR']
    },
    {
      slug: 'vue-composition-api',
      title: 'Vue 3 Composition API 深入解析',
      excerpt: 'Composition API 讓你的代碼更有組織性...',
      coverImage: 'https://picsum.photos/800/400?random=2',
      author: '李小華',
      publishedAt: '2024-01-20',
      tags: ['Vue', 'JavaScript', 'Composition API']
    },
    {
      slug: 'typescript-guide',
      title: 'TypeScript 完全指南',
      excerpt: 'TypeScript 讓你的 JavaScript 更安全...',
      coverImage: 'https://picsum.photos/800/400?random=3',
      author: '王大明',
      publishedAt: '2024-02-01',
      tags: ['TypeScript', 'JavaScript', 'Type Safety']
    }
  ]

  return posts
})
```

## Payload Extraction

Payload Extraction 將 API 回應資料提取到獨立的 JSON 檔案，避免重複嵌入到每個 HTML 中。

### 啟用 Payload Extraction

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  experimental: {
    payloadExtraction: true
  }
})
```

### 生成的檔案結構

```
.output/public/
├── index.html
├── blog.html
├── blog/
│   ├── nuxt-3-intro.html
│   └── vue-composition-api.html
└── _payload.json/
    ├── blog.json              # /blog 頁面的資料
    ├── blog/
    │   ├── nuxt-3-intro.json  # 文章資料
    │   └── vue-composition-api.json
```

### 優勢

1. **減少 HTML 大小**: 資料不嵌入 HTML
2. **可快取**: JSON 檔案可獨立快取
3. **首次載入更快**: HTML 檔案更小
4. **導航更快**: 切換頁面時只載入需要的 JSON

## 何時使用 SSG？

### ✅ 適合 SSG 的場景

1. **部落格和文件網站**
   - 內容變動頻率低
   - 所有文章路由可預先知道
   - 優秀的 SEO 表現

2. **行銷網站和登陸頁**
   - 靜態內容為主
   - 需要極快的載入速度
   - 降低伺服器成本

3. **作品集和個人網站**
   - 內容相對固定
   - 可部署到免費靜態主機

4. **電商產品目錄（小型）**
   - 產品數量有限（< 10,000）
   - 價格不頻繁變動
   - 可搭配 ISR 更新

### ❌ 不適合 SSG 的場景

1. **即時性內容**
   - 新聞網站（內容每分鐘更新）
   - 股票/體育比分
   - 聊天應用

2. **大量動態路由**
   - 百萬級產品的電商網站
   - 建置時間過長
   - 更適合 SSR + 快取

3. **用戶專屬內容**
   - 需要登入的頁面
   - 個人化推薦
   - 更適合 CSR 或 SSR

4. **頻繁變動的資料**
   - 庫存數量
   - 即時評論
   - 可考慮 Hybrid Rendering

## CDN 部署策略

### 1. Netlify 部署

```bash
# 安裝 Netlify CLI
npm install -g netlify-cli

# 建置
npm run generate

# 部署
netlify deploy --prod --dir=.output/public
```

`netlify.toml` 設定：

```toml
[build]
  command = "npm run generate"
  publish = ".output/public"

[[redirects]]
  from = "/*"
  to = "/index.html"
  status = 200
```

### 2. Vercel 部署

```bash
# 安裝 Vercel CLI
npm install -g vercel

# 部署
vercel --prod
```

`vercel.json` 設定：

```json
{
  "buildCommand": "npm run generate",
  "outputDirectory": ".output/public",
  "devCommand": "npm run dev"
}
```

### 3. Cloudflare Pages 部署

在 Cloudflare Pages 控制台設定：
- Build command: `npm run generate`
- Build output directory: `.output/public`

### 4. GitHub Pages 部署

```yaml
# .github/workflows/deploy.yml
name: Deploy to GitHub Pages

on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'

      - name: Install dependencies
        run: npm install

      - name: Generate static site
        run: npm run generate

      - name: Deploy to GitHub Pages
        uses: peaceiris/actions-gh-pages@v3
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: .output/public
```

### 5. AWS S3 + CloudFront

```bash
# 建置
npm run generate

# 上傳到 S3
aws s3 sync .output/public s3://your-bucket-name --delete

# 清除 CloudFront 快取
aws cloudfront create-invalidation --distribution-id YOUR_DIST_ID --paths "/*"
```

## 效能比較

### 載入速度測試

```
測試條件: 同一個部落格網站，50 篇文章

SSG (CDN):
- TTFB: 10-50ms ⚡⚡⚡
- FCP: 300-500ms ⚡⚡⚡
- TTI: 800-1200ms ⚡⚡⚡

SSR (Node.js):
- TTFB: 200-500ms ⚡⚡
- FCP: 500-1000ms ⚡⚡
- TTI: 1200-2000ms ⚡⚡

CSR (SPA):
- TTFB: 50-100ms ⚡⚡⚡
- FCP: 1500-3000ms ⚡
- TTI: 2000-4000ms ⚡
```

### 成本比較（每月 100,000 訪問）

```
SSG + CDN (Netlify/Vercel 免費層):
- 伺服器: $0
- CDN: $0
- 總計: $0 💰💰💰

SSR (VPS):
- 伺服器: $20-50
- CDN (可選): $5-20
- 總計: $25-70 💰

SSR (Serverless):
- 函數執行: $10-30
- CDN: $5-15
- 總計: $15-45 💰💰
```

## 最佳實踐

### 1. 使用增量建置

對於大型網站，使用增量建置只重建變更的頁面：

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  nitro: {
    prerender: {
      // 只預渲染變更的路由
      crawlLinks: false,
      routes: process.env.CHANGED_ROUTES?.split(',') || ['/']
    }
  }
})
```

### 2. 最佳化圖片

```vue
<template>
  <!-- 使用 Nuxt Image 最佳化圖片 -->
  <NuxtImg
    src="/images/hero.jpg"
    width="800"
    height="400"
    format="webp"
    quality="80"
    loading="lazy"
  />
</template>
```

### 3. 程式碼分割

```vue
<!-- 延遲載入大型組件 -->
<template>
  <div>
    <LazyHeavyComponent v-if="showComponent" />
  </div>
</template>
```

### 4. 設定適當的快取標頭

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  nitro: {
    routeRules: {
      // HTML 檔案 - 短期快取
      '/**': {
        headers: {
          'cache-control': 'public, max-age=3600, must-revalidate'
        }
      },
      // 靜態資源 - 長期快取
      '/_nuxt/**': {
        headers: {
          'cache-control': 'public, max-age=31536000, immutable'
        }
      }
    }
  }
})
```

### 5. 提供 404 頁面

```vue
<!-- error.vue -->
<template>
  <div class="error-page">
    <h1 v-if="error.statusCode === 404">頁面不存在</h1>
    <h1 v-else>發生錯誤</h1>
    <p>{{ error.message }}</p>
    <NuxtLink to="/">回到首頁</NuxtLink>
  </div>
</template>

<script setup>
const props = defineProps({
  error: Object
})
</script>
```

### 6. 使用 Sitemap

```bash
# 安裝 sitemap 模組
npm install @nuxtjs/sitemap
```

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  modules: ['@nuxtjs/sitemap'],

  sitemap: {
    hostname: 'https://example.com',
    gzip: true,
    routes: async () => {
      // 動態生成 sitemap 路由
      const posts = await fetch('https://api.example.com/posts')
        .then(r => r.json())
      return posts.map(post => `/blog/${post.slug}`)
    }
  }
})
```

## 完整建置腳本

```json
// package.json
{
  "scripts": {
    "dev": "nuxi dev",
    "build": "nuxi build",
    "generate": "nuxi generate",
    "preview": "nuxi preview",
    "deploy:netlify": "npm run generate && netlify deploy --prod --dir=.output/public",
    "deploy:vercel": "vercel --prod",
    "analyze": "nuxi analyze"
  }
}
```

## 總結

SSG 是最適合靜態內容網站的渲染策略：

### 優勢
- ⚡ 極快的載入速度（CDN 分發）
- 💰 極低的運行成本（無需伺服器）
- 🔒 更高的安全性（沒有伺服器端攻擊面）
- 📈 優秀的 SEO 表現
- ♾️ 幾乎無限的擴展性

### 限制
- 🔄 內容更新需要重新建置
- ⏱️ 大型網站建置時間長
- 📊 不適合即時性內容

### 選擇 SSG 的時機
- ✅ 部落格、文件、行銷網站
- ✅ 內容變動頻率低（每天或每週）
- ✅ 路由數量有限（< 10,000 頁）
- ✅ 追求極致效能和低成本
- ❌ 避免使用於即時性或大量動態內容

下一章我們將探討 **Hybrid Rendering**，學習如何混合使用不同的渲染策略！
