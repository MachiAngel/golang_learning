# 第 16 章：SEO 優化與 Meta Tags 管理

## 概述

SEO（搜尋引擎優化）對於網站的成功至關重要。Next.js 提供了強大的 SEO 工具，特別是內建的 `next/head` 組件。對於 Vue3 開發者來說，這類似於 vue-meta 或 Nuxt.js 的 head() 方法，但 Next.js 的實作更加簡潔直觀。

## Vue3 vs Next.js SEO 處理

### Vue3 的做法

```javascript
// Vue3 with vue-meta
export default {
  metaInfo: {
    title: '頁面標題',
    meta: [
      { name: 'description', content: '頁面描述' },
      { property: 'og:title', content: '頁面標題' }
    ]
  }
}

// Nuxt.js
export default {
  head() {
    return {
      title: '頁面標題',
      meta: [
        { hid: 'description', name: 'description', content: '頁面描述' }
      ]
    }
  }
}
```

### Next.js 的做法

```javascript
import Head from 'next/head'

export default function Page() {
  return (
    <>
      <Head>
        <title>頁面標題</title>
        <meta name="description" content="頁面描述" />
        <meta property="og:title" content="頁面標題" />
      </Head>

      <main>
        {/* 頁面內容 */}
      </main>
    </>
  )
}
```

## next/head 基本使用

### 基本 Meta Tags

```javascript
import Head from 'next/head'

export default function Page() {
  return (
    <>
      <Head>
        {/* 標題 */}
        <title>我的網站 - 首頁</title>

        {/* 基本 Meta */}
        <meta name="description" content="這是一個很棒的網站" />
        <meta name="keywords" content="Next.js, React, SEO, 教學" />
        <meta name="author" content="Your Name" />

        {/* Viewport */}
        <meta name="viewport" content="width=device-width, initial-scale=1" />

        {/* 字元編碼 */}
        <meta charSet="utf-8" />

        {/* Favicon */}
        <link rel="icon" href="/favicon.ico" />
        <link rel="apple-touch-icon" sizes="180x180" href="/apple-touch-icon.png" />

        {/* 語言 */}
        <meta httpEquiv="content-language" content="zh-TW" />
      </Head>

      <main>
        <h1>歡迎來到我的網站</h1>
      </main>
    </>
  )
}
```

### Open Graph (Facebook, LinkedIn)

```javascript
import Head from 'next/head'

export default function BlogPost({ post }) {
  return (
    <>
      <Head>
        <title>{post.title} - 部落格</title>
        <meta name="description" content={post.excerpt} />

        {/* Open Graph */}
        <meta property="og:type" content="article" />
        <meta property="og:title" content={post.title} />
        <meta property="og:description" content={post.excerpt} />
        <meta property="og:image" content={post.coverImage} />
        <meta property="og:url" content={`https://example.com/blog/${post.slug}`} />
        <meta property="og:site_name" content="我的部落格" />

        {/* 文章特定 */}
        <meta property="article:published_time" content={post.publishedAt} />
        <meta property="article:author" content={post.author.name} />
        <meta property="article:section" content={post.category} />
        <meta property="article:tag" content={post.tags.join(', ')} />
      </Head>

      <article>
        <h1>{post.title}</h1>
        <div dangerouslySetInnerHTML={{ __html: post.content }} />
      </article>
    </>
  )
}
```

### Twitter Card

```javascript
import Head from 'next/head'

export default function Product({ product }) {
  return (
    <>
      <Head>
        <title>{product.name} - 商品</title>
        <meta name="description" content={product.description} />

        {/* Twitter Card */}
        <meta name="twitter:card" content="summary_large_image" />
        <meta name="twitter:site" content="@yourhandle" />
        <meta name="twitter:creator" content="@yourhandle" />
        <meta name="twitter:title" content={product.name} />
        <meta name="twitter:description" content={product.description} />
        <meta name="twitter:image" content={product.image} />
      </Head>

      <div>
        <h1>{product.name}</h1>
        <p>{product.description}</p>
        <p>NT$ {product.price}</p>
      </div>
    </>
  )
}
```

## 建立可重用的 SEO 組件

### SEO 組件

```javascript
// components/SEO.js
import Head from 'next/head'

