# 第 15 章：圖片優化與 next/image 組件

## 概述

圖片通常是網頁中最大的資源，直接影響載入速度和使用者體驗。Next.js 的 `next/image` 組件提供了自動化的圖片優化功能。對於 Vue3 開發者來說，這是一個全新的概念，Vue 生態中沒有內建類似的解決方案。

## 傳統 img vs next/image

### 傳統 HTML img 標籤的問題

```html
<!-- Vue3 / 傳統 HTML -->
<img src="/images/photo.jpg" alt="照片" />
```

**問題：**
1. 沒有自動優化（檔案可能很大）
2. 沒有響應式處理（所有裝置載入相同大小）
3. 沒有懶載入（即使不在視窗內也會載入）
4. 可能造成 Layout Shift（CLS，影響 SEO）
5. 沒有現代格式支援（WebP、AVIF）

### Next.js Image 組件

```javascript
import Image from 'next/image'

export default function Page() {
  return (
    <Image
      src="/images/photo.jpg"
      alt="照片"
      width={800}
      height={600}
    />
  )
}
```

**優點：**
1. ✅ 自動優化圖片大小
2. ✅ 自動轉換為現代格式（WebP）
3. ✅ 自動懶載入
4. ✅ 防止 Layout Shift
5. ✅ 響應式圖片
6. ✅ 優先載入視窗內的圖片

## 基本使用

### 本地圖片

```javascript
import Image from 'next/image'
import profilePic from '../public/me.jpg'

export default function Profile() {
  return (
    <div>
      <h1>我的個人資料</h1>
      <Image
        src={profilePic}
        alt="個人照片"
        // width 和 height 會自動從導入的圖片獲取
        placeholder="blur" // 自動生成模糊預覽
      />
    </div>
  )
}
```

### 遠端圖片

```javascript
import Image from 'next/image'

export default function Avatar() {
  return (
    <Image
      src="https://example.com/avatar.jpg"
      alt="頭像"
      width={200}
      height={200}
    />
  )
}
```

**重要：** 使用遠端圖片時，需要在 `next.config.js` 中設定允許的網域：

```javascript
// next.config.js
module.exports = {
  images: {
    domains: ['example.com', 'images.unsplash.com', 'cdn.example.com'],
  },
}
```

## 圖片屬性詳解

### 必需屬性

```javascript
<Image
  src="/path/to/image.jpg"  // 圖片來源（必需）
  alt="描述"                 // 替代文字（必需，SEO 重要）
  width={800}                // 寬度（遠端圖片必需）
  height={600}               // 高度（遠端圖片必需）
/>
```

### layout 屬性（Next.js 12）

```javascript
// 1. intrinsic（預設）- 縮小以適應容器，但不放大
<Image
  src="/image.jpg"
  alt="圖片"
  width={800}
  height={600}
  layout="intrinsic"
/>

// 2. fixed - 固定大小，不響應
<Image
  src="/image.jpg"
  alt="圖片"
  width={200}
  height={200}
  layout="fixed"
/>

// 3. responsive - 響應式，縮小和放大以適應容器
<Image
  src="/image.jpg"
  alt="圖片"
  width={800}
  height={600}
  layout="responsive"
/>

// 4. fill - 填滿父容器（需要父容器設定 position: relative）
<div style={{ position: 'relative', width: '100%', height: '400px' }}>
  <Image
    src="/image.jpg"
    alt="圖片"
    layout="fill"
    objectFit="cover"
  />
</div>
```

### Next.js 13+ 的新語法

```javascript
// fill（取代 layout="fill"）
<div className="relative w-full h-96">
  <Image
    src="/image.jpg"
    alt="圖片"
    fill
    className="object-cover"
  />
</div>

// sizes 屬性用於響應式
<Image
  src="/image.jpg"
  alt="圖片"
  width={800}
  height={600}
  sizes="(max-width: 768px) 100vw, (max-width: 1200px) 50vw, 33vw"
/>
```

### 其他重要屬性

```javascript
<Image
  src="/image.jpg"
  alt="圖片"
  width={800}
  height={600}
  priority           // 優先載入（LCP 圖片）
  placeholder="blur" // 顯示模糊預覽
  blurDataURL="..."  // 自訂模糊預覽圖片
  quality={75}       // 圖片品質（1-100，預設 75）
  loading="lazy"     // 懶載入（預設）
  onLoadingComplete={(img) => console.log(img.naturalWidth)}
/>
```

