# 10 - SEO 優化與 Meta 標籤管理

## 概述

SEO（搜尋引擎優化）對於現代網站至關重要。Nuxt 3 提供了強大的 SEO 工具，包括 `useHead`、`useSeoMeta` 等 composables，讓你輕鬆管理頁面的 meta 標籤、Open Graph 資訊和結構化資料。

## 1. useHead composable

`useHead` 是管理頁面 `<head>` 標籤最基本的方式。

### 基本使用

```vue
<script setup lang="ts">
useHead({
  title: '我的網站標題',
  meta: [
    { name: 'description', content: '這是網站的描述' },
    { name: 'keywords', content: 'Nuxt, Vue, JavaScript' }
  ],
  link: [
    { rel: 'icon', type: 'image/x-icon', href: '/favicon.ico' }
  ]
})
</script>
```

### 動態標題

```vue
<script setup lang="ts">
const title = ref('首頁')

useHead({
  title: title,
  titleTemplate: '%s - 我的網站' // 結果：首頁 - 我的網站
})

// 動態更新標題
const updateTitle = (newTitle: string) => {
  title.value = newTitle
}
</script>
```

### 響應式 Meta 標籤

```vue
<script setup lang="ts">
const route = useRoute()

const title = computed(() => {
  const pageTitles: Record<string, string> = {
    '/': '首頁',
    '/about': '關於我們',
    '/contact': '聯絡我們'
  }
  return pageTitles[route.path] || '頁面'
})

useHead({
  title,
  meta: [
    {
      name: 'description',
      content: computed(() => `這是${title.value}的描述`)
    }
  ]
})
</script>
```

### Script 標籤管理

```vue
<script setup lang="ts">
useHead({
  script: [
    // Google Analytics
    {
      src: 'https://www.googletagmanager.com/gtag/js?id=GA_MEASUREMENT_ID',
      async: true
    },
    {
      children: `
        window.dataLayer = window.dataLayer || [];
        function gtag(){dataLayer.push(arguments);}
        gtag('js', new Date());
        gtag('config', 'GA_MEASUREMENT_ID');
      `
    },
    // Facebook Pixel
    {
      children: `
        !function(f,b,e,v,n,t,s)
        {if(f.fbq)return;n=f.fbq=function(){n.callMethod?
        n.callMethod.apply(n,arguments):n.queue.push(arguments)};
        if(!f._fbq)f._fbq=n;n.push=n;n.loaded=!0;n.version='2.0';
        n.queue=[];t=b.createElement(e);t.async=!0;
        t.src=v;s=b.getElementsByTagName(e)[0];
        s.parentNode.insertBefore(t,s)}(window, document,'script',
        'https://connect.facebook.net/en_US/fbevents.js');
        fbq('init', 'YOUR_PIXEL_ID');
        fbq('track', 'PageView');
      `,
      type: 'text/javascript'
    }
  ],
  noscript: [
    {
      children: `<img height="1" width="1" style="display:none"
        src="https://www.facebook.com/tr?id=YOUR_PIXEL_ID&ev=PageView&noscript=1" />`
    }
  ]
})
</script>
```

### Body 屬性

```vue
<script setup lang="ts">
const isDark = ref(false)

useHead({
  bodyAttrs: {
    class: computed(() => isDark.value ? 'dark-mode' : 'light-mode')
  },
  htmlAttrs: {
    lang: 'zh-TW',
    dir: 'ltr'
  }
})
</script>
```

## 2. useSeoMeta 使用

`useSeoMeta` 是專門為 SEO 優化設計的 composable，提供了更好的類型提示。

### 基本 SEO Meta

```vue
<script setup lang="ts">
useSeoMeta({
  title: '我的網站',
  description: '提供優質的產品和服務',
  keywords: 'Nuxt, Vue, SEO, 網站開發',
  author: '公司名稱',
  robots: 'index, follow',

  // 視口設定
  viewport: 'width=device-width, initial-scale=1.0',

  // 字元編碼
  charset: 'utf-8'
})
</script>
```

### Open Graph（Facebook、LinkedIn 等）

```vue
<script setup lang="ts">
useSeoMeta({
  ogTitle: '文章標題',
  ogDescription: '文章的簡短描述',
  ogImage: 'https://example.com/image.jpg',
  ogImageAlt: '文章封面圖片',
  ogImageWidth: '1200',
  ogImageHeight: '630',
  ogUrl: 'https://example.com/article',
  ogType: 'article',
  ogSiteName: '我的網站',
  ogLocale: 'zh_TW',

  // 文章特定資訊
  articlePublishedTime: '2024-01-15T10:00:00Z',
  articleModifiedTime: '2024-01-16T14:30:00Z',
  articleAuthor: ['作者名稱'],
  articleSection: '技術',
  articleTag: ['Nuxt', 'Vue', 'SEO']
})
</script>
```