export default function SEO({
  title,
  description,
  image,
  url,
  type = 'website',
  author,
  publishedTime,
  keywords,
  noindex = false
}) {
  const siteTitle = '我的網站'
  const fullTitle = title ? `${title} | ${siteTitle}` : siteTitle
  const siteUrl = 'https://example.com'
  const fullUrl = url ? `${siteUrl}${url}` : siteUrl
  const defaultImage = `${siteUrl}/og-image.jpg`
  const ogImage = image || defaultImage

  return (
    <Head>
      {/* 基本 Meta */}
      <title>{fullTitle}</title>
      <meta name="description" content={description} />
      {keywords && <meta name="keywords" content={keywords} />}
      {author && <meta name="author" content={author} />}

      {/* Canonical URL */}
      <link rel="canonical" href={fullUrl} />

      {/* Robots */}
      {noindex && <meta name="robots" content="noindex, nofollow" />}

      {/* Open Graph */}
      <meta property="og:type" content={type} />
      <meta property="og:title" content={fullTitle} />
      <meta property="og:description" content={description} />
      <meta property="og:image" content={ogImage} />
      <meta property="og:url" content={fullUrl} />
      <meta property="og:site_name" content={siteTitle} />

      {/* 文章特定 */}
      {type === 'article' && publishedTime && (
        <meta property="article:published_time" content={publishedTime} />
      )}
      {type === 'article' && author && (
        <meta property="article:author" content={author} />
      )}

      {/* Twitter Card */}
      <meta name="twitter:card" content="summary_large_image" />
      <meta name="twitter:title" content={fullTitle} />
      <meta name="twitter:description" content={description} />
      <meta name="twitter:image" content={ogImage} />
    </Head>
  )
}
```

### 使用 SEO 組件

```javascript
// pages/blog/[slug].js
import SEO from '../../components/SEO'

export default function BlogPost({ post }) {
  return (
    <>
      <SEO
        title={post.title}
        description={post.excerpt}
        image={post.coverImage}
        url={`/blog/${post.slug}`}
        type="article"
        author={post.author.name}
        publishedTime={post.publishedAt}
        keywords={post.tags.join(', ')}
      />

      <article>
        <h1>{post.title}</h1>
        <div dangerouslySetInnerHTML={{ __html: post.content }} />
      </article>
    </>
  )
}
```

## Structured Data (結構化資料)

### JSON-LD 基礎

```javascript
import Head from 'next/head'