## 實際應用案例

### 案例 1：Hero Banner

```javascript
// components/Hero.js
import Image from 'next/image'

export default function Hero() {
  return (
    <div className="relative w-full h-screen">
      <Image
        src="/images/hero-banner.jpg"
        alt="首頁橫幅"
        fill
        priority // Hero 圖片應該優先載入
        className="object-cover"
        quality={90} // 重要圖片使用較高品質
      />

      <div className="absolute inset-0 bg-black bg-opacity-40 flex items-center justify-center">
        <div className="text-center text-white">
          <h1 className="text-5xl font-bold mb-4">歡迎來到我們的網站</h1>
          <p className="text-xl mb-8">探索精彩內容</p>
          <button className="bg-white text-black px-8 py-3 rounded-lg font-semibold">
            開始探索
          </button>
        </div>
      </div>
    </div>
  )
}
```

### 案例 2：產品圖庫

```javascript
// components/ProductGallery.js
import Image from 'next/image'
import { useState } from 'react'

export default function ProductGallery({ images }) {
  const [selectedImage, setSelectedImage] = useState(0)

  return (
    <div className="max-w-4xl mx-auto">
      {/* 主圖 */}
      <div className="relative w-full h-96 mb-4">
        <Image
          src={images[selectedImage]}
          alt={`產品圖片 ${selectedImage + 1}`}
          fill
          className="object-contain"
          priority={selectedImage === 0} // 只有第一張優先載入
          sizes="(max-width: 768px) 100vw, 800px"
        />
      </div>

      {/* 縮圖 */}
      <div className="grid grid-cols-4 gap-4">
        {images.map((image, index) => (
          <div
            key={index}
            className={`relative h-24 cursor-pointer border-2 ${
              index === selectedImage ? 'border-blue-500' : 'border-gray-200'
            }`}
            onClick={() => setSelectedImage(index)}
          >
            <Image
              src={image}
              alt={`縮圖 ${index + 1}`}
              fill
              className="object-cover"
              sizes="200px"
            />
          </div>
        ))}
      </div>
    </div>
  )
}
```

```javascript
// 使用範例
export default function ProductPage() {
  const productImages = [
    '/products/shoe-1.jpg',
    '/products/shoe-2.jpg',
    '/products/shoe-3.jpg',
    '/products/shoe-4.jpg',
  ]

  return (
    <div>
      <ProductGallery images={productImages} />
    </div>
  )
}
```

### 案例 3：響應式網格

```javascript
// pages/gallery.js
import Image from 'next/image'

export default function Gallery({ images }) {
  return (
    <div className="container mx-auto px-4 py-8">
      <h1 className="text-4xl font-bold mb-8">圖片畫廊</h1>

      <div className="grid grid-cols-1 sm:grid-cols-2 md:grid-cols-3 lg:grid-cols-4 gap-4">
        {images.map((image, index) => (
          <div key={index} className="relative aspect-square">
            <Image
              src={image.src}
              alt={image.alt}
              fill
              className="object-cover rounded-lg hover:scale-105 transition-transform duration-300"
              sizes="(max-width: 640px) 100vw, (max-width: 768px) 50vw, (max-width: 1024px) 33vw, 25vw"
            />
          </div>
        ))}
      </div>
    </div>
  )
}

export async function getStaticProps() {
  const images = [
    { src: '/gallery/photo-1.jpg', alt: '照片 1' },
    { src: '/gallery/photo-2.jpg', alt: '照片 2' },
    { src: '/gallery/photo-3.jpg', alt: '照片 3' },
    // ... 更多圖片
  ]

  return {
    props: { images }
  }
}
```

### 案例 4：部落格文章頭圖