### Twitter Card

```vue
<script setup lang="ts">
useSeoMeta({
  twitterCard: 'summary_large_image',
  twitterSite: '@mywebsite',
  twitterCreator: '@author',
  twitterTitle: '文章標題',
  twitterDescription: '文章描述',
  twitterImage: 'https://example.com/twitter-image.jpg',
  twitterImageAlt: 'Twitter 卡片圖片'
})
</script>
```

### 組合使用

```vue
<script setup lang="ts">
const route = useRoute()

const seoData = {
  title: '產品名稱',
  description: '產品的詳細描述，包含主要特色和優勢',
  image: 'https://example.com/product.jpg',
  url: `https://example.com${route.path}`
}

useSeoMeta({
  // 基本 Meta
  title: seoData.title,
  description: seoData.description,

  // Open Graph
  ogTitle: seoData.title,
  ogDescription: seoData.description,
  ogImage: seoData.image,
  ogUrl: seoData.url,
  ogType: 'website',

  // Twitter Card
  twitterCard: 'summary_large_image',
  twitterTitle: seoData.title,
  twitterDescription: seoData.description,
  twitterImage: seoData.image
})
</script>
```

## 3. 動態 Meta 標籤

### 根據 API 資料設定 Meta

```vue
<script setup lang="ts">
interface Article {
  title: string
  excerpt: string
  content: string
  image: string
  author: {
    name: string
    twitter: string
  }
  publishedAt: string
  updatedAt: string
  tags: string[]
}

const route = useRoute()
const slug = route.params.slug as string

const { data: article } = await useFetch<Article>(`/api/articles/${slug}`)

// 動態設定 SEO
watchEffect(() => {
  if (article.value) {
    useSeoMeta({
      title: article.value.title,
      description: article.value.excerpt,

      // Open Graph
      ogTitle: article.value.title,
      ogDescription: article.value.excerpt,
      ogImage: article.value.image,
      ogType: 'article',

      // 文章特定資訊
      articlePublishedTime: article.value.publishedAt,
      articleModifiedTime: article.value.updatedAt,
      articleAuthor: [article.value.author.name],
      articleTag: article.value.tags,

      // Twitter
      twitterCard: 'summary_large_image',
      twitterCreator: article.value.author.twitter
    })
  }
})
</script>
```

### 產品頁面 SEO

```vue
<script setup lang="ts">
interface Product {
  id: number
  name: string
  description: string
  price: number
  currency: string
  images: string[]
  category: string
  brand: string
  inStock: boolean
  rating: number
  reviews: number
}

const route = useRoute()
const { data: product } = await useFetch<Product>(`/api/products/${route.params.id}`)

watchEffect(() => {
  if (product.value) {
    const structuredData = {
      '@context': 'https://schema.org',
      '@type': 'Product',
      name: product.value.name,
      description: product.value.description,
      image: product.value.images,
      brand: {
        '@type': 'Brand',
        name: product.value.brand
      },
      offers: {
        '@type': 'Offer',
        price: product.value.price,
        priceCurrency: product.value.currency,
        availability: product.value.inStock
          ? 'https://schema.org/InStock'
          : 'https://schema.org/OutOfStock'
      },
      aggregateRating: {
        '@type': 'AggregateRating',
        ratingValue: product.value.rating,
        reviewCount: product.value.reviews
      }
    }

    useSeoMeta({
      title: `${product.value.name} - ${product.value.brand}`,
      description: product.value.description,
      ogTitle: product.value.name,
      ogDescription: product.value.description,
      ogImage: product.value.images[0],
      ogType: 'product',
      productPrice: product.value.price.toString(),
      productCurrency: product.value.currency
    })

    useHead({
      script: [
        {
          type: 'application/ld+json',
          children: JSON.stringify(structuredData)
        }
      ]
    })
  }
})
</script>
```

## 4. Open Graph 設定

### 完整的 Open Graph 範例

```vue
<script setup lang="ts">
useSeoMeta({
  // 基本 OG 標籤
  ogTitle: '精選商品 - 限時優惠',
  ogDescription: '最新商品上架，現在購買享有特別折扣',
  ogImage: 'https://example.com/og-image.jpg',
  ogImageSecureUrl: 'https://example.com/og-image.jpg',
  ogImageWidth: '1200',
  ogImageHeight: '630',
  ogImageType: 'image/jpeg',
  ogImageAlt: '商品展示圖',

  // 網站資訊
  ogUrl: 'https://example.com/products',
  ogType: 'website',
  ogSiteName: '我的電商網站',
  ogLocale: 'zh_TW',
  ogLocaleAlternate: ['en_US', 'ja_JP'],

  // 音頻/視頻（如果適用）
  // ogVideo: 'https://example.com/video.mp4',
  // ogAudio: 'https://example.com/audio.mp3'
})
</script>
```

### 多圖片 Open Graph

```vue
<script setup lang="ts">
const product = {
  name: '產品名稱',
  images: [
    'https://example.com/image1.jpg',
    'https://example.com/image2.jpg',
    'https://example.com/image3.jpg'
  ]
}