export default function Article({ post }) {
  const structuredData = {
    '@context': 'https://schema.org',
    '@type': 'Article',
    headline: post.title,
    description: post.excerpt,
    image: post.coverImage,
    datePublished: post.publishedAt,
    dateModified: post.updatedAt,
    author: {
      '@type': 'Person',
      name: post.author.name,
      url: `https://example.com/authors/${post.author.slug}`
    },
    publisher: {
      '@type': 'Organization',
      name: '我的網站',
      logo: {
        '@type': 'ImageObject',
        url: 'https://example.com/logo.png'
      }
    }
  }

  return (
    <>
      <Head>
        <script
          type="application/ld+json"
          dangerouslySetInnerHTML={{ __html: JSON.stringify(structuredData) }}
        />
      </Head>

      <article>
        <h1>{post.title}</h1>
        <div dangerouslySetInnerHTML={{ __html: post.content }} />
      </article>
    </>
  )
}
```

### 產品結構化資料

```javascript
export default function Product({ product }) {
  const structuredData = {
    '@context': 'https://schema.org',
    '@type': 'Product',
    name: product.name,
    description: product.description,
    image: product.images,
    brand: {
      '@type': 'Brand',
      name: product.brand
    },
    offers: {
      '@type': 'Offer',
      url: `https://example.com/products/${product.id}`,
      priceCurrency: 'TWD',
      price: product.price,
      availability: product.inStock
        ? 'https://schema.org/InStock'
        : 'https://schema.org/OutOfStock',
      seller: {
        '@type': 'Organization',
        name: '我的商店'
      }
    },
    aggregateRating: {
      '@type': 'AggregateRating',
      ratingValue: product.rating,
      reviewCount: product.reviewCount
    }
  }

  return (
    <>
      <Head>
        <title>{product.name} - 商品</title>
        <script
          type="application/ld+json"
          dangerouslySetInnerHTML={{ __html: JSON.stringify(structuredData) }}
        />
      </Head>

      <div>
        <h1>{product.name}</h1>
        <p>{product.description}</p>
        <p>NT$ {product.price}</p>
      </div>
    </>
  )
}
```

### 麵包屑結構化資料

```javascript
export default function Category({ category, breadcrumbs }) {
  const breadcrumbList = {
    '@context': 'https://schema.org',
    '@type': 'BreadcrumbList',
    itemListElement: breadcrumbs.map((crumb, index) => ({
      '@type': 'ListItem',
      position: index + 1,
      name: crumb.name,
      item: `https://example.com${crumb.path}`
    }))
  }

  return (
    <>
      <Head>
        <script
          type="application/ld+json"
          dangerouslySetInnerHTML={{ __html: JSON.stringify(breadcrumbList) }}
        />
      </Head>

      <nav>
        {breadcrumbs.map((crumb, index) => (
          <span key={index}>
            {index > 0 && ' / '}
            <a href={crumb.path}>{crumb.name}</a>
          </span>
        ))}
      </nav>

      <h1>{category.name}</h1>
    </>
  )
}
```

## robots.txt 和 sitemap.xml

### 建立 robots.txt

```javascript
// pages/robots.txt.js
export default function Robots() {
  // 這個組件不會被渲染
  return null
}

export async function getServerSideProps({ res }) {
  const robotsTxt = `
User-agent: *
Allow: /
Disallow: /admin/
Disallow: /api/

Sitemap: https://example.com/sitemap.xml
  `.trim()

  res.setHeader('Content-Type', 'text/plain')
  res.write(robotsTxt)
  res.end()

  return {
    props: {}
  }
}
```

### 建立動態 Sitemap

```javascript
// pages/sitemap.xml.js
export default function Sitemap() {
  return null
}

export async function getServerSideProps({ res }) {
  // 獲取所有頁面
  const posts = await getAllPosts()
  const products = await getAllProducts()

  const baseUrl = 'https://example.com'

  const staticPages = [
    '',
    '/about',
    '/contact',
    '/blog',
    '/products'
  ]

  const sitemap = `<?xml version="1.0" encoding="UTF-8"?>
<urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">
  ${staticPages.map(path => `
    <url>
      <loc>${baseUrl}${path}</loc>
      <lastmod>${new Date().toISOString()}</lastmod>
      <changefreq>daily</changefreq>
      <priority>0.8</priority>
    </url>
  `).join('')}

  ${posts.map(post => `
    <url>
      <loc>${baseUrl}/blog/${post.slug}</loc>
      <lastmod>${post.updatedAt}</lastmod>
      <changefreq>weekly</changefreq>
      <priority>0.7</priority>
    </url>
  `).join('')}

  ${products.map(product => `
    <url>
      <loc>${baseUrl}/products/${product.slug}</loc>
      <lastmod>${product.updatedAt}</lastmod>
      <changefreq>daily</changefreq>
      <priority>0.9</priority>
    </url>
  `).join('')}
</urlset>
  `.trim()

  res.setHeader('Content-Type', 'text/xml')
  res.write(sitemap)
  res.end()

  return {
    props: {}
  }
}
```

## 實際應用案例

### 案例 1：電商網站 SEO

```javascript
// pages/products/[slug].js
import SEO from '../../components/SEO'