```javascript
// components/BlogPost.js
import Image from 'next/image'

export default function BlogPost({ post }) {
  return (
    <article className="max-w-4xl mx-auto">
      {/* 文章頭圖 */}
      <div className="relative w-full h-96 mb-8">
        <Image
          src={post.coverImage}
          alt={post.title}
          fill
          priority
          className="object-cover rounded-lg"
          placeholder="blur"
          blurDataURL={post.blurDataURL}
        />
      </div>

      <header className="mb-8">
        <h1 className="text-4xl font-bold mb-4">{post.title}</h1>
        <div className="flex items-center text-gray-600">
          <div className="relative w-10 h-10 mr-3">
            <Image
              src={post.author.avatar}
              alt={post.author.name}
              fill
              className="rounded-full object-cover"
            />
          </div>
          <div>
            <p className="font-semibold">{post.author.name}</p>
            <p className="text-sm">{post.publishedAt}</p>
          </div>
        </div>
      </header>

      <div className="prose max-w-none">
        {post.content}
      </div>
    </article>
  )
}
```

### 案例 5：電商產品卡片

```javascript
// components/ProductCard.js
import Image from 'next/image'
import Link from 'next/link'

export default function ProductCard({ product }) {
  return (
    <Link href={`/products/${product.id}`}>
      <div className="bg-white rounded-lg shadow-md overflow-hidden hover:shadow-xl transition-shadow cursor-pointer">
        {/* 產品圖片 */}
        <div className="relative w-full h-64">
          <Image
            src={product.image}
            alt={product.name}
            fill
            className="object-cover"
            sizes="(max-width: 640px) 100vw, (max-width: 1024px) 50vw, 33vw"
          />

          {/* 標籤 */}
          {product.isNew && (
            <span className="absolute top-2 right-2 bg-red-500 text-white text-xs font-bold px-2 py-1 rounded">
              新品
            </span>
          )}

          {product.discount && (
            <span className="absolute top-2 left-2 bg-yellow-400 text-gray-900 text-xs font-bold px-2 py-1 rounded">
              -{product.discount}%
            </span>
          )}
        </div>

        {/* 產品資訊 */}
        <div className="p-4">
          <h3 className="text-lg font-semibold mb-2 line-clamp-2">
            {product.name}
          </h3>

          <div className="flex items-center justify-between">
            <div>
              <span className="text-xl font-bold text-primary">
                NT$ {product.price.toLocaleString()}
              </span>
              {product.originalPrice && (
                <span className="ml-2 text-sm text-gray-400 line-through">
                  NT$ {product.originalPrice.toLocaleString()}
                </span>
              )}
            </div>

            <div className="flex items-center text-sm text-gray-600">
              <svg className="w-4 h-4 text-yellow-400 mr-1" fill="currentColor" viewBox="0 0 20 20">
                <path d="M9.049 2.927c.3-.921 1.603-.921 1.902 0l1.07 3.292a1 1 0 00.95.69h3.462c.969 0 1.371 1.24.588 1.81l-2.8 2.034a1 1 0 00-.364 1.118l1.07 3.292c.3.921-.755 1.688-1.54 1.118l-2.8-2.034a1 1 0 00-1.175 0l-2.8 2.034c-.784.57-1.838-.197-1.539-1.118l1.07-3.292a1 1 0 00-.364-1.118L2.98 8.72c-.783-.57-.38-1.81.588-1.81h3.461a1 1 0 00.951-.69l1.07-3.292z" />
              </svg>
              <span>{product.rating}</span>
            </div>
          </div>
        </div>
      </div>
    </Link>
  )
}
```

## 模糊預覽（Blur Placeholder）

### 自動生成（本地圖片）

```javascript
import Image from 'next/image'
import myImage from '../public/photo.jpg'

export default function Page() {
  return (
    <Image
      src={myImage}
      alt="照片"
      placeholder="blur" // 自動生成模糊預覽
    />
  )
}
```

### 自訂模糊預覽（遠端圖片）

```javascript
// 使用 plaiceholder 或 sharp 生成 base64
import Image from 'next/image'

export default function Page({ blurDataURL }) {
  return (
    <Image
      src="https://example.com/photo.jpg"
      alt="照片"
      width={800}
      height={600}
      placeholder="blur"
      blurDataURL={blurDataURL} // base64 編碼的小圖
    />
  )
}

export async function getStaticProps() {
  // 生成 blurDataURL 的邏輯
  const blurDataURL = await generateBlurDataURL('https://example.com/photo.jpg')

  return {
    props: { blurDataURL }
  }
}
```

## 圖片優化設定

### next.config.js 設定