useHead({
  meta: [
    { property: 'og:title', content: product.name },
    // 多張圖片
    ...product.images.map(img => ({
      property: 'og:image',
      content: img
    }))
  ]
})
</script>
```

## 5. Twitter Card 設定

### Summary Card（小圖）

```vue
<script setup lang="ts">
useSeoMeta({
  twitterCard: 'summary',
  twitterSite: '@mywebsite',
  twitterCreator: '@author',
  twitterTitle: '文章標題',
  twitterDescription: '文章簡短描述',
  twitterImage: 'https://example.com/image-square.jpg'
})
</script>
```

### Summary Large Image Card（大圖）

```vue
<script setup lang="ts">
useSeoMeta({
  twitterCard: 'summary_large_image',
  twitterSite: '@mywebsite',
  twitterCreator: '@author',
  twitterTitle: '視覺豐富的內容',
  twitterDescription: '使用大圖展示更多細節',
  twitterImage: 'https://example.com/large-image.jpg',
  twitterImageAlt: '大圖的替代文字'
})
</script>
```

### App Card（應用程式）

```vue
<script setup lang="ts">
useHead({
  meta: [
    { name: 'twitter:card', content: 'app' },
    { name: 'twitter:site', content: '@myapp' },
    { name: 'twitter:description', content: '下載我們的應用程式' },
    { name: 'twitter:app:name:iphone', content: 'My App' },
    { name: 'twitter:app:id:iphone', content: '123456789' },
    { name: 'twitter:app:name:ipad', content: 'My App' },
    { name: 'twitter:app:id:ipad', content: '123456789' },
    { name: 'twitter:app:name:googleplay', content: 'My App' },
    { name: 'twitter:app:id:googleplay', content: 'com.myapp.android' }
  ]
})
</script>
```

## 6. Sitemap 生成

### 安裝 Sitemap 模組

```bash
npm install @nuxtjs/sitemap
```

### 配置 Sitemap

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  modules: ['@nuxtjs/sitemap'],

  sitemap: {
    hostname: 'https://example.com',
    gzip: true,
    exclude: [
      '/admin/**',
      '/private/**'
    ],
    routes: async () => {
      // 動態路由
      const { data: posts } = await $fetch<{ data: Post[] }>('/api/posts')
      return posts.data.map(post => ({
        url: `/blog/${post.slug}`,
        changefreq: 'weekly',
        priority: 0.8,
        lastmod: post.updatedAt
      }))
    }
  }
})
```

### 手動生成 Sitemap

```typescript
// server/routes/sitemap.xml.ts
export default defineEventHandler(async (event) => {
  const posts = await $fetch<Post[]>('/api/posts')
  const products = await $fetch<Product[]>('/api/products')

  const sitemap = `<?xml version="1.0" encoding="UTF-8"?>
    <urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">
      <url>
        <loc>https://example.com/</loc>
        <changefreq>daily</changefreq>
        <priority>1.0</priority>
      </url>
      ${posts.map(post => `
        <url>
          <loc>https://example.com/blog/${post.slug}</loc>
          <lastmod>${post.updatedAt}</lastmod>
          <changefreq>weekly</changefreq>
          <priority>0.8</priority>
        </url>
      `).join('')}
      ${products.map(product => `
        <url>
          <loc>https://example.com/products/${product.id}</loc>
          <lastmod>${product.updatedAt}</lastmod>
          <changefreq>daily</changefreq>
          <priority>0.9</priority>
        </url>
      `).join('')}
    </urlset>
  `

  event.node.res.setHeader('Content-Type', 'application/xml')
  return sitemap
})
```