export default function ProductPage({ product }) {
  // 結構化資料
  const productSchema = {
    '@context': 'https://schema.org',
    '@type': 'Product',
    name: product.name,
    description: product.description,
    image: product.images,
    sku: product.sku,
    brand: {
      '@type': 'Brand',
      name: product.brand
    },
    offers: {
      '@type': 'Offer',
      url: `https://example.com/products/${product.slug}`,
      priceCurrency: 'TWD',
      price: product.price,
      availability: product.stock > 0
        ? 'https://schema.org/InStock'
        : 'https://schema.org/OutOfStock',
      priceValidUntil: '2025-12-31'
    },
    aggregateRating: product.reviews.length > 0 ? {
      '@type': 'AggregateRating',
      ratingValue: product.averageRating,
      reviewCount: product.reviews.length
    } : undefined,
    review: product.reviews.map(review => ({
      '@type': 'Review',
      reviewRating: {
        '@type': 'Rating',
        ratingValue: review.rating
      },
      author: {
        '@type': 'Person',
        name: review.author
      },
      reviewBody: review.comment
    }))
  }

  return (
    <>
      <SEO
        title={product.name}
        description={product.description}
        image={product.images[0]}
        url={`/products/${product.slug}`}
        type="product"
        keywords={product.tags.join(', ')}
      />

      <Head>
        <script
          type="application/ld+json"
          dangerouslySetInnerHTML={{ __html: JSON.stringify(productSchema) }}
        />
      </Head>

      <div>
        <h1>{product.name}</h1>
        <p>{product.description}</p>
        <p>NT$ {product.price}</p>
      </div>
    </>
  )
}
```

### 案例 2：部落格 SEO

```javascript
// pages/blog/[slug].js
export default function BlogPost({ post }) {
  const articleSchema = {
    '@context': 'https://schema.org',
    '@type': 'BlogPosting',
    headline: post.title,
    description: post.excerpt,
    image: post.coverImage,
    datePublished: post.publishedAt,
    dateModified: post.updatedAt || post.publishedAt,
    author: {
      '@type': 'Person',
      name: post.author.name,
      url: `https://example.com/authors/${post.author.slug}`
    },
    publisher: {
      '@type': 'Organization',
      name: '我的部落格',
      logo: {
        '@type': 'ImageObject',
        url: 'https://example.com/logo.png'
      }
    },
    mainEntityOfPage: {
      '@type': 'WebPage',
      '@id': `https://example.com/blog/${post.slug}`
    }
  }

  return (
    <>
      <SEO
        title={post.title}
        description={post.excerpt}
        image={post.coverImage}
        url={`/blog/${post.slug}`}
        type="article"
        author={post.author.name}
        publishedTime={post.publishedAt}
        keywords={post.tags.join(', ')}
      />

      <Head>
        <script
          type="application/ld+json"
          dangerouslySetInnerHTML={{ __html: JSON.stringify(articleSchema) }}
        />
      </Head>

      <article>
        <h1>{post.title}</h1>
        <p>作者：{post.author.name} | {new Date(post.publishedAt).toLocaleDateString('zh-TW')}</p>
        <img src={post.coverImage} alt={post.title} />
        <div dangerouslySetInnerHTML={{ __html: post.content }} />
      </article>
    </>
  )
}
```

### 案例 3：本地商家 SEO

```javascript
// pages/index.js
export default function HomePage() {
  const localBusinessSchema = {
    '@context': 'https://schema.org',
    '@type': 'LocalBusiness',
    name: '我的商店',
    image: 'https://example.com/store-photo.jpg',
    '@id': 'https://example.com',
    url: 'https://example.com',
    telephone: '+886-2-1234-5678',
    priceRange: '$$',
    address: {
      '@type': 'PostalAddress',
      streetAddress: '信義路五段7號',
      addressLocality: '台北市',
      postalCode: '110',
      addressCountry: 'TW'
    },
    geo: {
      '@type': 'GeoCoordinates',
      latitude: 25.033,
      longitude: 121.5654
    },
    openingHoursSpecification: [
      {
        '@type': 'OpeningHoursSpecification',
        dayOfWeek: ['Monday', 'Tuesday', 'Wednesday', 'Thursday', 'Friday'],
        opens: '09:00',
        closes: '18:00'
      },
      {
        '@type': 'OpeningHoursSpecification',
        dayOfWeek: 'Saturday',
        opens: '10:00',
        closes: '17:00'
      }
    ],
    sameAs: [
      'https://www.facebook.com/mystore',
      'https://www.instagram.com/mystore',
      'https://twitter.com/mystore'
    ]
  }

  return (
    <>
      <SEO
        title="首頁"
        description="我們提供最優質的產品和服務"
        keywords="商店, 台北, 產品"
      />

      <Head>
        <script
          type="application/ld+json"
          dangerouslySetInnerHTML={{ __html: JSON.stringify(localBusinessSchema) }}
        />
      </Head>

      <main>
        <h1>歡迎來到我的商店</h1>
      </main>
    </>
  )
}
```

## SEO 最佳實踐

### 1. 標題優化

```javascript
// ✅ 好的標題
<title>如何學習 Next.js - 完整指南 2024 | 我的部落格</title>