```javascript
// next.config.js
module.exports = {
  images: {
    // 允許的遠端圖片網域
    domains: [
      'images.unsplash.com',
      'cdn.example.com',
      's3.amazonaws.com'
    ],

    // 或使用更靈活的 remotePatterns（Next.js 13+）
    remotePatterns: [
      {
        protocol: 'https',
        hostname: '**.example.com',
        pathname: '/images/**',
      },
    ],

    // 圖片大小（用於響應式圖片）
    deviceSizes: [640, 750, 828, 1080, 1200, 1920, 2048, 3840],
    imageSizes: [16, 32, 48, 64, 96, 128, 256, 384],

    // 支援的格式
    formats: ['image/webp', 'image/avif'],

    // 最小化快取時間（秒）
    minimumCacheTTL: 60,

    // 禁用靜態圖片導入（不推薦）
    disableStaticImages: false,
  },
}
```

## 效能最佳化

### 1. 使用 priority 屬性

```javascript
// ✅ LCP（最大內容繪製）圖片使用 priority
<Image
  src="/hero.jpg"
  alt="英雄圖片"
  fill
  priority // 立即載入，不懶載入
/>

// ❌ 不要對所有圖片使用 priority
```

### 2. 正確設定 sizes

```javascript
// ✅ 根據實際顯示大小設定 sizes
<Image
  src="/image.jpg"
  alt="圖片"
  width={800}
  height={600}
  sizes="(max-width: 640px) 100vw, (max-width: 1024px) 50vw, 800px"
/>

// ❌ 不設定 sizes 會載入過大的圖片
```

### 3. 調整品質

```javascript
// 裝飾性圖片可以降低品質
<Image
  src="/background.jpg"
  alt="背景"
  fill
  quality={50}
/>

// 重要圖片使用較高品質
<Image
  src="/product.jpg"
  alt="產品"
  width={800}
  height={600}
  quality={90}
/>
```

### 4. 使用適當的圖片格式

```
- 照片：JPEG / WebP
- 插圖、圖標：PNG / SVG
- 動畫：GIF / WebP / MP4
```

## 常見問題

### Q1: 如何使用 SVG？

```javascript
// 方法 1：直接使用 img 標籤
<img src="/icon.svg" alt="圖標" />

// 方法 2：導入為 React 組件（需要配置 SVGR）
import Logo from '../public/logo.svg'

export default function Page() {
  return <Logo />
}
```

### Q2: 圖片尺寸未知怎麼辦？

```javascript
// 使用 fill + 容器控制大小
<div style={{ position: 'relative', width: '100%', height: '400px' }}>
  <Image
    src="/image.jpg"
    alt="圖片"
    fill
    className="object-cover"
  />
</div>
```

### Q3: 如何處理動態圖片？

```javascript
// 從 API 獲取的圖片列表
export default function Gallery({ images }) {
  return (
    <div className="grid grid-cols-3 gap-4">
      {images.map((image) => (
        <div key={image.id} className="relative aspect-square">
          <Image
            src={image.url}
            alt={image.title}
            fill
            className="object-cover"
          />
        </div>
      ))}
    </div>
  )
}
```

## 與 Vue3 的比較

| 功能 | Vue3 | Next.js Image |
|------|------|---------------|
| 圖片優化 | 需要手動或使用第三方工具 | 內建自動優化 |
| 懶載入 | 需要手動實作或使用套件 | 預設啟用 |
| 響應式 | 手動設定 srcset | 自動生成 |
| 格式轉換 | 需要構建工具 | 自動轉換 WebP |
| Layout Shift | 需要手動處理 | 自動防止 |

## 最佳實踐總結

1. ✅ 永遠使用 `next/image` 而不是 `<img>`
2. ✅ 為 LCP 圖片添加 `priority`
3. ✅ 正確設定 `alt` 屬性（SEO）
4. ✅ 使用 `placeholder="blur"` 改善用戶體驗
5. ✅ 根據實際需求調整 `quality`
6. ✅ 正確設定 `sizes` 屬性
7. ✅ 在 `next.config.js` 中配置遠端圖片網域
8. ❌ 不要對所有圖片使用 `priority`

## 小結

`next/image` 是 Next.js 最強大的功能之一，它自動處理了圖片優化的所有複雜細節。對於 Vue3 開發者來說，這是一個巨大的生產力提升，不需要手動配置複雜的構建工具或第三方服務，就能獲得業界最佳的圖片效能。