### 多語言 Sitemap

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  sitemap: {
    hostname: 'https://example.com',
    i18n: {
      locales: ['zh-TW', 'en', 'ja'],
      routesNameSeparator: '___'
    },
    routes: async () => {
      const posts = await $fetch('/api/posts')
      const locales = ['zh-TW', 'en', 'ja']

      return posts.flatMap(post =>
        locales.map(locale => ({
          url: `/${locale}/blog/${post.slug}`,
          links: locales.map(l => ({
            lang: l,
            url: `/${l}/blog/${post.slug}`
          }))
        }))
      )
    }
  }
})
```

## 7. robots.txt 設定

### 基本 robots.txt

```typescript
// server/routes/robots.txt.ts
export default defineEventHandler((event) => {
  const robotsTxt = `
User-agent: *
Allow: /

# 禁止索引的路徑
Disallow: /admin/
Disallow: /api/
Disallow: /private/

# Sitemap
Sitemap: https://example.com/sitemap.xml
  `.trim()

  event.node.res.setHeader('Content-Type', 'text/plain')
  return robotsTxt
})
```

### 環境特定的 robots.txt

```typescript
// server/routes/robots.txt.ts
export default defineEventHandler((event) => {
  const config = useRuntimeConfig()
  const isProduction = process.env.NODE_ENV === 'production'

  const robotsTxt = isProduction
    ? `
User-agent: *
Allow: /

Disallow: /admin/
Disallow: /api/

Sitemap: ${config.public.siteUrl}/sitemap.xml
    `.trim()
    : `
# 開發環境 - 禁止所有爬蟲
User-agent: *
Disallow: /
    `.trim()

  event.node.res.setHeader('Content-Type', 'text/plain')
  return robotsTxt
})
```

### 進階 robots.txt

```typescript
export default defineEventHandler(() => {
  return `
# Google
User-agent: Googlebot
Allow: /
Crawl-delay: 0

# Bing
User-agent: Bingbot
Allow: /
Crawl-delay: 1

# 其他爬蟲
User-agent: *
Allow: /

# 禁止路徑
Disallow: /admin/
Disallow: /api/
Disallow: /private/
Disallow: /*.json$
Disallow: /*?*sort=
Disallow: /*?*filter=

# 允許的路徑
Allow: /api/public/

# Sitemap
Sitemap: https://example.com/sitemap.xml
Sitemap: https://example.com/sitemap-images.xml
Sitemap: https://example.com/sitemap-videos.xml
  `.trim()
})
```

## 8. 完整 SEO 範例

### 部落格文章頁面

```vue
<!-- pages/blog/[slug].vue -->
<template>
  <article v-if="post" class="blog-post">
    <header>
      <h1>{{ post.title }}</h1>
      <div class="meta">
        <time :datetime="post.publishedAt">
          {{ formatDate(post.publishedAt) }}
        </time>
        <span>作者：{{ post.author.name }}</span>
      </div>
    </header>

    <img :src="post.image" :alt="post.title" class="cover-image" />

    <div class="content" v-html="post.content"></div>

    <footer>
      <div class="tags">
        <span v-for="tag in post.tags" :key="tag" class="tag">
          #{{ tag }}
        </span>
      </div>
    </footer>
  </article>
</template>

<script setup lang="ts">
interface Author {
  name: string
  bio: string
  avatar: string
  twitter?: string
}

interface Post {
  id: number
  slug: string
  title: string
  excerpt: string
  content: string
  image: string
  author: Author
  category: string
  tags: string[]
  publishedAt: string
  updatedAt: string
  readingTime: number
}

const route = useRoute()
const config = useRuntimeConfig()
const slug = route.params.slug as string

const { data: post } = await useFetch<Post>(`/api/posts/${slug}`)

if (!post.value) {
  throw createError({
    statusCode: 404,
    message: '文章不存在'
  })
}

// 完整的 SEO 設定
const siteUrl = config.public.siteUrl || 'https://example.com'
const postUrl = `${siteUrl}/blog/${post.value.slug}`

// 結構化資料（Schema.org）
const structuredData = {
  '@context': 'https://schema.org',
  '@type': 'BlogPosting',
  headline: post.value.title,
  description: post.value.excerpt,
  image: post.value.image,
  author: {
    '@type': 'Person',
    name: post.value.author.name,
    description: post.value.author.bio,
    image: post.value.author.avatar
  },
  publisher: {
    '@type': 'Organization',
    name: '我的部落格',
    logo: {
      '@type': 'ImageObject',
      url: `${siteUrl}/logo.png`
    }
  },
  datePublished: post.value.publishedAt,
  dateModified: post.value.updatedAt,
  mainEntityOfPage: {
    '@type': 'WebPage',
    '@id': postUrl
  },
  keywords: post.value.tags.join(', ')
}