// ❌ 不好的標題
<title>文章</title>
<title>首頁首頁首頁首頁首頁首頁首頁首頁首頁首頁</title> // 太長
```

**規則：**
- 長度：50-60 字元
- 包含關鍵字
- 有吸引力
- 每個頁面唯一

### 2. 描述優化

```javascript
// ✅ 好的描述
<meta name="description" content="這篇文章將教你如何從零開始學習 Next.js，包括路由、資料獲取、部署等完整內容。適合有 React 基礎的開發者。" />

// ❌ 不好的描述
<meta name="description" content="文章" />
```

**規則：**
- 長度：150-160 字元
- 包含關鍵字
- 描述頁面內容
- 有行動呼籲

### 3. 圖片 Alt 屬性

```javascript
// ✅ 好的 alt
<img src="/product.jpg" alt="藍色運動鞋 - Nike Air Max 2024" />

// ❌ 不好的 alt
<img src="/product.jpg" alt="圖片" />
<img src="/product.jpg" /> // 缺少 alt
```

### 4. 語義化 HTML

```javascript
// ✅ 使用語義化標籤
<article>
  <header>
    <h1>文章標題</h1>
    <time datetime="2024-01-01">2024年1月1日</time>
  </header>
  <section>
    <p>內容...</p>
  </section>
  <footer>
    <p>作者：John</p>
  </footer>
</article>

// ❌ 全部用 div
<div>
  <div>文章標題</div>
  <div>2024年1月1日</div>
  <div>內容...</div>
</div>
```

### 5. 使用 Canonical URL

```javascript
// 避免重複內容問題
<Head>
  <link rel="canonical" href="https://example.com/blog/post-title" />
</Head>
```

## 監控與測試工具

### 1. Google Search Console
- 監控搜尋表現
- 檢查索引狀態
- 發現錯誤

### 2. Google PageSpeed Insights
```bash
https://pagespeed.web.dev/
```

### 3. Schema.org Validator
```bash
https://validator.schema.org/
```

### 4. Open Graph Debugger
```bash
# Facebook
https://developers.facebook.com/tools/debug/

# Twitter
https://cards-dev.twitter.com/validator
```

## 與 Vue3 的比較總結

| 功能 | Vue3 | Next.js |
|------|------|---------|
| Meta Tags 管理 | vue-meta 或 Nuxt head() | next/head（內建） |
| SSR SEO | 需要 Nuxt.js | 內建支援 |
| 結構化資料 | 手動添加 | 手動添加（相同） |
| Sitemap | 需要額外工具 | 可用 API Routes 生成 |
| 效能優化 | 需要手動配置 | 自動優化 |

## 小結

SEO 是網站成功的關鍵因素，Next.js 提供了完整的 SEO 工具和最佳實踐。對於 Vue3 開發者來說，Next.js 的 SEO 處理更加簡潔和強大，特別是結合 SSG/SSR 時，可以獲得完美的 SEO 效果。

**關鍵要點：**
1. 使用 `next/head` 管理 Meta Tags
2. 為每個頁面設定唯一的標題和描述
3. 使用結構化資料增強搜尋結果
4. 建立 SEO 組件以保持一致性
5. 定期測試和監控 SEO 表現
6. 使用 `next/image` 優化圖片 SEO
7. 確保網站效能（Core Web Vitals）