// Breadcrumb 結構化資料
const breadcrumbData = {
  '@context': 'https://schema.org',
  '@type': 'BreadcrumbList',
  itemListElement: [
    {
      '@type': 'ListItem',
      position: 1,
      name: '首頁',
      item: siteUrl
    },
    {
      '@type': 'ListItem',
      position: 2,
      name: '部落格',
      item: `${siteUrl}/blog`
    },
    {
      '@type': 'ListItem',
      position: 3,
      name: post.value.title,
      item: postUrl
    }
  ]
}

// SEO Meta 標籤
useSeoMeta({
  // 基本資訊
  title: post.value.title,
  description: post.value.excerpt,
  author: post.value.author.name,

  // Open Graph
  ogTitle: post.value.title,
  ogDescription: post.value.excerpt,
  ogImage: post.value.image,
  ogImageAlt: post.value.title,
  ogImageWidth: '1200',
  ogImageHeight: '630',
  ogUrl: postUrl,
  ogType: 'article',
  ogSiteName: '我的部落格',
  ogLocale: 'zh_TW',

  // 文章特定
  articlePublishedTime: post.value.publishedAt,
  articleModifiedTime: post.value.updatedAt,
  articleAuthor: [post.value.author.name],
  articleSection: post.value.category,
  articleTag: post.value.tags,

  // Twitter Card
  twitterCard: 'summary_large_image',
  twitterSite: '@myblog',
  twitterCreator: post.value.author.twitter || '@myblog',
  twitterTitle: post.value.title,
  twitterDescription: post.value.excerpt,
  twitterImage: post.value.image,
  twitterImageAlt: post.value.title
})

// Head 標籤（包含結構化資料）
useHead({
  title: post.value.title,
  titleTemplate: '%s - 我的部落格',
  link: [
    { rel: 'canonical', href: postUrl }
  ],
  script: [
    // BlogPosting 結構化資料
    {
      type: 'application/ld+json',
      children: JSON.stringify(structuredData)
    },
    // Breadcrumb 結構化資料
    {
      type: 'application/ld+json',
      children: JSON.stringify(breadcrumbData)
    }
  ]
})

const formatDate = (dateString: string) => {
  return new Date(dateString).toLocaleDateString('zh-TW', {
    year: 'numeric',
    month: 'long',
    day: 'numeric'
  })
}
</script>
```

### 電商產品頁面

```vue
<!-- pages/products/[id].vue -->
<script setup lang="ts">
interface Product {
  id: number
  name: string
  description: string
  price: number
  currency: string
  images: string[]
  brand: string
  category: string
  sku: string
  inStock: boolean
  stockQuantity: number
  rating: number
  reviewCount: number
  reviews: Review[]
}

interface Review {
  author: string
  rating: number
  comment: string
  date: string
}

const route = useRoute()
const config = useRuntimeConfig()
const productId = route.params.id as string

const { data: product } = await useFetch<Product>(`/api/products/${productId}`)

if (!product.value) {
  throw createError({ statusCode: 404, message: '產品不存在' })
}

const siteUrl = config.public.siteUrl
const productUrl = `${siteUrl}/products/${product.value.id}`

// Product 結構化資料
const productSchema = {
  '@context': 'https://schema.org',
  '@type': 'Product',
  name: product.value.name,
  description: product.value.description,
  image: product.value.images,
  brand: {
    '@type': 'Brand',
    name: product.value.brand
  },
  sku: product.value.sku,
  offers: {
    '@type': 'Offer',
    url: productUrl,
    priceCurrency: product.value.currency,
    price: product.value.price,
    availability: product.value.inStock
      ? 'https://schema.org/InStock'
      : 'https://schema.org/OutOfStock',
    seller: {
      '@type': 'Organization',
      name: '我的商店'
    }
  },
  aggregateRating: {
    '@type': 'AggregateRating',
    ratingValue: product.value.rating,
    reviewCount: product.value.reviewCount
  },
  review: product.value.reviews.map(review => ({
    '@type': 'Review',
    author: {
      '@type': 'Person',
      name: review.author
    },
    datePublished: review.date,
    reviewRating: {
      '@type': 'Rating',
      ratingValue: review.rating
    },
    reviewBody: review.comment
  }))
}

useSeoMeta({
  title: `${product.value.name} - ${product.value.brand}`,
  description: product.value.description,

  ogTitle: product.value.name,
  ogDescription: product.value.description,
  ogImage: product.value.images[0],
  ogType: 'product',
  ogUrl: productUrl,

  productPrice: product.value.price.toString(),
  productCurrency: product.value.currency,
  productAvailability: product.value.inStock ? 'in stock' : 'out of stock',

  twitterCard: 'summary_large_image',
  twitterTitle: product.value.name,
  twitterDescription: product.value.description,
  twitterImage: product.value.images[0]
})

useHead({
  link: [
    { rel: 'canonical', href: productUrl }
  ],
  script: [
    {
      type: 'application/ld+json',
      children: JSON.stringify(productSchema)
    }
  ]
})
</script>

<template>
  <div v-if="product" class="product-page">
    <div class="product-images">
      <img
        v-for="(image, index) in product.images"
        :key="index"
        :src="image"
        :alt="`${product.name} - 圖片 ${index + 1}`"
      />
    </div>

    <div class="product-info">
      <h1>{{ product.name }}</h1>
      <p class="brand">{{ product.brand }}</p>
      <p class="price">{{ product.currency }} {{ product.price }}</p>

      <div class="rating">
        ⭐ {{ product.rating }} / 5 ({{ product.reviewCount }} 則評價)
      </div>

      <p class="description">{{ product.description }}</p>

      <div class="stock-info">
        <span v-if="product.inStock" class="in-stock">
          庫存：{{ product.stockQuantity }} 件
        </span>
        <span v-else class="out-of-stock">缺貨中</span>
      </div>

      <button :disabled="!product.inStock" class="add-to-cart">
        加入購物車
      </button>
    </div>
  </div>
</template>
```

## 最佳實踐建議

### 1. 標題優化

```vue
<script setup lang="ts">
// ✅ 好 - 描述性且包含關鍵字
useHead({
  title: 'Nuxt 3 完整教學 - 從入門到精通',
  titleTemplate: '%s | 我的技術部落格'
})

// ❌ 不好 - 太籠統
useHead({
  title: '教學'
})
</script>
```

### 2. Meta 描述

```vue
<script setup lang="ts">
// ✅ 好 - 150-160 字元，包含關鍵字和呼籲行動
useSeoMeta({
  description: '學習 Nuxt 3 的完整指南，包含 SEO 優化、效能調校和最佳實踐。適合想要掌握現代 Vue.js 框架的開發者。立即開始學習！'
})

// ❌ 不好 - 太短且不具描述性
useSeoMeta({
  description: 'Nuxt 教學'
})
</script>
```

### 3. 圖片優化

```vue
<script setup lang="ts">
// ✅ 好 - 使用適當尺寸和格式
useSeoMeta({
  ogImage: 'https://example.com/og-image-1200x630.jpg', // 1200x630px
  ogImageWidth: '1200',
  ogImageHeight: '630',
  ogImageAlt: '詳細描述圖片內容'
})

// Twitter 建議 2:1 比例
useSeoMeta({
  twitterImage: 'https://example.com/twitter-1200x600.jpg'
})
</script>
```

### 4. 結構化資料

```typescript
// ✅ 好 - 完整的結構化資料
const structuredData = {
  '@context': 'https://schema.org',
  '@type': 'FAQPage',
  mainEntity: [
    {
      '@type': 'Question',
      name: '什麼是 Nuxt 3？',
      acceptedAnswer: {
        '@type': 'Answer',
        text: 'Nuxt 3 是基於 Vue 3 的全端框架...'
      }
    }
  ]
}
```

### 5. Canonical URL

```vue
<script setup lang="ts">
// 避免重複內容
const canonicalUrl = 'https://example.com/original-page'

useHead({
  link: [
    { rel: 'canonical', href: canonicalUrl }
  ]
})
</script>
```

## 總結

Nuxt 3 提供了完整的 SEO 工具：

✅ **useHead** - 靈活的 head 標籤管理
✅ **useSeoMeta** - 專門的 SEO meta 標籤
✅ **結構化資料** - Schema.org 支援
✅ **Sitemap** - 自動生成網站地圖
✅ **robots.txt** - 爬蟲控制
✅ **Open Graph** - 社交媒體優化
✅ **Twitter Cards** - Twitter 展示優化

掌握這些 SEO 技巧，將大幅提升你的網站在搜尋引擎的排名和社交媒體的展示效果！
